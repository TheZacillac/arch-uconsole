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
    Server = https://TheZacillac.github.io/arch-uconsole/$arch

Then import the signing key:

    sudo pacman-key --add /path/to/uconsole-signing.asc
    sudo pacman-key --lsign-key 4B13F2558FF8B2D34E20F740A1A18BAF0DC6E364
    sudo pacman -Sy

## License

MIT for project code. Kernel builds we distribute are GPL-2.0 (inherited
from Linux). See `LICENSE`.
