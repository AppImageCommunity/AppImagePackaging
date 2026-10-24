---
name: make-standalone-appdir
description: Use when building a self-contained AppDir for AppImage or portable deployment - covers library bundling via ldd, loaders.cache patching, GResource preservation, icon theme assembly, GLib schema compilation, GTK modules for global-menu desktops (appmenu-gtk-module), AppRun env setup, dynamic linker bundling, host-driver loaders such as libva (bundle as a fallback, never the drivers), keeping AppRun's environment out of the host programs the app launches (an exec/posix_spawn preload hook; relative RPATH instead of LD_LIBRARY_PATH), Alpine/musl builds, multi-arch CI, and the pitfalls that break each of these. Applies to GTK3/GTK4, Qt, GNUstep, or any Linux GUI app that needs to run from anywhere without system dependencies.
---

# Make a Standalone AppDir

A self-contained AppDir is a directory that bundles an application and ALL its
runtime dependencies so it can run on any compatible Linux system (or be packed
into an AppImage). This skill covers the strategies, the exact commands, and
the pitfalls that waste hours when you get them wrong.

See also: [build-appimage-ci](../build-appimage-ci/SKILL.md) for how this
AppDir gets built and packaged inside a GitHub Actions workflow - including
its "run the watch-fix-push loop in small subagents" section, which applies
here too whenever you're chasing bundling errors (a missing `.so`, a broken
loaders.cache, a GResource corrupted by patchelf) across several rebuild
attempts; [troubleshoot-appimage-runtime](../troubleshoot-appimage-runtime/SKILL.md)
for diagnosing exactly *why* a bundled AppDir fails at runtime (missing
libraries, failed dlopen, hardcoded paths that need patching); and
[verify-appimage-portability](../verify-appimage-portability/SKILL.md)
for proving the resulting AppImage actually runs somewhere other than the
build machine.

## The Golden Rules

1. **Never use `appimagetool -s deploy`** if your app uses GResource (GTK3/GTK4).
   patchelf corrupts ELF `.gresource` sections, breaking all built-in icon loading.
2. **Always put the loaders dir on `LD_LIBRARY_PATH`**, not just `usr/lib`.
   GdkPixbuf's dlopen uses the LD path, not GDK_PIXBUF_MODULEDIR, to find .so files.
3. **Never bundle Adwaita SVG-only icon themes** without librsvg in the bundle.
   GTK falls back to `image-missing.svg` which crashes without an SVG decoder.
4. **The loaders.cache must have relative paths**, not absolute.
   Absolute paths break when the AppDir is mounted at a different location.
5. **Include `/usr/share` as a fallback in `XDG_DATA_DIRS`**.
   Without it, GTK cannot find system icon themes and falls through to
   GResource loading which may also fail.
6. **Alpine/musl produces smaller bundles** than glibc. The dynamic linker
   (`ld-musl-*.so.1`) is ~100KB vs glibc's ~200KB, and musl libs are
   generally smaller. Use Alpine for AppImage builds when possible.
7. **Bundle the dynamic linker** for full self-containment. Without it, the
   AppImage won't run on systems with a different libc or no compatible
   dynamic linker at `/lib64/ld-linux-x86-64.so.2`.
8. **Use relative ELF interpreter paths** with `cd`-based resolution in AppRun.
   The kernel resolves relative interpreter paths from cwd, so AppRun's
   `cd "$HERE"` before `exec` makes this work.
9. **A host GTK module can never be loaded into a musl bundle.** If the
   session exports `GTK_MODULES=...`, that module must be built and bundled
   inside the AppDir, or GTK just prints "Failed to load module" and moves on.
10. **Set `GTK_PATH`, never `GTK_MODULES`.** Bundling a module makes it
   *available*; forcing it on makes it *mandatory*, which breaks the app on
   every desktop that has nothing to receive the exported menu.

## Directory Layout

