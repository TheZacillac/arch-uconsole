# arch-uconsole — Plan 2: Kernel + Firmware + DTB Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship the full boot chain for the uConsole — `linux-uconsole-cm4` + `linux-uconsole-cm5` kernels, `uconsole-firmware-cm{4,5}`, and a shared `uconsole-dtbs` package — so a user can install these packages on a scratch Arch Linux ARM rootfs, flash it to an SD card, and boot to a TTY on real CM4 or CM5 uConsole hardware.

**Architecture:** Kernel PKGBUILDs pull `raspberrypi/linux` rpi-6.12.y, apply a curated patch series from ClockworkPi's fork, build with an aarch64 cross-compile toolchain on GitHub's x86_64 runners (~8 min per kernel vs ~30 min native). Firmware packages wrap `raspberrypi/rpi-firmware` with a pinned version. `uconsole-dtbs` ships uConsole-specific DTB overlays + `config.txt` / `cmdline.txt` templates per compute module. Bootstrap script detects the running compute module and installs the matching `uconsole-meta-cm{4,5}` dependency closure.

**Tech Stack:** bash / POSIX sh, Arch PKGBUILD, makepkg, aarch64 cross-compile toolchain (`aarch64-linux-gnu-gcc`), GitHub Actions matrix builds, Linux kernel build system (`make ARCH=arm64`), device tree compiler (`dtc`).

---

## Reference: File Structure After This Plan

```
arch_build/
├── packages/
│   ├── linux-uconsole-cm4/
│   │   ├── PKGBUILD
│   │   ├── config                 # kernel .config (text, committed)
│   │   ├── linux.install          # pacman hook snippets
│   │   └── patches/
│   │       ├── 0001-*.patch       # ClockworkPi CM4 patches, cherry-picked
│   │       ├── 0002-*.patch
│   │       └── ...
│   ├── linux-uconsole-cm5/
│   │   ├── PKGBUILD
│   │   ├── config
│   │   ├── linux.install
│   │   └── patches/
│   ├── uconsole-firmware-cm4/
│   │   └── PKGBUILD
│   ├── uconsole-firmware-cm5/
│   │   └── PKGBUILD
│   ├── uconsole-dtbs/
│   │   ├── PKGBUILD
│   │   ├── overlays/              # .dts sources for uConsole-specific overlays
│   │   │   ├── uconsole-base.dts
│   │   │   ├── uconsole-lora.dts
│   │   │   └── uconsole-4g.dts
│   │   └── boot/
│   │       ├── cm4/config.txt
│   │       ├── cm4/cmdline.txt
│   │       ├── cm5/config.txt
│   │       └── cm5/cmdline.txt
│   └── uconsole-hello/             # unchanged
├── bootstrap/
│   └── uconsole-bootstrap          # MODIFIED: detect CM, install the right kernel
├── .github/workflows/
│   └── build-packages.yml          # MODIFIED: matrix-by-package + cross toolchain
├── docs/
│   ├── upstream-refs.md            # NEW: ClockworkPi branch + rpi/linux branch pins
│   ├── hardware-test-cm4.md        # NEW: first-boot verification procedure CM4
│   └── hardware-test-cm5.md        # NEW: first-boot verification procedure CM5
└── (everything else unchanged)
```

**File responsibility per path:**
- `PKGBUILD` — package recipe. One per package dir. Do not split.
- `patches/NNNN-*.patch` — one logical change per patch. Numbered so `makepkg`'s `prepare()` applies in order.
- `config` — kernel `.config`, committed as text. Diff-trackable.
- `linux.install` — pacman hooks (post-install, post-upgrade, pre-remove). Copies kernel to `/boot`, runs `mkinitcpio`.
- `overlays/*.dts` — hand-written device tree source for uConsole-specific overlays that upstream RPi doesn't ship. Compiled to `.dtbo` in `build()`.
- `boot/cm{4,5}/config.txt` — template shipped as-is to `/boot/` by the dtbs package.
- `docs/upstream-refs.md` — ground-truth record of upstream git refs we pinned. Avoids drift.
- `docs/hardware-test-cm{4,5}.md` — user-facing "flash + boot + check" procedure for each compute module.

---

## Task 0: Research ClockworkPi's kernel branch pins

This task is **human-only**. I need you to open browsers and record facts about upstream.

- [ ] **Step 1: Find ClockworkPi's Linux kernel fork**

Open `https://github.com/clockworkpi` in a browser. Look for a repository named `linux`, `cpi-linux`, `clockworkpi-linux`, or similar.

Write down:
- **Repo URL** (e.g. `https://github.com/clockworkpi/linux`)
- **Branch that looks current for CM4** (the pattern we expect is `cpi-v6.x-cm4` or similar — pick the most recent 6.x branch with CM4 in the name)
- **Branch that looks current for CM5**
- **Commit SHA (full 40-char)** of the HEAD of each branch at the time you look (this pins us against a moving target)

- [ ] **Step 2: Check if `rpi-6.12.y` is a real branch on `raspberrypi/linux`**

Open `https://github.com/raspberrypi/linux/branches/all?query=rpi-6.12`.

- If `rpi-6.12.y` exists: record its HEAD SHA.
- If not: pick the highest 6.x branch that exists (e.g. `rpi-6.10.y`) and record its HEAD SHA instead. Tell me what you picked and we'll adjust downstream tasks.

- [ ] **Step 3: Record findings in `docs/upstream-refs.md`**

Create `docs/upstream-refs.md` with this content (substitute real values):

```markdown
# Upstream references

Pinned upstream git refs for arch-uconsole kernel and firmware packages.
Update in lockstep with PKGBUILD `pkgver`/`source` changes.

## raspberrypi/linux

- **Repo:** `https://github.com/raspberrypi/linux`
- **Branch:** `rpi-6.12.y` (or the highest rpi-6.x.y branch that exists)
- **Pinned commit (CM4 + CM5 base):** `<40-char-sha>`
- **Recorded:** 2026-04-15

## clockworkpi/linux

- **Repo:** `<url-from-step-1>`
- **Branch (CM4):** `<branch-from-step-1>`
- **Pinned commit (CM4):** `<40-char-sha>`
- **Branch (CM5):** `<branch-from-step-1>`
- **Pinned commit (CM5):** `<40-char-sha>`
- **Recorded:** 2026-04-15

## raspberrypi/rpi-firmware

- **Repo:** `https://github.com/raspberrypi/rpi-firmware`
- **Branch:** `master`
- **Pinned commit:** `<40-char-sha>` (capture at build time — see Task 8)
- **Recorded:** 2026-04-15
```

Fill in real values. For the rpi-firmware SHA, you can leave it `<filled in Task 8>` for now.

- [ ] **Step 4: Commit and report values back to me**

```bash
git add docs/upstream-refs.md
git commit -m "docs: record upstream git pins for kernel + firmware"
git push
```

Then paste back to me (or put in conversation):
- `raspberrypi/linux` branch + SHA
- `clockworkpi/linux` repo URL, CM4 branch + SHA, CM5 branch + SHA

Subsequent tasks substitute these values. If any are missing I'll pause to ask.

---

## Task 1: `.gitignore` updates for kernel source handling

**Files:**
- Modify: `.gitignore`

The existing globs ignore `*.tar.gz` and `*.tar.xz`, which would silently hide upstream kernel tarballs if any ever land in-tree. Refine to be directory-anchored so build-time downloads under `src/` are ignored (they already are via `src/`) but any legitimate in-tree tarball (e.g., future pinned vendor blobs) would still be visible.

- [ ] **Step 1: Edit `.gitignore`**

Replace the current top section:

```diff
-# Build artifacts
-*.pkg.tar.zst
-*.pkg.tar.zst.sig
-*.tar.gz
-*.tar.xz
-src/
-pkg/
+# Build artifacts produced by makepkg (any depth)
+**/src/
+**/pkg/
+**/*.pkg.tar.zst
+**/*.pkg.tar.zst.sig
+
+# Downloaded source tarballs cached by makepkg under src/
+# (already covered by **/src/, listed separately for clarity)
+**/linux-*.tar.*
+**/rpi-firmware-*.tar.*
```

Full new `.gitignore`:

```
# Build artifacts produced by makepkg (any depth)
**/src/
**/pkg/
**/*.pkg.tar.zst
**/*.pkg.tar.zst.sig

# Downloaded source tarballs cached by makepkg under src/
# (already covered by **/src/, listed separately for clarity)
**/linux-*.tar.*
**/rpi-firmware-*.tar.*

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

