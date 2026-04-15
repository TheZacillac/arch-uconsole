# arch-uconsole — Design Spec

**Date:** 2026-04-15
**Status:** Approved (brainstorm complete, ready for implementation planning)

## Summary

`arch-uconsole` is a project that brings Arch Linux ARM to the ClockworkPi
uConsole handheld for both the CM4 and CM5 compute modules. It ships three
layered deliverables: a custom pacman repo of PKGBUILDs, a bootstrap script
that turns a vanilla ALARM install into a uConsole-ready system, and an
image builder that produces flashable `.img.xz` artifacts for each compute
module.

The default userspace is a Hyprland session with Ly as the display manager,
tuned for the uConsole's 720×1280 landscape display and hardware hotkeys.

## Goals

1. Boot a flashed SD card on a CM4 or CM5 uConsole into a working Arch Linux
   system with all core peripherals functional (display, keyboard, audio,
   WiFi, Bluetooth, battery).
2. First-class support for expansion modules (4G/LTE, LoRa) with opt-in
   metapackage.
3. `pacman -Syu` works against our signed, publicly-hosted repo.
4. An existing ALARM user can uconsole-ify their system with a single
   bootstrap script.
5. `uconsole-arch-test` lets any user verify their install and report bugs
   with a structured pass/fail report.

## Non-Goals (v1)

- Support for compute modules other than CM4 / CM5.
- Support for ClockworkPi DevTerm (different form factor).
- Opinionated mobile shells (Phosh, Plasma Mobile). Users install those
  themselves if they want them.
- Dual-boot / multi-OS tooling.
- Automated image-to-image upgrades. Package updates via `pacman -Syu` are
  supported; major image revisions require reflashing.
- Self-hosted hardware-in-the-loop CI.

## Architecture

Three layers, each independently useful:

```
┌──────────────────────────────────────────────────────────────┐
│ Layer 3: Image Builder                                       │
│   mkimage.sh — pacstrap + qemu-aarch64-static chroot         │
│   Produces: uconsole-arch-cm4.img.xz, uconsole-arch-cm5.img.xz│
└──────────────────────────────────────────────────────────────┘
                            │ consumes
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ Layer 2: Bootstrap Script                                    │
│   uconsole-bootstrap — runs in chroot or on live ALARM       │
│   Adds [uconsole] repo, installs metapackage, applies tweaks │
└──────────────────────────────────────────────────────────────┘
                            │ consumes
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ Layer 1: PKGBUILDs + pacman repo                             │
│   linux-uconsole-cm{4,5}, uconsole-firmware, uconsole-dtbs,  │
│   uconsole-keyboard, uconsole-hyprland-config, uconsole-*    │
│   Hosted: GitHub Pages + Releases, signed with project key   │
└──────────────────────────────────────────────────────────────┘
```

Separate images for CM4 and CM5 (`uconsole-arch-cm4.img.xz` /
`uconsole-arch-cm5.img.xz`) built from the same pipeline with a target flag.
Userspace is identical across the two; only the kernel, firmware, and boot
files differ.

### Repository Layout

```
arch-uconsole/
├── packages/                    # PKGBUILDs, one dir each
│   ├── linux-uconsole-cm4/
│   ├── linux-uconsole-cm5/
│   ├── uconsole-firmware-cm4/
│   ├── uconsole-firmware-cm5/
│   ├── uconsole-dtbs/
│   ├── uconsole-keyboard/
│   ├── uconsole-audio/
│   ├── uconsole-power/
│   ├── uconsole-display/
│   ├── uconsole-wifi-bt/
│   ├── uconsole-modem/          # opt-in
│   ├── uconsole-lora/            # opt-in
│   ├── uconsole-hyprland-config/ # configs/assets only
│   ├── uconsole-hyprland/        # metapackage: config + Hyprland stack
│   ├── uconsole-welcome/
│   ├── uconsole-arch-test/
│   ├── uconsole-meta/            # default metapackage
│   └── uconsole-meta-expansion/  # adds modem + lora
├── bootstrap/
│   └── uconsole-bootstrap        # POSIX-sh install script
├── image-builder/
│   ├── mkimage.sh                # main entrypoint
│   ├── stages/                   # 00-prepare, 10-partition, ...
│   └── configs/                  # fstab templates, cmdline.txt, config.txt
├── ci/
│   ├── build-packages.yml        # GHA: build & sign PKGBUILDs on change
│   ├── build-repo.yml            # GHA: assemble uconsole.db, publish to Pages
│   ├── build-image.yml           # GHA: produce release images on tag
│   └── lint.yml                  # GHA: shellcheck, namcap, PR checks
├── docs/
│   ├── superpowers/specs/        # design specs
│   ├── MAINTAINERS.md            # signing-key handling, release process
│   └── README.md
└── LICENSE
```

