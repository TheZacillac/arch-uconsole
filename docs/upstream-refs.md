# Upstream references

Pinned upstream git refs for arch-uconsole kernel and firmware packages.
Update in lockstep with PKGBUILD `pkgver`/`source` changes.

## raspberrypi/linux

- **Repo:** `https://github.com/raspberrypi/linux`
- **Branch:** `rpi-6.12.y` (tentative; see "CM4 kernel base" note below)
- **rpi-6.12.y HEAD SHA (recorded 2026-04-15):**
  `bc0c440ce8a9ba7dbcd22fcee403ef91daf5a9ec`
- **ClockworkPi's official patch base SHA (April 2021, Linux ~5.10):**
  `3a33f11c48572b9dd0fecac164b3990fc9234da8`
- **Decision:** TBD — see note below.

### Note — CM4 kernel base

ClockworkPi's publicly published uConsole CM4 kernel patch (dated
2023-06-30 at `Code/patch/cm4/20230630/0001-patch-cm4.patch` in
`clockworkpi/uConsole`) targets upstream RPi commit
`3a33f11c48572b9dd0fecac164b3990fc9234da8` — a commit from April 2021,
which corresponds to roughly Linux 5.10 era. Their most recent OS image
(2026-01-09) ships kernel 6.12.62, but they have not published updated
patches against a modern kernel.

Two realistic options for Plan 2 v0.2.0:

1. **Follow ClockworkPi verbatim.** Build against `3a33f11c`, apply their
   2023 patch as-is. Produces an older kernel (~5.10) but is known to
   work on real uConsole hardware. Fast path to a shipping v0.2.0.

2. **Rebase the patch onto `rpi-6.12.y` HEAD.** 9521-line patch touching
   20 files — roughly half are new files (panel drivers, backlight
   driver) that will rebase cleanly; the rest (`bcm2711_defconfig`,
   `drivers/gpu/drm/vc4/vc4_dsi.c`, `drivers/mfd/axp20x.c`,
   `drivers/power/supply/axp20x_*.c`) will require hand-rebasing against
   5 years of upstream evolution. Produces a modern kernel but is real
   work requiring kernel-internals expertise.

## clockworkpi/uConsole (patch source)

- **Repo:** `https://github.com/clockworkpi/uConsole`
- **Default branch:** `master`
- **uConsole repo HEAD SHA (recorded 2026-04-15):**
  `ca7f52ff10d5760c0939944b16a70765bf3398fe`
- **CM4 patch path:** `Code/patch/cm4/20230630/0001-patch-cm4.patch`
- **CM4 patch base commit (against `raspberrypi/linux`):**
  `3a33f11c48572b9dd0fecac164b3990fc9234da8`

### Note — CM5 patches do not exist publicly

As of 2026-04-15 there is no `Code/patch/cm5/` directory in the
ClockworkPi uConsole repo. The only CM5-specific content published is
userspace scripts (`Code/scripts/uconsole-4g-cm5` — a 4G modem control
script). The CM5 kernel / device-tree / panel driver work has not been
published by ClockworkPi.

**Impact on Plan 2:** the CM5 half of the plan (Tasks 12–15) cannot
proceed until ClockworkPi publishes CM5 patches OR we undertake the
substantial work of writing them ourselves. Recommendation: scope v0.2.0
to **CM4 only**; add CM5 in a later plan once either of those unblocks.

## raspberrypi/rpi-firmware

- **Repo:** `https://github.com/raspberrypi/rpi-firmware`
- **Branch:** `master`
- **Pinned commit:** `2d264db2977759a12b755cd0b560ddd086d45bb6`
- **Recorded:** 2026-04-15
