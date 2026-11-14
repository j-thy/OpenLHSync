# Handoff: install and test OpenLHSync on Windows

For Claude Code running on An's Windows PC. Goal: install the openhouse and QuestLHSync builds from this repo, check the Quest Pro side over wireless ADB, start SteamVR, and confirm from the logs that both fixes work. Report what you saw, with log excerpts.

## Context (read first)

- Repo: `D:\Projects\Quest Pro\OpenLHSync` (private, `j-thy/OpenLHSync`). Submodules on branch `windows-room`:
  - `openhouse/` = `j-thy/openhouse`: open source replacement for SteamVR's lighthouse driver (`driver_lighthouse.dll`). Our change: publishes its "room" (base station poses, settled flag, epoch) on Windows through the named file mapping `Local\openhouse-room` (`src/openvr/room_publisher.cpp`).
  - `QuestLHSync/` = `j-thy/QuestLHSync`: SteamVR driver that aligns lighthouse devices to the Quest's tracking using the Quest's cameras. CreoleVR 1.28 + Sergio's fork (1.0 base station channel fix) + our changes: reads openhouse's room on Windows (`OurRoom`/`ReadOurRoom` in `src/driver/driver_main.cpp`) and, like Sergio's Linux app, uses only openhouse's well-placed stations (sigma <= 2.5 cm) and waits until those hold still for 3 s. openhouse is v1.0.3 + our two commits.
- Read `README.md` and `docs/openhouse-notes.md` for the rest.
- Git: author `j-thy <jonathan.ting07@gmail.com>`. **Never add Claude as a co-author or add session trailers.** Don't commit build outputs.
- Don't touch the user's other SteamVR drivers, Steam settings, or the Quest unless a step below says so.

## What only An does

These need a person; ask An and wait:

