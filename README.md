# postmarketOS on OnePlus Nord (avicii)

A working [postmarketOS](https://postmarketos.org) port for the **OnePlus Nord (AC2003, codename `avicii`)**,
used as a small always-on Linux server: Docker, Wi-Fi, battery care and a web dashboard,
on a phone that was never supported by mainline Linux.

> **Status:** personal project, works on my device every day. Not part of official postmarketOS.
> Use at your own risk — read [Warnings](#warnings) before flashing anything.

## What works

| Feature | Status | Notes |
|---|---|---|
| Boot from internal storage | ✅ | Both A/B slots, see [A/B slots](docs/ab-slots.md) |
| Display (text console) | ✅ | Framebuffer console, large font. No graphical UI |
| USB networking + SSH | ✅ | `ssh user@172.16.42.1` |
| Wi-Fi 2.4 / 5 GHz | ✅ | Without Android's `cnss-daemon`, see [Wi-Fi](docs/wifi.md) |
| Docker | ✅ | With workarounds for the 4.19 kernel, see [Docker](docs/docker.md) |
| Battery charge limit (70–80 %) | ✅ | Runs from mains with the battery idle, see [Battery](docs/battery.md) |
| CPU frequency scaling | ✅ | `schedutil`, −7…10 % power at idle vs. `performance` |
| Real-time clock | ✅ | Via `swclock-offset` (the PMIC RTC is read-only) |
| Monitoring dashboard | ✅ | Web UI on port 8080, see [Dashboard](docs/dashboard.md) |
| zram swap | ✅ | `lz4` (the kernel has no `zstd`) |
| System log (`logbookd`) | ✅ | Patched: upstream crashes on long kernel messages, see [Known issues](docs/known-issues.md) |
| Modem / calls / SMS / mobile data | ❌ | Modem firmware runs (Wi-Fi lives inside it), telephony not set up |
| Audio | ❌ | ADSP is up, audio stack not configured |
| Camera, GPU, Bluetooth | ❌ / untested | |
| Firewall | ✅ | `iptables-legacy` (`nftables` does not work on 4.19): from the LAN only SSH and the dashboards, see [Known issues](docs/known-issues.md#firewall-nftables-does-not-work) |

## Hardware and software

- **SoC:** Qualcomm Snapdragon 765G (SM7250, platform `lito`), 8 GB RAM
- **Kernel:** downstream **4.19.300** from [LineageOS `android_kernel_oneplus_sm7250`](https://github.com/LineageOS/android_kernel_oneplus_sm7250) (`lineage-21`), built with clang
- **Userspace:** postmarketOS edge, console UI, OpenRC

**Why downstream?** Mainline Linux has no support for SM7250 at all (no `sm7250.dtsi`),
so the only way to boot is the vendor (Android) kernel. That kernel expects Android
userspace, and most of the work in this port is replacing what Android normally does.

## Where things are

| Repository | Contents |
|---|---|
| **this repo** | Documentation, deploy script, notes on every problem solved |
| [ScYurii/wlfwd](https://github.com/ScYurii/wlfwd) | The Wi-Fi firmware helper, packaged by `avicii-wlan` |
| [ScYurii/pmaports, branch `oneplus-avicii`](https://github.com/ScYurii/pmaports/tree/oneplus-avicii) | The actual packages (aports), rebased on upstream `main` |

More detail — branches, files on the PC and on the phone: [Where things are](docs/layout.md).

Packages in the `oneplus-avicii` branch (`device/downstream/`):

| Package | What it does |
|---|---|
| `linux-oneplus-avicii` | Kernel + patches (idle kthreads not counted in load average, OPLUS `midas` crash in PID namespaces, `waitid(P_PIDFD)` backport), `schedutil` by default |
| `device-oneplus-avicii` | Device config: boot image, cmdline, A/B slot marking, first-boot firmware copy, console font, zram |
| `avicii-wlan` | Wi-Fi bring-up: QRTR name service, `rmtfs`, `tqftpserv`, `pd-mapper` (patched), [`wlfwd`](https://github.com/ScYurii/wlfwd) |
| `avicii-server` | Charge limiter, `avicii-power`, monitoring dashboard, Docker fixes for 4.19, bash |
| `temp/logbookd` | Fork of Alpine's `logbookd` with a fix for a stack overflow on long kernel messages |

## Quick start

> TODO: verify end to end on a clean install.

**You need:** unlocked bootloader, a Linux PC. The steps, each with its own page:

1. **Back up EFS and `persist`** — the per-unit calibration has no other copy.
   → [Firmware](docs/firmware.md#back-up-efs-and-persist-first)
2. **Prepare the PC:** packages, pmbootstrap 3.x, the pmaports fork (branch `oneplus-avicii`),
   `pmbootstrap init` with `avicii-server` in `extra_packages`, `fastboot` without sudo.
   → [Preparing the PC](docs/setup-pc.md)
3. **Build and install:** `pmbootstrap install`, then `pmbootstrap export`.
   → [Building and flashing](docs/building.md)
4. **Disable verified boot** once (empty `vbmeta` in both slots), otherwise the bootloader
   silently refuses the image. → [Building](docs/building.md#4-disable-verified-boot-once)
5. **Flash** the rootfs to `userdata` and the boot image to **both** slots.
   → [Building](docs/building.md#5-flash)
6. **First boot:** `ssh user@172.16.42.1` over USB, let firmware be copied, reboot once,
   join Wi-Fi with `nmcli`. → [Reinstall from scratch](docs/reinstall.md#first-boot)

Where each file lives — on the PC, in pmaports and on the phone: [Where things are](docs/layout.md).

> **RU:** Порядок: бэкап EFS и persist → подготовка ноута → сборка → выключение verified boot
> (один раз) → прошивка rootfs и `boot.img` в оба слота → первая загрузка. Что где лежит —
> в [layout.md](docs/layout.md).

For day-to-day kernel/device updates there is [`tools/avicii-deploy`](tools/): build, check the image,
flash both slots and report the state after boot in one command.

## Interesting problems solved

Each one cost hours; the details are in `docs/`.

- **Wi-Fi without Android.** The Wi-Fi firmware runs inside the modem. Android's closed `cnss-daemon`
  is replaced by a small QMI client (`wlfwd`) that sends the board data files; the calibration must come
  from the `persist` partition, files from the modem partition crash the firmware. → [Wi-Fi](docs/wifi.md)
- **A/B reboot loop.** The bootloader counts down `slot-retry-count` on every boot and switches to the
  Android slot at zero. → [A/B slots](docs/ab-slots.md)
- **Random reboots every ~10 minutes.** Missing video firmware made a driver hang, and the SoC escalated
  it to a reboot. → [Firmware](docs/firmware.md)
- **Docker on a 4.19 kernel.** `runc` needs `openat2` (Linux 5.6), `nf_tables` too old, a vendor cgroup
  controller that allows ~5 groups, a vendor driver that panics on PID namespaces. → [Docker](docs/docker.md)
- **Load average 9 on an idle system.** Vendor kernel threads sleeping in `TASK_UNINTERRUPTIBLE`. → [Kernel](docs/kernel.md)
- **No system log since the second boot.** `logbookd` overflowed a 1024-byte buffer on the 1006-byte
  "Kernel command line" record and crashed on every boot. → [Known issues](docs/known-issues.md)
- **Every script "fails" with status `0xffffffff`.** The vendor kernel has `pidfd_open()` but not
  `waitid(P_PIDFD)`, so GLib loses exit codes. → [Kernel](docs/kernel.md)
- **Keeping the battery healthy 24/7.** The OPLUS charger driver re-enables input behind the
  `input_suspend` vote. → [Battery](docs/battery.md)

## Warnings

- **Back up EFS / `persist` first.** Modem calibration lives there; losing it can break Wi-Fi and the modem for good.
- **Flash the boot image to both slots.** Otherwise a slot switch boots an Android kernel that cannot read the
  postmarketOS `userdata` and loops.
- **`boot.img` and rootfs come in pairs.** Each `pmbootstrap install` creates new filesystem UUIDs that are
  baked into the kernel cmdline; mixing them gives "root partition not found".

## Acknowledgements

[postmarketOS](https://postmarketos.org) and pmbootstrap, [LineageOS](https://lineageos.org) for the kernel tree,
[linux-msm](https://github.com/linux-msm) for `qrtr`, `rmtfs`, `tqftpserv` and `pd-mapper`.

## License

Documentation: [CC BY-SA 4.0](LICENSE). Scripts in `tools/`: [MIT](tools/LICENSE). Packages: see each aport.
