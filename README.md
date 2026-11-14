# OpenLHSync

openhouse (an open source SteamVR lighthouse driver) and QuestLHSync (lighthouse devices aligned to a Quest's own tracking), kept together with my changes on top.

## Layout

| Path | Repo | Branch | Upstream |
|---|---|---|---|
| `openhouse/` | [j-thy/openhouse](https://github.com/j-thy/openhouse) | `windows-room` | [YusufOzmen01/openhouse](https://github.com/YusufOzmen01/openhouse) |
| `QuestLHSync/` | [j-thy/QuestLHSync](https://github.com/j-thy/QuestLHSync) | `windows-room` | [CreoleVR/QuestLHSync](https://github.com/CreoleVR/QuestLHSync) + [YusufOzmen01/QuestLHSync](https://github.com/YusufOzmen01/QuestLHSync) |

Branches in the forks:

- `master`: untouched copy of upstream.
- QuestLHSync `openhouse`: CreoleVR 1.28 with Sergio's fork merged in (openhouse support, 1.0 channel fix, Linux/WiVRn port).
- `windows-room` (both): my changes, on top of the above.

## My changes

**Room settling on Windows.** openhouse publishes its room (base station poses, whether the room has settled, an epoch that changes when a station moves) so QuestLHSync can hold off on camera sightings while openhouse solves the room. Upstream only does this on Linux. `windows-room` adds the Windows side:

- openhouse: publishes the room through the named file mapping `Local\openhouse-room` (`src/openvr/room_publisher.cpp`, `include/lh/room_shm.h`).
- QuestLHSync: the Windows driver reads it (`OurRoom`/`ReadOurRoom` in `src/driver/driver_main.cpp`) the same way Sergio's Linux app does: the stations used are openhouse's well-placed ones (sigma <= 2.5 cm), sightings wait until those have held still for 3 s, a station moving more than 1 cm / 0.3 deg drops the older sightings (`openhouse station moved: epoch N`), and a different room starts the alignment over. A station openhouse hasn't placed yet is left out instead of holding everything back.

**Room settle timeout: removed (2026-11-12).** It covered openhouse's all-stations settled flag never clearing when one station wasn't placed. Judging the room on the well-placed stations only (Sergio's fork, ported to Windows above) fixes that directly.

**GitHub Actions build for openhouse: dropped (2026-11-13).** Upstream's own `.github/workflows/build.yml` now cross-builds the Windows driver (plus Linux and the firmware) on every push, so it builds `windows-room` too.

## Installing

Don't use `QuestLHSync-steamvr-installer.exe`: it installs and updates from CreoleVR's releases, which would replace this build with stock (and bring back the custom firmware break).

1. **openhouse** (PC): built by GitHub Actions on [j-thy/openhouse](https://github.com/j-thy/openhouse/actions) (upstream's workflow "build", on every push to `windows-room`). Download the run's `openhouse-windows-x64` artifact, extract it (it holds `openhouse-windows-x64.zip`; extract that too), run `openhouse-installer.exe` and pick 1 (pick 2 to go back to SteamVR's driver).
2. **QuestLHSync driver** (PC): extract `QuestLHSync-steamvr-openlhsync.zip` somewhere permanent and, with SteamVR closed, register the `questlhsync` folder:
   ```
   "C:\Program Files (x86)\Steam\steamapps\common\SteamVR\bin\win64\vrpathreg.exe" adddriver "C:\path\to\questlhsync"
   ```
   If a stock QuestLHSync is already registered, remove it first (`vrpathreg removedriver <its path>`, or uninstall it with the installer).
3. **Quest module** (rooted Quest): install `QuestLHSync-quest-module-openlhsync.zip` in Magisk (Modules > Install from storage) and reboot.
4. Turn off SpaceCalibrator / OpenVR-SpaceSync.

Checking it works, in QuestLHSync's log (`%LOCALAPPDATA%\QuestLHSync`): `openhouse room ...: sightings follow its settling`, then `openhouse room settling: sightings wait`, then `openhouse room settled`. A station that isn't well placed yet is simply left out; if fewer than two are, sightings keep waiting.

## Getting it

```
git clone --recurse-submodules https://github.com/j-thy/OpenLHSync
```

## Updating from upstream

```
cd openhouse
git remote add upstream https://github.com/YusufOzmen01/openhouse   # once
git fetch upstream && git checkout master && git merge --ff-only upstream/master && git push
git checkout windows-room && git merge master && git push

cd ../QuestLHSync
git remote add upstream https://github.com/CreoleVR/QuestLHSync     # once
git remote add sergio https://github.com/YusufOzmen01/QuestLHSync   # once
git fetch upstream && git fetch sergio
git checkout master && git merge --ff-only upstream/master && git push
git checkout openhouse && git merge upstream/master && git merge sergio/master && git push
git checkout windows-room && git merge openhouse && git push

cd .. && git add openhouse QuestLHSync && git commit -m "Bump submodules"
```

See [docs/openhouse-notes.md](docs/openhouse-notes.md) for notes from openhouse's author.
