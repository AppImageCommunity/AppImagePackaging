# Common pitfalls and how to avoid them

Every failure mode met while assembling AppDirs, with symptom, cause and fix. Part of the make-standalone-appdir skill.

## Common Pitfalls and How to Avoid Them

### Pitfall 1: patchelf corrupts .gresource sections

**Symptom:** App crashes with:
```
Gtk:ERROR: gtkiconhelper.c:495: ensure_surface_for_gicon: assertion failed:
Failed to load /org/gtk/libgtk/icons/16x16/actions/document-new.png
```

**Cause:** `appimagetool -s deploy` runs patchelf on every ELF to set
RUNPATH. patchelf appends padding that shifts sections. The `.gresource`
section contains absolute offsets to embedded data.

**Fix:** Never use `-s deploy`. Copy libraries manually via ldd.
Use patchelf only on the main binary (not on libraries with GResource).

### Pitfall 2: loaders not found at runtime

**Symptom:** `Bildlader-Modul konnte nicht geladen werden`

**Cause:** The loaders.cache has relative filenames, but dlopen searches
`LD_LIBRARY_PATH`. If the loaders dir isn't on that path, dlopen fails.

**Fix:** Add the loaders directory to `LD_LIBRARY_PATH` in AppRun.

### Pitfall 3: loaders.cache has absolute paths

**Symptom:** Same as Pitfall 2, but error shows absolute path.

**Cause:** Wrong sed command:
```sh
# WRONG - produces "/libfoo.so" (leading slash)
sed 's|"[^"]*loaders/|"/|g'

# CORRECT - produces "libfoo.so" (no leading slash)
sed 's|".*/loaders/|"|g'
```

### Pitfall 4: Adwaita SVG-only theme crashes

**Symptom:** `Failed to load image-missing.svg`

**Cause:** Bundled Adwaita is SVG-only. Without librsvg, SVG loader fails.

**Fix:** Bundle librsvg, or don't bundle Adwaita at all.

### Pitfall 5: Icons not showing in UI

**Symptom:** Tool icons (pencil, eraser) or standard icons (list-add,
edit-copy) don't appear.

**Cause:** Multiple possible causes:
- hicolor `index.theme` missing `scalable/actions` context
- Standard action icons not copied from Adwaita to hicolor
- GResource not registered (constructor not firing)

**Fix:**
1. Add `scalable/actions` to hicolor's `index.theme` Directories
2. Copy standard icons from Adwaita to `hicolor/scalable/actions/`
3. Verify GResource constructor exists: `nm binary | grep resource_constructor`

### Pitfall 6: pixbuf loader version mismatch

**Symptom:** `WARNING: No pixbuf loaders directory found`

**Cause:** Hardcoded version (e.g., `2.10.0`) doesn't match the actual
version on the build system. `pkg-config --variable=binary_version` may
return empty in Docker.

**Fix:** Use glob detection instead of pkg-config:
```sh
for dir in /usr/lib/gdk-pixbuf-2.0/*/loaders; do
    [ -d "$dir" ] && PIXBUF_LOADERS_DIR="$dir" && break
done
```

### Pitfall 7: Dynamic linker not bundled

**Symptom:** AppImage fails with `No such file or directory` when
trying to execute, or `cannot find ld-linux-*.so`

**Cause:** The bundled binary uses a relative interpreter path
(e.g., `../lib/ld-musl-x86_64.so.1`) but the dynamic linker isn't
in the AppDir.

**Fix:** Bundle the dynamic linker in `AppDir/lib/` and ensure AppRun
cd's to AppDir root before exec.

### Pitfall 8: File permissions in Docker

**Symptom:** Can't execute AppRun or binary after extraction.

**Cause:** Docker creates files as root with restrictive permissions.

**Fix:** Run `chmod -R a+rwx AppDir` inside the Docker container
before it exits.

### Pitfall 9: appimagetool needs desktop file at root

