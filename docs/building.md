# Building and flashing

This page shows how to build the port with pmbootstrap and put it on the phone.
It covers the first install and the normal update cycle with `tools/avicii-deploy`.
For the checks after the first boot, see [Reinstall from scratch](reinstall.md).

## Before you start

- Unlocked bootloader.
- A prepared PC: packages, pmbootstrap 3.x, the pmaports fork, `pmbootstrap init`,
  `fastboot` without `sudo` — all in [Preparing the PC](setup-pc.md).
- **pmbootstrap needs `sudo` with a real terminal (TTY).** It mounts images and
  uses loop devices. Run it in your own terminal, not from a script without a TTY.
  If the `sudo` timeout runs out during a long kernel build, the next step can fail
  (this happened once: `losetup` returned an error instead of JSON).
- `fastboot` itself works without `sudo` if your udev rules allow it.

To enter fastboot mode: power the phone off, then hold **Power + Volume Up**.

## 1. Get the packages

The packages live in the `oneplus-avicii` branch of
[ScYurii/pmaports](https://github.com/ScYurii/pmaports/tree/oneplus-avicii)
(`device/downstream/`). How to set up the checkout and `pmbootstrap init`
(including `avicii-server` in `extra_packages`): [Preparing the PC](setup-pc.md#5-pmaports-with-this-port).

Before a build, check that pmaports is on the right branch:

```sh
# should print oneplus-avicii
# должно быть oneplus-avicii
git -C "$(pmbootstrap config aports)" branch --show-current
```

## 2. Build

The kernel is built with clang. The cross toolchain is installed in the native chroot.

```sh
pmbootstrap checksum linux-oneplus-avicii   # only after you changed source files
pmbootstrap build --arch aarch64 linux-oneplus-avicii
pmbootstrap build --arch aarch64 device-oneplus-avicii
```

You do not have to build the other packages by hand: `pmbootstrap install`
builds missing dependencies (`avicii-wlan`, `temp/logbookd`) and the packages from
`extra_packages` (`avicii-server`).

## 3. Install and export

```sh
pmbootstrap install
pmbootstrap export
```

`install` creates the rootfs image and asks for the password of the user `user`.
`export` puts `boot.img` and the rootfs image into `/tmp/postmarketOS-export/`.

**`boot.img` and the rootfs are a pair.** Each `pmbootstrap install` creates new
filesystem UUIDs, and the root UUID is written into the kernel command line inside
`boot.img`. After a new `install`, always flash both. If they do not match, the boot
stops with "root partition not found".

## 4. Disable verified boot (once)

Without this step the bootloader silently refuses the flashed image and goes to
recovery. `fastboot boot` does not check the signature, so a test boot works,
and this hides the problem.

`avbtool` is installed in pmbootstrap's native chroot, not on the PC, so the image is
made there and then copied out:

```sh
# install avbtool in the native chroot and make an empty vbmeta with verification off
# поставить avbtool в native chroot и сделать пустой vbmeta с выключенной проверкой
pmbootstrap chroot -- apk add android-tools-avbtool
pmbootstrap chroot -- avbtool make_vbmeta_image --flags 2 --padding_size 4096 --output /vbmeta.img

# the chroot root is a directory in the work dir; copy the image out of it
# корень chroot — каталог в рабочем каталоге; копируем образ оттуда
cp "$(pmbootstrap config work)/chroot_native/vbmeta.img" /tmp/vbmeta.img

# flash both slots (phone in fastboot mode)
# прошить оба слота (телефон в режиме fastboot)
fastboot flash vbmeta_a /tmp/vbmeta.img
fastboot flash vbmeta_b /tmp/vbmeta.img
```

More details: [A/B slots and verified boot](ab-slots.md).

## 5. Flash

The rootfs goes to `userdata` (pmbootstrap's default for fastboot devices; deviceinfo
asks for a sparse image). The boot image goes to **both** slots.

```sh
pmbootstrap flasher flash_rootfs
fastboot flash boot_a /tmp/postmarketOS-export/boot.img
fastboot flash boot_b /tmp/postmarketOS-export/boot.img
fastboot reboot
```

**Why both slots?** If the bootloader switches to the other slot, it would start the
Android kernel. That kernel cannot read the postmarketOS `userdata` and the phone loops.
See [A/B slots](ab-slots.md).

Note: installing a new kernel package *on the phone* rebuilds `/boot/boot.img` there,
but does **not** write it to the boot partition. Flashing is always done from the PC.

## Updates with avicii-deploy

`tools/avicii-deploy` does the full update cycle in one command.
It uses `/tmp/postmarketOS-export` and the phone at `172.16.42.1` (USB network).

```sh
avicii-deploy             # kernel + device package
avicii-deploy kernel      # kernel only
avicii-deploy device      # device package only (cmdline, deviceinfo), fast
avicii-deploy --no-flash  # build and check, do not flash
avicii-deploy --no-build  # use packages already built (e.g. after a failed flash)
```

What it does, step by step:

1. **`pmbootstrap shutdown`** — removes stale mounts from earlier runs.
2. **`pmbootstrap checksum` + `pmbootstrap build --force`** for each package.
3. **`apk add -u` in the local rootfs chroot.** Without this, `export` makes an image
   with the *old* kernel.
4. **`pmbootstrap sideload`** to the phone, if it answers ping. The kernel signs its
   modules with a new key on every build. Old modules on the phone then fail with
   `Required key not available`, so the new package must be installed on the phone too.
   Sideload needs `openssh-sftp-server` on the phone.
5. **`pmbootstrap export`** — builds the boot image.
6. **Image checks** before flashing. The script stops if one fails:
   - size is not more than **96 MB** (the boot partition limit);
   - boot image header is **v2**;
   - cmdline contains `console=tty0` and `rd.info`, and does **not** contain `splash`;
   - it also prints the kernel build date and warns if the image is older than 30 minutes
     (a sign that the rebuild did not reach the image).
7. **Waits for fastboot** (up to 4 minutes) — put the phone into fastboot mode.
8. **Flashes `boot_a` and `boot_b`** and reboots.
9. **Waits for the network and SSH**, then prints the kernel version, load average and
   processes in `D` state.

## If the phone does not boot

- If the initramfs fails, it shows its logs as a USB drive named `PMOS_LOGS`.
- If the system boots but SSH does not work, boot the same image with a debug shell,
  without flashing it:

  ```sh
  fastboot --cmdline "<cmdline from boot.img> pmos.debug-shell" boot /tmp/postmarketOS-export/boot.img
  ```

  The shell appears on the USB serial port `/dev/ttyACM0`. There, `mount_subpartitions`
  makes the rootfs available as `/dev/loop0p2`. To get boot logs from OpenRC,
  set `rc_logger="YES"` in `/etc/rc.conf`; it writes `/var/log/rc.log`.
