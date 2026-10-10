# User Scripts fork (VozDeOuro/user.scripts) — design

Date: 2026-10-10
Status: approved in chat, awaiting spec review

## Why

Upstream Squidly271/user.scripts (v2025.06.21) is dormant (last push 2025-06-22) and broken on
Unraid 7.3.3: the script table's first cell is `nowrap` with `width=30%`, renders 1912px wide and
pushes Run / Background / schedule / log controls off-screen. Open upstream bugs make scheduling
unreliable (#17 `is_file()` on a directory, #19 malformed custom cron with an extra `*`, #20 custom
dispatcher ignoring the cron field), abort leaves child processes running (#18), and
`startBackground.php` fatals on an empty path (seen in Tower's phplog). There are no failure
notifications (#21, #23) and no run history.

## Goals

1. The page works on Unraid 7.3+ (desktop and mobile, all Unraid themes).
2. Schedules run exactly when configured; abort really stops a script.
3. Per-script notifications through Unraid's notify system (reaches Telegram via the existing agent).
4. Per-script run history and a better log viewer.
5. Drop-in replacement: same plugin name, same data on flash, rollback by reinstalling upstream.

## Non-goals

- No new data format for scripts. A script is still `/boot/config/plugins/user.scripts/scripts/<dir>/`
  with `script`, optional `name`, `description`.
- No build toolchain or JS framework. Vanilla PHP 8 + vanilla JS + CSS.
- No multi-user/permissions model (Unraid webGUI is root-only).
- Not "User Scripts Enhanced" compatibility beyond what upstream already has.

## Compatibility contract

Unchanged and read as-is:

- `/boot/config/plugins/user.scripts/scripts/<dir>/{script,name,description}`
- Script header variables (`#name=`, `#description=`, `#arrayStarted=`, `#foregroundOnly=`,
  `#backgroundOnly=`, `#clearLog=`, `#noParity=`, `#argumentDefault=`, `#directPHP=`)
- `/boot/config/plugins/user.scripts/schedule.json` (keyed by script path; `frequency`, `id`, `custom`)
- `/boot/config/plugins/user.scripts/customSchedule.cron` as the file `update_cron` picks up
- Runtime dir `/tmp/user.scripts/{running,finished,tmpScripts}`
- Event hooks `event/disks_mounted` (At Startup of Array / At First Array Start Only) and
  `event/stopping_svcs` (At Stopping of Array)

Added (ignored by upstream, so rollback stays safe):

- `schedule.json` entries may carry `notify` (`off` | `failure` | `always`, default `off`).
- `/boot/config/plugins/user.scripts/history/<dir>.jsonl`
- `/boot/config/plugins/user.scripts/settings.json` (UI prefs: default sort, history cap)

Cron migration: on install and on every Apply, lines in `customSchedule.cron` that point to a
script with no matching `custom` entry in `schedule.json` are imported into `schedule.json`
(frequency `custom`, cron from the line) before regenerating the file. On Tower this preserves the
hand-added `web-sites-status` (`* * * * *`) and `cf-dns-sync` (`*/5 * * * *`) entries if they
are not already in `schedule.json` (Tower currently has 7 custom entries and 7 cron lines; the
migration is a safety net and a no-op when they match). Blank lines and comments are dropped.

## Architecture

```
Userscripts.page  ── fetch JSON ──▶  api.php  ──▶  lib/ (Scripts, Schedule, History, Runner, Notify)
                                                     ▲
cron / customSchedule.cron / event hooks ──▶  run.php ┘
```

Files under `/usr/local/emhttp/plugins/user.scripts/`:

| File | Role |
|------|------|
| `Userscripts.page` | Page shell: header markup, loads `ui/app.js` + `ui/app.css`, passes CSRF token |
| `ui/app.js`, `ui/app.css` | The UI (list, drawers, editor, log viewer) |
| `api.php` | JSON endpoint, one `action` per request, CSRF-checked |
| `run.php` | The only way a script runs (all triggers) |
| `lib/Scripts.php` | List scripts, read header variables, create/rename/delete, path validation |
| `lib/Schedule.php` | Read/write `schedule.json`, cron validation, cron file generation + migration, next-run calc |
| `lib/History.php` | Append/read/cap JSONL history |
| `lib/Runner.php` | Prepare tmp copy, spawn, track pid/pgid, abort, log rotation |
| `lib/Notify.php` | Build and send Unraid notifications |
| `cron/*.sh`, `event/*` | Thin wrappers calling `run.php --frequency <x>` |
| `javascript/ace/` | Kept (editor) |

Legacy entry points (`exec.php`, `startSchedule.php`, `startCustom.php`, `startBackground.php`,
`backgroundScript.sh`, `startScript.sh`) are kept as thin shims that call `run.php`, so any cron
line or external caller written for upstream still works. `startCustom.php` with an empty or
missing path exits 0 silently (fixes the phplog fatal).

## Components

### Scripts (`lib/Scripts.php`)

- `list()` returns, per script dir: `id` (dir name), `name`, `description`, `path`, header
  variables, schedule, notify mode, running state, last history entry.
- `resolve(string $id): string` — the only way an API input becomes a path. `$id` must match an
  existing directory name under the scripts root (no `/`, no `..`, compared against `scandir`);
  otherwise throws. Every API action that touches a script goes through it. This closes upstream's
  arbitrary-path `convertScript`/`getScriptVariables` and the `rm -rf` on an unchecked name in
  `deleteScript`.
- Header parsing replaces `getRawVariables`: reads leading `#key=value` lines, tolerates empty
  files and missing files (returns `[]`), never fatals.
- Name sanitizing for create/rename keeps upstream's character rules.

### Schedule (`lib/Schedule.php`)

- Frequencies (exactly upstream's stored values, so rollback works): `disabled`, `hourly`,
  `daily`, `weekly`, `monthly`, `custom`, `start` (At Startup of Array), `boot` (At First Array
  Start Only), `stop` (At Stopping of Array). No new presets; the UI maps values to labels.
  (Tower today: 4 boot, 7 custom, 3 daily, 10 disabled.)
- `validCron(string)`: 5 fields; each field `*`, number, range, list, step, or (day-of-week) names
  `mon`–`sun`, (month) `jan`–`dec`, case-insensitive (fixes #12, #13). Rejects anything else, so a
  malformed line never reaches `customSchedule.cron` (fixes #19).
- `writeCron()`: one line per custom entry,
  `<cron> /usr/local/emhttp/plugins/user.scripts/run.php --script '<id>' --trigger custom > /dev/null 2>&1`,
  then `update_cron`. Each line carries its own cron (fixes #20). Migration step runs first.
- `nextRun(cron, now)`: minute-step search up to 366 days, used for the UI "next run" column.

### Runner (`lib/Runner.php`, `run.php`)

`run.php --script <id> --trigger <manual|background|hourly|daily|weekly|monthly|custom|start|boot|stop>`
or `run.php --frequency <x>` (runs every script scheduled with that frequency).

Per run:

1. Resolve script; skip (with a syslog line) if `arrayStarted` and array not started, `noParity`
   and a parity check/rebuild is running, `foregroundOnly` for non-manual triggers, or the script
   is already running (no overlapping runs; the skip is logged and recorded in history as
   `skipped`).
2. Copy to `/tmp/user.scripts/tmpScripts/<id>/script`, add `#!/bin/bash` if no shebang, strip CR,
   chmod +x (upstream behaviour). `directPHP` scripts keep upstream's path.
3. Rotate logs: `log.txt` → `log.1.txt` … keep 5. `clearLog` only clears the current log.
4. Spawn with `setsid` so the script leads its own process group; write pid to
   `running/<id>`; write `argumentDefault` as arguments.
5. Wait, capture exit code and duration, append history, remove `running/<id>`, write
   `finished/<id>`, call Notify.

Abort: `kill -TERM -<pgid>`, wait 5 s, `kill -KILL -<pgid>`; records exit `aborted` (fixes #18).

Foreground "Run Script" (live output in a modal) uses the same Runner with trigger `manual`, and
the UI tails the log instead of upstream's `openBox` iframe, so foreground runs also get history
and notifications.

### History (`lib/History.php`)

One JSON object per line in `history/<id>.jsonl`:
`{"start":1760065025,"end":1760065031,"exit":0,"trigger":"custom","status":"ok"}`
`status` ∈ `ok`, `failed`, `aborted`, `skipped`. Capped at 200 lines per script (configurable in
settings); the cap is applied by rewriting the file when it exceeds cap + 50, so flash writes stay
rare for every-minute scripts. Writes are atomic (temp file + rename).

Flash-wear note: an every-minute script produces ~1440 appends/day. To limit wear, history for
runs with trigger `custom`/`hourly` whose status equals the previous run's status and exit code is
buffered in `/tmp/user.scripts/history/` and flushed to flash every 15 minutes and on any status
change. The UI reads flash + buffer merged.

### Notify (`lib/Notify.php`)

- Mode per script from `schedule.json` `notify`: `off` (default), `failure`, `always`.
- Failure = exit ≠ 0 or aborted. Skipped runs never notify.
- Calls `/usr/local/emhttp/webGui/scripts/notify -e "User Scripts" -s "<name> failed (exit N)"
  -d "<trigger>, ran 6s" -m "<last 10 log lines>" -i alert|normal`.
- Failure notifications for the same script are not repeated while it keeps failing with the same
  exit code; the next success sends a "recovered" notice (only when mode is `failure` or `always`).

### API (`api.php`)

POST, form or JSON body, `csrf_token` checked against `/var/local/emhttp/var.ini`. Responses
`{"ok":true,...}` or `{"ok":false,"error":"..."}` with HTTP 4xx.

Actions: `list`, `run`, `background`, `abort`, `log` (id, which=0..5, offset for tail),
`clearLog`, `downloadLog`, `history` (id, limit), `get` (script body), `save`, `create`, `rename`,
`describe`, `delete`, `schedule.set` (id, frequency, custom, notify — validates cron), `apply`
(write schedule + cron), `settings.get`, `settings.set`.

### UI (`Userscripts.page`, `ui/app.js`, `ui/app.css`)

- Styling uses Unraid's theme CSS variables and existing classes so it follows azure/black/gray/
  white themes; layout is CSS grid that collapses to stacked cards below 800px.
- Toolbar: search box, filter chips (All, Scheduled, Running, Failed), Add Script button, Apply
  button (enabled when schedule edits are pending, as upstream).
- Script row: status dot + name + description; schedule chip with a human description of the cron
  ("every 5 minutes", "Mon at 03:00") and next run; last run (relative time, duration, status
  pill); actions: Run, Background, Abort (only while running), Log, ⋯ menu (Edit, Rename,
  Description, Notifications, Delete).
- Log drawer: run selector (current + last 5), live tail every 2 s while running, wrap toggle,
  download, clear. History tab: table of recent runs.
- Edit drawer: ACE editor (bundled), header-variable toggles that write `#key=value` lines at the
  top of the script, Save.
- Polling `list` every 5 s while the page is visible (pauses on `visibilitychange`).
- Keyboard: `/` focuses search, `Esc` closes drawers. Visible focus states.

## Error handling

- API never emits PHP warnings into JSON; errors are caught and returned as `{"ok":false}`.
- Invalid cron → the schedule field shows the error inline and Apply stays disabled.
- Missing/empty script file → row shows "script file missing", Run disabled.
- Runner failures (cannot copy, cannot spawn) recorded as `failed` history with exit `-1` and
  logged to syslog.

## Repo, build, release

- Public fork `VozDeOuro/user.scripts`, branch `master`, upstream remote kept.
- `.plg`: `author` "Andrew Zawadzki, VozDeOuro", `github` "VozDeOuro/user.scripts", version
  `YYYY.MM.DD` (newer than upstream's 2025.06.21), `min="7.0.0"`, `pluginURL` on raw.githubusercontent.
  Package `.txz` downloaded from the GitHub release for that version, md5 checked.
- `pkg_build.sh` builds `archive/user.scripts-<version>.txz` and prints the md5.
- GitHub Actions on tag `v*`: run tests, build txz, create release with the txz, commit the
  updated version + md5 into the `.plg`.
- Remove `.DS_Store` files and the stray `log.zip`.

## Testing

- PHPUnit (run in CI and locally via Docker `php:8.3-cli`): header parsing (empty/missing/CRLF),
  `resolve()` rejects `..`, `/`, unknown ids; `validCron` accept/reject table incl. names and
  #19's malformed line; cron generation + migration (fixture = Tower's current
  `customSchedule.cron` + `schedule.json`); `nextRun`; history append/cap/buffer flush; notify
  dedupe logic (notify binary stubbed).
- Runner integration test (bash, in CI container): script that spawns `sleep 300` child → abort
  kills both; exit code captured; overlapping run skipped.
- Playwright smoke test against Tower after install: page loads with no console errors, all
  scripts listed, controls visible at 1600px and 390px widths, open log drawer, run a harmless
  test script and see status go running → ok.

## Rollout on Tower

1. Back up `/boot/config/plugins/user.scripts/` and `user.scripts.plg` to
   `/boot/config/plugins/user.scripts-backup-<date>/`; save `crontab -l` output.
2. Install the fork `.plg` URL via Plugins → Install Plugin.
3. Verify: 24 scripts listed; `crontab -l` contains the same 7 custom schedules (now via
   `run.php`) plus hourly/daily/weekly/monthly wrappers; run a test script; check history and a
   test notification reach Telegram.
4. Watch for 24 h: every-minute `web-sites-status` keeps updating `status.json`; `cf-dns-sync`
   runs every 5 min.
5. Rollback: install upstream `.plg` URL; data is compatible (extra keys ignored). Cron lines then
   regenerate on next upstream Apply.
