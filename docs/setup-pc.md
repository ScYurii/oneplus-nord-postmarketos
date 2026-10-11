# Preparing the PC

This page shows how to prepare a Linux PC to build and flash this port, from an empty
machine to `pmbootstrap init`. After that, continue with
[Building and flashing](building.md). For what each directory is for, see
[Where things are](layout.md).

> **RU:** Подготовка ноутбука с нуля: пакеты, pmbootstrap, наша ветка pmaports,
> SSH-ключ, fastboot без sudo, интернет для телефона через USB. Дальше — сборка и
> прошивка в [building.md](building.md), а что где лежит — в [layout.md](layout.md).

> TODO: these steps are written from a working setup, not yet repeated on a clean machine.

## What the PC does

| Job | Tool | Needs `sudo` |
|---|---|---|
| Builds packages and the rootfs image in chroots | `pmbootstrap` | yes (mounts, loop devices) |
| Writes images to the phone partitions | `fastboot` | no, with udev rules |
| Logs in to the phone, copies packages | `ssh`, `pmbootstrap sideload` | no |
| Gives the phone Internet over USB (optional) | NAT with `iptables` | yes |

> **RU:** Ноут собирает пакеты и образ (pmbootstrap, нужен sudo), прошивает телефон
> (fastboot, без sudo при правильных udev-правилах), заходит на телефон по SSH и, по
> желанию, раздаёт ему интернет через USB.

Used for this port: Ubuntu 26.04, pmbootstrap 3.11.1 from git, Python 3.14,
`fastboot` 34.0.5. Any distribution with Python 3.10 or newer should work; the
package names below are for Ubuntu/Debian.

## Disk space

The pmbootstrap work directory is about **4–5 GB** after a few kernel builds:
the biggest parts are the compiler cache (~2 GB), the rootfs chroot (~1 GB) and
downloaded packages (~1 GB). A kernel build also needs a few GB of temporary space.
Keep **at least 15 GB free** in your home directory.

> **RU:** Рабочий каталог pmbootstrap после нескольких сборок ядра — 4–5 ГБ (кеш
> компилятора ~2 ГБ, chroot с rootfs ~1 ГБ, скачанные пакеты ~1 ГБ), плюс временное
> место под сборку ядра. Держите свободными хотя бы 15 ГБ.

## 1. Packages

```sh
# pmbootstrap needs git, python3 and openssl; fastboot and adb talk to the phone,
# android-udev-rules lets you use them without sudo
# pmbootstrap нужны git, python3 и openssl; fastboot и adb работают с телефоном,
# android-udev-rules даёт работать с ними без sudo
sudo apt install git python3 openssl fastboot adb android-udev-rules
```

The udev rules give access to users in the `plugdev` group. Check that you are in it:

```sh
# should print "plugdev" among the groups
# в списке групп должна быть plugdev
groups
# if not: add yourself, then log out and in again
# если нет — добавить себя и перелогиниться
sudo usermod -aG plugdev "$USER"
```

> **RU:** Ставим git, python3, openssl (нужны pmbootstrap), fastboot и adb, и
> udev-правила для Android-устройств. Правила дают доступ группе `plugdev` —
> проверьте, что вы в ней, иначе fastboot будет требовать sudo.

## 2. pmbootstrap

The port needs pmbootstrap **3.x**. Distribution packages are often older, so install it
from git and put a link to it in your `PATH`:

```sh
# clone pmbootstrap
# скачать pmbootstrap
mkdir -p ~/src
git clone https://gitlab.postmarketos.org/postmarketOS/pmbootstrap.git ~/src/pmbootstrap

# make "pmbootstrap" a command (~/.local/bin is in PATH on Ubuntu)
# сделать команду pmbootstrap (~/.local/bin в Ubuntu уже есть в PATH)
mkdir -p ~/.local/bin
ln -s ~/src/pmbootstrap/pmbootstrap.py ~/.local/bin/pmbootstrap

# check: should print 3.x
# проверка: должна быть версия 3.x
pmbootstrap --version
```

To update pmbootstrap later: `git -C ~/src/pmbootstrap pull`.

> **RU:** pmbootstrap нужен версии 3.x, в пакетах дистрибутивов он часто старее, поэтому
> ставим из git в `~/src/pmbootstrap` и делаем ссылку в `~/.local/bin`. Обновление —
> `git pull` в этом каталоге.

## 3. sudo

pmbootstrap calls `sudo` itself (for mounts and loop devices). It needs a **real terminal**
where `sudo` can ask for your password:

- run pmbootstrap and `avicii-deploy` in your own terminal, not from a script or tool
  without a TTY;
- a kernel build is long. If the `sudo` timeout runs out during the build, a later step can
  fail with a strange error (this happened once: `losetup` printed an error instead of JSON).
  Just run the command again.

> **RU:** pmbootstrap сам вызывает sudo и должен иметь возможность спросить пароль — запускайте
> его в обычном терминале. Если за время долгой сборки ядра sudo «забыл» пароль, следующий шаг
> может упасть со странной ошибкой — просто повторите команду.

## 4. SSH key

pmbootstrap copies your public SSH keys into the image, so you can log in to the phone
without a password. Create a key if you do not have one:

```sh
# skip if ~/.ssh/id_ed25519.pub already exists
# пропустить, если ~/.ssh/id_ed25519.pub уже есть
ssh-keygen -t ed25519
```

Keys are copied when `ssh_keys = True` is set in the pmbootstrap config. `pmbootstrap init`
asks about it; you can also set it later with `pmbootstrap config ssh_keys True`.

> **RU:** pmbootstrap кладёт ваши публичные SSH-ключи в образ — тогда вход на телефон без
> пароля. Если ключа нет, создайте его. Нужна настройка `ssh_keys = True` (её спрашивает
> `pmbootstrap init`).

## 5. pmaports with this port

The port is not in the official pmaports, so pmbootstrap must build from the fork.
pmbootstrap still needs the **official** pmaports as a git remote: it reads the list of
channels (`channels.cfg`) from its `main` branch and stops with an error if no remote
points to `gitlab.postmarketos.org`. So clone the official repository and add the fork
as a second remote, **before** the first `pmbootstrap init`:

```sh
# default pmbootstrap work directory
# рабочий каталог pmbootstrap по умолчанию
WORK=~/.local/var/pmbootstrap

# official pmaports, where pmbootstrap looks for it (remote "origin")
# официальные pmaports туда, где pmbootstrap их ищет (remote "origin")
mkdir -p "$WORK/cache_git"
git clone https://gitlab.postmarketos.org/postmarketOS/pmaports.git "$WORK/cache_git/pmaports"

# add the fork as remote "avicii" and switch to the port branch
# добавить форк как remote "avicii" и переключиться на ветку порта
cd "$WORK/cache_git/pmaports"
git remote add avicii https://github.com/ScYurii/pmaports.git
git fetch avicii
git checkout -b oneplus-avicii avicii/oneplus-avicii
```

If pmbootstrap was already initialised, pmaports is already cloned: run only the last three
commands in `$(pmbootstrap config aports)`.