- [ ] **Step 2: Verify no current tracked file is newly ignored**

```bash
git status --ignored
# Should list only expected directories (image-builder/{build,out,cache} don't exist yet, etc.)

git ls-files | xargs -I{} git check-ignore {} 2>/dev/null
# Should print nothing (no tracked files match the ignore patterns).
```

- [ ] **Step 3: Commit**

```bash
git add .gitignore
git commit -m "chore(gitignore): directory-anchor makepkg artifact globs"
git push
```

---

## Task 2: Skeleton for `packages/linux-uconsole-cm4/`

**Files:**
- Create: `packages/linux-uconsole-cm4/PKGBUILD` (stub)
- Create: `packages/linux-uconsole-cm4/patches/.keep`

Skeleton first; real PKGBUILD content lands in Task 5 after we have the patches.

- [ ] **Step 1: Create the directory with a stub PKGBUILD**

```bash
mkdir -p packages/linux-uconsole-cm4/patches
touch packages/linux-uconsole-cm4/patches/.keep
```

Create `packages/linux-uconsole-cm4/PKGBUILD` (stub — will be replaced in Task 5):

```bash
# Maintainer: arch-uconsole <noreply@example.org>
# Stub PKGBUILD — replaced in Plan 2 Task 5 with the real kernel recipe.

pkgname=linux-uconsole-cm4
pkgver=0.0.0
pkgrel=1
pkgdesc="uConsole CM4 kernel (stub — not yet built)"
arch=('aarch64')
url="https://github.com/TheZacillac/arch-uconsole"
license=('GPL2')

package() {
    install -dm755 "${pkgdir}/usr/share/doc/linux-uconsole-cm4"
    printf 'stub\n' > "${pkgdir}/usr/share/doc/linux-uconsole-cm4/STATUS"
    chmod 644 "${pkgdir}/usr/share/doc/linux-uconsole-cm4/STATUS"
}
```

- [ ] **Step 2: Commit**

```bash
git add packages/linux-uconsole-cm4/
git commit -m "feat(linux-uconsole-cm4): scaffold PKGBUILD directory"
git push
```

---

## Task 3: Harvest ClockworkPi patches into `linux-uconsole-cm4/patches/`

**Files:**
- Create: `packages/linux-uconsole-cm4/patches/NNNN-*.patch` (one per logical change)
- Create: `packages/linux-uconsole-cm4/patches/README.md`

Cherry-pick the uConsole-specific changes from ClockworkPi's CM4 branch onto our `rpi-6.12.y` base, one logical commit per patch file. Mechanical once the branch pins are known.

**Needs from Task 0:** `clockworkpi/linux` repo URL, CM4 branch name, CM4 HEAD SHA; `raspberrypi/linux` rpi-6.12.y HEAD SHA.

- [ ] **Step 1: Clone both repos into a scratch directory**

```bash
cd /tmp
rm -rf kernel-harvest && mkdir kernel-harvest && cd kernel-harvest

# Substitute real URLs/branches from docs/upstream-refs.md
RPI_REPO="https://github.com/raspberrypi/linux"
RPI_BRANCH="rpi-6.12.y"
CPI_REPO="<from docs/upstream-refs.md>"
CPI_BRANCH_CM4="<from docs/upstream-refs.md>"

git clone --depth 200 --branch "$RPI_BRANCH" "$RPI_REPO" rpi
git -C rpi remote add cpi "$CPI_REPO"
git -C rpi fetch --depth 200 cpi "$CPI_BRANCH_CM4"
```

- [ ] **Step 2: Identify the merge-base and list ClockworkPi-specific commits**

```bash
cd /tmp/kernel-harvest/rpi

# Find the common ancestor between RPi and ClockworkPi CM4 branch
BASE=$(git merge-base "cpi/$CPI_BRANCH_CM4" "$RPI_BRANCH")
echo "merge-base: $BASE"

# List commits on ClockworkPi side (newest first)
git log --oneline --no-merges "$BASE..cpi/$CPI_BRANCH_CM4" | head -50
```

Expected: anywhere from ~5 to ~50 commits. Each commit is a candidate patch.

- [ ] **Step 3: Inspect and filter commits**

Walk the commit list. For each, `git show --stat <sha>` and decide:
- **Keep** if it touches: DSI panel driver, I2C keyboard driver, WM8960 audio, AXP228 PMIC / battery, device tree for `uconsole-*`, or any obviously uConsole-specific file.
- **Drop** if it's: a merge commit, version bump, cosmetic whitespace change, or CI / tooling change unrelated to kernel source.

Record the keep-list (short SHAs + titles) in a scratch notebook — you'll need it for Step 4.

- [ ] **Step 4: Export kept commits as numbered patches**

```bash
cd /tmp/kernel-harvest/rpi
mkdir -p /tmp/kernel-harvest/patches

# For each kept SHA (in chronological order, oldest first):
# substitute your keep-list
KEPT_SHAS=(
  # "<sha> # title"
  "abcdef1 # drm/panel: add ClockworkPi uConsole DSI panel"
  "bcdef23 # input: cpi_keyboard: add i2c handler"
  # ... etc
)

n=1
for entry in "${KEPT_SHAS[@]}"; do
  sha="${entry%% *}"
  num=$(printf '%04d' "$n")
  git format-patch --output-directory /tmp/kernel-harvest/patches \
    --start-number "$n" -1 "$sha"
  n=$((n + 1))
done

ls /tmp/kernel-harvest/patches/
# Expected: 0001-foo.patch, 0002-bar.patch, ...
```

- [ ] **Step 5: Copy patches into the repo**

```bash
cp /tmp/kernel-harvest/patches/*.patch \
   /home/zac/Projects/arch_build/packages/linux-uconsole-cm4/patches/
rm /home/zac/Projects/arch_build/packages/linux-uconsole-cm4/patches/.keep
ls /home/zac/Projects/arch_build/packages/linux-uconsole-cm4/patches/
```

- [ ] **Step 6: Verify patches apply cleanly against the rpi-6.12.y tip**

```bash
cd /tmp/kernel-harvest/rpi
git checkout "$RPI_BRANCH"
git reset --hard  # just in case

for p in /home/zac/Projects/arch_build/packages/linux-uconsole-cm4/patches/*.patch; do
  echo "=== applying $(basename "$p") ==="
  if ! git apply --check "$p" 2>&1; then
    echo "FAILED: $p"
    exit 1
  fi
  git apply "$p"
done
echo "all patches applied cleanly"
```

If any patch fails, return to Step 3 — you may have included a commit that depends on an earlier ClockworkPi-specific commit you filtered out. Re-examine and either include the dependency or skip the dependent patch.

- [ ] **Step 7: Write `patches/README.md`**

Create `packages/linux-uconsole-cm4/patches/README.md`:

```markdown
# linux-uconsole-cm4 patch series

Cherry-picked from `clockworkpi/linux` onto `raspberrypi/linux
rpi-6.12.y`. Rebased onto whichever SHA `docs/upstream-refs.md`
currently pins. Applied in numerical order by `prepare()` in the
PKGBUILD.

## Adding patches

1. Ensure the patch applies against the current pinned RPi SHA (see
   Task 3 Step 6 for the verification procedure).
2. Name it `NNNN-short-title.patch` with the next number in sequence.
3. Commit with a message describing what the patch does and where it
   came from upstream.

## Rebasing on a new upstream

When bumping `pkgver` and the RPi branch HEAD:
1. Re-run Task 3 Steps 1-6 with the new SHA.
2. Patches that no longer apply cleanly: rebase manually, commit the
   new version, note the conflict in the commit message.
3. Patches that upstream has absorbed: delete the file.
```

- [ ] **Step 8: Commit**

```bash
cd /home/zac/Projects/arch_build
git add packages/linux-uconsole-cm4/patches/
git commit -m "feat(linux-uconsole-cm4): import ClockworkPi CM4 patch series"
git push
```

Expected patches commit: a single commit adding N `.patch` files + README.

---

## Task 4: Generate initial kernel config for CM4

**Files:**
- Create: `packages/linux-uconsole-cm4/config`

Derive from `bcm2711_defconfig` and apply uConsole-specific tweaks (enable modules for the DSI panel, I2C keyboard, WM8960, AXP228, LoRa SPI, etc.).

- [ ] **Step 1: Generate the base config inside the scratch tree**

