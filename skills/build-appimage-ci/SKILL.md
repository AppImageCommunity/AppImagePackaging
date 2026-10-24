---
name: build-appimage-ci
description: Use when writing or debugging a GitHub Actions workflow that packages a Linux GUI app (CMake or Meson, Qt or GTK) into an AppImage - picking a build environment that matches the AppImage tooling, linuxdeploy with plugin-qt versus manual AppDir assembly, packaging an existing release build instead of compiling twice, keeping CMake install() paths self-consistent, extracting AppImage tools on FUSE-less runners, publishing to GitHub Releases with plain gh (a rolling continuous pre-release whose tag follows the branch, attaching to a tag's release, zsync update information), why a tag created by GITHUB_TOKEN never triggers a workflow and the workflow_call fix, hardening (least-privilege permissions, persist-credentials, pinned actions, a checksum-verified appimagetool, the bash -e trap), the fork-and-push refspec pitfall, and delegating the watch-fix-push loop to subagents. Pairs with make-standalone-appdir and verify-appimage-portability.
---

# Build an AppImage in GitHub Actions

Lessons from two real pipelines: `lunduke-paint` (GTK3/GTKmm, Meson, built in an
Alpine container, multi-arch) and `OmaPhoto` (Qt6, CMake, built directly on an
Ubuntu runner, x86_64-only because of a vendored ONNX Runtime blob). Both end
in the same shape: build -> assemble AppDir -> appimagetool -> upload ->
(on push) publish a rolling "continuous" release.

See also: [make-standalone-appdir](../make-standalone-appdir/SKILL.md) for the
low-level AppDir/library-bundling mechanics referenced throughout this skill;
[troubleshoot-appimage-runtime](../troubleshoot-appimage-runtime/SKILL.md)
for diagnosing a build that succeeds but produces an AppImage that fails,
crashes, or misbehaves at runtime; and
[verify-appimage-portability](../verify-appimage-portability/SKILL.md)
for the next step once the build here goes green - proving the AppImage
actually runs.

## A project that already builds release binaries: package, don't rebuild

When the repository's release workflow already compiles the binary on a
runner you can install the runtime packages on (`ubuntu-24.04`,
`ubuntu-24.04-arm`), the AppImage is a packaging *step* after the existing
build, not a second workflow compiling the same thing: put the AppDir
assembly and `appimagetool` into a composite action
(`.github/actions/appimage/action.yml` with `binary`, `name` and `output`
inputs), call it from the release job's matrix right after the archive is
made, and add `dist/*.AppImage` to the artifact glob the publish job already
uploads. The same stripped binary lands in the archive and in the AppImage,
which keeps provenance attestations and checksum manifests honest; name the
file after the project's existing scheme (`<app>-<version>-<target>.AppImage`)
so the manifests and download logic need no special case. A separate
validation workflow that compiles the tree and runs the container tests is
still worth having, but gate it with `paths:` to the packaging scripts, the
action and the workflow itself (plus any source the AppImage depends on,
such as a sandbox module), so the double compile happens only when the
packaging changes.

Two details on the release-job integration: an `aarch64` matrix leg needs
the action to pick the loader and `appimagetool` by `uname -m` (two
checksums, one per architecture), and a checksum/signing script that
enumerates `*.tar.gz` by exact name keeps working untouched, because the
`.AppImage` is simply not in its set.

## Pick the build environment to match your constraints, not habit

- If every dependency is available for multiple architectures and you want
  `x86_64` + `i686` + `aarch64` + `armhf` in one workflow, build inside a
  pinned Alpine container per arch via `docker run --platform ...`
  (lunduke-paint's approach). Alpine/musl also makes the smallest bundles -
  see [make-standalone-appdir](../make-standalone-appdir/SKILL.md)'s "Golden
  Rules" for why.
- If a dependency is only shipped as a prebuilt glibc x86_64 blob (a vendor
  SDK, a `.tgz` release like ONNX Runtime), don't fight it - build directly
  on `ubuntu-24.04` with apt-get, skip the container, and ship x86_64 only.
  Trying to cross-compile or musl-ify a vendor blob costs more than it's worth.
- Either way, pin the base image/runner version explicitly
  (`alpine:3.20`, `ubuntu-24.04`, not `latest`) so a base-image bump doesn't
  silently change your glibc/musl ABI baseline between runs.

## Reproduce your Dockerfile's setup exactly, don't approximate it

If you already have a `Dockerfile.dev` (or similar) that another CI job uses
to build `.deb`/`.rpm`/`.pkg.tar.zst` via `cpack`, and you're now writing a
*separate* AppImage job that reinstalls the same toolchain as plain workflow
`run:` steps instead of literally reusing that Dockerfile, diff your steps
against the Dockerfile line by line. It is easy to drop a step that looks
like packaging-only busywork but that your own `CMakeLists.txt` actually
depends on.

Concretely: OmaPhoto's `Dockerfile.dev` copies the ONNX Runtime's `LICENSE`
and `ThirdPartyNotices.txt` into `/usr/local/share/doc/onnxruntime/` right
after installing the runtime. That looked like packaging boilerplate, so the
first version of the AppImage workflow skipped it - and `cmake` immediately
failed with `bundling ONNX Runtime needs
/usr/local/share/doc/onnxruntime/LICENSE`, because the project's own
`CMakeLists.txt` asserts that file exists before it will bundle the runtime.
One missing `cp` cost a full CI round trip. Read the *whole* setup script,
not just the parts that look build-relevant.

## Keep CMake's install() paths self-consistent

If the project uses `include(GNUInstallDirs)`, then `CMAKE_INSTALL_LIBDIR`
may resolve to `lib`, `lib64`, or a Debian multiarch triplet path like
`lib/x86_64-linux-gnu` depending on the distro. Any custom
`install(IMPORTED_RUNTIME_ARTIFACTS ...)` line and any matching
`INSTALL_RPATH` you set (e.g. for a bundled vendor `.so`) must both reference
`${CMAKE_INSTALL_LIBDIR}`, not a literal `"lib"`. If they don't agree, the
binary's RPATH points somewhere the library was never installed, and you'll
only find out at runtime (or when `verify-appimage-portability`'s screenshot
job reports a missing `.so`), not at build time.

## Qt apps: use linuxdeploy + linuxdeploy-plugin-qt

Manually chasing every Qt platform plugin, imageformat plugin, and
icon-engine `.so` is not worth it. Use `linuxdeploy` with the `qt` plugin:

```sh
wget -c https://github.com/linuxdeploy/linuxdeploy/releases/download/continuous/linuxdeploy-x86_64.AppImage
wget -c https://github.com/linuxdeploy/linuxdeploy-plugin-qt/releases/download/continuous/linuxdeploy-plugin-qt-x86_64.AppImage
chmod +x linuxdeploy-x86_64.AppImage linuxdeploy-plugin-qt-x86_64.AppImage

# GitHub-hosted runners have no /dev/fuse - extract instead of running the AppImage directly.
./linuxdeploy-x86_64.AppImage --appimage-extract && mv squashfs-root squashfs-root-linuxdeploy
./linuxdeploy-plugin-qt-x86_64.AppImage --appimage-extract && mv squashfs-root squashfs-root-linuxdeploy-qt

export PATH="$(pwd)/squashfs-root-linuxdeploy-qt/usr/bin:$PATH"   # so linuxdeploy finds the qt plugin
export QMAKE=/usr/lib/qt6/bin/qmake6                            # tell the qt plugin which Qt to deploy

./squashfs-root-linuxdeploy/usr/bin/linuxdeploy \
  --appdir AppDir \
  --executable AppDir/usr/bin/<binary> \
  --desktop-file AppDir/usr/share/applications/<id>.desktop \
  --icon-file packaging/icons/<icon>-256.png \
  --plugin qt
```

`--icon-file` does not need to be pre-renamed to match the desktop file's
`Icon=` key - linuxdeploy handles that. Install `qt6-base-dev-tools` (for
`qmake6`), `patchelf`, and `desktop-file-utils` alongside the normal Qt build
deps, or the plugin step fails obscurely.

## GTK apps: prefer manual AppDir assembly

`appimagetool -s deploy` and generic auto-deploy tooling can corrupt a GTK
app's `.gresource` ELF sections via patchelf. lunduke-paint instead ships its
own `.github/build-appmenu-module.sh` and `.github/setup-appdir.sh`, run
inside the same Alpine container as the build, to assemble the AppDir by
hand. See [make-standalone-appdir](../make-standalone-appdir/SKILL.md) for the
exact bundling rules (loaders.cache, GTK_PATH vs GTK_MODULES, icon theme
assembly, etc.) - that skill is the low-level companion to this one.

## Bundling AppImage tools: always extract, never exec directly

GitHub-hosted runners (and most CI containers) have no `/dev/fuse`, so any
tool distributed as an AppImage - `linuxdeploy`, `linuxdeploy-plugin-qt`,
`appimagetool` itself - must be extracted first:

```sh
wget -q ".../releases/download/<tag>/appimagetool-x86_64.AppImage" -O appimagetool
echo "<sha256>  appimagetool" | sha256sum -c -   # pinned release, see "Harden the workflow itself"
chmod +x appimagetool
./appimagetool --appimage-extract
export PATH="$(pwd)/squashfs-root/usr/bin:$PATH"
squashfs-root/usr/bin/appimagetool --updateinformation "..." AppDir Out.AppImage
```

Running the fetched `.AppImage` directly (`./appimagetool ...`) will fail on
a stock GitHub runner with a FUSE error.

## Publishing to GitHub Releases: continuous and tagged

Two streams, one job. A push to the default branch feeds a rolling
pre-release tagged `continuous` that always carries the newest build; a
pushed version tag gets the build attached to *that* tag's release. Both
belong in a single `release` job at the end of the workflow, after the
verification jobs (see verify-appimage-portability), so nothing that failed
its container test is ever published:

```yaml
permissions:
  contents: read          # workflow-wide; only the release job gets more
concurrency:
  group: appimage-${{ github.event_name }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # build, test ...
  release:
    needs: [build, test]
    if: github.event_name == 'push'      # never for a PR or a manual run
    runs-on: ubuntu-24.04
    permissions:
      contents: write
    steps:
      - uses: actions/download-artifact@<sha> # v4
        with: { name: appimage }

      - name: Publish the continuous build
        if: github.ref == 'refs/heads/main'
        env: { GH_TOKEN: "${{ github.token }}" }
        run: |
          set -euo pipefail
          cat >notes.md <<EOF
          Rolling build from [\`${GITHUB_SHA:0:9}\`](https://github.com/$GITHUB_REPOSITORY/commit/$GITHUB_SHA); replaced on every push that passes the tests.
          EOF
          # Recreate rather than edit, so the tag moves to this commit. The download
          # URL .../releases/download/continuous/<file> is stable across that.
          gh release delete continuous --repo "$GITHUB_REPOSITORY" --cleanup-tag --yes 2>/dev/null || true
          gh api -X DELETE "repos/$GITHUB_REPOSITORY/git/refs/tags/continuous" 2>/dev/null || true
          gh release create continuous --repo "$GITHUB_REPOSITORY" --target "$GITHUB_SHA" --prerelease \
            --title "Continuous build" --notes-file notes.md \
            App-*-x86_64.AppImage App-*-x86_64.AppImage.zsync

      - name: Attach to the tag's release
        if: startsWith(github.ref, 'refs/tags/')
        env: { GH_TOKEN: "${{ github.token }}", TAG: "${{ github.ref_name }}" }
        run: |
          set -euo pipefail
          case "$TAG" in
            v[0-9]*) ;;    # only this app's own version tags (see below)
            *) echo "::notice::$TAG is not a release tag of this app"; exit 0 ;;
          esac
          if ! gh release view "$TAG" --repo "$GITHUB_REPOSITORY" >/dev/null 2>&1; then
            case "$TAG" in *beta* | *rc* | *alpha*) pre=--prerelease ;; *) pre= ;; esac
            gh release create "$TAG" --repo "$GITHUB_REPOSITORY" --verify-tag $pre \
              --title "$TAG" --notes "Built from [\`${GITHUB_SHA:0:9}\`](https://github.com/$GITHUB_REPOSITORY/commit/$GITHUB_SHA)."
          fi
          gh release upload "$TAG" --repo "$GITHUB_REPOSITORY" --clobber App-*-x86_64.AppImage
```

`gh` is preinstalled on GitHub-hosted runners and `GITHUB_TOKEN` is enough,
so no third-party upload action is needed. (`TheAssassin/pyuploadtool`, run
with `GITHUB_CONTINUOUS_RELEASE_TYPE=prerelease`, does the continuous half
in one line if you prefer it; it has no equivalent for "upload to the
release of this tag, creating it if needed".)

Things worth getting right here, each learned the hard way:

- **The continuous release is a pre-release.** A rolling build is exactly
  the release you don't want a casual visitor to download by clicking the
  repo's "Latest release" badge. `--prerelease` keeps "Latest" pointing at
  an actual tagged release (or nothing, if you haven't cut one yet).
- **Move the tag, don't just replace the assets.** A `continuous` release
  that still points at last month's commit while carrying today's file is
  misleading. `gh release delete --cleanup-tag` plus `gh release create
  --target "$GITHUB_SHA"` moves it; the extra `gh api -X DELETE .../refs/tags`
  covers a tag left behind by a release somebody deleted by hand, which
  would otherwise pin the new release to the old commit.
- **A tag's release may already exist.** A release workflow or a maintainer
  may have created it before the AppImage was built; `gh release view`
  first, then `upload --clobber` onto whatever is there, and only create
  when nothing is. `--verify-tag` refuses to invent a tag from a typo.
- **Filter which tags count.** Monorepos and projects that ship several
  components tag them all in one repository (`tool-v1.2`, `lib-v0.3`, plain
  `33.0.0-beta6`). Without a `case "$TAG"` guard, the AppImage of one
  component lands on another component's release. Match your own tag
  scheme explicitly and `exit 0` otherwise.
- **A `concurrency` group serializes runs per ref**, so a fast double-push
  or a retry cannot race on the same `continuous` tag. Include
  `github.event_name` in the group if the workflow is also reusable (next
  section), so a call from another workflow never cancels a push run of
  the same ref.
- **If a later step needs to know which tag this run just published to**
  (say, to embed a screenshot link in the notes - see
  verify-appimage-portability), derive it from the event, never by asking
  the API for the most recently created release - another release stream
  can make that answer wrong:
  ```sh
  case "$GITHUB_REF" in
    refs/tags/*) TAG="${GITHUB_REF#refs/tags/}" ;;
    *) TAG="continuous" ;;
  esac
  ```

### Update information (zsync) follows the stream, not the file

appimagetool's `-u`/`--updateinformation` embeds where *this* AppImage should
look for its successor and writes the matching `.zsync` next to the output
(install the `zsync` package: it needs `zsyncmake`). AppImageUpdate then
fetches the next build as a delta instead of a full download. Embed it only
when the build is going to be published where the string says:

```sh
update=()
if [ "$GITHUB_EVENT_NAME" = push ] && [ "$GITHUB_REF" = refs/heads/main ]; then
  update=(-u "gh-releases-zsync|${GITHUB_REPOSITORY%/*}|${GITHUB_REPOSITORY#*/}|continuous|App-*-x86_64.AppImage.zsync")
fi
appimagetool --no-appstream "${update[@]}" AppDir "App-$VERSION-x86_64.AppImage"
```

- A PR or manual build gets none: it will never be on a release, and an
  AppImage that points at `continuous` would "update" its user from a test
  build onto master.
- A tagged release's update string can use `latest`
  (`gh-releases-zsync|owner|repo|latest|App-*-x86_64.AppImage.zsync`) **only
  if your tagged releases are real releases.** GitHub's `latest` ignores
  pre-releases, so a project whose tags are all betas would send its users
  to whatever non-pre-release happens to exist - possibly another
  component's. Leave update information off such builds rather than point
  it somewhere wrong.
- The glob in the string must match the file name you upload, version
  included (`App-*-x86_64.AppImage.zsync`), and the artifact upload must
  include the `.zsync` (`path:` with both patterns; `if-no-files-found:
  error` only fires when *nothing* matched, so a build without a `.zsync`
  still uploads fine).

## A tag created with GITHUB_TOKEN never starts a workflow

A release workflow that *creates* its own tag (`gh release create
--target "$SHA"` on a schedule, say a daily beta) does not trigger your
`on: push: tags:` AppImage workflow - GitHub deliberately never starts
workflows from events caused by `GITHUB_TOKEN`. The symptom is a release
with every platform's package except the AppImage, and nothing in the
Actions tab explaining why. Neither does the `release:` event help; it is
suppressed for the same reason.

Make the AppImage workflow reusable and have the release workflow call it:

```yaml
# appimage.yaml
on:
  push: { branches: [main], tags: ['*'] }
  pull_request:
  workflow_call:
    inputs:
      ref: { type: string, required: false, default: '' }
jobs:
  build:
    steps:
      - uses: actions/checkout@<sha>
        with:
          ref: ${{ inputs.ref }}   # empty outside workflow_call: the event's own commit
```

```yaml
# release.yaml
jobs:
  check:   # decides tag + sha
  appimage:
    needs: check
    uses: ./.github/workflows/appimage.yaml
    with: { ref: "${{ needs.check.outputs.sha }}" }
    permissions: { contents: read }
  release:
    needs: [check, build, appimage]
    if: always() && needs.check.result == 'success'
    steps:
      - uses: actions/download-artifact@<sha>
        with: { name: appimage, path: appimage }
        continue-on-error: true    # a missing AppImage is a platform that did not build
      - run: |
          for f in appimage/App-*-x86_64.AppImage; do
            [ -f "$f" ] && mv "$f" "dist/App-$VERSION-linux-x86_64.AppImage"
          done
          # ... sha256sum dist/*, notes, gh release create --target "$SHA" dist/*
```

Details that matter:

- In the called workflow `github.event_name` and `github.ref` are the
  **caller's** (`schedule`, `refs/heads/main`), so the called workflow's own
  `release` job (`if: github.event_name == 'push'`) correctly stays out of
  the way, and its zsync condition correctly embeds nothing. The caller
  publishes.
- An empty `ref:` input makes `actions/checkout` fall back to the event's
  own commit (for a `pull_request` that is the merge ref), so one checkout
  step serves both modes.
- Rename the artifact to the release's own naming scheme at upload time,
  and put it through the same `SHA256SUMS` as the other packages; the
  AppImage's file name is free.
- The artifacts of a called workflow live in the caller's run, so
  `download-artifact` by name works without `run-id` juggling.

## Harden the workflow itself

A few general GitHub Actions habits worth applying here, independent of
anything AppImage-specific:

- **Least-privilege `permissions`.** Set `contents: read` at the workflow
  level, and grant `contents: write` only on the specific job that creates
  or edits a release (`permissions:` under that one job, not the whole
  workflow) - a build or verification job has no business holding write
  access to the repo just because a later job in the same file needs it.
- **`persist-credentials: false` on every `actions/checkout`** that doesn't
  itself run `git push`. Nothing here does (releases go through the GitHub
  API via `gh`/`pyuploadtool` with an explicit token, not a git push), so
  there's no reason for the checkout to leave a credential helper configured
  at all.
- **Pin `actions/*` steps to a commit SHA**, not a floating major-version tag
  like `@v4` - `uses: actions/checkout@<sha> # v4.4.0`. Look the SHA up
  rather than guessing or reusing one from memory (`gh api
  repos/actions/checkout/tags --jq '.[] | select(.name=="v4.4.0") |
  .commit.sha'`); a wrong or stale SHA is worse than an unpinned tag.
- **Pin `appimagetool` to a tagged release and verify it.** It has had
  versioned releases for a while (`1.9.1` at the time of writing), and a
  reviewer *will* ask why your release pipeline downloads an unverified
  rolling binary and runs it on everything you ship. Keep version and
  checksum together and check before executing:
  ```yaml
  env:
    APPIMAGETOOL_VERSION: '1.9.1'
    APPIMAGETOOL_SHA256: <sha256 of .../releases/download/<tag>/appimagetool-x86_64.AppImage>
  run: |
    curl -fsSLo appimagetool "https://github.com/AppImage/appimagetool/releases/download/${APPIMAGETOOL_VERSION}/appimagetool-x86_64.AppImage"
    echo "${APPIMAGETOOL_SHA256}  appimagetool" | sha256sum -c -
    chmod +x appimagetool && ./appimagetool --appimage-extract >/dev/null
  ```
  and say in a README how to bump the pair. The exception stays for tools
  that really are only distributed as a rolling "continuous" build
  (`linuxdeploy`, `linuxdeploy-plugin-qt`, `pyuploadtool`): there's nothing
  stable to pin there, and hash-freezing today's continuous build isn't a
  security improvement, just an unannounced version freeze.
- **A plain `VAR=$(cmd)` assignment can make your own error handling
  unreachable.** GitHub Actions runs `run:` steps under `bash -e` by
  default. Under `set -e`, `VAR=$(some_command)` aborts the whole step
  immediately if `some_command` fails - including skipping whatever
  fallback logic you wrote to run right after it:
  ```sh
  # Looks reasonable, but if `identify` fails, the step exits right here -
  # the next two lines checking $? never execute.
  OUT=$(identify -format "%[fx:standard_deviation]" screenshot.png 2>&1)
  RC=$?
  if [ "$RC" -ne 0 ]; then ...
  ```
  A command's exit status is only exempt from `set -e` when it's the
  condition of an `if`/`while`/`until`, or part of an `&&`/`||` chain - so
  put the assignment there instead:
  ```sh
  if OUT=$(identify -format "%[fx:standard_deviation]" screenshot.png 2>&1); then
    STDDEV="$OUT"
  else
    echo "identify failed: $OUT" >&2
    STDDEV=0
  fi
  ```
  This is worth auditing for specifically in any CI script with a "try this,
  fall back on failure" pattern - the fallback can sit there for months,
  looking correct, without ever once actually running.

## The fork-and-push refspec trap

This pattern - clone an upstream project, add a CI workflow, push the result
to a *different* remote you control (a private mirror, a packaging fork) -
almost always leaves you with two local branches both plausibly named
`main`: the one `git clone` created, tracking the *upstream* `origin/main`,
and a new one you made for your own commits. If you then run:

```sh
git push origin main
```

Git resolves `main` to whichever **local** branch is literally named that -
which may be the untouched upstream-tracking one, not your work. Best case
this is rejected as non-fast-forward against the new origin; worst case it
silently pushes unrelated history. Always push a fork-and-add-CI branch with
an explicit refspec:

```sh
git push origin my-work-branch:main
```

and rename the leftover upstream-tracking local branch (`git branch -m main
upstream-main`) so the ambiguity can't recur later in the same session.

## Run the watch-fix-push loop in small subagents, not your main context

Getting an AppImage workflow green is rarely one push. It's push, watch the
run, pull the failing step's log, make a small fix, push again - and each
round trip drags a chunk of CI log into whatever context is driving it. Doing
that inline in your main conversation burns tokens fast and leaves the
transcript full of build noise nobody needs to re-read later.

Where it makes sense (a genuinely iterative fix-and-repush loop, not a single
one-line change), delegate it:

- Spawn a subagent with `git` + `gh` access, and hand it: the workflow file
  path, which job(s) it's allowed to touch (keep it scoped - don't let a
  subagent fixing the screenshot job also start editing the build job), the
  run id to watch, your commit-message conventions, and a bounded attempt
  budget (e.g. "stop after 5 fix-and-repush attempts and report back" -
  without a budget, a subagent can loop indefinitely on a problem that needs
  a human decision).
- Use the smallest model that can do the job. A subagent that only watches a
  run and reports the failing step/error back to you can run on a cheap
  model - there's no judgment call in `gh run watch` and grepping a log. A
  subagent that also has to diagnose the error and edit the workflow needs a
  more capable model, but still not necessarily your biggest one. Never
  default to the largest available model out of caution alone.
- Have it report back concisely - final status, what changed, the key error
  if it's still stuck - not the full log it read to get there. That's the
  whole point: the noisy part stays in the subagent's own context.
- If you have two independent things to get green at once (say, the build
  job and a separate verification job that only needs the build's artifact),
  run them as separate subagents in parallel rather than babysitting one
  loop to completion before starting the next.
