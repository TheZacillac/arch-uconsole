# Changelog

All notable changes documented here. Format follows Keep a Changelog.

## [Unreleased]

## [0.1.0] - 2026-04-15

### Added
- Initial repository scaffolding (LICENSE, README, `.gitignore`).
- Project GPG signing key (fingerprint
  `4B13F2558FF8B2D34E20F740A1A18BAF0DC6E364`), public half committed
  at `keys/uconsole-signing.asc`.
- `uconsole-hello` stub PKGBUILD for pipeline smoke-testing.
- GitHub Actions workflows: `lint` (shellcheck + namcap),
  `build-packages` (makepkg + sign on x86_64 running the official
  archlinux container, with agent-primed loopback pinentry), and
  `build-repo` (repo-add + GitHub Pages deploy).
- Public pacman repo at `https://thezacillac.github.io/arch-uconsole/aarch64/`
  with signed `uconsole.db` and package artifacts.
- `bootstrap/uconsole-bootstrap` script: registers the `[uconsole]` repo
  and locally-signs the project key on an existing Arch Linux ARM
  system.
- Maintainer documentation (`docs/MAINTAINERS.md`) covering signing-key
  generation, rotation, and CI secret handling.
- Design spec (`docs/superpowers/specs/2026-04-15-arch-uconsole-design.md`)
  and Plan 1 implementation plan
  (`docs/superpowers/plans/2026-04-15-foundation.md`).