```bash
cd /tmp/kernel-harvest/rpi

# Apply our patch series so config knobs for uConsole drivers exist:
for p in /home/zac/Projects/arch_build/packages/linux-uconsole-cm4/patches/*.patch; do
  git apply "$p"
done

# Install cross-compile toolchain locally if missing:
# Arch:   pacman -S aarch64-linux-gnu-gcc bc
# Ubuntu: apt install gcc-aarch64-linux-gnu bc flex bison libssl-dev

export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-

make bcm2711_defconfig
```

- [ ] **Step 2: Enable uConsole-specific drivers**

Use `make menuconfig` **OR** apply the delta directly with `scripts/config`:

```bash
# Apply deltas non-interactively:
./scripts/config \
  --enable CONFIG_DRM_PANEL_CLOCKWORKPI_UCONSOLE \
  --enable CONFIG_KEYBOARD_CPI_UCONSOLE \
  --enable CONFIG_SND_SOC_WM8960 \
  --enable CONFIG_MFD_AXP20X_I2C \
  --enable CONFIG_BATTERY_AXP20X \
  --module CONFIG_BATTERY_AXP20X \
  --module CONFIG_SND_SOC_WM8960 \
  --module CONFIG_KEYBOARD_CPI_UCONSOLE \
  --module CONFIG_DRM_PANEL_CLOCKWORKPI_UCONSOLE
make olddefconfig
```

If any of these `CONFIG_*` symbols don't exist (config name may differ from patch to patch), `scripts/config` silently skips them. Verify by grepping the resulting `.config`:

```bash
grep -E 'CLOCKWORKPI|CPI_UCONSOLE|WM8960|AXP20X' .config
# Expect: one =m or =y line per expected symbol.
```

If a symbol is missing, open `make menuconfig`, navigate by driver path, and enable manually. Save and exit.

- [ ] **Step 3: Copy `.config` into the repo**

```bash
cp /tmp/kernel-harvest/rpi/.config \
   /home/zac/Projects/arch_build/packages/linux-uconsole-cm4/config

cd /home/zac/Projects/arch_build
wc -l packages/linux-uconsole-cm4/config
# Expect: several thousand lines.
```

- [ ] **Step 4: Commit**

```bash
git add packages/linux-uconsole-cm4/config
git commit -m "feat(linux-uconsole-cm4): initial kernel config (bcm2711_defconfig + uConsole tweaks)"
git push
```

---

## Task 5: Real `linux-uconsole-cm4` PKGBUILD

**Files:**
- Modify: `packages/linux-uconsole-cm4/PKGBUILD` (replace stub)
- Create: `packages/linux-uconsole-cm4/linux.install`

Real recipe. Pulls upstream RPi tarball, applies our patch series, builds cross-compiled on x86_64, packages kernel + DTBs + modules.

**Needs from Task 0:** `raspberrypi/linux` rpi-6.12.y pinned SHA.

- [ ] **Step 1: Write the real PKGBUILD**

Replace `packages/linux-uconsole-cm4/PKGBUILD` contents with:

```bash
# Maintainer: arch-uconsole <noreply@example.org>
#
# linux-uconsole-cm4 — Linux kernel for the ClockworkPi uConsole CM4.
# Based on raspberrypi/linux rpi-6.12.y plus a patch series curated from
# clockworkpi/linux. Cross-compiled from x86_64 to aarch64.
#
# See docs/upstream-refs.md for the exact pinned commit SHAs.

pkgbase=linux-uconsole-cm4
pkgname=("${pkgbase}" "${pkgbase}-headers")
pkgver=6.12.0
_rpibranch=rpi-6.12.y
_rpisha=<FILL-FROM-docs/upstream-refs.md>
pkgrel=1
arch=('aarch64')
url="https://github.com/TheZacillac/arch-uconsole"
license=('GPL2')
makedepends=(
    'bc' 'kmod' 'cpio' 'rsync' 'perl' 'xz' 'pahole'
    'aarch64-linux-gnu-gcc'
)
options=('!strip')

source=(
    "linux-${_rpisha}.tar.gz::https://github.com/raspberrypi/linux/archive/${_rpisha}.tar.gz"
    'config'
    'linux.install'
)
sha256sums=(
    'SKIP'
    'SKIP'
    'SKIP'
)

_srcdir="linux-${_rpisha}"

prepare() {
    cd "${_srcdir}"

    echo "::> applying uConsole patch series"
    for p in "${startdir}"/patches/*.patch; do
        echo "    $(basename "$p")"
        patch -Np1 -i "$p"
    done

    echo "::> installing config"
    cp -v "${srcdir}/config" .config

    echo "::> olddefconfig"
    make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig

    _kernver="$(make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -s kernelrelease)"
    echo "::> kernel release: ${_kernver}"
}

build() {
    cd "${_srcdir}"
    make -j"$(nproc)" \
        ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
        Image.gz modules dtbs
}

_package() {
    pkgdesc="Linux kernel for ClockworkPi uConsole CM4"
    depends=('coreutils' 'kmod' 'initramfs')
    optdepends=(
        'linux-firmware: firmware images needed for some devices'
        'wireless-regdb: to set correct wireless channels'
    )
    backup=()
    install=linux.install

    cd "${_srcdir}"
    local _kernver
    _kernver="$(make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -s kernelrelease)"

    install -Dm644 "arch/arm64/boot/Image.gz" \
        "${pkgdir}/usr/lib/modules/${_kernver}/vmlinuz"

    echo "::> installing modules"
    make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
        INSTALL_MOD_PATH="${pkgdir}/usr" \
        INSTALL_MOD_STRIP=1 \
        modules_install

    # Drop firmware / source links created by modules_install
    rm -rf "${pkgdir}/usr/lib/modules/${_kernver}/build"
    rm -rf "${pkgdir}/usr/lib/modules/${_kernver}/source"

    echo "::> installing DTBs"
    install -dm755 "${pkgdir}/boot/dtbs/broadcom"
    install -Dm644 arch/arm64/boot/dts/broadcom/bcm2711-rpi-cm4*.dtb \
        -t "${pkgdir}/boot/dtbs/broadcom/"

    echo "::> installing DTB overlays from upstream RPi"
    install -dm755 "${pkgdir}/boot/overlays"
    install -Dm644 arch/arm64/boot/dts/overlays/*.dtbo \
        -t "${pkgdir}/boot/overlays/"

    # Write the kernel name expected by config.txt into a known path.
    # config.txt will be `kernel=kernel8.img` (CM4 convention).
    # The install hook copies vmlinuz -> /boot/kernel8.img on install.
    echo "${_kernver}" > "${pkgdir}/usr/lib/modules/${_kernver}/pkgbase"

    # Provide a 'linux' alias so generic tooling (mkinitcpio presets,
    # drop-in initramfs consumers) can find this kernel.
    install -dm755 "${pkgdir}/usr/lib/modules/${_kernver}"
}

_package-headers() {
    pkgdesc="Headers and scripts for building modules against ${pkgbase}"
    depends=('pahole')

    cd "${_srcdir}"
    local _kernver
    _kernver="$(make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -s kernelrelease)"
    local builddir="${pkgdir}/usr/lib/modules/${_kernver}/build"

    install -Dm644 Makefile        "${builddir}/Makefile"
    install -Dm644 .config         "${builddir}/.config"
    install -Dm644 Module.symvers  "${builddir}/Module.symvers"
    install -Dm644 System.map      "${builddir}/System.map"
    install -Dm644 vmlinux         "${builddir}/vmlinux"
    install -Dm644 arch/arm64/Makefile "${builddir}/arch/arm64/Makefile"

    cp -t "${builddir}" -a include scripts

    install -Dm644 arch/arm64/include/generated/asm/syscalls_table.h \
        "${builddir}/arch/arm64/include/generated/asm/" 2>/dev/null || true

    # Strip unneeded scripts to save space
    find "${builddir}/scripts" -name '*.o' -delete
    find "${builddir}/scripts" -name '*.cmd' -delete

    # Symlink expected by modprobe etc.
    install -dm755 "${pkgdir}/usr/src"
    ln -sr "${builddir}" "${pkgdir}/usr/src/${pkgbase}-${_kernver}"
}

eval "package_${pkgbase}() { _package; }"
eval "package_${pkgbase}-headers() { _package-headers; }"
```

**Critical substitution before commit:** Replace `<FILL-FROM-docs/upstream-refs.md>` with the actual rpi-6.12.y HEAD SHA recorded in Task 0.

