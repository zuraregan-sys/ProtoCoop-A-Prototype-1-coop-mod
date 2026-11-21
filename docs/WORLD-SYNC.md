# WORLD SYNC — experiments, evidence and current state

**Status: ON STANDBY.** World sync is disabled in the shipped package
(`sync_world=0`). Applying the host's positions to local cars/NPCs made their
physics fight the replication (jitter, cars pushed around, objects fighting over
position). With `sync_world=0` each PC simulates its own ambient traffic; the
remote player, powers, effects, damage, destruction and grabs still work.

The rest of this document records the experiments and findings as a reference
for resuming the work. None of it is active in the current package; in
particular `spawn_authority`, `budget_zero` and `spawn_suppress` are not in
`coop.ini`.

---

## 1. Goal

Both players should see and interact with the same world:

- the same NPCs and vehicles in the same positions,
- the same destruction (wrecked cars, fire, debris),
- the same spawns and despawns,
- identical effects, trails and decals,
- and one player's actions should affect the other player's world.

The mod also keeps, and does not touch:

- the lobby and UI,
- Steamworks on the game's real AppID (10150),
- friend invitations,
- co-op saves and save transfer,
- the partner marker on the minimap, map and screen.

---

## 2. Starting point: two independent simulations

The engine does not provide a shared network world. Each PC runs its own full
simulation:

- its own physics,
- its own NPC AI,
- its own spawns and despawns,
- its own destruction.

Everything attempted afterwards starts from that fact: there is no server that
simulates the world once.

---

## 3. Attempt A: waking the engine's original netcode

The retail binary retains part of the cancelled online system:

- `engine::SyncObject` / `SyncMessage`: 8-byte messages (factory, value, index)
  for press/event acknowledgements. The retail build has only one registered
  factory (id 2, one object).
- `OnlineControllerManager`, `TimeSync` and `ConsoleManager`: controller/state
  synchronisation and clock.
- `OnlineSync` / `SyncCommands`: game-state commands with a machine mask and
  callback.
- `proto::CoOpBehaviour`: death, respawn and pairing volumes.
- `OnlineSyncLocalWorldCellStreamingTrack` and
  `OnlineSetPairingVolumeStateTrack`: cell streaming and pairing zones as fight
  tracks.
- `DesyncHunter` / `Replay`: divergence detection and replay.
- `FUN_108BF1B0`: real session bootstrap (channels, peer,
  OnlineControllerManager, `StartSync`, desync hunter), launched by the
  `OLSPeer` constructor.

The original design was deterministic lockstep with synchronised inputs and
commands, not VFX replication. Effects appeared naturally because both machines
ran the same simulation.

What happened when trying to wake it:

1. The bootstrap needs the OLS service at `[OLS+0xA0]`. In retail that pointer
   is NULL. A fake OLS service with a no-op vtable was installed.
2. With the fake service the bootstrap advanced: it created `ChannelManager`,
   `ConsoleManagerPeer`, channels 0/1/2/3/4, the `OnlineControllerManager` and
   called `StartSync` (connection at `0x1130DC5C`).
3. The real channels opened and real traffic flowed (types 0..5, lengths 3..46).
   `OnlineManager` became the simulation driver.
4. The local player froze: the online driver waits for peer acks/clock.
5. When bridging the control/state channels, the client crashed at
   `0x108C54E3`. The 46-byte messages contain process-local values (stack
   pointers), not a wire format.
6. `Channel::Send` (`0x108C3F90`) copies the payload into a queue, but
   `Channel::Receive` (`0x108C4030`) returns the payload pointer when the flag is
   1. The channel layer is not a network protocol; it passes local pointers.
7. The real serialisation was in `ConsoleManagerPeer::Send`
   (`0x108C42E0`) and the Quazal packet constructor (`0x10903D50`): checksums,
   encryption, reliability. It is dead proprietary middleware, not a simple
   serialiser.

Why it does not work:

- The peer transport and serialisation do not exist in retail.
- Reimplementing that layer amounts to reimplementing Quazal, not writing a
  four-field serialiser.
