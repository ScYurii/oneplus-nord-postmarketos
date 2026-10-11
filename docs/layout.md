# Where things are

This page lists where the parts of the port live: the repositories, the files on the PC,
the packages in pmaports, and the partitions and files on the phone. Each table says
what a path is for. Paths on the PC are the defaults from [Preparing the PC](setup-pc.md).

> **RU:** Карта порта: какие репозитории, что лежит на ноуте, из чего состоят пакеты в
> pmaports, какие разделы и файлы на телефоне. В каждой таблице — путь и зачем он нужен.

## Repositories

| Repository | Branch | What is in it |
|---|---|---|
| [ScYurii/oneplus-nord-postmarketos](https://github.com/ScYurii/oneplus-nord-postmarketos) (this one) | `main` | Documentation, `tools/avicii-deploy` |
| [ScYurii/pmaports](https://github.com/ScYurii/pmaports) | `oneplus-avicii` | **Build from this branch.** The port packages on top of upstream pmaports, plus `avicii-server` and a commit that keeps `pkgrel` above the builds already on the phone |
| | `avicii-upstream` | The same port, cleaned up for a merge request to postmarketOS: one commit per package, `pkgrel=0`, no `avicii-server`. Passes the pmaports CI checks |
| [ScYurii/wlfwd](https://github.com/ScYurii/wlfwd) | `main`, tags `v*` | The Wi-Fi firmware helper (replaces Android's `cnss-daemon`). `avicii-wlan` downloads a tagged release |
| [LineageOS/android_kernel_oneplus_sm7250](https://github.com/LineageOS/android_kernel_oneplus_sm7250) | `lineage-21` | Kernel source. `linux-oneplus-avicii` downloads a fixed commit and applies its patches |
| [postmarketOS/pmaports](https://gitlab.postmarketos.org/postmarketOS/pmaports) | `main` | Upstream. Both branches of the fork are rebased on it from time to time |

> **RU:** Документация и `avicii-deploy` — в этом репозитории. Пакеты — в форке pmaports:
> собирать из ветки `oneplus-avicii`; `avicii-upstream` — та же работа, причёсанная для
> merge request в postmarketOS (без `avicii-server`). `wlfwd` — отдельный репозиторий,
> `avicii-wlan` скачивает его релиз по тегу. Исходники ядра — LineageOS, фиксированный коммит.

## On the PC

| Path | What it is | Notes |
|---|---|---|
| `~/src/pmbootstrap/` | pmbootstrap from git | Update with `git pull` |
| `~/.local/bin/pmbootstrap` | Link to `pmbootstrap.py` | Makes the `pmbootstrap` command |
| `~/.local/bin/avicii-deploy` | Copy of `tools/avicii-deploy` | Re-install after you change the script in the repo |
| `~/.config/pmbootstrap_v3.cfg` | pmbootstrap settings | Device, `extra_packages` (must include `avicii-server`), `ssh_keys` |
| `~/.ssh/id_ed25519.pub` | Your SSH key | Copied into the image by `pmbootstrap install` |
| `~/.local/var/pmbootstrap/` | pmbootstrap **work directory** | Everything below. `pmbootstrap config work` prints it |
| `/tmp/postmarketOS-export/` | Result of `pmbootstrap export` | **Links**, not copies: `boot.img` (from the rootfs chroot) and the rootfs image `oneplus-avicii.img` (from `chroot_native`, missing when that chroot was deleted). Gone after a PC reboot — run `export` again |

> **RU:** pmbootstrap и его настройки, `avicii-deploy`, SSH-ключ, рабочий каталог pmbootstrap и
> `/tmp/postmarketOS-export` — сюда `pmbootstrap export` кладёт **ссылки** на `boot.img` и образ
> rootfs (образа нет, если `chroot_native` удалён; после перезагрузки ноута каталог пропадает —
> снова `export`).

### Inside the work directory

| Path in `~/.local/var/pmbootstrap/` | What it is | Safe to delete? |
|---|---|---|
| `cache_git/pmaports/` | **pmaports checkout**, with the `oneplus-avicii` branch. The packages are built from here | No — this is the source |
| `packages/edge/aarch64/` | **Built packages** (`*.apk`) and their index | Yes, they are rebuilt. `avicii-deploy` and `sideload` take packages from here |
| `config_abuild/` | **Key that signs your packages** | **No.** The phone trusts this key; a new key means `apk` on the phone refuses your packages |
| `chroot_native/` | Alpine for the PC architecture: cross compilers, tools, and the rootfs image after `pmbootstrap install` (`home/pmos/rootfs/oneplus-avicii.img`) | Yes, recreated when needed. pmbootstrap itself deletes it before builds; then the rootfs image is gone too, flash it before the next build |
| `chroot_buildroot_aarch64/` | Build environment for aarch64 packages | Yes, deleted after each build |
| `chroot_rootfs_oneplus-avicii/` | **The phone system** before it becomes an image. `pmbootstrap install` makes the image from it, `export` takes the kernel from it | Yes, but then `pmbootstrap install` again (new UUIDs, see [Building](building.md#3-install-and-export)) |
| `cache_apk_aarch64/`, `cache_apk_x86_64/` | Downloaded Alpine/postmarketOS packages | Yes, downloaded again |
| `cache_ccache_*` | Compiler cache, makes kernel rebuilds fast | Yes, the next kernel build is slower |
| `cache_distfiles/` | Downloaded sources (kernel tarball, `wlfwd`, …) | Yes, downloaded again |
| `log.txt` | Log of every pmbootstrap command | Yes. Times have no date, so runs from different days mix up |

> **RU:** Главное в рабочем каталоге: `cache_git/pmaports` — исходники пакетов (наша ветка),
> `packages/edge/aarch64` — собранные пакеты, `config_abuild` — ключ подписи пакетов (**не
> удалять**: телефон доверяет только ему), `chroot_rootfs_oneplus-avicii` — система телефона до
> превращения в образ. Остальное — кеши и chroot-ы, их pmbootstrap пересоздаёт сам.
> В `log.txt` время без даты — прогоны разных дней перемешиваются.

## Packages (pmaports, `device/downstream/`)

| Package | Files in the aport | What they do |
|---|---|---|
| `linux-oneplus-avicii` | `APKBUILD` | Kernel commit and build options (clang) |
| | `config-oneplus-avicii.aarch64` | Kernel configuration |
| | `0001-…`, `0002-…`, `0003-…patch` | Kernel fixes, see [Kernel](kernel.md) |
| `device-oneplus-avicii` | `deviceinfo` | Boot image format, flash settings, zram algorithm |
| | `kernel-cmdline.conf` | Kernel command line (console on screen, no `splash`, Android options the vendor kernel needs) |
| | `modules-initfs` | Kernel modules in the initramfs |
| | `avicii-qbootctl.initd` | Marks the current A/B slot as successful, see [A/B slots](ab-slots.md) |
| | `avicii-firmware.initd`, `avicii-firmware.sh` | First-boot firmware copy, see [Firmware](firmware.md) |
| | `*.post-install`, `*.post-upgrade` | Enable the services above, large console font |
| `avicii-wlan` | `APKBUILD` | Builds `qrtr-ns`, `tqftpserv`, `pd-mapper` (linux-msm) and `wlfwd` |
| | `*-fw-dir.patch`, `*-jsn-dir.patch` | Firmware paths for this device |
| | `*.initd`, `avicii-wlan.sh` | The Wi-Fi services and the bring-up script, see [Wi-Fi](wifi.md) |
| `avicii-server` | `avicii-chglimit.*` | Battery charge limit, see [Battery](battery.md) |
| | `avicii-power.sh` | Live power draw in the terminal |
| | `avicii-monitor*` | Dashboard on port 8080, see [Dashboard](dashboard.md) |
| | `avicii-docker-*` | Docker settings for the 4.19 kernel, see [Docker](docker.md) |
| | `avicii-user-setup.*` | First boot: bash and the `docker` group for the user |
| `temp/logbookd` | `APKBUILD`, patches | Alpine's `logbookd` with a fix for long kernel messages, see [Known issues](known-issues.md) |

How they depend on each other: `device-oneplus-avicii` pulls in the kernel and `avicii-wlan`.
`avicii-server` is **not** a dependency — it comes from `extra_packages` in the pmbootstrap config.

> **RU:** Ядро — конфиг и три патча; device — параметры загрузки, cmdline, сервисы слота A/B и
> копирования прошивок; `avicii-wlan` — всё для Wi-Fi; `avicii-server` — заряд, дашборд, Docker;
> `temp/logbookd` — исправленный системный лог. device тянет ядро и `avicii-wlan`, а
> `avicii-server` ставится только через `extra_packages`.

## On the phone: partitions

Only the partitions that matter for this port:

| Partition | What is in it | Written by |
|---|---|---|
| `boot_a`, `boot_b` | `boot.img`: kernel, initramfs, device tree, command line | `fastboot flash`, always **both** slots |
| `vbmeta_a`, `vbmeta_b` | Verified boot data. Here: an empty image with verification off | Once, see [Building](building.md#4-disable-verified-boot-once) |
| `userdata` | **The whole postmarketOS system**: one image with two partitions inside, `pmOS_boot` and `pmOS_root` | `pmbootstrap flasher flash_rootfs` |
| `persist` | Per-unit calibration: Wi-Fi board files, MAC address | Never. Read-only for the port. **Back it up** |
| `modemst1`, `modemst2`, `fsg`, `fsc` | Modem EFS | Never (`rmtfs` runs read-only). **Back it up** |
| `modem_a`, `dsp_a`, `bluetooth_a`, `vendor_a` (inside `super`) | Vendor firmware | Never. Copied to the rootfs on the first boot |

> **RU:** На телефоне важны: `boot_a/boot_b` (ядро, всегда шьём оба), `vbmeta_a/b` (проверка
> загрузки, выключена один раз), `userdata` — вся система postmarketOS. `persist` и EFS
> (`modemst1/2`, `fsg`, `fsc`) порт не пишет никогда — но их обязательно нужно забэкапить.
> Разделы с прошивками вендора только читаются на первой загрузке.

## On the phone: files

| Path | What it is | From |
|---|---|---|
| `/boot/boot.img` | Boot image rebuilt **on the phone** after a kernel or initramfs update | `mkinitfs`. **Not** written to `boot_a/b` — flashing is done from the PC |
| `/lib/firmware/postmarketos/` | Vendor firmware copied from the phone partitions | `avicii-firmware`, first boot |
| `/lib/firmware/postmarketos/.avicii-firmware` | Marker: the copy is done, do not copy again | `avicii-firmware` |
| `/usr/lib/avicii-wlan/` | `wlfwd`, `qrtr-ns`, `tqftpserv`, `pd-mapper`, `pdr-restart`, the `avicii-wlan` script | `avicii-wlan` |
| `/var/lib/wlfwd/board/` | Wi-Fi board files (`bdwlan.bin`, `regdb.bin`) copied from `persist` | `avicii-wlan`, first start |
| `/var/lib/wlfwd/` | Calibration files saved by `wlfwd` (`wlfw_cal_*.bin`) | `wlfwd` |
| `/lib/firmware/postmarketos/wlan/qca_cld/` | Wi-Fi driver settings (`WCNSS_qcom_cfg.ini`) and MAC (`wlan_mac.bin`) from `persist` | `avicii-wlan`, first start |
| `/etc/avicii-wlan.disable` | If this file exists, `avicii-wlan` does not start Wi-Fi | Create by hand when needed |
| `/etc/conf.d/avicii-chglimit` | Charge limits (`CHG_MIN=70`, `CHG_MAX=80`) | `avicii-server`, edit if you like |
| `/etc/docker/daemon.json` | Docker settings for 4.19 (crun, legacy iptables, cgroup v1) | `avicii-server` |
| `/usr/lib/avicii-server/docker-bin/` | `iptables` → `iptables-legacy` links for Docker | `avicii-server` |
| `/var/lib/avicii-monitor/` | Dashboard history | `avicii-monitor` |
| `/var/lib/avicii-server/user-setup-done` | Marker: user already switched to bash and added to `docker` | `avicii-user-setup` |
| `/etc/NetworkManager/system-connections/` | Saved Wi-Fi networks with passwords | `nmcli`, by hand |
| `/etc/local.d/usbnet.start` | Default route and DNS over USB | **By hand**, see [Preparing the PC](setup-pc.md#9-internet-for-the-phone-over-usb-optional) |

> **RU:** Файлы на телефоне: прошивки в `/lib/firmware/postmarketos` (с маркером, что копия
> сделана), всё для Wi-Fi в `/usr/lib/avicii-wlan` и `/var/lib/wlfwd` (файлы платы из persist и
> калибровка), настройки заряда и Docker в `/etc`, история дашборда в `/var/lib/avicii-monitor`.
> Руками делаются только сети Wi-Fi (`nmcli`) и `usbnet.start`. `/boot/boot.img` на телефоне
> пересобирается, но в раздел сам не попадает — прошивка всегда с ноута.

### Logs

| Path | What |
|---|---|
| `/var/log/avicii-wlan.log` | Wi-Fi bring-up: `wlfwd` steps, `FW_READY`, `wlan ON` |
| `/var/log/pd-mapper.log`, `/var/log/tqftpserv-fw.log` | Modem helpers for Wi-Fi |
| `/var/log/avicii-chglimit.log` | Charge limiter decisions |
| `/var/log/avicii-monitor.log` | Dashboard collector |
| `/var/log/docker.log`, `/var/log/containerd.log` | Docker |
| `/var/log/logbookd.db` | System log (SQLite), read with `logread` |
| `/var/log/rc.log` | OpenRC boot log, only with `rc_logger="YES"` in `/etc/rc.conf` |

> **RU:** Логи: Wi-Fi — `avicii-wlan.log`, заряд — `avicii-chglimit.log`, дашборд —
> `avicii-monitor.log`, Docker — `docker.log`/`containerd.log`, общий системный лог —
> `logbookd.db` (читать `logread`), лог загрузки OpenRC — `rc.log` (если включён `rc_logger`).