1. Fully quit Steam when you ask (or approve you closing it).
2. Approve any UAC prompt (SteamVR's folder may be under Program Files).
3. For the test run: Quest on and streaming to the PC (Link / Air Link / Virtual Desktop / Steam Link), at least one tracker or controller on, base stations on, then look around slowly for ~30-60 s.
4. Wireless ADB pairing (step 5): on the headset, Settings > System > Developer Options > Wireless debugging > "Pair device with pairing code", then read you the **pairing code** and the **IP:port shown in that pairing dialog**, and the **different IP:port on the main Wireless debugging screen**. Approve the "Allow wireless debugging" and root (`su`) prompts in the headset.

Everything else is yours.

## Steps

### 1. Pre-flight (read-only)

- Steam path: `(Get-ItemProperty 'HKCU:\Software\Valve\Steam').SteamPath`. SteamVR is usually `<Steam>\steamapps\common\SteamVR`.
- Registered external drivers: `& "<SteamVR>\bin\win64\vrpathreg.exe" show`. Note any existing QuestLHSync registration (the stock installer puts it at `%LOCALAPPDATA%\QuestLHSync\questlhsync`).
- Is openhouse already installed? Check for `<SteamVR>\drivers\lighthouse\bin\win64\driver_lighthouse.dll.stock` (openhouse's backup of the stock DLL).
- Back up `%LOCALAPPDATA%\QuestLHSync` (state, headset alignment) to `D:\Projects\Quest Pro\backup\QuestLHSync-<date>` before changing anything.
- Report what you found before step 2.

### 2. Get the openhouse build

The installed build is in `D:\Projects\Quest Pro\builds\openhouse-<commit>\` for the `windows-room` commit it was built from (`BUILD.json` `source_commit`; the DLL contains `Local\openhouse-room`). If `windows-room` has moved past it, fetch the newer run as below. Older folders under `builds\` are earlier versions. The token in `D:\Projects\Quest Pro\.github-token` can download artifacts (`curl -H "Authorization: Bearer <token>" -L https://api.github.com/repos/j-thy/openhouse/actions/artifacts/<id>/zip`); never print or commit it.

The build comes from GitHub Actions on `j-thy/openhouse`, upstream's workflow "build" (jobs linux, windows, firmware), artifact `openhouse-windows-x64` (a zip holding `openhouse-windows-x64.zip`). The latest successful run is for the `windows-room` head; check with:

```
curl.exe -s "https://api.github.com/repos/j-thy/openhouse/actions/runs?per_page=5"
```

Pick the newest run with `conclusion: success` whose `head_sha` matches `git -C openhouse rev-parse windows-room`.

Downloading the artifact needs a GitHub login with Actions read access. Try in order:
- `gh run download <run id> -R j-thy/openhouse -D "D:\Projects\Quest Pro\builds\openhouse"` if `gh` is installed and logged in.
- Else ask An to download it from the run page (Artifacts section) and drop the zip in `D:\Projects\Quest Pro\builds`.

Since v1.0.3 the artifact contains `driver_lighthouse.dll`, `openhouse-installer.exe`, `settingsschema.vrsettings`, `apps\`, `BUILD.json` (no more .bat/.ps1 installer). Check `BUILD.json`'s `source_commit` matches the run's `head_sha`.

### 3. Install openhouse

- Steam and SteamVR must be closed; the installer offers to close them. Ask An first.
- From the extracted folder: `openhouse-installer.exe --install --yes` (add `--steamvr "<path>"` if SteamVR is in another Steam library). It keeps SteamVR's own driver as `driver_lighthouse.dll.stock` and registers the update check.
- Verify: `openhouse-installer.exe --status`, and the installed `<SteamVR>\drivers\lighthouse\bin\win64\driver_lighthouse.dll` has the same SHA256 as the artifact's.

### 4. Install the QuestLHSync driver

- Build: `D:\Projects\Quest Pro\builds\QuestLHSync-steamvr-06e3dd7.zip` (built from `QuestLHSync` `windows-room` @ `06e3dd7`). Do **not** use `QuestLHSync-steamvr-installer.exe`: it installs a release from GitHub (CreoleVR's, or Sergio's in his latest fork) without our Windows room reader, and offers to "update" back to it.
- Extract to a permanent folder: `D:\Projects\Quest Pro\installed\questlhsync`.
- If a stock QuestLHSync is registered (step 1), unregister it: `vrpathreg removedriver "<its path>"`.
- Register ours: `vrpathreg adddriver "D:\Projects\Quest Pro\installed\questlhsync"`, then `vrpathreg show` to confirm exactly one questlhsync.

### 5. Quest Pro over wireless ADB

The Quest Pro is rooted (the Singularity method: Developer Options and Wireless debugging enabled from Android settings, no Meta developer mode needed). QuestLHSync's headset side is a Magisk module, `questlhsync`, that runs `lhsyncd` (TCP 47280, discovery UDP 47281) and injects `lhsight` into the sensors HAL with Frida while a PC is connected. Logcat tag: `QuestLHSync`.

**Out of scope:** don't root, unroot, change Singularity's settings, telemetry/update blocking, or anything else on the headset beyond what's below.

**adb.** Use `adb` from PATH or `%LOCALAPPDATA%\Android\Sdk\platform-tools`. If there's none, ask An before downloading `https://dl.google.com/android/repository/platform-tools-latest-windows.zip` into `D:\Projects\Quest Pro\tools\`.

**Connect.** PC and headset on the same network. With the values An reads you (see "What only An does", item 4):

```
adb pair <pairing-dialog IP>:<pairing port> <pairing code>
adb connect <main-screen IP>:<main-screen port>
adb devices -l        # the Quest Pro should be listed as "device"
```

The pairing is remembered; the main-screen port changes when Wireless debugging restarts (reboot, toggle), so ask An for it again if `adb connect` fails later.

**Check the headset side (read-only):**

```
adb shell getprop ro.product.model                 # Quest Pro
adb shell su -c id                                 # uid=0 (An approves the root prompt the first time)
adb shell su -c "magisk -v; magisk -V"             # Magisk present (QuestLHSync's module needs it)
adb shell su -c "ls /data/adb/modules; cat /data/adb/modules/questlhsync/module.prop"
adb shell su -c "ls /data/adb/modules/questlhsync"  # no 'disable' or 'remove' file
adb shell pidof lhsyncd                             # running
adb logcat -d -s QuestLHSync | Select-Object -Last 50   # "lhsyncd v1.28 up: serial ..., TCP 47280, discovery UDP 47281"
```

If `su` works but `magisk` doesn't exist, the root method didn't install Magisk: stop and report, since the module can't be installed that way.

**Install our module (ask An first).** `builds\QuestLHSync-quest-module-openlhsync.zip` (Sergio's improved camera blob detection). The stock module also works with this driver, so this is optional. If An says yes:

```
adb push "D:\Projects\Quest Pro\builds\QuestLHSync-quest-module-openlhsync.zip" /sdcard/Download/
adb shell su -c "magisk --install-module /sdcard/Download/QuestLHSync-quest-module-openlhsync.zip"
adb reboot
```

After the reboot, reconnect (new main-screen port from An), and re-run the checks: `module.prop` should show `version=v1.28`, and `adb shell su -c "md5sum /data/adb/modules/questlhsync/lhsight.js"` should match the `lhsight.js` inside the zip.

### 6. Test run

1. Ask An to get set up (see "What only An does", item 3), then start SteamVR.
2. Let it run ~2 minutes while An moves around slowly.
3. While it runs, on the headset side: `adb logcat -d -s QuestLHSync` should show the PC connecting and lhsight starting (capturing), and `adb shell su -c "ps -A | grep -i frida"` should list frida-inject while a PC is connected.
4. Collect logs:
   - openhouse: `<Steam>\logs\vrserver.txt`, lines starting `lh:`.
   - QuestLHSync: `%LOCALAPPDATA%\QuestLHSync\questlhsync.log` (only this session's part: from the last `QuestLHSync ... driver starting`).
   - Headset: `adb logcat -d -s QuestLHSync > questlhsync-headset.txt`.
   - Copy all three into `D:\Projects\Quest Pro\test-logs\<date-time>\`.

### 7. Pass / fail

**openhouse is the running driver**
- `vrserver.txt` has `lh: ` lines and `room settled` / `room not settled: placed ... waiting for ...` lines.

**Quest side**
- `lhsyncd` running, and the PC driver finds it. questlhsync.log has `<model> at <ip>: connected`, then `<model>: firmware ..., module v1.28` and `camera calibration: ...`. Bad signs: `no QuestLHSync headset answers on the network`, `TCP connect failed`, `no camera frames in ...`, `nothing received for 10 s, reconnecting`.
- Camera spots arrive: `acquiring, last N s: X bright spots ...` with X > 0 once An looks at the base stations.
- Headset logcat shows lhsight capturing while SteamVR runs, no crash of the sensors HAL (headset tracking keeps working).

**Windows room fix**
- `questlhsync.log` has `openhouse room room-<n>: sightings follow its settling`. If it says `no openhouse room: stock driver` instead while openhouse is clearly running, the fix failed: report that.
- Then `openhouse room settling: sightings wait`, followed by `openhouse room settled` once openhouse's `room settled` appears.
- QuestLHSync settles on the well-placed stations even while openhouse's own log still says `room not settled ... waiting for` another one. A station that moves logs `openhouse station moved: epoch N`. There is no 60 s timeout any more.

**1.0 channel fix** (matters with the custom multi-station firmware)
- No `base station LHB-...: channel 1` (or 2/3/4) lines for 1.0 stations (model `HTC V2-XD/XE`).
- No `channel check: ... were ...` swap lines.

**Alignment works**
- QuestLHSync acquires an alignment: an `acquired: yaw ... support ...` line (repeated `acquiring, last N s: ...` lines without one means it isn't locking; those lines also say how many camera spots were used and why the rest weren't, including `while openhouse's room settled`). The QuestLHSync page in the SteamVR dashboard shows the state too.
- Trackers sit where they physically are. Ask An to confirm visually.

Report: pass/fail per item, the relevant log lines, anything unexpected (errors, `HMD pose break`, repeated recalibration, `openhouse room changed ... recalibrating` on every start).

## Rollback

- openhouse: `openhouse-installer.exe --uninstall` (puts `driver_lighthouse.dll.stock` back and removes the update check). Or verify SteamVR's files in Steam.
- QuestLHSync: `vrpathreg removedriver "D:\Projects\Quest Pro\installed\questlhsync"`, re-register the old one if there was one, restore `%LOCALAPPDATA%\QuestLHSync` from the backup.
- Quest module: `adb shell su -c "touch /data/adb/modules/questlhsync/remove"` and reboot (or uninstall in the Magisk app), then install CreoleVR's 1.28 module if it was there before.
- Wireless ADB: `adb disconnect`. Leave Wireless debugging and the pairing as An had them.
