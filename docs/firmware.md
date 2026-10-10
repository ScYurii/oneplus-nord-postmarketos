# Vendor firmware

The modem, DSPs, video decoder and Wi-Fi need closed firmware files from Qualcomm and
OnePlus. postmarketOS does not ship them. They already exist on the phone, in the
Android partitions. This page explains how the port gets them, why the standard tool
was replaced, and why one missing file caused random reboots.

## Back up EFS and persist first

Before you experiment, make a copy of these partitions:

- `modemst1`, `modemst2`, `fsg`, `fsc` — the modem EFS;
- `persist` — per-device calibration, including the Wi-Fi calibration (see below).

If they are lost or damaged, Wi-Fi and the modem can stop working for good, and there is
no other source for these files. Keep the backup off the phone and save checksums with it.

One way to do it over SSH (needs `sudo` without a password prompt on the phone):

```sh
for p in modemst1 modemst2 fsg fsc persist; do
	ssh user@172.16.42.1 "sudo dd if=/dev/disk/by-partlabel/$p" > "$p.img"
done
sha256sum *.img > SHA256SUMS
```

> TODO: the exact backup method used for this port was not recorded; the loop above is
> an example. Compare the checksums with `sha256sum` of the partitions on the phone.

The port protects EFS at runtime too: `rmtfs` runs with `-r`, so modem writes to EFS
stay in RAM and never reach the partitions.

## Where the firmware is

The vendor files are in these partitions: `modem_a`, `dsp_a`, `bluetooth_a`, `persist`
and `vendor_a`. `vendor_a` is inside the dynamic `super` partition, so it needs
`make-dynpart-mappings` before it can be mounted.

## Why not msm-firmware-loader

postmarketOS normally uses `msm-firmware-loader`. It mounts the partitions at every boot
and creates symlinks to the files. On this phone it did not work:

- it has a bug: it **mounts its tmpfs before it creates the directory** for it
  (`mount` before `mkdir`), so in `sysinit` it fails;
- drivers that start early did not find their files.

## avicii-firmware: copy once

The device package replaces it with **`avicii-firmware`**: an OpenRC service in the
`boot` runlevel that runs before `rmtfs-ro`, `tqftpserv-fw` and `avicii-wlan`.
It runs `/usr/libexec/avicii-firmware`, which does this:

1. If the marker file `/lib/firmware/postmarketos/.avicii-firmware` exists — stop, nothing to do.
2. If `/lib/firmware/postmarketos` already has **more than 100 files** — the firmware is
   already there (for example, copied by hand earlier). Create the marker and stop.
3. Otherwise create `/run/msm-firmware-loader` first (the workaround for the bug above),
   then run the stock `msm-firmware-loader.sh`. It mounts the partitions read-only.
4. Copy all files, with symlinks resolved, to `/lib/firmware/postmarketos`.
   This is the firmware path the kernel already uses.
5. Unmount the partitions and create the marker. If fewer than 100 files were found,
   no marker is created and it tries again on the next boot.

The device package also removes `msm-firmware-loader` from `sysinit`.

In a test, the copy from the partitions gave 342 files, the same byte for byte as the
working set. The Wi-Fi files from `persist` are not part of this; `avicii-wlan` copies them.

**After the first copy, reboot once.** Drivers that started before the copy did not get
their firmware. See [Reinstall from scratch](reinstall.md).

To force a new copy, remove the marker and the files, then reboot:

```sh
sudo rm -r /lib/firmware/postmarketos
sudo reboot
```

> TODO: this re-copy procedure follows from the script but was not tested on the phone.

## Random reboots every ~10 minutes

Early in the port the phone rebooted by itself about every 10 minutes. The cause was
missing **video firmware** (`venus`):

- the video driver failed with `Failed to locate venus.mdt`;
- `v4l_id` hung for 60 seconds on the broken device;
- the Qualcomm SoC escalated the failure into a full reboot.

After the firmware was copied into `/lib/firmware/postmarketos`, the reboots stopped.
This is the main reason the copy happens once, early, and does not depend on mounts.

## Wi-Fi calibration from persist

Wi-Fi needs per-device files. **They must come from `persist`**, not from the modem
partition:

| File | From `persist` | Copied to |
|---|---|---|
| `bdwlan.bin` (board data) | `/bdwlan.bin` | `/var/lib/wlfwd/board/` |
| `regdb.bin` (regulatory data) | `/regdb.bin` | `/var/lib/wlfwd/board/` |
| `WCNSS_qcom_cfg.ini` (driver config) | `/wlan/WCNSS_qcom_cfg.ini` | `/lib/firmware/postmarketos/wlan/qca_cld/` |
| `wlan_mac.bin` (factory MAC) | `/wlan_mac.bin` | `/lib/firmware/postmarketos/wlan/qca_cld/` |

The modem partition also has `bdwlan.bin` and `regdb.bin`, but with them the Wi-Fi
firmware **crashes right after the calibration report**. The `persist` copies hold the
OnePlus calibration for this unit; Android's `cnss-daemon` uses them too.

`avicii-wlan` copies these files on its first start. It mounts `persist` **read-only**
and does nothing if the files are already there. Without `WCNSS_qcom_cfg.ini` and
`wlan_mac.bin` the driver probe hangs. More in [Wi-Fi](wifi.md).
