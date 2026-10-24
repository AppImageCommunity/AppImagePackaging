---
name: troubleshoot-appimage-runtime
description: Use when a built AppImage fails, crashes, hangs or misbehaves only outside the machine that built it - "error while loading shared libraries", "cannot open shared object file", "version GLIBC_2.x not found", "symbol lookup error", a silently rejected Qt/GTK platform plugin, a wrong ELF interpreter, an app that reads the wrong files, themes or plugins, or one that never shows a window without any error. Covers reading dynamic-linker errors precisely, what ldd, readelf, patchelf, objdump and file each tell you, strace ENOENT tracing for ignored bundled files, QT_DEBUG_PLUGINS and GTK_DEBUG, attaching gdb to a stuck process in a container for a real backtrace, headless hangs from a missing D-Bus or AT-SPI service, GLVND's split OpenGL libraries, CI checks that swallow their own tool's errors, binary-patching hardcoded absolute paths, and glibc-symbol-version and ldconfig-cache gotchas on Alpine/musl. Pairs with make-standalone-appdir, build-appimage-ci and verify-appimage-portability.
---

# Troubleshoot an AppImage at Runtime

`make-standalone-appdir` covers building an AppDir correctly. This skill
covers what to do when it's already built and something is still wrong -
the app won't start on another machine, starts but looks wrong, or crashes
only in CI/on a different distro than the one that built it. Work through
these in order; each tool answers a specific, narrow question, and running
them in the wrong order wastes time guessing.

## Step 0: run it directly, not through the AppImage stub

Debugging the compressed `.AppImage` file itself adds a layer (FUSE mount or
`--appimage-extract-and-run`) that tells you nothing extra and can obscure
exit codes. Extract once and work against the AppDir directly:

```sh
./YourApp-x86_64.AppImage --appimage-extract
cd squashfs-root
./AppRun ; echo "exit: $?"
```

Note the exit code before doing anything else - it narrows the search fast:

| Exit code | Usual meaning |
|---|---|
| 127 | Shell couldn't find/exec the interpreter at all, or a dynamic-linker "cannot open shared object file" for a library needed just to *start* |
| 126 | Found the file but couldn't execute it - permissions, or wrong ELF interpreter path |
| 133 / "Exec format error" | Wrong architecture (ELF class/machine mismatch - e.g. an aarch64 binary on x86_64) |
| 132 (SIGILL) | Binary uses instructions the CPU doesn't have (e.g. built with `-march=native` on a fancier CPU than the target) |
| 139 (SIGSEGV) | Crashed after starting - usually a real bug, or ABI mismatch from a `gcompat`-class shim (see below), not a "missing file" problem |
| 0 but nothing visible | Toolkit started but didn't render - see `verify-appimage-portability`'s retry/blank-detection section, or a display/platform-plugin problem below |

## Reading dynamic-linker error text precisely

The exact wording tells you which of several different problems you have -
don't pattern-match "it says .so, must be a missing library" for all of
them:

- `error while loading shared libraries: libFoo.so.N: cannot open shared
  object file: No such file or directory` - the library genuinely isn't
  found anywhere the dynamic linker looked. Missing from the bundle, or an
  `LD_LIBRARY_PATH`/RPATH problem (see below) - not a symbol issue.
- `libFoo.so.N: version `GLIBC_2.38' not found (required by ...)` - the
  library **is** found, but it needs dynamic-linker symbol versions your
  libc doesn't provide. This is a libc-completeness problem (glibc too old,
  or a musl compat shim covering only part of glibc), not a missing-file
  problem - see "Symbol-version gaps" below. Adding the same library again
  or "reinstalling the package" does nothing for this.
- `<binary>: /lib64/ld-linux-x86-64.so.2: No such file or directory` (and
  `file <binary>` clearly shows the file exists) - the ELF interpreter
  (the dynamic linker itself) named in the binary's `PT_INTERP` segment
  doesn't exist at that path on this system. Classic Alpine/musl symptom for
  a glibc-linked binary with no compat layer installed yet.
- `symbol lookup error: <binary>: undefined symbol: some_symbol` - a
  library loaded, but a different, incompatible version of a *dependency*
  is what actually got picked up (wrong one earlier in `LD_LIBRARY_PATH`, or
  a host system library shadowing a bundled one it's not ABI-compatible
  with).

## The toolbox: what each command actually tells you

- **`file <binary>`** - architecture, ELF class (32/64-bit), and (further
  down its output on most `file` versions) the interpreter path. Always run
  this first on a "won't execute" mystery.