```
AppDir/
  AppRun                          # Entry point (shell script)
  org.example.App.desktop         # Desktop file at ROOT (appimagetool requires this)
  usr/
    bin/
      my-app                      # The main binary
    lib/
      ld-musl-x86_64.so.1        # Bundled dynamic linker (musl)
      libgtk-3.so.0               # Bundled shared libraries
      libglib-2.0.so.0
      libgdk_pixbuf-2.0.so.0
      ...                         # All transitive deps
      gtk-3.0/
        modules/
          libappmenu-gtk-module.so  # Optional: global-menu export
      gdk-pixbuf-2.0/
        2.10.0/                   # Version may vary; detect dynamically
          loaders/                # Pixbuf loader .so files
            libpixbufloader-png.so
            libpixbufloader_svg.so
            ...
          loaders.cache           # Rewritten with relative paths
    share/
      applications/
        org.example.App.desktop
      icons/
        hicolor/
          index.theme             # REQUIRED - must include actions context
          scalable/
            apps/                 # App icons
            actions/              # Standard GTK icons (list-add, edit-copy, etc.)
      Adwaita/                    # Optional: stock GTK icons
        index.theme
        scalable/
          actions/                # SVG action icons
      glib-2.0/
        schemas/
          gschemas.compiled       # Must be compiled, not just XML
```

## Step-by-Step Assembly

### 1. Build with Alpine/musl (Recommended)

Alpine produces smaller, more portable AppImages. Build in Docker:

```sh
docker run --rm \
  --platform linux/amd64 \
  -v "$(pwd):/workspace" \
  -w /workspace \
  alpine:3.20 sh -c '
  set -e
  apk add --no-cache \
    build-base meson ninja pkgconf \
    gtk+3.0-dev gtk+3.0 gtkmm3-dev cairomm-dev pangomm-dev libsigc++-dev \
    gdk-pixbuf-dev gdk-pixbuf librsvg-dev librsvg \
    libarchive-dev pugixml-dev \
    hicolor-icon-theme adwaita-icon-theme \
    glib-dev \
    git wget xz squashfs-tools desktop-file-utils \
    musl-utils patchelf shared-mime-info bash python3

  meson setup build -Dprefix=/usr
  meson compile -C build

  rm -rf /tmp/lunduke-install
  DESTDIR=/tmp/lunduke-install meson install -C build

  chmod +x .github/setup-appdir.sh
  ./.github/setup-appdir.sh

  chmod -R a+rwx AppDir
  '
```

**Why Alpine 3.20, not edge:** Alpine edge uses glycin (sandboxed image
loading via bwrap) which is unsuitable for AppDir/AppImage. Alpine 3.20
has traditional `.so` pixbuf loaders.

### 2. Copy the binary and desktop file

```sh
mkdir -p AppDir/usr/bin AppDir/usr/share/applications
cp /tmp/install/usr/bin/my-app AppDir/usr/bin/
cp /tmp/install/usr/share/applications/*.desktop AppDir/usr/share/applications/
# appimagetool needs the desktop file at AppDir root too
cp /tmp/install/usr/share/applications/*.desktop AppDir/
```

### 3. Bundle the dynamic linker

For musl-based builds, bundle the dynamic linker for full self-containment:

```sh
mkdir -p AppDir/lib
# Find and copy the dynamic linker
LINKER=$(readelf -l AppDir/usr/bin/my-app 2>/dev/null | grep "interpreter" | sed 's/.*: \(.*\)/\1/' | tr -d '[]')
# Or detect from ldd output
LINKER=$(ldd AppDir/usr/bin/my-app 2>/dev/null | grep "ld-musl" | awk '{print $1}')
if [ -n "$LINKER" ] && [ -f "$LINKER" ]; then
    cp -f "$LINKER" AppDir/lib/
    echo "Dynamic linker bundled: $(basename "$LINKER")"
fi
```

**Why bundle the linker:** Without it, the AppImage won't run on systems
that don't have musl's dynamic linker at the expected path.

### 4. Set RPATH on the binary

Use patchelf to set a relative RPATH (safe on the binary, not on libs with GResource):

```sh
# Set relative RPATH so the binary finds bundled libs
patchelf --set-rpath '$ORIGIN/../lib' AppDir/usr/bin/my-app
```

**IMPORTANT:** Only use patchelf on the main binary, never on libraries
that contain `.gresource` sections (like libgtk-3.so.0).

### 5. Copy shared libraries via ldd

```sh
mkdir -p AppDir/usr/lib

# Copy direct dependencies
ldd AppDir/usr/bin/my-app | awk '/=>/ { print $3 }' | while read lib; do
    [ -f "$lib" ] && cp -f "$lib" AppDir/usr/lib/
done

# Copy transitive dependencies of bundled libs
for lib in AppDir/usr/lib/lib*.so.*; do
    [ -f "$lib" ] || continue
    ldd "$lib" 2>/dev/null | awk '/=>/ { print $3 }' | while read dep; do
        [ -f "$dep" ] && [ ! -f "AppDir/usr/lib/$(basename "$dep")" ] \
            && cp -f "$dep" AppDir/usr/lib/
    done
done
```