```bash
# e.g. (use the real value from docs/upstream-refs.md):
RPI_SHA=$(awk '/^- \*\*Pinned commit \(CM4 \+ CM5 base\)/ {gsub(/`/,""); print $NF}' docs/upstream-refs.md)
echo "pinning to: $RPI_SHA"
sed -i "s|_rpisha=<FILL-FROM-docs/upstream-refs.md>|_rpisha=${RPI_SHA}|" \
    packages/linux-uconsole-cm4/PKGBUILD
grep ^_rpisha packages/linux-uconsole-cm4/PKGBUILD
```

- [ ] **Step 2: Write `linux.install`**

Create `packages/linux-uconsole-cm4/linux.install`:

```bash
post_install() {
    post_upgrade "$@"
}

post_upgrade() {
    local kver="$1"
    cp -f "/usr/lib/modules/${kver}/vmlinuz" /boot/kernel8.img
    depmod -a "${kver}"
    echo "linux-uconsole-cm4: installed /boot/kernel8.img (${kver})"
    if command -v mkinitcpio >/dev/null 2>&1; then
        mkinitcpio -p linux-uconsole-cm4 || true
    fi
}

pre_remove() {
    :
}
```

- [ ] **Step 3: Capture real source checksums**

Once the SHA is pinned, replace the `SKIP` for the tarball with a real sha256. Run:

```bash
cd packages/linux-uconsole-cm4
# Use the exact URL from PKGBUILD (with SHA substituted):
RPI_SHA=$(grep '^_rpisha=' PKGBUILD | cut -d= -f2)
curl -fsSL -o /tmp/rpi-kernel.tar.gz \
    "https://github.com/raspberrypi/linux/archive/${RPI_SHA}.tar.gz"
sha256sum /tmp/rpi-kernel.tar.gz
# Copy the hex hash and paste into PKGBUILD replacing the first 'SKIP'.
```

Leave `config` and `linux.install` as `SKIP` — they're local files.

- [ ] **Step 4: Run `namcap` locally if available**

```bash
namcap packages/linux-uconsole-cm4/PKGBUILD
# Acceptable warnings: "ELF" warnings about bundled files, depends-as-makedeps,
# or missing source.
# Not acceptable: syntax errors or missing required fields.
```

- [ ] **Step 5: Commit**

```bash
git add packages/linux-uconsole-cm4/PKGBUILD packages/linux-uconsole-cm4/linux.install
git commit -m "feat(linux-uconsole-cm4): real PKGBUILD (rpi-6.12.y + uConsole patch series)"
git push
```

---

## Task 6: CI — cross-compile toolchain + matrix-by-package

**Files:**
- Modify: `.github/workflows/build-packages.yml`

Kernel build needs the aarch64 cross-compile toolchain. Also, serial building of all PKGBUILDs becomes painful once kernels join the list — matrix-by-package runs builds in parallel.

- [ ] **Step 1: Replace `.github/workflows/build-packages.yml` with matrix version**

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
  discover:
    runs-on: ubuntu-24.04
    outputs:
      packages: ${{ steps.find.outputs.packages }}
    steps:
      - uses: actions/checkout@v4
      - id: find
        run: |
          set -euo pipefail
          pkgs=$(find packages -maxdepth 2 -name PKGBUILD -printf '%h\n' \
            | xargs -I{} basename {} \
            | jq -R . | jq -sc .)
          echo "packages=${pkgs}" >> "$GITHUB_OUTPUT"
          echo "discovered: ${pkgs}"

  build:
    needs: discover
    if: ${{ needs.discover.outputs.packages != '[]' }}
    strategy:
      fail-fast: false
      matrix:
        package: ${{ fromJson(needs.discover.outputs.packages) }}
    runs-on: ubuntu-24.04
    container:
      image: archlinux:latest
      options: --privileged
    steps:
      - uses: actions/checkout@v4

      - name: Install build environment
        run: |
          set -euo pipefail
          pacman -Syu --noconfirm --disable-sandbox base-devel git sudo
          # Cross toolchain for aarch64 kernel builds:
          pacman -S --noconfirm --needed --disable-sandbox \
            aarch64-linux-gnu-gcc aarch64-linux-gnu-binutils \
            bc kmod cpio rsync perl xz pahole flex bison
          useradd -m -s /bin/bash builder
          echo 'builder ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers
          chown -R builder:builder "$PWD"

      - name: Import signing key and configure loopback pinentry
        env:
          KEY_ASC: ${{ secrets.UCONSOLE_SIGNING_KEY_ASC }}
          KEY_FPR: ${{ secrets.UCONSOLE_SIGNING_KEY_FPR }}
          GPG_PASSPHRASE: ${{ secrets.UCONSOLE_SIGNING_KEY_PASSPHRASE }}
        run: |
          set -euo pipefail
          sudo -u builder mkdir -p /home/builder/.gnupg
          sudo -u builder chmod 700 /home/builder/.gnupg
          sudo -u builder tee /home/builder/.gnupg/gpg-agent.conf >/dev/null <<'EOF'
          allow-loopback-pinentry
          EOF
          sudo -u builder tee /home/builder/.gnupg/gpg.conf >/dev/null <<'EOF'
          pinentry-mode loopback
          use-agent
          EOF
          sudo -u builder gpgconf --launch gpg-agent
          printf '%s' "$KEY_ASC" | sudo -u builder gpg --batch --import
          printf '%s:6:\n' "$KEY_FPR" | sudo -u builder gpg --batch --import-ownertrust
          printf 'prime' | sudo -u builder gpg --batch --yes \
            --pinentry-mode loopback \
            --passphrase "$GPG_PASSPHRASE" \
            --local-user "$KEY_FPR" \
            --detach-sign --output /dev/null

      - name: Build ${{ matrix.package }}
        env:
          KEY_FPR: ${{ secrets.UCONSOLE_SIGNING_KEY_FPR }}
        run: |
          set -euo pipefail
          dir="packages/${{ matrix.package }}"
          [ -f "${dir}/PKGBUILD" ] || { echo "no PKGBUILD"; exit 2; }
          mkdir -p artifacts
          chown -R builder:builder .
          sudo -u builder bash -c "
            cd '${dir}' &&
            makepkg -s --noconfirm --sign --key '${KEY_FPR}'
          "
          mv "${dir}"/*.pkg.tar.zst "${dir}"/*.pkg.tar.zst.sig artifacts/
          ls -la artifacts/

      - uses: actions/upload-artifact@v4
        with:
          name: pkg-${{ matrix.package }}
          path: artifacts/
          retention-days: 30

  aggregate:
    # Downstream build-repo workflow expects a single 'uconsole-packages'
    # artifact. Aggregate the matrix results.
    needs: build
    if: ${{ always() && needs.build.result == 'success' }}
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/download-artifact@v4
        with:
          pattern: pkg-*
          merge-multiple: true
          path: artifacts/
      - run: |
          echo "=== aggregated artifacts ==="
          ls -la artifacts/
      - uses: actions/upload-artifact@v4
        with:
          name: uconsole-packages
          path: artifacts/
          retention-days: 30
```

- [ ] **Step 2: Sanity-check YAML locally**

```bash
python3 -c "import yaml, sys; yaml.safe_load(open('.github/workflows/build-packages.yml'))" \
  && echo "YAML valid"
# If python3-yaml not installed: skip and rely on CI feedback.
```

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/build-packages.yml
git commit -m "ci(build-packages): matrix-per-PKGBUILD + aarch64 cross toolchain"
git push
```

- [ ] **Step 4: Watch CI**

```bash
gh run watch --exit-status -R TheZacillac/arch-uconsole
```

Expected: `discover` passes, one build job per PKGBUILD in `packages/`, all succeed, `aggregate` publishes the combined `uconsole-packages` artifact, then `build-repo.yml` chains and publishes the repo. The kernel job will take ~8-15 minutes.

If the kernel job fails, fetch the log:

```bash
gh run view --log-failed -R TheZacillac/arch-uconsole \
  $(gh run list -R TheZacillac/arch-uconsole --limit 1 --json databaseId -q '.[0].databaseId')
