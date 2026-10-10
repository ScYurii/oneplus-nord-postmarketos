# Tools

## `avicii-deploy`

One command for the usual kernel or device update: build, check the image,
flash both slots and show the state of the phone after boot.

```sh
avicii-deploy            # kernel and device package
avicii-deploy kernel     # kernel only
avicii-deploy device     # device package only (cmdline, deviceinfo)
avicii-deploy --no-flash # build and check, but do not flash
avicii-deploy --no-build # use packages already built (after a failed flash)
```

What it does:

1. `pmbootstrap checksum` and `pmbootstrap build --force` for each package.
2. Updates the packages in the local rootfs chroot. Without this,
   `pmbootstrap export` builds the image with the old kernel.
3. If the phone answers on USB (`172.16.42.1`), installs the packages on it
   with `pmbootstrap sideload`, so the kernel modules match the new kernel.
4. `pmbootstrap export`, then checks `boot.img` before flashing:
   size (the boot partition is 96 MB), kernel build date, header version,
   and the command line (`console=tty0` and `rd.info` present, no `splash`).
5. Waits for the phone in fastboot (power off, then Power + Volume Up).
6. Flashes **both** `boot_a` and `boot_b`, see [A/B slots](../docs/ab-slots.md).
7. Reboots, waits for the network and SSH, prints the kernel version,
   the load average and processes stuck in the `D` state.

`pmbootstrap` asks for `sudo`, so run the script in a terminal where sudo can
ask for your password.

See [Building and flashing](../docs/building.md) for the full process.