**Why two passes:** The binary's direct deps have their own deps. A single
`ldd` pass on the binary misses transitive deps of libs like libgtk that
depend on libharfbuzz, libgraphite2, etc.

**Alpine/musl note:** musl's ldd output format is slightly different but
the `=>` pattern still works. Verify with `ldd --version` to know which libc.

### 6. Bundle GdkPixbuf loaders

**Detect the version dynamically** - never hardcode:

```sh
# Find the actual loaders directory using glob (avoids hardcoded versions)
PIXBUF_LOADERS_DIR=""
PIXBUF_CACHE_FILE=""

for dir in \
    /usr/lib/gdk-pixbuf-2.0/*/loaders \
    /usr/lib/x86_64-linux-gnu/gdk-pixbuf-2.0/*/loaders \
    /usr/lib64/gdk-pixbuf-2.0/*/loaders; do
    if [ -d "$dir" ]; then
        PIXBUF_LOADERS_DIR="$dir"
        PIXBUF_CACHE_FILE="$(dirname "$dir")/loaders.cache"
        break
    fi
done

# Extract version from the detected path
if [ -n "$PIXBUF_LOADERS_DIR" ]; then
    PIXBUF_VERSION=$(basename "$(dirname "$PIXBUF_LOADERS_DIR")")
else
    PIXBUF_VERSION="2.10.0"
fi
PIXBUF_BASE="gdk-pixbuf-2.0/$PIXBUF_VERSION"

mkdir -p "AppDir/usr/lib/$PIXBUF_BASE/loaders"
cp -a "$PIXBUF_LOADERS_DIR"/* "AppDir/usr/lib/$PIXBUF_BASE/loaders/"

# Rewrite the cache to use RELATIVE filenames only
if command -v gdk-pixbuf-query-loaders >/dev/null 2>&1; then
    gdk-pixbuf-query-loaders "AppDir/usr/lib/$PIXBUF_BASE/loaders/"* 2>/dev/null \
        | sed 's|".*/loaders/|"|g' \
        > "AppDir/usr/lib/$PIXBUF_BASE/loaders.cache" || true
elif [ -f "$PIXBUF_CACHE_FILE" ]; then
    sed 's|".*/loaders/|"|g' "$PIXBUF_CACHE_FILE" \
        > "AppDir/usr/lib/$PIXBUF_BASE/loaders.cache"
fi
```

**Why `pkg-config` fails:** `pkg-config --variable=binary_version gdk-pixbuf-2.0`
may not work in Docker containers or return empty. Use glob detection instead.

**PNG note:** On Debian 13 / Devuan 6, PNG is compiled INTO `libgdk_pixbuf-2.0.so.0`
(no separate `libpixbufloader-png.so`). The built-in decoder still works as long as
`libpng` is bundled. Don't panic if there's no PNG loader module.

### 7. Copy transitive dependencies of pixbuf loaders

Pixbuf loaders link to libraries like librsvg that may not be in the main
binary's dependency chain:

```sh
for loader in "AppDir/usr/lib/$PIXBUF_BASE/loaders/"*.so; do
    [ -f "$loader" ] || continue
    ldd "$loader" 2>/dev/null | while read -r line; do
        lib_path=$(echo "$line" | awk '/=>/ { print $3 }')
        if [ -n "$lib_path" ] && [ -f "$lib_path" ]; then
            lib_name=$(basename "$lib_path")
            if [ ! -f "AppDir/usr/lib/$lib_name" ]; then
                cp -f "$lib_path" "AppDir/usr/lib/" 2>/dev/null || true
                echo "Loader dependency bundled: $lib_name"
            fi
        fi
    done
done
```

### 8. Bundle GLib compiled schemas

```sh
mkdir -p AppDir/usr/share/glib-2.0/schemas
cp -a /usr/share/glib-2.0/schemas/* AppDir/usr/share/glib-2.0/schemas/
# Recompile to include only what we copied
glib-compile-schemas AppDir/usr/share/glib-2.0/schemas/ 2>/dev/null || true
```

### 9. Bundle icon themes

Copy `hicolor` and the theme the app asks for (with their `index.theme`), keep the
sizes the app uses, regenerate the icon cache, and never ship an SVG-only theme
without librsvg. The full procedure and the traps: [references/icon-themes.md](references/icon-themes.md).

