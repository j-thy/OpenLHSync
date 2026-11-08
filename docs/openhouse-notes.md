# openhouse notes

Source: posts and replies from Sergio Marquina ([@marquina_osu](https://x.com/marquina_osu)) on X, recorded 2026-11-08.

## What it is

openhouse is a reverse engineered, improved, open source version of SteamVR's lighthouse driver.

- Better tracking than the stock driver
- Supports more than 2 Lighthouse 1.0 basestations at the same time (requires custom basestation firmware)

Upstream: https://github.com/YusufOzmen01/openhouse

## Usage tips

- **Room solve phase:** the first ~30 seconds after starting SteamVR is when it solves the room. If devices jump around, let it solve by slowly moving the devices around.
- **Misbehaving device:** if a device repeatedly acts weird, turn it off and back on.

## Latest release (custom firmware fix)

A new release fixes issues, especially with the custom 1.0 basestation firmware and the pose solver.

If your 1.0 basestations show a **blinking red light**:
1. Flash your backup firmware
2. Then flash the new firmware

## Compatibility with other tools

Q (Tetsu, [@TetsuAfterDark](https://x.com/TetsuAfterDark)): How does it behave with SpaceSync, QuestLHSync, OVR-SmoothTracking, etc.?

A (Sergio):
- All of them should work.
- Exception: **QuestLHSync breaks if you're using the custom basestation firmware** (the one for running more than 2 1.0 basestations). Without the custom firmware, QuestLHSync works fine.
- Sergio has a fork of QuestLHSync for the custom basestation firmware: https://github.com/YusufOzmen01/QuestLHSync
- Original QuestLHSync: https://github.com/CreoleVR/QuestLHSync

## Using it with a Quest (no lighthouse headset)

Q (namesarehard, [@toasterarenice](https://x.com/toasterarenice)): It only loads the driver when a lighthouse headset is connected. How do I enable it with Quest?

A (Sergio):
- It should load whenever any lighthouse hardware is connected (trackers, controllers, etc.). A lighthouse headset isn't needed.
- If it doesn't load, try reinstalling openhouse.
- If it still doesn't work, DM Sergio your SteamVR log: `vrserver.txt`, in the `logs` folder of your Steam install (e.g. `C:\Program Files (x86)\Steam\logs\vrserver.txt`).

## Why stock QuestLHSync breaks with the custom firmware

openhouse sets each base station's mode label (`Prop_ModeLabel_String`) to its channel: "A", "b", "c" on stock 1.0 firmware, but the slot numbers "1".."4" on the multi-station firmware. Stock QuestLHSync reads any all-digit label as a Lighthouse 2.0 channel (`ReadChannel`, `src/driver/driver_main.cpp`) and maps it to a 2.0 rotor period (`kRotorTicks`, ~19.8-20 ms). 1.0 rotors actually turn at 16.667 ms, so the "channel check" (`Solver::CheckIdentity`, `src/driver/sync.cpp`) scores sightings against rotor periods that don't exist and can swap the stations' identities. Sergio's fork ignores the label on 1.0 stations (model "HTC..."), so only geometry identifies them, as with stock firmware.
