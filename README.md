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
- QuestLHSync: the Windows driver reads it (`OurRoom`/`ReadOurRoom` in `src/driver/driver_main.cpp`). Sightings wait while the room settles, a new epoch drops the older ones, and a different room starts the alignment over. Station poses still come from SteamVR.

Testing: start SteamVR with both drivers, then check QuestLHSync's log for `openhouse room ...: sightings follow its settling`, `openhouse room settling: sightings wait` and `openhouse room settled`.

## Getting it

```
git clone --recurse-submodules https://github.com/j-thy/OpenLHSync
```

## Updating from upstream

```
cd openhouse
git remote add upstream https://github.com/YusufOzmen01/openhouse   # once
git fetch upstream && git checkout master && git merge --ff-only upstream/master && git push
git checkout windows-room && git rebase master && git push --force-with-lease

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