## Kernel, Firmware, Device Tree

### `linux-uconsole-cm4` / `linux-uconsole-cm5`

- **Source:** `raspberrypi/linux`, branch `rpi-6.12.y` (or current LTS at v1
  cut). CM4 and CM5 pin different branches; CM5 tracks the `bcm2712`-era
  tree.
- **Patches:** `patches/` directory in the PKGBUILD, applied in `prepare()`.
  Initial set imported from ClockworkPi's `cpi-v6.x-cm{4,5}` branches,
  rebased onto upstream RPi. Expected patches: DSI panel driver / timings,
  I2C keyboard driver (`cpi_keyboard`), PMIC / audio-routing quirks.
- **Config:** derived from `bcm2711_defconfig` (CM4) / `bcm2712_defconfig`
  (CM5) with uConsole-needed modules enabled. Config checked into PKGBUILD.
- **Install:** `/boot/kernel8.img` (CM4) / `/boot/kernel_2712.img` (CM5) plus
  modules under `/usr/lib/modules/<ver>`.
- **Hooks:** pacman install hook runs `mkinitcpio` and copies the kernel to
  the boot partition.

### `uconsole-firmware-cm4` / `uconsole-firmware-cm5`

Wraps `raspberrypi/rpi-firmware` bits (CM4) and the CM5 equivalent firmware
set. Version-pinned to the corresponding `linux-uconsole-cm*` so the
tested kernel / firmware pair ships together, rather than relying on
whatever ALARM's `raspberrypi-firmware` happens to carry.

### `uconsole-dtbs`

Single shared package. Contains: base uConsole device tree overlay
(`uconsole.dtbo`), module-specific overlays (`uconsole-lora.dtbo`,
`uconsole-4g.dtbo`), installed to `/boot/overlays/`. Also ships `config.txt`
template fragments enabling them, per-compute-module.

### Kernel Maintenance

1. Upstream RPi cuts a new `rpi-6.N.y` release.
2. CI bot opens a PR bumping `pkgver` and re-running the patch series;
   conflicts surface as failed CI.
3. Maintainer rebases patches, merges, CI builds signed `.pkg.tar.zst`,
   publishes to the repo.

### Boot Files

- `config.txt` / `cmdline.txt` shipped via `uconsole-dtbs` (or a small
  `uconsole-bootfiles` package, decision deferred to implementation).
- Distinct CM4 vs CM5 templates (different `kernel=` filename, different
  overlays list).
- `cmdline.txt` uses `root=LABEL=ROOT` so the same image works regardless
  of SD card size.

## Hardware Support Packages

### `uconsole-keyboard`
Userspace keymap, systemd service, udev rule, Hyprland bindings for
uConsole hotkeys (brightness, volume, keyboard backlight). The kernel
driver itself lives in `linux-uconsole-*`.

### `uconsole-audio`
WirePlumber / PipeWire config profile for the WM8960 codec — speaker,
headphone detect, mic, HDMI sink for CM5 HDMI-out. ALSA UCM profile if
upstream is missing one.

### `uconsole-power`
Battery integration against the PMIC (exact chip verified during
implementation). Ships a systemd service watching `/sys/class/power_supply/`
and triggering suspend / poweroff at low-battery thresholds. Waybar reads
the same node. Hypridle defaults: dim at 2 min, suspend at 10 min on
battery.

### `uconsole-display`
Persistent backlight via systemd-backlight, any DSI panel quirks, X11
rotation config as fallback.

### `uconsole-wifi-bt`
Broadcom firmware overrides if RPi firmware doesn't cover the uConsole WiFi
/ BT module (validate during bring-up; package may end up a pure meta-dep
on `linux-firmware`). NetworkManager as the network stack.

### `uconsole-modem` (opt-in)
Depends on `modemmanager`, `usb-modeswitch`. Udev rule + ModemManager
config for the 4G expansion module's USB IDs. Wires modem as a
NetworkManager connection candidate.

