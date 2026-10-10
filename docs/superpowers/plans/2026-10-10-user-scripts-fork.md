# User Scripts Fork Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship VozDeOuro/user.scripts: a drop-in replacement for the Unraid User Scripts plugin with a working responsive UI, correct scheduling, overlap control, run history with low flash wear, and deduplicated failure notifications.

**Architecture:** Plain PHP 8.3 library classes in `lib/` (Paths, Settings, Vars, Scripts, Cron, Schedule, History, State, Alerts, Runner, Sys) used by two entry points: `run.php` (every execution: cron, events, UI) and `api.php` (JSON for the UI). The UI is vanilla JS/CSS loaded by `Userscripts.page`. Upstream entry points stay as thin shims. Data format on flash is unchanged; new data is additive.

**Tech Stack:** PHP 8.3 (Unraid 7.0) / 8.4 (Unraid 7.3.3), PHPUnit 11, bash, vanilla ES2020 JS, CSS grid, ACE editor (bundled), Slackware `makepkg`, GitHub Actions, Playwright (smoke test from the workstation).

**Spec:** `docs/superpowers/specs/2026-10-10-user-scripts-fork-design.md`

## Global Constraints

- Plugin name stays `user.scripts`; data root `/boot/config/plugins/user.scripts/`; runtime root `/tmp/user.scripts/`.
- Script layout unchanged: `scripts/<dir>/{script,name,description}`.
- `schedule.json` stays keyed by script path `/boot/config/plugins/user.scripts/scripts/<dir>/script` with `script`, `frequency`, `id` (`schedule<dir>`), `custom`; new keys only added: `notify`, `failAfter`, `remindHours`, `overlap`, `timeout`, `hangAfter`.
- Frequencies exactly: `disabled`, `hourly`, `daily`, `weekly`, `monthly`, `custom`, `start`, `boot`, `stop`.
- Option defaults: `notify` `off`, `failAfter` 1, `remindHours` 24, `overlap` `skip`, `timeout` 0, `hangAfter` 3.
- History statuses: `ok`, `failed`, `aborted`, `timeout`, `skipped`. History cap 200 per script (configurable).
- Per-run writes go only to tmpfs (`/tmp/user.scripts/history/`); flash is written by `--sync-history` (hourly default, on `stopping`/`unmounting_disks`) and immediately on status change.
- PHP syntax/functions limited to 8.3. No Composer runtime dependencies (PHPUnit is dev-only).
- `.plg`: `min="7.0.0"`; text "Developed and tested on Unraid 7.3.3, should work on 7.0+".
- No JS framework, no build step.
- Every API input that names a script goes through `Scripts::resolve()`.
- Git commits end with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.

## Review Focus

1. Script directory names with spaces, quotes, `&`, `%`, or unicode (Tower has `clear subtitles`): run, schedule, cron line, log and history must all work; `%` must be escaped in cron. Test added in Task 4 (cron line) and Task 7 (runner).
2. `customSchedule.cron` hand-edited lines in upstream format, including #19's malformed extra `*`: imported when valid, dropped when not, never duplicated. Test in Task 4.
3. Concurrent runs racing on `state.json` / history files (two every-minute scripts finishing together): no lost updates, no corrupt JSON. Test in Task 5 (concurrent append) and Task 6 (state lock).
4. History directory unavailable (array stopped, pool path missing): runs keep working in RAM, sync later. Test in Task 5.
5. Script that ignores SIGTERM or forks a background child: abort and timeout must still kill the whole group. Test in Task 7.

---

## File Structure

All plugin files live under `P = source/user.scripts/usr/local/emhttp/plugins/user.scripts/`.

| Path | Responsibility |
|------|----------------|
| `P/lib/Paths.php` | All filesystem locations; overridable via env for tests |
| `P/lib/Sys.php` | Side effects: syslog, var.ini, detached spawn, update_cron, atomic write |
| `P/lib/Settings.php` | `settings.json` with defaults |
| `P/lib/Vars.php` | Script header `#key=value` parsing |
| `P/lib/Scripts.php` | Script ids, `resolve()`, name/description, CRUD |
| `P/lib/Cron.php` | Cron validate/parse/next/describe |
| `P/lib/Schedule.php` | `schedule.json` read/write, options, cron migration + generation |
| `P/lib/History.php` | RAM history, collapse, cap, sync, seed |
| `P/lib/State.php` | Per-script alert state (locked read-modify-write) |
| `P/lib/Alerts.php` | Notification decision table + send |
| `P/lib/Runner.php` | Conditions, lock/overlap, spawn, timeout, abort, logs |
| `P/lib/bootstrap.php` | Requires all lib files, `Paths::fromEnv()` |
| `P/run.php` | CLI entry point |
| `P/api.php` | JSON API |
| `P/helpers.php` | Legacy functions (`getRawVariables`, …) on top of `Vars` |
| `P/startSchedule.php`, `startCustom.php`, `startBackground.php`, `backgroundScript.sh`, `startScript.sh`, `exec.php` | Legacy shims |
| `P/cron/user.script.start.{hourly,daily,weekly,monthly}.sh` | Call `run.php --frequency` |
| `P/event/{disks_mounted,stopping_svcs,stopping,unmounting_disks}` | Event hooks |
| `P/Userscripts.page` | Page shell |
| `P/ui/app.css`, `P/ui/app.js` | UI |
| `tests/` | PHPUnit tests + `tests/runner.sh` integration test |
| `composer.json`, `phpunit.xml`, `bin/test` | Dev tooling |
| `plugins/user.scripts.plg` | Plugin manifest |
| `pkg_build.sh`, `.github/workflows/release.yml` | Build/release |

Removed: `caCredits.php`, `scriptSupportFile.php`, `executeScript.php`, `showLog.php`, `arrayNotStarted.sh`, `log.zip`, all `.DS_Store`, `copy_to_git.sh`, unused ACE files.

---

### Task 1: Repo cleanup, test harness, Paths, Sys, Settings, Vars

**Files:**
- Delete: `.DS_Store`, `source/.DS_Store`, `copy_to_git.sh`, `P/log.zip`, `P/caCredits.php`, `P/scriptSupportFile.php`, `P/executeScript.php`, `P/showLog.php`, `P/arrayNotStarted.sh`
- Create: `composer.json`, `phpunit.xml`, `bin/test`, `.gitignore`, `P/lib/Paths.php`, `P/lib/Sys.php`, `P/lib/Settings.php`, `P/lib/Vars.php`, `P/lib/bootstrap.php`, `tests/TestCase.php`, `tests/VarsTest.php`, `tests/SettingsTest.php`

**Interfaces:**
- Produces: `Paths::$flash, $tmp, $plugin, $varIni, $notify, $updateCron` (string), `Paths::scripts(): string`, `Paths::fromEnv(): void`; `Sys::log(string $msg): void`, `Sys::var(): array`, `Sys::atomicWrite(string $file, string $data): void`, `Sys::spawn(string $cmd): void`, `Sys::updateCron(): void`; `Settings::get(): array`, `Settings::set(array $s): array`; `Vars::raw(string $file): array<string,string>`, `Vars::typed(string $file): array<string,mixed>`, `Vars::BOOLEANS`; `tests/TestCase` with `$this->root` sandbox and `makeScript(string $id, string $body, ?string $name=null): string`.

- [ ] **Step 1: Clean repo and add tooling**

```bash
cd ~/Desktop/things/user.scripts
P=source/user.scripts/usr/local/emhttp/plugins/user.scripts
git rm -q --cached .DS_Store source/.DS_Store 2>/dev/null; rm -f .DS_Store source/.DS_Store
git rm -q copy_to_git.sh $P/log.zip $P/caCredits.php $P/scriptSupportFile.php $P/executeScript.php $P/showLog.php $P/arrayNotStarted.sh
mkdir -p $P/lib $P/ui tests bin
```

`composer.json`:
```json
{
  "name": "vozdeouro/user.scripts",
  "description": "Unraid User Scripts plugin (VozDeOuro fork)",
  "require-dev": { "phpunit/phpunit": "^11.5" },
  "config": { "platform": { "php": "8.3.0" } }
}
```

`phpunit.xml`:
```xml
<?xml version="1.0"?>
<phpunit bootstrap="tests/TestCase.php" colors="true" cacheDirectory=".phpunit.cache">
  <testsuites><testsuite name="unit"><directory>tests</directory></testsuite></testsuites>
</phpunit>
```

`bin/test` (chmod +x):
```bash
#!/bin/bash
# Runs PHPUnit in Docker on the PHP version given (default 8.3), then the runner integration test.
set -e
cd "$(dirname "$0")/.."
V=${1:-8.3}
[ -d vendor ] || docker run --rm -u "$(id -u):$(id -g)" -v "$PWD":/app -w /app composer:2 install -q
docker run --rm -u "$(id -u):$(id -g)" -v "$PWD":/app -w /app php:$V-cli vendor/bin/phpunit "${@:2}"
docker run --rm -v "$PWD":/app -w /app php:$V-cli bash -c "apt-get -qq update >/dev/null && apt-get -qq install -y util-linux procps >/dev/null; bash tests/runner.sh"
```

`.gitignore`:
```
vendor/
.phpunit.cache/
composer.lock
```

- [ ] **Step 2: Write Paths, Sys, Settings, bootstrap**

`P/lib/Paths.php`:
```php
<?php
declare(strict_types=1);

// Every filesystem location the plugin touches. Tests point these at a sandbox via env.
final class Paths
{
    public static string $flash = '/boot/config/plugins/user.scripts';
    public static string $tmp = '/tmp/user.scripts';
    public static string $plugin = '/usr/local/emhttp/plugins/user.scripts';
    public static string $varIni = '/var/local/emhttp/var.ini';
    public static string $notify = '/usr/local/emhttp/webGui/scripts/notify';
    public static string $updateCron = '/usr/local/sbin/update_cron';

    private const ENV = [
        'flash' => 'US_FLASH', 'tmp' => 'US_TMP', 'plugin' => 'US_PLUGIN',
        'varIni' => 'US_VARINI', 'notify' => 'US_NOTIFY', 'updateCron' => 'US_UPDATE_CRON',
    ];

    public static function fromEnv(): void
    {
        foreach (self::ENV as $prop => $env) {
            $v = getenv($env);
            if ($v !== false && $v !== '') self::$$prop = $v;
        }
    }

    public static function scripts(): string { return self::$flash . '/scripts'; }
    public static function scheduleFile(): string { return self::$flash . '/schedule.json'; }
    public static function cronFile(): string { return self::$flash . '/customSchedule.cron'; }
    public static function settingsFile(): string { return self::$flash . '/settings.json'; }
    public static function ramHistory(): string { return self::$tmp . '/history'; }
    public static function locks(): string { return self::$tmp . '/locks'; }
    public static function running(): string { return self::$tmp . '/running'; }
    public static function finished(): string { return self::$tmp . '/finished'; }
    public static function aborted(): string { return self::$tmp . '/aborted'; }
    public static function work(string $id): string { return self::$tmp . '/tmpScripts/' . $id; }
}
```

`P/lib/Sys.php`:
```php
<?php
declare(strict_types=1);

// Side effects that touch the OS, kept in one place so tests can stub them via env.
final class Sys
{
    /** @var list<string> commands Sys::spawn() would have run (test mode only) */
    public static array $spawned = [];

    public static function log(string $msg): void
    {
        if (getenv('US_TEST')) { file_put_contents(Paths::$tmp . '/syslog', $msg . "\n", FILE_APPEND); return; }
        openlog('user.scripts', 0, LOG_USER);
        syslog(LOG_INFO, $msg);
        closelog();
    }

    /** var.ini as an array; missing file means "array started, no parity check" (dev/test boxes). */
    public static function var(): array
    {
        $v = is_file(Paths::$varIni) ? @parse_ini_file(Paths::$varIni) : false;
        return is_array($v) ? $v : ['mdState' => 'STARTED', 'mdResyncPos' => '0'];
    }

    public static function atomicWrite(string $file, string $data): void
    {
        $dir = dirname($file);
        if (!is_dir($dir)) mkdir($dir, 0777, true);
        $tmp = $dir . '/.' . basename($file) . '.' . getmypid() . '.tmp';
        if (file_put_contents($tmp, $data) === false) throw new RuntimeException("cannot write $file");
        rename($tmp, $file);
    }

    /** Run a shell command fully detached from the web request (survives php-fpm). */
    public static function spawn(string $cmd): void
    {
        if (getenv('US_TEST')) { self::$spawned[] = $cmd; return; }
        exec('echo ' . escapeshellarg($cmd) . ' | at -M NOW >/dev/null 2>&1');
    }

    public static function updateCron(): void
    {
        if (is_executable(Paths::$updateCron)) exec(escapeshellarg(Paths::$updateCron));
    }
}
```

`P/lib/Settings.php`:
```php
<?php
declare(strict_types=1);

final class Settings
{
    public static function defaults(): array
    {
        return ['historyCap' => 200, 'historyDir' => Paths::$flash . '/history', 'syncMinutes' => 60];
    }

    public static function get(): array
    {
        $raw = is_file(Paths::settingsFile()) ? json_decode((string)file_get_contents(Paths::settingsFile()), true) : null;
        return self::clean(array_merge(self::defaults(), is_array($raw) ? $raw : []));
    }

    public static function set(array $in): array
    {
        $s = self::clean(array_merge(self::get(), array_intersect_key($in, self::defaults())));
        Sys::atomicWrite(Paths::settingsFile(), json_encode($s, JSON_PRETTY_PRINT | JSON_UNESCAPED_SLASHES));
        return $s;
    }

    private static function clean(array $s): array
    {
        $s['historyCap'] = max(10, min(5000, (int)$s['historyCap']));
        $s['syncMinutes'] = in_array((int)$s['syncMinutes'], [15, 30, 60], true) ? (int)$s['syncMinutes'] : 60;
        $dir = rtrim((string)$s['historyDir'], '/');
        $s['historyDir'] = ($dir !== '' && str_starts_with($dir, '/') && !str_contains($dir, '..')) ? $dir : self::defaults()['historyDir'];
        return $s;
    }
}
```

`P/lib/bootstrap.php`:
```php
<?php
declare(strict_types=1);

foreach (['Paths', 'Sys', 'Settings', 'Vars', 'Scripts', 'Cron', 'Schedule', 'History', 'State', 'Alerts', 'Runner'] as $c) {
    if (is_file(__DIR__ . "/$c.php")) require_once __DIR__ . "/$c.php";
}
Paths::fromEnv();
```

`tests/TestCase.php`:
```php
<?php
declare(strict_types=1);

require_once __DIR__ . '/../vendor/autoload.php';
putenv('US_TEST=1');
require_once __DIR__ . '/../source/user.scripts/usr/local/emhttp/plugins/user.scripts/lib/bootstrap.php';

abstract class TestCase extends \PHPUnit\Framework\TestCase
{
    protected string $root;

    protected function setUp(): void
    {
        $this->root = sys_get_temp_dir() . '/us-test-' . bin2hex(random_bytes(4));
        mkdir($this->root . '/flash/scripts', 0777, true);
        mkdir($this->root . '/tmp', 0777, true);
        Paths::$flash = $this->root . '/flash';
        Paths::$tmp = $this->root . '/tmp';
        Paths::$varIni = $this->root . '/var.ini';
        Paths::$notify = $this->root . '/notify';
        Paths::$updateCron = $this->root . '/update_cron';
        Sys::$spawned = [];
    }

    protected function tearDown(): void { exec('rm -rf ' . escapeshellarg($this->root)); }

    protected function makeScript(string $id, string $body = "#!/bin/bash\necho hi\n", ?string $name = null): string
    {
        $dir = Paths::scripts() . '/' . $id;
        mkdir($dir, 0777, true);
        file_put_contents("$dir/script", $body);
        if ($name !== null) file_put_contents("$dir/name", $name);
        return $dir;
    }
}
```

- [ ] **Step 3: Write failing tests for Vars and Settings**

`tests/VarsTest.php`:
```php
<?php
declare(strict_types=1);

final class VarsTest extends TestCase
{
    public function testReadsLeadingHeaderLines(): void
    {
        $d = $this->makeScript('a', "#!/bin/bash\r\n#name=Backup\r\n#description=Nightly (rsync) a=b\r\n#arrayStarted=true\n\n#clearLog=false\necho x\n#late=ignored\n");
        $this->assertSame(['name' => 'Backup', 'description' => 'Nightly (rsync) a=b', 'arrayStarted' => 'true', 'clearLog' => 'false'], Vars::raw("$d/script"));
    }

    public function testMissingEmptyAndBlankPathNeverFatal(): void
    {
        $this->assertSame([], Vars::raw(''));
        $this->assertSame([], Vars::raw($this->root . '/nope'));
        $d = $this->makeScript('e', '');
        $this->assertSame([], Vars::raw("$d/script"));
    }

    public function testTypedConvertsBooleans(): void
    {
        $d = $this->makeScript('b', "#arrayStarted=true\n#noParity=no\n#argumentDefault=-v\n");
        $v = Vars::typed("$d/script");
        $this->assertTrue($v['arrayStarted']);
        $this->assertFalse($v['noParity']);
        $this->assertSame('-v', $v['argumentDefault']);
    }

    public function testCommentWithoutEqualsIsSkipped(): void
    {
        $d = $this->makeScript('c', "# just a comment\n#name=X\n");
        $this->assertSame(['name' => 'X'], Vars::raw("$d/script"));
    }
}
```

`tests/SettingsTest.php`:
```php
<?php
declare(strict_types=1);

final class SettingsTest extends TestCase
{
    public function testDefaultsAndClamping(): void
    {
        $this->assertSame(200, Settings::get()['historyCap']);
        $s = Settings::set(['historyCap' => 3, 'syncMinutes' => 7, 'historyDir' => '../etc', 'evil' => 1]);
        $this->assertSame(10, $s['historyCap']);
        $this->assertSame(60, $s['syncMinutes']);
        $this->assertSame(Paths::$flash . '/history', $s['historyDir']);
        $this->assertArrayNotHasKey('evil', Settings::get());
    }

    public function testCustomHistoryDirPersists(): void
    {
        Settings::set(['historyDir' => '/mnt/user/system/user.scripts/', 'syncMinutes' => 15]);
        $this->assertSame('/mnt/user/system/user.scripts', Settings::get()['historyDir']);
        $this->assertSame(15, Settings::get()['syncMinutes']);
    }
}
```

- [ ] **Step 4: Run to verify failure**

Run: `bin/test 8.3 --filter 'VarsTest|SettingsTest'` (the runner.sh line will fail until Task 7; run PHPUnit only with `docker run --rm -u "$(id -u):$(id -g)" -v "$PWD":/app -w /app php:8.3-cli vendor/bin/phpunit --filter 'VarsTest|SettingsTest'` after the composer install line).
Expected: FAIL, `Class "Vars" not found`.

- [ ] **Step 5: Implement Vars**

`P/lib/Vars.php`:
```php
<?php
declare(strict_types=1);

// Reads the "#key=value" lines at the top of a script (upstream format), never fatals.
final class Vars
{
    public const BOOLEANS = ['foregroundOnly', 'backgroundOnly', 'arrayStarted', 'clearLog', 'noParity', 'directPHP'];

    /** @return array<string,string> */
    public static function raw(string $file): array
    {
        if ($file === '' || !is_file($file)) return [];
        $out = [];
        foreach (explode("\n", str_replace("\r", '', (string)@file_get_contents($file))) as $line) {
            if (str_starts_with($line, '#!') || trim($line) === '' || str_starts_with($line, '<?')) continue;
            if (!str_starts_with($line, '#')) break;
            if (!str_contains($line, '=')) continue;
            [$k, $v] = explode('=', ltrim(trim($line), '#'), 2);
            $k = trim($k);
            if ($k !== '') $out[$k] = trim($v);
        }
        return $out;
    }

    /** @return array<string,mixed> */
    public static function typed(string $file): array
    {
        $v = self::raw($file);
        foreach (self::BOOLEANS as $b) {
            if (array_key_exists($b, $v)) $v[$b] = filter_var($v[$b], FILTER_VALIDATE_BOOLEAN);
        }
        return $v;
    }
}
```

- [ ] **Step 6: Run tests to verify pass**

Run: `docker run --rm -u "$(id -u):$(id -g)" -v "$PWD":/app -w /app php:8.3-cli vendor/bin/phpunit`
Expected: PASS (6 tests).

- [ ] **Step 7: Commit**

```bash
git add -A && git commit -m "chore: test harness, Paths/Sys/Settings/Vars, drop unused upstream files

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Scripts (ids, resolve, CRUD)

**Files:**
- Create: `P/lib/Scripts.php`, `tests/ScriptsTest.php`

**Interfaces:**
- Consumes: `Paths::scripts()`, `Vars::raw()`, `Sys::atomicWrite()`
- Produces: `Scripts::ids(): list<string>`, `Scripts::resolve(string $id): string` (dir, throws `InvalidArgumentException`), `Scripts::file(string $id): string`, `Scripts::name(string $id): string`, `Scripts::description(string $id): string`, `Scripts::sanitize(string $name): string`, `Scripts::create(string $name): string` (new id), `Scripts::rename(string $id, string $name): void`, `Scripts::describe(string $id, string $desc): void`, `Scripts::delete(string $id): void`, `Scripts::read(string $id): string`, `Scripts::save(string $id, string $body): void`, `Scripts::idFromPath(string $path): ?string`

- [ ] **Step 1: Write failing tests**

`tests/ScriptsTest.php`:
```php
<?php
declare(strict_types=1);

final class ScriptsTest extends TestCase
{
    public function testIdsSortedNaturallyAndOnlyDirs(): void
    {
        $this->makeScript('b2'); $this->makeScript('B10'); $this->makeScript('clear subtitles');
        touch(Paths::scripts() . '/stray-file');
        $this->assertSame(['b2', 'B10', 'clear subtitles'], Scripts::ids());
    }

    public function testResolveRejectsTraversalAndUnknown(): void
    {
        $this->makeScript('ok');
        $this->assertSame(Paths::scripts() . '/ok', Scripts::resolve('ok'));
        foreach (['', '.', '..', '../ok', 'ok/../ok', 'a/b', "ok\0", 'missing'] as $bad) {
            try { Scripts::resolve($bad); $this->fail("accepted: $bad"); } catch (InvalidArgumentException) { $this->addToAssertionCount(1); }
        }
    }

