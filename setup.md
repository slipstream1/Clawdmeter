# Clawdmeter setup session — status as of 2026-08-07

**RESOLVED as of 2026-08-07 06:47.** Everything below "What's done and
confirmed working" plus "Bug found, root-caused, and fixed" is now fully
verified end-to-end, including a real overnight shutdown/power-on cycle. See
"2026-08-07 post-reboot findings" near the end for what broke on the first
real reboot and how it was fixed — read that section if the tray icon is ever
missing again after a future reboot, since it's a different failure mode than
the `winrt` crash bug and is the more likely recurrence.

## Hardware

- Board: **Waveshare ESP32-S3-Touch-AMOLED-2.16** (the original/square 480×480 port).
  PlatformIO env: `waveshare_amoled_216`.
- Connects over USB (native ESP32-S3 USB-JTAG/serial, `/dev/ttyACM0` in WSL) for
  flashing, and over **Bluetooth LE** to a host daemon for usage data at runtime.

## Manual firmware flashing (WSL2)

Do this whenever firmware source changes — after pulling in upstream updates,
or after any local edit to `firmware/src/`.

**1. Make sure the board's USB device is visible to WSL.** If it's freshly
plugged in (or WSL was restarted), Windows needs to hand it off via
`usbipd-win` — from an **admin PowerShell**:
```powershell
usbipd list                          # find the BUSID for "Espressif USB JTAG/serial debug unit"
usbipd bind --busid <BUSID>           # one-time per device; skip if already bound
usbipd attach --wsl --busid <BUSID>   # needed every time it's unplugged/replugged or Windows reboots
```
Then confirm from WSL:
```bash
lsusb | grep -i espressif
ls -la /dev/ttyACM0
```

**2. Build:**
```bash
~/.platformio/penv/bin/pio run -d /mnt/d/code/Clawdmeter/firmware -e waveshare_amoled_216
```

**3. Flash:**
```bash
~/.platformio/penv/bin/pio run -d /mnt/d/code/Clawdmeter/firmware -e waveshare_amoled_216 -t upload --upload-port /dev/ttyACM0
```
Look for `======================== [SUCCESS] ========================` at the
end of each step — flashing finishes with `Hash of data verified.` and
`Hard resetting via RTS pin.`.

**Notes:**
- `pio` lives at `~/.platformio/penv/bin/pio` in this WSL install — it's not
  on `PATH`. Installed via the official `get-platformio.py` script, not
  `pip install`, because Debian's PEP 668 guard blocks a bare `pip install`
  here (see "if `pio` isn't on PATH" in the project's root `CLAUDE.md` for the
  fallback options on other machines).
- Both steps run noticeably slower than a typical PlatformIO build/flash
  (build ~38 min, flash ~22 min the first time) because the repo lives on
  `/mnt/d`, a 9p-mounted external drive — small-file I/O across that mount is
  slow. A rebuild from a warm `.pio` cache is much faster; only a totally
  clean rebuild (or first flash) is this slow.
- The board boots to its splash screen after flashing and only advances on a
  physical button press — that's expected, not a sign anything's wrong.
