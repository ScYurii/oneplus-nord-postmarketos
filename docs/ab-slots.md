# A/B slots and verified boot

The OnePlus Nord has two copies of the boot partitions: slot A (`boot_a`) and slot B
(`boot_b`). The bootloader decides which slot to start. Android tells the bootloader
"this slot works" on every boot. postmarketOS does not do this by default, and that
leads to a reboot loop. This page explains the problem and the fix. It also covers
`vbmeta`, the other thing that stops a flashed image from booting.

## The problem: the boot loop

- Every slot has a counter, `slot-retry-count`.
- **On every boot the bootloader decreases this counter by one.**
- Android resets it when the boot is good (it marks the slot as *successful*).
- When the counter reaches zero, the bootloader marks the slot as unbootable and
  **switches to the other slot**.
- The other slot still had the Android kernel. That kernel cannot read the
  postmarketOS `userdata`, so the phone reboots again and again (or ends in recovery).

So the phone works fine for a few boots and then "breaks" without any change.

## The fix: mark the slot successful on every boot

`qbootctl -m` marks the current slot as successfully booted. It must run on **every** boot.

The `qbootctl` package has its own service, but it is written as a daemon and does
not work with OpenRC (it stays `stopped`). The first workaround was a line
`qbootctl -m` in `/etc/local.d/qbootctl.start`. Now the device package has its own
OpenRC service, **`avicii-qbootctl`**, in the `boot` runlevel:

```sh
#!/sbin/openrc-run
description="Mark the current A/B slot as successfully booted"

depend() {
	need localmount
}

start() {
	ebegin "Marking boot slot as successful"
	qbootctl -m >/dev/null
	eend $?
}
```

The package `post-install` script adds it to `boot`. To check it on the phone:

```sh
rc-status boot | grep avicii-qbootctl   # should be "started"
sudo qbootctl                           # shows slots; current one: active, successful
```

If you still have `/etc/local.d/qbootctl.start` from an old setup, remove it.
The service replaces it.

## Insurance: flash both slots

Even with the service, flash `boot.img` to **both** slots:

```sh
fastboot flash boot_a /tmp/postmarketOS-export/boot.img
fastboot flash boot_b /tmp/postmarketOS-export/boot.img
```

Then a slot switch is harmless: both slots start the postmarketOS kernel.
`tools/avicii-deploy` always does this (see [Building and flashing](building.md)).

You can see the bootloader's view of the slots in fastboot mode:

```sh
fastboot getvar all 2>&1 | grep -E 'slot|current'
```

## Verified boot: vbmeta

The second trap is verified boot (AVB):

- `fastboot boot boot.img` starts an image **without** checking it. So a test boot works.
- A **flashed** image is checked. If the check fails, the bootloader goes to recovery
  without any clear error.

The fix is an empty `vbmeta` image with verification disabled (`--flags 2`),
written to both `vbmeta` slots. You need to do this only once, unless something
(for example, flashing Android again) overwrites `vbmeta`.

```sh
pmbootstrap chroot -- apk add android-tools-avbtool
avbtool make_vbmeta_image --flags 2 --padding_size 4096 --output vbmeta.img
fastboot flash vbmeta_a vbmeta.img
fastboot flash vbmeta_b vbmeta.img
```

## Summary

| Symptom | Cause | Fix |
|---|---|---|
| Works for a few boots, then loops or ends in recovery | `slot-retry-count` reached zero, bootloader switched slots | `avicii-qbootctl` service + `boot.img` in both slots |
| `fastboot boot` works, flashed image goes to recovery | Verified boot rejects the image | `vbmeta` with `--flags 2` in `vbmeta_a` and `vbmeta_b` |
| "root partition not found", but partitions are visible in `dmesg` | `boot.img` and rootfs from different installs (UUIDs) | Flash both from the same `pmbootstrap install` |

> TODO: the starting value of `slot-retry-count` and the number of boots before the
> switch were not recorded.