**Symptom:** `No top-level desktop file found in AppDir, aborting`

**Fix:** Copy the desktop file to both locations:
```sh
cp my-app.desktop AppDir/usr/share/applications/
cp my-app.desktop AppDir/
```

### Pitfall 10: Binary RUNPATH overriding LD_LIBRARY_PATH

**Symptom:** App loads system libs instead of bundled ones.

**Fix:** Check for RPATH:
```sh
readelf -d AppDir/usr/bin/my-app | grep -i 'rpath\|runpath'
```
If present, use `patchelf --set-rpath '$ORIGIN/../lib'` on just the binary.

### Pitfall 11: global menu silently does nothing

**Symptom:** On a global-menu desktop the app keeps its menu bar inside the
window like a non-native app. The only evidence is one stderr line:

```
Gtk-Message: Failed to load module "appmenu-gtk-module"
```

**Cause:** The session exports `GTK_MODULES=appmenu-gtk-module`, but the
AppDir has no `usr/lib/gtk-3.0/modules/`, and the host's module is
glibc-linked so this musl-linked GTK cannot load it. GTK treats a module that
fails to load as a warning, not an error, so the app starts normally.

**Fix:** Build and bundle the module (step 11) and set `GTK_PATH` (step 12).

**Do NOT "fix" it by exporting `GTK_MODULES` from AppRun.** That forces the
module on everywhere, and on a desktop with no global menu to receive the
export the menu bar is stripped from the window and goes nowhere - the app
ends up with no menus at all.

**Verify by X property, not by eye** - a global menu bar shows the *focused*
window's menu, so a screenshot proves nothing if something else has focus:

```sh
xprop -id <window-id> _GTK_MENUBAR_OBJECT_PATH
# _GTK_MENUBAR_OBJECT_PATH(UTF8_STRING) = "/org/appmenu/gtk/window/0"
```

Empty stderr plus that property is the real pass condition. Note the app may
own several X windows; check them all, since only the real toplevel carries it.

### Pitfall 12: libraries that load the host's drivers (libva and friends)

Some libraries are thin loaders for drivers that belong to the host's GPU
stack: **libva** (VA-API, loads `*_drv_video.so`), libvdpau, the Vulkan
loader, libGL/libEGL (GLVND). Neither extreme works for them:

- **Excluding them** (e.g. linuxdeploy `--exclude-library 'libva*'`) makes the
  AppImage fail to *start* wherever the library isn't installed: minimal
  installs, containers, CI test systems:
  `error while loading shared libraries: libva.so.2: cannot open shared object file`.
  FFmpeg built with `--enable-vaapi` links libva as a hard `NEEDED`
  dependency, so "most distros have it" isn't enough.