- Without it, `OnlineManager` cannot advance the clock or game state.

This path was abandoned.

---

## 4. Attempt B: player proxy, pose and effects (works)

What became stable and is kept:

- The local player is sent by transform (`P1`, ~20 Hz) and pose (`A1`, ~30 Hz).
- The client creates a puppet cloned from the player and applies the pose and
  matrix.
- Powers are applied to the puppet through the game's Lua API
  (`go_SetDesiredTransformation` / `go_EnableTransformationForSlot`).
- Player action effects are replicated with `StartEffect` (`0x1081E6B0`),
  resolving the template by hash.
- Attached effects (charge tendrils, glows) are replicated and receive a stop
  (`FindEffect` `0x1081B3F0` + remove `0x1081D790`) when they stop restarting,
  so they do not loop forever.
- The partner marker works on screen, minimap and pause map.

Limit: this synchronises the player and its effects, not the world.

Note (2026-11): the remote avatar had jitter because old code wrote the matrix
directly and skipped `SetPosition`; the engine's physics/transform reverted it
(the real position froze and snapped). It now always calls `SetPosition`,
smooths rotation separately, sends positions as floats (previously 1/8 unit) and
does not pair the puppet with world actors. Per-packet logging moved behind
`follow_log` to avoid `fflush` 20 times per second.

---

## 5. Attempt C: world position snapshots

Idea: the host sends nearby object positions and the client applies them.

What was done:

- Scan the handle table at `g_engine+0x128348C` (8-byte entries: object at +0,
  serial at +4).
- Send `W1` with up to 512 objects, 80 m, 10 Hz, in packets of 64.
- Apply with `SetPosition` (`0x7E08E0`).

Bugs found and fixed:

- Initially the applied 20/20 were placeholder objects (serial 0, position
  0,0,0). Real NPCs never entered. They were filtered out.
- Writing the matrix 3x3 basis directly corrupted physics objects: crash at
  `0x10838E59`. Reverted to `SetPosition`.
- `SetPosition` on objects with an invalid transformation attribute crashed at
  `0x107E08E9`. The pointer, vtable and translation are now validated before
  moving.
- Freezing behaviours so the client does not simulate: only one relevant
  behaviour (`Physics`) was found in the list; AI kept moving everything else.

Why it was not enough:

- It synchronises positions of objects that already exist on both sides, but
  does not create or destroy objects, and does not equalise AI or destruction
  state.
- When one simulation has destroyed a car and the other has not, no position
  fixes the difference.
- Between corrections each machine keeps simulating; the result is jitter and
  divergence, not a shared world.

---

## 6. Attempt D: shared health and destruction (works)

What works and is kept:

- `engine::HealthBehaviour` stores health at `+0xC` and dead at `+0x24`.
- `SetHealth` (`0x108480D0`) also runs `CheckDeath`.
- Both peers send `H1` with `{index, serial, health}` for nearby objects with a
  `HealthBehaviour`, every 400 ms, in chunks of 100.
- The receiver applies only the lower health (concurrent damage converges).
- `CheckDeath` (`0x10848100`) is hooked to send an immediate health-0 event when
  an object dies and disappears quickly.
- A flag prevents a death loop between the two sides.
- If the host handle does not match the client handle, `H1` resolves through the
  actor link (template + position) created by the world snapshot.

Limit:

- It shares damage and destruction from that moment, but does not reconcile a
  world that already diverged before connecting.
- It does not create or remove objects; if one side never had the object, there
  is nothing to kill.

---

## 7. Attempt E: mapping a live object to its template

### 7.1 The path that failed: looking backwards from the live object

- `g_spawns` stores objects returned by `GameObjectSpawner::Create`, but the
  handle table contains objects created by other paths; pointers did not match
  and the host sent 0 entries.
- The decompilation says `Create` writes `tmpl->name` into the
  `gMetaObjectTemplateName` attribute. In the assembly the key is at
  `0x11283518` (runtime value `BC6CFE1E:2E53EFD6`). Dumping live object
  attributes found no object with that key, neither in the static nor the
  dynamic list.