```

Common failure modes + fixes:
- **Missing `CONFIG_*`:** a symbol referenced in `config` doesn't exist in this kernel tree → rerun `make olddefconfig` locally, commit updated `config`.
- **Patch hunk mismatch:** upstream moved. Either rebase the patch or drop it if upstream absorbed the change.
- **Out of disk space:** kernel builds use ~3-5GB. The default runner has ~14GB; should fit. If not, add a cleanup step earlier.

---

## Task 7: `uconsole-firmware-cm4` PKGBUILD

**Files:**
- Create: `packages/uconsole-firmware-cm4/PKGBUILD`

Wraps `raspberrypi/rpi-firmware` into an installable package that drops `start*.elf`, `fixup*.dat`, `bootcode.bin` into `/boot/`. Version-locked to a known-good commit.

- [ ] **Step 1: Pin the firmware commit**

```bash
cd /tmp
rm -rf rpi-fw-check
git clone --depth 1 https://github.com/raspberrypi/rpi-firmware rpi-fw-check
cd rpi-fw-check
FW_SHA=$(git rev-parse HEAD)
echo "pinning rpi-firmware to: $FW_SHA"
```

Edit `docs/upstream-refs.md` and fill in the `<filled in Task 8>` placeholder for `rpi-firmware` with this SHA. Commit:

```bash
cd /home/zac/Projects/arch_build
# Manually edit docs/upstream-refs.md to substitute the SHA.
git add docs/upstream-refs.md
git commit -m "docs(upstream-refs): pin raspberrypi/rpi-firmware SHA"
```

- [ ] **Step 2: Write the PKGBUILD**

Create `packages/uconsole-firmware-cm4/PKGBUILD`:

```bash
# Maintainer: arch-uconsole <noreply@example.org>
#
# uconsole-firmware-cm4 — RPi boot firmware bits needed for CM4 to boot
# (bootcode.bin, start*.elf, fixup*.dat). Wraps raspberrypi/rpi-firmware
# pinned to a tested SHA (see docs/upstream-refs.md).

pkgname=uconsole-firmware-cm4
pkgver=20260415
_fwsha=<FILL-FROM-docs/upstream-refs.md>
pkgrel=1
pkgdesc="Raspberry Pi boot firmware (pinned) for uConsole CM4"
arch=('any')
url="https://github.com/TheZacillac/arch-uconsole"
license=('custom:rpi-firmware')
provides=('raspberrypi-firmware')
conflicts=('raspberrypi-firmware')

source=(
    "rpi-firmware-${_fwsha}.tar.gz::https://github.com/raspberrypi/rpi-firmware/archive/${_fwsha}.tar.gz"
)
sha256sums=('SKIP')

package() {
    local src="${srcdir}/rpi-firmware-${_fwsha}"
    install -dm755 "${pkgdir}/boot"

    # CM4 boot files (VideoCore VI bootchain):
    local files=(
        bootcode.bin
        fixup.dat
        fixup_cd.dat
        fixup_x.dat
        fixup4.dat
        fixup4cd.dat
        fixup4x.dat
        start.elf
        start_cd.elf
        start_x.elf
        start4.elf
        start4cd.elf
        start4x.elf
    )
    for f in "${files[@]}"; do
        if [ -f "${src}/${f}" ]; then
            install -Dm644 "${src}/${f}" "${pkgdir}/boot/${f}"
        else
            echo "::! ${f} missing upstream; skipping"
        fi
    done
}
```

Substitute `<FILL-FROM-docs/upstream-refs.md>`:

```bash
FW_SHA=$(awk '/raspberrypi.rpi-firmware/,/Recorded:/' docs/upstream-refs.md \
    | awk '/\*\*Pinned commit\*\*/ {gsub(/`/,""); print $NF}')
sed -i "s|_fwsha=<FILL-FROM-docs/upstream-refs.md>|_fwsha=${FW_SHA}|" \
    packages/uconsole-firmware-cm4/PKGBUILD
grep ^_fwsha packages/uconsole-firmware-cm4/PKGBUILD
```

- [ ] **Step 3: Capture the real sha256 for the firmware tarball**

```bash
FW_SHA=$(grep ^_fwsha packages/uconsole-firmware-cm4/PKGBUILD | cut -d= -f2)
curl -fsSL -o /tmp/rpi-fw.tar.gz \
  "https://github.com/raspberrypi/rpi-firmware/archive/${FW_SHA}.tar.gz"
sha256sum /tmp/rpi-fw.tar.gz
# Paste the hex into PKGBUILD, replacing 'SKIP' in sha256sums.
```

- [ ] **Step 4: Commit**

```bash
git add packages/uconsole-firmware-cm4/
git commit -m "feat(uconsole-firmware-cm4): wrap raspberrypi/rpi-firmware for CM4"
git push
```

CI should pick up the new package automatically via the matrix discovery job.

---

## Task 8: `uconsole-dtbs` PKGBUILD (CM4-only boot config for now)

**Files:**
- Create: `packages/uconsole-dtbs/PKGBUILD`
- Create: `packages/uconsole-dtbs/overlays/uconsole-base.dts` (stub)
- Create: `packages/uconsole-dtbs/boot/cm4/config.txt`
- Create: `packages/uconsole-dtbs/boot/cm4/cmdline.txt`

Shared package. CM5 boot templates added in Task 13.

- [ ] **Step 1: Create `boot/cm4/config.txt`**

Create `packages/uconsole-dtbs/boot/cm4/config.txt`:

```
# uConsole CM4 boot configuration
# Installed by pacman package uconsole-dtbs.

arm_64bit=1
enable_uart=1
kernel=kernel8.img

# DSI display (5" 720x1280 uConsole panel)
dtoverlay=uconsole-base

# Enable I2C for the keyboard + PMIC
dtparam=i2c_arm=on,i2c_arm_baudrate=400000

# Enable SPI for the optional LoRa expansion (harmless if absent)
dtparam=spi=on

# Audio (WM8960 on I2S)
dtparam=audio=off
dtoverlay=wm8960-soundcard

# HDMI fallback — keep enabled so the user can plug into a TV for recovery
dtparam=i2c_vc=on
hdmi_force_hotplug=1
```

- [ ] **Step 2: Create `boot/cm4/cmdline.txt`**

Create `packages/uconsole-dtbs/boot/cm4/cmdline.txt`:

```
root=LABEL=ROOT rw rootwait console=serial0,115200 console=tty1 fsck.repair=yes
```

(Single line, no trailing newline by convention.)

- [ ] **Step 3: Create a minimal stub overlay `overlays/uconsole-base.dts`**

Create `packages/uconsole-dtbs/overlays/uconsole-base.dts`. **This is a placeholder** — the real overlay's panel / keyboard binding come from kernel patches merged in Task 3. For now, ship a no-op overlay so the package compiles and installs:

```
/dts-v1/;
/plugin/;

/ {
    compatible = "brcm,bcm2711";

    // Placeholder overlay. Replaced in Plan 4 with full panel + keyboard
    // bindings once the kernel driver DT bindings are finalized.

    fragment@0 {
        target-path = "/";
        __overlay__ {
            uconsole-base-marker = "placeholder";
        };
    };
};
```

- [ ] **Step 4: Write the PKGBUILD**

Create `packages/uconsole-dtbs/PKGBUILD`:

```bash
# Maintainer: arch-uconsole <noreply@example.org>
#
# uconsole-dtbs — uConsole device tree overlays + boot config templates.
# Shared between CM4 and CM5. Compiles .dts overlays to .dtbo at build
# time; boot/<target>/{config,cmdline}.txt templates installed to
# /boot/<target>/ for the bootstrap script to symlink on first boot.

pkgname=uconsole-dtbs
pkgver=20260415
pkgrel=1
pkgdesc="uConsole device tree overlays and boot config templates (CM4/CM5)"
arch=('any')
url="https://github.com/TheZacillac/arch-uconsole"
license=('MIT')
makedepends=('dtc')

source=(
    'overlays/uconsole-base.dts'
    'boot/cm4/config.txt'
    'boot/cm4/cmdline.txt'
)
sha256sums=(
    'SKIP' 'SKIP' 'SKIP'
)

build() {
    cd "${srcdir}"
    echo "::> compiling DTB overlays"
    for dts in uconsole-base.dts; do
        out="${dts%.dts}.dtbo"
        dtc -@ -I dts -O dtb -o "${out}" "${dts}"
        ls -la "${out}"
    done
}

