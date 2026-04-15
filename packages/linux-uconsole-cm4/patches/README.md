# linux-uconsole-cm4 patch series

**No patches currently needed.**

As of rpi-6.12.y (commit `bc0c440ce8a9ba7dbcd22fcee403ef91daf5a9ec`,
recorded 2026-04-15), the Raspberry Pi kernel has absorbed the entirety
of ClockworkPi's uConsole driver work:

- DSI panel drivers (`panel-cwd686.c`, `panel-cwu50.c`)
- Backlight driver (`ocp8178_bl.c`)
- DT overlays (`devterm-{bt,misc,panel,panel-uc,pmu,wifi}-overlay.dts`)
- vc4 DSI host modifications
- Kconfig + Makefile entries for all of the above
- Overlay Makefile entries

The kernel config (`config` file alongside PKGBUILD) enables these as
modules; `make olddefconfig` resolves dependencies. No source patches
are required.

## Historical context

ClockworkPi published a single monolithic patch (9521 lines, 20 files)
at `clockworkpi/uConsole:Code/patch/cm4/20230630/0001-patch-cm4.patch`,
targeting `raspberrypi/linux@3a33f11c` (~Linux 5.10, April 2021). That
patch was rebased onto `rpi-6.12.y` during Plan 2 development, at which
point we discovered that upstream had absorbed all changes. The rebased
patch was removed as redundant.

One minor delta was identified and intentionally not carried:
3-line VBUS register toggle in `axp20x_ac_power.c` IRQ handler — a
power-switching workaround whose necessity has not been validated on
rpi-6.12.y.

## Adding patches in the future

If upstream ever diverges or we need uConsole-specific customization:
1. Name patches `NNNN-short-title.patch` in this directory.
2. The PKGBUILD's `prepare()` loops over `${startdir}/patches/*.patch`.
3. Update `docs/upstream-refs.md` with the new pinned SHA.
4. Test via CI + hardware before merging.
