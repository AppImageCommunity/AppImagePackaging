---
name: verify-appimage-portability
description: Use when a GitHub Actions AppImage build goes green and you need to prove the AppImage runs somewhere other than the box that built it - running it headless in a minimal container as the portability test, why a minimal glibc distro (debian-slim) is the primary release gate and a stricter Alpine/musl target is worth adding once the bundle is self-contained (it exposes gaps such as GLVND's split libGL/libGLX/libGLdispatch/libOpenGL that a glibc container's own copies mask), keeping test containers free of app libraries, Xvfb plus a window manager plus screenshotting, requiring a window owned by the app before trusting pixels, retrying until it paints, checking /proc/<pid>/maps for host copies of bundled libraries, a deliberately hostile container with the host's own toolkit, telling a silent hang (missing D-Bus/AT-SPI) from a crash with gdb and xwininfo, and publishing the screenshot into the GitHub Release. Pairs with build-appimage-ci, troubleshoot-appimage-runtime and make-standalone-appdir.
---

# Verify an AppImage Actually Runs (Not Just Builds)

A green "build the AppImage" CI job only proves the packaging step didn't
crash. It proves nothing about whether the AppImage runs on a machine that
isn't the exact Ubuntu image it was built on - which is the entire point of
shipping an AppImage. Prove it by launching the built AppImage headless in a
minimal container and screenshotting it.

See also: [build-appimage-ci](../build-appimage-ci/SKILL.md) for producing
the AppImage this skill verifies - including its "run the watch-fix-push loop
in small subagents" section, which applies just as much here: discovering
the minimal package set below is exactly the same iterate-on-a-real-error
loop, and is worth delegating the same way;
[troubleshoot-appimage-runtime](../troubleshoot-appimage-runtime/SKILL.md)
for the tools and technique behind diagnosing each "missing library" or
blank-screen failure this job turns up (strace, `objdump -T`, symbol-version
gaps, hardcoded-path patching) rather than guessing package names; and
[make-standalone-appdir](../make-standalone-appdir/SKILL.md) for the
low-level bundling rules behind most of the "missing library" errors you'll
hit here.

## Pick your primary target: a minimal glibc distro

AppImage's actual portability promise is "runs on other glibc-based
distros" - that's what the format and tooling (linuxdeploy, appimagetool)
are designed around. Make **that** your release gate: `debian:trixie-slim`
(or similar) has no desktop packages preinstalled, needs no compat layer at
all, and tests exactly the guarantee AppImage is actually making. It's also
the cheaper job to get green, so get it solid first.