package() {
    cd "${srcdir}"

    # Compiled overlays shared across all CMs:
    install -dm755 "${pkgdir}/boot/overlays"
    install -Dm644 uconsole-base.dtbo "${pkgdir}/boot/overlays/"

    # Per-CM boot config templates (bootstrap picks the right one):
    install -Dm644 "${startdir}/boot/cm4/config.txt" \
        "${pkgdir}/usr/share/uconsole/boot/cm4/config.txt"
    install -Dm644 "${startdir}/boot/cm4/cmdline.txt" \
        "${pkgdir}/usr/share/uconsole/boot/cm4/cmdline.txt"
}
```

Note: sources reference local files. Because the boot templates live under subdirectories `boot/cm4/`, makepkg flattens them into `srcdir` by the filename only (`config.txt`, `cmdline.txt`). The package uses `${startdir}/boot/cm4/...` to read the original tree for install.

- [ ] **Step 5: Commit**

```bash
git add packages/uconsole-dtbs/
git commit -m "feat(uconsole-dtbs): initial package (CM4 boot config + stub base overlay)"
git push
```

- [ ] **Step 6: Watch CI build the new package**

```bash
gh run watch --exit-status -R TheZacillac/arch-uconsole
```

Expected: `uconsole-dtbs` shows up in the matrix and builds successfully.

---

## Task 9: Bootstrap script — detect CM and install the right kernel stack

**Files:**
- Modify: `bootstrap/uconsole-bootstrap`

Extend the existing script so that after `pacman -Sy`, it detects whether we're on CM4 or CM5 and `pacman -S`'s the matching kernel + firmware + dtbs.

- [ ] **Step 1: Add CM detection + kernel install to the script**

Edit `bootstrap/uconsole-bootstrap`. Replace the `main()` function and add a `detect_cm()` + `install_kernel_stack()`:

```sh
detect_cm() {
    # Returns: "cm4" | "cm5" | "" (unknown — running in container / non-uConsole)
    local model=""
    if [ -r /proc/device-tree/model ]; then
        model=$(tr -d '\0' < /proc/device-tree/model)
    fi
    case "$model" in
        *"Compute Module 4"*) echo cm4 ;;
        *"Compute Module 5"*) echo cm5 ;;
        *) echo "" ;;
    esac
}

install_kernel_stack() {
    local cm="$1"
    if [ -z "$cm" ]; then
        log "no ClockworkPi / CM detected; skipping kernel install"
        log "(you can re-run this script on-device to install the kernel)"
        return 0
    fi
    log "detected compute module: ${cm}"
    log "installing linux-uconsole-${cm}, uconsole-firmware-${cm}, uconsole-dtbs"
    pacman -S --noconfirm --needed \
        "linux-uconsole-${cm}" \
        "uconsole-firmware-${cm}" \
        uconsole-dtbs

    # Copy per-CM boot templates into /boot/ if not already present.
    local tmpl_dir="/usr/share/uconsole/boot/${cm}"
    if [ -d "${tmpl_dir}" ]; then
        for f in config.txt cmdline.txt; do
            if [ ! -e "/boot/${f}" ]; then
                log "installing default /boot/${f} from ${tmpl_dir}"
                install -Dm644 "${tmpl_dir}/${f}" "/boot/${f}"
            else
                log "/boot/${f} already exists; leaving untouched"
            fi
        done
    fi
}

main() {
    require_root
    add_repo
    trust_key
    refresh
    install_kernel_stack "$(detect_cm)"
    finalize
    log "done."
}
```

Full script (replace entire file):

```sh
#!/bin/sh
# uconsole-bootstrap — add the [uconsole] pacman repo, trust its signing
# key, and (when running on real uConsole hardware) install the matching
# kernel + firmware + dtbs packages.
#
# Usage (as root, on an Arch Linux ARM install):
#   uconsole-bootstrap
#
# Running on non-uConsole hardware (e.g., in a container) is safe — the
# repo is still added, but the kernel stack install is skipped.

set -eu

REPO_URL_DEFAULT="https://TheZacillac.github.io/arch-uconsole/\$arch"
KEY_URL_DEFAULT="https://TheZacillac.github.io/arch-uconsole/aarch64/uconsole-signing.asc"
KEY_FPR_DEFAULT="4B13F2558FF8B2D34E20F740A1A18BAF0DC6E364"
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
    # shellcheck disable=SC2064
    trap "rm -f '$tmp'" EXIT
    curl -fsSL "$KEY_URL_DEFAULT" -o "$tmp"
    pacman-key --add "$tmp"
    pacman-key --lsign-key "$KEY_FPR_DEFAULT"
}

refresh() {
    pacman -Sy
}

detect_cm() {
    local model=""
    if [ -r /proc/device-tree/model ]; then
        model=$(tr -d '\0' < /proc/device-tree/model)
    fi
    case "$model" in
        *"Compute Module 4"*) echo cm4 ;;
        *"Compute Module 5"*) echo cm5 ;;
        *) echo "" ;;
    esac
}

install_kernel_stack() {
    cm="$1"
    if [ -z "$cm" ]; then
        log "no ClockworkPi / CM detected; skipping kernel install"
        log "(re-run this script on real hardware to install the kernel)"
        return 0
    fi
    log "detected compute module: ${cm}"
    log "installing linux-uconsole-${cm}, uconsole-firmware-${cm}, uconsole-dtbs"
    pacman -S --noconfirm --needed \
        "linux-uconsole-${cm}" \
        "uconsole-firmware-${cm}" \
        uconsole-dtbs

    tmpl_dir="/usr/share/uconsole/boot/${cm}"
    if [ -d "${tmpl_dir}" ]; then
        for f in config.txt cmdline.txt; do
            if [ ! -e "/boot/${f}" ]; then
                log "installing default /boot/${f} from ${tmpl_dir}"
                install -Dm644 "${tmpl_dir}/${f}" "/boot/${f}"
            else
                log "/boot/${f} already exists; leaving untouched"
            fi
        done
    fi
}

