# Reinstall from scratch

This is a checklist for a clean install on the phone. The goal: after one install and
a few manual steps, the phone works the same as before. Most of the setup is in packages.
Only a few things must be done by hand.

> TODO: this checklist has not yet been tested end to end on a clean install.

## Before you start

- **Back up EFS and `persist`** if you have not done it yet. See [Firmware](firmware.md#back-up-efs-and-persist-first).
- Prepare the PC as described in [Preparing the PC](setup-pc.md).
- Build and flash as described in [Building and flashing](building.md):
  rootfs to `userdata`, `boot.img` to **both** slots, `vbmeta` with verification disabled.
- **`boot.img` and rootfs are a pair.** The filesystem UUIDs are created by
  `pmbootstrap install` and written into the kernel cmdline. Do not flash a `boot.img`
  from an older install with a new rootfs (or the other way around) — the boot stops with
  "root partition not found".

## What is packaged

All of this comes with `device-oneplus-avicii` and its dependencies, except `avicii-server`:
it must be in `extra_packages` of the pmbootstrap config (see
[Preparing the PC](setup-pc.md#6-pmbootstrap-init)). You do not need to set the rest up by hand.

| Package | What it brings |
|---|---|
| `linux-oneplus-avicii` | Kernel 4.19.300 with patches, `schedutil` as default governor |
| `device-oneplus-avicii` | Boot image config and cmdline (no `splash`), `avicii-qbootctl` (marks the A/B slot as good), `avicii-firmware` (first-boot firmware copy), large console font, zram with `lz4` |
| `avicii-wlan` | Wi-Fi services: `qrtr-ns`, `rmtfs-ro`, `tqftpserv-fw`, `pd-mapper`, `avicii-wlan` (copies board files from `persist` on first start) |
| `avicii-server` | `avicii-chglimit`, `avicii-power`, dashboard (`avicii-monitor`, `avicii-monitor-web`), Docker settings for 4.19 (crun, legacy iptables, cgroup v1), bash |
| `logbookd` (`temp/` in the branch) | Fixed system log daemon (the stock one crashes on a long kernel cmdline) |

## What is manual

- **Wi-Fi password.** It is not stored in any package. You join the network with `nmcli`
  on the first boot (see below).
- **Docker containers and images** are not kept. Pull and start them again.
- **Dashboard history** (`/var/lib/avicii-monitor/`) is not kept. It starts empty.

## First boot

1. **Connect over USB.** The phone gives itself `172.16.42.1` on the USB network:

   ```sh
   ssh user@172.16.42.1
   ```

2. **Let the firmware be copied.** On the first boot `avicii-firmware` copies the vendor
   firmware from the phone partitions to `/lib/firmware/postmarketos`. Check its result:

   ```sh
   ls /lib/firmware/postmarketos/.avicii-firmware   # marker file: copy is done
   find /lib/firmware/postmarketos -type f | wc -l  # should be more than 100
   ```

   See [Firmware](firmware.md) for details.

3. **Reboot once.** Drivers that started before the copy did not get their firmware.
   They only get it after the next boot.

   ```sh
   sudo reboot
   ```

4. **Join Wi-Fi.** After the reboot, `avicii-wlan` should bring up `wlan0`
   (this takes about 20 seconds after boot):

   ```sh
   nmcli device wifi list
   sudo nmcli device wifi connect "<ssid>" password "<password>"
   ```

   If it stays in `need authentication` on a WPA2/WPA3 network, disable PMF in the profile:
   `sudo nmcli connection modify "<ssid>" wifi-sec.pmf disable`. More in [Wi-Fi](wifi.md).

5. **Check the services.**

   ```sh
   rc-status boot      # avicii-qbootctl, avicii-firmware, consolefont: started
   rc-status default   # qrtr-ns, rmtfs-ro, tqftpserv-fw, pd-mapper, avicii-wlan,
                       # avicii-chglimit, avicii-monitor, avicii-monitor-web: started
   sudo qbootctl       # current slot should be marked successful
   ip addr show wlan0
   cat /var/log/avicii-wlan.log
   ```

   Two services are expected to fail on this kernel and can be ignored:
   `systemd-sysusers` (needs Linux 5.8+) and `nftables` (`nf_tables` in 4.19 is too old).

6. **Connect over Wi-Fi** from now on: `ssh user@<phone-ip>`. The dashboard is at
   `http://<phone-ip>:8080/`.

## Things to check after a fresh install

The packages do almost everything. Check these points:

- **User setup.** On the first boot the `avicii-server` service `avicii-user-setup`
  switches `user` to bash and adds it to the `docker` group. Check:

  ```sh
  getent passwd user        # ends with /bin/bash
  id user                   # includes docker
  rc-update show | grep -E 'docker|containerd'
  ```

  See [Docker](docker.md).
- **Firewall.** `avicii-firewall` (from `avicii-server`) leaves only SSH and the dashboards
  open to the home network; USB stays fully open. Check: `sudo iptables-legacy -S AVICII-IN`.
  If your home network is not `192.168.1.0/24`, set `LAN` in `/etc/conf.d/avicii-firewall`.