Once it's genuinely self-contained (see "Bundle what the app needs" below),
add a **second** job against Alpine (musl) too - not as a stretch goal that
might be a dead end, but because it's a stricter test that catches real
bundling gaps the glibc target can hide. A first attempt at Alpine, before
the AppImage bundled its own libraries, hit a hard wall: the payload's
glibc-linked libraries ended up loading Alpine's own **musl-compiled**
system libraries (X11, mesa, etc.) into the same process, and that mixing
produced an indefinite, silent hang. That was a real dead end - but it was
caused by an *incomplete bundle*, not by musl and glibc being fundamentally
incompatible in one process. Once every library the app needs is actually
bundled (so the process never touches Alpine's own X11/GL/etc. at all),
Alpine works - and it's worth keeping precisely because Debian's container
had incidental system copies of a few GLVND libraries that silently
papered over gaps in the bundle list; Alpine's more minimal container had
no such copies and surfaced each one as a clean, fixable missing-library
error. Treat the two jobs as a pair: Debian is the gate that must pass to
release, Alpine is the stricter test that tends to find what Debian's
container was accidentally covering for - see "A second, stricter target:
Alpine" near the end for the exact recipe.

## Shape of the job

The AppImage is meant to be **self-standing**: every library the app needs
must be bundled into it by the `build` job (see build-appimage-ci's
`--library=` bundling), never installed as a package in this verification
container. This container should end up needing almost nothing beyond
genuine test infrastructure - if you find yourself adding an X11/GL/toolkit
library *here* to make `AppRun` start, that means the build job's bundling
is incomplete; fix it there, not here. A stock minimal system with nothing
app-specific preinstalled is the entire point of this test.

```yaml
screenshot-debian:
  needs: build
  runs-on: ubuntu-24.04
  container:
    image: debian:trixie-slim
    # Needed for the gdb backtrace step below (Docker's default seccomp blocks ptrace).
    options: --cap-add=SYS_PTRACE
  steps:
    - uses: actions/download-artifact@v4
      with:
        name: <your-build-job's-artifact-name>

    - name: Install test infrastructure (display, screenshot tooling - no app libraries)
      run: |
        apt-get update
        apt-get install -y --no-install-recommends \
          fonts-dejavu-core xvfb ffmpeg imagemagick ca-certificates x11-utils gdb icewm

    - name: Launch under Xvfb and screenshot it
      run: |
        chmod +x YourApp-x86_64.AppImage
        ./YourApp-x86_64.AppImage --appimage-extract   # no /dev/fuse in a container either

        Xvfb :99 -screen 0 1280x800x24 &
        XVFB_PID=$!
        sleep 2
        export DISPLAY=:99
        # A window manager isn't optional set-dressing: without one to complete the
        # ICCCM/EWMH handshake around mapping the window, some toolkits never receive
        # the Expose event that triggers the first paint - the window exists but stays
        # forever blank. (Prior art: AppImage's own catalog-testing project runs one too.)
        icewm > /dev/null 2>&1 &
        ICEWM_PID=$!
        sleep 2
        export QT_QPA_PLATFORM=xcb   # or leave unset for GTK's default X11 backend
        # This container has no D-Bus daemon and no AT-SPI registry. Without these,
        # Qt/GTK code that reaches for either can hang instead of failing fast - see
        # troubleshoot-appimage-runtime's "Headless container hangs" section.
        export QT_ACCESSIBILITY=0
        export DBUS_SESSION_BUS_ADDRESS=disabled:
        ./squashfs-root/AppRun &
        APP_PID=$!

        # Retry for up to 15s instead of one blind sleep-then-grab: a fixed sleep is
        # either wastefully long or, worse, too short and silently ships a blank
        # screenshot as if it were a passing test. Require an ACTUAL window owned by
        # the app (matched on WM_CLASS) before ever trusting the pixel check below -
        # a window manager's own taskbar/desktop chrome is non-flat by itself, so a
        # run where the app crashed on startup can still show non-trivial stddev with
        # nothing but the window manager's own furniture on screen. This is not a
        # hypothetical: it produced a real false pass once a missing bundled library
        # crashed the app immediately, with icewm's desktop alone driving the stddev
        # over threshold.
        RENDERED=0
        WINDOW_FOUND=0
        for _ in $(seq 1 7); do
          sleep 2
          if xwininfo -tree -root 2>/dev/null | grep -q '"YourAppWmClass")'; then
            WINDOW_FOUND=1
          else
            echo "app window not found yet" >&2
            continue
          fi
          ffmpeg -y -f x11grab -video_size 1280x800 -i "$DISPLAY" -frames:v 1 screenshot.png
          # The real property is `standard_deviation`, not `image.stdDev` - an invalid
          # fx expression here would silently error and, if you swallow stderr with a
          # naive `|| echo 0` fallback, look identical to a genuinely blank frame in
          # every later log. Don't swallow it; print it if it happens.
          STDDEV=$(identify -format "%[fx:standard_deviation]" screenshot.png 2>&1) || {
            echo "identify failed: $STDDEV" >&2; STDDEV=0; }
          echo "window found, frame stddev: $STDDEV" >&2
          if awk -v v="$STDDEV" 'BEGIN{exit !(v > 0.02)}'; then RENDERED=1; break; fi
        done

        # Distinguish "still running but never painted" from "already exited" before
        # killing it - and if still running, get a real backtrace instead of guessing
        # at the next env var. See troubleshoot-appimage-runtime for both techniques.
        echo "xwininfo window list:" >&2
        timeout 10 xwininfo -tree -root >&2 2>&1 || true
        if kill -0 "$APP_PID" 2>/dev/null; then
          echo "AppRun is still running - backtracing it:" >&2
          timeout 15 gdb -p "$APP_PID" --batch -ex "thread apply all bt" >&2 2>&1 || true
        else
          echo "AppRun has already exited" >&2
        fi

        kill "$APP_PID" 2>/dev/null || true
        kill "$ICEWM_PID" 2>/dev/null || true
        kill "$XVFB_PID" 2>/dev/null || true

        if [ "$WINDOW_FOUND" -ne 1 ]; then
          echo "::error::no window owned by the app ever appeared in 15s - it likely crashed or failed to start"
          exit 1
        fi
        test -s screenshot.png
        if [ "$RENDERED" -ne 1 ]; then
          echo "::error::the app's window exists but screenshot.png is still blank/solid-color after 15s"
          exit 1
        fi

    - uses: actions/upload-artifact@v4
      if: always()
      with:
        name: screenshot
        path: screenshot.png
        if-no-files-found: ignore
```

No `libc6-compat`, no glibc-compat shim, no musl-specific library juggling -
a glibc-on-glibc target removes an entire category of problems for free.
Upload the screenshot unconditionally (`if: always()`), not only on success -
a genuinely failing frame is worth looking at directly, and you can't if the
job never produced the artifact.

## Bundle what the app needs; only test infrastructure lives here

AppImage tooling (linuxdeploy, appimagetool's own bundling) generally
blacklists "always assume this is already on the host" libraries - core
glibc, and core X11 client libraries like `libX11`/`libxcb` - on the theory
that any Linux desktop already has them. That assumption is exactly what a
self-standing AppImage shouldn't need to make. So when `AppRun` fails with a
missing library (`libx11-6`/`libxcb1`/`libxkbcommon0`/`libxkbcommon-x11-0`,
X11 session-management's `libsm6`/`libice6`, GL/EGL's
`libgl1`/`libegl1`/`libopengl0`, Qt's xcb-plugin helpers like
`libxcb-icccm4`/`libxcb-image0`/`libxcb-keysyms1`/`libxcb-render-util0`/
`libxcb-cursor0`, or a transitive dependency of any of those like
`libgpg-error0`), the fix belongs in the **build** job: add the
corresponding runtime package there (so its `.so` exists to copy from) and
pass it to linuxdeploy as an explicit `--library=<resolved path>` - see
build-appimage-ci's bundling section for the exact mechanism (resolving
each soname via `ldconfig -p` rather than hardcoding a lib directory).

**GLVND's split libraries are an easy set to under-bundle.** If the app
touches GL/EGL at all, `libGL.so.1` is not one self-sufficient library on a
GLVND system (which Ubuntu/Debian are) - it's a thin dispatcher that itself
needs `libGLX.so.0`, `libGLdispatch.so.0`, and (for GL core profiles)
`libOpenGL.so.0` as separate vendor/support libraries. Bundling only
`libGL.so.1` looks fine on a **glibc** test target, because that container's
own base image or one of its other packages often already has system copies
of the other three sitting around, quietly satisfying the dependency for
you - the gap in your bundle is invisible. It stops being invisible on a
stricter target with no such incidental copies (see "A second, stricter
target: Alpine" below): each missing piece then surfaces one at a time as
its own "error while loading shared libraries", in the order the dynamic
linker happens to need them. When in doubt, bundle the whole small GLVND set
together rather than discovering it as several separate round trips.

This container should only ever need genuine test infrastructure that has
nothing to do with the app itself: `fonts-dejavu-core` (a toolkit with zero
fonts available can fail to start or render only blank text - unlike a
shared library this is a resource file, not something linuxdeploy bundles,
so it's reasonable to still treat it as host-provided here), `xvfb` (the
headless display), `ffmpeg` (`-f x11grab` is preferable to ImageMagick's
`import` for the *capture* - it doesn't depend on ImageMagick having been
built with X11 delegate support, which varies), `imagemagick` (for
`identify`'s statistics on the *already-captured* PNG, which has nothing to
do with X11 delegates), `ca-certificates` (for the artifact download step),
and `x11-utils`/`gdb` if you add the window-list/backtrace diagnostics above.

Diagnose each failure from `gh run view <id> --log-failed` before adding
anything: a dynamic-linker "error while loading shared libraries: libFoo.so:
cannot open shared object file" names the exact missing library - search
the package manager for what provides it, add it to the **build** job's
apt-get line and `--library=` list, rather than reaching for the nearest
`apt-get install` in this job because it's the file you happen to be
looking at.

## Retry until it actually renders, and fail if it never does

A single blind `sleep N` before one `ffmpeg` grab has two failure modes: too
short, and it silently uploads a blank frame as if the job passed (a false
positive - worse than a red build, because nobody looks twice at a green
one); too long, and every run pays the worst case even when the app painted
in under a second. Retry instead: grab a frame every couple of seconds up to
a fixed budget (15s is a reasonable start), and stop as soon as one frame
looks like real content.

"Real content" means the frame isn't (near-)a single flat color - check with
ImageMagick's `identify -format "%[fx:standard_deviation*100]"` on the PNG
you just captured; a value near zero is still a blank/solid frame (all
black, all one background color), anything meaningfully above it is a
rendered UI. If the retry budget runs out and every frame was still flat,
**fail the job** (`exit 1` with an `::error::` annotation) instead of
uploading the last blank frame and letting `test -s` (which only checks the
file is non-empty, not that it shows anything) wave it through as a pass.
Then go find out why it never painted - a missing platform theme, a crash
right after connecting to the Xvfb display, a plugin silently failing to
load - the same way you'd diagnose any other failure in this job: read the
actual error, don't just widen the timeout further.

If nothing errors at all - no crash, no dynamic-linker message, `AppRun`
confirmed still running via `kill -0`, `xwininfo -tree -root` showing zero
windows ever created - that's a genuine hang, not a timing problem, and no
amount of extra sleep fixes it. Get a real backtrace (see the job shape
above and troubleshoot-appimage-runtime's "Diagnosing a silent hang"
section) before guessing at env vars: a headless container is commonly
missing a background service (D-Bus session bus, AT-SPI) that some toolkit
code assumes it can reach and hangs instead of failing when it can't - see
troubleshoot-appimage-runtime's "Headless container hangs" section for the
two confirmed cases and their one-line fixes. Only if the backtrace instead
shows real forward progress that simply stops cold at a library boundary
(not a service-reachability wait) does the Alpine musl/glibc-mixing
hypothesis in "Going stricter" below become the likelier explanation.

## Check what the running process actually mapped

A rendered window proves the app started, not that it started from the
bundle. Read `/proc/<pid>/maps` of the app (not of the `dbus-run-session`
or `AppRun` wrapper - `pgrep -x <binary>`) while it is up:

```sh
grep -o '/[^ ]*\.so[^ ]*' "/proc/$PID/maps" | sort -u | grep -v "$APPDIR"
```

Everything listed came from the host. Two checks on that list, in this
order of usefulness:

- **A host copy of something the binary links is the failure.** Don't
  maintain an allow-list of permitted host libraries by name - the GPU
  stack alone (Mesa, LLVM, `libdrm_*`, `libz3`, `libsensors`,
  `libpciaccess`, `libwayland-server`, the `xcb-dri*` helpers...) differs
  per distribution and version, and the list takes several red runs to
  converge. Invert it: for each host path, if a library of that name is in
  the bundle *and* in the binary's own closure (`<bundled ld.so>
  --library-path AppDir/usr/lib --list AppDir/usr/bin/<app>`), fail. That
  catches the real bug (the host's GTK/Qt/glib/librsvg won over the bundled
  one) and ignores everything the host is *supposed* to provide. Scoping
  to the closure matters: a bundled plugin's dependency that never loaded
  (ncurses, pulled in by GStreamer's `cacasink`) can share a name with
  something the host's Mesa/LLVM pulls in (`libtinfo` via `libedit`), and
  the host copy is then mapped while the bundled one is not - correct
  behaviour, not shadowing. Pruning plugins the app has no use for before
  computing the closure (terminal video sinks in a file manager) is the
  cheaper fix for that and shrinks the bundle.
- **The core stack must be inside.** For a handful of named libraries -
  `libc.so.6`, the toolkit (`libgtk-4.so.1`, `libQt6Core.so.6`), the
  dlopen'ed plugins that matter (`libpixbufloader-svg.so`, `libqsvg.so`) -
  require a mapping *under* `$APPDIR`. This is the check that catches a
  plugin silently not loading at all (the icons quietly fall back to
  placeholders rather than erroring).

Two traps in the "the container must lack X" sanity checks that usually
accompany this:

- Test infrastructure drags in app-like libraries: ImageMagick depends on
  librsvg on Debian and Fedora, `icewm` on a chunk of X11, Mesa on
  LLVM. Assert absence only for the toolkit itself, and let the maps check
  above prove the rest.
- Run one container deliberately **hostile**: install the host's own copy
  of the toolkit, a different version than you bundle (`libgtk-4-1` on
  Debian 13 against a GTK from Ubuntu 24.04; `qt6-svg-plugins` against your
  Qt). The maps check then proves the bundle wins even when the host
  offers an alternative. This is the only test that reproduces the classic
  desktop crash where a bundled toolkit loads a host plugin of another
  version.

If you run several such containers on one machine with `--network host`
(e.g. to reach a local proxy), they share the abstract-socket namespace:
give each its own Xvfb display number or the second one fails to start.

## Match the catalog's conditions once

AppImageHub (appimage.github.io) tests every submitted AppImage the same
way this skill does, with fixed conditions: a container of the oldest Ubuntu
LTS still in standard support, nothing installed beyond test tooling, X11
only, the C locale, no dark-mode preference, and **no network** (`firejail
--net=none`); the main window must be visible within 30 seconds and show
content, not an error dialog. Add one matrix entry with exactly those
conditions (`ubuntu:<oldest LTS>`, `docker run --network none`, `LANG=C`) so
that a catalog submission passes first time; the publish-to-appimagehub skill
lists the static checks (`desktop-file-validate`, `appstreamcli`, `.DirIcon`,
file naming) to run next to it.

## Publish the screenshot in the Release

A screenshot sitting only in a CI artifact is easy to miss - artifacts
expire, and nobody browses a build log looking for proof the app runs.
Once the verification job reliably produces a real screenshot, attach it to
the same release the AppImage itself is published in, and embed it in the
release notes so it's visible without opening anything:

```yaml
upload:
  needs: [build, screenshot-debian, screenshot-alpine]
  # Debian is the real gate; Alpine is the stricter, optional one - a release
  # still ships without its screenshot if that job fails.
  if: |
    always() && github.event_name == 'push' &&
    needs.build.result == 'success' && needs.screenshot-debian.result == 'success'
  ...
  steps:
    - uses: actions/download-artifact@v4   # no `name:` - pulls every artifact into <name>/ dirs

    - name: Create release and upload
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      run: |
        FILES="screenshot/screenshot.png"
        [ -f screenshot-alpine/screenshot-alpine.png ] && FILES="$FILES screenshot-alpine/screenshot-alpine.png"
        ./pyuploadtool-x86_64.AppImage **/YourApp*.AppImage* $FILES

    - name: Add the screenshot(s) to the release notes
      env:
        GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      run: |
        case "$GITHUB_REF" in
          refs/tags/*) TAG="${GITHUB_REF#refs/tags/}" ;;
          *) TAG="continuous" ;;
        esac
        gh release view "$TAG" --repo "${{ github.repository }}" --json body --jq .body > notes.md
        {
          echo ""; echo ""
          echo "## Running on a stock minimal Debian"; echo ""
          echo "![Screenshot](https://github.com/${{ github.repository }}/releases/download/$TAG/screenshot.png)"
        } >> notes.md
        if [ -f screenshot-alpine/screenshot-alpine.png ]; then
          {
            echo ""; echo "## Running on a stock minimal Alpine"; echo ""
            echo "![Screenshot](https://github.com/${{ github.repository }}/releases/download/$TAG/screenshot-alpine.png)"
          } >> notes.md
        fi
        gh release edit "$TAG" --repo "${{ github.repository }}" --notes-file notes.md
```

Give the two jobs' screenshots different filenames (`screenshot.png`,
`screenshot-alpine.png`) - GitHub release assets share one namespace, so two
artifacts both called `screenshot.png` collide. Derive the tag from
`$GITHUB_REF` (see build-appimage-ci's continuous-release section) rather
than asking the API for the most-recently-created release - a concurrent run
or another release stream can make that answer point at the wrong release,
where deriving it from the event that's actually running cannot.

## A second, stricter target: Alpine (musl) - achievable once truly self-contained

Add this only after the Debian job is solid and the AppImage is genuinely
self-standing - the first attempt at Alpine, made before the bundle was
complete, hung indefinitely because the payload's glibc libraries ended up
loading Alpine's own musl-compiled system libraries into the same process.
That was real, but it wasn't a musl/glibc dead end - it was a symptom of an
incomplete bundle. Once nothing the app needs is left for Alpine to provide,
the same recipe as Debian works, plus what Alpine still needs just to start
a glibc-linked binary at all:

1. **`libc6-compat` first, always.** `actions/checkout`, `actions/download-
   artifact`, and every other JS-based action run through a Node binary the
   Actions runner injects from the *host*, which is glibc-linked. Inside a
   plain `container: image: alpine:...` job, that injected Node can't even
   start - every `uses:` step after the container starts fails immediately
   without `apk add --no-cache libc6-compat` as the very first step. This
   is a separate, smaller package from `gcompat` - install it regardless of
   which compat layer you end up needing for the app itself.
2. **A compat layer for the payload's own glibc.** Try `apk add gcompat`
   (Alpine's official shim) first - it's a partial glibc reimplementation
   and works for a surprising number of apps. If the dynamic linker reports
   missing `GLIBC_2.3x`-era symbols (`__isoc23_strtol`, `renameat2`,
   `close_range`, `pthread_cond_clockwait`, etc.), `gcompat` doesn't cover
   them and you need a real glibc build instead - the long-standing
   `sgerrand/alpine-pkg-glibc` repo has historically lagged behind current
   glibc releases (stuck around 2.35 at last check, which predates
   `GLIBC_2.38`), so check its version against what `objdump -T` (see
   troubleshoot-appimage-runtime) says you actually need. A more current
   fork of that same packaging (e.g. `dalet-oss/alpine-glibc`) may track
   newer releases - this is still a third-party apk repo and signing key,
   not an official Alpine package, so say so explicitly wherever you add it:
   ```sh
   GLIBC_VER=2.44
   wget -q -O /etc/apk/keys/alpine-packaging-rsa.pub \
     https://raw.githubusercontent.com/dalet-oss/alpine-glibc/master/alpine-packaging-rsa.pub
   wget -q "https://github.com/dalet-oss/alpine-glibc/releases/download/${GLIBC_VER}-amd64-r1/glibc-${GLIBC_VER}.apk"
   wget -q "https://github.com/dalet-oss/alpine-glibc/releases/download/${GLIBC_VER}-amd64-r1/glibc-bin-${GLIBC_VER}.apk"
   apk add --no-cache --force-overwrite "glibc-${GLIBC_VER}.apk" "glibc-bin-${GLIBC_VER}.apk"
   ```
3. **The same launch sequence as the Debian job** - Xvfb, a window manager
   (`apk add icewm`), `QT_ACCESSIBILITY=0`, `DBUS_SESSION_BUS_ADDRESS=disabled:`,
   the window-existence check before trusting pixel stddev, the gdb backtrace
   on a real hang. None of that is glibc/musl-specific; it's the same gaps a
   minimal container has regardless of libc.
4. **Expect the GLVND completeness gap from above to show up here first.**
   If a system copy of `libGLX.so.0`/`libGLdispatch.so.0`/`libOpenGL.so.0` was
   quietly covering for an incomplete bundle on Debian, Alpine's container
   won't have one, and `AppRun` will fail with a clean "error while loading
   shared libraries" for each missing piece in turn. This is exactly the
   value of running both targets: fix each one in the **build** job's
   bundling list (never by installing it into either test container), and
   both jobs end up more honestly self-contained than either would alone.

If Alpine still won't get past a missing-library error even after bundling
everything `ldconfig -p`/`objdump -T` can find, or the failure is a genuine
silent hang with no window ever created and no library error at all, treat
that as a real, open problem rather than iterating blindly - see
troubleshoot-appimage-runtime's diagnostic sections, and consider accepting
the Debian job as your only release gate and keeping Alpine as an informal,
`continue-on-error:` check rather than a hard requirement.
