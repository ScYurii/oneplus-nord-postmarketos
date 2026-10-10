# Wi-Fi without Android

Wi-Fi works on 2.4 GHz and 5 GHz. It does not use any Android code at runtime.
This page explains how it works, how to connect, and how to debug it.

All of this is packaged in `avicii-wlan` (see the [README](../README.md)).
If the package is installed, Wi-Fi starts at boot. You do not need to do anything by hand.

## The main idea: Wi-Fi lives inside the modem

On SM7250 the WLAN firmware does not run on its own chip. It runs **inside the modem**,
as a "protection domain" called `wlan_pd`. So to get Wi-Fi, we must first boot the modem,
and the modem expects several helper programs in userspace. On Android, these helpers
come from the vendor partition. Here we use open source versions from
[linux-msm](https://github.com/linux-msm), with small patches for the old 4.19 kernel.

The 4.19 kernel has two gaps that break the normal tools:

- There is **no in-kernel QRTR name service**.
- There is **no `/sys/class/remoteproc`**. Newer tools use it to find firmware files.

## The pieces

Each piece is required. If one is missing, Wi-Fi does not come up.

| Service | What it does | Why it is special here |
|---|---|---|
| `qrtr-ns` | QRTR name service: the "phone book" for services on the modem | Built from linux-msm/qrtr **v1.0**. qrtr 1.2 dropped it, because new kernels have it inside. 4.19 does not. |
| `rmtfs-ro` | Gives the modem its EFS partitions (`modemst1/2`, `fsg`, `fsc`) | Runs `rmtfs -P -r`: `-r` keeps the modem's writes **in RAM only**, EFS is never written. No `-s`: it needs remoteproc. Without rmtfs the modem crashes after about 65 s (`DOG detects stalled initialization`). |
| `tqftpserv-fw` | TFTP server; the modem downloads `wlanmdsp.mbn` from it | Patched: reads files from `TQFTPSERV_FW_DIR` (default `/lib/firmware/postmarketos`) instead of remoteproc. Logs every file as `served` or `missing`. |
| `pd-mapper` | Tells the modem which protection domains exist | Patched: reads `*.jsn` files from `PD_MAPPER_JSN_DIR`. Without it the modem never starts `wlan_pd`. |
| `avicii-wlan` | Boots the subsystems, runs `wlfwd`, turns the driver on | Our own wrapper script, see below. |

The binaries are installed into `/usr/lib/avicii-wlan/`, so they do not clash with Alpine's
`qrtr`, `tqftpserv` and `pd-mapper` packages.

## wlfwd: replacing cnss-daemon

On Android, the closed `cnss-daemon` talks to the WLAN firmware over QMI (service WLFW).
The kernel driver (`icnss`, version 1) does **not** do this part itself. Without it, the
firmware never reports `FW_READY` and the driver cannot start.

`wlfwd` is a small C program (about 700 lines, `wlfwd.c` in the aport) that does the same job:

1. Registers for indications, waits for `FW_MEM_READY`.
2. Sends `regdb.bin` (regulatory database).
3. Sends `bdwlan.bin` (board data file, with calibration for this phone).
4. Sends `CAL_REPORT` and the MAC address.
5. Stays running and saves calibration data the firmware sends back (to `/var/lib/wlfwd/`).

`wlfwd` also watches the WLFW service through the QRTR name service. If the WLAN domain
restarts (for example after a firmware crash), WLFW comes back on a new port, and `wlfwd`
runs the whole sequence again. Wi-Fi comes back in about 20–30 seconds, no reboot needed.

Usage: `wlfwd <firmware dir> <MAC> [state dir]`.

### Calibration must come from `persist`

This was the hardest part. The modem partition also contains `bdwlan.bin` and `regdb.bin`,
but **with those files the firmware crashes** right after `CAL_REPORT`. The correct files
are the OnePlus ones in the `persist` partition. Android's `cnss-daemon` also reads them from there.

On first start the wrapper mounts `persist` **read-only** and copies:

- `bdwlan.bin`, `regdb.bin` → `/var/lib/wlfwd/board/`
- `wlan/WCNSS_qcom_cfg.ini`, `wlan_mac.bin` → `/lib/firmware/postmarketos/wlan/qca_cld/`

The driver (`qcacld`) needs `WCNSS_qcom_cfg.ini` and `wlan_mac.bin` when it probes.
Without them it hangs waiting for firmware. The factory MAC address is read from `wlan_mac.bin`.

> **Back up `persist` before you start.** If it is lost, Wi-Fi calibration is lost too.

## What avicii-wlan does at start

The wrapper `/usr/lib/avicii-wlan/avicii-wlan` runs these steps:

1. Copies the board files from `persist` (only once).
2. `echo 1 > /dev/ipa` — starts IPA. **Only this way**: opening `/dev/subsys_ipa_fws`
   directly causes a kernel panic in TrustZone.
3. Boots ADSP and CDSP (`/sys/kernel/boot_adsp/boot`, `/sys/kernel/boot_cdsp/boot`).
4. Opens `/dev/subsys_modem` and keeps it open. Closing it shuts the modem down,
   so the script holds it for the whole service lifetime.
5. Starts `wlfwd`, then waits up to 40 s for `FW_READY`.
6. `echo ON > /dev/wlan` — loads the WLAN driver, `wlan0` appears.

The firmware files themselves (`wlanmdsp.mbn`, `*.jsn`, ...) are copied from the phone's
partitions on first boot by `avicii-firmware`, see [Firmware](firmware.md).

## Service order (OpenRC)

```
qrtr-ns → rmtfs-ro, tqftpserv-fw, pd-mapper → avicii-wlan → networkmanager
```

`avicii-wlan` has `need qrtr-ns rmtfs-ro tqftpserv-fw pd-mapper` and `before networkmanager`.

**Do not add `after dropbear` (or `after sshd`) to `avicii-wlan`.** We had it once, so that
SSH would start before Wi-Fi. But it is a dependency loop: `dropbear` needs `net`, `net` is
provided by `networkmanager`, and `networkmanager` starts after `avicii-wlan`. OpenRC breaks
such a loop silently. After more services were added, it broke the loop by **not starting
dropbear at all** — SSH was dead over both Wi-Fi and USB. `/var/log/rc.log` showed:

```
ERROR: cannot start dropbear as networkmanager would not start
```

**Kill switch:** to stop Wi-Fi bring-up at boot, create `/etc/avicii-wlan.disable`.

## Connecting

```sh
nmcli device wifi list
nmcli device wifi connect "<ssid>" password "<password>"
```

If the connection stops at `need authentication` (seen with a WPA2/WPA3 router),
disable PMF (Protected Management Frames) on the profile:

```sh
nmcli connection modify "<ssid>" wifi-sec.pmf disable
nmcli connection up "<ssid>"
```

> TODO: the root cause of the PMF problem is not known yet.

Check the result:

```sh
iw dev wlan0 link
ip route get 1.1.1.1
```

Then SSH works over Wi-Fi: `ssh user@<phone-ip>`.

## Debugging

| Where to look | What it tells you |
|---|---|
| `rc-status` | Are all five services `started`? |
| `/var/log/avicii-wlan.log` | `modem opened`, `FW_READY after Ns`, `wlan ON`, wlfwd output |
| `/var/log/tqftpserv-fw.log` | `[TQFTP] served ...` / `missing ...` — which files the modem asked for |
| `/var/log/pd-mapper.log` | pd-mapper errors |
| `/sys/kernel/debug/icnss/stats` | `FW_READY` counter (needs debugfs mounted) |
| `dmesg \| grep -i icnss` | Driver side of the handshake |

Tips:

- `echo ON > /dev/wlan` returns **EINVAL** if the driver did not get `FW_READY` in time.
  Look at the wlfwd output first.
- The driver loads **only once per boot**. After a failed attempt, reboot before you try again.
- If the modem crashes twice in a row, the phone may reboot into recovery.
  Choose "Reboot system now".
- To test recovery without a reboot, restart only the WLAN domain:
  ```sh
  sudo /usr/lib/avicii-wlan/pdr-restart 0 2 msm/modem/wlan_pd
  ```
  `wlfwd` should replay the sequence and `wlan0` should reconnect in about 30 s.
- If SSH is gone after a change to services, see [Known issues](known-issues.md) and use the
  initramfs debug shell over USB (`pmos.debug-shell` on the kernel command line).