finalize() {
    mkdir -p "$SENTINEL_DIR"
    {
        printf 'bootstrap_version=2\n'
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
    install_kernel_stack "$(detect_cm)"
    finalize
    log "done."
}

main "$@"
```

- [ ] **Step 2: Shellcheck**

```bash
shellcheck bootstrap/uconsole-bootstrap || true
```

Acceptable: warnings about `local` in POSIX `sh` (the shebang is `/bin/sh` but most systems alias it to bash or dash; `local` works in both for this script's simple usage). If you want to be strict, remove `local` keywords — the vars are scoped by convention.

- [ ] **Step 3: Commit**

```bash
git add bootstrap/uconsole-bootstrap
git commit -m "feat(bootstrap): detect CM4/CM5 and install matching kernel stack"
git push
```

---

## Task 10: Hardware test procedure for CM4

**Files:**
- Create: `docs/hardware-test-cm4.md`

User-facing document for physically flashing + booting a CM4 uConsole.

- [ ] **Step 1: Write the procedure**

Create `docs/hardware-test-cm4.md`:

```markdown
# Hardware test — ClockworkPi uConsole CM4

End-to-end verification that `linux-uconsole-cm4` + `uconsole-firmware-cm4`
+ `uconsole-dtbs` boot a real CM4 uConsole to a TTY.

## Requirements

- ClockworkPi uConsole with CM4 installed
- microSD card (≥ 4 GB)
- A working Linux host with `pacstrap`, `arch-install-scripts`,
  `qemu-user-static`, and `parted` installed
- Network access (to reach `thezacillac.github.io` and an Arch Linux ARM
  mirror)

## Procedure

1. **Flash a fresh ALARM rootfs to the SD card.**

   ```bash
   # Replace /dev/sdX with your SD card (lsblk to confirm).
   export SD=/dev/sdX
   sudo parted -s "$SD" mklabel msdos
   sudo parted -s "$SD" mkpart primary fat32 1MiB 257MiB
   sudo parted -s "$SD" mkpart primary ext4 257MiB 100%
   sudo parted -s "$SD" set 1 boot on
   sudo mkfs.vfat -F32 -n BOOT "${SD}1"
   sudo mkfs.ext4 -L ROOT "${SD}2"

   sudo mkdir -p /mnt/uconsole-root /mnt/uconsole-boot
   sudo mount "${SD}2" /mnt/uconsole-root
   sudo mkdir -p /mnt/uconsole-root/boot
   sudo mount "${SD}1" /mnt/uconsole-root/boot

   curl -fsSLO http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
   sudo bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C /mnt/uconsole-root
   sync
   ```

2. **Copy the bootstrap script + qemu-aarch64-static into the chroot.**

   ```bash
   # Assumes you have the arch-uconsole repo cloned at
   # /home/zac/Projects/arch_build (adjust if elsewhere):
   REPO=/home/zac/Projects/arch_build

   sudo cp /usr/bin/qemu-aarch64-static /mnt/uconsole-root/usr/bin/
   sudo cp "$REPO/bootstrap/uconsole-bootstrap" /mnt/uconsole-root/usr/local/bin/
   sudo chmod +x /mnt/uconsole-root/usr/local/bin/uconsole-bootstrap
   ```

3. **Chroot in and run the bootstrap.**

   ```bash
   sudo arch-chroot /mnt/uconsole-root /bin/bash <<'CHROOT'
   pacman-key --init
   pacman-key --populate archlinuxarm
   /usr/local/bin/uconsole-bootstrap
   CHROOT

   # The chroot cannot detect CM4 (we're running on an x86_64 host under
   # qemu emulation), so the bootstrap skips the kernel install.
   # Install the kernel stack manually against our CM4 target:
   sudo arch-chroot /mnt/uconsole-root pacman -S --noconfirm \
       linux-uconsole-cm4 uconsole-firmware-cm4 uconsole-dtbs
   sudo arch-chroot /mnt/uconsole-root bash -c '
       cp /usr/share/uconsole/boot/cm4/config.txt /boot/
       cp /usr/share/uconsole/boot/cm4/cmdline.txt /boot/
   '
   ```

4. **Unmount + eject.**

   ```bash
   sudo umount /mnt/uconsole-root/boot /mnt/uconsole-root
   sync
   ```

5. **Insert SD into uConsole CM4. Power on.**

6. **Expected:** within 30 seconds, the 5" display shows a TTY login
   prompt (or at minimum kernel boot messages). ALARM default credentials:
   user `alarm`, password `alarm`.

7. **Verify the kernel is ours:**

   ```
   login: alarm
   password: alarm
   $ uname -a
   Linux alarm 6.12.0-<...>-uconsole-cm4 #1 SMP PREEMPT ... aarch64 GNU/Linux
   $ cat /var/lib/uconsole/bootstrap.done
   bootstrap_version=2
   ...
   ```

## Failure modes + debugging

- **Rainbow splash, hangs.** Firmware loaded but kernel didn't. Check
  `/boot/kernel8.img` exists and is our build (size ~20 MB compressed).
- **No display.** DSI panel overlay didn't bind. Connect via UART
  (GPIO 14/15 at 115200 baud) to get console output.
- **Black screen, beeping from speaker.** `config.txt` not being read.
  Verify `/boot/config.txt` has `kernel=kernel8.img`.
- **Kernel panic "unable to mount root."** `cmdline.txt` has wrong
  `root=LABEL=ROOT`. Our `mkfs.ext4 -L ROOT` above ensures the label
  matches. Re-check with `sudo blkid`.
```

- [ ] **Step 2: Commit**

```bash
git add docs/hardware-test-cm4.md
git commit -m "docs: hardware test procedure for CM4"
git push
```

---

## Task 11: Execute the CM4 hardware test

This task is **human-only and requires physical CM4 hardware**.

- [ ] **Step 1: Run the procedure in `docs/hardware-test-cm4.md`**

Follow the steps end-to-end. Report back:
- Did it boot?
- Output of `uname -a`
- Output of `ls /boot/`
- Any failure mode encountered

If the kernel fails to boot: capture UART output (or serial adapter log), paste here, and I'll triage.

If it boots but the display is dead / keyboard doesn't work / audio silent: that's expected for Plan 2 scope. The DSI panel and keyboard drivers land in Plan 4 (hardware support packages). For Plan 2 we only need `uname -a` to return our kernel version.

- [ ] **Step 2: If the test passed — commit a "tested on hardware" note**

```bash
cat >> docs/hardware-test-cm4.md <<'EOF'

## Test runs

- **2026-04-DD:** CM4 boots to TTY on real hardware. `uname -a` reports
  `<kernel version>`. (Display/keyboard/audio not yet wired — expected
  for Plan 2 scope.)
EOF

git add docs/hardware-test-cm4.md
git commit -m "docs: record successful CM4 hardware boot"
git push
```

---

## Task 12: Clone CM4 package dir to CM5

**Files:**
- Create: `packages/linux-uconsole-cm5/` (copied from cm4, tweaked)
- Create: `packages/uconsole-firmware-cm5/`

Shortcut: duplicate the CM4 scaffold for CM5. Many steps parallel Task 2-7 but with different pin values.

- [ ] **Step 1: Copy cm4 package directory to cm5**

```bash
cp -r packages/linux-uconsole-cm4 packages/linux-uconsole-cm5

# Rename pkgname references inside PKGBUILD + install hook:
sed -i 's|linux-uconsole-cm4|linux-uconsole-cm5|g; s|bcm2711|bcm2712|g' \
    packages/linux-uconsole-cm5/PKGBUILD \
    packages/linux-uconsole-cm5/linux.install

# CM5 uses kernel_2712.img, not kernel8.img:
sed -i 's|/boot/kernel8.img|/boot/kernel_2712.img|g' \
    packages/linux-uconsole-cm5/linux.install

# Description update:
sed -i 's|"Linux kernel for ClockworkPi uConsole CM4"|"Linux kernel for ClockworkPi uConsole CM5"|' \
    packages/linux-uconsole-cm5/PKGBUILD

# CM5 DTB glob: bcm2712-rpi-cm5*.dtb, not bcm2711-rpi-cm4*.dtb
sed -i 's|bcm2711-rpi-cm4|bcm2712-rpi-cm5|g' packages/linux-uconsole-cm5/PKGBUILD
```

- [ ] **Step 2: Delete cm4 patches — they're CM4-specific**

```bash
rm packages/linux-uconsole-cm5/patches/*.patch
touch packages/linux-uconsole-cm5/patches/.keep
```

- [ ] **Step 3: Harvest CM5 patches (same procedure as Task 3, CM5 branch)**

Repeat Task 3 Steps 1–7 against `clockworkpi/linux` branch for CM5.
Substitute the CM5 branch name and SHA from `docs/upstream-refs.md`.
Place harvested patches in `packages/linux-uconsole-cm5/patches/`.

If ClockworkPi's CM5 branch is based on an older RPi kernel (e.g.,
`rpi-6.10.y`) because CM5 support is newer and less upstreamed: update
the `_rpibranch=` and `_rpisha=` in `packages/linux-uconsole-cm5/PKGBUILD`
to match. Document the mismatch in the PKGBUILD comment. Everything else
stays the same.

- [ ] **Step 4: Regenerate CM5 kernel config**

Repeat Task 4 Steps 1–4 but with `bcm2712_defconfig` as the base and
saving to `packages/linux-uconsole-cm5/config`:

```bash
# In your scratch tree:
cd /tmp/kernel-harvest/rpi
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- bcm2712_defconfig
# apply cm5 patches
for p in /home/zac/Projects/arch_build/packages/linux-uconsole-cm5/patches/*.patch; do
    git apply "$p" || true
done
# ... scripts/config enable knobs as in Task 4
cp .config /home/zac/Projects/arch_build/packages/linux-uconsole-cm5/config
```

- [ ] **Step 5: Duplicate and retarget `uconsole-firmware-cm5`**

```bash
cp -r packages/uconsole-firmware-cm4 packages/uconsole-firmware-cm5
sed -i 's|uconsole-firmware-cm4|uconsole-firmware-cm5|g; s|CM4|CM5|g' \
    packages/uconsole-firmware-cm5/PKGBUILD
```

The CM5 firmware bootchain is different — VideoCore VII doesn't use
`start4.elf`. Instead it uses the EEPROM firmware baked into the CM5
itself, plus a different set of start/fixup files. Adjust the `files=()`
array in `packages/uconsole-firmware-cm5/PKGBUILD` to:

```bash
    local files=(
        # CM5 / Pi 5-era bootchain:
        bootcode.bin
        fixup_2712.dat
        start_2712.elf
        # (Keep extras if upstream rpi-firmware ships them)
    )
```

Actual filenames verified by checking what's in the pinned `rpi-firmware`
tree:

```bash
ls /tmp/rpi-fw-check/ | grep -E '2712|start|fixup'
```

Update the array to match.

- [ ] **Step 6: Commit CM5 scaffold**

```bash
git add packages/linux-uconsole-cm5/ packages/uconsole-firmware-cm5/
git commit -m "feat(cm5): clone CM4 package scaffold for CM5 (2712 / VideoCore VII)"
git push
```

- [ ] **Step 7: Watch CI**

```bash
gh run watch --exit-status -R TheZacillac/arch-uconsole
```

Expected: four kernel-matrix jobs now (cm4, cm5 + their headers split),
plus firmware-cm4, firmware-cm5, uconsole-dtbs, uconsole-hello. All
should pass. CM5 is riskier — if the config or patch series doesn't
cleanly apply, triage with `gh run view --log-failed`.

---

## Task 13: Add CM5 boot config templates to `uconsole-dtbs`

**Files:**
- Create: `packages/uconsole-dtbs/boot/cm5/config.txt`
- Create: `packages/uconsole-dtbs/boot/cm5/cmdline.txt`
- Modify: `packages/uconsole-dtbs/PKGBUILD`

Now that CM5 has a kernel, add its boot templates to the shared dtbs
package.

> **Note (Task 8 learned):** makepkg's local `source=()` entries must be
> bare filenames in the PKGBUILD directory — subpaths like
> `'overlays/foo.dts'` fail with "not found in the build directory". The
> Task 8 implementer flattened the layout accordingly, so CM5 just adds
> two more bare files (`config-cm5.txt`, `cmdline-cm5.txt`) alongside
> the existing flat files.

- [ ] **Step 1: Create `config-cm5.txt`**

Create `packages/uconsole-dtbs/config-cm5.txt`:

```
# uConsole CM5 boot configuration
# Installed by pacman package uconsole-dtbs.

arm_64bit=1
enable_uart=1
kernel=kernel_2712.img

dtoverlay=uconsole-base
dtparam=i2c_arm=on,i2c_arm_baudrate=400000
dtparam=spi=on
dtparam=audio=off
dtoverlay=wm8960-soundcard
dtparam=i2c_vc=on
hdmi_force_hotplug=1
```

- [ ] **Step 2: Create `cmdline-cm5.txt`**

Create `packages/uconsole-dtbs/cmdline-cm5.txt` (one line, NO trailing
newline — same as CM4):

```
root=LABEL=ROOT rw rootwait console=serial0,115200 console=tty1 fsck.repair=yes
```

(Identical to CM4 for now; differences may emerge during testing.)

- [ ] **Step 3: Update the dtbs PKGBUILD to ship both sets**

Modify `packages/uconsole-dtbs/PKGBUILD`. Add the two CM5 filenames to
the `source=()` array and let `package()` loop over both CMs:

```bash
source=(
    'uconsole-base.dts'
    'config-cm4.txt'
    'cmdline-cm4.txt'
    'config-cm5.txt'
    'cmdline-cm5.txt'
)
sha256sums=(
    'SKIP' 'SKIP' 'SKIP' 'SKIP' 'SKIP'
)
```

Update `package()`:

```bash
package() {
    cd "${srcdir}"

    install -dm755 "${pkgdir}/boot/overlays"
    install -Dm644 uconsole-base.dtbo "${pkgdir}/boot/overlays/uconsole-base.dtbo"

    for cm in cm4 cm5; do
        install -Dm644 "config-${cm}.txt" \
            "${pkgdir}/usr/share/uconsole/boot/${cm}/config.txt"
        install -Dm644 "cmdline-${cm}.txt" \
            "${pkgdir}/usr/share/uconsole/boot/${cm}/cmdline.txt"
    done
}
```

- [ ] **Step 4: Commit**

```bash
git add packages/uconsole-dtbs/
git commit -m "feat(uconsole-dtbs): add CM5 boot config templates"
git push
```

- [ ] **Step 5: Watch CI**

```bash
gh run watch --exit-status -R TheZacillac/arch-uconsole
```

---

## Task 14: Hardware test procedure for CM5

**Files:**
- Create: `docs/hardware-test-cm5.md`

- [ ] **Step 1: Write the CM5 procedure**

Create `docs/hardware-test-cm5.md`. Content is identical to
`docs/hardware-test-cm4.md` with these substitutions:

- `CM4` → `CM5`
- `cm4` → `cm5`
- `linux-uconsole-cm4` → `linux-uconsole-cm5`
- `uconsole-firmware-cm4` → `uconsole-firmware-cm5`
- `kernel8.img` → `kernel_2712.img`

Easiest mechanical approach:

```bash
sed \
  -e 's|CM4|CM5|g' \
  -e 's|cm4|cm5|g' \
  -e 's|kernel8\.img|kernel_2712.img|g' \
  docs/hardware-test-cm4.md > docs/hardware-test-cm5.md
```

Read it after; fix any awkward substitutions (e.g., "ClockworkPi CM5 CM5" or doubled references).

Also add at the top, under the title:

```markdown
> **Note:** CM5 support is newer and less mature than CM4. If this
> procedure fails at any step, collect UART output and file a detailed
> report — expect teething issues for the first few iterations.
```

- [ ] **Step 2: Commit**

```bash
git add docs/hardware-test-cm5.md
git commit -m "docs: hardware test procedure for CM5"
git push
```

---

## Task 15: Execute the CM5 hardware test

This task is **human-only and requires physical CM5 hardware**.

- [ ] **Step 1: Run the procedure in `docs/hardware-test-cm5.md`**

Same shape as Task 11. Report back `uname -a` + any failures.

- [ ] **Step 2: Record result**

Append to `docs/hardware-test-cm5.md`:

```markdown

## Test runs

- **2026-04-DD:** [PASS/FAIL]. `uname -a` reports <version>. Notes: <...>
```

Commit.

```bash
git add docs/hardware-test-cm5.md
git commit -m "docs: record CM5 hardware test result"
git push
```

---

## Task 16: Release v0.2.0

**Files:**
- Modify: `CHANGELOG.md`

- [ ] **Step 1: Move [Unreleased] to [0.2.0]**

Edit `CHANGELOG.md` top section to:

```markdown
# Changelog

All notable changes documented here. Format follows Keep a Changelog.

## [Unreleased]

## [0.2.0] - 2026-04-DD

### Added
- `linux-uconsole-cm4` + `linux-uconsole-cm4-headers` PKGBUILDs —
  cross-compiled from `raspberrypi/linux` rpi-6.12.y with a curated
  ClockworkPi patch series.
- `linux-uconsole-cm5` + `linux-uconsole-cm5-headers` PKGBUILDs —
  same pattern for CM5 (VideoCore VII / bcm2712).
- `uconsole-firmware-cm4` + `uconsole-firmware-cm5` — pinned RPi boot
  firmware (bootcode.bin, start*.elf, fixup*.dat) per compute module.
- `uconsole-dtbs` — shared DTB overlays + `config.txt` / `cmdline.txt`
  templates for CM4 and CM5.
- `bootstrap/uconsole-bootstrap` now detects compute module via
  `/proc/device-tree/model` and installs the matching kernel stack.
- `docs/upstream-refs.md` recording upstream kernel and firmware git
  pins.
- `docs/hardware-test-cm4.md` + `docs/hardware-test-cm5.md` covering
  flash-and-boot verification procedures.

### Changed
- `build-packages.yml` now uses a matrix strategy — one job per
  PKGBUILD — and installs the aarch64 cross-compile toolchain so kernel
  builds complete in ~8 minutes on x86_64 rather than ~30 minutes on
  native aarch64.
```

(Today's date in place of `2026-04-DD`.)

- [ ] **Step 2: Tag and push**

```bash
git add CHANGELOG.md
git commit -m "docs: release 0.2.0 (kernel + firmware + DTBs for CM4 and CM5)"
git tag -a v0.2.0 -m "Plan 2 — boot chain for CM4 and CM5"
git push origin main v0.2.0
```

---

## Done

**Plan 2 is complete when:**

1. `linux-uconsole-cm4` installs cleanly on a scratch ALARM rootfs and
   boots on a physical CM4 uConsole to a TTY prompt with `uname -a`
   reporting our kernel version.
2. Same for `linux-uconsole-cm5` on CM5 hardware (or documented as
   beta-with-known-issues if hardware reveals problems).
3. The `[uconsole]` pacman repo publishes all new packages with valid
   signatures; `pacman -Syu` on an existing ALARM install pulls them.
4. `v0.2.0` is tagged and CI is green on `main`.

**Next:** Plan 3 — bootstrap enhancements + image builder (`mkimage.sh`).
From here on, users get a flashable `.img.xz` rather than assembling the
rootfs themselves.
