# gpu-ntfy-monitor

Push GPU and CPU temperatures of your GPU servers to your phone with [ntfy](https://ntfy.sh).

One small bash script, run from cron on any always-on Linux machine. Every minute it queries your servers over ssh (in parallel, nothing is installed on them), keeps 7 days of samples, and sends the **5-minute average**:

- a periodic report (every hour by default), and/or
- an immediate alert when a server runs hot or cannot be reached.

```
2026-10-21, 13:00:01
✅ gpu-server-1: 61°C | 42,41,40,39°C | 85% | 319W
🔥 gpu-server-2: 58°C | 77,78,76,79°C | 97% | 410W
⚠️ gpu-server-3: FAILED
```

Per server: `<icon> <name>: <cpu temp> | <temp of each GPU> | <mean GPU utilization> | <mean GPU power>`.
✅ normal, 🔥 all GPUs at or above the threshold, ⚠️ the server could not be queried. Alerts have a higher ntfy priority than reports.

## Set it up with Claude Code

Open [Claude Code](https://claude.com/claude-code) **on the machine that will run the monitor**, in a clone of this repo:

```
git clone https://github.com/jiho264/gpu-ntfy-monitor
cd gpu-ntfy-monitor
claude
```

Say "set this up". `CLAUDE.md` tells Claude to check the requirements, pick your servers from `~/.ssh/config`, ask you the open questions (thresholds, report schedule, alert mode, ntfy server), generate a private topic, install the script, add the cron entry, and send a test notification. Nothing user-specific is committed to the repo.

## Requirements

- **Monitor machine** (always on, Linux): bash 4+, GNU date, `flock`, `curl`, `ssh`, `awk`, cron.
- **GPU servers**: reachable from the monitor machine by **key-based ssh** through a host alias in `~/.ssh/config` (see `ssh-config.example`), with `nvidia-smi` and bash. `lm-sensors` is optional (CPU temperature).
- **Phone**: the ntfy app, subscribed to your topic.

## Manual setup

```bash
mkdir -p ~/.local/bin ~/.config/gpu-ntfy-monitor
cp bin/gpu-ntfy-monitor ~/.local/bin/ && chmod +x ~/.local/bin/gpu-ntfy-monitor
cp config.example ~/.config/gpu-ntfy-monitor/config && chmod 600 ~/.config/gpu-ntfy-monitor/config
$EDITOR ~/.config/gpu-ntfy-monitor/config      # hosts, thresholds, a random topic
gpu-ntfy-monitor --dry-run                     # print the message, send nothing
gpu-ntfy-monitor --test                        # send it now
(crontab -l 2>/dev/null; echo '* * * * * $HOME/.local/bin/gpu-ntfy-monitor') | crontab -
```

Subscribe to your topic in the ntfy app (server `ntfy.sh` unless you self-host).

## Settings

All settings live in `~/.config/gpu-ntfy-monitor/config` (see `config.example`; override the path with `GPU_NTFY_CONFIG`).

| Setting | Default | Meaning |
|---|---|---|
| `NTFY_SERVER` | `https://ntfy.sh` | ntfy server |
| `TOPIC` | required | topic to publish to; treat it like a password |
| `HOSTS` | required | ssh host aliases to watch |
| `LIMIT` | none | per-host threshold in °C; hot = all GPUs at or above it |
| `CPU_SENSOR` | none | per-host `chip:label` from `sensors`; omit to skip the CPU |
| `REPORT_EVERY_MINUTES` | `60` | periodic report interval, aligned to midnight; `0` = off |
| `ALERTS` | `on` | push immediately when a host is hot or unreachable |
| `PRIO_NORMAL` / `PRIO_ALERT` | `3` / `4` | ntfy priorities |
| `LOG_DAYS` | `7` | days of samples kept |
| `WINDOW_SECONDS` | `295` | averaging window |

## How it works

- Samples go to `~/.local/state/gpu-ntfy-monitor/samples.csv`, one row per host per minute: `time,epoch,host,cpu,util,power,gpu temps...` (`NA` = query failed, `-` = no CPU sensor).
- GPU count is detected with `nvidia-smi`, so servers can have different numbers of GPUs.
- `flock` stops overlapping runs; every ssh and the push have timeouts.
- Utilization and power are the mean over the GPUs of a server.

## Limitations

- While a server stays hot or unreachable and `ALERTS=on`, you get a push every minute. Use `ALERTS=off` for reports only.
- The topic is the only secret. Anyone who knows it can read your messages; use a self-hosted ntfy with access control if that matters. Authenticated ntfy servers are not supported yet.
- The monitor machine must be Linux (GNU date, `flock`).

## Uninstall

Remove the cron line, `~/.local/bin/gpu-ntfy-monitor`, `~/.config/gpu-ntfy-monitor/` and `~/.local/state/gpu-ntfy-monitor/`.

## License

MIT
