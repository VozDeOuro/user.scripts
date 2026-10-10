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

- `schedule.json` entries may carry per-script options (all optional, defaults in brackets):
  `notify` (`off` | `failure` | `always`) [`off`], `failAfter` (consecutive failures before the
  first alert) [1], `remindHours` (repeat while still failing; 0 = never) [24], `overlap`
  (`skip` | `queue` | `parallel`) [`skip`], `timeout` (seconds, 0 = none) [0],
  `hangAfter` (consecutive overlap skips before a "looks stuck" alert) [3].
- `<historyDir>/<dir>.jsonl` and `<historyDir>/state.json` (see History); `historyDir` defaults to
  `/boot/config/plugins/user.scripts/history`.
- `/boot/config/plugins/user.scripts/settings.json` (UI prefs, history cap, `historyDir`, flush
  interval)

Cron migration: on install and on every Apply, upstream-format lines (`startCustom.php <path>`) in
`customSchedule.cron` that point to a script with no matching `custom` entry in `schedule.json` are
imported (lines in our generated `run.php` format are never re-imported, so disabling a schedule
sticks) into `schedule.json`
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
   and a parity check/rebuild is running, or `foregroundOnly` for non-manual triggers.
2. Overlap guard (see "Overlapping runs" below).
3. Copy to `/tmp/user.scripts/tmpScripts/<id>/script`, add `#!/bin/bash` if no shebang, strip CR,
   chmod +x (upstream behaviour). `directPHP` scripts keep upstream's path.
4. Rotate logs: `log.txt` → `log.1.txt` … keep 5. `clearLog` only clears the current log.
5. Spawn with `setsid` so the script leads its own process group; write pid to
   `running/<id>`; write `argumentDefault` as arguments. If `timeout` > 0, the Runner aborts the
   group after that many seconds and records status `timeout`.
6. Wait, capture exit code and duration, append history, remove `running/<id>`, write
   `finished/<id>`, call Notify.