- To target a different board port, swap `-e waveshare_amoled_216` for the
  right PlatformIO env (see the board list in the project's root `CLAUDE.md`)
  and adjust the upload port if it's not `/dev/ttyACM0`.

## What's done and confirmed working

1. **USB passthrough (WSL2) for flashing** — already worked out of the box via
   `usbipd-win` (`ID 303a:1001 Espressif USB JTAG/serial debug unit`). No action
   needed here; this was pre-existing/already set up before this session.

2. **PlatformIO installed in WSL** — via the official installer script into
   `~/.platformio/penv` (pip's system-managed-environment guard blocked a plain
   `pip install`, so used `get-platformio.py`, not `pip`).

3. **Firmware built and flashed successfully**:
   ```bash
   ~/.platformio/penv/bin/pio run -d /mnt/d/code/Clawdmeter/firmware -e waveshare_amoled_216
   ~/.platformio/penv/bin/pio run -d /mnt/d/code/Clawdmeter/firmware -e waveshare_amoled_216 -t upload --upload-port /dev/ttyACM0
   ```
   Both reported `[SUCCESS]`. Flash: 1,826,912 bytes written, hash verified, board
   hard-reset via RTS. **Note:** build took ~38 min and flash ~22 min because the
   repo lives on `/mnt/d` (a 9p-mounted external drive) — small-file I/O across
   that mount is slow. Rebuilding from a warm `.pio` cache should be much faster;
   a totally clean rebuild will be slow again for the same reason.

4. **Decided AGAINST running the daemon inside WSL2.** Initially set up
   `usbipd-win` Bluetooth passthrough for the Intel adapter (busid `2-10`,
   `Intel(R) Wireless Bluetooth(R)`, `8087:0aaa`) into WSL2 — this **did** work
   (`bluetoothd` up, `hci0` up, `bluetoothctl` saw it fine). But the repo's own
   `daemon/README-windows.md` says plainly: *"Must run on real Windows — not
   WSL. BLE will not work under WSL."* Also, passing the adapter into WSL takes
   it away from Windows entirely (Windows loses all Bluetooth while it's
   attached) — bad trade for a permanent setup.
   **Action taken: the adapter was detached back to Windows** with
   `usbipd detach --busid 2-10` and has NOT been re-attached to WSL since. If a
   future session finds Windows Bluetooth missing, check
   `usbipd list` for busid `2-10` state — it should say `Not shared`.

5. **Repo location quirk**: the user symlinks their Windows `code` folder to an
   external drive for space. So the SAME physical files are reachable as:
   - `D:\code\Clawdmeter` (native Windows path)
   - `/mnt/d/code/Clawdmeter` (WSL view of the same drive — used by this Claude session)
   - `C:\Users\<winuser>\code\Clawdmeter` (symlink → `D:\code\Clawdmeter`; this is the
     path that appears in all Windows daemon logs/tracebacks, and where
     `install-windows.ps1` was actually run from)

   All three are confirmed identical (`diff` clean). Edits made via this WSL
   session at `/mnt/d/code/Clawdmeter/...` show up immediately on the Windows
   side at `C:\Users\<winuser>\code\Clawdmeter\...` — no copy step needed.

6. **Windows-native daemon installed**:
   ```powershell
   cd C:\Users\<winuser>\code\Clawdmeter   # (or D:\code\Clawdmeter — same files)
   powershell -ExecutionPolicy Bypass -File install-windows.ps1
   ```
   This created `.venv` (Python **3.14.3**, `C:\Python314\python.exe` as base
   interpreter), installed `bleak==3.0.2`, `winrt-runtime` + `winrt-windows-*`
   packages (all `3.2.1`), `pystray==0.19.5`, `Pillow==12.3.0`, `httpx==0.28.1`,
   registered the `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` autostart
   value (`Clawdmeter`), and launched the tray app.

7. **Clawdmeter device paired with Windows Bluetooth** (Settings → Bluetooth &
   devices → Add device). Bonded address: `<BONDED_MAC>`. This is a
   one-time pairing step and persists across reboots.

8. **Credentials wiring**: the user only ever runs `claude login` inside WSL
   (`/home/<wsluser>/.claude/.credentials.json`), never natively on Windows. The
   Windows daemon was pointed at that file directly via a persistent user env var:
   ```powershell
   setx CLAUDE_CREDENTIALS_PATH "\\wsl.localhost\Ubuntu\home\<wsluser>\.claude\.credentials.json"
   ```
   This persists across reboots (`HKCU\Environment`). Reading it requires WSL to
   be reachable — accessing a `\\wsl.localhost\...` path auto-starts WSL if it's
   not already running, but expect a few seconds' delay on the very first read
   after a fresh boot.

9. **Confirmed end-to-end working, including a real reboot.** Running the
   daemon manually connected over BLE and polled the Anthropic API
   successfully on 2026-08-06. After an actual overnight shutdown and
   power-on the next morning (2026-08-07), the daemon log shows fresh
   `Connected` + successful `Sending: {...,"ok":true}` entries starting
   `06:44:48` — live usage data flowing automatically after login, no manual
   intervention. See "2026-08-07 post-reboot findings" for the one thing that
   needed fixing first.

## Bug found, root-caused, and fixed

**Symptom**: tray showed `Error: daemon crashed: ModuleNotFoundError` — every
time — repeating with exponential backoff (2s, 4s, 8s... capped at 30s), whether
launched via the tray installer's autostart mechanism or a manual
`python daemon\tray_windows.py` (using `C:\Python314\python.exe`, the BASE
interpreter resolved via `PATH`).

**Exact error**:
```
ModuleNotFoundError: No module named 'winrt.windows.foundation.collections'
```
raised from inside `bleak`'s WinRT backend (`bleak/backends/winrt/client.py`,
`_ensure_success`) while unwrapping a GATT services result during `connect()`.

