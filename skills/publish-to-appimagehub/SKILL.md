---
name: publish-to-appimagehub
description: Use when an AppImage is to be listed in AppImageHub, the community catalog at appimage.github.io (a pull request adding one file to its data/ directory), or when such a pull request failed - what the catalog's automated test does (downloads the newest release's x86_64 AppImage, mounts it as SquashFS, lints the AppDir, starts it on the oldest supported Ubuntu LTS on X11 with the C locale and no network, screenshots it within 30 seconds), the rules it enforces (AppRun, .DirIcon, exactly one root desktop file passing desktop-file-validate with Icon and Categories once, AppStream metainfo validated by appstreamcli, no glibc newer than the test system unless glibc is bundled, light theme and English by default, no empty window or error dialog, no network needed), how to name the data/ file and the AppImage, release channels, and a pre-submission check to run in CI so the catalog's test passes first time. Pairs with make-standalone-appdir, build-appimage-ci and verify-appimage-portability.
---

# Publish an AppImage to AppImageHub

[AppImageHub](https://appimage.github.io) is the community catalog that app
stores and command-line installers read (`feed.json`). It does not host
files: an entry is one file in `data/` of the repository
`AppImage/appimage.github.io` containing a link, and everything shown on the
page (name, description, icon, screenshot, license, update information) is
read out of the AppImage itself by an automated test. The test runs on every
pull request and posts its result as a comment, with a screenshot on success
and a labelled error on failure. Most first submissions fail on something
that takes a minute to fix in the packaging, so run the checks below in CI
before opening the pull request.

Everything here follows the catalog's README and its `code/` scripts
(`worker.sh`, `appdir-lint.sh`, `check-name.sh`, `check-libc.sh`,
`check-screenshot.sh`, `diagnose.sh`); read them when a rule seems to have
changed.

## What the catalog's test does, step by step

1. **Resolves the link** in the `data/` file. For a GitHub, Codeberg or GitLab
   repository link it takes the newest release (or, if no release has an
   AppImage, the newest pre-release) and among its files ending in `.AppImage`
   the one for **x86_64** that is not a debug or nightly build; other
   architectures are left out. For a directory link ending in `/` it reads the
   listing and descends into the newest version. If several AppImages fit
   (`App-qt5-x86_64.AppImage` and `App-qt6-x86_64.AppImage`), it fails
   (`error-no-appimage-in-release`) rather than guess.
2. **Downloads with `wget`.** No login, no cookies, no HTML landing page
   (`error-download`, `error-not-an-appimage`).
3. **Mounts it with a separate runtime**, never by executing the AppImage's
   own runtime. Only type 2 AppImages with a **SquashFS** payload mount;
   DwarFS payloads (uruntime, quick-sharun) fail with `error-not-squashfs`.
4. **Lints the AppDir** (`appdir-lint.sh`, the rules are listed below) and
   reads update information and signature.
5. **Classifies the C library** (`check-libc.sh`): `X-AppImage-Libc=bundled`
   when the payload ships `libc.so.6` plus `ld-linux*.so*` (or musl's
   `ld-musl-*.so.1`), `none` when nothing is dynamically linked, otherwise
   `host`, with the newest `GLIBC_2.xx` symbol version any ELF file in the
   payload references. `X-AppImage-Runtime=static` when the runtime does not
   need the host's libc either. The page shows "self-contained" for
   `bundled`+`static`.
6. **Checks the names** (`check-name.sh`, see below) and finds the icon.
7. **Runs the application** under `firejail --net=none` on Xvfb (800x600x24,
   icewm, no tray, C locale, no dark-mode preference), waits 10 s, then polls
   up to 20 s more for a visible window. An application whose desktop file
   sets `Terminal=true` runs as `--help` in xterm instead. If no window
   appeared and its files mention a tray API, it runs again with a tray
   (stalonetray) and clicks the icon.
8. **Screenshots the active window** and runs `check-screenshot.sh` on it:
   error if the window is almost one colour with no text (`error-blank-window`)
   or OCR finds an error message (`error-message-on-screen`) or text mostly
   in Chinese, Arabic, Cyrillic or Greek script (`error-not-english`); warning
   if it is mostly dark (60 % of pixels darker than 35 % grey).
9. **Writes the entry** from the desktop file, the AppStream metainfo
   (summary, license, your own screenshot), the icon, the update information.

The process runs on the **oldest Ubuntu LTS release still in standard
support** with nothing installed beyond test tooling. Whatever the AppImage
needs and does not bring fails there, which is the point.

## Technical requirements the AppImage must meet

Everything `appdir-lint.sh` calls `FATAL` fails the test; `WARNING` is shown
but passes.

- `AppRun` at the root, executable. (A file that is not executable anywhere in
  the start path gives `error-not-executable`; Tauri and linuxdeploy builds
  have shipped `AppRun.wrapped` as 0770 owned by root. `chmod -R a+rX AppDir`
  before `appimagetool`.)
- `.DirIcon` at the root, a PNG (SVG passes with a warning). If it is a
  symlink, its target must be relative and inside the AppDir; an absolute
  symlink into `/home/runner/...` gives `error-no-icon` on every other
  machine.
- **Exactly one** `*.desktop` file at the root. It must pass
  `desktop-file-validate` and have `Icon=` and `Categories=` exactly once in
  `[Desktop Entry]`. `Categories` needs at least one registered main category
  (`Utility;`, `Graphics;`, ...) and a trailing `;`.
- An icon matching `Icon=`: `usr/share/icons/hicolor/scalable/apps/<Icon>.svg`,
  or `<Icon>.png` under `128x128`, `256x256` or `512x512`, or `<Icon>.png` /
  `.svg` at the root. Without one the test falls back to `.DirIcon`.
- AppStream metainfo in `usr/share/metainfo/<id>.appdata.xml` or
  `<id>.metainfo.xml` is recommended (the page gets your summary, license and
  your own screenshots instead of the test's). If present, **it must pass**
  `appstreamcli validate-tree AppDir`: an invalid file is fatal, a missing one
  only a warning. Common failures: an `<id>` that does not match the desktop
  file's name, a `<project_license>` that is not an SPDX expression, a
  `<release>` list in descending order, a screenshot URL that is not `https`.
- No file from the AppImage
  [excludelist](https://github.com/AppImage/AppImages/blob/master/excludelist)
  (`libGL.so.1`, `libEGL.so.1`, `libdrm.so.2`, `libxcb.so.1`, `libwayland-*`,
  `libfontconfig.so.1`, `libgbm.so.1`, `libasound.so.2`, the GLVND and Mesa
  libraries, ...). Each one found is a warning, not a failure, but each is a
  real portability bug: those libraries belong to the host's GPU, display and
  audio stack. make-standalone-appdir's pitfalls 12 and 13 explain which, and
  why.
- Starts on the oldest supported Ubuntu LTS. Either build on that release (the
  oldest `ubuntu-*` runner GitHub still offers, or a container of it) so that no
  binary references a newer `GLIBC_`/`GLIBCXX_`/`CXXABI_` symbol version, or
  bundle glibc together with its loader as make-standalone-appdir describes;
  then the host's glibc is irrelevant and the entry says so. Otherwise:
  `error-glibc-too-new`. Every ELF file in the payload counts, including
  plugins that are rarely loaded.
- Shows its **main window within 30 seconds** when started without arguments,
  **without network**. An application that exits because it cannot reach a
  server, a login, a licence check or an update endpoint fails with
  `error-app-exits`; one that waits for the network and never shows a window
  with `error-no-window`. Show the window first, connect afterwards, and show
  a message (an offline page for Electron) rather than exit.
- The window must show content: a menu bar over a blank canvas is
  `error-blank-window`, a crude "could not connect" dialog is
  `error-message-on-screen`. A first-run wizard or a welcome screen is fine.
  Whatever appears is the catalog's screenshot unless the metainfo provides
  one.
- **Light theme by default.** The test session has no dark-mode preference.
  Follow the portal's colour-scheme setting (or the toolkit's) and default to
  light when there is none; a hard-coded dark theme gives a warning and loses
  the `screenshot-ok` label. Games and video players with a dark canvas are
  accepted.
- **English interface** with the C locale: fall back to English, not to the
  author's language, when the system language is unset or unsupported.
- Works on **X11**. There is no Wayland in the test and none is planned.
- Must not need `unprivileged_userns_clone`-style tricks beyond what the test
  enables; Electron's SUID sandbox helper failing is `error-electron-sandbox`
  (ship the AppImage with `--no-sandbox` handling or the setuid helper
  properly, see the catalog's issue 2563).
- Recommended, warnings only: embedded update information
  (`appimagetool -u 'gh-releases-zsync|owner|repo|latest|App-*-x86_64.AppImage.zsync'`
  with the `.zsync` uploaded next to the file; build-appimage-ci covers which
  release stream to point at) and a GPG signature (`appimagetool -s`).

## Naming

The file in `data/` becomes the entry's name and page
(`https://appimage.github.io/<Name>/`). For new files these are errors
(`error-filename`); for existing entries only warnings.

- The application's name as spelled in its desktop file `Name=`, with its
  capitals: `data/Krita`, not `data/krita`. (An all-lowercase name without
  `Terminal=true` draws a note asking whether it is really a command-line
  tool.)
- Blanks become `_`, also where the AppImage's file name uses `-`:
  "XI on Anything" is `data/XI_on_Anything`.
- No version, no architecture, no `.AppImage`, no extension such as `.md`.
- No "App" as a separate word (`Foo-App`, `foo_app`); `WhatsApp` is fine.
- No "AppImage" or "Linux" unless part of the application's name.
- No `:`, `?`, `#`, `%` (they break the page URL).
- Not an existing entry in different capitalisation; change that file instead.

The AppImage's own file name should be `App-<version>-x86_64.AppImage`, "App"
spelled like the repository and the desktop file, blanks as `_`, no "linux".
Keep `x86_64` (or `aarch64`) in the name: the test picks the x86_64 build by
it. Keep one x86_64 AppImage per release, or put a direct link or a channel
suffix into the `data/` file.

## The data file and the pull request

The file contains **one line**: ideally the repository page
(`https://github.com/User/App`, or the Codeberg/GitLab equivalent), so that
the test always finds the newest release and the entry never goes stale.
Alternatives: a download directory whose listing shows versions
(`https://download.kde.org/stable/digikam/`, an openSUSE Build Service
`.../AppImage/` directory) or a direct link that does not change with the
version. Links that embed the version number are accepted but get no
`Download` link on the page.

Channels and variants: `https://github.com/User/App#nightly` makes this entry
the nightly channel; without a suffix, releases and files containing
`nightly`, `beta`, `esr`, ... are skipped so that one repository can have one
entry per channel. The part after `#` is a word of the file name or a pattern
with `*` and `?` (`#App_*ubuntu-24.04`), which is also how to choose among
several AppImages of one release.

The pull request must **add or change exactly that one file**, one application
per pull request (`error-multiple-files` otherwise). A change that moves an
entry to a different owner (another user, organisation or website) gets
`manual-check-needed`; explain it in the pull request with a link to the
project's announcement.

After submitting: the comment with the result arrives within minutes.
`/retest` runs it again (after publishing a fixed release); a comment saying
the fix is out ("fixed in 1.2.3") re-runs a failed test by itself.
`/appstream` posts how to supply your own screenshot through the metainfo
file. `error-*` labels name the failure, `screenshot-ok` a clean pass,
`manual-check-needed` a maintainer's turn. Merging takes a maintainer and some
time. "Please remove my app" opts the application out; removing a merged
entry is a pull request that deletes the `data/` file.

## Run the catalog's checks before submitting

Put this in the CI job that builds the AppImage (build-appimage-ci), after
`appimagetool`, or run it locally. It needs `desktop-file-utils`,
`appstream` and `binutils`; `squashfs-root` comes from
`./App.AppImage --appimage-extract` (works without FUSE). It implements the
lint's fatal rules and the glibc and name checks; the catalog's vendored
`code/appdir-lint.sh` is the reference if the two ever disagree.

```bash
#!/usr/bin/env bash
# appimagehub-check.sh App-1.0-x86_64.AppImage [MaxGlibc e.g. 2.35]
set -euo pipefail
APPIMAGE=$1; MAX_GLIBC=${2:-}
fail() { echo "FAIL: $*" >&2; RC=1; }; RC=0
rm -rf squashfs-root; "./$APPIMAGE" --appimage-extract >/dev/null
A=squashfs-root

[ -x "$A/AppRun" ] || fail "AppRun missing or not executable"
[ -e "$A/.DirIcon" ] || fail ".DirIcon missing"
case "$(readlink "$A/.DirIcon" || true)" in /*) fail ".DirIcon is an absolute symlink" ;; esac
file -b "$(readlink -f "$A/.DirIcon")" | grep -q 'PNG image' || echo "WARN: .DirIcon is not PNG"
mapfile -t D < <(find "$A" -maxdepth 1 -name '*.desktop')
[ ${#D[@]} -eq 1 ] || fail "need exactly one root .desktop file, found ${#D[@]}"
if [ ${#D[@]} -eq 1 ]; then
  desktop-file-validate "${D[0]}" || fail "desktop-file-validate"
  for k in Icon Categories; do
    n=$(awk -v k="$k" '/^\[/{s=($0=="[Desktop Entry]")} s&&index($0,k"=")==1{c++} END{print c+0}' "${D[0]}")
    [ "$n" -eq 1 ] || fail "$k= must appear exactly once in [Desktop Entry] (found $n)"
  done
  ICON=$(sed -n 's/^Icon=//p' "${D[0]}" | head -1)
  find "$A" \( -name "$ICON.png" -o -name "$ICON.svg" \) | grep -q . || fail "no icon file for Icon=$ICON"
  grep -q '^Terminal=true' "${D[0]}" || true
fi
if find "$A/usr/share/metainfo" "$A/usr/share/appdata" -maxdepth 1 \
     \( -name '*.appdata.xml' -o -name '*.metainfo.xml' \) 2>/dev/null | grep -q .; then
  appstreamcli validate-tree "$A" || fail "appstreamcli validate-tree"
else
  echo "WARN: no AppStream metainfo (recommended)"
fi
# Excludelist libraries that belong to the host (the full list is upstream)
for lib in libGL.so.1 libEGL.so.1 libGLX.so.0 libGLdispatch.so.0 libOpenGL.so.0 libgbm.so.1 \
           libdrm.so.2 libxcb.so.1 libX11.so.6 libwayland-client.so.0 libwayland-egl.so.1 \
           libfontconfig.so.1 libasound.so.2 libnvidia-glcore.so; do
  find "$A" -name "$lib" | grep -q . && echo "WARN: excludelist library bundled: $lib"
done
# Newest glibc symbol version any ELF needs, unless glibc is bundled
if find "$A" -name 'libc.so.6' | grep -q . && find "$A" -name 'ld-linux*.so*' | grep -q .; then
  echo "glibc: bundled (X-AppImage-Libc=bundled)"
elif [ -n "$MAX_GLIBC" ]; then
  NEED=$(find "$A" -type f -exec sh -c 'head -c4 "$1" | grep -q ELF' _ {} \; -print \
    | xargs -r objdump -T 2>/dev/null | grep '\*UND\*' | grep -oE 'GLIBC_[0-9.]+' \
    | sed 's/GLIBC_//' | sort -uV | tail -1 || true)
  echo "glibc: host, newest symbol version needed: ${NEED:-none}"
  [ -z "$NEED" ] || [ "$(printf '%s\n%s\n' "$NEED" "$MAX_GLIBC" | sort -V | tail -1)" = "$MAX_GLIBC" ] \
    || fail "needs GLIBC_$NEED, test system has $MAX_GLIBC"
fi
case "$APPIMAGE" in *[Ll]inux*) echo "WARN: 'linux' in the file name tells nothing";; esac
case "$APPIMAGE" in *x86_64*|*aarch64*|*armhf*|*i686*) ;; *) echo "WARN: no architecture in the file name";; esac
exit $RC
```

For `MaxGlibc`, pass the glibc of the oldest Ubuntu LTS in standard support
(check `https://ubuntu.com/about/release-cycle`; `ldd --version` in a
container of that release tells the number). A workflow step that runs the
AppImage in exactly that container, offline, is the rest of the proof:
verify-appimage-portability's job shape with `ubuntu:<oldest LTS>` as the
image, `--network none` on `docker run`, `LANG=C`, and the window check
within 30 s matches what the catalog does. Grep the screenshot's OCR text or
the app's log for the strings the test treats as error dialogs if the app has
an offline warning that might be mistaken for one.

## Reading a failed test

The result comment names the label and a hint. Map them back:

| Label | Where to look |
| --- | --- |
| `error-glibc-too-new` | Build on the oldest LTS or bundle glibc with its loader (make-standalone-appdir). |
| `error-missing-library` | "error while loading shared libraries": bundle it, then re-run verify-appimage-portability's stock-container test. |
| `error-not-executable` | File modes inside the AppDir; `chmod -R a+rX` before packaging. |
| `error-app-exits` | Needs network, a server, a display feature or crashes at start; run it with `--net=none` in firejail or `docker run --network none` and read stderr (troubleshoot-appimage-runtime). |
| `error-no-window` | Nothing visible in 30 s: slow start, hidden main window, waits for network, or crashed silently. |
| `error-blank-window`, `error-message-on-screen` | What the app shows first when offline on a fresh profile. |
| `error-not-english` | Locale fallback. |
| `error-no-icon`, `error-missing-file` | `.DirIcon`, `Icon=` match, `AppRun`, desktop file. |
| `error-not-squashfs`, `error-not-an-appimage` | Payload format or the link points at an HTML page. |
| `error-no-appimage-in-release` | No `.AppImage` for x86_64 in the newest release, or several; name the architecture, use a `#pattern`, or a direct link. |
| `error-filename`, `error-multiple-files`, `error-data-file` | The pull request itself: file name rules, exactly one file, URL on the first line. |
| `error-electron-sandbox` | Electron's SUID sandbox under the test's user namespaces. |

Fix in the packaging, publish a release, comment `/retest`.