Abort: `kill -TERM -<pgid>`, wait 5 s, `kill -KILL -<pgid>`; records exit `aborted` (fixes #18).

#### Overlapping runs

The guard is an exclusive `flock` on `/tmp/user.scripts/locks/<id>.lock`, taken by `run.php`
before spawning and held (inherited fd) until the script's process group exits. A kernel lock is
released automatically if `run.php` or the script dies, so a crash never leaves a stale "running"
state the way upstream's pid file can; `running/<id>` is kept only for UI/legacy compatibility and
is cleaned if its pid is dead.

Per-script `overlap` mode when the lock is held:

- `skip` (default): do not start; record a `skipped` history entry (collapsed, see History) and
  increment the script's `skips` counter in state. When `skips` reaches `hangAfter` (default 3), send
  one "looks stuck" notification ("still running since 14:02 (2h 10m), 3 scheduled runs skipped")
  if `notify` is not `off`. Counter resets when a run starts.
- `queue`: wait on the lock (`flock` blocking) but at most one waiter: a second non-blocking lock
  on `<id>.queue` decides who waits; further triggers are skipped. For backups and syncs where a
  missed run matters.
- `parallel`: no guard (upstream behaviour), for scripts that are safe to run concurrently.

Manual Run / Background from the UI while the script holds the lock returns HTTP 409 "already
running" and the buttons are disabled; Abort is shown instead. `timeout` (above) is the fix for
scripts that hang forever, so `skip` does not silently suppress them for days.

Foreground "Run Script" (live output in a modal) uses the same Runner with trigger `manual`, and
the UI tails the log instead of upstream's `openBox` iframe, so foreground runs also get history
and notifications.

### History (`lib/History.php`)

One JSON object per line:
`{"start":1760065025,"end":1760065031,"exit":0,"trigger":"custom","status":"ok"}`
`status` ∈ `ok`, `failed`, `aborted`, `timeout`, `skipped`. Capped at 200 lines per script
(configurable).

#### Flash wear

RAM is the live copy, persistent storage is a periodic snapshot.

- Every run writes only to `/tmp/user.scripts/history/<id>.jsonl` and
  `/tmp/user.scripts/history/state.json` (tmpfs, zero flash wear).
- Steady-state runs are collapsed: if a run has the same status and exit code as the previous
  entry, the previous entry gets `"repeat": N, "last": <ts>` updated in place instead of a new line.
  An every-minute script that keeps succeeding stays one line, and the history still shows
  "ok ×1440 since 00:00". Durations keep min/avg/max on the collapsed line.
- A sync job (`run.php --sync-history`, own cron line, default hourly) copies changed RAM files to
  `historyDir` with temp file + rename, only when their content hash changed since the last sync.
  It also runs from the `stopping` and `unmounting_disks` emhttp events, so a clean shutdown or
  array stop loses nothing.
- Status changes (ok→failed, failed→ok, timeout, aborted) trigger an immediate sync of that one
  script, because those are the entries worth surviving a power loss.
- On boot (`disks_mounted` / plugin install) RAM history is seeded from `historyDir`.
- `historyDir` can be moved off flash to a pool path in settings (e.g.
  `/mnt/user/system/user.scripts`); if that path is unavailable (array stopped) history stays in
  RAM and syncs once it appears.

Worst case on Tower (7 custom scripts, 2 every-minute): ~24 small rewrites/day per changed script
plus status-change writes, versus upstream-style append-per-run ~2900/day. An unclean power loss
loses at most the last hour of steady-state repeats, never a status change.

### Notify (`lib/Notify.php`)

- Mode per script from `schedule.json` `notify`: `off` (default), `failure`, `always`.
- Failure = exit ≠ 0, aborted by timeout, or "looks stuck" (overlap). Manual aborts and
  skipped runs never notify.
- Calls `/usr/local/emhttp/webGui/scripts/notify -e "User Scripts" -s "<subject>"
  -d "<detail>" -m "<last 10 log lines>" -i alert|warning|normal`.

#### Repeat failures

Per-script state in `state.json`: `health` (`ok` | `failing`), `fails` (consecutive), `firstFail`,
`lastExit`, `lastAlert`, `skips`.

| Event | Sends |
|-------|-------|
| failure, `fails` < `failAfter` | nothing (absorbs one-off blips; set `failAfter: 3` for every-minute checks) |
| failure, `fails` reaches `failAfter` | **alert** "<name> failing: exit 1 (3 runs since 14:02)" |
| further failures, same exit code | nothing |
| failure with a different exit code | **warning** "<name> now failing with exit 2" (new failure mode) |
| still failing `remindHours` after `lastAlert` | **warning** reminder "<name> still failing for 1d 2h, 1,440 failed runs" |
| first success after an alert | **normal** "<name> recovered after 1h 12m (73 failed runs)" |
| success in `always` mode while healthy | **normal** "<name> finished in 6s" |
| `skips` reaches `hangAfter` | **warning** "<name> looks stuck: running 2h 10m, 3 runs skipped" (once per stuck run) |

A recovery is only announced if a failure alert was sent, so blips below `failAfter` stay
silent end to end. State lives in RAM and is synced with history, so a reboot does not re-alert
an old failure.

### API (`api.php`)

POST, form or JSON body, `csrf_token` checked against `/var/local/emhttp/var.ini`. Responses
`{"ok":true,...}` or `{"ok":false,"error":"..."}` with HTTP 4xx.

Actions: `list`, `run`, `background`, `abort`, `log` (id, which=0..5, offset for tail),
`clearLog`, `history` (id, limit), `get` (script body), `save`, `create`, `rename`,
`describe`, `delete`, `schedule.set` (id, frequency, custom, notify, failAfter, remindHours,
overlap, timeout, hangAfter — validates cron and ranges), `apply` (write schedule + cron),
`settings.get`, `settings.set`. `run`/`background` return 409 while the script holds its lock.

### UI (`Userscripts.page`, `ui/app.js`, `ui/app.css`)

- Styling uses Unraid's theme CSS variables and existing classes so it follows azure/black/gray/
  white themes; layout is CSS grid that collapses to stacked cards below 800px.
- Toolbar: search box, filter chips (All, Scheduled, Running, Failed), Add Script button, Apply
  button (enabled when schedule edits are pending, as upstream).
- Script row: status dot + name + description; schedule chip with a human description of the cron
  ("every 5 minutes", "Mon at 03:00") and next run; last run (relative time, duration, status
  pill, "ok ×1440" for collapsed repeats, "failing since 14:02" badge); actions: Run, Background,
  Abort (only while running), Log, ⋯ menu (Edit, Rename, Description, Run options, Delete).
- Run options dialog: notifications (off/failure/always, alert after N failures, remind every N h),
  overlap (skip/queue/parallel), timeout, stuck alert after N skips.
- Log drawer: run selector (current + last 5), live tail every 2 s while running, wrap toggle,
  download (client-side from the fetched text, no extra endpoint), clear.
- Settings dialog: history folder, save interval (15/30/60 min), entries kept per script. History tab: table of recent runs.
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
  `YYYY.MM.DD` (newer than upstream's 2025.06.21), `pluginURL` on raw.githubusercontent.
  Package `.txz` downloaded from the GitHub release for that version, md5 checked.

#### Unraid version support

- Developed and tested on Unraid 7.3.3 (PHP 8.4); declared `min="7.0.0"`.
- Code limited to PHP 8.3 syntax and functions (Unraid 7.0 ships PHP 8.3); CI runs PHPUnit on both
  `php:8.3-cli` and `php:8.4-cli`.
- Only tools present on 7.0 are used: `flock`, `setsid`, `update_cron`, `notify`, emhttp events
  `disks_mounted`, `stopping`, `stopping_svcs`, `unmounting_disks`.
- UI relies on Unraid's theme CSS variables where present, with fallbacks defined in `ui/app.css`
  so 7.0–7.2 themes that lack a variable still render correctly.
- The `.plg` `<CHANGES>` and README state "tested on 7.3.3, should work on 7.0+"; installs on 6.x
  are refused by `min`.
- `pkg_build.sh` builds `archive/user.scripts-<version>.txz` and prints the md5.
- `bin/release` builds the txz locally, stamps version + md5 into the `.plg`, tags and creates the
  GitHub release with that exact file (a CI rebuild would change the md5). GitHub Actions runs the
  tests on PHP 8.3 and 8.4 on every push and verifies the published asset's md5 matches the `.plg`.
- Remove `.DS_Store` files and the stray `log.zip`.

## Testing

- PHPUnit (run in CI and locally via Docker `php:8.3-cli`): header parsing (empty/missing/CRLF),
  `resolve()` rejects `..`, `/`, unknown ids; `validCron` accept/reject table incl. names and
  #19's malformed line; cron generation + migration (fixture = Tower's current
  `customSchedule.cron` + `schedule.json`); `nextRun`; history append/cap, collapse of repeats,
  RAM→disk sync only on hash change, seed on boot; notify state machine table above (notify binary
  stubbed), including failAfter, new exit code, reminder and recovery.
- Runner integration test (bash, in CI container): script that spawns `sleep 300` child → abort
  kills both; exit code captured; overlap `skip` records skipped and fires stuck alert at
  `hangAfter`; `queue` runs exactly one waiter; `parallel` runs both; lock released after the
  runner is `kill -9`ed; `timeout` kills the group and records `timeout`.
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
