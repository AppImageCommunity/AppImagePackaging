# Keeping AppRun's environment out of the programs the app starts

The two layers of defence (relative RPATH instead of LD_LIBRARY_PATH; a preloaded exec/posix_spawn hook that restores the original environment), the app-side hook needed when the app sandboxes its own helpers with bubblewrap, and generating the pixbuf loader cache at start. Part of the make-standalone-appdir skill.

## Keep AppRun's environment out of the programs the app starts

Everything AppRun exports for the bundled toolkit - `LD_LIBRARY_PATH`,
`GDK_PIXBUF_MODULE_FILE`, `GIO_MODULE_DIR`, `GSETTINGS_BACKEND`,
`GTK_EXE_PREFIX`, `QT_PLUGIN_PATH`, `GCONV_PATH`, ... - is inherited by every
process the app spawns. Most of those are **host** programs: the browser an
app opens for a sign-in, `xdg-open`, `/bin/sh` behind a `system()`, `ps`,
`notify-send`. A host browser that picks up your `GDK_PIXBUF_MODULE_FILE`
dlopens *your* `libpixbufloader-svg.so`, which drags in *your* librsvg and
glib, linked against the bundled glibc - and a host program given a bundled
`LD_LIBRARY_PATH` with a newer glibc on it segfaults before `main`. These
crashes show up as "the app works but clicking a link kills Firefox", far
from anything that looks like a bundling problem.

Two layers of defence, in order:

1. **Don't export `LD_LIBRARY_PATH` at all.** Give every bundled ELF a
   relative `DT_RPATH` instead (`patchelf --force-rpath --set-rpath
   '$ORIGIN/../lib'`, computed per file relative to `usr/lib`). RPATH, unlike
   RUNPATH, is searched *before* `LD_LIBRARY_PATH` and applies to the
   dependencies of what it loads, so a session that exports its own
   `LD_LIBRARY_PATH` (Nix, GNUstep, vendor-driver setups) cannot redirect the
   bundle either. Skip `ld-linux*` itself. (Pixbuf loaders then need absolute
   paths in `loaders.cache` - see below.)
2. **Restore the original environment on `exec`.** The remaining variables
   have no RPATH equivalent, and a shell script cannot un-export anything for
   its children. A tiny preloaded shim can: AppRun records each variable it
   touches (`APPIMAGE_ORIG_<NAME>` holds the old value, absent if it was
   unset; `APPIMAGE_RESTORE_VARS` lists the names) and preloads
   `usr/lib/exec.so`, which rewrites `envp` in `execve`, `execvpe`, `execv`,
   `execvp`, `posix_spawn` and `posix_spawnp`: restore each listed variable,
   drop the bookkeeping and `LD_PRELOAD` itself. About 100 lines of C,
   compiled in the AppDir build with `gcc -shared -fPIC -o usr/lib/exec.so
   exec.c -ldl`, no dependencies beyond libc.

   ```sh
   # AppRun
   APPIMAGE_RESTORE_VARS=""
   remember() { eval "if [ -n \"\${$1+x}\" ]; then export APPIMAGE_ORIG_$1=\"\$$1\"; fi"
                APPIMAGE_RESTORE_VARS="$APPIMAGE_RESTORE_VARS${APPIMAGE_RESTORE_VARS:+ }$1"; }
   setenv()   { remember "$1" && export "$1=$2"; }
   unsetenv() { remember "$1" && unset "$1"; }
   setenv LD_PRELOAD "$HERE/usr/lib/exec.so${LD_PRELOAD:+:$LD_PRELOAD}"
   setenv GDK_PIXBUF_MODULE_FILE "..."
   unsetenv GTK_MODULES
   export APPIMAGE_RESTORE_VARS
   exec "$HERE/usr/bin/app" "$@"
   ```

   Hook `posix_spawn` as well as the `exec*` family: Rust's
   `std::process::Command` and newer glibc `system()`/`popen()` go through
   it, not `fork`+`execve`. `execl*` cannot be hooked (glibc calls `execve`
   internally, not through the PLT), which has not mattered in practice.

   Test it where the build runs, not only in the app: run a host program
   through the bundled loader with the shim preloaded and the bookkeeping
   variables set, spawn `/usr/bin/env` from it, and assert the leaked
   variables are gone and the original value is back:
   ```sh
   APPIMAGE_RESTORE_VARS='XDG_DATA_DIRS LD_PRELOAD' APPIMAGE_ORIG_XDG_DATA_DIRS=/original \
   XDG_DATA_DIRS=/leaked LD_PRELOAD="$(pwd)/AppDir/usr/lib/exec.so" \
   AppDir/lib64/ld-linux-x86-64.so.2 --library-path AppDir/usr/lib /usr/bin/python3 -c \
     'import subprocess; print(subprocess.run(["/usr/bin/env"], capture_output=True, text=True).stdout)' \
     | grep -qx 'XDG_DATA_DIRS=/original'
   ```
   (The shim is linked against the bundled glibc, so this only works on a
   host with the same or newer glibc - the build runner qualifies.)

