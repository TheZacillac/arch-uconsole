# arch-uconsole — Plan 1: Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the repository scaffolding, GPG signing, CI pipelines, and a public `[uconsole]` pacman repo so a stub package can be installed on a vanilla Arch ARM system with signature verification.

**Architecture:** Monorepo at `/home/zac/Projects/arch_build` on `main`. GitHub Actions (native aarch64 runners) build and sign PKGBUILDs; `repo-add` assembles `uconsole.db`; GitHub Pages publishes the repo at `https://<GITHUB_OWNER>.github.io/arch-uconsole/aarch64/`. A single GPG signing key (private in Actions secrets, public committed in-repo) signs both individual packages and the repo database.

**Tech Stack:** bash / POSIX sh, makepkg + repo-add (pacman tooling), GitHub Actions, GPG (GnuPG), shellcheck, namcap.

---

## Reference: File Structure After This Plan

```
arch_build/
├── .github/
│   └── workflows/
│       ├── lint.yml
│       ├── build-packages.yml
│       └── build-repo.yml
├── .gitignore
├── CHANGELOG.md
├── LICENSE
├── README.md
├── bootstrap/
│   └── uconsole-bootstrap
├── docs/
│   ├── MAINTAINERS.md
│   └── superpowers/
│       ├── plans/2026-04-15-foundation.md   (this file)
│       └── specs/2026-04-15-arch-uconsole-design.md
├── keys/
│   └── uconsole-signing.asc                 (public key, committed)
└── packages/
    └── uconsole-hello/
        └── PKGBUILD
```

Each file has a single clear responsibility:
- Workflow YAMLs: one workflow = one pipeline stage (lint / build / publish).
- `bootstrap/uconsole-bootstrap`: repo + key trust only in this plan; package install logic lands in Plan 3.
- `packages/uconsole-hello/PKGBUILD`: stub that exercises the build + sign + publish path end-to-end.

---

## Task 0: Choose the GitHub owner handle

The plan uses the placeholder `<GITHUB_OWNER>` (e.g. `zacbarton` or an org like `arch-uconsole`). Substitute it once here, then reuse the same value everywhere it appears in later tasks.

- [ ] **Step 1: Decide and record the owner handle**

Pick a value. Record it in a shell variable you keep for the rest of the plan:

```bash
export GH_OWNER="<your-chosen-owner>"
echo "$GH_OWNER"
```

Every subsequent task that writes a URL or path containing `<GITHUB_OWNER>` must substitute this value literally before committing. (Do not commit the word `<GITHUB_OWNER>` to any tracked file.)

- [ ] **Step 2: Verify you own or can create the repo**