- A value scan of the first 0x100 bytes and all attributes did not find the
  known hashes.
- A hook on `GameObject::SetName` (slot 6, `0x76830`) did not receive traffic
  cars.

Conclusion: the data is not on the live object, or the engine clears it.
Looking backwards is not viable.

### 7.2 The path that works: intercept creation

- Hook `GameObjectSpawner::Create` (`0x107F25C0`): store the returned pointer and
  template.
- Additional hook on `MetaObjectTemplate::Create` (`0x10606A60`), the lowest
  point through which every template-created object passes. It captures objects
  created by paths other than the spawner.
- Each record stores `{object, tmpl, name[0], name[1]}` in a custom table (no STL
  inside the hooks: no `malloc`/reentrancy).
- Verified at runtime: 517 objects created and 233 alive in the handle table;
  every live object with a template resolves it by pointer. The 48 objects
  without a record are lights/audio created by direct factories, not templates,
  and do not need a hash.

### 7.3 The real hashes

- `0445412B`, `FC0AA38C` and `C54FAADB` were props/initial objects, not traffic
  cars.
- Real traffic cars are the families `CD9B925D`/`CD9B925C`/`CD9B925A`,
  `D8A528D7`, `1F44437C`/`1F44437B`, `BFA628AF`, etc.
- Ambient AI actors are identified by behaviours:
  `VehicleBehaviourAmbient` (cars), `PedBehaviour`/`InfectedPedBehaviour`
  (pedestrians) and `CharacterIntentionBehaviour` (full characters).

---

## 8. Attempt F: freezing the client without disabling behaviours

### 8.1 What failed

- `SetEnabled(false)` on `CharacterMotionBehaviour`, `MotionTreeBehaviour`,
  `MotionStateBehaviour`, `CharacterIntentionBehaviour` and
  `PhysicsBehaviour`: the client crashed at `0x70941B48` (read at `0x11`).
- The behaviour list was found by scanning begin/end pairs around `object+0x20`.
  That scan sometimes landed on another member and saw only one behaviour;
  disabling the rest blindly broke the engine.
- The real controller is embedded at `object+0x20` and its behaviour array
  starts at `object+0x30` (begin/end/capacity). Reading it directly finds the
  complete list (15-30 behaviours on a character).

### 8.2 The solution: empty intention, not disabled behaviours

- The character has `CharacterIntentionBehaviour`; its intention state is at
  `behaviour+0x10`, with pointers to buttons (`+0x20`) and axes (`+0x2C`).
- On the client, every actor paired with the host registers its four locomotion
  axes.
- Hook `CharacterIntentionAxis::Set` (`0x107E25D0`): if the axis belongs to a
  remote actor, force the value to 0. AI keeps running and animating, but gets
  an empty controller, exactly what the engine does with an idle character.
- The hook is written in assembly (naked function) and jumps to the trampoline
  without touching the stack or FPU. A compiler-generated detour corrupted x87
  state and crashed at `0x107E3335` (read at `0x2C`).
- No `SetEnabled`, no components removed. No crash.

### 8.3 Pairing actors when handles differ

- The client first tries the host's handle+serial.
- If the serial does not match (streaming created objects in another order) and
  the entry is an actor, it finds the local object with the same template and
  nearest position (40 m radius), without reusing the same object for two actors
  in one packet.
- The link is stored by host handle and validated against the handle table on
  every packet, to avoid moving dead pointers.
- Before `SetPosition`, it checks that the object is still a live `GameObject`
  (controller at `+0x20` with a valid vtable). Without that check the client
  crashed at `0x107E0834` reading `0x3F800018`.

### 8.4 Measured result

- Two-instance test, 2.5 minutes: zero crashes.
- The host sends 48-169 nearby objects every 100 ms.
- The client applies batches of 48/48 by handle and 34/35 actors by
  template+position when streaming differs.
- Both players see the same street, cars and NPCs; the puppet and marker keep
  working.

### 8.5 Disguise

