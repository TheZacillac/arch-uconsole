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
