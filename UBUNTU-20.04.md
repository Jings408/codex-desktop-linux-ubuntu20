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
| `scripts/lib/package-common.sh` | Adds `normalize_upstream_deb_depends()`, which relaxes `libc6 (>= ...)` to `libc6`, maps `libgdk-pixbuf-2.0-0 (>= ...)` to `libgdk-pixbuf-2.0-0 \| libgdk-pixbuf2.0-0`, maps the tss2 entries to Focal's `libtss2-esys0`, and removes the unsatisfiable `libssl3`. |
| `scripts/build-deb.sh` | Applies the normalization to the upstream `Depends` field and drops `nodejs` from the updater dependencies. |
| `packaging/linux/codex-desktop.spec` | Drops `nodejs` from the RPM `Requires`. |
| `packaging/linux/PKGBUILD.template` | Drops `nodejs` from the pacman `depends`. |

All other upstream dependencies, the official Electron runtime, native modules,
bundled `codex`/`rg`, plugins, libraries, locales, and Owl metadata are
preserved unchanged. With no ASAR-changing feature enabled and no required core
patch, `resources/app.asar` remains byte-for-byte identical to the official
package. The current upstream revision requires the core patch
`quit-confirmation-focus`, so the built bundle differs from the official one by
that patch only.

## Focal fixes retained after upstream synchronization

The repository was synchronized with upstream `main` on 2026-09-04, again on
2026-09-15, again on 2026-09-23, and again on 2026-09-29, and the
Focal-specific changes were replayed on top of the current upstream code. The
following distinction is intentional:

* The package fix is in this repository. It normalizes the official dependency
  metadata only in the generated native package; it does not modify the signed
  upstream payload or `resources/app.asar`.
* Official package `26.917.61114` added `libssl3`, `libtss2-esys-3.0.2-0`,
  `libtss2-mu0 | libtss2-mu-4.0.1-0t64`, and `libtss2-tcti-device0`. Focal has
  no `libssl3`, so that entry is removed; the three tss2 entries are mapped to
  Focal's `libtss2-esys0`, which provides the same
  `libtss2-{esys,mu,tcti-device}.so.0` sonames. The only payload consumer is
  `resources/native/remote-control-device-key.node`, a lazily loaded addon that
  also requires `libcrypto.so.3` and therefore cannot load on Focal; the rest of
  the application is unaffected.
* The `git add --sparse` failure is a host-toolchain issue, not an ASAR patch.
  Git 2.25.1 on Ubuntu 20.04 does not understand `--sparse`; upgrading Git to
  2.34 or newer prevents the failed child process and the follow-up Electron
  `write EPIPE` crash.
* After an old EPIPE crash, fully exit the application and its background
  process before relaunching. A partial Electron process can otherwise retain
  the broken state.

The current upstream package pin is `26.924.50649`. Keep the Focal dependency
normalization when syncing future upstream commits; a plain fast-forward is not
possible because this fork also removes upstream-only CI files.

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
cd ~/codex-desktop-linux
make install-native
```

`make install-native` resolves the current official package through the signed
stable metadata, rebuilds `codex-app/`, creates the native `.deb`, and installs
it. If build dependencies are missing, use `make bootstrap-native` once first.
`make install` alone only installs an already-built artifact from `dist/`; it
does not fetch or rebuild a newer upstream package.

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
git fetch upstream main
git rebase upstream/main        # or: git merge upstream/main
make install-native
```

Create a backup branch before a future rebase if you want an easy rollback:

```bash
git branch codex/pre-upstream-sync-$(date +%Y%m%d)
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

The Git/EPIPE prerequisite can be checked independently of the build:

```bash
git --version                         # Git 2.34 or newer
git add -h 2>&1 | grep -- --sparse   # must print --sparse
```

Upstream's own instructions (features, AppImage, Nix, uninstall, and
troubleshooting) continue to apply; see
[the upstream README](https://github.com/ilysenko/codex-desktop-linux).