- **`readelf -d <binary>`** (or `-l` for the interpreter/program headers) -
  lists `NEEDED` entries (direct link-time dependencies), `RPATH`/`RUNPATH`,
  and the interpreter. This is what the binary *asks for* - not proof any of
  it is satisfiable on this system.
- **`patchelf --print-needed <binary>`, `--print-rpath <binary>`** - same
  information as `readelf -d`, easier to script against; `patchelf
  --set-rpath '$ORIGIN/../lib'` is how you fix a wrong one (see
  make-standalone-appdir for keeping this consistent with your actual
  install layout).
- **`ldd <binary>`** - actually *resolves* every dependency using the
  current environment (`LD_LIBRARY_PATH`, ld.so.cache, RPATH) and shows
  "not found" for anything it can't. More useful than `readelf` for "is this
  actually satisfiable right now", but two caveats: it executes the dynamic
  linker against the file (never run it on an untrusted binary), and it
  reports against *your current shell's* environment - if you haven't
  exported the same `LD_LIBRARY_PATH` your `AppRun` sets, `ldd` will show
  failures that don't happen at real runtime (and vice versa). Reproduce
  `AppRun`'s environment before trusting `ldd`'s answer.
- **`objdump -T <binary>` / `objdump -T <library>`** - lists dynamic symbols
  including the *symbol version* each undefined reference needs (the
  `GLIBC_2.38` in `some_symbol@GLIBC_2.38`). To find the single highest
  glibc version a binary or library actually requires:
  ```sh
  objdump -T <binary> | grep -oP 'GLIBC_2\.\d+' | sort -V -u | tail -1
  ```
  Compare that against what your compat layer actually provides (see below)
  *before* you even try running it on the target - this check is cheap and
  catches the problem before a confusing crash does.
- **`nm -D --undefined-only <binary>`** - plain list of undefined symbols,
  no version info; faster gut-check for "does this pull in anything unusual"
  before reaching for `objdump -T`.

## strace: the single most useful tool for "it ignored my bundled files"

When the app starts but reads the wrong theme, ignores a bundled schema,
can't find a plugin directory, or silently falls back to some default,
`ldd`/`readelf` won't help - none of that is a link-time dependency, it's a
runtime `open()`/`stat()` call that failed and was silently handled. Trace
it directly:

```sh
strace -f -e trace=openat,open,stat,access -o trace.log ./AppRun
grep ENOENT trace.log | grep -v '\.so'   # filter out normal library-search misses
```

Every `ENOENT` on a path under your AppDir (or where you expected one) is a
concrete, unambiguous lead - far faster than guessing from symptoms. `ltrace`
(traces library calls, including `dlopen`/`dlsym`) can show *which* dlopen
call is failing and why, but is unreliable on modern optimized/stripped
binaries - prefer `strace`'s file-open view first and only reach for
`ltrace` if you specifically need to see a `dlopen()` return value.

## `LD_DEBUG` (glibc only)

`LD_DEBUG=libs ./AppRun` (or `LD_DEBUG=all` for very verbose output) makes
glibc's dynamic linker narrate every search path it tries and why it
rejected each candidate - the most direct way to understand an RPATH/
`LD_LIBRARY_PATH` ordering problem. This is a **glibc** feature: it does
nothing under musl's dynamic linker (Alpine's default libc), and does
nothing useful for a glibc-linked binary running under `gcompat` (which
isn't glibc's own linker). On musl/gcompat targets, use `strace` instead.

## Toolkit-specific debug switches

- **Qt**: `QT_DEBUG_PLUGINS=1` - explains exactly why each candidate
  platform/imageformat/iconengine plugin was accepted or rejected (wrong
  ABI tag, missing dependency, etc.), instead of Qt's default "no platform
  plugin could be found" with zero detail. Also try `QT_QPA_PLATFORM=xcb`
  (or `minimal`/`offscreen` to isolate "does the app logic even start" from
  "does the display backend work") explicitly rather than relying on
  auto-detection.
- **GTK/GLib**: `GTK_DEBUG=all` and `G_MESSAGES_DEBUG=all` surface module-
  loading, theme-loading, and icon-loading diagnostics GTK normally
  swallows.
