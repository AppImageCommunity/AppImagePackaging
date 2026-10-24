# Verification checklist

Commands to run against a finished AppDir or an extracted AppImage before shipping it. Part of the make-standalone-appdir skill.

## Verification Checklist

```sh
# 1. Extract and check for patchelf corruption
./my-app.AppImage --appimage-extract
md5sum /usr/lib/libgtk-3.so.0 squashfs-root/usr/lib/libgtk-3.so.0
# Must match

# 2. Check loaders.cache has relative paths
head -10 squashfs-root/usr/lib/gdk-pixbuf-2.0/2.10.0/loaders.cache
# Should show "libpixbufloader-foo.so" not "/usr/.../libpixbufloader-foo.so"

# 3. Check dynamic linker is bundled
ls squashfs-root/lib/ld-*

# 4. Check index.theme exists and has actions
grep -A2 "scalable/actions" squashfs-root/usr/share/icons/hicolor/index.theme

# 5. Check gschemas.compiled exists
ls squashfs-root/usr/share/glib-2.0/schemas/gschemas.compiled

# 6. Check standard icons are in hicolor
ls squashfs-root/usr/share/icons/hicolor/scalable/actions/

# 7. Run from a different directory
cp my-app.AppImage /tmp/
cd /tmp && ./my-app.AppImage

# 8. If a GTK module was bundled: module present, RPATH reaches usr/lib,
#    parser symlink not dangling, schema compiled in
ls squashfs-root/usr/lib/gtk-3.0/modules/
patchelf --print-rpath squashfs-root/usr/lib/gtk-3.0/modules/libappmenu-gtk-module.so
# Expect: $ORIGIN/../..
ls -l squashfs-root/usr/lib/ | grep appmenu
strings squashfs-root/usr/share/glib-2.0/schemas/gschemas.compiled | grep -c org.appmenu.gtk-module
# Expect: non-zero

# 9. Run it and confirm stderr is EMPTY (no "Failed to load module")
./my-app.AppImage > run.log 2>&1 &
```
