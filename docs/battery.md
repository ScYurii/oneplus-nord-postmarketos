# Battery charge limiting

The phone runs 24/7 on a USB charger. A lithium battery that sits at 100 % all the
time ages fast. The `avicii-chglimit` service keeps the charge between **70 and 80 %**.
Most of the time the phone runs from mains and the battery is idle: it neither
charges nor discharges.

Both tools on this page come from the `avicii-server` package:

| Command / service | What it does |
|---|---|
| `avicii-chglimit` (OpenRC service) | Keeps the battery in the 70–80 % window |
| `avicii-power` | Shows the power draw live in the terminal |

## The charger knobs

The OnePlus (OPLUS) charger driver gives us two sysfs files. Together they cover
all three states we need:

| File | Value | Effect |
|---|---|---|
| `/sys/class/oplus_chg/battery/mmi_charging_enable` | `0` | Input stays on, the phone runs from mains, **the battery does not charge** |
| | `1` | Normal charging |
| `/sys/class/power_supply/battery/input_suspend` | `1` | USB input is off, **the phone runs from the battery** |
| | `0` | USB input is on |

To see what the battery is doing, read `battery/current_now`. On this driver the
value is in mA, and the sign is:

- `> 0` — the battery discharges,
- `0` — the phone runs from mains, the battery is idle,
- `< 0` — the battery charges.

Do not use `usb/input_current_now` for this check. It shows hundreds of mA even
when the phone runs from the battery.

## How avicii-chglimit works

The script checks the charge once per `INTERVAL` and picks one of three modes:

| Charge | Mode | Settings |
|---|---|---|
| above `CHG_MAX` | **drain** — run from the battery down to `CHG_MAX` | `mmi_charging_enable=0`, `input_suspend=1` |
| between the limits | **hold** — run from mains, battery idle | `input_suspend=0`, `mmi_charging_enable=0` |
| `CHG_MIN` or lower | **charge** — normal charging up to `CHG_MAX` | `input_suspend=0`, `mmi_charging_enable=1` |

Once charging starts at `CHG_MIN`, it continues until the charge reaches `CHG_MAX`.
Then the mode switches to hold.

The script writes the values again on every check, not only when the mode changes.
The reason: the driver resets `mmi_charging_enable` when the cable is plugged in again.

When the service stops (`INT` or `TERM`), the script restores normal charging:
`input_suspend=0`, `mmi_charging_enable=1`. If the script is killed hard
(`kill -9`), `supervise-daemon` starts it again after 10 seconds.

### The driver turns the input back on

This was the hardest part. After `mmi_charging_enable=1` is written (for example,
on the switch from charge to drain, or when the cable is plugged in again), the
OPLUS driver calls `charger_unsuspend()`. This turns the USB input back on
**directly**, behind the `input_suspend` vote. sysfs still shows
`input_suspend = 1`, but the phone runs from mains and even charges the battery
(in a test the battery current was −135 mA).

The script handles this in two ways:

1. In drain mode it writes `mmi_charging_enable=0` **first**, and only then
   `input_suspend=1`.
2. On every check in drain mode it reads `battery/current_now`. If the battery
   does not discharge (`current_now < 50`), it toggles the vote `0 → 1` again and
   writes this line to the log:
   ```
   drain: вход снова включён драйвером, отключаю повторно
   ```
   (Russian for "drain: input re-enabled by the driver, disabling again".)

Test: `mmi_charging_enable=1` was written by hand during drain. The battery charged
for about 50 seconds, then the service turned discharging back on at the next check
(~155 mA, stable).

### Why the charge drops after you plug in the cable

This is expected. If the battery is above `CHG_MAX` (for example at 100 % after Android
or after a normal charge), the service is in **drain** mode: it keeps the charger input
off, and the phone runs from the battery although the cable is connected. The charge
goes down at about 150–230 mA until it reaches `CHG_MAX`. Then the service switches to
**hold**: the phone runs from mains and the battery current is 0, so the charge stays
at 80 % for weeks. To skip the wait, you can unplug the cable: the effect is the same.

## Configuration

File `/etc/conf.d/avicii-chglimit`:

```sh
CHG_MAX=80     # above this: run from the battery
CHG_MIN=70     # at or below this: charge up to CHG_MAX
INTERVAL=60    # check period, seconds
```

`CHG_MIN` must be lower than `CHG_MAX`, otherwise the script exits with an error.
After a change, restart the service:

```sh
sudo rc-service avicii-chglimit restart
```

The log is `/var/log/avicii-chglimit.log` (readable by root only). It has a line
on start, on stop and on every mode change (with the charge and the battery
temperature). The messages are in Russian.

```sh
sudo tail /var/log/avicii-chglimit.log
```

To turn the limit off and go back to normal charging:

```sh
sudo rc-service avicii-chglimit stop
sudo rc-update del avicii-chglimit default
```

## avicii-power: live power readout

```sh
avicii-power        # one line per second, Ctrl+C to exit
avicii-power 5      # one line every 5 seconds
```

No `sudo` is needed. Each line shows the time, the mode, and three power values:

- **system** = USB input + battery,
- **USB input** = `usb/voltage_now × usb/input_current_now`,
- **battery** = `battery/voltage_now × current_now` (`+` discharge, `−` charge).

When the input is off (`usb/online = 0`), the USB sensor shows garbage, so the
script ignores it. The system current is shown at the battery voltage.

The column titles and mode names are in Russian: `батарея` (battery),
`сеть` (mains), `сеть+заряд` (mains + charging).

## Measurements

- From the battery: **0.7–1.0 W** (about 160–250 mA).
- From mains with normal charging: about 1.07 W at the input (232 mA × 4.59 V);
  about 0.5 W of that went into the battery.
- The charger is detected as `USB_DCP`, but the input limit is **250 mA**
  (`input_current_settled`). This is enough for the server, but charging an
  empty battery is slow.

## Side effects

- In hold mode the driver writes `oplus_chg_fast_switch_check] mmi_chg,return`
  to `dmesg` every 5 seconds. It is harmless.
- The [dashboard](dashboard.md) reads the current mode from sysfs and shows
  every mode change from the log as an event.

See also: [Dashboard](dashboard.md), [Known issues](known-issues.md).
