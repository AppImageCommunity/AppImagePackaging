# Multi-arch CI with GitHub Actions

Step 14 of the assembly in SKILL.md, in full: one Alpine container per architecture via docker run --platform, and the release job. Part of the make-standalone-appdir skill.

### 14. Multi-arch CI with GitHub Actions

Use a matrix build with native runners and Docker:

```yaml
strategy:
  fail-fast: false
  matrix:
    include:
      - arch: x86_64
        runner: ubuntu-latest
        platform: linux/amd64
      - arch: i686
        runner: ubuntu-latest
        platform: linux/386
      - arch: aarch64
        runner: ubuntu-24.04-arm
        platform: linux/arm64
      - arch: armhf
        runner: ubuntu-24.04-arm
        platform: linux/arm/v7

runs-on: ${{ matrix.runner }}
steps:
  - name: Build in Alpine container
    run: |
      docker run --rm \
        --platform ${{ matrix.platform }} \
        -v "${{ github.workspace }}:/workspace" \
        -w /workspace \
        alpine:3.20 sh -c '...'
```

**No QEMU needed:** x86_64 runs i686 natively, aarch64 runs armhf natively.
Docker's `--platform` flag selects the right image variant without emulation.

**File permissions:** Docker creates files as root. Run `chmod -R a+rwx AppDir`
inside the Docker container before it exits.

**Separate upload job:** Use a separate job after all builds to create one
release with all architectures:

```yaml
upload:
  needs: build
  runs-on: ubuntu-latest
  steps:
    - uses: actions/download-artifact@v4
    - name: Create release
      run: |
        wget -q https://github.com/TheAssassin/pyuploadtool/releases/download/continuous/pyuploadtool-x86_64.AppImage
        chmod +x pyuploadtool-x86_64.AppImage
        GITHUB_RELEASE_NAME="$VERSION" GITHUB_CONTINUOUS_RELEASE_TYPE=prerelease \
            ./pyuploadtool-x86_64.AppImage **/App*.AppImage*
```