    public function testNameAndDescriptionFallbacks(): void
    {
        $this->makeScript('plain');
        $this->makeScript('hdr', "#!/bin/bash\n#name=From Header\n#description=Hdr desc\n");
        $this->makeScript('file', "#name=Ignored\n", 'From File');
        file_put_contents(Paths::scripts() . '/file/description', "File desc\r\n");
        $this->assertSame('plain', Scripts::name('plain'));
        $this->assertSame('From Header', Scripts::name('hdr'));
        $this->assertSame('From File', Scripts::name('file'));
        $this->assertSame('Hdr desc', Scripts::description('hdr'));
        $this->assertSame('File desc', Scripts::description('file'));
        $this->assertSame('', Scripts::description('plain'));
    }

    public function testCreateSanitizesAndAvoidsCollision(): void
    {
        $id = Scripts::create('My "backup"; job?');
        $this->assertSame('My backup job', $id);
        $this->assertSame("#!/bin/bash\n", Scripts::read($id));
        $id2 = Scripts::create('My backup job');
        $this->assertNotSame($id, $id2);
        $this->assertStringStartsWith('My backup job', $id2);
    }

    public function testRenameDescribeSaveDelete(): void
    {
        $this->makeScript('x');
        Scripts::rename('x', 'New <Name>');
        $this->assertSame('New Name', Scripts::name('x'));
        Scripts::describe('x', "  hello  ");
        $this->assertSame('hello', Scripts::description('x'));
        Scripts::save('x', "#!/bin/bash\r\necho 1\r\n");
        $this->assertSame("#!/bin/bash\necho 1\n", Scripts::read('x'));
        Scripts::delete('x');
        $this->assertSame([], Scripts::ids());
    }

    public function testIdFromPath(): void
    {
        $this->makeScript('clear subtitles');
        $this->assertSame('clear subtitles', Scripts::idFromPath(Paths::scripts() . '/clear subtitles/script'));
        $this->assertSame('clear subtitles', Scripts::idFromPath(Paths::$tmp . '/tmpScripts/clear subtitles/script'));
        $this->assertNull(Scripts::idFromPath(''));
        $this->assertNull(Scripts::idFromPath('/etc/passwd'));
    }
}
```

- [ ] **Step 2: Run to verify failure**

Run: `docker run --rm -u "$(id -u):$(id -g)" -v "$PWD":/app -w /app php:8.3-cli vendor/bin/phpunit --filter ScriptsTest`
Expected: FAIL, `Class "Scripts" not found`.

- [ ] **Step 3: Implement**

`P/lib/Scripts.php`:
```php
<?php
declare(strict_types=1);

// A script is a directory under Paths::scripts(); its id is the directory name.
final class Scripts
{
    // upstream's forbidden characters for names
    private const BAD = ['\\', '/', '*', '?', '.', "'", ';', ':', ',', '|', '+', '=', '<', '>', '[', ']', '"', '!', '(', ')', '&'];

    /** @return list<string> */
    public static function ids(): array
    {
        $root = Paths::scripts();
        $ids = array_values(array_filter(@scandir($root) ?: [], fn($d) => $d !== '.' && $d !== '..' && is_dir("$root/$d")));
        natcasesort($ids);
        return array_values($ids);
    }

    public static function resolve(string $id): string
    {
        if ($id === '' || str_contains($id, '/') || str_contains($id, "\0") || !in_array($id, self::ids(), true)) {
            throw new InvalidArgumentException('unknown script');
        }
        return Paths::scripts() . '/' . $id;
    }

    public static function file(string $id): string { return self::resolve($id) . '/script'; }

    public static function name(string $id): string
    {
        $dir = self::resolve($id);
        $n = is_file("$dir/name") ? trim((string)file_get_contents("$dir/name")) : '';
        return $n !== '' ? $n : (Vars::raw("$dir/script")['name'] ?? $id) ?: $id;
    }

    public static function description(string $id): string
    {
        $dir = self::resolve($id);
        $d = is_file("$dir/description") ? trim(str_replace("\r", '', (string)file_get_contents("$dir/description"))) : '';
        return $d !== '' ? $d : (Vars::raw("$dir/script")['description'] ?? '');
    }

    public static function sanitize(string $name): string
    {
        $n = str_replace(self::BAD, '', html_entity_decode($name, ENT_QUOTES));
        return trim((string)preg_replace('/ {2,}/', ' ', $n));
    }

    public static function create(string $name): string
    {
        $id = self::sanitize($name);
        if ($id === '') throw new InvalidArgumentException('empty name');
        $base = $id;
        while (is_dir(Paths::scripts() . '/' . $id)) $id = $base . '-' . mt_rand(1000, 9999);
        mkdir(Paths::scripts() . '/' . $id, 0777, true);
        file_put_contents(Paths::scripts() . "/$id/script", "#!/bin/bash\n");
        file_put_contents(Paths::scripts() . "/$id/name", $base);
        return $id;
    }

    public static function rename(string $id, string $name): void
    {
        $n = self::sanitize($name);
        if ($n === '') throw new InvalidArgumentException('empty name');
        file_put_contents(self::resolve($id) . '/name', $n);
    }

    public static function describe(string $id, string $desc): void
    {
        file_put_contents(self::resolve($id) . '/description', trim(str_replace("\r", '', $desc)));
    }

    public static function delete(string $id): void
    {
        exec('rm -rf ' . escapeshellarg(self::resolve($id)));
    }

    public static function read(string $id): string
    {
        $f = self::file($id);
        return is_file($f) ? str_replace("\r", '', (string)file_get_contents($f)) : '';
    }

    public static function save(string $id, string $body): void
    {
        file_put_contents(self::file($id), str_replace("\r", '', $body));
    }

    /** Maps an upstream-style path (flash or tmpScripts copy) to an existing id. */
    public static function idFromPath(string $path): ?string
    {
        if ($path === '' || basename($path) !== 'script') return null;
        $id = basename(dirname($path));
        return in_array($id, self::ids(), true) ? $id : null;
    }
}
```

- [ ] **Step 4: Run tests**

Run: `docker run --rm -u "$(id -u):$(id -g)" -v "$PWD":/app -w /app php:8.3-cli vendor/bin/phpunit --filter ScriptsTest`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat: Scripts library with path validation

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Cron (validate, next run, describe)

**Files:**
- Create: `P/lib/Cron.php`, `tests/CronTest.php`

**Interfaces:**
- Produces: `Cron::valid(string $expr): bool`, `Cron::parse(string $expr): array` (5 lists of ints; throws `InvalidArgumentException`), `Cron::next(string $expr, int $from): ?int` (first matching minute strictly after `$from`, null if none in 366 days), `Cron::describe(string $expr): string`

- [ ] **Step 1: Write failing tests**

`tests/CronTest.php`:
```php
<?php
declare(strict_types=1);

final class CronTest extends TestCase
{
    public static function validProvider(): array
    {
        return array_map(fn($e) => [$e], ['* * * * *', '*/5 * * * *', '0 3 * * 1', '0 */12 * * *', '15,45 8-18 * * mon-fri',
            '0 0 1 JAN,jul *', '0 4 * * 7', '30 2 1-7 * sun', '0 0 * * 0', '5-55/10 * * * *']);
    }

    public static function invalidProvider(): array
    {
        return array_map(fn($e) => [$e], ['', '* * * *', '* * * * * *', '60 * * * *', '* 24 * * *', '* * 0 * *', '* * * 13 *',
            '* * * * 8', '*/0 * * * *', 'a * * * *', '* * * * mon-', '1-  * * * *', '@daily', '0 3 * * 1 /bin/x']);
    }

    /** @dataProvider validProvider */
    public function testValid(string $e): void { $this->assertTrue(Cron::valid($e), $e); }

    /** @dataProvider invalidProvider */
    public function testInvalid(string $e): void { $this->assertFalse(Cron::valid($e), $e); }

    public function testNext(): void
    {
        date_default_timezone_set('America/Sao_Paulo');
        $t = strtotime('2026-10-10 14:02:30');
        $this->assertSame(strtotime('2026-10-10 14:03:00'), Cron::next('* * * * *', $t));
        $this->assertSame(strtotime('2026-10-10 14:05:00'), Cron::next('*/5 * * * *', $t));
        $this->assertSame(strtotime('2026-10-12 03:00:00'), Cron::next('0 3 * * 1', $t)); // Monday
        $this->assertSame(strtotime('2026-11-01 00:00:00'), Cron::next('0 0 1 * *', $t));
        $this->assertSame(strtotime('2027-02-28 12:00:00'), Cron::next('0 12 28 feb *', $t));
        $this->assertNull(Cron::next('0 0 31 2 *', $t));
        // dom and dow both restricted = OR (Vixie cron)
        $this->assertSame(strtotime('2026-10-11 00:00:00'), Cron::next('0 0 13 * sun', $t));
    }

    public function testDescribe(): void
    {
        $this->assertSame('every minute', Cron::describe('* * * * *'));
        $this->assertSame('every 5 minutes', Cron::describe('*/5 * * * *'));
        $this->assertSame('hourly at :15', Cron::describe('15 * * * *'));
        $this->assertSame('every 12 hours at :00', Cron::describe('0 */12 * * *'));
        $this->assertSame('daily at 03:00', Cron::describe('0 3 * * *'));
        $this->assertSame('Mon at 03:00', Cron::describe('0 3 * * 1'));
        $this->assertSame('Mon, Wed, Fri at 08:30', Cron::describe('30 8 * * mon,wed,fri'));
        $this->assertSame('monthly on day 1 at 00:00', Cron::describe('0 0 1 * *'));
        $this->assertSame('15,45 8-18 * * 1-5', Cron::describe('15,45 8-18 * * 1-5'));
    }
}
```

- [ ] **Step 2: Run to verify failure**

Run: `docker run --rm -u "$(id -u):$(id -g)" -v "$PWD":/app -w /app php:8.3-cli vendor/bin/phpunit --filter CronTest`
Expected: FAIL, `Class "Cron" not found`.

- [ ] **Step 3: Implement**

`P/lib/Cron.php`:
```php
<?php
declare(strict_types=1);

// Standard 5-field cron: minute hour day-of-month month day-of-week.
final class Cron
{
    private const RANGES = [[0, 59], [0, 23], [1, 31], [1, 12], [0, 7]];
    private const MONTHS = ['jan' => 1, 'feb' => 2, 'mar' => 3, 'apr' => 4, 'may' => 5, 'jun' => 6,
        'jul' => 7, 'aug' => 8, 'sep' => 9, 'oct' => 10, 'nov' => 11, 'dec' => 12];
    private const DAYS = ['sun' => 0, 'mon' => 1, 'tue' => 2, 'wed' => 3, 'thu' => 4, 'fri' => 5, 'sat' => 6];
    private const DAY_NAMES = ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat'];

    public static function valid(string $expr): bool
    {
        try { self::parse($expr); return true; } catch (InvalidArgumentException) { return false; }
    }

    /** @return array{0:list<int>,1:list<int>,2:list<int>,3:list<int>,4:list<int>,5:bool,6:bool} sets + domStar + dowStar */
    public static function parse(string $expr): array
    {
        $f = preg_split('/\s+/', trim($expr));
        if (!$f || count($f) !== 5) throw new InvalidArgumentException('cron needs 5 fields');
        $sets = [];
        foreach ($f as $i => $field) $sets[$i] = self::field(strtolower($field), $i);
        $sets[4] = array_values(array_unique(array_map(fn($d) => $d === 7 ? 0 : $d, $sets[4])));
        $sets[5] = $f[2] === '*';
        $sets[6] = $f[4] === '*';
        return $sets;
    }

    /** @return list<int> */
    private static function field(string $field, int $i): array
    {
        [$lo, $hi] = self::RANGES[$i];
        $names = $i === 3 ? self::MONTHS : ($i === 4 ? self::DAYS : []);
        $out = [];
        foreach (explode(',', $field) as $part) {
            if (!preg_match('~^(\*|[a-z0-9]+(?:-[a-z0-9]+)?)(?:/(\d+))?$~', $part, $m)) throw new InvalidArgumentException("bad field $field");
            $step = isset($m[2]) ? (int)$m[2] : 1;
            if ($step < 1) throw new InvalidArgumentException('bad step');
            if ($m[1] === '*') { $a = $lo; $b = $hi; }
            else {
                $ends = explode('-', $m[1]);
                $a = self::value($ends[0], $names, $lo, $hi);
                $b = isset($ends[1]) ? self::value($ends[1], $names, $lo, $hi) : (isset($m[2]) ? $hi : $a);
                if ($a > $b) throw new InvalidArgumentException('bad range');
            }
            for ($v = $a; $v <= $b; $v += $step) $out[] = $v;
        }
        sort($out);
        return array_values(array_unique($out));
    }

    private static function value(string $s, array $names, int $lo, int $hi): int
    {
        if (isset($names[$s])) return $names[$s];
        if (!ctype_digit($s)) throw new InvalidArgumentException("bad value $s");
        $v = (int)$s;
        if ($v < $lo || $v > $hi) throw new InvalidArgumentException("out of range $s");
        return $v;
    }

    public static function next(string $expr, int $from): ?int
    {
        [$min, $hour, $dom, $mon, $dow, $domStar, $dowStar] = self::parse($expr);
        $t = (new DateTimeImmutable('@' . ($from - $from % 60 + 60)))->setTimezone(new DateTimeZone(date_default_timezone_get()));
        $end = $from + 366 * 86400;
        while ($t->getTimestamp() <= $end) {
            if (!in_array((int)$t->format('n'), $mon, true)) { $t = $t->modify('first day of next month')->setTime(0, 0); continue; }
            $dm = in_array((int)$t->format('j'), $dom, true);
            $dw = in_array((int)$t->format('w'), $dow, true);
            $dayOk = ($domStar || $dowStar) ? ($dm && $dw) : ($dm || $dw);
            if (!$dayOk) { $t = $t->modify('+1 day')->setTime(0, 0); continue; }
            if (!in_array((int)$t->format('G'), $hour, true)) { $t = $t->setTime((int)$t->format('G'), 0)->modify('+1 hour'); continue; }
            if (!in_array((int)$t->format('i'), $min, true)) { $t = $t->modify('+1 minute'); continue; }
            return $t->getTimestamp();
        }
        return null;
    }

    public static function describe(string $expr): string
    {
        $f = preg_split('/\s+/', strtolower(trim($expr)));
        if (!self::valid($expr)) return $expr;
        [$mi, $h, $dom, $mon, $dow] = $f;
        $num = fn($s) => ctype_digit($s);
        $hm = fn() => sprintf('%02d:%02d', (int)$h, (int)$mi);
        if ($mon !== '*') return $expr;
        if ($mi === '*' && $h === '*' && $dom === '*' && $dow === '*') return 'every minute';
        if (preg_match('~^\*/(\d+)$~', $mi, $m) && $h === '*' && $dom === '*' && $dow === '*') return "every {$m[1]} minutes";
        if ($num($mi) && $h === '*' && $dom === '*' && $dow === '*') return sprintf('hourly at :%02d', (int)$mi);
        if ($num($mi) && preg_match('~^\*/(\d+)$~', $h, $m) && $dom === '*' && $dow === '*') return sprintf('every %d hours at :%02d', (int)$m[1], (int)$mi);
        if ($num($mi) && $num($h) && $dom === '*' && $dow === '*') return 'daily at ' . $hm();
        if ($num($mi) && $num($h) && $dom === '*' && preg_match('~^[a-z0-9]+(,[a-z0-9]+)*$~', $dow)) {
            $days = array_map(fn($d) => self::DAY_NAMES[(self::DAYS[$d] ?? (int)$d) % 7], explode(',', $dow));
            return implode(', ', $days) . ' at ' . $hm();
        }
        if ($num($mi) && $num($h) && $num($dom) && $dow === '*') return "monthly on day $dom at " . $hm();
        return $expr;
    }
}
```

- [ ] **Step 4: Run tests**

Run: `docker run --rm -u "$(id -u):$(id -g)" -v "$PWD":/app -w /app php:8.3-cli vendor/bin/phpunit --filter CronTest`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat: cron validation, next-run and human description (fixes #12, #13)

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Schedule (schedule.json, options, cron migration + generation)

**Files:**
- Create: `P/lib/Schedule.php`, `tests/ScheduleTest.php`, `tests/fixtures/tower-schedule.json`, `tests/fixtures/tower-customSchedule.cron`

**Interfaces:**
- Consumes: `Scripts::ids()`, `Scripts::resolve()`, `Cron::valid()`, `Sys::atomicWrite()`, `Sys::updateCron()`, `Settings::get()`
- Produces: `Schedule::FREQUENCIES`, `Schedule::DEFAULTS`, `Schedule::all(): array<string id, array entry>` (normalized, every script present), `Schedule::get(string $id): array`, `Schedule::set(string $id, array $fields): array` (validates, saves, no cron write), `Schedule::apply(array $changes): void` (map id => fields; saves all then `writeCron()`), `Schedule::byFrequency(string $f): list<string>`, `Schedule::migrateCron(): int`, `Schedule::writeCron(): void`, `Schedule::cronLine(string $expr, string $id): string`. Entry keys: `script, frequency, id, custom, notify, failAfter, remindHours, overlap, timeout, hangAfter`.

- [ ] **Step 1: Capture Tower fixtures**

```bash
ssh tower 'cat /boot/config/plugins/user.scripts/schedule.json' > tests/fixtures/tower-schedule.json
ssh tower 'cat /boot/config/plugins/user.scripts/customSchedule.cron' > tests/fixtures/tower-customSchedule.cron
printf '* * * * * * /usr/local/emhttp/plugins/user.scripts/startCustom.php /boot/config/plugins/user.scripts/scripts/broken19/script > /dev/null 2>&1\n' >> tests/fixtures/tower-customSchedule.cron
```

- [ ] **Step 2: Write failing tests**

`tests/ScheduleTest.php`:
```php
<?php
declare(strict_types=1);

final class ScheduleTest extends TestCase
{
    private function towerFixture(): void
    {
        $json = file_get_contents(__DIR__ . '/fixtures/tower-schedule.json');
        $data = json_decode($json, true);
        foreach (array_keys($data) as $path) $this->makeScript(basename(dirname($path)));
        foreach (['web-sites-status', 'cf-dns-sync', 'plex_tmp_watch', 'unifi-iptables', 'pattrmm', 'Check_paid_plex', 'BKP_NEW', 'broken19'] as $id) {
            if (!is_dir(Paths::scripts() . "/$id")) $this->makeScript($id);
        }
        file_put_contents(Paths::scheduleFile(), str_replace('/boot/config/plugins/user.scripts', Paths::$flash, $json));
        file_put_contents(Paths::cronFile(), str_replace('/boot/config/plugins/user.scripts', Paths::$flash, file_get_contents(__DIR__ . '/fixtures/tower-customSchedule.cron')));
    }

    public function testAllNormalizesWithDefaults(): void
    {
        $this->makeScript('a');
        $e = Schedule::all()['a'];
        $this->assertSame('disabled', $e['frequency']);
        $this->assertSame('off', $e['notify']);
        $this->assertSame('skip', $e['overlap']);
        $this->assertSame(Paths::scripts() . '/a/script', $e['script']);
        $this->assertSame('schedulea', $e['id']);
    }

    public function testSetValidates(): void
    {
        $this->makeScript('a');
        Schedule::set('a', ['frequency' => 'custom', 'custom' => '*/5 * * * *', 'notify' => 'failure', 'failAfter' => 3, 'overlap' => 'queue', 'timeout' => 600]);
        $saved = json_decode(file_get_contents(Paths::scheduleFile()), true)[Paths::scripts() . '/a/script'];
        $this->assertSame('custom', $saved['frequency']);
        $this->assertSame(3, $saved['failAfter']);
        foreach ([['frequency' => 'yearly'], ['frequency' => 'custom', 'custom' => '* * *'], ['notify' => 'loud'], ['overlap' => 'x'], ['failAfter' => 0], ['timeout' => -1], ['remindHours' => 9999]] as $bad) {
            try { Schedule::set('a', $bad); $this->fail(json_encode($bad)); } catch (InvalidArgumentException) { $this->addToAssertionCount(1); }
        }
    }

    public function testMigrationImportsMissingAndDropsInvalid(): void
    {
        $this->towerFixture();
        $before = Schedule::all();
        $imported = Schedule::migrateCron();
        $all = Schedule::all();
        $this->assertSame('* * * * *', $all['web-sites-status']['custom']);
        $this->assertSame('custom', $all['cf-dns-sync']['frequency']);
        $this->assertSame('*/5 * * * *', $all['cf-dns-sync']['custom']);
        $this->assertSame('disabled', $all['broken19']['frequency'], 'malformed #19 line must not import');
        $this->assertSame('boot', $all[array_key_first(array_filter($before, fn($e) => $e['frequency'] === 'boot'))]['frequency']);
        $this->assertGreaterThanOrEqual(0, $imported);
        $this->assertSame(0, Schedule::migrateCron(), 'second run is a no-op');
    }

    public function testWriteCronOneLinePerCustomEntryWithOwnExpr(): void
    {
        $this->towerFixture();
        Schedule::migrateCron();
        Schedule::writeCron();
        $cron = file_get_contents(Paths::cronFile());
        $lines = array_values(array_filter(explode("\n", $cron), fn($l) => $l !== '' && $l[0] !== '#'));
        $custom = array_filter(Schedule::all(), fn($e) => $e['frequency'] === 'custom' && $e['custom'] !== '');
        $this->assertCount(count($custom) + 1, $lines, 'custom lines + history sync line');
        $this->assertStringContainsString("*/5 * * * * " . Paths::$plugin . "/run.php --script 'cf-dns-sync' --trigger custom", $cron);
        $this->assertStringContainsString('run.php --sync-history', $cron);
        foreach ($lines as $l) $this->assertTrue(Cron::valid(implode(' ', array_slice(preg_split('/\s+/', $l), 0, 5))), $l);
    }