Once this exists, the only things a child inherits from the bundle are
`APPDIR`/`APPIMAGE` from the runtime, which are harmless.

### An app that sandboxes its own helpers cannot see the bundle

Some applications run their own binary again inside bubblewrap for risky
work (thumbnailers, document parsers, archive extraction), with
`--clearenv`, a fresh `--tmpfs /tmp` and only the host's `/usr`, `/lib` and
`/lib64` bound read-only. Inside that sandbox nothing of the AppImage exists:
the mount point under `/tmp` is hidden by the tmpfs, the fixed-path loader
directory likewise, and the helper is usually bound at a neutral path such as
`/app/<name>`, so even a relative RPATH (`$ORIGIN/../lib`) resolves to the
host's `/lib`. The symptom is an app that starts and browses fine but shows
no previews, because every helper dies before `main` ("fail closed").

No packaging trick fixes this: the only user-writable, sandbox-visible
location would have to be one the app binds, and the bind list is in the
app. It needs a small hook in the application (a reasonable upstream ask;
about 40 lines with tests): an environment variable naming extra read-only
roots (`<APP>_SANDBOX_EXTRA_ROOTS=$APPDIR:$LOADER_DIR`, colon-separated
absolute paths, bound after the tmpfs so they are not hidden by it), the
helper run from its own path instead of the neutral one when it lives under
such a root (so its interpreter and `$ORIGIN` resolve), and a fixed
allow-list of relocation variables passed through while such roots are
bound (`GDK_PIXBUF_MODULE_FILE`, `GCONV_PATH`, `GST_PLUGIN_SYSTEM_PATH_1_0`,
`GST_PLUGIN_SCANNER`, `FONTCONFIG_FILE`), since the sandbox clears the
environment. Keep generated runtime files (the pixbuf cache) inside the
loader directory so one bind covers them. The exec hook above does not
interfere: bubblewrap is a host program and gets the host's environment,
and the hook is not preloaded into the sandboxed helper because the
sandbox clears `LD_PRELOAD` anyway.

Test the sandboxed path directly in the container test rather than through
the GUI: build the same bubblewrap command the app builds (keep it in step
with the app's source), add the extra roots, and run the helper on a small
PNG and a hand-written minimal PDF; assert the outputs exist. The container
needs `--privileged` for bubblewrap's user namespaces, and a host whose
bubblewrap predates an option the app passes (`--size` on Ubuntu 22.04's
0.6) cannot run the sandbox however the app is installed: skip the check
there with a notice and document it as a host requirement next to the GPU
drivers.

### Pixbuf loaders without `LD_LIBRARY_PATH`: generate the cache at start

Golden Rule 4's "relative paths in `loaders.cache`" works *because* the
loaders directory is on `LD_LIBRARY_PATH`: gdk-pixbuf hands the bare module
name to `dlopen`, which finds it through the search path. gdk-pixbuf does
**not** resolve a relative cache entry against `GDK_PIXBUF_MODULEDIR` (that
code path is Windows-only), so without `LD_LIBRARY_PATH` every SVG icon
fails with `Unable to load image-loading module: libpixbufloader-svg.so:
cannot open shared object file`. The mount point is unknown until the
AppImage runs, so AppRun fills in a template:

```sh
# build time: gdk-pixbuf-query-loaders AppDir/usr/lib/gdk-pixbuf-2.0/2.10.0/loaders/*.so |
#             sed -E 's|^"/.*/(libpixbufloader-[^/"]+\.so)"|"@LOADERS@/\1"|' > .../loaders.cache.in
# AppRun:
sed "s|@LOADERS@|$HERE/usr/lib/gdk-pixbuf-2.0/2.10.0/loaders|" \
    "$HERE/usr/lib/gdk-pixbuf-2.0/2.10.0/loaders.cache.in" >"$RUNTIME_DIR/loaders.cache"
setenv GDK_PIXBUF_MODULE_FILE "$RUNTIME_DIR/loaders.cache"
```

where `$RUNTIME_DIR` is somewhere writable and private to the user
(`$XDG_RUNTIME_DIR`, or the directory you already create for a bundled
dynamic linker at a fixed path). Only the loaders the toolkit really needs
belong in the cache (SVG via librsvg; PNG and JPEG are usually compiled into
`libgdk_pixbuf` on current distributions - check `loaders/` on the build
host).

Related: `GTK_EXE_PREFIX="$HERE/usr"` keeps GTK 4 from loading the **host's**
media backends and print backends from its compiled-in `libdir/gtk-4.0/`
(it will, silently, if the host happens to have GTK at the same path - seen
as `libmedia-gstreamer.so` from `/usr/lib` inside a bundled-GTK process),
and libadwaita without a settings portal reads
`org.gnome.desktop.interface`, which is in `gsettings-desktop-schemas`, not
GTK - a runner can lack it even when the build host had it.
