# hermes-ctl — Build Tasks

CLI tool to manage the Hermes Discord agent as a systemd user service with power profile integration.

---

## Phase 1 — Systemd Service

**Goal:** Hermes runs as a proper background service, survives reboots, logs to journald.

- [ ] Create `~/.config/systemd/user/` directory if it doesn't exist
- [ ] Write `hermes-agent.service` unit file
  - `ExecStart`: full venv python path → `/home/sy/.hermes/hermes-agent/venv/bin/python -m hermes_cli.main gateway run`
  - `WorkingDirectory`: `~/.hermes/hermes-agent/`
  - `Restart=on-failure`, `RestartSec=5`
  - `StandardOutput=journal`, `StandardError=journal`
  - `[Install] WantedBy=default.target`
- [ ] Run `systemctl --user daemon-reload`
- [ ] Test: `systemctl --user start hermes-agent` and confirm it stays running
- [ ] Test: `systemctl --user status hermes-agent` shows active
- [ ] Test: `journalctl --user -u hermes-agent -f` shows live logs

---

## Phase 2 — Scaffold + Help

**Goal:** `hermes-ctl` exists on PATH, handles unknown commands gracefully, prints good help.

- [ ] Create `~/.local/bin/hermes-ctl` (bash script, executable)
- [ ] Add shebang `#!/usr/bin/env bash` and `set -euo pipefail`
- [ ] Define color vars using bold/dim/reset only (inherits terminal theme)
  - `BOLD`, `DIM`, `RESET`, `OK` (bold), `ERR` (bold), `WARN` (bold)
- [ ] Source config file if it exists (`~/.config/hermes-ctl/config`)
- [ ] Implement `cmd_help()` — grouped output:
  - Section: COMMANDS (start, stop, restart, status, logs, boot, timer, update)
  - Section: OPTIONS (none yet — placeholder)
  - Section: EXAMPLES (3–4 copy-paste lines)