**Log location** (for next time): `%LOCALAPPDATA%\Clawdmeter\daemon.log`
(`C:\Users\<winuser>\AppData\Local\Clawdmeter\daemon.log`), rotating file logger,
readable directly from WSL via `/mnt/c/Users/<winuser>/AppData/Local/Clawdmeter/daemon.log`.

**Wrong theory (tried and reverted)**: first assumed this was a threading issue
— `bleak`'s background daemon thread doing the first-ever lazy import of a
`pywinrt` submodule mid-COM-callback. Added eager warm-up imports of several
`winrt.windows.*` submodules on the main thread in `tray_windows.main()`. This
did NOT fix it — a later run crashed on the exact warm-up import line itself,
on the main thread, before the daemon thread even started, disproving the
threading theory outright. **This fix was reverted** — it's not in the repo
anymore.

**Actual root cause**: `daemon/autostart_windows.py` deliberately launches the
tray via the **base** interpreter's `pythonw.exe` (`sys.base_exec_prefix`), not
the venv's own `pythonw.exe` — because the venv's `pythonw.exe` is a known
CPython launcher-bug redirector that respawns a console `python.exe` as a
child and pops a visible console window. To make the venv's dependencies
resolve under that base interpreter, `tray_windows.py` used to call
`site.addsitedir(<venv site-packages>)` at import time.

That `addsitedir` approach reliably fails to resolve
`winrt.windows.foundation.collections` specifically (confirmed empirically:
`.venv\Scripts\python.exe -c "import winrt.windows.foundation.collections"`
succeeds standalone, every time; the identical import under
base-`python.exe` + `addsitedir` fails, every time). This is most likely a
Python 3.14 + `winrt` 3.2.1 interaction specific to how `addsitedir`-injected
paths resolve that one compiled, multi-distribution namespace submodule — not
a general environment or install problem, since every other `winrt.windows.*`
submodule resolves fine via the same mechanism.

**Real fix applied** (`daemon/tray_windows.py`, `main()`): removed the
`site.addsitedir()` block entirely. Instead, at the very top of `main()` —
before the single-instance mutex is acquired — the tray now detects whether it
is running under the base interpreter (`sys.executable` != the venv's
`python.exe`), and if so, re-execs itself into the venv's own `python.exe` via
`subprocess.Popen(..., creationflags=subprocess.CREATE_NO_WINDOW, stdin/stdout/stderr=DEVNULL)`,
then returns immediately. This:
- Gives the daemon the exact interpreter environment already proven to work
  (running directly under `.venv\Scripts\python.exe`), sidestepping the
  `addsitedir` bug entirely.
- Stays windowless despite `python.exe` being a console-subsystem binary
  (`CREATE_NO_WINDOW` suppresses the console Windows would otherwise allocate).
- Never triggers the venv's own buggy `pythonw.exe` redirector (we call
  `python.exe` ourselves, with explicit no-window flags, instead).
- Must run before the single-instance mutex: the base-interpreter launcher
  process exits without ever touching the lock, so only the real re-exec'd
  (venv) instance ever holds it — no race.

Also updated a stale comment in `daemon/autostart_windows.py::_command()` that
referenced the old (removed) `addsitedir` approach.

**Committed locally as `6ef4bba`** (`daemon: fix winrt ModuleNotFoundError
under base-interpreter autostart`) — **not pushed anywhere**, just a local
commit on `main` so `git pull` can layer upstream changes on top cleanly (see
"Upgrading to new upstream releases" below). The full diff, embedded here in
case that commit is ever lost, reverted, or overwritten by a future rebase:

```diff
diff --git a/daemon/autostart_windows.py b/daemon/autostart_windows.py
index 3bded88..6c59407 100644
--- a/daemon/autostart_windows.py
+++ b/daemon/autostart_windows.py
@@ -47,9 +47,10 @@ def _command(tray_script: str | None = None) -> str:
     The base pythonw loads in-process and is genuinely windowless.
 
     The path is never hard-coded (D-08, CLAUDE.md "repoint ExecStart" lesson);
-    both paths are quoted for space safety.  tray_windows.py adds the venv's
-    site-packages to sys.path itself, so the venv's deps still resolve under the
-    base interpreter.
+    both paths are quoted for space safety.  tray_windows.py detects it is
+    running under this base interpreter and re-execs itself into the venv's
+    own python.exe (CREATE_NO_WINDOW, so still windowless) before touching any
+    venv dependency — see the re-exec block at the top of tray_windows.main().
 
     Args:
         tray_script: absolute path to the tray entry script.  Defaults to this
diff --git a/daemon/tray_windows.py b/daemon/tray_windows.py
index 7caeffa..8882c1e 100644
--- a/daemon/tray_windows.py
+++ b/daemon/tray_windows.py
@@ -31,16 +31,6 @@ _REPO_ROOT = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
 if _REPO_ROOT not in sys.path:
     sys.path.insert(0, _REPO_ROOT)
 
-# Autostart launches us with the BASE interpreter's pythonw.exe, not the venv's
-# (see autostart_windows._command — the venv pythonw redirector pops a console
-# window). The base interpreter does NOT see the venv's site-packages, so add
-# them here to resolve pystray/bleak/PIL. os.path.isdir guards the no-venv and
-# already-inside-venv cases; site.addsitedir is a no-op on a missing dir anyway.
-_VENV_SITE = os.path.join(_REPO_ROOT, ".venv", "Lib", "site-packages")
-if os.path.isdir(_VENV_SITE):
-    import site
-    site.addsitedir(_VENV_SITE)
-
 # ---------------------------------------------------------------------------
 # TrayState — thread-safe scalar bridge (loop -> tray)
 # ---------------------------------------------------------------------------
@@ -167,6 +157,39 @@ def main() -> None:
     so the module can be imported on a GTK-less Linux dev box for unit tests
     of the pure helpers (TrayState, header_text) without pystray failing.
     """
+    # Autostart launches us with the BASE interpreter's pythonw.exe, not the
+    # venv's (see autostart_windows._command — the venv's own pythonw.exe is a
+    # redirector stub that respawns the console python.exe as a child and pops
+    # a window, a documented CPython venv-launcher bug). Injecting the venv's
+    # site-packages onto the base interpreter via site.addsitedir() looked like
+    # an equivalent fix but is NOT: winrt's compiled multi-distribution
+    # submodule `winrt.windows.foundation.collections` reliably raises a
+    # spurious ModuleNotFoundError under base-python + addsitedir, even though
+    # `.venv\Scripts\python.exe -c "import winrt.windows.foundation.collections"`
+    # succeeds standalone every time (confirmed empirically). Re-exec into the
+    # venv's own python.exe — CREATE_NO_WINDOW keeps it windowless despite
+    # python.exe being a console-subsystem binary — sidesteps the bug by giving
+    # winrt the exact interpreter environment it was proven to work under.
+    #
+    # This MUST run before the single-instance mutex below: this base-process
+    # invocation is just a launcher and has to exit without ever acquiring the
+    # lock, leaving it free for the re-exec'd (venv) instance to take.
+    venv_python = os.path.join(_REPO_ROOT, ".venv", "Scripts", "python.exe")
+    if sys.platform == "win32" and os.path.isfile(venv_python) and (
+        os.path.normcase(os.path.abspath(sys.executable))
+        != os.path.normcase(os.path.abspath(venv_python))
+    ):
+        import subprocess
+        subprocess.Popen(
+            [venv_python, os.path.abspath(__file__)],
+            creationflags=subprocess.CREATE_NO_WINDOW,
+            close_fds=True,
+            stdin=subprocess.DEVNULL,
+            stdout=subprocess.DEVNULL,
+            stderr=subprocess.DEVNULL,
+        )
+        return
+
     # Single-instance guard FIRST — before icons, the daemon thread, or any BLE
     # work. If another tray already owns the session mutex (e.g. ARSO restored a
     # console instance and the headless autostart also fired), exit silently.
```

**Verification done**:
- `.venv\Scripts\python.exe daemon\tray_windows.py` (direct venv invocation) —
  confirmed working: `Connected`, BLE GATT services resolved fine, no crash.
- Existing unit tests re-run in WSL (`python3 -m pytest daemon/tests/test_windows_tray.py daemon/tests/test_windows_autostart.py -q`):
  **22 passed**. 2 tests deselected/failing only because `httpx`/`bleak` aren't
  installed in this WSL Python at all (Windows-only daemon deps, pre-existing
  gap unrelated to this change — not a regression).
- Confirmed the edit is live through all three path variants (`diff` clean
  across `/mnt/d/...`, `/mnt/c/Users/<winuser>/...`, i.e. what Windows sees at
  `C:\Users\<winuser>\...` and `D:\...`).