    public function testDisablingIsNotUndoneByOldCronLine(): void
    {
        $this->towerFixture();
        Schedule::writeCron();
        Schedule::apply(['cf-dns-sync' => ['frequency' => 'disabled']]);
        $this->assertSame('disabled', Schedule::get('cf-dns-sync')['frequency']);
        $this->assertStringNotContainsString("'cf-dns-sync'", file_get_contents(Paths::cronFile()));
        Schedule::writeCron();
        $this->assertSame('disabled', Schedule::get('cf-dns-sync')['frequency']);
    }

    public function testCronLineEscapesQuotesAndPercent(): void
    {
        $line = Schedule::cronLine('0 3 * * *', "it's 100% done");
        $this->assertSame("0 3 * * * " . Paths::$plugin . "/run.php --script 'it'\\''s 100\\% done' --trigger custom > /dev/null 2>&1", $line);
    }

    public function testSyncLineFollowsSettings(): void
    {
        $this->makeScript('a');
        Settings::set(['syncMinutes' => 15]);
        Schedule::writeCron();
        $this->assertStringContainsString('*/15 * * * * ' . Paths::$plugin . '/run.php --sync-history', file_get_contents(Paths::cronFile()));
    }

    public function testByFrequencySkipsDeletedScripts(): void
    {
        $this->makeScript('a'); $this->makeScript('b');
        Schedule::apply(['a' => ['frequency' => 'daily'], 'b' => ['frequency' => 'daily']]);
        Scripts::delete('b');
        $this->assertSame(['a'], Schedule::byFrequency('daily'));
    }
}
```

- [ ] **Step 3: Run to verify failure**

Run: `docker run --rm -u "$(id -u):$(id -g)" -v "$PWD":/app -w /app php:8.3-cli vendor/bin/phpunit --filter ScheduleTest`
Expected: FAIL, `Class "Schedule" not found`.

- [ ] **Step 4: Implement**

`P/lib/Schedule.php`:
```php
<?php
declare(strict_types=1);

// schedule.json (upstream format, keyed by script path) plus our additive per-script options.
final class Schedule
{
    public const FREQUENCIES = ['disabled', 'hourly', 'daily', 'weekly', 'monthly', 'custom', 'start', 'boot', 'stop'];
    public const DEFAULTS = ['notify' => 'off', 'failAfter' => 1, 'remindHours' => 24, 'overlap' => 'skip', 'timeout' => 0, 'hangAfter' => 3];
    private const INTS = ['failAfter' => [1, 1000], 'remindHours' => [0, 720], 'timeout' => [0, 604800], 'hangAfter' => [1, 1000]];

    private static function path(string $id): string { return Paths::scripts() . "/$id/script"; }

    private static function raw(): array
    {
        $f = Paths::scheduleFile();
        $d = is_file($f) ? json_decode((string)file_get_contents($f), true) : null;
        return is_array($d) ? $d : [];
    }

    /** @return array<string,array> keyed by id, every existing script present */
    public static function all(): array
    {
        $raw = self::raw();
        $out = [];
        foreach (Scripts::ids() as $id) {
            $e = $raw[self::path($id)] ?? [];
            $out[$id] = array_merge(self::DEFAULTS, [
                'script' => self::path($id), 'frequency' => 'disabled', 'id' => "schedule$id", 'custom' => '',
            ], array_intersect_key($e, array_flip(['frequency', 'custom', ...array_keys(self::DEFAULTS)])));
            if (!in_array($out[$id]['frequency'], self::FREQUENCIES, true)) $out[$id]['frequency'] = 'disabled';
        }
        return $out;
    }

    public static function get(string $id): array
    {
        Scripts::resolve($id);
        return self::all()[$id];
    }

    private static function validate(array $e): array
    {
        if (!in_array($e['frequency'], self::FREQUENCIES, true)) throw new InvalidArgumentException('bad frequency');
        $e['custom'] = trim((string)$e['custom']);
        if ($e['frequency'] === 'custom' && !Cron::valid($e['custom'])) throw new InvalidArgumentException('invalid cron expression');
        if (!in_array($e['notify'], ['off', 'failure', 'always'], true)) throw new InvalidArgumentException('bad notify');
        if (!in_array($e['overlap'], ['skip', 'queue', 'parallel'], true)) throw new InvalidArgumentException('bad overlap');
        foreach (self::INTS as $k => [$lo, $hi]) {
            if (filter_var($e[$k], FILTER_VALIDATE_INT) === false || (int)$e[$k] < $lo || (int)$e[$k] > $hi) throw new InvalidArgumentException("$k must be $lo-$hi");
            $e[$k] = (int)$e[$k];
        }
        return $e;
    }

    public static function set(string $id, array $fields): array
    {
        Scripts::resolve($id);
        $all = self::all();
        $all[$id] = self::validate(array_merge($all[$id], array_intersect_key($fields, array_flip(['frequency', 'custom', ...array_keys(self::DEFAULTS)]))));
        self::save($all);
        return $all[$id];
    }

    /** @param array<string,array> $changes */
    public static function apply(array $changes): void
    {
        $all = self::all();
        foreach ($changes as $id => $fields) {
            Scripts::resolve((string)$id);
            $all[$id] = self::validate(array_merge($all[$id], array_intersect_key((array)$fields, array_flip(['frequency', 'custom', ...array_keys(self::DEFAULTS)]))));
        }
        self::save($all);
        self::writeCron();
    }

    private static function save(array $byId): void
    {
        $out = [];
        foreach ($byId as $e) $out[$e['script']] = $e;
        $json = json_encode($out, JSON_PRETTY_PRINT | JSON_UNESCAPED_SLASHES);
        Sys::atomicWrite(Paths::scheduleFile(), $json);
        Sys::atomicWrite(Paths::$tmp . '/schedule.json', $json); // upstream reads this copy; keeps rollback consistent
    }

    /** @return list<string> */
    public static function byFrequency(string $f): array
    {
        return array_keys(array_filter(self::all(), fn($e) => $e['frequency'] === $f));
    }

    /**
     * Imports hand-added upstream-format lines (startCustom.php) that schedule.json does not know.
     * Lines in our own run.php format are generated from schedule.json, so they are never imported:
     * otherwise disabling a schedule would be undone by its old cron line on the next write.
     */
    public static function migrateCron(): int
    {
        $f = Paths::cronFile();
        if (!is_file($f)) return 0;
        $all = self::all();
        $n = 0;
        foreach (explode("\n", (string)file_get_contents($f)) as $line) {
            $line = trim($line);
            if ($line === '' || $line[0] === '#') continue;
            if (!preg_match('~^((?:\S+\s+){4}\S+)\s+\S*startCustom\.php\s+(.*?)/script(?:\s|$)~', $line, $m)) continue;
            $id = basename($m[2]);
            $expr = preg_replace('/\s+/', ' ', $m[1]);
            if (!isset($all[$id]) || !Cron::valid($expr)) continue;
            if ($all[$id]['frequency'] === 'custom' && $all[$id]['custom'] !== '') continue;
            $all[$id]['frequency'] = 'custom';
            $all[$id]['custom'] = $expr;
            $n++;
        }
        if ($n) self::save($all);
        return $n;
    }

    public static function cronLine(string $expr, string $id): string
    {
        $arg = str_replace('%', '\\%', escapeshellarg($id));
        return "$expr " . Paths::$plugin . "/run.php --script $arg --trigger custom > /dev/null 2>&1";
    }

    public static function writeCron(): void
    {
        self::migrateCron();
        $mins = Settings::get()['syncMinutes'];
        $sync = $mins === 60 ? '17 * * * *' : "*/$mins * * * *";
        $out = "# Generated cron schedule for user.scripts\n";
        foreach (self::all() as $id => $e) {
            if ($e['frequency'] === 'custom' && Cron::valid($e['custom'])) $out .= self::cronLine($e['custom'], $id) . "\n";
        }
        $out .= "$sync " . Paths::$plugin . "/run.php --sync-history > /dev/null 2>&1\n";
        Sys::atomicWrite(Paths::cronFile(), $out);
        Sys::updateCron();
    }
}
```

- [ ] **Step 5: Run tests**

Run: `docker run --rm -u "$(id -u):$(id -g)" -v "$PWD":/app -w /app php:8.3-cli vendor/bin/phpunit --filter ScheduleTest`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add -A && git commit -m "feat: Schedule with options, cron migration and per-entry cron lines (fixes #17, #19, #20)

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: History (RAM, collapse, cap, sync, seed)

**Files:**
- Create: `P/lib/History.php`, `tests/HistoryTest.php`

**Interfaces:**
- Consumes: `Paths::ramHistory()`, `Settings::get()`, `Sys::atomicWrite()`
- Produces: `History::record(string $id, array $run): bool` (run keys `start,end,exit,trigger,status`; returns true when status/exit changed vs previous entry), `History::read(string $id, int $limit = 50): list<array>` newest first, `History::last(string $id): ?array`, `History::sync(?string $id = null): int`, `History::seed(): void`, `History::diskDir(): ?string` (null when unavailable). Collapsed entry adds `repeat` (int ≥2), `first` (start of first run), `dmin`, `dmax`, `davg` (seconds).

- [ ] **Step 1: Write failing tests**

`tests/HistoryTest.php`:
```php
<?php
declare(strict_types=1);

final class HistoryTest extends TestCase
{
    private function run1(int $start, int $exit = 0, string $status = 'ok', int $dur = 2): array
    {
        return ['start' => $start, 'end' => $start + $dur, 'exit' => $exit, 'trigger' => 'custom', 'status' => $status];
    }

    public function testCollapsesRepeatsAndTracksDurations(): void
    {
        $this->makeScript('a');
        $this->assertTrue(History::record('a', $this->run1(100, 0, 'ok', 2)));
        $this->assertFalse(History::record('a', $this->run1(160, 0, 'ok', 4)));
        $this->assertFalse(History::record('a', $this->run1(220, 0, 'ok', 6)));
        $h = History::read('a');
        $this->assertCount(1, $h);
        $this->assertSame(3, $h[0]['repeat']);
        $this->assertSame(100, $h[0]['first']);
        $this->assertSame(220, $h[0]['start']);
        $this->assertSame([2, 6, 4], [$h[0]['dmin'], $h[0]['dmax'], $h[0]['davg']]);
        $this->assertTrue(History::record('a', $this->run1(280, 1, 'failed')));
        $this->assertTrue(History::record('a', $this->run1(340, 2, 'failed')), 'new exit code is a change');
        $this->assertSame(['failed', 'failed', 'ok'], array_column(History::read('a'), 'status'));
    }

    public function testNoFlashWriteOnRecord(): void
    {
        $this->makeScript('a');
        History::record('a', $this->run1(100));
        $this->assertFileDoesNotExist(Paths::$flash . '/history/a.jsonl');
        $this->assertFileExists(Paths::ramHistory() . '/a.jsonl');
    }

    public function testCap(): void
    {
        $this->makeScript('a');
        Settings::set(['historyCap' => 10]);
        for ($i = 0; $i < 80; $i++) History::record('a', $this->run1(100 + $i * 60, $i % 2, $i % 2 ? 'failed' : 'ok'));
        $n = count(file(Paths::ramHistory() . '/a.jsonl'));
        $this->assertLessThanOrEqual(60, $n);
        $this->assertCount(10, History::read('a', 1000));
    }

    public function testSyncOnlyWhenChangedAndSeed(): void
    {
        $this->makeScript('a');
        History::record('a', $this->run1(100));
        $this->assertSame(1, History::sync());
        $this->assertSame(0, History::sync(), 'unchanged content is not rewritten');
        History::record('a', $this->run1(160));
        $this->assertSame(1, History::sync('a'));
        exec('rm -rf ' . escapeshellarg(Paths::ramHistory()));
        History::seed();
        $this->assertSame(2, History::read('a')[0]['repeat']);
        $this->assertSame(0, History::sync(), 'seeded content counts as synced');
    }

    public function testUnavailableDiskDirKeepsWorkingInRam(): void
    {
        $this->makeScript('a');
        Settings::set(['historyDir' => $this->root . '/mnt/user/system/us']);
        History::record('a', $this->run1(100));
        $this->assertNull(History::diskDir());
        $this->assertSame(0, History::sync());
        mkdir($this->root . '/mnt/user', 0777, true);
        $this->assertSame(1, History::sync());
        $this->assertFileExists($this->root . '/mnt/user/system/us/a.jsonl');
    }

    public function testConcurrentRecordsDoNotLoseLines(): void
    {
        $this->makeScript('a');
        $boot = __DIR__ . '/../source/user.scripts/usr/local/emhttp/plugins/user.scripts/lib/bootstrap.php';
        $env = 'US_TEST=1 US_FLASH=' . escapeshellarg(Paths::$flash) . ' US_TMP=' . escapeshellarg(Paths::$tmp);
        $code = 'require ' . var_export($boot, true) . '; for($i=0;$i<40;$i++) History::record("a", ["start"=>$i,"end"=>$i,"exit"=>$i%2,"trigger"=>"x","status"=>$i%2?"failed":"ok"]);';
        exec("$env php -r " . escapeshellarg($code) . " & $env php -r " . escapeshellarg($code) . " & wait");
        $lines = file(Paths::ramHistory() . '/a.jsonl', FILE_IGNORE_NEW_LINES);
        $this->assertCount(80, $lines);
        foreach ($lines as $l) $this->assertIsArray(json_decode($l, true));
    }
}
```

- [ ] **Step 2: Run to verify failure**

Run: `docker run --rm -u "$(id -u):$(id -g)" -v "$PWD":/app -w /app php:8.3-cli vendor/bin/phpunit --filter HistoryTest`
Expected: FAIL, `Class "History" not found`.

- [ ] **Step 3: Implement**

`P/lib/History.php`:
```php
<?php
declare(strict_types=1);

// Run history: live in tmpfs, snapshotted to historyDir by sync() (see spec "Flash wear").
final class History
{
    private static function ram(string $id): string { return Paths::ramHistory() . '/' . rawurlencode($id) . '.jsonl'; }

    /** historyDir if it can be written now, else null (array stopped, pool path missing). */
    public static function diskDir(): ?string
    {
        $dir = Settings::get()['historyDir'];
        if (is_dir($dir)) return $dir;
        // create at most two levels: /mnt/user/system/x needs /mnt/user, which only exists while the
        // array is started, so history never gets written into an empty mount point
        if (!is_dir(dirname(dirname($dir)))) return null;
        return @mkdir($dir, 0777, true) || is_dir($dir) ? $dir : null;
    }

    /** @return bool true when status or exit changed vs the previous entry */
    public static function record(string $id, array $run): bool
    {
        @mkdir(Paths::ramHistory(), 0777, true);
        $file = self::ram($id);
        $lock = fopen($file . '.lock', 'c');
        flock($lock, LOCK_EX);
        try {
            $lines = is_file($file) ? file($file, FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES) : [];
            $prev = $lines ? json_decode((string)end($lines), true) : null;
            $dur = max(0, (int)$run['end'] - (int)$run['start']);
            $run = ['start' => (int)$run['start'], 'end' => (int)$run['end'], 'exit' => (int)$run['exit'], 'trigger' => (string)$run['trigger'], 'status' => (string)$run['status']];
            $changed = !$prev || $prev['status'] !== $run['status'] || (int)$prev['exit'] !== $run['exit'];
            if ($changed) {
                $lines[] = json_encode($run + ['dur' => $dur]);
            } else {
                $n = (int)($prev['repeat'] ?? 1);
                $pd = (int)($prev['dur'] ?? 0);
                $merged = $run + [
                    'dur' => $dur,
                    'repeat' => $n + 1,
                    'first' => (int)($prev['first'] ?? $prev['start']),
                    'dmin' => min((int)($prev['dmin'] ?? $pd), $dur),
                    'dmax' => max((int)($prev['dmax'] ?? $pd), $dur),
                    'davg' => (int)round(((int)($prev['davg'] ?? $pd) * $n + $dur) / ($n + 1)),
                ];
                $lines[count($lines) - 1] = json_encode($merged);
            }
            $cap = Settings::get()['historyCap'];
            if (count($lines) > $cap + 50) $lines = array_slice($lines, -$cap);
            Sys::atomicWrite($file, implode("\n", $lines) . "\n");
            return $changed;
        } finally {
            flock($lock, LOCK_UN);
            fclose($lock);
        }
    }

    /** @return list<array> newest first */
    public static function read(string $id, int $limit = 50): array
    {
        $file = self::ram($id);
        $lines = is_file($file) ? file($file, FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES) : [];
        $lines = array_slice($lines, -min($limit, Settings::get()['historyCap']));
        return array_values(array_filter(array_map(fn($l) => json_decode($l, true), array_reverse($lines)), 'is_array'));
    }

    public static function last(string $id): ?array { return self::read($id, 1)[0] ?? null; }

    private static function hashes(): array
    {
        $f = Paths::ramHistory() . '/.synced.json';
        return is_file($f) ? (json_decode((string)file_get_contents($f), true) ?: []) : [];
    }

    /** Copies changed RAM files to historyDir; $id limits to one script (+ state). @return int files written */
    public static function sync(?string $id = null): int
    {
        $dir = self::diskDir();
        if ($dir === null || !is_dir(Paths::ramHistory())) return 0;
        $names = $id === null
            ? array_values(array_filter(scandir(Paths::ramHistory()) ?: [], fn($f) => str_ends_with($f, '.jsonl') || $f === 'state.json'))
            : [basename(self::ram($id)), 'state.json'];
        $hashes = self::hashes();
        $n = 0;
        foreach ($names as $name) {
            $src = Paths::ramHistory() . "/$name";
            if (!is_file($src)) continue;
            $data = (string)file_get_contents($src);
            $h = md5($data);
            if (($hashes[$name] ?? '') === $h) continue;
            Sys::atomicWrite("$dir/$name", $data);
            $hashes[$name] = $h;
            $n++;
        }
        if ($n) Sys::atomicWrite(Paths::ramHistory() . '/.synced.json', json_encode($hashes));
        return $n;
    }

    /** Boot: fill RAM from historyDir for anything not already in RAM. */
    public static function seed(): void
    {
        $dir = self::diskDir();
        if ($dir === null) return;
        @mkdir(Paths::ramHistory(), 0777, true);
        $hashes = self::hashes();
        foreach (scandir($dir) ?: [] as $name) {
            if (!str_ends_with($name, '.jsonl') && $name !== 'state.json') continue;
            $dst = Paths::ramHistory() . "/$name";
            if (is_file($dst)) continue;
            $data = (string)file_get_contents("$dir/$name");
            Sys::atomicWrite($dst, $data);
            $hashes[$name] = md5($data);
        }
        Sys::atomicWrite(Paths::ramHistory() . '/.synced.json', json_encode($hashes));
    }
}
```

- [ ] **Step 4: Run tests**

Run: `docker run --rm -u "$(id -u):$(id -g)" -v "$PWD":/app -w /app php:8.3-cli vendor/bin/phpunit --filter HistoryTest`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat: run history in RAM with collapse, cap and hash-based flash sync

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: State + Alerts (repeat-failure state machine, notify)

**Files:**
- Create: `P/lib/State.php`, `P/lib/Alerts.php`, `tests/AlertsTest.php`

**Interfaces:**
- Consumes: `Paths::ramHistory()`, `Paths::$notify`, `Sys::atomicWrite()`, `Sys::log()`
- Produces: `State::update(string $id, callable $fn): array` (locked read-modify-write of one script's state; `$fn(array $s): array`), `State::get(string $id): array`, `State::all(): array`, `State::BLANK`; `Alerts::decide(array $state, array $run, array $opts, string $name, int $now): array{0:array,1:?array}` (pure; notification = `['level'=>'alert'|'warning'|'normal','subject'=>string,'detail'=>string]`), `Alerts::onRun(string $id, string $name, array $run, array $opts, string $logTail): void`, `Alerts::onSkip(string $id, string $name, array $opts, int $runningSince, int $now): void`, `Alerts::onStart(string $id): void`, `Alerts::send(array $n, string $message): void`, `Alerts::human(int $seconds): string`

- [ ] **Step 1: Write failing tests**

`tests/AlertsTest.php`:
```php
<?php
declare(strict_types=1);

final class AlertsTest extends TestCase
{
    private array $opts = ['notify' => 'failure', 'failAfter' => 1, 'remindHours' => 24, 'hangAfter' => 3];

    private function run1(int $start, int $exit, string $status): array
    {
        return ['start' => $start, 'end' => $start + 5, 'exit' => $exit, 'trigger' => 'custom', 'status' => $status];
    }

    private function play(array $events, array $opts): array
    {
        $s = State::BLANK; $sent = [];
        foreach ($events as [$t, $exit, $status]) {
            [$s, $n] = Alerts::decide($s, $this->run1($t, $exit, $status), $opts, 'Job', $t + 5);
            if ($n) $sent[] = $n['level'] . ':' . $n['subject'];
        }
        return [$s, $sent];
    }

    public function testFailAfterAbsorbsBlips(): void
    {
        [, $sent] = $this->play([[0, 1, 'failed'], [60, 0, 'ok'], [120, 1, 'failed'], [180, 1, 'failed'], [240, 1, 'failed']], ['failAfter' => 3] + $this->opts);
        $this->assertSame(['alert:Job failing: exit 1 (3 runs since 02:00)'], array_map(fn($x) => preg_replace('/since \d\d:\d\d/', 'since 02:00', $x), $sent));
    }

    public function testNoRepeatSameExitThenNewExitWarns(): void
    {
        [, $sent] = $this->play([[0, 1, 'failed'], [60, 1, 'failed'], [120, 1, 'failed'], [180, 2, 'failed']], $this->opts);
        $this->assertCount(2, $sent);
        $this->assertStringStartsWith('alert:Job failing: exit 1', $sent[0]);
        $this->assertSame('warning:Job now failing with exit 2', $sent[1]);
    }

    public function testReminderAfterRemindHours(): void
    {
        [, $sent] = $this->play([[0, 1, 'failed'], [3600, 1, 'failed'], [86400 + 10, 1, 'failed'], [86400 + 70, 1, 'failed']], $this->opts);
        $this->assertCount(2, $sent);
        $this->assertStringStartsWith('warning:Job still failing for 1d', $sent[1]);
        $this->assertStringContainsString('3 failed runs', $sent[1]);
    }