- Disguise is not a normal transformation: it uses the `Disguise` slot (hash
  `ED51DF8D`/`BD0AEDB8`), with the generic description in `desired` and the
  concrete uniform in `activeDescription` (`78BFEEFBBDB29025`).
- `TransformSlotString` only recognised Attack and Defense, so the slot was
  dropped and the puppet never received it.
- It now recognises Disguise and `ApplyLuaTransformState` forces
  `go_SetDesiredTransformation(puppet, "Disguise", "Disguise")`, which resolves
  the real uniform internally. Verified on both instances:
  `lua: Disguise Disguise set=1 enable=1`.
- The visible disguise model is the shot body (`alexshotbody_disguise`), not a
  full third-person body swap; the slot and faction state are synchronised.

### 8.6 Grab

- `GrabBehaviour` stores its slots at `+0xC` (0x10 bytes: name, object handle
  and flags). Both peers send `B1` with the slot name, handle, template/name
  hash and object position.
- The receiver finds its local copy by handle, instance name or hash + nearest
  position, and sends the engine's `SetGrabbedObjectMessage` to the puppet
  (type `0x1F`, id 1). The puppet's grab system attaches it to the hand and
  plays the carry pose; releasing sends `ReleaseGrabbedObjectMessage` (type
  `0x1F`, id 2).
- Grabbed objects are excluded from the world snapshot so `SetPosition` does not
  separate them from the hand.
- Verified with a forced grab: the client logged `grab apply` and the puppet
  raised the carry pose, no crash.

### 8.7 Attacks: effects, trails and decals

- Attack effects and particles are replicated through `StartEffect`. The client
  receives whip/claw effects and resolves puppet parts without crashing;
  weapon-attached effects are dropped if the part does not exist.
- Destruction window: effects created after `Action::Start` returns
  (explosions, debris, dust) are also sent: 2.5 s after a player action starts,
  or 8 m around a death for 0.7 s. Previously they were lost by the
  "only during the action" filter or by being more than 3 m from the player.
- Hit decals do **not** go through `DecalBehaviour::AddDecal`.
  `DecalAction::Slot5` (`0x104FBAD0`) builds an `AttachedEffectSetup` and calls
  `EffectManager::CreateAttached` (`0x1081CBB0`) at `0x104FBD7E`. The plugin
  hooks `CreateAttached`, recognises those calls by return address and sends the
  0x44-byte setup in a `C1` packet. The other side rebuilds the decal with
  `CreateAttached`, keeping the owner object if the same handle exists (the
  setup position is object-local), otherwise attaching it to the world at that
  position. The shader is resolved by hash in the local inventory; if the guest
  has not loaded the impact package, the manager uses its default shader and the
  material may not be exact. This is what happens with ground cracks: each hit
  has its shader variant, loaded only when that hit runs in the process. Sending
  the name text or injecting the shader from the root inventory does not fix it;
  the hit action/track must be reproduced on the puppet so it loads the shader
  and creates the decal itself.
- Verified with the host fighting and the guest idle: the host produced about
  2,500 natural decal creations, the `C1` packets arrived and the client logged
  decal applies at the same positions. `sync_decals=1` in the package.
- Trails: implemented and fixed to use the real `PartLookup`
  (`gDrawableAttributeName`) and the drawable handle
  (`gDrawableHandleAttributeName`), with `Drawable::IsValid` checks and joint
  names. Even so, `CreateMotionTrail` crashes the client at `0x10708320` with
  both Claws and WhipFist: the failure is in the engine's internal trail→pose
  mapping. `sync_trails=0` by default.
- The trail crash was reproduced with Claws too, so it is not caused by the
  `power_test` input or the whip's initial state.

---

## 9. Current world architecture

- Only the host sends the world (`W2`). The client does not send positions.
- `W2`: role, count, and per object `{index, serial, template hash, flags,
  x/y/z/yaw quantised}`. 48 objects per packet, 10 Hz, 80 m radius.