- **Bundling an old one** breaks hardware support on newer systems. libva
  calls the driver's `__vaDriverInit_<major>_<minor>`, trying its own minor
  version and then older ones only
  ([va.c](https://github.com/intel/libva/blob/master/va/va.c)): a newer libva
  accepts older drivers, but an old bundled libva rejects drivers built for a
  newer one. The bundled copy also looks for drivers in its compiled-in
  directory (`VA_DRIVERS_PATH`, e.g. `/usr/lib/x86_64-linux-gnu/dri`), which
  differs between distributions (`/usr/lib64/dri` on Fedora) unless
  `LIBVA_DRIVERS_PATH` is set.

**Pattern: bundle it as a fallback, used only when the host has none.** Put
the loader libraries in a directory that is *not* in the RPATH or
`LD_LIBRARY_PATH`, and let AppRun add it only when the host lacks the library:

```sh
# AppDir/usr/lib/libva-fallback/{libva.so.2,libva-drm.so.2,libva-x11.so.2}
HERE="$(dirname "$(readlink -f "$0")")"
if ! /sbin/ldconfig -p 2>/dev/null | grep -q 'libva\.so\.2 ' ; then
  export LD_LIBRARY_PATH="$HERE/usr/lib/libva-fallback${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
fi
```

- **Host has libva:** its own libva and drivers are used, so hardware
  decoding keeps working.
- **Host has no libva:** the bundled copy loads, finds no driver, and FFmpeg
  falls back to software decoding. The app starts instead of failing.
- On distributions without a usable `ldconfig -p` (e.g. NixOS), the fallback
  is used, which is still safe.

Never bundle the drivers themselves (`*_drv_video.so`, Mesa DRI drivers, the
Nvidia libraries). They must match the host's kernel and GPU. See also the
GLVND section in troubleshoot-appimage-runtime.

Real case: cbird (Qt + FFmpeg) excluded libva "or else we are missing
symbols in newer os", and then failed to start on systems without libva:
https://github.com/AppImage/appimage.github.io/pull/5511

### Pitfall 13: bundling the host's display/EGL stack (blank window on Wayland / Mesa 25+)

Same principle as Pitfall 12, one layer up: the **windowing and EGL client
libraries** — `libwayland-client.so.0`, `libwayland-egl.so.1`,
`libwayland-cursor.so.0`, `libxkbcommon.so.0`, `libxcb*.so` (and `libEGL` /
`libGLX` / `libgbm`, see the GLVND section in troubleshoot-appimage-runtime) —
belong to the **host**, not the app. They have to match the host's compositor
and its Mesa/EGL/GBM driver. Bundle them and they shadow the host's copies via
`LD_LIBRARY_PATH`; an older bundled `libwayland-egl`/`libxkbcommon` against a
host running Mesa 25+ then cannot create an EGL display.

**Symptom:** the app starts but shows a **blank/white window** (no crash, no
missing-library error). GTK/WebKitGTK apps log
`Could not create default EGL display: EGL_BAD_PARAMETER` or
`AcceleratedSurfaceDMABuf was unable to construct a complete framebuffer`. It
appears on newer hosts (Mesa 25+, and pure-Wayland sessions) but not on the
older box the AppImage was built on. A second failure mode is an AppRun that
forces `GDK_BACKEND=x11`: on a Wayland host without XWayland the GTK hook then
crashes.

**Do not "fix" it by forcing software rendering.** The common crutch —
`WEBKIT_DISABLE_COMPOSITING_MODE=1` and `WEBKIT_DISABLE_DMABUF_RENDERER=1` —
pushes the whole UI onto the CPU. It hides the blank window but throws away GPU
acceleration, so it is a fallback (and the right thing for a *headless CI test*,
which has no GPU), not a fix for shipped AppImages.

**The fix: don't bundle the display/EGL client stack, and don't pin the
backend.** Exclude `libwayland-*`, `libxkbcommon*`, `libxcb*` and the
`libEGL`/`libGLX`/`libgbm` GLVND loaders (they are host-coupled, exactly like
the drivers in Pitfall 12, and are on the AppImage excludelist for this reason)
so the host's own copies are used. Leave `GDK_BACKEND` unset so the app follows
the host session (Wayland or X11). The app then renders with GPU acceleration on
old and new hosts alike.

Real case: Tauri shipped precisely this fix in
[tauri-apps/tauri#16062](https://github.com/tauri-apps/tauri/pull/16062)
("fix(bundler): update linuxdeploy and linuxdeploy-plugin-gtk", released in
`@tauri-apps/cli` 2.12.0 / `tauri-bundler` 2.10.0): it updated
linuxdeploy-plugin-gtk to stop over-bundling libwayland/libxkbcommon/libxcb and
removed the forced `GDK_BACKEND=x11` (it also rewrites the cached plugin script
so stale copies with the old setting don't linger). This replaced the earlier
`WEBKIT_DISABLE_*` software-rendering workaround and let Tauri re-enable AppImage
output with full GPU support. See esoltys/luminous#1301 and the upstream reports
tauri-apps/tauri #11988, #15665 and #15976.