**Verified 2026-08-07**: the re-exec fix itself works correctly.
`Get-CimInstance Win32_Process` on the re-exec'd child showed
`ExecutablePath = C:\Python314\python.exe` but
`CommandLine = ...\.venv\Scripts\python.exe ...\tray_windows.py` — this is
expected, not a bug: since CPython 3.11, a venv's `Scripts\python.exe` is a
tiny native launcher stub (not a separate binary) that reads `pyvenv.cfg` and
hands off to the base interpreter directly, while `Get-CimInstance` still
displays the original invocation as `CommandLine`. No console window
appeared — confirmed by direct observation — because both hops are
console-subsystem (no subsystem mismatch, unlike the venv's `pythonw.exe`
stub, which redirects into a *different*-subsystem child and is the actual
documented buggy case this whole design avoids).

## 2026-08-07 post-reboot findings

The user shut down for the night on 2026-08-06 and powered on the next
morning. **No tray icon appeared.** This turned out to be unrelated to the
`winrt`/`addsitedir` bug above — a different, simpler problem:

**Root cause**: the `HKCU\Software\Microsoft\Windows\CurrentVersion\Run\Clawdmeter`
registry value was **completely absent** — checked directly with
`Get-ItemProperty ... -Name Clawdmeter` (no output) and confirmed by listing
the whole `Run` key (no `Clawdmeter` entry among OneDrive/Teams/OpenVPN/etc.).
No process was running either (`Get-Process pythonw,python` returned nothing).
So autostart hadn't silently failed at logon — it was simply never registered
to fire in the first place (most likely: the tray's "Start at login" menu
toggle got unchecked during the many manual test-run cycles the previous
day, which calls `autostart_windows.disable()` and deletes the value; this
was never independently re-verified after the original `install-windows.ps1`
run).

**Fix**: re-register it directly, without re-running the whole installer:
```powershell
cd C:\Users\<winuser>\code\Clawdmeter
.venv\Scripts\python.exe -c "import daemon.autostart_windows as a; a.enable(tray_script=r'C:\Users\<winuser>\code\Clawdmeter\daemon\tray_windows.py')"
```
Then verified the value was written, then manually simulated the exact
autostart invocation (`& "C:\Python314\pythonw.exe" "...\tray_windows.py"`) —
confirmed it produced the correct two-process pattern (see the verification
note above), no console flash, and the tray went green with live data
flowing within seconds.

**If this happens again** (no tray icon after a reboot): check the `Run` key
first, before assuming the `winrt` crash bug has resurfaced — check
`daemon.log` for the actual failure signature (see below) to tell the two
apart quickly.

## If you're starting fresh after a reboot that didn't work

1. Check whether the tray icon is present at all (bottom-right notification
   area). If absent, check the Run key first — this was the actual cause the
   one time this happened (see "2026-08-07 post-reboot findings" above):
   ```powershell
   Get-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" -Name Clawdmeter
   ```
   No output = not registered. Re-register with:
   ```powershell
   cd C:\Users\<winuser>\code\Clawdmeter
   .venv\Scripts\python.exe -c "import daemon.autostart_windows as a; a.enable(tray_script=r'C:\Users\<winuser>\code\Clawdmeter\daemon\tray_windows.py')"
   ```
   Then simulate the autostart launch directly rather than waiting for another
   reboot:
   ```powershell
   & "C:\Python314\pythonw.exe" "C:\Users\<winuser>\code\Clawdmeter\daemon\tray_windows.py"
   ```
2. If the Run key IS present but the icon still didn't appear, or it's
   present but red/amber, check the log:
   ```powershell
   Get-Content "$env:LOCALAPPDATA\Clawdmeter\daemon.log" -Tail 60
   ```
3. If you see the `ModuleNotFoundError: winrt.windows.foundation.collections`
   crash again, the re-exec fix may not have applied correctly — verify
   `daemon/tray_windows.py`'s `main()` has the re-exec block (search for
   `CREATE_NO_WINDOW`) BEFORE the `_acquire_single_instance()` call.
4. If it's a token/401 issue: check `~/.claude/.credentials.json`'s
   `claudeAiOauth.expiresAt` in WSL before assuming you need `claude login`
   again — a stale read (e.g. via a cached `\\wsl.localhost\...` SMB view) can
   produce a spurious 401 even when the token is genuinely still valid. Retry
   before re-authenticating.
