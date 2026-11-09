# Handoff: install and test OpenLHSync on Windows

For Claude Code running on An's Windows PC. Goal: install the openhouse and QuestLHSync builds from this repo, start SteamVR, and confirm from the logs that both fixes work. Report what you saw, with log excerpts.

## Context (read first)

- Repo: `D:\Projects\Quest Pro\OpenLHSync` (private, `j-thy/OpenLHSync`). Submodules on branch `windows-room`:
  - `openhouse/` = `j-thy/openhouse`: open source replacement for SteamVR's lighthouse driver (`driver_lighthouse.dll`). Our change: publishes its "room" (base station poses, settled flag, epoch) on Windows through the named file mapping `Local\openhouse-room` (`src/openvr/room_publisher.cpp`).
  - `QuestLHSync/` = `j-thy/QuestLHSync`: SteamVR driver that aligns lighthouse devices to the Quest's tracking using the Quest's cameras. CreoleVR 1.28 + Sergio's fork (1.0 base station channel fix) + our changes: reads openhouse's room on Windows (`OurRoom`/`ReadOurRoom` in `src/driver/driver_main.cpp`) and stops waiting after 60 s if the room never settles (`kRoomWaitMax`).
- Read `README.md` and `docs/openhouse-notes.md` for the rest.
- Git: author `j-thy <jonathan.ting07@gmail.com>`. **Never add Claude as a co-author or add session trailers.** Don't commit build outputs.
- Don't touch the user's other SteamVR drivers, Steam settings, or the Quest unless a step below says so.

## What only An does

These need a person; ask An and wait:

1. Fully quit Steam when you ask (or approve you closing it).
2. Approve any UAC prompt (SteamVR's folder may be under Program Files).
3. For the test run: Quest on and streaming to the PC (Link / Air Link / Virtual Desktop / Steam Link), at least one tracker or controller on, base stations on, then look around slowly for ~30-60 s.
4. Optional: plug in the Quest for the Magisk module update (step 5).

Everything else is yours.

## Steps

### 1. Pre-flight (read-only)

- Steam path: `(Get-ItemProperty 'HKCU:\Software\Valve\Steam').SteamPath`. SteamVR is usually `<Steam>\steamapps\common\SteamVR`.
- Registered external drivers: `& "<SteamVR>\bin\win64\vrpathreg.exe" show`. Note any existing QuestLHSync registration (the stock installer puts it at `%LOCALAPPDATA%\QuestLHSync\questlhsync`).
- Is openhouse already installed? Check for `<SteamVR>\drivers\lighthouse\bin\win64\driver_lighthouse.dll.stock` (openhouse's backup of the stock DLL).
- Back up `%LOCALAPPDATA%\QuestLHSync` (state, headset alignment) to `D:\Projects\Quest Pro\backup\QuestLHSync-<date>` before changing anything.
- Report what you found before step 2.

### 2. Get the openhouse build

**Already done (2026-11-09):** `D:\Projects\Quest Pro\builds\openhouse\` holds the extracted artifact of run 42610241222 (`BUILD.json` `source_commit` b3dedf6, clean; the DLL contains `Local\openhouse-room`). Use it unless `windows-room` has moved past b3dedf6; then fetch the newer run as below. The token in `D:\Projects\Quest Pro\.github-token` can download artifacts (`curl -H "Authorization: Bearer <token>" -L https://api.github.com/repos/j-thy/openhouse/actions/artifacts/<id>/zip`); never print or commit it.

The build comes from GitHub Actions on `j-thy/openhouse`, workflow "Build Windows". The latest successful run is for the `windows-room` head; check with:

```
curl.exe -s "https://api.github.com/repos/j-thy/openhouse/actions/runs?per_page=5"
```

Pick the newest run with `conclusion: success` whose `head_sha` matches `git -C openhouse rev-parse windows-room`.

Downloading the artifact needs a GitHub login with Actions read access. Try in order:
- `gh run download <run id> -R j-thy/openhouse -D "D:\Projects\Quest Pro\builds\openhouse"` if `gh` is installed and logged in.
- Else ask An to download it from the run page (Artifacts section) and drop the zip in `D:\Projects\Quest Pro\builds`.

The artifact contains `driver_lighthouse.dll`, `Install-Driver.bat`, `Uninstall-Driver.bat`, `openhouse-driver.ps1`, `apps\`, `BUILD.json`. Check `BUILD.json`'s `source_commit` matches the run's `head_sha`.

### 3. Install openhouse

- Steam must be fully closed (the script refuses if `steam`, `vrserver`, `vrmonitor`, `vrcompositor` are running). Ask An.
- Run `Install-Driver.bat` from the extracted folder (or `powershell -NoProfile -ExecutionPolicy Bypass -File openhouse-driver.ps1 Install`). It saves the stock DLL as `driver_lighthouse.dll.stock` the first time.
- Verify: `openhouse-driver.ps1 Status` shows the installed SHA256 equal to the artifact's `driver_lighthouse.dll`, and "Stock backup present: True".

### 4. Install the QuestLHSync driver

- Build: `D:\Projects\Quest Pro\builds\QuestLHSync-steamvr-openlhsync.zip` (built from `QuestLHSync` `windows-room` @ `637e2cb`). Do **not** use `QuestLHSync-steamvr-installer.exe`: it installs CreoleVR's stock release and offers to "update" back to it.
- Extract to a permanent folder: `D:\Projects\Quest Pro\installed\questlhsync`.
- If a stock QuestLHSync is registered (step 1), unregister it: `vrpathreg removedriver "<its path>"`.
- Register ours: `vrpathreg adddriver "D:\Projects\Quest Pro\installed\questlhsync"`, then `vrpathreg show` to confirm exactly one questlhsync.

### 5. Quest module (optional, ask An)

`builds\QuestLHSync-quest-module-openlhsync.zip` has Sergio's improved camera blob detection. The stock module still works with this driver, so skip unless An wants it. If yes and `adb devices` shows the Quest:

```
adb push "D:\Projects\Quest Pro\builds\QuestLHSync-quest-module-openlhsync.zip" /sdcard/Download/
adb shell su -c "magisk --install-module /sdcard/Download/QuestLHSync-quest-module-openlhsync.zip"
adb reboot
```

(`su` needs An to grant root on the headset the first time.)

### 6. Test run

1. Ask An to get set up (see "What only An does", item 3), then start SteamVR.
2. Let it run ~2 minutes while An moves around slowly.
3. Collect logs:
   - openhouse: `<Steam>\logs\vrserver.txt`, lines starting `lh:`.
   - QuestLHSync: `%LOCALAPPDATA%\QuestLHSync\questlhsync.log` (only this session's part: from the last `QuestLHSync ... driver starting`).
   - Copy both into `D:\Projects\Quest Pro\test-logs\<date-time>\`.

### 7. Pass / fail

**openhouse is the running driver**
- `vrserver.txt` has `lh: ` lines and `room settled` / `room not settled: placed ... waiting for ...` lines.

**Windows room fix**
- `questlhsync.log` has `openhouse room room-<n>: sightings follow its settling`. If it says `no openhouse room: stock driver` instead while openhouse is clearly running, the fix failed: report that.
- Then `openhouse room settling: sightings wait`, followed by `openhouse room settled` once openhouse's `room settled` appears.
- If it never settles: `openhouse room still settling after 60 s ...: sightings used anyway` after a minute. Note which bases openhouse lists under `waiting for`.

**1.0 channel fix** (matters with the custom multi-station firmware)
- No `base station LHB-...: channel 1` (or 2/3/4) lines for 1.0 stations (model `HTC V2-XD/XE`).
- No `channel check: ... were ...` swap lines.

**Alignment works**
- QuestLHSync acquires an alignment: an `acquired: yaw ... support ...` line (repeated `acquiring, last N s: ...` lines without one means it isn't locking; those lines also say how many camera spots were used and why the rest weren't, including `while openhouse's room settled`). The QuestLHSync page in the SteamVR dashboard shows the state too.
- Trackers sit where they physically are. Ask An to confirm visually.

Report: pass/fail per item, the relevant log lines, anything unexpected (errors, `HMD pose break`, repeated recalibration, `openhouse room changed ... recalibrating` on every start).

## Rollback

- openhouse: close Steam, run `Uninstall-Driver.bat` (restores `driver_lighthouse.dll.stock`). Or verify SteamVR's files in Steam.
- QuestLHSync: `vrpathreg removedriver "D:\Projects\Quest Pro\installed\questlhsync"`, re-register the old one if there was one, restore `%LOCALAPPDATA%\QuestLHSync` from the backup.
- Quest module: uninstall it in Magisk, or install CreoleVR's 1.28 module.