    public function testRecoveryOnlyAfterAlert(): void
    {
        [, $sent] = $this->play([[0, 1, 'failed'], [60, 0, 'ok']], ['failAfter' => 2] + $this->opts);
        $this->assertSame([], $sent);
        [$s, $sent] = $this->play([[0, 1, 'failed'], [3600, 0, 'ok']], $this->opts);
        $this->assertSame('normal:Job recovered after 1h 0m (1 failed run)', $sent[1]);
        $this->assertSame('ok', $s['health']);
        $this->assertSame(0, $s['fails']);
    }

    public function testAlwaysModeAndOffMode(): void
    {
        [, $sent] = $this->play([[0, 0, 'ok']], ['notify' => 'always'] + $this->opts);
        $this->assertSame(['normal:Job finished in 5s'], $sent);
        [$s, $sent] = $this->play([[0, 1, 'failed'], [60, 0, 'ok']], ['notify' => 'off'] + $this->opts);
        $this->assertSame([], $sent);
        $this->assertSame('ok', $s['health']);
    }

    public function testTimeoutCountsAsFailureManualAbortDoesNot(): void
    {
        [, $sent] = $this->play([[0, 124, 'timeout']], $this->opts);
        $this->assertSame(['alert:Job timed out (1 run since ' . date('H:i', 0) . ')'], $sent);
        [$s, $sent] = $this->play([[0, 137, 'aborted']], $this->opts);
        $this->assertSame([], $sent);
        $this->assertSame(0, $s['fails']);
    }

    public function testStuckAlertOncePerStuckRun(): void
    {
        $this->makeScript('a');
        file_put_contents(Paths::$notify, "#!/bin/sh\necho \"\$@\" >> " . escapeshellarg($this->root . '/sent') . "\n");
        chmod(Paths::$notify, 0755);
        for ($i = 0; $i < 5; $i++) Alerts::onSkip('a', 'Job', $this->opts, 1000, 1000 + 7800 + $i * 60);
        $sent = file($this->root . '/sent');
        $this->assertCount(1, $sent);
        $this->assertStringContainsString('Job looks stuck: running 2h 10m, 3 runs skipped', $sent[0]);
        Alerts::onStart('a');
        $this->assertSame(0, State::get('a')['skips']);
    }

    public function testOnRunSendsViaNotifyBinaryWithLogTail(): void
    {
        $this->makeScript('a');
        file_put_contents(Paths::$notify, "#!/bin/sh\nfor a in \"\$@\"; do echo \"[\$a]\"; done >> " . escapeshellarg($this->root . '/sent') . "\n");
        chmod(Paths::$notify, 0755);
        Alerts::onRun('a', 'Job', $this->run1(0, 1, 'failed'), $this->opts, "line1\nline2");
        $out = file_get_contents($this->root . '/sent');
        $this->assertStringContainsString('[User Scripts]', $out);
        $this->assertStringContainsString('[alert]', $out);
        $this->assertStringContainsString("[line1\nline2]", $out);
        $this->assertSame('failing', State::get('a')['health']);
    }

    public function testStateUpdatesAreSerialized(): void
    {
        $boot = __DIR__ . '/../source/user.scripts/usr/local/emhttp/plugins/user.scripts/lib/bootstrap.php';
        $env = 'US_TEST=1 US_FLASH=' . escapeshellarg(Paths::$flash) . ' US_TMP=' . escapeshellarg(Paths::$tmp);
        $code = 'require ' . var_export($boot, true) . '; for($i=0;$i<50;$i++) State::update("a", function($s){$s["skips"]++; return $s;});';
        exec("$env php -r " . escapeshellarg($code) . " & $env php -r " . escapeshellarg($code) . " & wait");
        $this->assertSame(100, State::get('a')['skips']);
    }
}
```

- [ ] **Step 2: Run to verify failure**

Run: `docker run --rm -u "$(id -u):$(id -g)" -v "$PWD":/app -w /app php:8.3-cli vendor/bin/phpunit --filter AlertsTest`
Expected: FAIL, `Class "State" not found`.

- [ ] **Step 3: Implement State**

`P/lib/State.php`:
```php
<?php
declare(strict_types=1);

// Per-script alert state in RAM (synced to disk with history). All writes go through update().
final class State
{
    public const BLANK = ['health' => 'ok', 'fails' => 0, 'firstFail' => 0, 'lastExit' => 0, 'lastAlert' => 0, 'alerted' => false, 'skips' => 0, 'stuckAlerted' => false];

    private static function file(): string { return Paths::ramHistory() . '/state.json'; }

    public static function all(): array
    {
        $f = self::file();
        $d = is_file($f) ? json_decode((string)file_get_contents($f), true) : null;
        return is_array($d) ? $d : [];
    }

    public static function get(string $id): array { return array_merge(self::BLANK, self::all()[$id] ?? []); }

    public static function update(string $id, callable $fn): array
    {
        @mkdir(Paths::ramHistory(), 0777, true);
        $lock = fopen(self::file() . '.lock', 'c');
        flock($lock, LOCK_EX);
        try {
            $all = self::all();
            $all[$id] = $fn(array_merge(self::BLANK, $all[$id] ?? []));
            Sys::atomicWrite(self::file(), json_encode($all));
            return $all[$id];
        } finally {
            flock($lock, LOCK_UN);
            fclose($lock);
        }
    }
}
```

- [ ] **Step 4: Implement Alerts**

`P/lib/Alerts.php`:
```php
<?php
declare(strict_types=1);

// Decides which runs deserve a notification (spec "Repeat failures") and sends them via Unraid notify.
final class Alerts
{
    public static function human(int $s): string
    {
        if ($s < 60) return "{$s}s";
        if ($s < 3600) return intdiv($s, 60) . 'm ' . ($s % 60) . 's';
        if ($s < 86400) return intdiv($s, 3600) . 'h ' . intdiv($s % 3600, 60) . 'm';
        return intdiv($s, 86400) . 'd ' . intdiv($s % 86400, 3600) . 'h';
    }

    private static function runs(int $n): string { return number_format($n) . ($n === 1 ? ' failed run' : ' failed runs'); }

    /** @return array{0:array,1:?array} new state, notification or null */
    public static function decide(array $s, array $run, array $opts, string $name, int $now): array
    {
        $s = array_merge(State::BLANK, $s);
        $on = ($opts['notify'] ?? 'off') !== 'off';
        $status = $run['status'];
        if ($status === 'aborted' || $status === 'skipped') return [$s, null];

        if ($status === 'ok') {
            $n = null;
            if ($s['alerted'] && $on) {
                $n = ['level' => 'normal', 'subject' => "$name recovered after " . self::human($now - $s['firstFail']) . ' (' . self::runs($s['fails']) . ')', 'detail' => ''];
            } elseif (($opts['notify'] ?? 'off') === 'always') {
                $n = ['level' => 'normal', 'subject' => "$name finished in " . self::human(max(0, $run['end'] - $run['start'])), 'detail' => ''];
            }
            return [array_merge(State::BLANK, ['skips' => 0]), $n];
        }

        // failed or timeout
        $s['fails']++;
        if ($s['fails'] === 1) $s['firstFail'] = (int)$run['start'];
        $s['health'] = 'failing';
        $exit = (int)$run['exit'];
        $what = $status === 'timeout' ? 'timed out' : "failing: exit $exit";
        $n = null;
        if (!$s['alerted']) {
            if ($s['fails'] >= (int)($opts['failAfter'] ?? 1) && $on) {
                $n = ['level' => 'alert', 'subject' => "$name $what ({$s['fails']} run" . ($s['fails'] === 1 ? '' : 's') . ' since ' . date('H:i', $s['firstFail']) . ')', 'detail' => ''];
                $s['alerted'] = true;
                $s['lastAlert'] = $now;
            }
        } elseif ($exit !== $s['lastExit'] && $on) {
            $n = ['level' => 'warning', 'subject' => $status === 'timeout' ? "$name now timing out" : "$name now failing with exit $exit", 'detail' => ''];
            $s['lastAlert'] = $now;
        } elseif ((int)($opts['remindHours'] ?? 24) > 0 && $now - $s['lastAlert'] >= (int)$opts['remindHours'] * 3600 && $on) {
            $n = ['level' => 'warning', 'subject' => "$name still failing for " . self::human($now - $s['firstFail']) . ', ' . self::runs($s['fails']), 'detail' => ''];
            $s['lastAlert'] = $now;
        }
        $s['lastExit'] = $exit;
        return [$s, $n];
    }

    public static function onRun(string $id, string $name, array $run, array $opts, string $logTail): void
    {
        $note = null;
        State::update($id, function (array $s) use ($run, $opts, $name, &$note) {
            [$s, $note] = self::decide($s, $run, $opts, $name, time());
            return $s;
        });
        if ($note) {
            $note['detail'] = "{$run['trigger']} run, " . self::human(max(0, $run['end'] - $run['start']));
            self::send($note, $logTail);
        }
    }

    public static function onSkip(string $id, string $name, array $opts, int $runningSince, int $now): void
    {
        $send = false;
        $s = State::update($id, function (array $s) use ($opts, &$send) {
            $s['skips']++;
            if ($s['skips'] >= (int)($opts['hangAfter'] ?? 3) && !$s['stuckAlerted'] && ($opts['notify'] ?? 'off') !== 'off') {
                $s['stuckAlerted'] = true;
                $send = true;
            }
            return $s;
        });
        if ($send) {
            self::send(['level' => 'warning', 'subject' => "$name looks stuck: running " . self::human($now - $runningSince) . ", {$s['skips']} runs skipped",
                'detail' => 'still running since ' . date('H:i', $runningSince)], '');
        }
    }

    public static function onStart(string $id): void
    {
        State::update($id, function (array $s) { $s['skips'] = 0; $s['stuckAlerted'] = false; return $s; });
    }

    public static function send(array $n, string $message): void
    {
        if (!is_executable(Paths::$notify)) { Sys::log("notify unavailable: {$n['subject']}"); return; }
        $cmd = escapeshellarg(Paths::$notify) . ' -e ' . escapeshellarg('User Scripts') . ' -s ' . escapeshellarg($n['subject'])
            . ' -d ' . escapeshellarg($n['detail']) . ' -i ' . escapeshellarg($n['level'])
            . ($message !== '' ? ' -m ' . escapeshellarg($message) : '');
        exec($cmd . ' >/dev/null 2>&1');
    }
}
```

- [ ] **Step 5: Run tests**

Run: `docker run --rm -u "$(id -u):$(id -g)" -v "$PWD":/app -w /app php:8.3-cli vendor/bin/phpunit --filter AlertsTest`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add -A && git commit -m "feat: failure alert state machine with dedupe, reminders, recovery and stuck alerts (#21, #23)

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Runner + run.php (conditions, lock/overlap, spawn, timeout, abort, logs)

**Files:**
- Create: `P/lib/Runner.php`, `P/run.php`, `tests/RunnerTest.php`, `tests/runner.sh`

**Interfaces:**
- Consumes: `Scripts::*`, `Vars::typed()`, `Schedule::get()`, `Schedule::byFrequency()`, `History::record()`, `History::sync()`, `History::seed()`, `Alerts::onRun()/onSkip()/onStart()`, `Sys::var()`, `Sys::log()`
- Produces: `Runner::run(string $id, string $trigger): string` (returns status or `'blocked'` when a condition prevented the run), `Runner::runFrequency(string $f): void`, `Runner::isRunning(string $id): bool`, `Runner::runningSince(string $id): ?int`, `Runner::abort(string $id): bool`, `Runner::log(string $id, int $which = 0, int $offset = 0): array{text:string,size:int}`, `Runner::clearLogs(string $id): void`, `Runner::tail(string $id, int $lines = 10): string`, `Runner::TRIGGERS`. CLI: `run.php --script <id> --trigger <t>`, `run.php --frequency <f>`, `run.php --sync-history`, `run.php --seed`, `run.php --abort <id>`.

- [ ] **Step 1: Write failing tests**

`tests/RunnerTest.php`:
```php
<?php
declare(strict_types=1);

final class RunnerTest extends TestCase
{
    public function testRunsAndRecordsExitAndLog(): void
    {
        $this->makeScript('ok', "#!/bin/bash\necho hello \$1\n#not a header\n");
        $this->makeScript('bad', "echo nope\nexit 3\n");
        $this->assertSame('ok', Runner::run('ok', 'manual'));
        $this->assertSame('failed', Runner::run('bad', 'custom'));
        $this->assertSame(3, History::last('bad')['exit']);
        $this->assertStringContainsString('hello', Runner::log('ok')['text']);
        $this->assertStringContainsString('nope', Runner::log('bad')['text']);
        $this->assertFalse(Runner::isRunning('ok'));
    }

    public function testArgumentDefaultAndNameWithSpacesAndPercent(): void
    {
        $this->makeScript('it is 100% ok', "#!/bin/bash\n#argumentDefault=one two\necho \"args=\$*\"\n");
        $this->assertSame('ok', Runner::run('it is 100% ok', 'custom'));
        $this->assertStringContainsString('args=one two', Runner::log('it is 100% ok')['text']);
    }

    public function testConditionsBlock(): void
    {
        file_put_contents(Paths::$varIni, "mdState=\"STOPPED\"\nmdResyncPos=\"0\"\n");
        $this->makeScript('arr', "#arrayStarted=true\necho x\n");
        $this->makeScript('fg', "#foregroundOnly=true\necho x\n");
        $this->assertSame('blocked', Runner::run('arr', 'daily'));
        $this->assertSame('blocked', Runner::run('fg', 'daily'));
        $this->assertSame('ok', Runner::run('fg', 'manual'));
        file_put_contents(Paths::$varIni, "mdState=\"STARTED\"\nmdResyncPos=\"12345\"\n");
        $this->makeScript('np', "#noParity=true\necho x\n");
        $this->assertSame('blocked', Runner::run('np', 'daily'));
        $this->assertNull(History::last('np'), 'blocked runs are not history');
    }

    public function testLogRotationKeepsFive(): void
    {
        $this->makeScript('r', "echo run \$RANDOM\n");
        for ($i = 0; $i < 7; $i++) Runner::run('r', 'custom');
        $this->assertFileExists(Paths::work('r') . '/log.5.txt');
        $this->assertFileDoesNotExist(Paths::work('r') . '/log.6.txt');
        $this->assertNotSame('', Runner::log('r', 5)['text']);
    }

    public function testClearLogVarSkipsRotation(): void
    {
        $this->makeScript('c', "#clearLog=true\necho x\n");
        Runner::run('c', 'custom'); Runner::run('c', 'custom');
        $this->assertFileDoesNotExist(Paths::work('c') . '/log.1.txt');
    }

    public function testFrequencyRunsScheduledScripts(): void
    {
        $this->makeScript('d1', "echo a\n"); $this->makeScript('d2', "echo b\n"); $this->makeScript('h', "echo c\n");
        Schedule::apply(['d1' => ['frequency' => 'daily'], 'd2' => ['frequency' => 'daily'], 'h' => ['frequency' => 'hourly']]);
        Runner::runFrequency('daily');
        $this->assertSame('daily', History::last('d1')['trigger']);
        $this->assertNotNull(History::last('d2'));
        $this->assertNull(History::last('h'));
    }

    public function testStatusChangeSyncsImmediately(): void
    {
        $this->makeScript('s', "exit 0\n");
        Runner::run('s', 'custom');
        $this->assertFileExists(Paths::$flash . '/history/s.jsonl', 'first entry is a change');
        $before = file_get_contents(Paths::$flash . '/history/s.jsonl');
        Runner::run('s', 'custom');
        $this->assertSame($before, file_get_contents(Paths::$flash . '/history/s.jsonl'), 'repeat stays in RAM');
    }
}
```

`tests/runner.sh` (process-level behaviour that needs real signals and parallel processes):
```bash
#!/bin/bash
# Integration tests for run.php: abort, timeout, overlap modes, stale lock. Needs setsid, pkill, ps.
set -u
ROOT=$(mktemp -d); trap 'rm -rf "$ROOT"' EXIT
export US_TEST=1 US_FLASH="$ROOT/flash" US_TMP="$ROOT/tmp" US_VARINI="$ROOT/var.ini" US_NOTIFY="$ROOT/notify" US_UPDATE_CRON="$ROOT/none"
RUN="php $(pwd)/source/user.scripts/usr/local/emhttp/plugins/user.scripts/run.php"
mkdir -p "$US_FLASH/scripts" "$US_TMP"
printf '#!/bin/sh\necho "$@" >> %s/sent\n' "$ROOT" > "$US_NOTIFY"; chmod +x "$US_NOTIFY"
fail=0
mk() { mkdir -p "$US_FLASH/scripts/$1"; printf '%b' "$2" > "$US_FLASH/scripts/$1/script"; }
opt() { php -r "require '$(pwd)/source/user.scripts/usr/local/emhttp/plugins/user.scripts/lib/bootstrap.php'; Schedule::set('$1', json_decode('$2', true));"; }
last() { tail -n1 "$US_TMP/history/$(php -r "echo rawurlencode('$1');").jsonl" | php -r '$j=json_decode(stream_get_contents(STDIN),true); echo $j["'"$2"'"];'; }
check() { if [ "$2" = "$3" ]; then echo "ok   $1"; else echo "FAIL $1: expected '$3' got '$2'"; fail=1; fi; }
wait_running() { for _ in $(seq 50); do [ -f "$US_TMP/running/$1" ] && return; sleep 0.1; done; }

# abort kills the whole group, including a TERM-ignoring background child
mk abortme '#!/bin/bash\ntrap "" TERM\n(trap "" TERM; sleep 300) &\nsleep 300\n'
$RUN --script abortme --trigger background & R=$!
wait_running abortme
$RUN --abort abortme
wait $R
check "abort status" "$(last abortme status)" aborted
check "abort leaves no sleep" "$(pgrep -f 'sleep 300' | wc -l | tr -d ' ')" 0

# timeout
mk slow '#!/bin/bash\nsleep 30\n'
opt slow '{"timeout":2}'
$RUN --script slow --trigger custom
check "timeout status" "$(last slow status)" timeout
check "timeout exit" "$(last slow exit)" 124

# overlap: skip (default) records skipped and alerts stuck at hangAfter
mk long '#!/bin/bash\nsleep 4\n'
opt long '{"notify":"failure","hangAfter":2}'
$RUN --script long --trigger custom & R=$!
wait_running long
$RUN --script long --trigger custom; $RUN --script long --trigger custom
check "skip recorded" "$(last long status)" skipped
check "stuck alert sent" "$(grep -c 'looks stuck' "$ROOT/sent" 2>/dev/null)" 1
wait $R
check "first run ok" "$(last long status)" ok

# overlap: queue runs exactly one waiter
mk q '#!/bin/bash\necho run >> '"$ROOT"'/q.count\nsleep 2\n'
opt q '{"overlap":"queue"}'
$RUN --script q --trigger custom & A=$!
wait_running q
$RUN --script q --trigger custom & B=$!
sleep 0.3
$RUN --script q --trigger custom   # third: skipped because one is already waiting
wait $A $B
check "queue ran twice" "$(wc -l < "$ROOT/q.count" | tr -d ' ')" 2

# overlap: parallel runs both
mk p '#!/bin/bash\necho run >> '"$ROOT"'/p.count\nsleep 2\n'
opt p '{"overlap":"parallel"}'
$RUN --script p --trigger custom & A=$!
wait_running p
$RUN --script p --trigger custom
wait $A
check "parallel ran twice" "$(wc -l < "$ROOT/p.count" | tr -d ' ')" 2

# a kill -9 of runner and script leaves no stale lock
mk stale '#!/bin/bash\nsleep 300\n'
$RUN --script stale --trigger custom & R=$!
wait_running stale
PG=$(cat "$US_TMP/running/stale"); kill -9 "$R" 2>/dev/null; kill -9 -- "-$PG" 2>/dev/null; sleep 0.5
php -r "require '$(pwd)/source/user.scripts/usr/local/emhttp/plugins/user.scripts/lib/bootstrap.php'; echo Runner::isRunning('stale') ? 'running' : 'free';" > "$ROOT/st"
check "no stale lock" "$(cat "$ROOT/st")" free

exit $fail
```

- [ ] **Step 2: Run to verify failure**

Run: `docker run --rm -u "$(id -u):$(id -g)" -v "$PWD":/app -w /app php:8.3-cli vendor/bin/phpunit --filter RunnerTest`
Expected: FAIL, `Class "Runner" not found`.

- [ ] **Step 3: Implement Runner**

`P/lib/Runner.php`:
```php
<?php
declare(strict_types=1);

// The only code that executes scripts. One process per run; holds the script's flock for its lifetime.
final class Runner
{
    public const TRIGGERS = ['manual', 'background', 'hourly', 'daily', 'weekly', 'monthly', 'custom', 'start', 'boot', 'stop'];
    private const LOGS = 5;
    private const ENV = ['PATH' => '/usr/local/sbin:/usr/sbin:/sbin:/usr/local/bin:/usr/bin:/bin', 'HOME' => '/root', 'USER' => 'root', 'SHELL' => '/bin/bash'];

    private static function lockFile(string $id): string { return Paths::locks() . '/' . rawurlencode($id) . '.lock'; }
    private static function logFile(string $id, int $which = 0): string { return Paths::work($id) . ($which ? "/log.$which.txt" : '/log.txt'); }

    public static function isRunning(string $id): bool
    {
        $f = self::lockFile($id);
        if (!is_file($f)) return false;
        $h = fopen($f, 'c');
        $free = flock($h, LOCK_EX | LOCK_NB);
        if ($free) flock($h, LOCK_UN);
        fclose($h);
        if ($free) @unlink(Paths::running() . "/$id"); // pid file left by a crash
        return !$free;
    }

    public static function runningSince(string $id): ?int
    {
        $f = Paths::running() . "/$id";
        return self::isRunning($id) && is_file($f) ? (int)filemtime($f) : null;
    }

    public static function runFrequency(string $f): void
    {
        foreach (Schedule::byFrequency($f) as $id) self::run($id, $f);
    }