5. Kill stray processes before retrying anything — the single-instance mutex
   means a second launch silently no-ops if an old instance (even a crash-looping
   one) is still alive:
   ```powershell
   Get-Process pythonw,python -ErrorAction SilentlyContinue | Stop-Process -Force
   ```

## Upgrading to new upstream releases

This repo is now set up as a proper fork:

```
origin    -> https://github.com/slipstream1/Clawdmeter.git   (your fork — push here)
upstream  -> https://github.com/HermannBjorgvin/Clawdmeter.git (the original project — pull from here)
```

The local `winrt`/autostart fix is committed (currently `785e928`, authored by
`slipstream1`) and already pushed to `origin/main` — see the full diff earlier
in this doc if that commit is ever lost.

### How to pull in updates and update your fork

```bash
cd /mnt/d/code/Clawdmeter   # or D:\code\Clawdmeter / C:\Users\<winuser>\code\Clawdmeter — same files

git fetch upstream
git log HEAD..upstream/main --oneline   # see what's new before touching anything

git rebase upstream/main                # replays your fix commit on top of upstream's new commits
git push --force-with-lease origin main # update your fork (rebase rewrites history, so a plain push won't work)
```

**Why `rebase` and not a plain merge**: it keeps your commit as a clean,
single patch sitting on top of whatever upstream has added, instead of a messy
merge commit. If upstream's new commits don't touch the same lines, the
rebase finishes with no conflict at all.

**If the rebase stops with a conflict** — most likely in
`daemon/tray_windows.py` or `daemon/autostart_windows.py` — it means upstream
touched the exact same code your fix touches, most plausibly because they
shipped their **own** fix for the same `winrt`/autostart bug. Take theirs:
```bash
git checkout --theirs daemon/tray_windows.py daemon/autostart_windows.py
git add daemon/tray_windows.py daemon/autostart_windows.py
git rebase --continue
```
then re-verify (see the table below) rather than assuming it's still needed —
an official upstream fix supersedes this local one.

### What does and doesn't need to be redone after updating

| After pulling in updates | Action needed? |
|---|---|
| Firmware source changed (`firmware/src/...`) | **Yes** — rebuild and reflash, see "Manual firmware flashing" above |
| `daemon/requirements-windows.txt` changed | **Yes** — reinstall into the existing venv: `.venv\Scripts\python.exe -m pip install -r daemon\requirements-windows.txt` (no need to recreate the venv) |
| `daemon/tray_windows.py` / `autostart_windows.py` changed by upstream | **Maybe** — only if it conflicts with the local fix (see above); if it's unrelated, your fix rebases on top automatically |
| BLE pairing with Windows | **No** — OS-level bonding, untouched by any git operation |
| `HKCU\Run` autostart registration | **No** — untouched by git; only re-run `autostart_windows.enable()` if you separately notice the tray missing after a reboot (see the post-reboot findings above) |
| `CLAUDE_CREDENTIALS_PATH` env var | **No** — a persistent Windows user env var, unrelated to the repo contents |
| WSL/PlatformIO/`usbipd` setup | **No** — all environment-level, not repo-level |

After any firmware reflash or daemon dependency change, re-verify by checking
`daemon.log` for a fresh `Connected` + `Sending: {...,"ok":true}` line, same as
the post-reboot check above — don't just assume the pull worked.

## Key reference paths

| What | Path |
|---|---|
| Repo (native Windows) | `D:\code\Clawdmeter` |
| Repo (Windows symlink) | `C:\Users\<winuser>\code\Clawdmeter` → `D:\code\Clawdmeter` |
| Repo (WSL view) | `/mnt/d/code/Clawdmeter` |
| Git `origin` (your fork) | `https://github.com/slipstream1/Clawdmeter.git` |
| Git `upstream` (original project) | `https://github.com/HermannBjorgvin/Clawdmeter.git` |
| Venv | `<repo>\.venv` (Python 3.14.3, `C:\Python314\python.exe` base) |
| Daemon log | `C:\Users\<winuser>\AppData\Local\Clawdmeter\daemon.log` |
| WSL Claude credentials | `/home/<wsluser>/.claude/.credentials.json` |
| Windows env var pointing at it | `CLAUDE_CREDENTIALS_PATH` = `\\wsl.localhost\Ubuntu\home\<wsluser>\.claude\.credentials.json` |
| Bonded device BLE address | `<BONDED_MAC>` |
| Autostart registry value | `HKCU\Software\Microsoft\Windows\CurrentVersion\Run\Clawdmeter` |