### 10. Bundle GTK GResource files

On Alpine, GTK's UI templates (gtkcombobox.ui etc) are in separate
`.gresource` files alongside the `.so`:

```sh
for gtk_lib in AppDir/usr/lib/libgtk-3*.so.*; do
    [ -f "$gtk_lib" ] || continue
    gresource="${gtk_lib}.gresource"
    if [ ! -f "$gresource" ]; then
        sys_gr="/usr/lib/$(basename "$gresource")"
        if [ -f "$sys_gr" ]; then
            cp -f "$sys_gr" AppDir/usr/lib/
        fi
    fi
done
```

### 11. Bundle GTK modules (global menu / appmenu)

A host GTK module can never load into a bundled (especially musl) GTK: build it
against the bundled toolkit, ship its schema, and expose it through `GTK_PATH`,
never `GTK_MODULES`. Procedure: [references/gtk-modules.md](references/gtk-modules.md).

### 12. Write the AppRun launcher

For musl builds with bundled dynamic linker, cd to AppDir root so the
kernel resolves the relative interpreter path:

```sh
cat > AppDir/AppRun << 'APPRUN'
#!/bin/sh
set -eu

SELF="$0"
while [ -L "$SELF" ]; do
    LINK="$(readlink "$SELF")"
    case "$LINK" in
        /*) SELF="$LINK" ;;
        *)  SELF="$(dirname "$SELF")/$LINK" ;;
    esac
done

HERE="$(CDPATH= cd -- "$(dirname -- "$SELF")" && pwd -P)"
PREFIX="$HERE/usr"

# Find loaders
GDK_PIXBUF_BASE="$PREFIX/lib/gdk-pixbuf-2.0"
GDK_PIXBUF_LOADERS=""
GDK_PIXBUF_CACHE=""
if [ -d "$GDK_PIXBUF_BASE" ]; then
    for d in "$GDK_PIXBUF_BASE"/*; do
        if [ -d "$d" ]; then
            GDK_PIXBUF_LOADERS="$d/loaders"
            GDK_PIXBUF_CACHE="$d/loaders.cache"
            break
        fi
    done
fi

LOADERS_PATH=""
[ -n "$GDK_PIXBUF_LOADERS" ] && [ -d "$GDK_PIXBUF_LOADERS" ] && LOADERS_PATH="$GDK_PIXBUF_LOADERS"

export LD_LIBRARY_PATH="$PREFIX/lib${LOADERS_PATH:+:$LOADERS_PATH}${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"

[ -n "$GDK_PIXBUF_LOADERS" ] && [ -d "$GDK_PIXBUF_LOADERS" ] \
    && export GDK_PIXBUF_MODULEDIR="$GDK_PIXBUF_LOADERS"
[ -n "$GDK_PIXBUF_CACHE" ] && [ -f "$GDK_PIXBUF_CACHE" ] \
    && export GDK_PIXBUF_MODULE_FILE="$GDK_PIXBUF_CACHE"

export XDG_DATA_DIRS="$PREFIX/share:/usr/share${XDG_DATA_DIRS:+:$XDG_DATA_DIRS}"
export GTK_DATA_PREFIX="$PREFIX"
export GTK_EXE_PREFIX="$PREFIX"

# Point GTK's module search at the bundle. A host GTK_PATH would offer
# glibc-linked modules this musl GTK cannot load. GTK_MODULES is deliberately
# NOT set: only a session that asks for the global menu gets it.
export GTK_PATH="$PREFIX/lib/gtk-3.0"

[ -d "$PREFIX/share/glib-2.0/schemas" ] \
    && export GSETTINGS_SCHEMA_DIR="$PREFIX/share/glib-2.0/schemas"

# cd to AppDir root so the kernel resolves the relative interpreter path
cd "$HERE"
exec "$PREFIX/bin/my-app" "$@"
APPRUN
chmod +x AppDir/AppRun
```

### 13. Build the AppImage

