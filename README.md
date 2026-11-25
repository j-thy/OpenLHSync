# OpenLHSync

openhouse (an open source SteamVR lighthouse driver) and QuestLHSync (lighthouse devices aligned to a Quest's own tracking), kept together with my changes on top.

## Layout

| Path | Repo | Branch | Upstream |
|---|---|---|---|
| `openhouse/` | [j-thy/openhouse](https://github.com/j-thy/openhouse) | `windows-room` | [YusufOzmen01/openhouse](https://github.com/YusufOzmen01/openhouse) |
| `QuestLHSync/` | [j-thy/QuestLHSync](https://github.com/j-thy/QuestLHSync) | `windows-room` | [CreoleVR/QuestLHSync](https://github.com/CreoleVR/QuestLHSync) + [YusufOzmen01/QuestLHSync](https://github.com/YusufOzmen01/QuestLHSync) |

Branches in the forks:

- `master`: untouched copy of upstream (may be stale; nothing uses it).
- openhouse `windows-room`: upstream master as is (2026-11-25). Upstream now publishes the room on Windows itself, which was our only openhouse change.
- QuestLHSync `openhouse`: Sergio's fork, now one "openhouse support" commit on CreoleVR 1.28 (Windows room reader, well-placed stations only, 1.0 channel fix). He dropped the Linux/WiVRn port.
- QuestLHSync `windows-room`: Sergio's fork + my changes below.

Both upstreams rewrote their history in November. Instead of force pushing, the forks take each new version as a merge commit whose files are exactly the new version, so the old history stays reachable.

## My changes (QuestLHSync only)

Tracking fixes from the Nov 9-10 testing (see the PC's `test-logs`), on top of Sergio's fork:

- drop camera sightings from well below the horizon (`minSightingElevation`)
- don't stay locked on support from before a pose break
- resync the headset clock when it jumps
- restart the headset's camera reader when frames stop while it is worn
- re-acquire sooner after a sleep that moved the headset space, only after real sleeps and for a repeated candidate

**Gone because upstream now covers them:** the Windows room publisher in openhouse, the Windows room reader in QuestLHSync (and its 60 s settle timeout), the 1.0 channel fix, and our openhouse GitHub Actions workflow.

## Installing

Don't use `QuestLHSync-steamvr-installer.exe`: it installs and updates from a GitHub release (CreoleVR's or Sergio's), which would replace this build and drop the fixes above.

1. **openhouse** (PC): built by GitHub Actions on [j-thy/openhouse](https://github.com/j-thy/openhouse/actions) (upstream's workflow "build", on every push to `windows-room`). Download the run's `openhouse-windows-x64` artifact, extract it (it holds `openhouse-windows-x64.zip`; extract that too), run `openhouse-installer.exe` and pick 1 (pick 2 to go back to SteamVR's driver).
2. **QuestLHSync driver** (PC): extract the QuestLHSync build zip (`QuestLHSync-steamvr-<commit>.zip`) somewhere permanent and, with SteamVR closed, register the `questlhsync` folder:
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
