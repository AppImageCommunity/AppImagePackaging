# AppImage Packaging

**AppImage Packaging** is a Claude plugin that turns a Linux desktop application
into a single self-contained [AppImage](https://appimage.org) file and keeps it
that way in CI. Its five skills cover assembling the AppDir (every library down
to glibc, GTK and Qt runtime data, icons, loaders), building and publishing the
AppImage from GitHub Actions (continuous and tagged releases, zsync updates),
verifying that it runs on distributions it was not built on, and troubleshooting
an AppImage that fails only outside the build machine, and getting it listed
in the AppImageHub catalog. Everything in it was
distilled from real pipelines, with the failures and their fixes written down.

Ask Claude things like:

- "Make an AppImage of this project that runs on any Linux distribution."
- "Add an AppImage to our GitHub release workflow, with a continuous build."
- "The AppImage starts on Ubuntu but crashes on Fedora. Why?"
- "Prove in CI that the AppImage runs on a stock Debian container."
- "Our app runs its helpers in bubblewrap; previews are blank from the AppImage."
- "Submit our AppImage to AppImageHub." or "The AppImageHub test failed with error-glibc-too-new."

## Install

Claude Code (one step, v2.1.275 or later):

```
/plugin install appimage-packaging --marketplace AppImageCommunity/AppImagePackaging
```

or from a shell: `claude plugin marketplace add AppImageCommunity/AppImagePackaging`,
then `claude plugin install appimage-packaging@appimage-packaging`. The skills load as
`appimage-packaging:make-standalone-appdir` and so on; `claude plugin update
appimage-packaging@appimage-packaging` fetches a new version.

claude.ai, Cowork: add it from [Anthropic's directory](https://claude.ai/directory)
once listed, or upload the skill folders under `skills/` one `.zip` at a time
(Settings > Capabilities > Skills > Upload skill; paid plans).

Other agents (Codex, Cursor, OpenCode, ...), through the
[skills CLI](https://github.com/vercel-labs/skills):

```
npx skills add AppImageCommunity/AppImagePackaging
```

## The skills

- [make-standalone-appdir](skills/make-standalone-appdir/SKILL.md): the
  low-level AppDir and library-bundling mechanics: `loaders.cache`, icon themes,
  GLib schemas, GTK modules, bundling the dynamic linker, keeping AppRun's
  environment out of the host programs the app launches, apps that sandbox
  their own helpers. The foundation the other skills build on.
- [build-appimage-ci](skills/build-appimage-ci/SKILL.md): the GitHub Actions
  workflow: build environment, linuxdeploy versus manual assembly, packaging an
  existing release build, the FUSE-less-runner workaround, publishing to GitHub
  Releases (continuous and tagged, zsync update information, tags created by
  `GITHUB_TOKEN`), hardening.
- [verify-appimage-portability](skills/verify-appimage-portability/SKILL.md):
  proving the AppImage runs elsewhere: headless start in a minimal glibc
  container (the release gate) and a stricter Alpine one, what the process
  really mapped, screenshots as CI proof in the release notes.
- [troubleshoot-appimage-runtime](skills/troubleshoot-appimage-runtime/SKILL.md):
  an AppImage that fails, crashes or hangs only outside the build machine:
  dynamic-linker errors, strace ENOENT tracing, toolkit debug variables,
  hardcoded paths, glibc symbol versions on musl.
- [publish-to-appimagehub](skills/publish-to-appimagehub/SKILL.md): listing
  the AppImage in the AppImageHub catalog (appimage.github.io): what its
  automated test does and requires (SquashFS, oldest Ubuntu LTS, X11, offline,
  30 seconds to a window, `desktop-file-validate`, `appstreamcli`, light theme,
  English), naming the entry, channels, a pre-submission check for CI, and
  what each `error-*` label means.

The skills cross-reference each other and are plain Markdown; nothing here runs
code on its own. Claude reads a skill when a task matches its description and
follows its instructions with the tools it already has.

## Layout

```
.claude-plugin/plugin.json       plugin manifest (name, version, license)
.claude-plugin/marketplace.json  lists this repository as marketplace "appimage-packaging"
skills/<name>/SKILL.md           one skill per folder, with references/ for detail
```

Releases: bump `version` in `plugin.json`; the directory and `claude plugin
update` pick it up from `main`.

## License

MIT, see [LICENSE](LICENSE).