- A crash with no useful message: `gdb --batch -ex run -ex bt --args
  ./AppRun` for a backtrace, even against a stripped binary (frame names
  may be missing, but library-level attribution and the crash address
  usually aren't).

## Diagnosing a silent hang (not a crash): a real gdb thread backtrace

The hardest case is an app that neither crashes nor prints an error - it
just never shows a window. `strace`, `ldd`, `objdump -T`, none of the tools
above apply, because nothing *failed*; something is just stuck. Guessing env
vars one at a time here can burn several CI round trips for nothing.
Instead, attach to the still-running process and ask it directly where it
is:

```sh
./squashfs-root/AppRun &
APP_PID=$!
sleep 15   # or however long your retry budget is
if kill -0 "$APP_PID" 2>/dev/null; then
  gdb -p "$APP_PID" --batch -ex "thread apply all bt"
fi
```

Two things that trip this up specifically inside a CI container:

- **Docker's default seccomp profile blocks `ptrace`.** In a GitHub Actions
  `container:` job this shows up as gdb failing to attach with a permission
  error. Add `options: --cap-add=SYS_PTRACE` to the job's `container:` block.
- **The PID you launched may not be the PID you need to attach to.** Some
  `AppRun` scripts do `exec -a "AppRun" "$HERE/usr/bin/realbinary" "$@"` to
  keep a friendly process name in `ps`, which can make the main thread's
  `comm` show as `AppRun` even though it's genuinely running your
  application - don't assume you attached to the wrong process just because
  the thread name looks like the launcher script. If in doubt, check that
  the backtrace contains frames from your own bundled libraries (e.g.
  `libQt6Core.so.6`) - if it does, you're in the right process regardless of
  what the thread is named.

Read every thread, not just the main one - a real example: the main thread
was idle in its normal Qt event loop (unremarkable), but a **second**
thread named `QDBusConnection` was also present and blocked, in a container
that had no D-Bus daemon at all. That thread's mere existence, next to
`xwininfo -tree -root` showing zero windows had ever been created (not "a
blank window" - none at all), was the actual lead: see "Headless container
hangs" below for what it meant and the fix. The backtrace doesn't have to be
fully symbolized to be useful - which threads exist and what they're named
is often as informative as the frames themselves.

## Headless container hangs: a service Qt/GTK assumes exists, but doesn't

A minimal verification container deliberately has almost nothing installed.
Some of what's missing isn't a library the linker complains about - it's a
background service some toolkit code silently assumes it can reach, and
instead of failing when it can't, it hangs trying. Two confirmed cases:

- **D-Bus session bus autolaunch.** If `DISPLAY` is set (as it will be,
  since you're running under Xvfb) but there is no `dbus-daemon` process and
  no `DBUS_SESSION_BUS_ADDRESS` set, libdbus doesn't just fail - it attempts
  an X11-property-based autolaunch handshake to start a session bus, and
  that handshake has nothing to talk to in a container with no
  `dbus-daemon`/`dbus-launch` binaries at all. Anything that opens a D-Bus
  connection (directly, or indirectly - Qt6 itself can reach for the
  `org.freedesktop.portal.Settings` D-Bus service for things like dark-mode
  detection, independent of anything the app does explicitly) can then hang
  indefinitely instead of erroring. Fix: set the environment variable to
  libdbus's own "no bus, don't try" sentinel before launching:
  ```sh
  export DBUS_SESSION_BUS_ADDRESS=disabled:
  ```
- **The accessibility (AT-SPI) bridge.** Similarly, with no AT-SPI registry
  daemon running, Qt's accessibility bridge can block trying to reach it
  over D-Bus at startup instead of failing fast. Fix: `export
  QT_ACCESSIBILITY=0` (GTK's equivalent is `export NO_AT_BRIDGE=1`).

Both look identical from the outside - the app "just hangs", no error, no
crash - which is exactly why the gdb backtrace above matters: it's what
tells you *which* missing service to suspect, instead of pattern-matching
from a growing list of "things headless containers don't have."

## Prior art: AppImage's own catalog project has done this for years

`AppImage/appimage.github.io`'s `code/` directory (the scripts behind the
AppImageHub catalog, which test-launches and screenshots essentially every
public AppImage) is worth reading directly for more of this - in
particular:
- `worker.sh` gives a launched app a generous grace period (10s, then polls
  up to 20s more) and detects a window's existence with `xwininfo -tree
  -root` rather than pixel analysis, explicitly distinguishing "the process
  already exited" (`kill -0`) from "still running, still no window" - the
  same distinction this skill's Step 0 exit-code table and the hang section
  above are built around.
- `take-screenshot.sh` handles the case where a window exists but its own
  contents can't be read directly (`import -window <id>` failing with
  "unable to read X window image" - a known issue for OpenGL/SDL surfaces),
  falling back to capturing the screen region under that window, then the
  whole screen.
- `check-libc.sh` is a scriptable version of this skill's `objdump -T`
  glibc-version check, but scans **every** ELF file in the AppDir (not just
  the main binary) to compute whether the whole payload is self-contained
  and, if not, the single newest glibc symbol version anything in it needs -
  a good starting point for an automated self-containment gate in CI.

## Hardcoded absolute paths: qt.conf first, binary patching as last resort

A toolkit or app compiled with `CMAKE_INSTALL_PREFIX=/usr` bakes `/usr/...`
into its binary as the default search path for plugins, data files, or
schemas. Inside an AppImage that isn't actually installed to `/usr`, that
compiled-in path is simply wrong on any system where something else already
lives at `/usr`.

**For Qt**, don't patch the binary - ship a `qt.conf` next to the executable
(linuxdeploy-plugin-qt does this for you automatically):

```ini
[Paths]
Prefix = ..
```

Qt reads this at startup and rewrites its compiled-in prefix relative to the
executable's own location, before it ever tries to load a plugin. This is
the intended, supported override mechanism - always prefer it over touching
the binary.

**For anything without an equivalent override file** (many GTK apps, custom
C/C++ code that calls a hardcoded path directly), the classic AppImage trick
is patching the compiled-in string *in place*, replacing `/usr` with a
same-length relative prefix:

```sh
sed -i 's|/usr|././|g' AppDir/usr/bin/your-binary
```

Why `././` specifically: it's exactly 4 bytes, identical to `/usr` - so the
`.rodata` string table shifts nothing and no other embedded string moves.
`././share/foo` used as a path is interpreted as `./` + `./share/foo`,
i.e. relative to the process's current working directory. **This only works
if `AppRun` `cd`s into the AppDir root before exec-ing the real binary** -
without that `cd`, the now-relative path resolves against whatever
directory the AppImage happened to be launched from, which is wrong just as
often as the original absolute path was:

```sh
#!/bin/sh
HERE="$(dirname "$(readlink -f "${0}")")"
cd "$HERE" || exit 1
exec "$HERE/usr/bin/your-binary" "$@"
```

Ground rules for this technique:
- The replacement **must** be the exact same byte length as the original,
  including that you're not introducing or removing a NUL terminator
  position - a length mismatch corrupts every string after it in the same
  section. Verify with `strings -a binary | grep -c '/usr'` before and
  `grep -c '\./\./'` after, on a **copy**, before shipping it.
- Audit what you're about to patch first - `strings -a binary | grep -E
  '^/(usr|etc|opt)/'` lists every candidate compiled-in absolute path so you
  patch deliberately, not by trial and error from crash to crash.
- Only patch paths that are genuinely wrong inside the bundle. Don't
  reflexively replace every `/usr` occurrence in a binary - some may be
  legitimate references to the *host* system (e.g. a fallback icon theme
  path that's supposed to also check the real `/usr/share/icons`).

## Symbol-version gaps (Alpine/musl + a glibc-linked payload)

If `objdump -T` (above) shows the binary or a bundled library needs a
`GLIBC_2.3x` symbol version, and your compat layer doesn't provide it, no
amount of adding "the right package" fixes it - you need a more complete
compat layer:

1. `gcompat` (Alpine's official shim) implements a subset of glibc and
   commonly lacks newer symbol versions (`GLIBC_2.38`-era additions like
   `__isoc23_strtol`, `renameat2`, `close_range`, `pthread_cond_clockwait`,
   etc.) if the payload was built against a newer glibc than gcompat
   targets.
2. The long-standing unofficial `sgerrand/alpine-pkg-glibc` repo provides a
   more complete real glibc build, but has historically lagged (e.g. stuck
   around glibc 2.35) - check its current version against what
   `objdump -T` said you need before assuming it will help.
3. Actively maintained forks of that packaging (check for one tracking a
   current glibc release) are worth checking specifically because "some
   third-party glibc package for Alpine" is not interchangeable with
   "a third-party glibc package new enough for this specific binary" -
   verify the version, don't just install the first one you find.

Whichever compat layer you use is still a third-party apk repo and signing
key outside Alpine's own - call that out explicitly wherever you add it,
same as in verify-appimage-portability.

## The ldconfig-cache-staleness trap in containers

If you create a new library symlink (e.g. to satisfy a soname the bundled
app expects, like pointing `libGLX.so.0` at an already-installed
`libGL.so.1`) and the dynamic linker still reports it missing, check whether
something already ran `ldconfig` (directly, or as a package post-install
trigger) *before* you created the symlink. glibc's dynamic linker consults
`ld.so.cache` first, which was already built without your new symlink in
it - re-running `ldconfig` is one fix, but inside a short-lived CI container
step it's simpler and more reliable to just export the symlink's directory
via `LD_LIBRARY_PATH` for the launch command, since that's consulted
directly at load time and never depends on a cache being current.

## GResource corruption (GTK) - one-line reminder

If a GTK3/GTK4 app's built-in icons/resources vanish or the app crashes on
icon lookup only in the packaged AppImage, check whether anything ran
`appimagetool -s deploy` or another generic patchelf-based auto-deploy over
a binary containing an embedded `.gresource` ELF section - patchelf can
corrupt it. See make-standalone-appdir's Golden Rule 1 for the fix (assemble
the AppDir manually instead).

## GLVND's split OpenGL libraries: a missing-library gap that hides itself

If the app touches GL/EGL at all, don't bundle just `libGL.so.1` and call it
done. On a GLVND system (Ubuntu/Debian), `libGL.so.1` is a thin dispatcher
that itself needs `libGLX.so.0`, `libGLdispatch.so.0`, and (for GL core
profiles) `libOpenGL.so.0` as separate libraries - `readelf -d libGL.so.1`
shows them as its own `NEEDED` entries, the same as any other dependency.
The trap: a test container built on the same distro family as the build
machine (e.g. a Debian verification job for an app built on Ubuntu) often
already has system copies of the other three installed as a transitive
dependency of something else entirely, which silently satisfies the missing
`--library=` entries and hides the gap in the bundle. A stricter, more
minimal container (Alpine, or any distro without that particular incidental
package) has no such copies, and `AppRun` fails outright with a plain
"error while loading shared libraries" for each missing piece - discovered
one at a time, in whatever order the dynamic linker happens to need them.
Bundle the whole small GLVND set together up front rather than treating each
as a surprise. See verify-appimage-portability for why running a second,
stricter verification target is what actually catches this class of gap.

## Blank window, no crash: the opposite of GLVND (over-bundling the display stack)

GLVND above is *under*-bundling; this is the mirror image. The app starts,
opens a window, and never crashes, but the window is **blank/white**. GTK and
WebKitGTK apps (GTK apps, wxWebView, Electron, Tauri) log
`Could not create default EGL display: EGL_BAD_PARAMETER` or
`AcceleratedSurfaceDMABuf was unable to construct a complete framebuffer`. The
tell: it's fine on the (older) build box and breaks on a newer host - Mesa 25+,
or a pure-Wayland session.

The cause is usually that the AppDir **bundled the host's windowing/EGL client
libraries** - `libwayland-client`/`libwayland-egl`, `libxkbcommon`, `libxcb*`
(and sometimes `libEGL`/`libgbm`) - which then shadow the host's copies. Those
libraries must match the host's compositor and Mesa/EGL driver; a bundled older
copy against a newer host cannot create an EGL display. Confirm by listing what
the AppImage bundles (`--appimage-extract`, then look under `usr/lib` for
`libwayland*`, `libxkbcommon*`, `libxcb*`) and by re-running with those removed
or with `LD_LIBRARY_PATH` pointed away from them.

The fix is a build-time one - stop bundling the display/EGL stack and don't pin
`GDK_BACKEND`; see make-standalone-appdir's "Pitfall 13". Setting
`WEBKIT_DISABLE_COMPOSITING_MODE=1` and `WEBKIT_DISABLE_DMABUF_RENDERER=1`
forces CPU rendering and will make it paint, but that only confirms the
diagnosis (and is the right setting for a headless, GPU-less CI screenshot) -
it is not the fix for a shipped AppImage, which should render on the GPU.

## Verify your own test harness before blaming the app

Twice in one debugging session, the actual bug was in the *check*, not the
app: an `identify` call used `%[fx:image.stdDev]`, which is not a real
ImageMagick property (the correct one is `standard_deviation`) - it errored
every time, and a naive `2>/dev/null || echo 0` fallback turned that error
into "0", indistinguishable in the log from a genuinely blank screenshot.
Every earlier "still blank" verdict from that job was consistent with the
check itself being broken the whole time, not the app failing to render.
Separately, a bare pixel-stddev check without first confirming the app's own
window exists produced the opposite mistake - a **false pass**: a window
manager's taskbar and desktop chrome are non-flat by themselves, so a build
where the app crashed on startup (a missing bundled library) still showed
enough stddev from icewm's own furniture to clear the threshold and report
success.

Two general habits this argues for, useful for any CI check, not just this
one: **never silently swallow a diagnostic tool's own error output** - print
it, so "the check itself failed" and "the thing being checked is broken"
can't collapse into the same signal; and **require a positive precondition
that only the real subject can satisfy** (here, a window whose `WM_CLASS`
actually names the app) before trusting a derived metric (pixel stddev) that
something else entirely could also produce. A check that can fail silently
into looking like the exact failure you're hunting will cost you a lot more
debugging time than the extra few lines to make it fail loudly instead.