    public static function run(string $id, string $trigger): string
    {
        $file = Scripts::file($id);
        $name = Scripts::name($id);
        $vars = Vars::typed($file);
        $opts = Schedule::get($id);
        $sys = Sys::var();
        if (!empty($vars['arrayStarted']) && ($sys['mdState'] ?? '') !== 'STARTED') { Sys::log("$name: array not started, not running"); return 'blocked'; }
        if (!empty($vars['noParity']) && (int)($sys['mdResyncPos'] ?? 0) > 0) { Sys::log("$name: parity check/rebuild in progress, not running"); return 'blocked'; }
        if (!empty($vars['foregroundOnly']) && $trigger !== 'manual') return 'blocked';

        @mkdir(Paths::locks(), 0777, true);
        $lock = fopen(self::lockFile($id), 'c');
        if (!flock($lock, LOCK_EX | LOCK_NB)) {
            $mode = $opts['overlap'];
            $queued = null;
            if ($mode === 'queue') {
                $queued = fopen(self::lockFile($id) . '.queue', 'c');
                if (flock($queued, LOCK_EX | LOCK_NB)) { flock($lock, LOCK_EX); flock($queued, LOCK_UN); fclose($queued); $mode = 'go'; }
            }
            if ($mode === 'parallel') { fclose($lock); $lock = null; }
            elseif ($mode !== 'go') {
                $now = time();
                History::record($id, ['start' => $now, 'end' => $now, 'exit' => 0, 'trigger' => $trigger, 'status' => 'skipped']);
                Alerts::onSkip($id, $name, $opts, self::runningSince($id) ?? $now, $now);
                Sys::log("$name: still running, $trigger run skipped");
                return 'skipped';
            }
        }

        try {
            Alerts::onStart($id);
            return self::execute($id, $name, $file, $vars, $opts, $trigger, $lock);
        } finally {
            if ($lock) { flock($lock, LOCK_UN); fclose($lock); }
        }
    }

    /** @param resource|null $lock */
    private static function execute(string $id, string $name, string $file, array $vars, array $opts, string $trigger, $lock): string
    {
        $work = Paths::work($id);
        @mkdir($work, 0777, true);
        foreach ([Paths::running(), Paths::finished(), Paths::aborted()] as $d) @mkdir($d, 0777, true);
        @unlink(Paths::aborted() . "/$id");

        $body = str_replace("\r", '', (string)@file_get_contents($file));
        if (!str_starts_with($body, '#!')) $body = "#!/bin/bash\n" . $body;
        file_put_contents("$work/script", $body);
        chmod("$work/script", 0755);
        self::rotate($id, !empty($vars['clearLog']));

        $log = self::logFile($id);
        $start = time();
        file_put_contents($log, 'Script Starting ' . date('M d, Y  H:i.s') . "\n\nFull logs for this script are available at $log\n\n", FILE_APPEND);
        $pidFile = Paths::running() . "/$id";
        $args = preg_split('/\s+/', trim((string)($vars['argumentDefault'] ?? '')), -1, PREG_SPLIT_NO_EMPTY);
        $target = !empty($vars['directPHP']) ? ['/usr/bin/php', "$work/script"] : ["$work/script"];
        // the session leader writes its own pid (== pgid) before exec'ing the script
        $cmd = ['setsid', '-w', '/bin/bash', '-c', 'echo $$ > "$0"; shift; exec "$@"', $pidFile, '_', ...$target, ...$args];
        $spec = [0 => ['file', '/dev/null', 'r'], 1 => ['file', $log, 'a'], 2 => ['file', $log, 'a']];
        if ($lock) $spec[9] = $lock; // children inherit the lock: it is held until the whole group exits
        $proc = proc_open($cmd, $spec, $pipes, $work, self::ENV + array_filter(['LANG' => getenv('LANG') ?: null, 'TZ' => getenv('TZ') ?: null]));
        if (!is_resource($proc)) {
            $run = ['start' => $start, 'end' => time(), 'exit' => -1, 'trigger' => $trigger, 'status' => 'failed'];
            History::record($id, $run);
            History::sync($id);
            Sys::log("$name: could not start");
            return 'failed';
        }

        $status = null;
        $exit = 0;
        $timeout = (int)$opts['timeout'];
        while (true) {
            $st = proc_get_status($proc);
            if (!$st['running']) { $exit = (int)$st['exitcode']; break; }
            if ($timeout > 0 && time() - $start >= $timeout && $status === null) {
                $status = 'timeout';
                file_put_contents($log, "\nTimed out after {$timeout}s\n", FILE_APPEND);
                self::killGroup($id);
            }
            usleep(200000);
        }
        proc_close($proc);
        if ($status === 'timeout') $exit = 124;
        elseif (is_file(Paths::aborted() . "/$id")) $status = 'aborted';
        else $status = $exit === 0 ? 'ok' : 'failed';
        @unlink(Paths::aborted() . "/$id");
        @unlink($pidFile);
        file_put_contents(Paths::finished() . "/$id", $status);
        file_put_contents($log, "\nScript Finished " . date('M d, Y  H:i.s') . "\n\nFull logs for this script are available at $log\n\n", FILE_APPEND);

        $run = ['start' => $start, 'end' => time(), 'exit' => $exit, 'trigger' => $trigger, 'status' => $status];
        if (History::record($id, $run)) History::sync($id);
        Alerts::onRun($id, $name, $run, $opts, self::tail($id, 10));
        return $status;
    }

    private static function rotate(string $id, bool $clear): void
    {
        $cur = self::logFile($id);
        if ($clear) { @unlink($cur); return; }
        if (!is_file($cur) || filesize($cur) === 0) return;
        @unlink(self::logFile($id, self::LOGS));
        for ($i = self::LOGS - 1; $i >= 1; $i--) {
            if (is_file(self::logFile($id, $i))) rename(self::logFile($id, $i), self::logFile($id, $i + 1));
        }
        rename($cur, self::logFile($id, 1));
    }

    private static function killGroup(string $id): void
    {
        $pg = (int)@file_get_contents(Paths::running() . "/$id");
        if ($pg <= 1) return;
        exec("kill -TERM -- -$pg 2>/dev/null");
        for ($i = 0; $i < 25; $i++) {
            exec("kill -0 -- -$pg 2>/dev/null", $o, $rc);
            if ($rc !== 0) return;
            usleep(200000);
        }
        exec("kill -KILL -- -$pg 2>/dev/null");
    }

    public static function abort(string $id): bool
    {
        Scripts::resolve($id);
        if (!self::isRunning($id)) return false;
        @mkdir(Paths::aborted(), 0777, true);
        file_put_contents(Paths::aborted() . "/$id", '1');
        file_put_contents(self::logFile($id), "\nExecution was aborted by user\n", FILE_APPEND);
        self::killGroup($id);
        return true;
    }

    /** @return array{text:string,size:int} */
    public static function log(string $id, int $which = 0, int $offset = 0): array
    {
        Scripts::resolve($id);
        $f = self::logFile($id, max(0, min(self::LOGS, $which)));
        if (!is_file($f)) return ['text' => '', 'size' => 0];
        $size = (int)filesize($f);
        $offset = max(0, min($offset, $size));
        $max = 262144;
        if ($size - $offset > $max) $offset = $size - $max;
        $text = (string)file_get_contents($f, false, null, $offset);
        return ['text' => mb_convert_encoding($text, 'UTF-8', 'UTF-8'), 'size' => $size];
    }

    public static function clearLogs(string $id): void
    {
        Scripts::resolve($id);
        for ($i = 0; $i <= self::LOGS; $i++) @unlink(self::logFile($id, $i));
    }

    public static function tail(string $id, int $lines = 10): string
    {
        $f = self::logFile($id);
        if (!is_file($f)) return '';
        $all = array_filter(explode("\n", (string)file_get_contents($f, false, null, max(0, (int)filesize($f) - 16384))),
            fn($l) => !str_starts_with($l, 'Full logs for this script') && !str_starts_with($l, 'Script Finished') && trim($l) !== '');
        return implode("\n", array_slice($all, -$lines));
    }
}
```

- [ ] **Step 4: Implement run.php**

`P/run.php`:
```php
#!/usr/bin/php
<?php
declare(strict_types=1);

// CLI entry point for every execution path: cron, emhttp events, the UI and legacy shims.
require_once __DIR__ . '/lib/bootstrap.php';

$o = getopt('', ['script:', 'trigger:', 'frequency:', 'abort:', 'sync-history', 'seed']);
try {
    if (isset($o['sync-history'])) { History::sync(); exit(0); }
    if (isset($o['seed'])) { History::seed(); Schedule::writeCron(); exit(0); }
    if (isset($o['abort'])) { exit(Runner::abort((string)$o['abort']) ? 0 : 1); }
    if (isset($o['frequency'])) {
        if (!in_array($o['frequency'], Schedule::FREQUENCIES, true)) throw new InvalidArgumentException('bad frequency');
        Runner::runFrequency((string)$o['frequency']);
        exit(0);
    }
    if (isset($o['script'])) {
        $trigger = (string)($o['trigger'] ?? 'manual');
        if (!in_array($trigger, Runner::TRIGGERS, true)) throw new InvalidArgumentException('bad trigger');
        $status = Runner::run((string)$o['script'], $trigger);
        exit($status === 'ok' || $status === 'blocked' || $status === 'skipped' ? 0 : 1);
    }
    fwrite(STDERR, "usage: run.php --script <id> [--trigger t] | --frequency f | --abort id | --sync-history | --seed\n");
    exit(2);
} catch (InvalidArgumentException $e) {
    Sys::log('run.php: ' . $e->getMessage());
    exit(2);
}
```

```bash
chmod +x source/user.scripts/usr/local/emhttp/plugins/user.scripts/run.php tests/runner.sh
```

- [ ] **Step 5: Run unit + integration tests**

Run: `bin/test 8.3`
Expected: PHPUnit PASS; runner.sh prints only `ok` lines and exits 0.

- [ ] **Step 6: Commit**

```bash
git add -A && git commit -m "feat: Runner with flock overlap modes, setsid group abort, timeout and log rotation (fixes #18)

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Legacy shims, cron wrappers, event hooks

**Files:**
- Modify (rewrite): `P/helpers.php`, `P/startSchedule.php`, `P/startCustom.php`, `P/startBackground.php`, `P/backgroundScript.sh`, `P/startScript.sh`, `P/exec.php`, `P/cron/user.script.start.{hourly,daily,weekly,monthly}.sh`, `P/event/disks_mounted`, `P/event/stopping_svcs`
- Create: `P/event/stopping`, `P/event/unmounting_disks`, `tests/ShimsTest.php`

**Interfaces:**
- Consumes: `run.php` CLI, `Vars`, `Scripts::idFromPath()`, `Runner::abort()`, `Runner::clearLogs()`
- Produces: upstream-compatible entry points; `helpers.php` keeps `startsWith()`, `getRawVariables()`, `getScriptVariables()`, `logger()`.

- [ ] **Step 1: Write failing test**

`tests/ShimsTest.php`:
```php
<?php
declare(strict_types=1);

final class ShimsTest extends TestCase
{
    private function php(string $file, string $args): array
    {
        $P = __DIR__ . '/../source/user.scripts/usr/local/emhttp/plugins/user.scripts';
        $env = 'US_TEST=1 US_FLASH=' . escapeshellarg(Paths::$flash) . ' US_TMP=' . escapeshellarg(Paths::$tmp) . ' US_VARINI=' . escapeshellarg(Paths::$varIni);
        exec("$env php $P/$file $args 2>&1", $out, $rc);
        return [$rc, implode("\n", $out)];
    }

    public function testStartCustomEmptyPathExitsCleanly(): void
    {
        [$rc, $out] = $this->php('startCustom.php', '');
        $this->assertSame(0, $rc);
        $this->assertStringNotContainsString('Fatal', $out);
        [$rc] = $this->php('startBackground.php', "''");
        $this->assertSame(0, $rc);
    }

    public function testStartCustomRunsUpstreamPathWithSpaces(): void
    {
        $this->makeScript('clear subtitles', "echo cleaned\n");
        [$rc] = $this->php('startCustom.php', escapeshellarg(Paths::scripts() . '/clear subtitles/script'));
        $this->assertSame(0, $rc);
        $this->assertSame('custom', History::last('clear subtitles')['trigger']);
    }

    public function testStartScheduleDelegates(): void
    {
        $this->makeScript('d', "echo d\n");
        Schedule::apply(['d' => ['frequency' => 'daily']]);
        $this->php('startSchedule.php', 'daily');
        $this->assertSame('daily', History::last('d')['trigger']);
    }

    public function testHelpersLegacyFunctions(): void
    {
        require_once __DIR__ . '/../source/user.scripts/usr/local/emhttp/plugins/user.scripts/helpers.php';
        $d = $this->makeScript('h', "#arrayStarted=true\n");
        $this->assertSame([], getRawVariables(''));
        $this->assertTrue(getScriptVariables("$d/script")['arrayStarted']);
        $this->assertTrue(startsWith('#!/bin/bash', '#!'));
    }
}
```

- [ ] **Step 2: Run to verify failure**

Run: `docker run --rm -u "$(id -u):$(id -g)" -v "$PWD":/app -w /app php:8.3-cli vendor/bin/phpunit --filter ShimsTest`
Expected: FAIL (upstream `startCustom.php` hardcodes `/usr/local/emhttp/...` requires and fatals).

- [ ] **Step 3: Rewrite shims**

`P/helpers.php`:
```php
<?php
// Legacy functions kept for anything that still includes helpers.php (e.g. User Scripts Enhanced).
require_once __DIR__ . '/lib/bootstrap.php';

if (!function_exists('startsWith')) {
    function startsWith($haystack, $needle) { return $needle === '' || str_starts_with((string)$haystack, (string)$needle); }
}
if (!function_exists('getRawVariables')) {
    function getRawVariables($filename) { return Vars::raw((string)$filename); }
}
if (!function_exists('getScriptVariables')) {
    function getScriptVariables($filename) { return Vars::typed((string)$filename); }
}
if (!function_exists('logger')) {
    function logger($string) { Sys::log((string)$string); }
}
```

`P/startSchedule.php`:
```php
#!/usr/bin/php
<?php
// Legacy entry point: startSchedule.php <frequency>
require_once __DIR__ . '/lib/bootstrap.php';
$f = (string)($argv[1] ?? '');
if (in_array($f, Schedule::FREQUENCIES, true)) Runner::runFrequency($f);
```

`P/startCustom.php`:
```php
#!/usr/bin/php
<?php
// Legacy entry point: startCustom.php <script path> (path may contain unquoted spaces)
require_once __DIR__ . '/lib/bootstrap.php';
$id = Scripts::idFromPath(trim(implode(' ', array_slice($argv, 1))));
if ($id !== null) Runner::run($id, 'custom');
```

`P/startBackground.php`:
```php
#!/usr/bin/php
<?php
// Legacy entry point: startBackground.php <script path>
require_once __DIR__ . '/lib/bootstrap.php';
$id = Scripts::idFromPath(trim(implode(' ', array_slice($argv, 1))));
if ($id !== null) Runner::run($id, 'background');
```

`P/backgroundScript.sh`:
```bash
#!/bin/bash
# Legacy entry point: backgroundScript.sh <script path>
id=$(basename "$(dirname "$1")")
echo "/usr/local/emhttp/plugins/user.scripts/run.php --script $(printf '%q' "$id") --trigger background" | at -M NOW >/dev/null 2>&1
echo "Script $id started in the background"
```

`P/startScript.sh`:
```bash
#!/bin/bash
# Legacy entry point: startScript.sh <script path>; runs in the foreground and prints the log
id=$(basename "$(dirname "$1")")
/usr/local/emhttp/plugins/user.scripts/run.php --script "$id" --trigger manual
cat "/tmp/user.scripts/tmpScripts/$id/log.txt" 2>/dev/null
```

`P/exec.php`:
```php
<?php
// Legacy AJAX endpoint. The UI uses api.php; only the two actions external tools used remain.
require_once __DIR__ . '/lib/bootstrap.php';
$name = urldecode((string)($_POST['name'] ?? ''));
try {
    switch ($_POST['action'] ?? '') {
        case 'abortScript': Runner::abort($name); echo 'ok'; break;
        case 'deleteLog': Runner::clearLogs($name); echo 'ok'; break;
        default: http_response_code(410); echo 'use /plugins/user.scripts/api.php';
    }
} catch (InvalidArgumentException) {
    http_response_code(404);
}
```

Cron wrappers, each file one line after the shebang (`hourly` shown; daily/weekly/monthly substitute the word):
```bash
#!/bin/bash
/usr/local/emhttp/plugins/user.scripts/run.php --frequency hourly
```

`P/event/disks_mounted`:
```bash
#!/bin/bash
/usr/local/emhttp/plugins/user.scripts/run.php --seed
/usr/local/emhttp/plugins/user.scripts/run.php --frequency start
if [ ! -e /tmp/user.scripts/booted ]; then
  mkdir -p /tmp/user.scripts
  echo booted > /tmp/user.scripts/booted
  /usr/local/emhttp/plugins/user.scripts/run.php --frequency boot
fi
```

`P/event/stopping_svcs`:
```bash
#!/bin/bash
/usr/local/emhttp/plugins/user.scripts/run.php --frequency stop
/usr/local/emhttp/plugins/user.scripts/run.php --sync-history
```

`P/event/stopping` and `P/event/unmounting_disks` (identical):
```bash
#!/bin/bash
/usr/local/emhttp/plugins/user.scripts/run.php --sync-history
```

```bash
chmod +x $P/*.php $P/*.sh $P/cron/*.sh $P/event/*
```

- [ ] **Step 4: Run tests**

Run: `bin/test 8.3`
Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat: legacy shims delegate to run.php; history sync on array stop/shutdown

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: api.php (JSON API)

**Files:**
- Create: `P/api.php`, `P/lib/Api.php`, `tests/ApiTest.php`

**Interfaces:**
- Consumes: all lib classes
- Produces: `Api::handle(array $in): array` (pure dispatch, throws `ApiError(string $msg, int $code)`), `ApiError`. HTTP: `POST /plugins/user.scripts/api.php` with header `X-CSRF-Token` and JSON body `{"action": "...", ...}`; response `{"ok":true,...}` / `{"ok":false,"error":"..."}`.
- Actions and payloads (used verbatim by `ui/app.js` in Task 10):
  - `list` → `{scripts:[{id,name,description,missing,vars,schedule,cron,next,running,last,health}], settings}`; `cron` = human text, `next` = unix ts or null, `running` = `{since}` or null, `health` = State entry.
  - `run` / `background` `{id}` → `{started:true}`; 409 when running and overlap ≠ parallel; 400 when `foregroundOnly` for background / `backgroundOnly` for run.
  - `abort` `{id}` → `{aborted:bool}`
  - `log` `{id, which?, offset?}` → `{text,size,running}`
  - `clearLog` `{id}` → `{}`
  - `history` `{id, limit?}` → `{runs:[...]}`
  - `get` `{id}` → `{body}`; `save` `{id, body}` → `{}`
  - `create` `{name}` → `{id}`; `rename` `{id,name}`; `describe` `{id,description}`; `delete` `{id}` (409 while running)
  - `schedule.set` `{id, fields}` → `{schedule}` then cron rewrite; `apply` `{changes:{id:fields}}` → `{}`
  - `cron.check` `{expr}` → `{valid, text, next}`
  - `settings.get` → `{settings}`; `settings.set` `{settings}` → `{settings}`

- [ ] **Step 1: Write failing tests**

`tests/ApiTest.php`:
```php
<?php
declare(strict_types=1);

final class ApiTest extends TestCase
{
    public function testListShape(): void
    {
        $this->makeScript('a', "#!/bin/bash\n#description=Hello\nexit 0\n");
        Schedule::apply(['a' => ['frequency' => 'custom', 'custom' => '*/5 * * * *']]);
        Runner::run('a', 'custom');
        $r = Api::handle(['action' => 'list']);
        $s = $r['scripts'][0];
        $this->assertSame(['id', 'name', 'description', 'missing', 'vars', 'schedule', 'cron', 'next', 'running', 'last', 'health'], array_keys($s));
        $this->assertSame('every 5 minutes', $s['cron']);
        $this->assertSame('ok', $s['last']['status']);
        $this->assertIsInt($s['next']);
        $this->assertArrayHasKey('settings', $r);
    }

    public function testUnknownScriptIs404AndUnknownActionIs400(): void
    {
        foreach ([['action' => 'get', 'id' => '../../etc'], ['action' => 'delete', 'id' => 'nope']] as $in) {
            try { Api::handle($in); $this->fail(); } catch (ApiError $e) { $this->assertSame(404, $e->getCode()); }
        }
        try { Api::handle(['action' => 'rm -rf']); $this->fail(); } catch (ApiError $e) { $this->assertSame(400, $e->getCode()); }
    }

    public function testRunSpawnsDetachedAndRespectsLock(): void
    {
        $this->makeScript('a', "sleep 1\n");
        Api::handle(['action' => 'background', 'id' => 'a']);
        $this->assertStringContainsString("run.php --script 'a' --trigger background", Sys::$spawned[0]);
        @mkdir(Paths::locks(), 0777, true);
        $h = fopen(Paths::locks() . '/a.lock', 'c'); flock($h, LOCK_EX);
        try { Api::handle(['action' => 'run', 'id' => 'a']); $this->fail(); } catch (ApiError $e) { $this->assertSame(409, $e->getCode()); }
        try { Api::handle(['action' => 'delete', 'id' => 'a']); $this->fail(); } catch (ApiError $e) { $this->assertSame(409, $e->getCode()); }
        flock($h, LOCK_UN);
    }

    public function testForegroundOnlyCannotBackground(): void
    {
        $this->makeScript('fg', "#foregroundOnly=true\n");
        try { Api::handle(['action' => 'background', 'id' => 'fg']); $this->fail(); } catch (ApiError $e) { $this->assertSame(400, $e->getCode()); }
    }

    public function testScheduleValidationIs400WithMessage(): void
    {
        $this->makeScript('a');
        try { Api::handle(['action' => 'schedule.set', 'id' => 'a', 'fields' => ['frequency' => 'custom', 'custom' => 'bogus']]); $this->fail(); }
        catch (ApiError $e) { $this->assertSame(400, $e->getCode()); $this->assertSame('invalid cron expression', $e->getMessage()); }
        $r = Api::handle(['action' => 'cron.check', 'expr' => '0 3 * * 1']);
        $this->assertTrue($r['valid']);
        $this->assertSame('Mon at 03:00', $r['text']);
    }

    public function testCrudRoundTrip(): void
    {
        $id = Api::handle(['action' => 'create', 'name' => 'New one'])['id'];
        Api::handle(['action' => 'save', 'id' => $id, 'body' => "#!/bin/bash\necho x\n"]);
        $this->assertSame("#!/bin/bash\necho x\n", Api::handle(['action' => 'get', 'id' => $id])['body']);
        Api::handle(['action' => 'describe', 'id' => $id, 'description' => 'D']);
        Api::handle(['action' => 'rename', 'id' => $id, 'name' => 'Renamed']);
        $this->assertSame('Renamed', Api::handle(['action' => 'list'])['scripts'][0]['name']);
        Api::handle(['action' => 'delete', 'id' => $id]);
        $this->assertSame([], Api::handle(['action' => 'list'])['scripts']);
    }
}
```

