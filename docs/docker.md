# Docker on the 4.19 kernel

Docker works on this phone: Docker 29, overlayfs storage, `crun` as the runtime,
cgroup v1. But the downstream 4.19 kernel is much older than what current Docker
expects. Out of the box, Docker fails in six different ways. This page lists each
problem, the reason, and the fix.

All fixes are in the **`avicii-server`** package. You do not need to do them by hand.

## Problems and fixes

| Problem | Fix |
|---|---|
| `runc` needs `openat2` (Linux 5.6) | `crun` as the default runtime |
| `nf_tables` in 4.19 is too old for iptables 1.8 | `iptables-legacy` through a PATH wrapper |
| Kernel modules do not load on demand | Load `xt_addrtype`, `br_netfilter`, `xt_CHECKSUM` at boot |
| cgroup v2 has no `devices` controller | cgroup v1 (`rc_cgroup_mode="legacy"`) |
| Vendor `schedtune` cgroup allows only ~5 groups | Unmount `/sys/fs/cgroup/schedtune` |
| Vendor OPLUS `midas` driver panics with PID namespaces | Kernel patch `0002` |

### 1. runc → crun

`runc` 1.5 uses libpathrs, which needs the `openat2` system call. It appeared in
Linux 5.6, so it does not exist on 4.19. Docker only shows:

```
OCI runtime create failed: runc did not terminate successfully: exit status 2
```

Docker does not keep the stderr of `runc`. Run by hand (`runc spec`, then
`runc run -b <bundle>`), it panics: `at least one candidate /proc/thread-self path should work`.

`crun` is written in C and does not need new system calls. The package installs
`/etc/docker/daemon.json`:

```json
{
  "default-runtime": "crun",
  "runtimes": { "crun": { "path": "/usr/bin/crun" } }
}
```

### 2. iptables-legacy

The `iptables` command in Alpine works through `nf_tables`. The `nf_tables` code in
4.19 is too old for iptables 1.8, so every call fails with
`Could not fetch rule set generation id: Invalid argument`, and Docker cannot create
its NAT chain. The classic `ip_tables` is built into the kernel and works.

Docker calls `iptables` by name from `PATH`. So the package creates a directory with
two symlinks:

```
/usr/lib/avicii-server/docker-bin/iptables  -> /usr/bin/iptables-legacy
/usr/lib/avicii-server/docker-bin/ip6tables -> /usr/bin/ip6tables-legacy
```

and puts this directory first in `PATH`, only for the Docker service (see
[What the package changes](#what-the-package-changes)). The rest of the system is
not affected.

### 3. Kernel modules

Module autoloading is broken on this kernel (`depmod` warns that it cannot open
`modules.builtin.modinfo`). Docker adds a rule with `-m addrtype --dst-type LOCAL`,
which needs `xt_addrtype`. The package loads the needed modules at boot from
`/etc/modules-load.d/avicii-docker.conf`:

```
xt_addrtype
br_netfilter
xt_CHECKSUM
```

### 4. cgroup v1

In cgroup v2 there is no `devices` controller; the runtime controls devices with eBPF
instead, and eBPF on 4.19 is not reliable. So the system uses cgroup v1. The package
sets this in `/etc/rc.conf`:

```sh
rc_cgroup_mode="legacy"
```

A reboot is needed after the first install.

### 5. schedtune

`schedtune` is a vendor (Android) cgroup controller. It supports only about 5 groups.
`crun` creates a group in every mounted controller and fails:

```
creating cgroup directory /sys/fs/cgroup/schedtune/docker/...: Out of memory
```

The fix is to unmount it just before Docker starts:
`umount /sys/fs/cgroup/schedtune`. Turning it off in the kernel config is not a good
idea: the scheduler code (`rt.c`, `fair.c` and a vendor file) calls it without
`#ifdef`, so patching the scheduler is riskier than one `umount`.

### 6. OPLUS midas kernel panic

The vendor `midas` driver collects CPU time per process in a timer interrupt. It looks
up the process with `find_task_by_vpid()`, which searches in the PID namespace of the
*current* task. Inside a container this returns `NULL`, and the next line reads
`task->comm` → kernel panic. Any root process that lives a bit longer in a container
crashed the whole phone.

The patch `0002-oplus-midas-pidns-null-deref.patch` in `linux-oneplus-avicii` takes
the name from `p->group_leader->comm` instead. See [Kernel](kernel.md).

## What the package changes

`/etc/conf.d/docker` and `/etc/rc.conf` belong to other packages. So the
`avicii-server` install script (also run on upgrade) adds its lines as a **marked
block**, and only once:

```sh
# avicii-server begin
export PATH="/usr/lib/avicii-server/docker-bin:$PATH"
umount /sys/fs/cgroup/schedtune 2>/dev/null || true
# avicii-server end
```

OpenRC reads this file right before it starts Docker. The `|| true` is needed:
without it, OpenRC reports an error loading `conf.d/docker` when `schedtune` is
already unmounted.

On removal, the package stops its services and deletes the marked block. It does not
change `rc_cgroup_mode` or the login shell back.

## Bash completion

The package also installs `bash`, `bash-completion` and `docker-bash-completion`,
and switches the default user to `bash` (see below). After a new login,
`docker <Tab>` completes commands, container and image names. The shell of `root`
is not changed.

## First setup

The package does this itself:

- adds `containerd` and `docker` to the `default` runlevel;
- adds the default user (UID 10000, normally `user`) to the `docker` group and
  changes its login shell from `ash` to `bash` (a shell you chose yourself is kept).

pmbootstrap creates the user only after the packages are installed. So on a fresh
install the user part is done by the one-shot boot service `avicii-user-setup`
on the first boot; it leaves the marker `/var/lib/avicii-server/user-setup-done`
and does nothing after that.

After the first install, reboot once: cgroup v1 (`rc_cgroup_mode="legacy"`) is
used only from the next boot.

Check with `rc-update show` that both services are in `default`. Then test:

```sh
docker info | grep -iE 'runtime|cgroup'
docker run --rm hello-world
```

## Containers after a reboot

By default a container has the restart policy `no`. After a reboot of the phone it
stays stopped, even when Docker starts. To bring a container back automatically:

```sh
docker update --restart unless-stopped <container>
```

For a new container, add `--restart unless-stopped` to `docker run`.

## Limits

- **No firewall.** `nftables` does not work on 4.19, so ports published by containers
  are open to the whole LAN. See [Known issues](known-issues.md).
- Images must be built for `arm64` (`linux/arm64/v8`).

See also: [Kernel](kernel.md), [Dashboard](dashboard.md) (shows container state and events).
