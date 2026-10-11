# Known issues

Most problems here come from one fact: the kernel is **4.19** (from 2018), but the
userspace is current postmarketOS edge. New programs sometimes need kernel features
that 4.19 does not have.

Each issue has a **symptom**, a **cause** and a **status**:
open, fixed, or accepted (we know about it and keep it).

## Open or accepted

### Firewall: nftables does not work

- **Symptom:** the `nftables` service fails at boot. `nft` prints
  `netlink: Error: cache initialization failed: Invalid argument`.
  No rules are loaded, so **all ports (SSH, dashboard 8080, ...) are open on the LAN**.
- **Cause:** `nf_tables` in 4.19 is too old for the current `nft` userspace.
- **Status:** worked around with the old `iptables-legacy` (Docker uses it too, see [Docker](docker.md)).
  The `avicii-firewall` service from `avicii-server` applies the rules at boot:

  | From | Allowed |
  |---|---|
  | USB (`usb0`) | everything — the way back in if a rule goes wrong |
  | LAN (`192.168.1.0/24`) | TCP 22 (SSH), 8080 (power dashboard), 8081 (portfolio dashboard), ICMP |
  | IPv6 | the same ports, only from link-local addresses (no global IPv6 on Wi-Fi) |
  | anything else | dropped |

  Ports published by Docker never reach the `INPUT` chain: they are DNAT-ed and go through
  `FORWARD`. Their limit lives in the `DOCKER-USER` chain (new connections from Wi-Fi only from
  the LAN). Rules are added in one go: if the kernel rejects one, the script stops before the
  final `DROP`. Settings: `/etc/conf.d/avicii-firewall`. To switch off:
  `sudo rc-service avicii-firewall stop`.

### systemd-sysusers fails at boot

- **Symptom:** `systemd-sysusers` (and `systemd-tmpfiles`) print
  `Failed to chase and open directory ... No error information`. `systemd-sysusers` stays `stopped`.
- **Cause:** systemd 261 needs `STATX_MNT_ID` from `statx()`, which exists only since Linux 5.8.
  It will never work on 4.19.
- **Status:** accepted, harmless. All users and groups already exist (apk hooks create them),
  and all `tmpfiles.d` rules are already applied by the image and udev (except cleaning
  `/var/tmp` of files older than 30 days). The tmpfiles services ignore the error and show `started`.
  Removing sysusers from the runlevel does not help: `postmarketos-base-openrc` adds it back on every upgrade.

### Wi-Fi needs PMF disabled

- **Symptom:** NetworkManager stops at `need authentication` on a WPA2/WPA3 router.
- **Cause:** unknown. TODO: find out.
- **Status:** workaround: `nmcli connection modify "<ssid>" wifi-sec.pmf disable`. See [Wi-Fi](wifi.md).

### Kernel module autoloading is broken

- **Symptom:** some modules are not loaded when needed. `depmod` prints
  `WARNING: could not open modules.builtin.modinfo`.
- **Cause:** TODO: not investigated (probably the vendor build does not create `modules.builtin.modinfo`).
- **Status:** workaround. Modules that Docker needs (`xt_addrtype`, `br_netfilter`, `xt_CHECKSUM`)
  are listed in `/etc/modules-load.d/`. See [Docker](docker.md).

### The Wi-Fi driver loads once per boot

- **Symptom:** after a failed Wi-Fi start, `echo ON > /dev/wlan` does not work again.
- **Cause:** the vendor driver sets `hdd_loaded` once.
- **Status:** by design of the vendor driver. Reboot to try again. Firmware crashes (not driver
  failures) recover without a reboot, see [Wi-Fi](wifi.md).

### Slow charging from an empty battery

- **Symptom:** the charger input is limited to about 250 mA (`input_current_settled`),
  even on a wall charger detected as `USB_DCP`.
- **Cause:** TODO: not investigated.
- **Status:** open. It is enough to run the phone as a server, but charging an empty battery is slow.
  See [Battery](battery.md).

### Kernel log spam while the battery is on hold

- **Symptom:** `oplus_chg_fast_switch_check] mmi_chg,return` in `dmesg` every 5 s.
- **Cause:** the OPLUS charger driver logs this while charging is disabled by the charge limiter.
- **Status:** accepted, harmless.

### OS name shown as "Nura"

- **Symptom:** the console says `Welcome to Nura!`; `apk` redirects to `mirror.nura.eco`.
- **Cause:** upstream postmarketOS renamed the system to Nura in September 2026
  (`os-release`, `issue`, `motd`).
- **Status:** not a bug. Nothing to do.

### Not supported

- **Audio:** ADSP runs (Wi-Fi needs it), but the audio stack is not configured.
- **Telephony:** calls, SMS, mobile data are not set up. The modem itself runs, because
  Wi-Fi lives inside it. `rmtfs` keeps modem writes in RAM only, EFS is never written.
- **Camera, GPU, Bluetooth:** not working or not tested.
- **Graphical UI:** only the text console.

## Fixed

| Issue | Symptom | Cause | Fix |
|---|---|---|---|
| zram swap | `zramctl: failed to set algorithm: Invalid argument`, swap = 0 | Default algorithm is `zstd`; the kernel has only `lzo` and `lz4` | `deviceinfo_zram_swap_algo="lz4"` in the device package |
| logbookd crash | logbookd `crashed` at every boot, no boots in the log | Stack overflow: the 1006-byte "Kernel command line" record from `/dev/kmsg` is copied into a 1024-byte stack buffer with a ~20-byte header. The bug is also in upstream 0.2.6 | Fork `temp/logbookd` (0.2.4-r5) with patch 0003: the message is cut only for logread clients, the database keeps it whole |
| GLib child status | nm-dispatcher: `exited with unknown status 0xffffffff` | No `waitid(P_PIDFD)` in 4.19 | Kernel patch 0003, see [Kernel](kernel.md) |
| Load average 9 | High load on an idle system | Vendor threads in `TASK_UNINTERRUPTIBLE` | Kernel patch 0001 |
| Docker panic | Kernel panic with containers | OPLUS `midas` NULL dereference in PID namespaces | Kernel patch 0002 |
| No SSH after an update | `connection refused` over Wi-Fi and USB | OpenRC dependency loop: `avicii-wlan` was `after dropbear` | Removed, see [Wi-Fi](wifi.md#service-order-openrc) |
| Clock in 1970 | Wrong time after boot until NTP | The PMIC RTC is read-only | `swclock-offset` |
| Random reboots | Reboot every ~10 minutes | Missing video firmware | Copy firmware on first boot, see [Firmware](firmware.md) |
| A/B reboot loop | Phone boots Android slot and loops | `slot-retry-count` runs out | `qbootctl -m` at every boot, see [A/B slots](ab-slots.md) |

The logbookd fork should be dropped when the fix is in upstream and in Alpine.

## Not tested yet

- Quick start from the [README](../README.md) on a clean install, end to end.
- The dashboard event "mains power lost" (needs the cable pulled out).
- The charge limiter branch "start charging at 70 %" in real use.