- [ ] **Step 2: Run to verify failure**

Run: `docker run --rm -u "$(id -u):$(id -g)" -v "$PWD":/app -w /app php:8.3-cli vendor/bin/phpunit --filter ApiTest`
Expected: FAIL, `Class "Api" not found`.

- [ ] **Step 3: Implement Api**

Add `'Api'` to the class list in `P/lib/bootstrap.php` (after `'Runner'`).

`P/lib/Api.php`:
```php
<?php
declare(strict_types=1);

final class ApiError extends RuntimeException {}

// JSON API for the UI. Pure dispatch: input array in, response array out, ApiError on failure.
final class Api
{
    private static function id(array $in): string
    {
        $id = (string)($in['id'] ?? '');
        try { Scripts::resolve($id); } catch (InvalidArgumentException) { throw new ApiError('unknown script', 404); }
        return $id;
    }

    private static function row(string $id): array
    {
        $file = Paths::scripts() . "/$id/script";
        $sched = Schedule::get($id);
        $since = Runner::runningSince($id);
        $custom = $sched['frequency'] === 'custom' && Cron::valid($sched['custom']);
        return [
            'id' => $id,
            'name' => Scripts::name($id),
            'description' => Scripts::description($id),
            'missing' => !is_file($file),
            'vars' => Vars::typed($file),
            'schedule' => $sched,
            'cron' => $custom ? Cron::describe($sched['custom']) : '',
            'next' => $custom ? Cron::next($sched['custom'], time()) : null,
            'running' => $since !== null ? ['since' => $since] : null,
            'last' => History::last($id),
            'health' => State::get($id),
        ];
    }

    private static function spawnRun(string $id, string $trigger): void
    {
        Sys::spawn(Paths::$plugin . '/run.php --script ' . escapeshellarg($id) . ' --trigger ' . $trigger);
    }

    public static function handle(array $in): array
    {
        try {
            return self::dispatch($in);
        } catch (InvalidArgumentException $e) {
            throw new ApiError($e->getMessage(), 400);
        }
    }

    private static function dispatch(array $in): array
    {
        switch ($in['action'] ?? '') {
            case 'list':
                return ['scripts' => array_map([self::class, 'row'], Scripts::ids()), 'settings' => Settings::get()];
            case 'run':
            case 'background':
                $id = self::id($in);
                $vars = Vars::typed(Scripts::file($id));
                if ($in['action'] === 'background' && !empty($vars['foregroundOnly'])) throw new ApiError('this script can only run in the foreground', 400);
                if ($in['action'] === 'run' && !empty($vars['backgroundOnly'])) throw new ApiError('this script can only run in the background', 400);
                if (Runner::isRunning($id) && Schedule::get($id)['overlap'] !== 'parallel') throw new ApiError('already running', 409);
                self::spawnRun($id, $in['action'] === 'run' ? 'manual' : 'background');
                return ['started' => true];
            case 'abort':
                return ['aborted' => Runner::abort(self::id($in))];
            case 'log':
                $id = self::id($in);
                return Runner::log($id, (int)($in['which'] ?? 0), (int)($in['offset'] ?? 0)) + ['running' => Runner::isRunning($id)];
            case 'clearLog':
                Runner::clearLogs(self::id($in));
                return [];
            case 'history':
                return ['runs' => History::read(self::id($in), max(1, min(1000, (int)($in['limit'] ?? 50))))];
            case 'get':
                return ['body' => Scripts::read(self::id($in))];
            case 'save':
                Scripts::save(self::id($in), (string)($in['body'] ?? ''));
                return [];
            case 'create':
                return ['id' => Scripts::create((string)($in['name'] ?? ''))];
            case 'rename':
                Scripts::rename(self::id($in), (string)($in['name'] ?? ''));
                return [];
            case 'describe':
                Scripts::describe(self::id($in), (string)($in['description'] ?? ''));
                return [];
            case 'delete':
                $id = self::id($in);
                if (Runner::isRunning($id)) throw new ApiError('abort the script before deleting it', 409);
                Scripts::delete($id);
                Schedule::writeCron();
                return [];
            case 'schedule.set':
                $s = Schedule::set(self::id($in), (array)($in['fields'] ?? []));
                Schedule::writeCron();
                return ['schedule' => $s];
            case 'apply':
                $changes = (array)($in['changes'] ?? []);
                foreach (array_keys($changes) as $id) self::id(['id' => (string)$id]);
                Schedule::apply($changes);
                return [];
            case 'cron.check':
                $e = (string)($in['expr'] ?? '');
                $ok = Cron::valid($e);
                return ['valid' => $ok, 'text' => $ok ? Cron::describe($e) : '', 'next' => $ok ? Cron::next($e, time()) : null];
            case 'settings.get':
                return ['settings' => Settings::get()];
            case 'settings.set':
                $s = Settings::set((array)($in['settings'] ?? []));
                Schedule::writeCron();
                return ['settings' => $s];
            default:
                throw new ApiError('unknown action', 400);
        }
    }
}
```

`P/api.php`:
```php
<?php
declare(strict_types=1);

// HTTP wrapper around Api::handle(). Unraid's local_prepend already rejects POSTs with a bad
// csrf_token; we check again so the endpoint is safe even if that ever changes.
require_once __DIR__ . '/lib/bootstrap.php';

header('Content-Type: application/json');
header('Cache-Control: no-store');
ini_set('display_errors', '0');

function respond(int $code, array $body): never
{
    http_response_code($code);
    echo json_encode($body, JSON_UNESCAPED_SLASHES | JSON_INVALID_UTF8_SUBSTITUTE);
    exit;
}

if (($_SERVER['REQUEST_METHOD'] ?? '') !== 'POST') respond(405, ['ok' => false, 'error' => 'POST only']);
$token = $_SERVER['HTTP_X_CSRF_TOKEN'] ?? ($_POST['csrf_token'] ?? '');
$var = Sys::var();
if (!isset($var['csrf_token']) || !hash_equals((string)$var['csrf_token'], (string)$token)) respond(403, ['ok' => false, 'error' => 'bad csrf token']);

$in = json_decode((string)file_get_contents('php://input'), true);
if (!is_array($in)) $in = $_POST;
try {
    respond(200, ['ok' => true] + Api::handle($in));
} catch (ApiError $e) {
    respond($e->getCode() ?: 400, ['ok' => false, 'error' => $e->getMessage()]);
} catch (Throwable $e) {
    Sys::log('api: ' . $e->getMessage());
    respond(500, ['ok' => false, 'error' => 'internal error']);
}
```

- [ ] **Step 4: Run tests**

Run: `bin/test 8.3`
Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat: JSON API with CSRF check, path validation and lock-aware run/delete

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: UI shell, list, toolbar, polling

**Files:**
- Modify (rewrite): `P/Userscripts.page`
- Create: `P/ui/app.css`, `P/ui/app.js`

**Interfaces:**
- Consumes: `api.php` actions from Task 9.
- Produces (used by Task 11, same file `ui/app.js`): global `US` object with `US.api(action, data): Promise<object>`, `US.state` (`{scripts, filter, query, pending: {id: fields}}`), `US.render()`, `US.refresh()`, `US.toast(msg, kind)`, `US.fmtAgo(ts)`, `US.fmtDur(s)`, `US.fmtWhen(ts)`, `US.esc(s)`, and hooks `US.openLog(id)`, `US.openEdit(id)`, `US.openOptions(id)` (defined in Task 11; Task 10 calls them only from click handlers).

- [ ] **Step 1: Write the page shell**

`P/Userscripts.page`:
```php
Menu="Utilities"
Icon="file-text-o"
Title="User Scripts"
Tag="file-text-o"
---
<?php
/* User Scripts, Copyright 2016-2024 Andrew Zawadzki; VozDeOuro fork 2026. */
require_once '/usr/local/emhttp/plugins/user.scripts/lib/bootstrap.php';
@mkdir(Paths::scripts(), 0777, true);
?>
<link rel="stylesheet" href="<?autov('/plugins/user.scripts/ui/app.css')?>">
<div id="us-app" class="us" data-csrf="<?=htmlspecialchars($var['csrf_token'] ?? '')?>" data-theme="<?=htmlspecialchars($display['theme'] ?? 'white')?>">
  <div class="us-toolbar" role="toolbar" aria-label="User Scripts">
    <input type="search" id="us-search" class="us-search" placeholder="Search scripts  ( / )" aria-label="Search scripts">
    <div class="us-filters" role="tablist" aria-label="Filter">
      <button type="button" class="us-chip" data-filter="all" aria-pressed="true">All</button>
      <button type="button" class="us-chip" data-filter="scheduled" aria-pressed="false">Scheduled</button>
      <button type="button" class="us-chip" data-filter="running" aria-pressed="false">Running</button>
      <button type="button" class="us-chip" data-filter="failing" aria-pressed="false">Failing</button>
    </div>
    <span class="us-spacer"></span>
    <button type="button" id="us-settings">Settings</button>
    <button type="button" id="us-add">Add script</button>
    <button type="button" id="us-apply" disabled>Apply</button>
  </div>
  <div id="us-list" class="us-list" aria-live="polite"></div>
  <p id="us-empty" class="us-empty" hidden>No scripts match. Clear the search or add a script.</p>
  <div id="us-drawer" class="us-drawer" role="dialog" aria-modal="true" aria-labelledby="us-drawer-title" hidden>
    <div class="us-drawer-panel">
      <header class="us-drawer-head"><h2 id="us-drawer-title"></h2><button type="button" class="us-icon" id="us-drawer-close" aria-label="Close">✕</button></header>
      <div id="us-drawer-body" class="us-drawer-body"></div>
    </div>
  </div>
  <div id="us-toast" class="us-toast" role="status" aria-live="polite" hidden></div>
</div>
<script src="<?autov('/plugins/user.scripts/javascript/ace/ace.js')?>"></script>
<script src="<?autov('/plugins/user.scripts/ui/app.js')?>"></script>
```

- [ ] **Step 2: Write app.css**

`P/ui/app.css`:
```css
/* Follows Unraid theme variables (7.3); fallbacks keep 7.0-7.2 themes readable. */
.us {
  --us-bg: var(--background-color, #f2f2f2);
  --us-fg: var(--text-color, #1c1b1b);
  --us-muted: var(--alt-text-color, #606e7f);
  --us-line: var(--border-color, #d0d0d0);
  --us-row: var(--table-background-color, transparent);
  --us-hover: var(--hover-table-row-background-color, rgba(127,127,127,.08));
  --us-input: var(--input-bg-color, transparent);
  --us-accent: var(--brand-orange, #f2711c);
  --us-ok: #2e9d58; --us-fail: #d64545; --us-warn: #d99a1e; --us-run: #3b82c4;
  color: var(--us-fg);
  font-size: 1.3rem;
}
.us *, .us *::before, .us *::after { box-sizing: border-box; }
.us-toolbar { display: flex; flex-wrap: wrap; gap: .8rem; align-items: center; margin: 1rem 0 1.2rem; }
.us-search { flex: 1 1 22rem; max-width: 36rem; margin: 0; background: var(--us-input); }
.us-filters { display: flex; gap: .4rem; flex-wrap: wrap; }
.us-chip { margin: 0; padding: .4rem 1rem; min-width: 0; border-radius: 99px; font-size: 1.2rem; }
.us-chip[aria-pressed="true"] { border-color: var(--us-accent); color: var(--us-accent); }
.us-spacer { flex: 1; }
.us-toolbar > button { margin: 0; }

.us-list { display: grid; border-top: 1px solid var(--us-line); }
.us-row {
  display: grid;
  grid-template-columns: minmax(0, 2.4fr) minmax(0, 1.6fr) minmax(0, 1.4fr) auto;
  gap: 1rem 1.6rem; align-items: center;
  padding: 1rem .8rem; border-bottom: 1px solid var(--us-line); background: var(--us-row);
}
.us-row:hover { background: var(--us-hover); }
.us-name { display: flex; gap: .8rem; align-items: baseline; min-width: 0; }
.us-name h3 { margin: 0; font-size: 1.45rem; font-weight: 600; overflow-wrap: anywhere; }
.us-desc { margin: .2rem 0 0; color: var(--us-muted); font-size: 1.2rem; overflow-wrap: anywhere; }
.us-dot { flex: none; width: .9rem; height: .9rem; border-radius: 50%; background: var(--us-line); transform: translateY(.1rem); }
.us-dot.ok { background: var(--us-ok); } .us-dot.failed, .us-dot.timeout { background: var(--us-fail); }
.us-dot.running { background: var(--us-run); animation: us-pulse 1.2s ease-in-out infinite; }
@keyframes us-pulse { 50% { opacity: .35; } }
@media (prefers-reduced-motion: reduce) { .us-dot.running { animation: none; } }

.us-sched select { margin: 0; width: 100%; max-width: 22rem; background: var(--us-input); }
.us-sched input { margin: .4rem 0 0; width: 100%; max-width: 22rem; font-family: monospace; background: var(--us-input); }
.us-sched input[aria-invalid="true"] { border-color: var(--us-fail); }
.us-meta { display: block; margin-top: .3rem; color: var(--us-muted); font-size: 1.15rem; }
.us-meta.err { color: var(--us-fail); }

.us-last { font-size: 1.2rem; }
.us-pill { display: inline-block; padding: .05rem .7rem; border-radius: 99px; font-size: 1.1rem; font-weight: 600; border: 1px solid currentColor; }
.us-pill.ok { color: var(--us-ok); } .us-pill.failed, .us-pill.timeout { color: var(--us-fail); }
.us-pill.aborted, .us-pill.skipped { color: var(--us-warn); } .us-pill.running { color: var(--us-run); }
.us-badge { display: inline-block; margin-top: .3rem; color: var(--us-fail); font-size: 1.1rem; }

.us-actions { display: flex; gap: .4rem; flex-wrap: wrap; justify-content: flex-end; }
.us-actions button { margin: 0; padding: .5rem 1rem; min-width: 0; font-size: 1.2rem; }
.us-menu { position: relative; }
.us-menu-list { position: absolute; right: 0; top: calc(100% + .3rem); z-index: 20; min-width: 16rem; margin: 0; padding: .4rem 0; list-style: none;
  background: var(--us-bg); border: 1px solid var(--us-line); border-radius: 6px; box-shadow: var(--small-shadow, 0 4px 14px rgba(0,0,0,.18)); }
.us-menu-list button { display: block; width: 100%; margin: 0; padding: .6rem 1.2rem; text-align: left; border: 0; background: none; color: var(--us-fg); font-size: 1.25rem; }
.us-menu-list button:hover, .us-menu-list button:focus-visible { background: var(--us-hover); }
.us-menu-list .danger { color: var(--us-fail); }
.us-empty { color: var(--us-muted); padding: 2rem .8rem; }

.us-drawer { position: fixed; inset: 0; z-index: 1000; background: rgba(0,0,0,.4); display: flex; justify-content: flex-end; }
.us-drawer[hidden] { display: none; }
.us-drawer-panel { width: min(92rem, 100%); height: 100%; display: flex; flex-direction: column; background: var(--us-bg); color: var(--us-fg); box-shadow: -6px 0 24px rgba(0,0,0,.25); }
.us-drawer-head { display: flex; align-items: center; gap: 1rem; padding: 1.2rem 1.6rem; border-bottom: 1px solid var(--us-line); }
.us-drawer-head h2 { margin: 0; flex: 1; font-size: 1.6rem; overflow-wrap: anywhere; }
.us-icon { margin: 0; min-width: 0; padding: .4rem .8rem; }
.us-drawer-body { flex: 1; min-height: 0; display: flex; flex-direction: column; gap: 1rem; padding: 1.2rem 1.6rem; overflow: auto; }
.us-tabs { display: flex; gap: .4rem; }
.us-tabs button { margin: 0; }
.us-tabs button[aria-selected="true"] { border-color: var(--us-accent); color: var(--us-accent); }
.us-logbar { display: flex; flex-wrap: wrap; gap: .6rem; align-items: center; }
.us-logbar select, .us-logbar button { margin: 0; }
.us-log { flex: 1; min-height: 20rem; margin: 0; padding: 1rem; overflow: auto; white-space: pre; font: 1.2rem/1.45 monospace;
  background: #111; color: #ddd; border-radius: 4px; }
.us-log.wrap { white-space: pre-wrap; overflow-wrap: anywhere; }
.us-hist { width: 100%; border-collapse: collapse; font-size: 1.2rem; }
.us-hist th, .us-hist td { text-align: left; padding: .5rem .6rem; border-bottom: 1px solid var(--us-line); }
.us-editor { flex: 1; min-height: 30rem; border: 1px solid var(--us-line); border-radius: 4px; }
.us-vars { display: flex; flex-wrap: wrap; gap: .6rem 1.6rem; }
.us-vars label { display: inline-flex; gap: .5rem; align-items: center; }
.us-form { display: grid; grid-template-columns: max-content minmax(0, 1fr); gap: 1rem 1.6rem; align-items: center; max-width: 60rem; }
.us-form input, .us-form select { margin: 0; width: 100%; max-width: 24rem; }
.us-form .hint { grid-column: 2; margin: -.6rem 0 0; color: var(--us-muted); font-size: 1.15rem; }
.us-foot { display: flex; gap: .8rem; justify-content: flex-end; }
.us-foot button { margin: 0; }
.us-toast { position: fixed; right: 2rem; bottom: 4rem; z-index: 1100; max-width: 40rem; padding: 1rem 1.4rem; border-radius: 6px; background: #222; color: #fff; box-shadow: 0 6px 20px rgba(0,0,0,.3); }
.us-toast.err { background: var(--us-fail); }
.us :focus-visible { outline: 2px solid var(--us-accent); outline-offset: 2px; }

@media (max-width: 800px) {
  .us-row { grid-template-columns: minmax(0, 1fr); gap: .8rem; }
  .us-actions { justify-content: flex-start; }
  .us-sched select, .us-sched input { max-width: none; }
  .us-form { grid-template-columns: minmax(0, 1fr); }
  .us-form .hint { grid-column: 1; }
  .us-form input, .us-form select { max-width: none; }
}
```

- [ ] **Step 3: Write app.js (core)**

