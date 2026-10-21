# gpu-ntfy-monitor: setup guide for Claude Code

This repo is a GPU/CPU temperature monitor that pushes to ntfy (`bin/gpu-ntfy-monitor`, run from cron). When the user opens Claude Code here and asks to set it up (or just says hello), run the **Setup** procedure below. Ask questions in the user's language; keep files, comments and config in English.

## Rules

- Run setup **on the machine that will run the monitor** (an always-on Linux box). If you are on another machine, say so and ask whether to continue over ssh.
- Ask one topic at a time, offer the default, and wait for the answer. Do not invent answers for the open questions.
- Never create, copy, move or print ssh keys, and never edit `~/.ssh/config` without asking. If key auth to a server fails, explain what is needed and let the user do `ssh-copy-id` themselves.
- The topic is a password. Print it once at the end, write it only to the config file (`chmod 600`), and never put it, host names, user names or the config in anything committed to this repo.
- Before touching the crontab, show the current one and keep every other entry. Do not add a duplicate line.

## Setup

1. **Requirements.** Check `bash --version` (4+), `date -d @0`, `flock`, `curl`, `ssh`, `crontab`. Report what is missing and stop until it is fixed.
2. **Servers.** List the `Host` entries (no wildcards) from `~/.ssh/config` and ask which to monitor. For each, test `ssh -o BatchMode=yes -o ConnectTimeout=10 <alias> 'nvidia-smi -L'`. Servers that fail are not added until the problem is fixed.
3. **CPU temperature (optional).** For each server run `ssh <alias> sensors`. If a CPU package/Tctl line exists, propose `chip:label` (for example `coretemp-isa-0000:Package`) and ask whether to include it; otherwise skip the CPU for that server.
4. **Thresholds.** Query each server once (`nvidia-smi --query-gpu=temperature.gpu,utilization.gpu,power.draw --format=csv`) and show the temperatures. Ask for a threshold in °C per server (a server is hot when all its GPUs are at or above it). There is no default: the right value depends on the hardware and cooling. Servers may be left without one.
5. **Notifications.** Ask, with defaults:
   - periodic report every N minutes (default 60, aligned to midnight; 0 = none),
   - immediate alerts when hot or unreachable (default on),
   - the alert cooldown in minutes (default 30): the same alert, for the same server and the same reason (hot or unreachable), is not repeated within this time; a reason that clears and returns alerts at once; 0 = every minute,
   - priorities (default 3 for reports, 4 for alerts),
   - how long to keep samples (default 7 days).
6. **ntfy.** Ask whether to use the public `https://ntfy.sh` (default) or their own server. Generate the topic with `echo "gpu-$(LC_ALL=C tr -dc a-z0-9 </dev/urandom | head -c 50)"`. Explain that the topic works as a password and that they must subscribe to it in the ntfy app.
7. **Install.**
   - `mkdir -p ~/.local/bin ~/.config/gpu-ntfy-monitor`; copy `bin/gpu-ntfy-monitor` to `~/.local/bin/` (executable).
   - Write `~/.config/gpu-ntfy-monitor/config` from `config.example` with the answers; `chmod 600`.
   - Run `gpu-ntfy-monitor --dry-run` and show the message. Fix any FAILED host before going on.
   - Ask the user to subscribe to the topic, then run `gpu-ntfy-monitor --test` and ask them to confirm it arrived.
   - Show `crontab -l`, then add `* * * * * $HOME/.local/bin/gpu-ntfy-monitor` (with the absolute home path).
   - Wait for the next cron run and check `~/.local/state/gpu-ntfy-monitor/last_run` and `samples.csv` are updating.
8. **Summary.** Tell the user the topic, where the config, samples and script live, how to change a setting (edit the config, then `--dry-run`), and how to uninstall (see README).

## Maintaining this repo

- The whole tool is `bin/gpu-ntfy-monitor`; `config.example` documents every setting and must be kept in sync with the defaults at the top of the script and with the table in `README.md`.
- After a change: `bash -n bin/gpu-ntfy-monitor`, then test on a Linux host with a scratch config (`GPU_NTFY_CONFIG`, `XDG_STATE_HOME`) and a throwaway topic, using `--dry-run` and `--test`.
- Alert cooldown: run the script without flags (the cron way; `--test` and `--dry-run` never touch the `alerts` state) with `REPORT_EVERY_MINUTES=0`, one hot host (a low `LIMIT`) and one nonexistent host. Expect one push at the first run and none at the second; edit the epoch in `alerts` to age one entry and expect a push; clear the hot reason (high `LIMIT`), bring it back and expect an immediate push; with `ALERT_COOLDOWN_MINUTES=0` expect a push per run. ntfy's message cache lags by a few seconds, so wait before counting messages.
- This is a public repo. Examples use only generic names (`gpu-server-1`, `example.org`, `192.0.2.x`). Before every commit and push, scan the tree and the git history for real host names, user names, e-mail addresses, IP addresses, topics and tokens.
