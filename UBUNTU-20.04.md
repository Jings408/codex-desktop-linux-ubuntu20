# Ubuntu 20.04 (Focal) support

This fork of
[`ilysenko/codex-desktop-linux`](https://github.com/ilysenko/codex-desktop-linux)
builds and installs the **ChatGPT Community for Linux** package on Ubuntu 20.04
(Focal).

OpenAI's official Linux app is only supported on Ubuntu 24.04+, Debian 13+,
Fedora 43/44, and similar newer distributions. Its signed `.deb` declares
runtime dependencies that Focal (glibc 2.31) cannot satisfy — even though the
core application runs fine. This fork relaxes those over-strict dependencies
for the native `.deb`/RPM/pacman packages while keeping the official payload
byte-for-byte intact.

## Why Focal normally fails

The official package's `Depends` field contains three problems for Focal:

1. **`libc6 (>= 2.35)`** — Focal ships glibc 2.31. The constraint comes from the
   bundled computer-use (`sky`) helper binary; the Electron/ChatGPT runtime
   itself only needs glibc 2.25.
2. **`libgdk-pixbuf-2.0-0 (>= 2.36.9)`** — that package name is the Ubuntu
   22.04+ naming. Focal names the same library `libgdk-pixbuf2.0-0` (no hyphen
   before `2.0`), so the dependency is unsatisfiable as written.
3. **`nodejs`** — a hard dependency, even though the package bundles its own
   managed Node.js runtime.

There is a second, unrelated launch problem that is common on Focal: the app
invokes `git add --sparse` internally (for turn-diff capture), which requires
Git >= 2.34. Focal ships Git 2.25.1, so the internal capture fails and
eventually crashes the Electron main process (`Error: write EPIPE`), breaking
the chat view. This is fixed by installing a newer Git; it is not a packaging
change.

## What this fork changes

| File | Change |
| --- | --- |
| `scripts/lib/package-common.sh` | Adds `normalize_upstream_deb_depends()`, which relaxes `libc6 (>= ...)` to `libc6` and `libgdk-pixbuf-2.0-0 (>= ...)` to `libgdk-pixbuf-2.0-0 \| libgdk-pixbuf2.0-0`. |
| `scripts/build-deb.sh` | Applies the normalization to the upstream `Depends` field and drops `nodejs` from the updater dependencies. |
| `packaging/linux/codex-desktop.spec` | Drops `nodejs` from the RPM `Requires`. |
| `packaging/linux/PKGBUILD.template` | Drops `nodejs` from the pacman `depends`. |

All other upstream dependencies, the official Electron runtime, native modules,
bundled `codex`/`rg`, plugins, libraries, locales, and Owl metadata are
preserved unchanged. With no ASAR-changing feature enabled, `resources/app.asar`
remains byte-for-byte identical to the official package.

## Install on Ubuntu 20.04

### 1. Install a recent Git (>= 2.34)

Required to avoid the `git add --sparse` / `write EPIPE` crash:

```bash
sudo add-apt-repository ppa:git-core/ppa
sudo apt update
sudo apt install git

git --version          # should be >= 2.34
git add -h 2>&1 | grep -i sparse
```

### 2. Build and install

```bash
git clone https://github.com/<your-user>/codex-desktop-linux.git
cd codex-desktop-linux
make bootstrap-native
```

`make bootstrap-native` installs build dependencies first, then builds
`codex-app/`, creates the native `.deb`, and installs it. If the dependencies
are already present, use `make install-native` instead.

Build dependencies: Node.js 20+, npm, Python 3, curl, `gpgv`, `dpkg-deb`, tar,
`make`, and a C/C++ toolchain. Rust is required only for the updater and enabled
native feature helpers; if you do not want Rust, build without the updater:

```bash
PACKAGE_WITH_UPDATER=0 make install-native
```

### 3. Verify the relaxed dependencies

```bash
make deb
dpkg-deb -f dist/codex-desktop_*_amd64.deb Depends
```

The `Depends` line must reference plain `libc6` (no `>= 2.35`),
`libgdk-pixbuf-2.0-0 | libgdk-pixbuf2.0-0` (no `>= 2.36.9`), and no `nodejs`.

### 4. Restart cleanly

After installing, fully quit the application (including its background process),
then relaunch. A previous `EPIPE` crash leaves the Electron main process in a
partially broken state that only a full restart clears.

## Keeping up with the official repository

This fork tracks upstream. To pull the latest official changes and re-apply this
fork's dependency relaxation on top of them:

```bash
git remote add upstream https://github.com/ilysenko/codex-desktop-linux.git
git fetch upstream
git rebase upstream/main        # or: git merge upstream/main
```

The dependency relaxation is confined to these files:

- `scripts/lib/package-common.sh` (`normalize_upstream_deb_depends`)
- `scripts/build-deb.sh`
- `packaging/linux/codex-desktop.spec`
- `packaging/linux/PKGBUILD.template`
- `scripts/lib/package-common.test.js`

If upstream rewrites one of these files, the rebase may conflict. Resolve the
conflict by keeping the `normalize_upstream_deb_depends` call site, the removed
`nodejs` entries, and the updated tests, then verify:

```bash
bash -n install.sh scripts/lib/*.sh launcher/start.sh.template
node --test scripts/lib/package-common.test.js
make deb
dpkg-deb -f dist/codex-desktop_*_amd64.deb Depends
```

Upstream's own instructions (features, AppImage, Nix, uninstall, and
troubleshooting) continue to apply; see
[the upstream README](https://github.com/ilysenko/codex-desktop-linux).