### `uconsole-lora` (opt-in)
LoRa userspace daemon (package selection finalized at implementation time,
guided by ClockworkPi's current recipe). Udev rule for the LoRa SPI device.
Config under `/etc/uconsole/lora/`.

### `uconsole-welcome`
First-login script. `/etc/profile.d/uconsole-welcome.sh` fires
`/usr/bin/uconsole-welcome` once (sentinel at
`/var/lib/uconsole/welcome.done`). Walks: passwd → `useradd` → `iwctl`
WiFi → timezone → reboot prompt.

### `uconsole-arch-test`
CLI that runs probes (display present, keyboard events, audio device
detected, WiFi associated, BT adapter up, battery reading sane, optional
modem / LoRa if present) and prints a pass/fail report with troubleshooting
hints.

## Userspace / Desktop

### Display Manager
`ly` (TUI). Enabled via `ly.service`.

### Compositor & Environment
Hyprland with a tuned config for 720×1280 landscape:
- `monitor=DSI-1,720x1280@60,0x0,1,transform,1`
- uConsole hotkey bindings wired to `brightnessctl` / `pamixer` /
  keyboard-backlight control.

### `uconsole-hyprland` metapackage depends on:

| Role | Package |
|---|---|
| Compositor | `hyprland` |
| Config | `uconsole-hyprland-config` |
| Status bar | `waybar` |
| Notifications | `mako` |
| Launcher | `fuzzel` |
| Lock screen | `hyprlock` |
| Idle | `hypridle` |
| Wallpaper | `hyprpaper` |
| Desktop portal | `xdg-desktop-portal-hyprland` |
| Terminal | `ghostty` |
| Browser | `firefox` |
| Screenshot | `grim`, `slurp` |
| Clipboard (current) | `wl-clipboard` |
| Clipboard history | `cliphist` |
| Audio | `pipewire`, `wireplumber`, `pamixer` |
| Brightness | `brightnessctl` |
| On-screen kbd | `wvkbd` |
| Display manager | `ly` |
| Fonts | `ttf-jetbrains-mono-nerd`, `noto-fonts`, `noto-fonts-emoji` |

Cliphist is wired via Hyprland's `exec-once`:
```
exec-once = wl-paste --type text --watch cliphist store
exec-once = wl-paste --type image --watch cliphist store
bind = $mod, V, exec, cliphist list | fuzzel --dmenu | cliphist decode | wl-copy
```

### Additional userland pulled in by `uconsole-meta`
- `yazi` (file manager)
- `neovim` (editor)
- `blueman` (Bluetooth GUI)
- `nm-connection-editor` (NetworkManager GUI)

## Image Builder & Bootstrap

### `bootstrap/uconsole-bootstrap`

POSIX sh. Runs in three contexts: inside the image builder's chroot, on a
running ALARM install, manually for recovery. Idempotent.

Steps:
1. Add `[uconsole]` section to `/etc/pacman.conf` with signing key URL.
2. `pacman-key --recv` and `--lsign` the project key.
3. `pacman -Sy`; detect compute module via `/proc/device-tree/model` →
   select CM4 or CM5.
4. `pacman -S --needed` the correct `uconsole-meta` variant (different
   kernel/firmware pulled in via alternative-provides).
5. Enable services: `NetworkManager`, `bluetooth`, `ly`, `fstrim.timer`,
   `uconsole-power`.
6. Install `/boot/config.txt` and `/boot/cmdline.txt` from the
   module-specific template.
7. Default `/etc/hostname` = `uconsole`.
8. Write sentinel `/var/lib/uconsole/bootstrap.done` with version and
   compute-module target.

### `image-builder/mkimage.sh`

Entry: `./mkimage.sh --target cm4|cm5 [--size 4G] [--output out/]`

Stages:
1. **`00-prepare.sh`** — verify host deps (`qemu-user-static`,
   `arch-install-scripts`, `parted`, `dosfstools`, `e2fsprogs`, `xz`,
   binfmt registration). Refuse with a clear error if any are missing.
2. **`10-partition.sh`** — create loopback image, `parted` to 256 MB FAT32
   `BOOT` + ext4 `ROOT`, format, mount under `build/rootfs/`, bind `BOOT` to
   `build/rootfs/boot`.
3. **`20-pacstrap.sh`** — fetch + cache latest ALARM aarch64 tarball,
   extract into `build/rootfs/`. The pinned tarball sha256 is declared per
   release.
4. **`30-chroot-bootstrap.sh`** — register `qemu-aarch64-static`, bind
   `/dev`, `/proc`, `/sys`, `/run`, copy bootstrap into chroot, run
   `arch-chroot build/rootfs/ /usr/bin/uconsole-bootstrap --target=cm{4,5}`.
5. **`40-configure.sh`** — `/etc/fstab` using `LABEL=BOOT` / `LABEL=ROOT`,
   locale, `uconsole-resize-rootfs.service` oneshot, default credentials
   (`alarm:alarm`, `root:root` — ALARM convention), `pacman -Scc`.
6. **`50-finalize.sh`** — unmount, trim loopback, `xz -9 --threads=0` →
   `out/uconsole-arch-cm{4,5}-YYYYMMDD.img.xz`. Write `.sha256`; sign with
   the project key if the signing material is present.

### First-boot Flow
1. `uconsole-resize-rootfs.service` expands the ROOT partition via
   `growpart` + `resize2fs` to fill the SD card, then self-disables.
2. Auto-login as `alarm` on TTY.
3. `uconsole-welcome` walks passwd → user creation → WiFi → timezone →
   reboot.
4. On reboot: Ly → Hyprland.

## CI, Releases, Signing

### Signing key
Single project GPG key (RSA 4096 or ed25519). Public fingerprint baked into
`uconsole-bootstrap` and documented in README. Private key stored in
GitHub Actions secrets (`UCONSOLE_SIGNING_KEY_ASC`,
`UCONSOLE_SIGNING_KEY_PASSPHRASE`). Offline backup handling documented in
`docs/MAINTAINERS.md`.

### Workflows

All jobs use `runs-on: ubuntu-24.04-arm` (free native aarch64 runners for
public repos).

- **`build-packages.yml`** — triggers on push to `main` affecting
  `packages/**`. Matrix over changed package dirs. Runs `makepkg -s --sign`.
  Uploads artifacts.
- **`build-repo.yml`** — runs after `build-packages` succeeds. Downloads
  current package artifacts, runs `repo-add --sign uconsole.db.tar.gz
  *.pkg.tar.zst`, publishes to GitHub Pages
  (`https://<owner>.github.io/arch-uconsole/aarch64/`).
- **`build-image.yml`** — triggers on `v*` tags. Matrix `{cm4, cm5}`. Runs
  `mkimage.sh`, uploads `.img.xz` + `.sha256` + `.sig` as release assets.
- **`lint.yml`** — on every PR. `shellcheck`, `namcap`, PKGBUILD sanity
  checks, bootstrap dry-run.

### Release cadence
- **Packages:** continuous. Every merge to `main` touching `packages/**`
  republishes the repo. Users see updates via `pacman -Syu`.
- **Images:** tagged releases, cut when there's a meaningful change
  (kernel bump, userland batch). No fixed cadence.
- **Versioning:** images use `YYYY.MM.DD`; packages follow upstream version
  + `-<pkgrel>` per Arch convention.

### Reproducibility
- `SOURCE_DATE_EPOCH` set from commit time → deterministic package and
  image timestamps.
- `/etc/uconsole-release` inside the image records version + commit SHA.
- ALARM rootfs tarball sha256 pinned per release; bumps are explicit
  commits.

## Success Criteria (v1)

1. Fresh `uconsole-arch-cm5-<date>.img.xz` flashed to microSD boots a
   physical CM5 uConsole to Ly.
2. Same for CM4.
3. After first-boot wizard, Hyprland starts with working display, keyboard,
   audio, WiFi, Bluetooth, and visible battery status.
4. `uconsole-arch-test` reports all core checks passing.
5. `pacman -Syu` pulls updates from the Pages-hosted repo with valid
   signatures.
6. Bootstrap script can uconsole-ify a vanilla ALARM install on a uConsole
   and produce a working system.

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| ClockworkPi's patch series may not rebase cleanly onto current `rpi-6.12.y` | First implementation step is "get kernel building with patches"; fall back to tracking ClockworkPi's fork directly if blocked. |
| CM5 drivers still maturing at ClockworkPi | Ship CM5 image as "beta" in README until `uconsole-arch-test` is green; CM4 may ship "stable" earlier. |
| Hyprland on VideoCore VI (CM4) feels sluggish | Default config disables blur / animations on CM4. Keep a single `uconsole-hyprland-config` with runtime-detected include if feasible; otherwise split into `-cm4` / `-cm5` config packages. |
| ALARM aarch64 tarball availability / breakage | Pinned sha256 per release; "roll your own rootfs" fallback documented for maintainers. |
| Ghostty is AUR-only at build time | Build AUR deps ourselves, publish to our repo so images don't depend on AUR at install. |
| WM8960 audio routing edge cases | Ship a usable default profile; triage regressions via `uconsole-arch-test`. |

## License

- **Code** (scripts, PKGBUILDs, configs): **MIT**.
- **Patches imported from ClockworkPi's kernel**: inherit GPL-2.0 (Linux).
- **Kernel distribution**: GPL-2.0.
- **Documentation**: CC-BY-4.0.
