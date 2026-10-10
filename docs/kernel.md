# Kernel patches and config

The kernel package is `linux-oneplus-avicii`. It builds the vendor (Android) kernel,
because mainline Linux has no support for SM7250 (see the [README](../README.md)).

## Source

- Tree: [LineageOS `android_kernel_oneplus_sm7250`](https://github.com/LineageOS/android_kernel_oneplus_sm7250), branch `lineage-21`
- Pinned commit: `fb4b4374d3b9ad0f10ba38d159585129f092fb3d` (Linux **4.19.300**)
- Compiler: **clang** (`CC=clang`, `HOSTCC=clang`), with the GCC aarch64 cross toolchain for binutils
- Config: `config-oneplus-avicii.aarch64`, based on `arch/arm64/configs/vendor/lito-perf_defconfig`
- Device tree: base `vendor/20801/lito.dtb`. The device overlay is **not** installed:
  the bootloader takes it from the `dtbo` partition and applies it itself.

The commit is pinned, so a build is repeatable. Moving to a newer commit means
checking all patches again.

## Config notes

The vendor defconfig is made for Android. Main changes:

| Option | Why |
|---|---|
| `CPU_FREQ_DEFAULT_GOV_SCHEDUTIL=y` | Default governor was `performance`: CPUs ran at max clock even when idle. Measured on the phone: `schedutil` uses **7–10 % less power at idle** (672 vs 724 mW on average). `performance` is still built in, you can switch at runtime. |
| `FRAMEBUFFER_CONSOLE=y` | Without it `console=tty0` shows nothing on the screen. |
| `PID_NS`, `CGROUP_PIDS`, `CGROUP_DEVICE`, `CFS_BANDWIDTH`, `OVERLAY_FS`, `BRIDGE_NETFILTER`, ... | Needed by Docker, see [Docker](docker.md). |
| `UID_SYS_STATS` off, `OPLUS_FEATURE_FEEDBACK` off | Android telemetry, not needed. |

**cgroup v1.** The system runs cgroups in v1 ("legacy") mode: `rc_cgroup_mode="legacy"` in
`/etc/rc.conf`. In v2 there is no `devices` controller, and container runtimes must use eBPF
for it, which is fragile on 4.19. Details in [Docker](docker.md).

**Careful with vendor options.** Do not disable vendor options at random. Some vendor code calls
other vendor code without checking the config. For example `subsys-pil-tz.c` called the
feedback telemetry without `#ifdef`, and the build failed at link time (fixed in patch 0001).

**Module signing.** `MODULE_SIG_FORCE=y`, and a new signing key is made on **every build**.
So modules from an old build do not load in a new kernel (`Required key not available`).
Always install the kernel package on the phone together with the new `boot.img`.

## Patches

### 0001 — idle kernel threads not counted in load average

**Symptom:** load average about **9** on a fully idle system. CPU was free.

**Cause:** nine vendor kernel threads sleep forever in `TASK_UNINTERRUPTIBLE` (state `D` in `ps`).
Linux counts such threads in the load average.

**Fix:** let them sleep in an idle state instead (`TASK_IDLE`, state `I`), which is not counted:

- `wait_event` → `wait_event_idle` in `panic_flush.c`, `sde_hdcp_2x.c`, `dp_hdcp2p2.c`
- `TASK_UNINTERRUPTIBLE` → `TASK_IDLE` in `oplus_charger.c` (short-circuit check thread)
- `msleep` → `msleep_interruptible` in `uboot_log.c`, `oplus_usbtemp.c`, `kernel_fb.c`
  (the USB port temperature protection still works)
- `subsys-pil-tz.c`: call the feedback telemetry only if `CONFIG_OPLUS_FEATURE_FEEDBACK` is on
- Two noisy `pr_err` messages (every 5 s) lowered to info/debug level

Result: load average 0 when idle.

### 0002 — OPLUS `midas` NULL dereference in PID namespaces

**Symptom:** kernel panic when a container runs, for example a build inside Docker.

**Cause:** the vendor driver `drivers/soc/oplus/midas/v1/midas_ioctl.c` runs in the timer interrupt
and calls `find_task_by_vpid()` with a global PID. Inside a container (a different PID namespace)
it returns NULL, and the next line reads `task->comm` → panic. Any long-running root process
in a container could crash the phone.

**Fix:** use `p->group_leader->comm`. The thread group leader is already known, no lookup needed.

### 0003 — backport `waitid(P_PIDFD)`

**Symptom:** nm-dispatcher logs `exited with unknown status 0xffffffff` for every script,
even when the script returns 0. GLib prints `waitid(pid:..., pidfd=...) failed: Invalid argument`.

**Cause:** this kernel has `pidfd_open()` (an Android backport), but not `waitid(P_PIDFD)`
(upstream Linux 5.4, commit `3695eae5fee0`). GLib sees `pidfd_open` and then waits with
`waitid(P_PIDFD)`, which fails. So **every GLib program loses the exit status of its child
processes**. NetworkManager is just the most visible case.

**Fix:** minimal backport: `P_PIDFD` in `include/uapi/linux/wait.h` and a local
`pidfd_get_pid()` helper in `kernel/exit.c` (the tree has no `pidfd_pid()`).

Check on the phone:

```sh
python3 -c 'import os, sys
pid = os.fork()
if pid == 0: os._exit(7)
fd = os.pidfd_open(pid)
print(os.waitid(os.P_PIDFD, fd, os.WEXITED).si_status)'
```

It must print `7`.

## Kernel command line

The device package installs `kernel-cmdline.conf` as
`/usr/lib/kernel-cmdline.d/50-device-oneplus-avicii.conf`. Other packages add their own files
there. `generate-kernel-cmdline` joins them when the boot image is built. Two rules:

- A file in `/etc/kernel-cmdline.d/` with the **same name** as a file in
  `/usr/lib/kernel-cmdline.d/` **replaces** it.
- A line `-arg` **removes** the parameter `arg`, even if another file adds it.

We use the second rule for `-splash`. The `splash` token comes from `postmarketos-bootsplash`
(which `postmarketos-ui-console` needs, so it cannot be removed). With `splash`, the text
console stays empty.

Two other important lines: `video=Virtual-1:d video=DSI-1:1080x2400@60e`. The vendor display
driver has a virtual writeback output, and the framebuffer chose it instead of the real panel.

The command line is baked into `boot.img`. After a change, rebuild and flash the boot image.

## Changing the config and rebuilding

On the build PC:

```sh
pmbootstrap kconfig edit linux-oneplus-avicii    # menuconfig, saves back to the aport
pmbootstrap checksum linux-oneplus-avicii        # update sha512sums in APKBUILD
```

Then raise `pkgrel` in `APKBUILD` and build:

```sh
pmbootstrap build linux-oneplus-avicii --arch aarch64
```

To add a patch: put the `.patch` file into the aport, add it to `source=`, run
`pmbootstrap checksum`. `default_prepare` applies all patches in order.

Flash it with `tools/avicii-deploy kernel`: it builds, checks the image, flashes **both**
slots and installs the package on the phone. Manual steps are in [Building and flashing](building.md).