Either create a new empty GitHub repo at `https://github.com/$GH_OWNER/arch-uconsole` (no README, no license — we'll push our own), OR verify you have permissions to push to an existing empty repo.

---

## Task 1: Repository scaffolding (LICENSE, README, .gitignore, CHANGELOG)

**Files:**
- Create: `LICENSE`
- Create: `README.md`
- Create: `.gitignore`
- Create: `CHANGELOG.md`

- [ ] **Step 1: Write `LICENSE` (MIT)**

Create `LICENSE` with the standard MIT template:

```
MIT License

Copyright (c) 2026 arch-uconsole contributors

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

- [ ] **Step 2: Write `README.md`**

Create `README.md`:

```markdown
# arch-uconsole

Arch Linux ARM for the ClockworkPi uConsole, targeting both CM4 and CM5
compute modules.

**Status:** Pre-release. See `docs/superpowers/specs/` for the full design.

## Repository layout

- `packages/` — PKGBUILDs published to the `[uconsole]` pacman repo.
- `bootstrap/` — `uconsole-bootstrap` script that adds our repo to an
  existing Arch Linux ARM install.
- `image-builder/` — produces flashable `.img.xz` artifacts for CM4 / CM5
  (arrives in Plan 3).
- `keys/` — public signing key. Private key lives only in CI secrets.
- `docs/` — specs, plans, maintainer documentation.

## Using the `[uconsole]` pacman repo

Add to `/etc/pacman.conf`:

    [uconsole]
    SigLevel = Required DatabaseOptional
    Server = https://<GITHUB_OWNER>.github.io/arch-uconsole/$arch

Then import the signing key:

    sudo pacman-key --add /path/to/uconsole-signing.asc
    sudo pacman-key --lsign-key <KEY-FINGERPRINT>
    sudo pacman -Sy

## License

MIT for project code. Kernel builds we distribute are GPL-2.0 (inherited
from Linux). See `LICENSE`.
```

(Replace `<GITHUB_OWNER>` with `$GH_OWNER` from Task 0 before committing.)

- [ ] **Step 3: Write `.gitignore`**

Create `.gitignore`:

```
# Build artifacts
*.pkg.tar.zst
*.pkg.tar.zst.sig
*.tar.gz
*.tar.xz
src/
pkg/

# Image builder outputs
image-builder/build/
image-builder/out/
image-builder/cache/

# Editor cruft
.vscode/
.idea/
*.swp
*~
.DS_Store

# GPG secret exports (should never be committed)
*.gpg
*.asc.private
private-key*
```

- [ ] **Step 4: Write `CHANGELOG.md`**

Create `CHANGELOG.md`:

```markdown
# Changelog

All notable changes documented here. Format follows Keep a Changelog.

## [Unreleased]

### Added
- Initial repository scaffolding, signing key, pacman repo publishing, and
  a stub `uconsole-hello` package to exercise the pipeline end-to-end.
```

- [ ] **Step 5: Commit**

```bash
git add LICENSE README.md .gitignore CHANGELOG.md
git commit -m "chore: initial repo scaffolding (license, README, gitignore, changelog)"
```

---

## Task 2: Maintainer documentation — signing key workflow

**Files:**
- Create: `docs/MAINTAINERS.md`

- [ ] **Step 1: Write `docs/MAINTAINERS.md`**

Create `docs/MAINTAINERS.md`:

```markdown
# Maintainer Guide

## Signing key

The `[uconsole]` pacman repo is signed with a single project GPG key.
The public key is committed at `keys/uconsole-signing.asc`. The private
key is stored ONLY as GitHub Actions secrets.

### One-time setup (new project / key rotation)

1. Generate on a trusted machine (not CI):

   ```bash
   gpg --quick-gen-key "arch-uconsole signing <noreply@example.org>" ed25519 sign 2y
   ```

2. Record the fingerprint:

   ```bash
   gpg --list-secret-keys --keyid-format=long
   # Note the 40-character fingerprint for the new key.
   ```

3. Export the public key into the repo:

   ```bash
   gpg --armor --export <FPR> > keys/uconsole-signing.asc
   git add keys/uconsole-signing.asc
   git commit -m "chore: add project signing public key <FPR-short>"
   ```

4. Export the private key for CI:

   ```bash
   gpg --armor --export-secret-keys <FPR> > /tmp/uconsole-signing.private.asc
   ```

5. Store three values as GitHub Actions repository secrets:
   - `UCONSOLE_SIGNING_KEY_ASC` — contents of
     `/tmp/uconsole-signing.private.asc`
   - `UCONSOLE_SIGNING_KEY_PASSPHRASE` — the key's passphrase (empty string
     if the key has none; prefer having one)
   - `UCONSOLE_SIGNING_KEY_FPR` — the 40-char fingerprint

6. Delete the private export and lock the on-disk key:

   ```bash
   shred -u /tmp/uconsole-signing.private.asc
   ```

7. Offline backup: export the private key to removable media kept offline.
   This is the only copy beside the CI secret. If both are lost, the key is
   gone forever and users must re-trust a new one.

### Key rotation

Out of scope for v1 docs — revisit when the first key expires.

## Release signing