- [ ] Default (no args) → call `cmd_status`
- [ ] Unknown command → print error + one-line usage, exit 1
- [ ] Test: `hermes-ctl help` renders correctly
- [ ] Test: `hermes-ctl` with no args calls status (even if status isn't built yet, just confirm routing)
- [ ] Test: `hermes-ctl bogus` exits 1 with a useful message

---

## Phase 3 — Start & Stop

**Goal:** Core service lifecycle with power profile integration.

- [ ] Implement `cmd_start()`
  - Check if service is already running → prompt "restart or cancel?" (y/N)
  - If starting fresh: `• Starting service… ✔` step output
  - `systemctl --user start hermes-agent`
  - Confirm active via `systemctl --user is-active`
  - If `powerprofilesctl` is available: `• Switching to power-saver… ✔`
  - `powerprofilesctl set power-saver`
  - Exit 0 on success
- [ ] Implement `cmd_stop()`
  - Check if already stopped → print "already stopped", exit 0 (no profile prompt)
  - `• Stopping service… ✔`
  - `systemctl --user stop hermes-agent`
  - If `powerprofilesctl` available: prompt "Restore power profile:" with list from `powerprofilesctl list`
  - Apply selected profile
  - Exit 0
- [ ] `ppd_available()` helper — checks for `/usr/bin/powerprofilesctl`, returns 0/1
- [ ] Test: `hermes-ctl start` from stopped state — both steps print ✔
- [ ] Test: `hermes-ctl start` when already running — prompt appears
- [ ] Test: `hermes-ctl stop` from running state — profile prompt appears with real list
- [ ] Test: `hermes-ctl stop` when already stopped — exits cleanly

---

## Phase 4 — Status

**Goal:** Rich two-column status table with all relevant info.

- [ ] Implement `cmd_status()`
- [ ] Detect service state: `systemctl --user is-active hermes-agent`
- [ ] Get PID: `systemctl --user show hermes-agent --property=MainPID --value`
- [ ] Calculate uptime from `ActiveEnterTimestamp` → format as `2h 14m`
  - Helper `format_uptime <seconds>` — outputs e.g. `2h 14m`, `45m`, `3d 2h`
- [ ] Get current power profile: `powerprofilesctl get` (skip if ppd not available)
- [ ] Detect active timer (Phase 6 prerequisite — stub "none" for now)
- [ ] Render two-column table:
  ```
  Service   ●  running (PID 12345)    ← or ○ stopped
  Power     ⚡  power-saver           ← or current profile / unavailable
  Uptime    ⏱   2h 14m               ← hidden when stopped
  Timer     ⏳  stops in 1h 22m      ← hidden when no timer active
  Version   🤖  hermes commit abc1234 ← from git -C ~/.hermes/hermes-agent log
  ```
- [ ] Test: status when running — all rows show correctly
- [ ] Test: status when stopped — Uptime row hidden, correct profile shown
- [ ] Test: status when ppd not available — Power row shows "unavailable" without error

---

## Phase 5 — Restart & Logs

**Goal:** Convenience commands that wrap existing primitives.

- [ ] Implement `cmd_restart()`
  - Stop (skip profile prompt — staying in power-saver)
  - Start (skip "already running" check — we just stopped)
  - Single step output: `• Restarting… ✔`
- [ ] Implement `cmd_logs()`
  - Read `LOG_LINES` from config (default: 50)
  - `journalctl --user -u hermes-agent -n "$LOG_LINES" -f`
- [ ] Test: `hermes-ctl restart` stops and starts cleanly
- [ ] Test: `hermes-ctl logs` streams live output, Ctrl-C exits cleanly

---

## Phase 6 — Timer

**Goal:** Auto-stop after a duration or at a wall-clock time.

- [ ] Implement `cmd_timer()`
  - Sub-commands: `<duration|HH:MM>`, `status`, `cancel`
  - Parse argument:
    - `HH:MM` format → convert to systemd `OnCalendar` time
    - `Nh` / `Nm` / `NhMm` → systemd `OnActiveSec` duration
  - Cancel any existing `hermes-ctl-autostop.timer` before setting new one
  - Use `systemd-run --user --on-active=<duration>` (or `--on-calendar=<time>`)
  - Unit name: `hermes-ctl-autostop`
  - `ExecStart`: `hermes-ctl stop` (so power profile prompt fires)
  - On fire: `notify-send "Hermes Agent" "Auto-stopped — timer expired"` + journal log
- [ ] Implement `cmd_timer_status()`
  - Check if `hermes-ctl-autostop.timer` is active
  - Get `NextElapseUSecRealtime` → format as `stops in 1h 22m (at 16:45)`
- [ ] Implement `cmd_timer_cancel()`
  - `systemctl --user stop hermes-ctl-autostop.timer` + print confirmation
- [ ] Wire timer row into `cmd_status()` (replace stub from Phase 4)
- [ ] Test: `hermes-ctl timer 30m` → timer set, status shows countdown
- [ ] Test: `hermes-ctl timer 14:30` → timer set for wall-clock time
- [ ] Test: `hermes-ctl timer status` shows remaining time
- [ ] Test: `hermes-ctl timer cancel` removes the timer
- [ ] Test: timer fires → notify-send appears + service stops + profile prompt fires

---

## Phase 7 — Boot Toggle

**Goal:** Enable/disable Hermes autostart on login.

- [ ] Implement `cmd_boot()`
  - Arg: `on` or `off`
  - Unknown arg → error + usage
  - `on`: prompt `Enable autostart on login? [y/N]` → `systemctl --user enable hermes-agent` → print `✔ Autostart enabled`
  - `off`: prompt `Disable autostart on login? [y/N]` → `systemctl --user disable hermes-agent` → print `✔ Autostart disabled`
- [ ] Test: `hermes-ctl boot on` → prompts, enables, confirms
- [ ] Test: `hermes-ctl boot off` → prompts, disables, confirms
- [ ] Test: `hermes-ctl boot` with no arg → error

---

## Phase 8 — Update

**Goal:** Pull latest Hermes code, show diff, offer restart.

- [ ] Implement `cmd_update()`
  - Check `~/.hermes/hermes-agent/` is a git repo (`git -C ... rev-parse`)
  - Hard fail with clear error if not a git repo
  - Record if service is currently running
  - `git -C ~/.hermes/hermes-agent fetch`
  - Show diff: `git -C ~/.hermes/hermes-agent log HEAD..@{u} --oneline`
  - If no new commits → print "Already up to date", exit 0
  - Prompt `Apply update? [y/N]`
  - `git -C ~/.hermes/hermes-agent pull`
  - If was running → prompt `Restart Hermes now? [y/N]` → `cmd_restart`
- [ ] Test: update when already up to date → exits cleanly
- [ ] Test: update when commits available → shows log, prompts, pulls
- [ ] Test: update when not a git repo → clear error

---

## Phase 9 — Config File

**Goal:** Persistent user preferences loaded at startup.

- [ ] Define config path: `~/.config/hermes-ctl/config`
- [ ] Supported keys:
  - `LOG_LINES=50` — default lines for `hermes-ctl logs`
  - `RESTORE_PROFILE=balanced` — skips stop prompt if set
  - `NOTIFY_ON_TIMER=true` — toggle notify-send on timer fire
  - `SERVICE_NAME=hermes-agent` — override systemd unit name
- [ ] Source config at top of script (after defaults are set)
- [ ] `cmd_config()` — print current effective config (file values + defaults)
- [ ] Test: set `LOG_LINES=100` in config → `hermes-ctl logs` uses 100
- [ ] Test: set `RESTORE_PROFILE=balanced` → stop skips prompt, uses it directly
- [ ] Test: `hermes-ctl config` shows current effective values

---

## Phase 10 — Polish & Edge Cases

**Goal:** Production-ready, handles messy real-world state.

- [ ] Handle `systemd --user` not available (non-systemd systems) — warn and exit 1
- [ ] Handle `notify-send` not available — skip silently (same as ppd)
- [ ] Ensure all prompts handle EOF / Ctrl-C gracefully (trap EXIT)
- [ ] Confirm `set -euo pipefail` doesn't break any interactive prompts
- [ ] Add `hermes-ctl version` fallback if someone runs it — route to status
- [ ] Final help text pass — verify all commands listed, examples accurate
- [ ] Verify script is POSIX-compatible enough to survive `bash --posix`
- [ ] Verify colors degrade gracefully in a no-color terminal (`NO_COLOR` env var)
- [ ] End-to-end test: fresh machine simulation (stop everything, run start → timer → auto-stop)

---

## Design Reference

| Setting | Value |
|---|---|
| Service unit | `~/.config/systemd/user/hermes-agent.service` |
| Script path | `~/.local/bin/hermes-ctl` |
| Config path | `~/.config/hermes-ctl/config` |
| Hermes dir | `~/.hermes/hermes-agent/` |
| Venv python | `~/.hermes/hermes-agent/venv/bin/python` |
| Power on start | `power-saver` |
| Power on stop | prompt (or `RESTORE_PROFILE` from config) |
| Timer unit | `hermes-ctl-autostop` (via systemd-run) |
| Output style | Bold/dim/reset, unicode icons, inherits terminal theme |
