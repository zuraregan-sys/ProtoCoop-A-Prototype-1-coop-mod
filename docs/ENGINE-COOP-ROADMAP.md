# Engine co-op revival roadmap

> **Exploratory, not active:** the current branch uses the Steam 10150 +
> movie-lobby + puppet/event implementation. This document is the alternative
> path of reviving the engine's own online layer; it is kept for future work.

Goal: stop growing a parallel puppet/VFX replication layer and revive the
engine's own online co-op layer, so both machines run the same simulation and
effects, powers, trails, decals, disguise and whip come out natively. The lobby,
Steam session, invitations, save transfer and player marker stay as they are.

Reference material: the `Arefaris/prototype_decomp` project (see
`DECOMP-AUDIT.md`; reference only, do not copy code) and a local Ghidra
project. Findings are recorded in `ENGINE-NOTES.md`.

## What the original layer is

- `engine::SyncObject` / `SyncMessage`: 8-byte press/release acks for inputs and
  events; one factory (id 2) is registered in the retail build.
- `OnlineControllerManager` (`FUN_108C8BF0`), `TimeSync`, `ConsoleManager`:
  controller/state synchronisation and time sync.
- `OnlineSync` command protocol (`FUN_108C7D00`): command ids with per-machine
  masks and callbacks, used for game-state requests (load/start/transitions).
- `proto::CoOpBehaviour`: death, respawn and pairing-volume state.
- `OnlineSyncLocalWorldCellStreamingTrack` / `OnlineSetPairingVolumeStateTrack`:
  world cell streaming and pairing zones as fight tracks.
- `DesyncHunter` / `Replay`: divergence detection and replay.
- `FUN_108BF1B0`: real session bootstrap (channels, peer, OnlineControllerManager,
  `StartSync`, desync hunter), called from the `OLSPeer` constructor.

Consequence: the original design is deterministic lockstep with synchronised
state transitions and world streaming, not individual VFX messages. That is the
architecture to restore.

## Phase 0 - guardrails (done)

- Keep lobby, Steam AppID 10150, invitations, save folder/transfer and marker.
- Keep the current proxy sync behind `engine_coop=0` as fallback.
- Never ship experimental code enabled by default until verified.

## Phase 1 - session bootstrap

1. Map `OnlineManager` (global `DAT_11318530`, 0x9C bytes) and `OLSPeer` startup.
2. Hook the network constructors/peer so the bootstrap can run on our fake
   `ChannelManager`/`Channel` (no Quazal, no real sockets).
3. Call the real `FUN_108BF1B0(manager, role, 10, 100)` on both peers.
4. Verify: OnlineControllerManager, TimeSync, channels and `StartSync` exist;
   no crash; the local player is still controllable.
5. Fallback: if the real bootstrap cannot be tamed, reproduce the minimal part
   (OnlineControllerManager + StartSync + driver) with our objects.

## Phase 2 - transport bridge

1. Back `ChannelManager::Open`/`Channel::Send`/`Channel::Receive` with the
   existing transport (UDP loopback for tests, SteamNetworkingMessages online).
2. Preserve channel id, message type, source machine mask and reliability.
3. Verify: both peers exchange the bootstrap/state messages and stay alive.

## Phase 3 - player index and second player

1. Make `online_DeterminePlayerIndex` return 0 on one peer and 1 on the other.
2. Start the co-op state/pairing flow so the game spawns the partner entity.
3. Verify: each instance controls its own player, both players exist in the same
   world and each camera follows its own player.

## Phase 4 - world alignment

1. Both peers load the same mission/save at the same time.
2. Sync world-cell streaming (`OnlineSyncLocalWorldCellStreamingTrack`).
3. Verify: same NPCs/vehicles/props near both players; DesyncHunter reports no
   divergence for a stable window.

## Phase 5 - input and state commands

1. Ensure the game's input/command path is active in online mode.
2. Synchronise game-state requests (load, start, pause, death, respawn).
3. Verify: movement, attacks, powers and menus work on both sides without the
   lockstep freeze seen with a bare `StartSync`.

## Phase 6 - native gameplay effects

1. Retire the proxy effect/trail replication once the native simulation is
   authoritative.
2. Verify: charge-jump tendrils, full-charge sweep, landing decals, wall-run
   traces, demolition fist and power effects are identical on both peers.
3. Fix disguise and whip through the native transformation/damage paths; no
   crashes when both players use them.

## Phase 7 - co-op behaviours

1. Death/respawn, pairing volumes and progression flags.
2. Verify: a player dies and respawns, a partner leaves/joins a pairing volume,
   and the mission keeps progressing.

## Phase 8 - packaging and regression

1. Keep the lobby/Steam/marker tests green.
2. Package `dist/PrototypeCoop.zip`, update `LEEME.txt` and `ENGINE-NOTES.md`.
3. Run the two-instance regression (boot, lobby, load, movement, effects,
   invites) before calling it done.

## Known risks

- The retail scripts/content may not include the second player spawn, cameras
  and HUD that the online layer expects.
- Determinism: physics and streaming must match; DesyncHunter detects but does
  not fix divergence.
- The dead OLS/Quazal service is replaced at the channel seam; session/auth must
  be ours (Steam lobby already is).
- Player damage: the proxy currently uses `go_SetDamageScale(0)` /
  `go_SetDamageable(false)`; the native path must keep players from killing each
  other while allowing hits.
