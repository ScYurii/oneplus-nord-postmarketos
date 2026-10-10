# Monitoring dashboard

`avicii-monitor` is a small web dashboard for the phone. It shows power, battery,
CPU, memory, temperature, Wi-Fi, services and Docker containers, with history for
one week. Open it in a browser on the same network:

```
http://<phone-ip>:8080/
```

It is part of the `avicii-server` package. There is no database and no framework:
a Python script collects the data, and a static HTML page draws the charts in the
browser. The page labels are in Russian.

## Two services

| Service | What it does | Runs as |
|---|---|---|
| `avicii-monitor` | Collector: `/usr/sbin/avicii-monitor`, Python 3, standard library only | root |
| `avicii-monitor-web` | Web server: `busybox-extras httpd` on port 8080 | `nobody:nogroup` |

The collector runs as root because it reads the Docker socket and the
[charge limiter](battery.md) log. The web server only serves files. Its root is
`/usr/share/avicii-monitor/www/`: `index.html` plus a symlink `data → /run/avicii-monitor`.

Both services are added to the `default` runlevel on install. Logs:
`/var/log/avicii-monitor.log`.

To use another port, set `PORT` in `/etc/conf.d/avicii-monitor-web` (the init
script reads `${PORT:-8080}`):

```sh
echo 'PORT=8081' | sudo tee /etc/conf.d/avicii-monitor-web
sudo rc-service avicii-monitor-web restart
```

## What is collected

Every **5 seconds**: power (USB input and battery), battery charge, current and
temperature, charge mode (drain / hold / charge), cable present, CPU load per
cluster and frequency, iowait, CPU temperature, load average, memory.

Every **30 seconds** (slower checks): Wi-Fi link (`iw`, `ip`), OpenRC service state
(from `/run/openrc/`), Docker containers (through `/var/run/docker.sock`), disk usage.

## Data tiers

To draw any time range fast, the collector keeps the data at three levels of detail:

| Tier | Step | Kept | Where |
|---|---|---|---|
| raw | 5 s | 24 hours | `/run/avicii-monitor/raw/<hour>.csv` (tmpfs, one file per hour) |
| minute | 1 min | 1 week | `/var/lib/avicii-monitor/minute.csv` → `/run/avicii-monitor/min/<day>.json` |
| history | 5 min | 1 week | `/var/lib/avicii-monitor/history.csv` → `/run/avicii-monitor/history.json` |

Notes:

- File names use `t // 3600` (hour) and `t // 86400` (day), so the page loads only
  the pieces it needs.
- Raw files are in tmpfs, so they are **lost on reboot**. Minute and 5-minute data
  are on flash and survive a reboot. Each row is about 100 bytes.
- The CSV files on flash are cut back to one week once a day.
- If the collector restarts, it reloads the last hour from the raw files.
- `now.json` holds the current state for the tiles.

## Events

Events are stored in `/var/lib/avicii-monitor/events.jsonl` (last 300) and shown
in the feed and as marks on the power chart:

- **boot** — system start (once per boot),
- **cable** — mains power lost / back (`usb/present`),
- **chglimit** — every new line in the [charge limiter](battery.md) log,
- **Wi-Fi** — connected / disconnected,
- **services** — state change of `avicii-wlan`, `avicii-chglimit`,
  `avicii-monitor-web`, `docker`, `containerd`, `dropbear`, `networkmanager`, `chronyd`,
- **containers** — state change of any Docker container,
- **battery temperature** — at 40 °C or more (cleared below 38 °C).

## The page

- **Tiles:** power and source, charge mode, charge with the 70–80 % window, CPU,
  memory, Wi-Fi.
- **Charts:** power, battery charge, CPU load, memory, temperature (battery and CPU),
  Wi-Fi signal. Also blocks for system info, services, Docker and events.
- **Presets:** 1 hour, 24 hours, 7 days. The browser remembers the last one.
- **Zoom:** drag with the mouse (or a finger, horizontally) over any chart. All
  charts redraw for that time range. The range is saved in the URL as `#t0-t1`
  (Unix seconds), so you can share a link to it.
- **Back:** the `← Назад` button or `Esc` returns to the previous view.
  **Reset:** the `Сбросить` button or a double click on a chart.
- While zoomed, the charts stop updating; the tiles keep updating.
- **Summary panel** for the visible range: energy (mWh), average / min / max power,
  CPU, memory, peak temperature, charge at start and end, time without mains, events,
  and which tiers were used.
- One shared cursor over all charts.

### Tier merging

The page picks the tier by the width of the range:

| Range | Tier |
|---|---|
| up to 2 hours | raw, 5 s |
| up to 36 hours | minute |
| longer | 5 minutes |

Where the detailed tier has no data (for example, raw data before the last reboot),
the page fills the edges from a coarser tier. The summary then says, for example,
"5 s, in places 5 min". Past chunks are cached in the browser.

### Smoothing

Raw 5-second power values jump between 0.6 and 1.0 W. Over one hour this looked like
a comb. Now the page splits the range into equal buckets of about 3 pixels. For each
bucket it draws the **average as a line** and the **min–max spread as a light band**.
For 1-minute and 5-minute points, the top of the band is the stored maximum. When you
zoom in far enough, each bucket has one point, and you see every sample.

## Cost on the phone

About **0.31–0.37 s of CPU per minute** (around 0.5 % of one core) and 16 MB of
memory. Most of it is system time spent reading the battery and USB sensors: the
vendor driver talks to the charger chip over I2C on every read.

## Security

> **There is no authentication.** Anyone on the network can open the page. It shows
> the Wi-Fi network name, the phone's IP address, service and container names.

The firewall does not work on this kernel (see [Known issues](known-issues.md)), so
port 8080 is open to the whole LAN. Use the dashboard on a trusted home network only.
Do not forward the port to the internet. For remote access, use an SSH tunnel:

```sh
ssh -L 8080:localhost:8080 user@<phone-ip>
# then open http://localhost:8080/
```

See also: [Battery](battery.md), [Docker](docker.md).
