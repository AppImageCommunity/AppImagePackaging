# Bundling GTK modules (global menu / appmenu)

Step 11 of the assembly in SKILL.md, in full: building appmenu-gtk-module against the bundled GTK, its GSettings schema, RPATH, and GTK_PATH versus GTK_MODULES. Part of the make-standalone-appdir skill.

### 11. Bundle GTK modules (global menu / appmenu)

Skip this unless the app should hand its menu bar to a global menu bar
(Gershwin, Unity, and some KDE/XFCE setups). Those sessions export
`GTK_MODULES=appmenu-gtk-module`, and GTK then tries to `dlopen` that module
inside your app.

**The host's copy can never serve.** Your bundle ships a musl-linked GTK stack;
the host module is glibc-linked. The load always fails, the app keeps its menu
bar trapped in the window, and the only clue is one line on stderr:

```
Gtk-Message: Failed to load module "appmenu-gtk-module"
```

Alpine does not package `appmenu-gtk-module`, so build it in the container.
Pin the tarball by sha256 - the GitLab upload URL is opaque and carries no
version:

```sh
VERSION="25.04"
SHA256="2ab8cc56c4a14cb9e0dd24351392c3c8d965a65c4e02c155c1a2e8cba5b21b86"
URL="https://gitlab.com/-/project/6865053/uploads/4f517338d3c65a0ea6f49faf36a4f3e6/appmenu-gtk-module-${VERSION}.tar.xz"

wget -q -O src.tar.xz "$URL"
echo "$SHA256  src.tar.xz" | sha256sum -c -
tar -xJf src.tar.xz

# --prefix=/usr, NOT a DESTDIR: the GSettings schema then lands in
# /usr/share/glib-2.0/schemas, where step 8 already harvests from.
# Run this BEFORE the AppDir assembly script.
meson setup "appmenu-gtk-module-$VERSION/build" "appmenu-gtk-module-$VERSION" \
    --prefix=/usr --buildtype=release -Dgtk=3 -Dtests=false -Dgtk_doc=false
meson compile -C "appmenu-gtk-module-$VERSION/build"
meson install -C "appmenu-gtk-module-$VERSION/build"
```

Finding the URL again when the version moves: the release JSON at
`https://gitlab.com/api/v4/projects/vala-panel-project%2Fvala-panel-appmenu/releases`
lists `/uploads/<hash>/appmenu-gtk-module-<ver>.tar.xz`. That path 404s to HTML
as written - it only downloads under `https://gitlab.com/-/project/6865053/uploads/<hash>/...`.

Then copy it into the AppDir:

```sh
mkdir -p AppDir/usr/lib/gtk-3.0/modules
cp -a /usr/lib/gtk-3.0/modules/libappmenu-gtk-module.so AppDir/usr/lib/gtk-3.0/modules/
cp -a /usr/lib/libappmenu-gtk3-parser.so.* AppDir/usr/lib/

# The module sits one level deeper than usr/lib, so it needs its own RPATH -
# a loop over AppDir/usr/lib/lib*.so.* never reaches it.
patchelf --set-rpath '$ORIGIN/../..' \
    AppDir/usr/lib/gtk-3.0/modules/libappmenu-gtk-module.so

# ldd both and bundle whatever is still missing (same pattern as step 7)
```

Three things that are not obvious:

- **The GSettings schema is mandatory.** `blacklist.c` and `support.c` call
  `g_settings_new("org.appmenu.gtk-module")`, and GLib *aborts the process*
  when a schema is missing. Installing with `--prefix=/usr` before step 8 gets
  `org.appmenu.gtk-module.gschema.xml` into the bundled `gschemas.compiled`.
- **The parser library is not `.so.0`.** meson's `soversion: '0'` plus the
  project version produces `libappmenu-gtk3-parser.so.25.04` with a `.so.0`
  symlink pointing at it. Glob `libappmenu-gtk3-parser.so.*` so `cp -a` takes
  both, or you ship a dangling symlink.
- **`appmenu-gtk-module` does not use libdbusmenu.** It exports a GMenuModel
  over `org.gtk.Menus`; `ldd` shows no dbusmenu at all. Bundling
  `libdbusmenu-gtk3` does nothing for the global menu - it is a separate
  protocol used by tray icons.

AppRun needs `GTK_PATH` (see step 12). GTK3 searches each `GTK_PATH` entry as
`<path>/<binary-version>/<host>/modules`, `<path>/<binary-version>/modules`,
`<path>/<host>/modules` and `<path>/modules`, so `GTK_PATH=$PREFIX/lib/gtk-3.0`
resolves `$PREFIX/lib/gtk-3.0/modules/libappmenu-gtk-module.so`.