`P/ui/app.js`:
```js
/* User Scripts UI (VozDeOuro fork). Vanilla JS, talks to api.php. */
'use strict';
const US = (() => {
  const root = document.getElementById('us-app');
  const $ = (sel, el = root) => el.querySelector(sel);
  const FREQ = [
    ['disabled', 'Schedule disabled'], ['hourly', 'Hourly'], ['daily', 'Daily'], ['weekly', 'Weekly'], ['monthly', 'Monthly'],
    ['start', 'At startup of array'], ['boot', 'At first array start only'], ['stop', 'At stopping of array'], ['custom', 'Custom'],
  ];
  const state = { scripts: [], filter: 'all', query: '', pending: {}, menuOpen: null };

  const esc = (s) => String(s ?? '').replace(/[&<>"']/g, (c) => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]));
  const fmtDur = (s) => s < 60 ? `${s}s` : s < 3600 ? `${Math.floor(s / 60)}m ${s % 60}s` : `${Math.floor(s / 3600)}h ${Math.floor(s % 3600 / 60)}m`;
  const fmtAgo = (ts) => { const d = Math.max(0, Date.now() / 1000 - ts); return d < 60 ? 'just now' : d < 3600 ? `${Math.floor(d / 60)} min ago` : d < 86400 ? `${Math.floor(d / 3600)} h ago` : `${Math.floor(d / 86400)} d ago`; };
  const fmtWhen = (ts) => { const d = new Date(ts * 1000), now = new Date(); const t = d.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
    return d.toDateString() === now.toDateString() ? `today ${t}` : `${d.toLocaleDateString([], { weekday: 'short', day: 'numeric', month: 'short' })} ${t}`; };

  async function api(action, data = {}) {
    const r = await fetch('/plugins/user.scripts/api.php', {
      method: 'POST', headers: { 'Content-Type': 'application/json', 'X-CSRF-Token': root.dataset.csrf },
      body: JSON.stringify({ action, ...data }),
    });
    let j; try { j = await r.json(); } catch { j = { ok: false, error: `HTTP ${r.status}` }; }
    if (!j.ok) { const e = new Error(j.error || 'request failed'); e.status = r.status; throw e; }
    return j;
  }

  let toastTimer;
  function toast(msg, kind = '') {
    const t = $('#us-toast'); t.textContent = msg; t.className = `us-toast ${kind}`; t.hidden = false;
    clearTimeout(toastTimer); toastTimer = setTimeout(() => { t.hidden = true; }, kind === 'err' ? 6000 : 3000);
  }

  function status(s) { return s.running ? 'running' : (s.last?.status === 'skipped' ? (s.health.health === 'failing' ? 'failed' : 'ok') : s.last?.status || ''); }

  function visible() {
    const q = state.query.toLowerCase();
    return state.scripts.filter((s) => {
      if (q && !`${s.name} ${s.description} ${s.id}`.toLowerCase().includes(q)) return false;
      const sched = state.pending[s.id]?.frequency ?? s.schedule.frequency;
      if (state.filter === 'scheduled') return sched !== 'disabled';
      if (state.filter === 'running') return !!s.running;
      if (state.filter === 'failing') return s.health.health === 'failing' || ['failed', 'timeout'].includes(s.last?.status);
      return true;
    });
  }

  function schedCell(s) {
    const p = { ...s.schedule, ...(state.pending[s.id] || {}) };
    const opts = FREQ.map(([v, l]) => `<option value="${v}"${v === p.frequency ? ' selected' : ''}>${l}</option>`).join('');
    const custom = p.frequency === 'custom'
      ? `<input type="text" class="us-cron" value="${esc(p.custom)}" placeholder="*/5 * * * *" aria-label="Custom cron for ${esc(s.name)}" spellcheck="false">` : '';
    let meta = '';
    if (state.pending[s.id]) meta = `<span class="us-meta" data-meta>${esc(state.pending[s.id]._text || 'unsaved, press Apply')}</span>`;
    else if (s.cron) meta = `<span class="us-meta">${esc(s.cron)}${s.next ? ` · next ${esc(fmtWhen(s.next))}` : ''}</span>`;
    return `<select class="us-freq" aria-label="Schedule for ${esc(s.name)}">${opts}</select>${custom}${meta}`;
  }

  function lastCell(s) {
    if (s.running) return `<span class="us-pill running">running</span> <span class="us-meta">since ${esc(fmtWhen(s.running.since))}</span>`;
    const l = s.last;
    if (!l) return '<span class="us-meta">never run</span>';
    const rep = l.repeat ? ` ×${Number(l.repeat).toLocaleString()}` : '';
    const dur = l.status === 'skipped' ? '' : `, ${fmtDur(l.dur ?? 0)}`;
    const badge = s.health.health === 'failing' && s.health.firstFail ? `<span class="us-badge">failing since ${esc(fmtWhen(s.health.firstFail))}</span>` : '';
    return `<span class="us-pill ${esc(l.status)}">${esc(l.status)}${rep}</span> <span class="us-meta">${esc(fmtAgo(l.start))}${dur}</span>${badge}`;
  }

  function actionsCell(s) {
    const v = s.vars || {};
    const dis = s.missing ? ' disabled' : '';
    const run = s.running
      ? `<button type="button" data-act="abort">Abort</button>`
      : `${v.backgroundOnly ? '' : `<button type="button" data-act="run"${dis}>Run</button>`}${v.foregroundOnly ? '' : `<button type="button" data-act="background"${dis}>Background</button>`}`;
    const menu = state.menuOpen === s.id ? `<ul class="us-menu-list" role="menu">
      <li><button type="button" role="menuitem" data-act="edit">Edit script</button></li>
      <li><button type="button" role="menuitem" data-act="options">Run options</button></li>
      <li><button type="button" role="menuitem" data-act="rename">Rename</button></li>
      <li><button type="button" role="menuitem" data-act="describe">Edit description</button></li>
      <li><button type="button" role="menuitem" data-act="delete" class="danger"${s.running ? ' disabled' : ''}>Delete</button></li></ul>` : '';
    return `${run}<button type="button" data-act="log">Log</button><span class="us-menu"><button type="button" data-act="menu" aria-haspopup="menu" aria-expanded="${state.menuOpen === s.id}" aria-label="More actions for ${esc(s.name)}">⋯</button>${menu}</span>`;
  }

  function render() {
    const list = $('#us-list');
    const focus = document.activeElement?.closest?.('.us-row') ? { id: document.activeElement.closest('.us-row').dataset.id, cls: document.activeElement.className, act: document.activeElement.dataset.act } : null;
    const rows = visible();
    list.innerHTML = rows.map((s) => `<div class="us-row" data-id="${esc(s.id)}">
      <div><div class="us-name"><span class="us-dot ${esc(status(s))}" aria-hidden="true"></span><h3>${esc(s.name)}</h3></div>
        ${s.description ? `<p class="us-desc">${esc(s.description)}</p>` : ''}${s.missing ? '<p class="us-desc us-meta err">script file missing</p>' : ''}</div>
      <div class="us-sched">${schedCell(s)}</div>
      <div class="us-last">${lastCell(s)}</div>
      <div class="us-actions">${actionsCell(s)}</div></div>`).join('');
    $('#us-empty').hidden = rows.length > 0;
    $('#us-apply').disabled = Object.keys(state.pending).length === 0 || Object.values(state.pending).some((p) => p._invalid);
    if (focus) {
      const row = list.querySelector(`.us-row[data-id="${CSS.escape(focus.id)}"]`);
      const el = row && (focus.act ? row.querySelector(`[data-act="${focus.act}"]`) : row.querySelector(`.${focus.cls.split(' ')[0]}`));
      if (el) el.focus();
    }
  }

  let timer;
  async function refresh() {
    try { const r = await api('list'); state.scripts = r.scripts; if (!editingInline()) render(); }
    catch (e) { toast(`Could not load scripts: ${e.message}`, 'err'); }
  }
  const editingInline = () => document.activeElement?.classList?.contains('us-cron') || !!state.menuOpen;
  function poll() { clearTimeout(timer); if (!document.hidden) timer = setTimeout(async () => { await refresh(); poll(); }, 5000); }

  const byId = (id) => state.scripts.find((s) => s.id === id);

  async function act(id, a) {
    const s = byId(id);
    try {
      if (a === 'menu') { state.menuOpen = state.menuOpen === id ? null : id; render(); return; }
      state.menuOpen = null;
      if (a === 'run') { await api('run', { id }); toast(`${s.name} started`); US.openLog(id); }
      else if (a === 'background') { await api('background', { id }); toast(`${s.name} started in the background`); }
      else if (a === 'abort') { await api('abort', { id }); toast(`${s.name} aborted`); }
      else if (a === 'log') US.openLog(id);
      else if (a === 'edit') US.openEdit(id);
      else if (a === 'options') US.openOptions(id);
      else if (a === 'rename') { const n = prompt('New name', s.name); if (n !== null && n.trim()) { await api('rename', { id, name: n }); toast('Renamed'); } }
      else if (a === 'describe') { const d = prompt('Description', s.description); if (d !== null) { await api('describe', { id, description: d }); toast('Description saved'); } }
      else if (a === 'delete') { if (confirm(`Delete "${s.name}"? Its script file and logs are removed.`)) { await api('delete', { id }); toast(`${s.name} deleted`); } }
      await refresh();
    } catch (e) { toast(e.message, 'err'); await refresh(); }
  }

  let cronCheck;
  function stage(id, fields) {
    const s = byId(id);
    const p = { ...(state.pending[id] || {}), ...fields };
    const same = p.frequency === s.schedule.frequency && (p.frequency !== 'custom' || (p.custom ?? s.schedule.custom) === s.schedule.custom);
    if (same) delete state.pending[id]; else state.pending[id] = { custom: s.schedule.custom, ...p };
  }

  root.addEventListener('click', (e) => {
    const b = e.target.closest('button[data-act]');
    if (b) { act(b.closest('.us-row').dataset.id, b.dataset.act); return; }
    if (state.menuOpen && !e.target.closest('.us-menu')) { state.menuOpen = null; render(); }
  });
  root.addEventListener('change', (e) => {
    if (!e.target.classList.contains('us-freq')) return;
    const id = e.target.closest('.us-row').dataset.id;
    stage(id, { frequency: e.target.value, _invalid: false, _text: '' });
    if (e.target.value === 'custom') {
      const p = state.pending[id];
      if (p && !p.custom) { p._invalid = true; p._text = 'enter a cron expression'; }
    }
    render();
    if (e.target.value === 'custom') root.querySelector(`.us-row[data-id="${CSS.escape(id)}"] .us-cron`)?.focus();
  });
  root.addEventListener('input', (e) => {
    if (!e.target.classList.contains('us-cron')) return;
    const input = e.target, id = input.closest('.us-row').dataset.id, expr = input.value.trim();
    stage(id, { frequency: 'custom', custom: expr });
    const p = state.pending[id];
    clearTimeout(cronCheck);
    cronCheck = setTimeout(async () => {
      const r = await api('cron.check', { expr }).catch(() => ({ valid: false }));
      const meta = input.parentElement.querySelector('[data-meta]') || input.insertAdjacentElement('afterend', Object.assign(document.createElement('span'), { className: 'us-meta' }));
      meta.setAttribute('data-meta', '');
      if (p) { p._invalid = !r.valid; p._text = r.valid ? `${r.text}${r.next ? ` · next ${fmtWhen(r.next)}` : ''} (unsaved)` : 'invalid cron expression'; }
      input.setAttribute('aria-invalid', String(!r.valid));
      meta.className = `us-meta${r.valid ? '' : ' err'}`;
      meta.textContent = p ? p._text : (r.valid ? r.text : 'invalid cron expression');
      $('#us-apply').disabled = Object.keys(state.pending).length === 0 || Object.values(state.pending).some((x) => x._invalid);
    }, 300);
  });

  $('#us-apply').addEventListener('click', async () => {
    const changes = Object.fromEntries(Object.entries(state.pending).map(([id, p]) => [id, { frequency: p.frequency, custom: p.custom ?? '' }]));
    try { await api('apply', { changes }); state.pending = {}; toast('Schedule applied'); await refresh(); }
    catch (e) { toast(e.message, 'err'); }
  });
  $('#us-add').addEventListener('click', async () => {
    const name = prompt('Name for the new script');
    if (!name || !name.trim()) return;
    try { const r = await api('create', { name }); await refresh(); US.openEdit(r.id); } catch (e) { toast(e.message, 'err'); }
  });
  $('#us-settings').addEventListener('click', () => US.openSettings());
  $('#us-search').addEventListener('input', (e) => { state.query = e.target.value; render(); });
  root.querySelectorAll('.us-chip').forEach((c) => c.addEventListener('click', () => {
    state.filter = c.dataset.filter;
    root.querySelectorAll('.us-chip').forEach((x) => x.setAttribute('aria-pressed', String(x === c)));
    render();
  }));
  document.addEventListener('keydown', (e) => {
    if (e.key === '/' && !e.target.closest('input, textarea, select, .ace_editor')) { e.preventDefault(); $('#us-search').focus(); }
    if (e.key === 'Escape' && state.menuOpen) { state.menuOpen = null; render(); }
  });
  document.addEventListener('visibilitychange', () => { if (!document.hidden) { refresh(); poll(); } });
  window.addEventListener('beforeunload', (e) => { if (Object.keys(state.pending).length) { e.preventDefault(); e.returnValue = ''; } });

  refresh().then(poll);
  return { api, state, render, refresh, toast, esc, fmtAgo, fmtDur, fmtWhen, byId, $, openLog() {}, openEdit() {}, openOptions() {}, openSettings() {} };
})();
```

- [ ] **Step 4: Deploy the files to Tower for a visual check (no plugin install yet)**

Before the plugin install in Task 12, test against a copy so the live plugin is untouched:
```bash
P=source/user.scripts/usr/local/emhttp/plugins/user.scripts
ssh tower 'cp -a /boot/config/plugins/user.scripts /boot/config/plugins/user.scripts-backup-20261010 && cp -a /usr/local/emhttp/plugins/user.scripts /tmp/user.scripts.plugin-orig'
rsync -a --delete --exclude javascript $P/ tower:/usr/local/emhttp/plugins/user.scripts/
ssh tower 'chmod +x /usr/local/emhttp/plugins/user.scripts/*.php /usr/local/emhttp/plugins/user.scripts/*.sh /usr/local/emhttp/plugins/user.scripts/event/* /usr/local/emhttp/plugins/user.scripts/cron/*'
```
(The live cron still points at `startCustom.php`, which is now the shim, so scheduled scripts keep running.)

- [ ] **Step 5: Screenshot desktop + mobile**

```bash
cd /tmp/claude-1000/-home-lucas-Desktop-things/e68f0603-e34e-45d5-8762-950a549c1178/scratchpad/pw
node docker-shot.js http://192.168.0.10/Settings/Userscripts us-new.png 1600
node docker-shot.js http://192.168.0.10/Settings/Userscripts us-new-m.png 390
```
Expected: every row shows schedule, last run and actions inside the viewport at both widths; console free of errors (`node us-err.js` from earlier session prints no `PAGEERROR`).

- [ ] **Step 6: Commit**