- On the client:
  1. resolve by index+serial,
  2. if it is an actor and does not match, resolve through the stored link,
  3. otherwise find by template hash + nearest position,
  4. register the remote actor's intention (axes to zero),
  5. validate that the object is a live `GameObject`,
  6. `SetPosition` with the host position.
- Health: `H1` every 400 ms; if the handle does not match, apply to the linked
  object. Immediate death through `CheckDeath`.

---

## 10. Online lobby: clicks and invitations

The friend list is drawn over the main-menu buttons, so friend rows are the same
clips as PLAY/NEW/INVITE. Two details broke invitations when clicking a friend:

- The background menu does not support the mouse by itself: the front end sends
  `onRelease` to the clip under the cursor, so each row now carries its own
  `onRelease` (no `onPress`, which ran the action twice: on press and release).
- Mouse handlers do not check `sID` (a click does not pass "A"); gamepad
  handlers do, because focus calls with `(controlID, sID, selectedIndex)`.

Also, `SetSelectedByIndex(index)` takes a single argument: passing an extra
empty string did not move the selection, and the A button kept firing the
previous row (INVITE reopened the list instead of inviting the friend).

The lobby page state (main, friends, invitation) now lives in the plugin
(`coop_SetMode`/`coop_Mode`), not in a movie variable: that variable was not
visible from the frame handler.

With that, both the gamepad (A on the friend) and the mouse (click on the name)
send one invitation, and the ACCEPT/DECLINE prompt responds to both.

Friend list:

- Four friends per page, with NEXT when there are more and BACK always.
- NEXT and BACK occupy the first free rows (after the friends), so a short list
  has no gap before BACK.
- The list is capped at 256 entries and paged four at a time: 100 online friends
  are 25 pages, and the last page simply has no NEXT.
- `COOP_FAKE_FRIENDS=<n>` in the environment fills the list with n fake names
  (negative = none) to test paging without touching real accounts.
- The focus table is built once when the screen opens: `ClearForScreen` and
  `OnNewScreen` break the clips' mouse events and the gamepad A, so pages only
  change labels and handlers, never the table.

SEARCH (public lobbies of strangers):

- The lobby is created public (`SteamCreateLobby` type 2) so it appears in other
  players' searches.
- Opening ONLINE no longer searches: it only publishes the lobby. SEARCH is the
  only query and runs only when pressed (600 ms debounce).
- The query does not join automatically: found lobbies are shown on their own
  page with the host name and ping in ms. Three results per page, NEXT when
  there are more, SEARCH to refresh and BACK to leave.
- Ping is measured with a `Q0<seq>`/`Q1<seq>` exchange over
  `SteamNetworkingMessages` against the lobby owner; if there is no answer it
  shows `--`.
- While Steam answers, the SEARCH row blinks between `SEARCHING` and
  `SEARCHING...` (`coop_SearchBlink`, half a second per state).
- Solo, the first row no longer shows the player's own name (the player does
  not need to read it): it says `WAITING FOR PLAYER...` and leaves room for the
  actions. In co-op both names are kept.
- BACK always lives on the last row (the clip that used to be `mcBtnDev`), so
  result and friend pages reuse that already-labelled row: the front end
  restores the cached label of the focused row, and changing that row's label
  from the movie is not visible until focus moves.
- The row that keeps focus must not be used for NEXT: NEXT goes on the fourth
  row (no focus when the page opens) and SEARCH on the fifth.

---

## 11. What is still not synchronised

- Host world appearance on the client: the client still shows its own ambient
  world. Render-only `Drawable` proxies were tested and discarded (section 15).
- Spawn/despawn of new ambient actors: each machine still creates and removes
  its own traffic. A car that exists only on the host does not appear on the
  client until both stream the same cell. Authoritative spawn (`S3`) is
  implemented but disabled (`spawn_authority=0`).
- World animation: transform is replicated and intention is emptied, but the
  motion tree is not replicated. A remote NPC can stay idle while the host sees
  it run.
- Vehicles: they have no `CharacterIntentionBehaviour`. The host position is
  imposed at 10 Hz, but local AI keeps accelerating between packets; visible
  jitter and local-collision destruction can occur that the other side does not
  have.