Tagged releases (`v*`) trigger `build-image.yml` (landing in a later plan).
Image artifacts are signed with the same key. Users verify with
`gpg --verify uconsole-arch-cm5-<date>.img.xz.sig`.

## Repo URL

The published pacman repo lives at
`https://<GITHUB_OWNER>.github.io/arch-uconsole/aarch64/`. The bootstrap
script hardcodes this URL + the key fingerprint.
```

(Substitute `<GITHUB_OWNER>` with `$GH_OWNER`.)

- [ ] **Step 2: Commit**

```bash
git add docs/MAINTAINERS.md
git commit -m "docs: add MAINTAINERS.md with signing key workflow"
```

---

## Task 3: Generate and commit the project signing public key

**Files:**
- Create: `keys/uconsole-signing.asc`

This task has a **human-only step**. Do not automate signing-key generation in CI or leave the private key anywhere on disk after export.

- [ ] **Step 1: Generate the signing key (human, trusted machine)**

```bash
gpg --quick-gen-key "arch-uconsole signing <noreply@$GH_OWNER.github.io>" ed25519 sign 2y
```

When prompted, set a passphrase (don't skip it).

- [ ] **Step 2: Record the fingerprint**

```bash
gpg --list-secret-keys --keyid-format=long
```

Copy the 40-character fingerprint (lines prefixed with the fingerprint after the `sec` entry). Export it for use in later steps:

```bash
export UCONSOLE_FPR="<40-char-fingerprint>"
echo "$UCONSOLE_FPR" | wc -c  # Should output 41 (40 chars + newline)
```

- [ ] **Step 3: Export the public key into the repo**

```bash
mkdir -p keys
gpg --armor --export "$UCONSOLE_FPR" > keys/uconsole-signing.asc
```

Verify:

```bash
head -1 keys/uconsole-signing.asc
# Should print: -----BEGIN PGP PUBLIC KEY BLOCK-----
```

- [ ] **Step 4: Export private key + passphrase into GitHub Actions secrets**

```bash
gpg --armor --export-secret-keys "$UCONSOLE_FPR" > /tmp/uconsole-signing.private.asc
```

Open `https://github.com/$GH_OWNER/arch-uconsole/settings/secrets/actions` and add three repository secrets:

| Name | Value |
|---|---|
| `UCONSOLE_SIGNING_KEY_ASC` | entire contents of `/tmp/uconsole-signing.private.asc` |
| `UCONSOLE_SIGNING_KEY_PASSPHRASE` | the passphrase set in Step 1 |
| `UCONSOLE_SIGNING_KEY_FPR` | contents of `$UCONSOLE_FPR` |

Then shred the export:

```bash
shred -u /tmp/uconsole-signing.private.asc
```

- [ ] **Step 5: Back up the private key offline**

Export one more copy to encrypted removable media (LUKS USB, hardware token, age-encrypted file on offline storage) and physically separate it. This is the only non-CI copy. Delete any remaining on-disk copy afterward.

- [ ] **Step 6: Update README with the real fingerprint**

Edit `README.md`'s pacman usage section — replace `<KEY-FINGERPRINT>` with the actual `$UCONSOLE_FPR`.

```bash
sed -i "s|<KEY-FINGERPRINT>|$UCONSOLE_FPR|g" README.md
```

Verify no placeholder remains:

```bash
grep -n "<KEY-FINGERPRINT>" README.md || echo "ok: no placeholder left"
```

- [ ] **Step 7: Commit**

```bash
git add keys/uconsole-signing.asc README.md
git commit -m "chore: add project signing public key"
```

---

## Task 4: Stub package — `uconsole-hello`

**Files:**
- Create: `packages/uconsole-hello/PKGBUILD`

- [ ] **Step 1: Write `packages/uconsole-hello/PKGBUILD`**