Which branch is which: `oneplus-avicii` is the one to build from;
`avicii-upstream` is the cleaned-up copy prepared for upstream postmarketOS and has no
`avicii-server`. See [Where things are](layout.md#repositories).

> **RU:** Порта нет в официальных pmaports, поэтому собираем из форка. Но pmbootstrap всё
> равно нужен remote на официальный репозиторий: из его ветки `main` он читает список каналов
> и без него падает с ошибкой. Поэтому клонируем официальные pmaports, добавляем форк вторым
> remote и переключаемся на ветку `oneplus-avicii` — **до** первого `pmbootstrap init`.
> Если pmbootstrap уже настроен — только последние три команды. Собирать
> нужно из `oneplus-avicii`; `avicii-upstream` — версия для отправки в postmarketOS, без
> `avicii-server`.

## 6. pmbootstrap init

```sh
pmbootstrap init
```

Answers for this port:

| Question | Answer | Why |
|---|---|---|
| Work path | default (`~/.local/var/pmbootstrap`) | Must match the place of the clone in step 5 |
| Channel | `edge` | The `oneplus-avicii` branch is based on edge. Another channel makes pmbootstrap check out another branch |
| Vendor / device | `oneplus` / `avicii` | |
| User interface | `console` | The port has no graphics |
| Init system | `openrc` | All services of the port are OpenRC services |
| Extra packages | include `avicii-server` (see below) | Not pulled in by the device package |
| SSH keys | yes | Step 4 |

Then add the extra packages. **`avicii-server` must be in this list**, otherwise the
charge limit, the dashboard and the Docker settings for 4.19 are not installed:

```sh
# avicii-server is required; the rest is what this port uses daily
# avicii-server обязателен; остальное — то, чем порт пользуется каждый день
pmbootstrap config extra_packages avicii-server,docker,docker-cli-compose,docker-openrc,nano,htop,usbutils,evtest
```

Check the result:

```sh
# should show device = oneplus-avicii and the extra_packages list
# должно быть device = oneplus-avicii и список extra_packages
cat ~/.config/pmbootstrap_v3.cfg
```

> **RU:** В `pmbootstrap init` выбираем канал `edge` (иначе pmbootstrap переключит pmaports на
> другую ветку), устройство `oneplus`/`avicii`, интерфейс `console`, OpenRC, SSH-ключи — да.
> Потом обязательно добавляем `avicii-server` в `extra_packages`: device-пакет его не тянет,
> и без него не будет ограничения заряда, дашборда и настроек Docker.

## 7. fastboot without sudo

Put the phone in fastboot mode: power off, then hold **Power + Volume Up**.
Connect it with a USB cable and check:

```sh
# should print one device, without sudo
# должно показать одно устройство, без sudo
fastboot devices
```

If the list is empty, but `sudo fastboot devices` shows the phone, the udev rules or the
`plugdev` group from step 1 are not active yet: log out and in again, or reconnect the cable.

> **RU:** Режим fastboot: выключить телефон, зажать Power + громкость вверх. `fastboot devices`
> должен видеть телефон без sudo. Если видит только с sudo — не применились udev-правила или
> группа plugdev: перелогиниться или переподключить кабель.

## 8. avicii-deploy

`tools/avicii-deploy` does the normal update cycle (build, check, flash, check after boot)
in one command. Install it from this repository:

```sh
# clone this repository (documentation and tools)
# склонировать этот репозиторий (документация и скрипты)
git clone https://github.com/ScYurii/oneplus-nord-postmarketos.git ~/src/oneplus-nord-postmarketos

# install the script as a command
# поставить скрипт как команду
install -m755 ~/src/oneplus-nord-postmarketos/tools/avicii-deploy ~/.local/bin/avicii-deploy
```

It also needs `python3` (image checks) and `ping`, both are on Ubuntu by default.
Usage: [Building and flashing](building.md#updates-with-avicii-deploy).

> **RU:** `avicii-deploy` — сборка, проверка образа, прошивка обоих слотов и проверка после
> загрузки одной командой. Ставим его из этого репозитория в `~/.local/bin`.

## 9. Internet for the phone over USB (optional)

Over USB the phone has the address `172.16.42.1`, and the PC gets `172.16.42.2`.
Before Wi-Fi works (first boot) the phone has no Internet. To give it Internet through the
PC, enable NAT on the PC:

```sh
# the PC interface that has Internet (Wi-Fi or Ethernet), see "ip route | grep default"
# интерфейс ноута с интернетом (Wi-Fi или Ethernet), смотрите "ip route | grep default"
OUT=wlp0s20f3

# forward packets and hide the phone behind the PC address; lost after a PC reboot
# пересылать пакеты и прятать телефон за адресом ноута; сбрасывается после перезагрузки ноута
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -s 172.16.42.0/24 -o "$OUT" -j MASQUERADE
```

On the phone, a default route via the PC and a DNS server are needed. In this port this is
a manual file, not part of any package: `/etc/local.d/usbnet.start`
(run at boot by the OpenRC `local` service):

```sh
#!/bin/sh
# metric 1000: Wi-Fi (NM, metric 600) is the primary route, USB is the fallback
# metric 1000: основной маршрут — Wi-Fi (NM, metric 600), USB — запасной
ip route add default via 172.16.42.2 metric 1000 2>/dev/null
echo nameserver 1.1.1.1 > /etc/resolv.conf
```

```sh
# on the phone: make it executable and enable the local service
# на телефоне: сделать исполняемым и включить сервис local
sudo chmod +x /etc/local.d/usbnet.start
sudo rc-update add local default
```

> **RU:** По USB у телефона адрес 172.16.42.1, у ноута — 172.16.42.2. Пока нет Wi-Fi, интернет
> телефону можно раздать с ноута через NAT (правила сбрасываются после перезагрузки ноута).
> На телефоне нужен маршрут через ноут и DNS — это ручной файл `/etc/local.d/usbnet.start`,
> он не входит ни в один пакет.

## Next

1. **Back up EFS and `persist`** before anything else — see [Firmware](firmware.md#back-up-efs-and-persist-first).
2. Build and flash — [Building and flashing](building.md).
3. First boot — [Reinstall from scratch](reinstall.md#first-boot).

> **RU:** Дальше: сначала бэкап EFS и persist, потом сборка и прошивка, потом первая загрузка.