```sh
# Download appimagetool (static binary, auto-downloads runtime)
ARCH=x86_64  # or aarch64, i686, armhf
# A tagged appimagetool release, verified before anything in it runs. The
# checksum is that of releases/download/<tag>/appimagetool-<arch>.AppImage;
# 1.9.1: x86_64 ed4ce84f0d9caff66f50bcca6ff6f35aae54ce8135408b3fa33abfc3cb384eb0,
#        aarch64 f0837e7448a0c1e4e650a93bb3e85802546e60654ef287576f46c71c126a9158.
APPIMAGETOOL_VERSION=1.9.1
APPIMAGETOOL_SHA256=ed4ce84f0d9caff66f50bcca6ff6f35aae54ce8135408b3fa33abfc3cb384eb0
wget -q "https://github.com/AppImage/appimagetool/releases/download/${APPIMAGETOOL_VERSION}/appimagetool-${ARCH}.AppImage" -O appimagetool
echo "${APPIMAGETOOL_SHA256}  appimagetool" | sha256sum -c -
chmod +x appimagetool
./appimagetool --appimage-extract

# Build with update information for AppImageUpdate
export PATH="$(pwd)/squashfs-root/usr/bin:${PATH}"
./squashfs-root/usr/bin/appimagetool \
    --updateinformation "gh-releases-zsync://OWNER/REPO/continuous/ARCH-REPO.AppImage.zsync" \
    AppDir
```

### 14. Multi-arch CI with GitHub Actions

Build inside a pinned Alpine container per architecture (`docker run --platform`),
then publish all AppImages from one release job. The workflow:
[references/multi-arch-ci.md](references/multi-arch-ci.md); the release and
hardening details live in the build-appimage-ci skill.

## Keep AppRun's environment out of the programs the app starts

Everything AppRun exports for the bundled toolkit is inherited by every host
program the app starts (a browser, `xdg-open`, `/bin/sh`, bubblewrap), and a host
program that picks up a bundled `LD_LIBRARY_PATH` or pixbuf-loader cache crashes.
Two layers: give every bundled ELF a relative `DT_RPATH` instead of exporting
`LD_LIBRARY_PATH`, and preload a small `exec.so` that restores the original
environment on every `exec` and `posix_spawn`. An app that re-executes itself
inside bubblewrap additionally needs a hook of its own to bind the bundle, and
without `LD_LIBRARY_PATH` the pixbuf cache must be generated with absolute paths
at start. All of it, with the code:
[references/child-environment.md](references/child-environment.md).

## Common Pitfalls

Thirteen failure modes with symptom, cause and fix are in
[references/pitfalls.md](references/pitfalls.md):

- Pitfall 1: patchelf corrupts .gresource sections
- Pitfall 2: loaders not found at runtime
- Pitfall 3: loaders.cache has absolute paths
- Pitfall 4: Adwaita SVG-only theme crashes
- Pitfall 5: Icons not showing in UI
- Pitfall 6: pixbuf loader version mismatch
- Pitfall 7: Dynamic linker not bundled
- Pitfall 8: File permissions in Docker
- Pitfall 9: appimagetool needs desktop file at root
- Pitfall 10: Binary RUNPATH overriding LD_LIBRARY_PATH
- Pitfall 11: global menu silently does nothing
- Pitfall 12: libraries that load the host's drivers (libva and friends)
- Pitfall 13: bundling the host's display/EGL stack (blank window on Wayland / Mesa 25+)

## Verification Checklist

Before shipping, run the checks in
[references/verification-checklist.md](references/verification-checklist.md):
binary and loader present, RPATH set, loaders.cache relative, schemas compiled,
icon theme and gresource present, and a headless start. The portability test
proper is the verify-appimage-portability skill.

## Reference: Why Each Env Var Matters

| Variable | Purpose | What breaks without it |
|---|---|---|
| `LD_LIBRARY_PATH` | Tells dlopen where to find bundled .so files | Binary can't load any bundled libs |
| `GDK_PIXBUF_MODULEDIR` | Tells gdk-pixbuf where loader .so files are | Loaders not found for format detection |
| `GDK_PIXBUF_MODULE_FILE` | Points to the loaders.cache | gdk-pixbuf doesn't know about any loaders |
| `XDG_DATA_DIRS` | Where to find icon themes, MIME info | GTK falls back to GResource, may crash |
| `GTK_DATA_PREFIX` | Root for GTK data files | GTK can't find its own data |
| `GSETTINGS_SCHEMA_DIR` | Where compiled GSettings schemas are | GSettings operations fail |
| `GTK_PATH` | Where GTK looks for modules (immodules, theme engines, appmenu) | GTK tries the host's glibc modules and cannot load them |
| `GTK_MODULES` | Which modules to load. **Leave alone** - inherit it from the session | Setting it forces the module on desktops that cannot use it |