```bash
git add -A && git commit -m "feat: responsive UI shell, script list, filters, schedule staging and polling

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: UI drawers (log + history, editor, run options)

**Files:**
- Modify: `P/ui/app.js` (append after the `US` IIFE)

**Interfaces:**
- Consumes: `US.api`, `US.byId`, `US.esc`, `US.fmtWhen`, `US.fmtDur`, `US.toast`, `US.refresh`, `US.$`; API `log`, `history`, `clearLog`, `get`, `save`, `schedule.set`; ACE global `ace`.
- Produces: `US.openLog(id)`, `US.openEdit(id)`, `US.openOptions(id)`, `US.openSettings()`, `US.closeDrawer()`. Consumes API `settings.get` / `settings.set` too.

- [ ] **Step 1: Append drawer code**

Append to `P/ui/app.js`:
```js
/* Drawers: log/history, editor, run options. */
(() => {
  const { api, esc, fmtWhen, fmtDur, toast, $ } = US;
  const drawer = $('#us-drawer'), title = $('#us-drawer-title'), body = $('#us-drawer-body');
  let cleanup = null, lastFocus = null, dirty = () => false;

  function open(t, html, onClose) {
    if (cleanup) cleanup();
    lastFocus = document.activeElement;
    title.textContent = t; body.innerHTML = html; drawer.hidden = false;
    cleanup = onClose || null;
    (body.querySelector('[autofocus]') || $('#us-drawer-close')).focus();
  }
  function close() {
    if (dirty() && !confirm('Discard unsaved changes?')) return;
    if (cleanup) cleanup();
    cleanup = null; dirty = () => false; drawer.hidden = true; body.innerHTML = '';
    lastFocus?.focus?.();
  }
  $('#us-drawer-close').addEventListener('click', close);
  drawer.addEventListener('click', (e) => { if (e.target === drawer) close(); });
  document.addEventListener('keydown', (e) => {
    if (drawer.hidden) return;
    if (e.key === 'Escape' && !e.target.closest('.ace_search')) close();
    if (e.key === 'Tab') { // keep focus inside the drawer
      const f = [...drawer.querySelectorAll('button, input, select, textarea, [tabindex]:not([tabindex="-1"])')].filter((x) => !x.disabled && x.offsetParent);
      if (!f.length) return;
      if (e.shiftKey && document.activeElement === f[0]) { e.preventDefault(); f[f.length - 1].focus(); }
      else if (!e.shiftKey && document.activeElement === f[f.length - 1]) { e.preventDefault(); f[0].focus(); }
    }
  });

  US.openLog = (id) => {
    const s = US.byId(id);
    open(`Log: ${s.name}`, `
      <div class="us-tabs" role="tablist"><button type="button" role="tab" aria-selected="true" data-tab="log">Log</button><button type="button" role="tab" aria-selected="false" data-tab="hist">History</button></div>
      <div data-pane="log" style="display:contents">
        <div class="us-logbar">
          <label>Run <select id="us-which"><option value="0">Current / latest</option>${[1, 2, 3, 4, 5].map((i) => `<option value="${i}">${i} run${i > 1 ? 's' : ''} before</option>`).join('')}</select></label>
          <label><input type="checkbox" id="us-wrap"> Wrap lines</label>
          <button type="button" id="us-dl">Download</button><button type="button" id="us-clear">Clear logs</button>
          <span class="us-meta" id="us-logstate"></span>
        </div>
        <pre class="us-log" id="us-log" tabindex="0" aria-label="Script output"></pre>
      </div>
      <div data-pane="hist" hidden><table class="us-hist"><thead><tr><th>Started</th><th>Status</th><th>Exit</th><th>Duration</th><th>Trigger</th></tr></thead><tbody id="us-hist"></tbody></table></div>`, () => clearTimeout(tm));
    const pre = $('#us-log'), which = $('#us-which');
    let offset = 0, tm, size = -1;
    async function tick(reset) {
      if (reset) { offset = 0; pre.textContent = ''; size = -1; }
      try {
        const r = await api('log', { id, which: +which.value, offset });
        if (r.size < size) { pre.textContent = ''; offset = 0; return tick(false); } // rotated: new run started
        const atBottom = pre.scrollTop + pre.clientHeight >= pre.scrollHeight - 20;
        pre.textContent += r.text; offset = r.size; size = r.size;
        if (atBottom || reset) pre.scrollTop = pre.scrollHeight;
        $('#us-logstate').textContent = r.running && which.value === '0' ? 'running, updating live' : (r.size ? '' : 'no log yet');
        if (r.running && which.value === '0') tm = setTimeout(() => tick(false), 2000);
        else if (which.value === '0') tm = setTimeout(() => tick(false), 5000);
      } catch (e) { toast(e.message, 'err'); }
    }
    which.addEventListener('change', () => { clearTimeout(tm); tick(true); });
    $('#us-wrap').addEventListener('change', (e) => pre.classList.toggle('wrap', e.target.checked));
    $('#us-dl').addEventListener('click', () => {
      const a = Object.assign(document.createElement('a'), { href: URL.createObjectURL(new Blob([pre.textContent], { type: 'text/plain' })), download: `${id}-log${which.value !== '0' ? '.' + which.value : ''}.txt` });
      a.click(); setTimeout(() => URL.revokeObjectURL(a.href), 1000);
    });
    $('#us-clear').addEventListener('click', async () => { if (!confirm('Delete all saved logs for this script?')) return; await api('clearLog', { id }); tick(true); });
    body.querySelectorAll('[data-tab]').forEach((t) => t.addEventListener('click', async () => {
      body.querySelectorAll('[data-tab]').forEach((x) => x.setAttribute('aria-selected', String(x === t)));
      body.querySelector('[data-pane="log"]').hidden = t.dataset.tab !== 'log';
      body.querySelector('[data-pane="log"]').style.display = t.dataset.tab === 'log' ? 'contents' : 'none';
      body.querySelector('[data-pane="hist"]').hidden = t.dataset.tab !== 'hist';
      if (t.dataset.tab === 'hist') {
        const r = await api('history', { id, limit: 200 });
        $('#us-hist').innerHTML = r.runs.map((h) => `<tr><td>${esc(fmtWhen(h.start))}${h.repeat ? `<br><span class="us-meta">×${Number(h.repeat).toLocaleString()} since ${esc(fmtWhen(h.first))}</span>` : ''}</td>
          <td><span class="us-pill ${esc(h.status)}">${esc(h.status)}</span></td><td>${esc(h.exit)}</td>
          <td>${h.status === 'skipped' ? '' : esc(fmtDur(h.dur ?? 0))}${h.repeat ? `<br><span class="us-meta">min ${esc(fmtDur(h.dmin))} · avg ${esc(fmtDur(h.davg))} · max ${esc(fmtDur(h.dmax))}</span>` : ''}</td>
          <td>${esc(h.trigger)}</td></tr>`).join('') || '<tr><td colspan="5" class="us-meta">No runs recorded yet.</td></tr>';
      }
    }));
    tick(true);
  };

  const VARS = [['arrayStarted', 'Only run when the array is started'], ['noParity', 'Skip while a parity check runs'], ['foregroundOnly', 'Foreground only'],
    ['backgroundOnly', 'Background only'], ['clearLog', 'Do not keep previous logs']];

  function setHeaderVar(text, key, on) {
    const lines = text.split('\n');
    const start = lines[0]?.startsWith('#!') ? 1 : 0;
    let end = start; while (end < lines.length && lines[end].startsWith('#')) end++;
    const idx = lines.slice(start, end).findIndex((l) => new RegExp(`^#\\s*${key}\\s*=`).test(l));
    if (idx >= 0) { if (on) lines[start + idx] = `#${key}=true`; else lines.splice(start + idx, 1); }
    else if (on) lines.splice(end, 0, `#${key}=true`);
    return lines.join('\n');
  }

  US.openEdit = async (id) => {
    const s = US.byId(id);
    let text;
    try { text = (await api('get', { id })).body; } catch (e) { toast(e.message, 'err'); return; }
    open(`Edit: ${s.name}`, `
      <div class="us-vars">${VARS.map(([k, l]) => `<label><input type="checkbox" data-var="${k}"${s.vars?.[k] ? ' checked' : ''}> ${l}</label>`).join('')}</div>
      <div id="us-ace" class="us-editor"></div>
      <p class="us-meta">Saved to /boot/config/plugins/user.scripts/scripts/${esc(id)}/script. Ctrl+S saves.</p>
      <div class="us-foot"><button type="button" id="us-cancel">Close</button><button type="button" id="us-save">Save</button></div>`, () => ed?.destroy());
    const dark = /black|gray/.test(US.$('#us-app').dataset.theme);
    const ed = window.ace ? ace.edit('us-ace', { mode: 'ace/mode/' + (/^#!.*\b(php)\b/.test(text) ? 'php' : /^#!.*python/.test(text) ? 'python' : 'sh'),
      theme: dark ? 'ace/theme/tomorrow_night' : 'ace/theme/tomorrow', value: text, fontSize: 13, showPrintMargin: false, tabSize: 2, useSoftTabs: true }) : null;
    let saved = text;
    const get = () => ed ? ed.getValue() : text;
    dirty = () => get() !== saved;
    ed?.focus();
    body.querySelectorAll('[data-var]').forEach((c) => c.addEventListener('change', () => {
      const v = setHeaderVar(get(), c.dataset.var, c.checked); const pos = ed.getCursorPosition(); ed.setValue(v, -1); ed.moveCursorToPosition(pos);
    }));
    const save = async () => {
      try { await api('save', { id, body: get() }); saved = get(); toast('Script saved'); US.refresh(); } catch (e) { toast(e.message, 'err'); }
    };
    ed?.commands.addCommand({ name: 'save', bindKey: { win: 'Ctrl-S', mac: 'Command-S' }, exec: save });
    $('#us-save').addEventListener('click', save);
    $('#us-cancel').addEventListener('click', close);
  };

  US.openOptions = (id) => {
    const s = US.byId(id), o = s.schedule;
    const sel = (name, opts) => `<select name="${name}" id="us-o-${name}">${opts.map(([v, l]) => `<option value="${v}"${String(o[name]) === String(v) ? ' selected' : ''}>${l}</option>`).join('')}</select>`;
    open(`Run options: ${s.name}`, `
      <form class="us-form" id="us-opts" novalidate>
        <label for="us-o-notify">Notifications</label>${sel('notify', [['off', 'Off'], ['failure', 'On failure'], ['always', 'Every run']])}
        <label for="us-o-failAfter">Alert after</label><input type="number" min="1" max="1000" name="failAfter" id="us-o-failAfter" value="${esc(o.failAfter)}">
        <p class="hint">consecutive failures. Use 3 for scripts that run every minute so one blip stays quiet.</p>
        <label for="us-o-remindHours">Remind every</label><input type="number" min="0" max="720" name="remindHours" id="us-o-remindHours" value="${esc(o.remindHours)}">
        <p class="hint">hours while it keeps failing. 0 never reminds. Recovery is always announced after an alert.</p>
        <label for="us-o-overlap">If still running</label>${sel('overlap', [['skip', 'Skip the new run'], ['queue', 'Queue one run'], ['parallel', 'Run anyway (parallel)']])}
        <label for="us-o-hangAfter">Stuck alert after</label><input type="number" min="1" max="1000" name="hangAfter" id="us-o-hangAfter" value="${esc(o.hangAfter)}">
        <p class="hint">skipped runs in a row.</p>
        <label for="us-o-timeout">Timeout</label><input type="number" min="0" max="604800" name="timeout" id="us-o-timeout" value="${esc(o.timeout)}">
        <p class="hint">seconds. 0 means no limit. The whole process group is stopped when it is reached.</p>
      </form>
      <div class="us-foot"><button type="button" id="us-cancel">Cancel</button><button type="button" id="us-save">Save</button></div>`);
    const form = $('#us-opts');
    $('#us-o-notify').focus();
    const initial = new URLSearchParams(new FormData(form)).toString();
    dirty = () => new URLSearchParams(new FormData(form)).toString() !== initial;
    $('#us-cancel').addEventListener('click', close);
    $('#us-save').addEventListener('click', async () => {
      const f = Object.fromEntries(new FormData(form));
      for (const k of ['failAfter', 'remindHours', 'hangAfter', 'timeout']) f[k] = Number(f[k]);
      try { await api('schedule.set', { id, fields: f }); dirty = () => false; toast('Run options saved'); close(); US.refresh(); }
      catch (e) { toast(e.message, 'err'); }
    });
  };

  US.openSettings = async () => {
    let st;
    try { st = (await api('settings.get')).settings; } catch (e) { toast(e.message, 'err'); return; }
    open('User Scripts settings', `
      <form class="us-form" id="us-set" novalidate>
        <label for="us-s-dir">History folder</label><input type="text" name="historyDir" id="us-s-dir" value="${esc(st.historyDir)}" spellcheck="false" style="max-width:none">
        <p class="hint">Default is on the flash drive. A pool path such as /mnt/user/system/user.scripts saves flash writes; while the array is stopped, history stays in RAM and is saved once the path is back.</p>
        <label for="us-s-sync">Save history every</label><select name="syncMinutes" id="us-s-sync">${[15, 30, 60].map((m) => `<option value="${m}"${m === st.syncMinutes ? ' selected' : ''}>${m} minutes</option>`).join('')}</select>
        <p class="hint">Failures and recoveries are always saved immediately.</p>
        <label for="us-s-cap">Keep</label><input type="number" min="10" max="5000" name="historyCap" id="us-s-cap" value="${esc(st.historyCap)}">
        <p class="hint">history entries per script. Repeated identical results count as one entry.</p>
      </form>
      <div class="us-foot"><button type="button" id="us-cancel">Cancel</button><button type="button" id="us-save">Save</button></div>`);
    const form = $('#us-set');
    $('#us-s-dir').focus();
    const initial = new URLSearchParams(new FormData(form)).toString();
    dirty = () => new URLSearchParams(new FormData(form)).toString() !== initial;
    $('#us-cancel').addEventListener('click', close);
    $('#us-save').addEventListener('click', async () => {
      const f = Object.fromEntries(new FormData(form));
      f.syncMinutes = Number(f.syncMinutes); f.historyCap = Number(f.historyCap);
      try { await api('settings.set', { settings: f }); dirty = () => false; toast('Settings saved'); close(); }
      catch (e) { toast(e.message, 'err'); }
    });
  };

  US.closeDrawer = close;
})();
```

- [ ] **Step 2: Deploy and exercise on Tower**

```bash
rsync -a --exclude javascript source/user.scripts/usr/local/emhttp/plugins/user.scripts/ tower:/usr/local/emhttp/plugins/user.scripts/
```
Create a harmless test script through the UI ("Add script" → `zz-ui-test`, body `#!/bin/bash\nfor i in 1 2 3; do echo tick $i; sleep 1; done`), Run it: log drawer tails live, status goes running → ok, History tab shows the run. Open Run options, set Notifications "Every run", Save, Background-run it: a "zz-ui-test finished in 3s" notification appears in Unraid and on Telegram. Edit script: toggle "Only run when the array is started", header line appears, Ctrl+S saves. Delete the test script. Open Settings, change "Save history every" to 30 minutes, Save, and confirm `ssh tower grep sync-history /boot/config/plugins/user.scripts/customSchedule.cron` shows `*/30`; set it back to 60.

- [ ] **Step 3: Screenshot desktop + mobile with a drawer open**

```bash
cd /tmp/claude-1000/-home-lucas-Desktop-things/e68f0603-e34e-45d5-8762-950a549c1178/scratchpad/pw
node docker-shot.js http://192.168.0.10/Settings/Userscripts us-new.png 1600
node docker-shot.js http://192.168.0.10/Settings/Userscripts us-new-m.png 390
```
Fix every layout defect found in one batch, re-shoot once.

- [ ] **Step 4: Commit**

```bash
git add -A && git commit -m "feat: log/history drawer, ACE editor with header toggles, run options dialog

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: Packaging, .plg, CI release, README

**Files:**
- Modify (rewrite): `plugins/user.scripts.plg`, `pkg_build.sh`, `source/user.scripts/pkg_build.sh`, `P/README.md`
- Create: `README.md`, `.github/workflows/ci.yml`, `.github/workflows/release.yml`, `bin/release`
- Delete: unused ACE files under `P/javascript/ace/` (keep `ace.js`, `mode-sh.js`, `mode-php.js`, `mode-python.js`, `theme-tomorrow.js`, `theme-tomorrow_night.js`, `ext-searchbox.js`), `archive/*.txz` from upstream

**Interfaces:**
- Consumes: built tree under `source/user.scripts/`
- Produces: release asset `user.scripts-<version>-x86_64-1.txz` on GitHub release `v<version>`; `.plg` at `https://raw.githubusercontent.com/VozDeOuro/user.scripts/master/plugins/user.scripts.plg`.

- [ ] **Step 1: Trim ACE and old archives**

```bash
cd source/user.scripts/usr/local/emhttp/plugins/user.scripts/javascript/ace
ls | grep -vE '^(ace\.js|mode-(sh|php|python)\.js|theme-tomorrow(_night)?\.js|ext-searchbox\.js)$' | xargs git rm -q
cd ~/Desktop/things/user.scripts && git rm -q archive/*.txz 2>/dev/null; mkdir -p archive && touch archive/.gitkeep
```

- [ ] **Step 2: Rewrite build script**

`source/user.scripts/pkg_build.sh`:
```bash
#!/bin/bash
# Builds archive/user.scripts-<version>-x86_64-1.txz. Needs makepkg (Slackware) or falls back to tar+xz,
# which produces the same layout upgradepkg accepts.
set -euo pipefail
DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
version=${1:?usage: pkg_build.sh <version>}
archive="$(dirname "$(dirname "$DIR")")/archive"
out="$archive/user.scripts-$version-x86_64-1.txz"
tmp=$(mktemp -d); trap 'rm -rf "$tmp"' EXIT
mkdir -p "$archive"
(cd "$DIR" && find usr -type f -print0 | xargs -0 cp --parents -t "$tmp")
find "$tmp" -type d -exec chmod 0755 {} +
find "$tmp" -type f -exec chmod 0644 {} +
find "$tmp" \( -name '*.sh' -o -name 'run.php' -o -name 'start*.php' \) -exec chmod 0755 {} +
find "$tmp"/usr/local/emhttp/plugins/user.scripts/event -type f -exec chmod 0755 {} +
if command -v makepkg >/dev/null; then
  (cd "$tmp" && makepkg -l y -c y "$out" >/dev/null)
else
  (cd "$tmp" && tar --owner=0 --group=0 -cJf "$out" usr)
fi
md5sum "$out" | cut -d' ' -f1
```

`pkg_build.sh` (repo root):
```bash
#!/bin/bash
exec "$(dirname "$0")/source/user.scripts/pkg_build.sh" "$@"
```

- [ ] **Step 3: Rewrite the .plg**

`plugins/user.scripts.plg`:
```xml
<?xml version='1.0' standalone='yes'?>
<!DOCTYPE PLUGIN [
<!ENTITY name      "user.scripts">
<!ENTITY author    "Andrew Zawadzki, VozDeOuro">
<!ENTITY version   "2026.10.10">
<!ENTITY md5       "0000000000000000000000000000000">
<!ENTITY github    "VozDeOuro/user.scripts">
<!ENTITY pluginURL "https://raw.githubusercontent.com/&github;/master/plugins/&name;.plg">
]>
<PLUGIN name="&name;" author="&author;" version="&version;" pluginURL="&pluginURL;" min="7.0.0" support="https://github.com/&github;/issues" icon="file-text-o">
<CHANGES>
###2026.10.10
- VozDeOuro fork. Developed and tested on Unraid 7.3.3, should work on 7.0+.
- New responsive UI: works on Unraid 7.3, mobile, all themes. Search, filters, live log tail, run history.
- Scheduling fixes: custom cron lines carry their own schedule (#19, #20), no silent skips (#17), day/month names (#12, #13).
- Abort and timeout stop the whole process group, including child processes (#18).
- Per-script notifications with dedupe, reminders and recovery notices (#21, #23).
- Overlap control per script: skip, queue or parallel, with a stuck alert.
- History kept in RAM and synced to flash hourly and on status changes, to spare the USB drive.
</CHANGES>

<FILE Run="/bin/bash">
<INLINE>
# remove packages from older versions (upstream or ours)
rm -f $(ls /boot/config/plugins/&name;/&name;*.txz 2>/dev/null | grep -v '&version;')
</INLINE>
</FILE>

<FILE Name="/boot/config/plugins/&name;/&name;-&version;-x86_64-1.txz" Run="upgradepkg --install-new">
<URL>https://github.com/&github;/releases/download/v&version;/&name;-&version;-x86_64-1.txz</URL>
<MD5>&md5;</MD5>
</FILE>

<FILE Run="/bin/bash">
<INLINE>
mkdir -p /boot/config/plugins/&name;/scripts /tmp/user.scripts
rm -rf /usr/local/emhttp/plugins/&name;/scripts
if [ -f /boot/config/plugins/&name;/schedule.json ]; then
  sed -i 's#/usr/local/emhttp/plugins/user.scripts/scripts/#/boot/config/plugins/user.scripts/scripts/#g' /boot/config/plugins/&name;/schedule.json
fi
for f in hourly daily weekly monthly; do
  cp /usr/local/emhttp/plugins/&name;/cron/user.script.start.$f.sh /etc/cron.$f/
done
# seed RAM history from flash and regenerate customSchedule.cron (imports hand-added lines first)
/usr/local/emhttp/plugins/&name;/run.php --seed
echo ""
echo "----------------------------------------------------"
echo " &name; &version; installed (VozDeOuro fork)"
echo " Developed and tested on Unraid 7.3.3"
echo "----------------------------------------------------"
</INLINE>
</FILE>

<FILE Run="/bin/bash" Method="remove">
<INLINE>
/usr/local/emhttp/plugins/&name;/run.php --sync-history
for f in hourly daily weekly monthly; do rm -f /etc/cron.$f/user.script.start.$f.sh; done
removepkg &name;-&version;-x86_64-1
rm -rf /tmp/user.scripts
</INLINE>
</FILE>
</PLUGIN>
```

- [ ] **Step 4: Release tooling and CI**

`bin/release` (chmod +x):
```bash
#!/bin/bash
# bin/release [suffix] : builds today's package, stamps version+md5 into the .plg, commits, tags, pushes.
set -euo pipefail
cd "$(dirname "$0")/.."
v=$(date +%Y.%m.%d)${1:-}
md5=$(./pkg_build.sh "$v")
sed -i -E "s#(ENTITY version +\")[^\"]+#\1$v#; s#(ENTITY md5 +\")[^\"]+#\1$md5#" plugins/user.scripts.plg
grep -q "^###$v" plugins/user.scripts.plg || { echo "add a ###$v section to <CHANGES> first"; exit 1; }
git add plugins/user.scripts.plg
git commit -m "release $v

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
git tag "v$v"
git push origin master "v$v"
gh release create "v$v" "archive/user.scripts-$v-x86_64-1.txz" --title "$v" --notes "$(sed -n "/^###$v/,/^###/p" plugins/user.scripts.plg | sed '1d;/^###/d')"
```

Note on the package and the md5: the release script builds the `.txz` locally, writes its md5 into the `.plg`, and uploads that same file. CI must not rebuild it (a rebuild changes the md5).

`.github/workflows/ci.yml`:
```yaml
name: ci
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix: { php: ['8.3', '8.4'] }
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with: { php-version: '${{ matrix.php }}', tools: composer }
      - run: composer install -q
      - run: vendor/bin/phpunit
      - run: bash tests/runner.sh
      - run: bash -n source/user.scripts/usr/local/emhttp/plugins/user.scripts/*.sh
      - run: xmllint --noout --noent plugins/user.scripts.plg || (sudo apt-get -qq install -y libxml2-utils && xmllint --noout --noent plugins/user.scripts.plg)
```

`.github/workflows/release.yml`:
```yaml
name: release-check
on:
  release: { types: [published] }
jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { ref: master }
      - name: asset md5 matches the .plg
        env: { GH_TOKEN: '${{ github.token }}' }
        run: |
          v=$(grep -oP 'ENTITY version +"\K[^"]+' plugins/user.scripts.plg)
          want=$(grep -oP 'ENTITY md5 +"\K[^"]+' plugins/user.scripts.plg)
          gh release download "v$v" -p "*.txz" -D dl
          got=$(md5sum dl/*.txz | cut -d' ' -f1)
          test "$want" = "$got" || { echo "md5 mismatch: plg=$want asset=$got"; exit 1; }
```

- [ ] **Step 5: README**

`README.md`:
```markdown
# User Scripts (VozDeOuro fork)

Fork of [Squidly271/user.scripts](https://github.com/Squidly271/user.scripts) for Unraid.
Developed and tested on Unraid 7.3.3, should work on 7.0+.

Install: Plugins → Install Plugin →
`https://raw.githubusercontent.com/VozDeOuro/user.scripts/master/plugins/user.scripts.plg`

Your existing scripts and schedules keep working: the data format on the flash drive is unchanged.
To go back, install the upstream `.plg`.

## What is different

- Responsive UI with search, filters, live log tail, last run, next run and run history.
- Custom cron schedules run exactly as written; day and month names work.
- Abort and timeout stop the whole process group.
- Per-script notifications: on failure or every run, alert after N failures, reminders, recovery notices.
- Overlap control: skip, queue one, or run in parallel; "looks stuck" alert.
- History lives in RAM and is synced to flash hourly and on status changes.

## Development

    bin/test 8.3      # PHPUnit + runner integration test in Docker
    bin/release       # build, stamp the .plg, tag and publish a GitHub release
```

Replace `P/README.md` (shown in the Plugins page) with:
```markdown
**User Scripts** (VozDeOuro fork)

Run and schedule your own scripts, with run history, live logs and failure notifications.
Developed and tested on Unraid 7.3.3, should work on 7.0+.
```

- [ ] **Step 6: Build locally and validate**

Run: `./pkg_build.sh 2026.10.10 && tar -tJf archive/user.scripts-2026.10.10-x86_64-1.txz | head -20 && xmllint --noout --noent plugins/user.scripts.plg && bin/test 8.4`
Expected: an md5 printed, the archive lists `usr/local/emhttp/plugins/user.scripts/run.php` etc., the plg parses, tests pass on 8.4.

- [ ] **Step 7: Commit (no release yet)**

```bash
echo 'archive/*.txz' >> .gitignore
git add -A && git commit -m "build: package script, fork .plg (min 7.0.0), CI on PHP 8.3/8.4, release tooling, trimmed ACE

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
git push
```

---

### Task 13: Release and rollout on Tower

**Files:**
- Create: `tests/smoke/userscripts.smoke.js` (Playwright, run from the workstation)

**Interfaces:**
- Consumes: Task 12 release; the existing Playwright helper `scratchpad/pw/common.js` (`open({width,height}) → {browser,page}` with the Unraid session cookie).

- [ ] **Step 1: Write the smoke test**

`tests/smoke/userscripts.smoke.js`:
```js
// Usage: node tests/smoke/userscripts.smoke.js <path-to-pw/common.js>
// Loads the page at desktop and phone width, checks controls are on screen and there are no errors.
const { open } = require(process.argv[2]);
(async () => {
  let fail = 0;
  for (const width of [1600, 390]) {
    const { browser, page } = await open({ width, height: 1000 });
    const errs = [];
    page.on('pageerror', (e) => errs.push(String(e)));
    page.on('response', (r) => { if (r.url().includes('/plugins/user.scripts/') && r.status() >= 400) errs.push(`HTTP ${r.status()} ${r.url()}`); });
    await page.goto('http://192.168.0.10/Settings/Userscripts', { waitUntil: 'networkidle' });
    await page.waitForSelector('.us-row');
    const r = await page.evaluate(() => {
      const rows = [...document.querySelectorAll('.us-row')];
      const off = [...document.querySelectorAll('.us-row button, .us-row select')].filter((el) => { const b = el.getBoundingClientRect(); return b.right > innerWidth + 1; }).length;
      return { rows: rows.length, off, scrollX: document.documentElement.scrollWidth > innerWidth + 1 };
    });
    await page.click('.us-row button[data-act="log"]');
    await page.waitForSelector('#us-drawer:not([hidden]) #us-log');
    await page.screenshot({ path: `smoke-${width}.png`, fullPage: false });
    console.log(width, JSON.stringify(r), errs.length ? errs : 'no errors');
    if (r.rows < 1 || r.off || r.scrollX || errs.length) fail = 1;
    await browser.close();
  }
  process.exit(fail);
})();
```

- [ ] **Step 2: Restore the live plugin, then publish the release**

Tasks 10–11 rsynced files over the running plugin; put the original back so the install is a clean upgrade:
```bash
ssh tower 'rm -rf /usr/local/emhttp/plugins/user.scripts && cp -a /tmp/user.scripts.plugin-orig /usr/local/emhttp/plugins/user.scripts'
```
Add the `###2026.10.10` section check passes (already in the `.plg`), then:
```bash
bin/release
```
Expected: tag `v2026.10.10` pushed, GitHub release with the `.txz`, `release-check` workflow green.

- [ ] **Step 3: Back up and record the before state**

```bash
ssh tower 'set -e; B=/boot/config/plugins/user.scripts-backup-$(date +%Y%m%d-%H%M); cp -a /boot/config/plugins/user.scripts $B; cp /boot/config/plugins/user.scripts.plg $B/; crontab -l > $B/crontab.before; ls /boot/config/plugins/user.scripts/scripts | wc -l; echo $B'
```
Expected: `24` and the backup path.

- [ ] **Step 4: Install the fork**

```bash
ssh tower 'plugin install https://raw.githubusercontent.com/VozDeOuro/user.scripts/master/plugins/user.scripts.plg 2>&1 | tail -8'
```
Expected: "user.scripts 2026.10.10 installed (VozDeOuro fork)".

- [ ] **Step 5: Verify**

```bash
ssh tower 'grep -c "run.php --script" /boot/config/plugins/user.scripts/customSchedule.cron; crontab -l | grep -c "user.scripts/run.php"; for s in web-sites-status cf-dns-sync; do grep -F "'"'"'$s'"'"'" /boot/config/plugins/user.scripts/customSchedule.cron; done; sleep 70; tail -n1 /tmp/user.scripts/history/web-sites-status.jsonl; stat -c %Y /mnt/user/appdata/web-sites/status/status.json; date +%s; grep -c PHP /var/log/phplog'
node tests/smoke/userscripts.smoke.js /tmp/claude-1000/-home-lucas-Desktop-things/e68f0603-e34e-45d5-8762-950a549c1178/scratchpad/pw/common.js
```
Expected: 7 custom lines + sync line in crontab; both hand-added scripts present with their own cron; a `web-sites-status` history entry with status `ok`; `status.json` mtime within the last 2 minutes; no new PHP fatals; smoke test exits 0 at both widths.

- [ ] **Step 6: Configure notifications for the scripts that matter**

In the UI, Run options: `web-sites-status` and `cf-dns-sync` → On failure, alert after 3, remind 24 h; `BKP_NEW` → On failure, overlap queue, timeout 21600. Trigger one deliberate failure with a temporary script (`exit 1`, notify on failure) and confirm the Telegram alert arrives once, then a "recovered" after fixing it; delete the temporary script.

- [ ] **Step 7: Watch 24 h, then commit the smoke test**

Next day: `ssh tower 'ls -la /boot/config/plugins/user.scripts/history | head; tail -n2 /tmp/user.scripts/history/cf-dns-sync.jsonl'` shows hourly-synced files and collapsed repeats.

```bash
git add tests/smoke && git commit -m "test: Playwright smoke test for the installed plugin

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>" && git push
```

Rollback (if anything is wrong): `ssh tower 'plugin install https://raw.githubusercontent.com/Squidly271/user.scripts/master/plugins/user.scripts.plg'`, then open User Scripts and press Apply once so upstream regenerates its cron lines.