```bash
# Maintainer: arch-uconsole <noreply@example.org>

pkgname=uconsole-hello
pkgver=0.1.0
pkgrel=1
pkgdesc="Stub package for smoke-testing the [uconsole] pacman repo"
arch=('any')
url="https://github.com/<GITHUB_OWNER>/arch-uconsole"
license=('MIT')

package() {
    install -dm755 "${pkgdir}/usr/share/doc/uconsole-hello"
    printf 'hello from uconsole-hello %s\n' "${pkgver}" \
        > "${pkgdir}/usr/share/doc/uconsole-hello/HELLO"
    chmod 644 "${pkgdir}/usr/share/doc/uconsole-hello/HELLO"
}
```

(Substitute `<GITHUB_OWNER>` with `$GH_OWNER`.)

- [ ] **Step 2: Local build smoke test**

On an Arch or Arch-compatible system (if you're not on Arch, skip this step and rely on CI in Task 6):

```bash
cd packages/uconsole-hello
makepkg -f
```

Expected: a `uconsole-hello-0.1.0-1-any.pkg.tar.zst` file appears in the current directory.

Clean it up — CI will produce the real artifact:

```bash
rm -f uconsole-hello-*.pkg.tar.zst
rm -rf pkg/ src/
cd ../..
```

- [ ] **Step 3: Run `namcap` locally (optional, if installed)**

```bash
namcap packages/uconsole-hello/PKGBUILD
```

Expected: no errors. Warnings about missing maintainer email format are acceptable.

- [ ] **Step 4: Commit**

```bash
git add packages/uconsole-hello/PKGBUILD
git commit -m "feat(packages): add uconsole-hello stub for repo smoke-test"
```

---

## Task 5: Lint workflow (`.github/workflows/lint.yml`)

**Files:**
- Create: `.github/workflows/lint.yml`

- [ ] **Step 1: Write the workflow**

```yaml
name: Lint

on:
  pull_request:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  shellcheck:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4

      - name: Install shellcheck
        run: sudo apt-get update && sudo apt-get install -y shellcheck

      - name: Run shellcheck on shell scripts
        run: |
          set -euo pipefail
          files=$(find bootstrap image-builder scripts 2>/dev/null -type f \
            \( -name '*.sh' -o -name 'uconsole-*' \) \
            -not -name '*.md' || true)
          if [ -z "$files" ]; then
            echo "No shell scripts to lint yet; skipping."
            exit 0
          fi
          echo "Linting: $files"
          # shellcheck disable=SC2086
          shellcheck $files

  namcap:
    runs-on: ubuntu-24.04-arm
    container:
      image: archlinux:latest
    steps:
      - uses: actions/checkout@v4

      - name: Install namcap
        run: pacman -Syu --noconfirm namcap

      - name: Run namcap on every PKGBUILD
        run: |
          set -euo pipefail
          shopt -s nullglob
          found=0
          for dir in packages/*/; do
            if [ -f "${dir}PKGBUILD" ]; then
              found=1
              echo "--- ${dir}PKGBUILD"
              namcap "${dir}PKGBUILD"
            fi
          done
          if [ "$found" = 0 ]; then
            echo "No PKGBUILDs found; skipping."
          fi
```

- [ ] **Step 2: Commit and push**

```bash
git add .github/workflows/lint.yml
git commit -m "ci: add lint workflow (shellcheck + namcap)"
git push -u origin main
```

- [ ] **Step 3: Verify the workflow runs and passes**

Open `https://github.com/$GH_OWNER/arch-uconsole/actions/workflows/lint.yml` and wait for the first run triggered by the push. Both `shellcheck` and `namcap` jobs must succeed.

If `namcap` fails on `uconsole-hello/PKGBUILD`, read the warnings — usually they're pedantic (missing license file, etc.) and can be silenced with `# namcap-ignore` comments or by fixing the PKGBUILD. Fix and re-commit before moving on.

---

## Task 6: Package build workflow (`.github/workflows/build-packages.yml`)

**Files:**
- Create: `.github/workflows/build-packages.yml`

- [ ] **Step 1: Write the workflow**

```yaml
name: Build Packages

on:
  push:
    branches: [main]
    paths:
      - 'packages/**'
      - '.github/workflows/build-packages.yml'
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-24.04-arm
    container:
      image: archlinux:latest
      options: --privileged
    steps:
      - uses: actions/checkout@v4

      - name: Install build environment
        run: |
          pacman -Syu --noconfirm base-devel git sudo
          useradd -m -s /bin/bash builder
          echo 'builder ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers
          chown -R builder:builder "$PWD"

      - name: Import signing key
        env:
          KEY_ASC: ${{ secrets.UCONSOLE_SIGNING_KEY_ASC }}
          KEY_FPR: ${{ secrets.UCONSOLE_SIGNING_KEY_FPR }}
        run: |
          mkdir -p /home/builder/.gnupg
          chmod 700 /home/builder/.gnupg
          chown -R builder:builder /home/builder/.gnupg
          echo "$KEY_ASC" | sudo -u builder gpg --batch --import
          echo "${KEY_FPR}:6:" | sudo -u builder gpg --batch --import-ownertrust

      - name: Build and sign every PKGBUILD
        env:
          KEY_FPR: ${{ secrets.UCONSOLE_SIGNING_KEY_FPR }}
          GPG_PASSPHRASE: ${{ secrets.UCONSOLE_SIGNING_KEY_PASSPHRASE }}
        run: |
          set -euo pipefail
          mkdir -p artifacts
          for dir in packages/*/; do
            [ -f "${dir}PKGBUILD" ] || continue
            echo "--- Building ${dir}"
            sudo -u builder bash -c "
              cd '${dir}' && \
              echo '${GPG_PASSPHRASE}' | \
              makepkg -s --noconfirm --sign --key '${KEY_FPR}' \
                --gpg-options '--pinentry-mode loopback --batch --passphrase-fd 0'
            "
            mv "${dir}"*.pkg.tar.zst "${dir}"*.pkg.tar.zst.sig artifacts/
          done
          ls -la artifacts/

      - uses: actions/upload-artifact@v4
        with:
          name: uconsole-packages
          path: artifacts/
          retention-days: 30
```

- [ ] **Step 2: Commit and push**

```bash
git add .github/workflows/build-packages.yml
git commit -m "ci: add package build + sign workflow on aarch64 runners"
git push
```

- [ ] **Step 3: Verify the workflow builds the stub successfully**

Watch `https://github.com/$GH_OWNER/arch-uconsole/actions/workflows/build-packages.yml`. Expected:
- Job runs on `ubuntu-24.04-arm`.
- `uconsole-hello-0.1.0-1-any.pkg.tar.zst` and `.sig` appear in the uploaded `uconsole-packages` artifact.

Download the artifact from the GitHub UI and verify locally:

```bash
unzip uconsole-packages.zip -d /tmp/artifacts
ls /tmp/artifacts/
# Expected: uconsole-hello-0.1.0-1-any.pkg.tar.zst
#           uconsole-hello-0.1.0-1-any.pkg.tar.zst.sig
gpg --verify /tmp/artifacts/uconsole-hello-0.1.0-1-any.pkg.tar.zst.sig \
             /tmp/artifacts/uconsole-hello-0.1.0-1-any.pkg.tar.zst
# Expected: "Good signature from ..."
```

If the signature fails or the pipeline errors, inspect logs (most common causes: passphrase not being piped via `loopback`, or the ownertrust import didn't land). Fix and re-run before continuing.

---

## Task 7: Publish workflow (`.github/workflows/build-repo.yml`)

**Files:**
- Create: `.github/workflows/build-repo.yml`

- [ ] **Step 1: Enable GitHub Pages for the repo**

In the GitHub UI: `Settings → Pages → Source: GitHub Actions`. No branch deploy; the workflow below uses the `actions/deploy-pages` flow.

- [ ] **Step 2: Write the workflow**

```yaml
name: Build & Publish Repo

on:
  workflow_run:
    workflows: ["Build Packages"]
    types: [completed]
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  assemble:
    if: ${{ github.event.workflow_run.conclusion == 'success' || github.event_name == 'workflow_dispatch' }}
    runs-on: ubuntu-24.04-arm
    container:
      image: archlinux:latest
    outputs:
      artifact-name: ${{ steps.package.outputs.name }}
    steps:
      - uses: actions/checkout@v4

      - name: Install tooling
        run: pacman -Syu --noconfirm pacman-contrib git unzip

      - name: Download built packages
        uses: actions/download-artifact@v4
        with:
          name: uconsole-packages
          path: aarch64/
          run-id: ${{ github.event.workflow_run.id }}
          github-token: ${{ secrets.GITHUB_TOKEN }}

      - name: Import signing key
        env:
          KEY_ASC: ${{ secrets.UCONSOLE_SIGNING_KEY_ASC }}
          KEY_FPR: ${{ secrets.UCONSOLE_SIGNING_KEY_FPR }}
        run: |
          echo "$KEY_ASC" | gpg --batch --import
          echo "${KEY_FPR}:6:" | gpg --batch --import-ownertrust

      - name: Assemble repo database
        env:
          KEY_FPR: ${{ secrets.UCONSOLE_SIGNING_KEY_FPR }}
          GPG_PASSPHRASE: ${{ secrets.UCONSOLE_SIGNING_KEY_PASSPHRASE }}
        run: |
          set -euo pipefail
          cd aarch64
          repo-add --sign --key "$KEY_FPR" uconsole.db.tar.gz *.pkg.tar.zst
          ls -la

      - name: Copy public key alongside repo
        run: |
          mkdir -p aarch64
          cp keys/uconsole-signing.asc aarch64/uconsole-signing.asc

      - name: Write landing index.html
        run: |
          cat > index.html <<'HTML'
          <!doctype html>
          <meta charset="utf-8">
          <title>arch-uconsole pacman repo</title>
          <h1>arch-uconsole</h1>
          <p>pacman repo for the <a href="https://github.com/${{ github.repository }}">arch-uconsole</a> project.</p>
          <p>Add to <code>/etc/pacman.conf</code>:</p>
          <pre>[uconsole]
          SigLevel = Required DatabaseOptional
          Server = https://${{ github.repository_owner }}.github.io/arch-uconsole/$arch</pre>
          <p>Public signing key: <a href="aarch64/uconsole-signing.asc">uconsole-signing.asc</a></p>
          HTML

      - name: Stage Pages artifact
        run: |
          mkdir -p _site
          cp -r aarch64 _site/
          cp index.html _site/

      - uses: actions/upload-pages-artifact@v3
        with:
          path: _site/

  deploy:
    needs: assemble
    runs-on: ubuntu-24.04
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

- [ ] **Step 3: Commit and push**

```bash
git add .github/workflows/build-repo.yml
git commit -m "ci: assemble uconsole.db and publish to GitHub Pages"
git push
```

- [ ] **Step 4: Trigger manually (first run) and verify**

Because the workflow triggers on `workflow_run` of "Build Packages" completing, and the previous build-packages run already finished before this workflow existed, trigger it manually from the Actions UI (`Run workflow → main`).

- [ ] **Step 5: Verify the published repo**

After deploy completes:

```bash
curl -fsSL "https://$GH_OWNER.github.io/arch-uconsole/aarch64/uconsole.db" -o /tmp/uconsole.db
curl -fsSL "https://$GH_OWNER.github.io/arch-uconsole/aarch64/uconsole.db.sig" -o /tmp/uconsole.db.sig
curl -fsSL "https://$GH_OWNER.github.io/arch-uconsole/aarch64/uconsole-hello-0.1.0-1-any.pkg.tar.zst" -o /tmp/uconsole-hello.pkg.tar.zst
curl -fsSL "https://$GH_OWNER.github.io/arch-uconsole/aarch64/uconsole-signing.asc" -o /tmp/uconsole-signing.asc

gpg --import /tmp/uconsole-signing.asc
gpg --verify /tmp/uconsole.db.sig /tmp/uconsole.db
# Expected: "Good signature from arch-uconsole signing ..."
```

If any curl 404s, the Pages deployment didn't include that path. Check the `Stage Pages artifact` step logs to confirm the file made it into `_site/`.

---

## Task 8: Bootstrap script scaffold (`bootstrap/uconsole-bootstrap`)

**Files:**
- Create: `bootstrap/uconsole-bootstrap`

This scaffold handles only Plan 1's responsibilities: add the repo, trust the key. Package installation logic arrives in Plan 3.

- [ ] **Step 1: Write the script**

```sh
#!/bin/sh
# uconsole-bootstrap — add the [uconsole] pacman repo and trust the signing key.
#
# Scope (Plan 1 of arch-uconsole): repo registration + key trust only.
# Package installation logic arrives in Plan 3.
#
# Usage (as root):
#   curl -fsSL https://<GITHUB_OWNER>.github.io/arch-uconsole/aarch64/uconsole-signing.asc \
#     | uconsole-bootstrap --trust-key -
#   uconsole-bootstrap --add-repo
#
# Or with everything in one go:
#   uconsole-bootstrap

set -eu

REPO_URL_DEFAULT="https://<GITHUB_OWNER>.github.io/arch-uconsole/\$arch"
KEY_URL_DEFAULT="https://<GITHUB_OWNER>.github.io/arch-uconsole/aarch64/uconsole-signing.asc"
KEY_FPR_DEFAULT="<UCONSOLE_FPR>"
PACMAN_CONF="${PACMAN_CONF:-/etc/pacman.conf}"
SENTINEL_DIR="/var/lib/uconsole"
SENTINEL_FILE="${SENTINEL_DIR}/bootstrap.done"

log() { printf '[uconsole-bootstrap] %s\n' "$*" >&2; }
die() { log "ERROR: $*"; exit 1; }

require_root() {
    [ "$(id -u)" -eq 0 ] || die "must run as root"
}

add_repo() {
    if grep -q '^\[uconsole\]' "$PACMAN_CONF"; then
        log "[uconsole] already present in $PACMAN_CONF; skipping"
        return 0
    fi
    log "appending [uconsole] section to $PACMAN_CONF"
    cat >> "$PACMAN_CONF" <<EOF

[uconsole]
SigLevel = Required DatabaseOptional
Server = ${REPO_URL_DEFAULT}
EOF
}

trust_key() {
    log "fetching signing key from ${KEY_URL_DEFAULT}"
    tmp=$(mktemp)
    trap 'rm -f "$tmp"' EXIT
    curl -fsSL "$KEY_URL_DEFAULT" -o "$tmp"
    pacman-key --add "$tmp"
    pacman-key --lsign-key "$KEY_FPR_DEFAULT"
}

refresh() {
    pacman -Sy
}

finalize() {
    mkdir -p "$SENTINEL_DIR"
    {
        printf 'bootstrap_version=1\n'
        printf 'repo_url=%s\n' "$REPO_URL_DEFAULT"
        printf 'key_fpr=%s\n' "$KEY_FPR_DEFAULT"
        printf 'timestamp=%s\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
    } > "$SENTINEL_FILE"
}

main() {
    require_root
    add_repo
    trust_key
    refresh
    finalize
    log "done. you can now: pacman -S uconsole-hello"
}

main "$@"
```

Substitute `<GITHUB_OWNER>` → `$GH_OWNER`, and `<UCONSOLE_FPR>` → `$UCONSOLE_FPR`:

```bash
sed -i "s|<GITHUB_OWNER>|$GH_OWNER|g; s|<UCONSOLE_FPR>|$UCONSOLE_FPR|g" bootstrap/uconsole-bootstrap
chmod +x bootstrap/uconsole-bootstrap
```

Verify no placeholders remain:

```bash
grep -nE '<GITHUB_OWNER>|<UCONSOLE_FPR>' bootstrap/uconsole-bootstrap \
  || echo "ok: no placeholders left"
```

- [ ] **Step 2: Shellcheck locally**

If `shellcheck` is installed:

```bash
shellcheck bootstrap/uconsole-bootstrap
```

Expected: no errors. If on a system without shellcheck, CI will catch issues at push time.

- [ ] **Step 3: Commit and push**

```bash
git add bootstrap/uconsole-bootstrap
git commit -m "feat(bootstrap): initial uconsole-bootstrap (repo + key trust)"
git push
```

- [ ] **Step 4: Confirm CI lint passes**

Watch the lint workflow run. `shellcheck` job must be green.

---

## Task 9: End-to-end verification on a vanilla Arch aarch64 system

This task requires either:
- A physical aarch64 Arch Linux ARM machine (RPi, other SBC), or
- A QEMU aarch64 VM running ALARM, or
- A podman/docker `archlinux` container with `--platform linux/arm64` (acceptable for this smoke test since we're only checking the repo works — no kernel needed).

Pick whichever is quickest for you.

- [ ] **Step 1: Spin up a clean aarch64 Arch environment**

Example using podman:

```bash
podman run --rm -it --platform linux/arm64 archlinux:latest bash
```

Inside the container:

```bash
pacman -Syu --noconfirm curl
```

- [ ] **Step 2: Run the bootstrap script from the published repo**

```bash
curl -fsSL "https://$GH_OWNER.github.io/arch-uconsole/aarch64/uconsole-signing.asc" \
    -o /tmp/uconsole-signing.asc
pacman-key --init   # required in fresh containers
pacman-key --populate archlinux 2>/dev/null || true
pacman-key --add /tmp/uconsole-signing.asc
pacman-key --lsign-key "$UCONSOLE_FPR"

cat >> /etc/pacman.conf <<EOF

[uconsole]
SigLevel = Required DatabaseOptional
Server = https://$GH_OWNER.github.io/arch-uconsole/\$arch
EOF

pacman -Sy
```

Expected output includes a line for the `uconsole` database with a successful signature check.

- [ ] **Step 3: Install the stub package**

```bash
pacman -S --noconfirm uconsole-hello
cat /usr/share/doc/uconsole-hello/HELLO
# Expected: "hello from uconsole-hello 0.1.0"
```

- [ ] **Step 4: Record the success in CHANGELOG**

Back on your dev machine, update `CHANGELOG.md` — move the items under `[Unreleased]` into a new `[0.1.0] - YYYY-MM-DD` section using today's date:

```markdown
## [0.1.0] - 2026-04-15

### Added
- Initial repository scaffolding, signing key, pacman repo publishing, and
  a stub `uconsole-hello` package to exercise the pipeline end-to-end.

## [Unreleased]
```

- [ ] **Step 5: Commit**

```bash
git add CHANGELOG.md
git commit -m "docs: release 0.1.0 (foundation)"
git tag v0.1.0
git push origin main v0.1.0
```

---

## Done

**Plan 1 is complete when:**

1. `https://<GITHUB_OWNER>.github.io/arch-uconsole/aarch64/uconsole.db` resolves with a valid signature.
2. A clean aarch64 Arch environment can `pacman -S uconsole-hello` from that repo.
3. All three workflows (`lint`, `build-packages`, `build-repo`) are green on `main`.
4. `v0.1.0` is tagged.

**Next:** Plan 2 — Kernel + Firmware + DTB for CM4 (then CM5). Everything needed to produce `linux-uconsole-cm4` as a real, installable kernel package.
