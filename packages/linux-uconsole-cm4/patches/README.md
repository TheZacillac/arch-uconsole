# linux-uconsole-cm4 patch series

Single consolidated patch (`0001-uConsole-CM4-patches-rebased-onto-rpi-6.12.y.patch`)
rebased from ClockworkPi's original 2023-06-30 CM4 patch onto
`raspberrypi/linux` `rpi-6.12.y` HEAD (commit
`bc0c440ce8a9ba7dbcd22fcee403ef91daf5a9ec` at the time of rebase,
2026-04-15).

## Upstream source

- ClockworkPi original: `clockworkpi/uConsole:Code/patch/cm4/20230630/0001-patch-cm4.patch`
- ClockworkPi's stated base: `raspberrypi/linux@3a33f11c48572b9dd0fecac164b3990fc9234da8` (April 2021, Linux ~5.10)
- Our rebase target: `raspberrypi/linux@bc0c440ce8a9ba7dbcd22fcee403ef91daf5a9ec` (rpi-6.12.y HEAD 2026-04-15)

## What the patch contains (17 files, 1513 insertions, 8 deletions)

### New files (applied cleanly)

- **Device-tree overlays** (`arch/arm/boot/dts/overlays/devterm-*.dts`): 6
  overlays for uConsole hardware — Bluetooth, DSI panel (generic +
  uConsole-specific variant), misc GPIO / I2C, PMU (AXP209), WiFi.
- **DRM panel drivers** (`drivers/gpu/drm/panel/panel-{cwd686,cwu50}.c`):
  drivers for the ClockworkPi CWD686 and CWU50 DSI panels (uConsole uses
  CWU50).
- **Backlight driver** (`drivers/video/backlight/ocp8178_bl.c`): driver
  for the OCP8178 PWM backlight controller.

### Modified files (applied cleanly)

- `drivers/gpu/drm/panel/{Kconfig,Makefile}` — Kconfig + Makefile entries
  for the new panel drivers.
- `drivers/gpu/drm/vc4/vc4_dsi.c` — DSI host modifications needed to
  drive the uConsole panel (non-trivial code changes, applied cleanly
  via fuzzy context match).
- `drivers/power/supply/{axp20x_ac_power,axp20x_battery}.c` — AXP PMIC
  power/battery support improvements (prop handlers).
- `drivers/video/backlight/{Kconfig,Makefile}` — Kconfig + Makefile
  entries for `ocp8178_bl`.

### Hand-ported rejects (applied via manual merge)

- `arch/arm/boot/dts/overlays/Makefile` — six `devterm-*.dtbo` entries
  added to the per-SoC dtbo list; context shifted since 2021 but content
  is purely additive.
- `drivers/gpu/drm/panel/Makefile` — two `obj-$(CONFIG_DRM_PANEL_*) +=`
  lines appended; original anchor moved as upstream added more panels.

## What got dropped (non-boot-critical, documented for Plan 4 followup)

1. **`arch/arm64/configs/bcm2711_defconfig`** — the original patch
   rebased the entire defconfig (7726-line `.rej`). ClockworkPi had
   shoved a fully-expanded `.config` into defconfig position rather
   than the minimal set of symbol changes. We manage kernel config
   separately: start from upstream `bcm2711_defconfig`, apply the
   `scripts/config --enable` deltas in Task 4, `make olddefconfig`.

2. **`drivers/mfd/axp20x.c` `pm_power_off` workaround** — ClockworkPi
   set `pm_power_off = 0;` unconditionally before a null-check in
   `axp20x_device_probe`. The `pm_power_off` symbol was removed from
   this file in modern kernels (power-off handlers use the
   `sys_off_handler` API now). The workaround is obsolete; system can
   still shut down via `/sbin/poweroff` → sys_off_handler chain.

3. **`drivers/staging/vc04_services/bcm2835-audio/bcm2835.c` module
   parameter defaults** — ClockworkPi switched `enable_hdmi` and
   `enable_headphones` defaults. Upstream already has
   `enable_headphones = true`; the remaining deltas
   (`enable_hdmi=true`, `enable_compat_alsa=false`) are cosmetic
   module-param defaults that can be overridden via boot args
   (`snd-bcm2835.enable_hdmi=1`).

4. **`drivers/power/supply/axp20x_battery.c` init-register writes and
   property-array addition** — the original patch added
   `POWER_SUPPLY_PROP_ENERGY_{FULL,NOW}` to the exposed props array and
   ran 5 `regmap_update_bits()` calls in `axp20x_power_probe()` to set
   VBUS/OFF/CHRG/PEK/GPIO behaviour. The function has been refactored
   upstream; anchors don't match. The ENERGY case in `_get_prop()` DID
   apply (dead code until the props-array hunk is reinstated).
   Consequence: battery CAPACITY property (percentage) still works;
   ENERGY_FULL / ENERGY_NOW read 0 until someone forward-ports the
   remaining two hunks against current kernel layout.

## Adding or updating patches

1. Ensure the patch applies against the pinned RPi SHA in
   `docs/upstream-refs.md` (regenerate the rebase per the Plan 2 Task 3
   procedure).
2. For multi-hunk patches, preserve numbered ordering: `NNNN-short.patch`.
3. Commit with a message describing intent AND what upstream context
   shifted.

## Rebasing on a new upstream

When bumping `pkgver` to a newer `rpi-6.N.y` HEAD:

1. Clone the new target branch.
2. `git apply --reject packages/linux-uconsole-cm4/patches/0001-*.patch`
3. For each `.rej` file, hand-port against the new context.
4. `git add -A && git commit` → `git format-patch -1` → replace the
   patch file in this directory.
5. Update `docs/upstream-refs.md` with the new pinned SHA.
6. Run through CI + hardware test before merging.
