# Mission synchronization scope

Status: **partial mission synchronization is the accepted final scope**
(owner sign-off, 2026-11-25), after the general-actor experiment crashed the
guest (`C0000005`, address 0) and was removed.

## Known issue: starting a new campaign from the ONLINE lobby

The `ONLINE -> NEW` shortcut does not play the game's opening flow correctly:
the pre-rendered intro can be skipped/black and the mission can start with the
cutscene pending, leaving both players without control. This is a front-end
start-path bug, not the mission-state sync.

**Workaround:** start the campaign **solo** and play the morgue until you
escape the doctors; save at the checkpoint, then host `ONLINE -> PLAY` and have
the guest load that same save. The mission sync then runs from normal gameplay.

## Supported and tested

- **Scenario state machines (J1):** the host publishes each machine's root,
  current and next branch; the guest applies it. Forced-divergence test and
  final `in_sync + missing == sent` check.
- **ONLINE PLAY save load (G1/G2):** the host sends the co-op saves and both
  load the same game. `NEW` (G3) opens the difficulty screen on the guest and
  syncs the mission state, but the opening issue above applies.
- **NIS begin/load (J2):** the host publishes active sequence players by name
  and state; the guest sends the same play/stop message and reaches the same
  loaded state.
- **Checkpoint file transfer (`checkpoint_sync`):** a checkpoint/autosave the
  host writes is sent and stored by the guest.
- **Mission-actor parity:** the spawned mission actors/markers near the player
  have the same template hashes and positions on both sides.
- **Mission-only world state (`mission_world`):** an explicit list of up to
  eight spawn-template hashes whose positions are host-authoritative; the guest
  applies them. Ambient traffic and everything else stays local.

Suites: `installer`, `update`, `states`, `chat`, `controller`, `missionsync`,
`nissync`, `checkpointsync`, `objectparity`, `missionworld`, `newcampaign`
(`tests\run-tests.ps1`). Artifacts: `tests\artifacts\`.

## Explicitly unsupported

- **Live in-memory checkpoint/respawn.** The running guest keeps the save it
  loaded; the transferred checkpoint is used on the next load. Applying the
  checkpoint stream mid-session (or a controlled reload) is not implemented.
- **Ambient world/traffic streaming.** General world transform sync stays off
  (`sync_world=0`) because host transforms made the client physics jitter; the
  general actor experiment (`mission_world=*`, 240 entries) crashed the guest
  with `C0000005` and was reverted. Only the explicit whitelist is supported.
- **Full NIS camera/sequence timing.** The begin/load is synchronized; each game
  plays the sequence with its own clock and camera.

These are deliberately out of scope for this release; adding any of them is a
new subsystem with its own two-instance suite.
