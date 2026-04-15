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
`https://TheZacillac.github.io/arch-uconsole/aarch64/`. The bootstrap
script hardcodes this URL + the key fingerprint.

## Known gotchas

### Pacman Landlock sandbox in containers

Recent `pacman` (>= 7.0) applies a Landlock-based filesystem sandbox to
its download phase, running as the `alpm` user. That sandbox cannot be
established in most rootless Docker / Podman containers — `pacman -Sy`
errors with:

    error: restricting filesystem access failed because the Landlock
    ruleset could not be applied: Operation not permitted
    error: switching to sandbox user 'alpm' failed!

Two workarounds (either works):

    # Per invocation
    pacman -Sy --disable-sandbox

    # Persistent for the container session
    sed -i 's|^DownloadUser|#DownloadUser|' /etc/pacman.conf

This is a container-runtime limitation, not a repo defect. The
`uconsole-bootstrap` script is intended to run on a real Arch Linux ARM
install (TTY or chroot), where the sandbox functions correctly.