- Destruction that diverged before connecting: health only shares changes from
  that moment; it does not rebuild the initial state.
- World destruction VFX: the VFX of a car destroyed by AI are not replicated
  like player effects (`StartEffect` only covers player actions and attached
  effects).
- WhipFist on the puppet: solved. The whip's second pose requests 25,936 bytes
  from content heap 112, which is full; the allocation returns null and
  `SetRestPose` writes at `0x4380`. `HeapAlloc`, `HeapAllocAligned` and
  `HeapAllocFlags` are hooked: a failed heap-112 allocation is retried on heap 1
  (system). The pose and its shaders then come from the same allocator and the
  `0x107087FA` crash disappears. Tested with both players using WhipFist for
  2.5 minutes: zero crashes. Example log:
  `heap retry: heap=112 size=25936 -> 1 result=...`.
- World animation and destruction VFX: not implemented.

---

## 12. Useful flags and addresses

Config (`coop.ini`):

- `transport=steam|udp`
- `sync_world=0` (host-authoritative world on standby; 1 resumes the
  experiments)
- `sync_damage=1` (health/destruction)
- `sync_grab=1` (objects held in the remote avatar's hands)
- `spawn_authority=0` (experimental; not complete)
- `engine_coop=0` (experimental; original netcode bootstrap)
- `puppet_combat=1` (puppet collisions without damage)

Verified addresses:

- `0x107F25C0` `GameObjectSpawner::Create`
- `0x10606A60` `MetaObjectTemplate::Create` (slot 9)
- `0x11283518` `gMetaObjectTemplateName` (attribute key)
- `0x107E25D0` `CharacterIntentionAxis::Set` (`__thiscall`, `ret 8`)
- `0x107E2580` `CharacterIntentionButton::Press` (`__thiscall`, `ret 4`)
- `0x107E3335` crash point of the compiler-generated detour
- `0x107E0834` `SetPosition` crash on a non-live object
- `0x108480D0` `HealthBehaviour::SetHealth`
- `0x10848100` `HealthBehaviour::CheckDeath`
- `0x1081E6B0` `EffectManager::StartEffect`
- `0x1081B3F0` `FindEffect`
- `0x1081D790` `RemoveEffect`
- `0x108076B0` `EffectAction::Slot5`
- `0x107E0810` `GameObject::ReceiveMessage`
- `0x007E08E0` `SetPosition`
- `0x007EEB50` `Behaviour::SetEnabled`
- `0x007EEF60` `GetHandle`
- `0x10833380` `StartSync`
- `0x108BF1B0` online session bootstrap
- `0x11318530` `OnlineManager` global
- `0x1130DC5C` `gSyncConnection`
- `0x108C3F90` `Channel::Send`
- `0x108C4030` `Channel::Receive`
- `0x108C42E0` `ConsoleManagerPeer::Send` (Quazal)
- `0x10903D50` Quazal packet constructor

Test logs:

- `<game folder>/coop.log`
- `<game folder>/xinput_proxy.log`
- Logs are appended live; close the process for a complete file.

---

## 13. Conclusion

The world is no longer two blind simulations: the host sends and the client
applies. What was achieved:

- mapping a live object to its template by intercepting creation, not by
  looking backwards,
- pairing actors by template and position when handles differ,
- emptying remote actors' intention without disabling behaviours,
- sharing transform, health and death of paired objects.

What is missing for a fully authoritative world:

- host world appearance on the client (render-only proxies tested and
  discarded; section 15),
- ambient spawn/despawn (authoritative spawn `S3` is halfway),
- world animation,
- destruction VFX for non-player objects.

Publication state: the mod ships without that host world appearance. Each
machine keeps its local ambient world; what travels is the puppet, its pose,
effects, damage, grab, camera and the verified attacks.

This document will be updated if the work resumes.

---

## 14. Determinism experiment (2026-11-20)

Goal: check whether two independent instances with the same save, no input and
`DisableAmbientManager` produce the same state. That decides whether the
engine's lockstep is viable.

Method:

- Plugin probe: `determinism_probe=1` hashes the master table every 60 updates
  (vtable, quantised position, health, template hash) and writes `det:` lines to
  `coop_live.log`.
- Hashes: raw (includes index/serial), spatial (sorted), worldSpatial (without
  player/puppet, without template hash), playerHash.
- `determinism_debug=1` dumps world objects as `detobj:` at rel=600, 1200, 1800
  and 2400.
- Comparators: `tools/compare_det.py` and `tools/diff_detobj_nohash.py`.

Static result (`DisableAmbientManager`, no input):

- 263/263 steps: playerHash and object count equal.
- 218/218 objects equal by vtable + position + health in all four dumps.
- Only 4 distant objects had a different template-hash label (`C262E2E5` in A
  vs `000E2FDF`/0 in B); position and health were equal.
- Conclusion: the static world and player physics **are deterministic**. The
  initial worldSpatial difference was a probe false negative from template
  resolution, not engine state.

Result with ambient (same save, no input):

- rel=60: A 84 objects, B 88; player identical, world hash different.
- rel=600: the 218 static objects still match, but 68 objects exist only in A
  and 59 only in B (ambient traffic/pedestrians).
- Counts keep separating: 446/426, 444/437, etc.
- `fixed_tick=30` on both instances (`SimulationDriver` at 30 Hz) does **not**
  fix the divergence: rel=60 still has 63/63 objects and different hashes.
- Conclusion: the ambient director is not deterministic between independent
  sessions. It depends on real time, RNG, task order and cell streaming; fixing
  the tick is not enough.

Implication for lockstep:

- The original netcode is missing (transport, serialisation,
  `OnlineClock`/`OnlineSync`).
- The AmbientManager/streamer would also have to be made deterministic, or
  ambient disabled.
- Lockstep without ambient gives an empty city; with ambient it requires
  rebuilding the cancelled online layer and deterministic streaming. It is not
  "filling in a few functions": it is rebuilding the online mode.
- Recommendation: do **not** invest in lockstep. Continue with
  host-authoritative. The positive result is that the static world is
  reproducible: the host only needs to replicate dynamic actors. (Render-only
  proxies were tested later and also discarded; see section 15.)

Probe flags (not for the package):

- `determinism_probe=1`
- `determinism_debug=1`
- `fixed_tick=30`

---

## 15. Render-only proxies from donor drawables (2026-11-20, discarded)

Goal: show host actors on the client without creating `GameObject`s (which
crashed in physics/constraints) and without moving borrowed local actors (which
caused teleports). The idea was to create a render-only `Drawable` from a local
donor's drawable and move it with `SetTransform`: no `GameObject`, physics, AI or
health.

Short result: **not viable with the current knowledge.** The drawable is
created, moves and inherits animation, but only rigid parts (hands, boots) draw;
the skinned body is invisible. The client is stable: no crash in any test. It is
discarded for the package.

Package context:

- An empty client (`DisableAmbientManager`) does not have the actor art packages
  loaded. The host sends `sourcePackage=0` in spawn snapshots, so archetype
  lookup by name finds nothing.
- Test workaround: use a live local donor drawable (same template hash) as the
  source. The host package is not needed.

Verified engine API (RVA on `prototypeenginef.dll`):

- `0x10818920` `GetDrawableData(handle*)`, cdecl. Returns the 16-byte slot:
  `+0x00` render object (`pure3d::Drawable*`), `+0x0C` archetype
  (`DrawableArchetype*`).
- `0x107ED5F0` `Drawable::CreateInstance(archetype, registerOnCreate,
  poseFlag)`. Hidden return pointer: `(out*, archetype, bool, bool)`. The flag
  becomes `CreateOptions(b2)` and is the same one `RenderBehaviour` uses with
  `poseAttribute != 0`.
- `0x107ED700` `Drawable::CreateInstanceOf(other)`. Also hidden return:
  `(out*, other)`.
- `0x107ED770` `Drawable::Clone() const`. `__thiscall(this=&handle, out*)`,
  `ret 4`. Creates with poseFlag=false and only copies skin state
  (`0x106E09A0`): it does **not** copy bindings or pose.
- `0x107E86E0` `Drawable::SetTransform`. `__thiscall(this=&handle, matrix*)`.
  Only applies if the drawable is registered. Translation at `m[12..14]`.
- `0x107EDA10` `Drawable::ShareBindings(other)`: creates a `PartBinding` per
  part from the donor's binding source (`0x107E8FF0`) and fills
  `batch.shader`.
- `0x107EDA40` `Drawable::SharePose(other)`: shares the donor drawable's pose.
- `0x107EDA60` `Drawable::CopyPoseState(other)`: copies pose state into the
  drawable's own pose.
- `0x107E8740` `Drawable::CopyPoseStateFrom(pose*)`: what `RenderBehaviour` uses
  every `Update` to push the object pose into the drawable.
- `RenderBehaviour`: drawable at `+0x14`, source at `+0x2C` (usually null after
  creation; `GetSource` does not keep it), sourceName `+0x18`, sourcePackage
  `+0x30`. Its real creation is
  `CreateInstance(archetype, true, poseAttribute != 0)` and then
  `SetPose(archetype->CreatePoseInstance())`.
- Drawable render object layout: parts at `+0x54`/`+0x58`, stride `0x12C`.
  `DrawablePart`: primitive `+0x90`, state `+0x94`, binding `+0x98`, batch
  `+0xA0`, batch shader `+0x10C`, renderBits `+0x128`.

Tests and result:

1. `CreateInstanceOf` on the donor drawable: the new handle is distinct and
   does not crash, but it renders a gray untextured silhouette in the rest pose.
2. `Clone()` (`0x107ED770`): same gray silhouette. The clone handle is confirmed
   distinct from the donor.
3. `SetTransform`: verified. The clone appears exactly 2.5 m from the player (a
   visual test with the model floating in the air), so transform is not the
   problem.
4. `ShareBindings` + `SharePose` after `Clone`: the pose follows the donor
   animation (it visibly copies the player idle), but only hands/boots draw; the
   skinned body does not appear.
5. `CreateInstance(archetype, poseFlag=true)` + `ShareBindings` +
   `CopyPoseState` every snapshot: same result. Animation is copied, the body is
   not visible.
6. Enumerated the player's 38 behaviours: there is exactly one
   `engine::RenderBehaviour` (index 17, drawable `0003270D`) and one
   `proto::BlobShadowBehaviour`. There is no other body drawable to clone.

Diagnosis:

- The clone inherits parts, bindings and pose, but not the per-object render
  state that `RenderBehaviour` maintains (segment visibility, skin data tied to
  the GameObject pose attribute, per-instance palette/materials).
- As a result only rigid parts draw; the skinned body is invisible. It is not a
  transform or animation problem.
- Fixing it requires reproducing the complete per-object render state the game
  builds, or finding the game's own visual-copy routine (crowd/reflection
  style), not `Drawable::Clone`.

Separate unresolved problem:

- `hide_local_actors` disables `engine::RenderBehaviour` per object, but local
  actors use `proto::LODRenderBehaviour` and remain visible (100+ actors and 98
  vehicles counted, still visible). The client's local world obscures any visual
  test and would have to be hidden another way.

Flags used (all default 0; not in the package):

- `mirror_drawables=1`, `mirror_drawable_limit=N`
- `mirror_drawable_test=1` (place the clone in front of the player)
- `mirror_drawable_player=1` (clone the local player drawable)
- `mirror_probe=1`, `mirror_behaviour_probe=1`, `mirror_render_probe=1`
- `hide_local_actors=1`, `freeze_hidden_actors=1`, `vehicle_render_off=1`
- `donor_borrow=1`

Conclusion: the render-only `Drawable::Clone` path is discarded. The mod ships
without synchronising the host world appearance: each machine keeps its local
ambient world, and what works is synchronised (puppet, pose, effects, damage,
grab, camera, devastator/claws). If resumed, start with per-object render state
or the native visual-copy routine, not `Clone`.
