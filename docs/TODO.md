# TODO

Current line: Steam (App ID 10150) + C# launcher + movie lobby + puppet/event replication. PrototypeFix is required separately and is not bundled; the launcher checks port 27015.
World sync is on standby and disabled in the shipped config.

## Done

- [x] Native tabbed launcher (Play, Install, Game settings, Status, About) with
      live diagnostics and a PrototypeFix INI editor.
- [x] Launcher Game settings include the vanilla video values (resolution, AA,
      V-Sync, shadows, quality, gamma) and a LOD/draw-distance bias; the plugin
      applies them through `GameDatabase::SetInt`/`rm_SetGeoLODBias` when
      `video_override=1`.
- [x] Online-only co-op session: main-menu Continue/New Game stay single-player;
      only Online PLAY/NEW arms replication and puppet spawn.
- [x] Launcher cog settings: language picker, Dark (default)/Light UI modes,
      `launcher.ini`, and a `COOP_LAUNCHER_LANG` override.
- [x] Launcher language follows Windows UI (Spanish/English); crash ZIP reports
      and GitHub issue links.
- [x] Steam transport over `SteamNetworkingMessages` and public co-op lobbies.
- [x] `SEARCH`: host list with name + ping in ms, three per page, `NEXT`
      and `BACK`; no automatic search or join.
- [x] Friend invitations (`INVITE`) with an accept/reject prompt.
- [x] Lobby layout without the local player's own name; fixed `BACK` row and
      searching feedback (`SEARCHING` / `SEARCHING...`).
- [x] Co-op saves in `Prototype Coop`; host sends its slots on `PLAY`.
- [x] Player transform, 67-joint pose, powers, disguise, WhipFist on the puppet.
- [x] Damage/destruction, grabs, particles/explosions and hit-decal setup sync.
- [x] World sync off by default; effect/death windows added for delayed VFX.
- [x] Test flow that actually loads a co-op save; host-fights / guest-idle mode.
- [x] Ground-breaking cracks and hit decals on the guest: `CreateAttached` is
      `(out, setup)`; the fix sends the real setup and rebuilds it with the
      guest's local shader pair, ring key and physics handle per family.
- [x] Vehicle dents: `DeformAction`'s type `0x3D`/id `0x0B` message is replayed
      on the same local target (hash/position fallback).
- [x] Environment dust/debris: `StartEffect` calls attached to a map object
      (`target != 0xFFFF`, outside the player action) are forwarded with the
      sender's world position and replayed there, with the full effect setup.
- [x] Grab throws: when a held object leaves its slot, the host reads its
      physics velocity and sends a `B2` packet; the guest applies it to its
      local copy after releasing the puppet's attach, so the object flies away
      instead of dropping in place.
- [x] Grab carry visibility: `B1` now carries the sender's grab-slot flags and
      the receiver applies them to the puppet's slot before the attach message,
      so the engine's carry path runs instead of leaving the object invisible.
- [x] Effect stops: the flush remembers each forwarded effect's local drawable
      and sends `E2` when that instance disappears or stops emitting, covering
      one-shot looping dust/shockwaves that never restart.
- [x] Decals: if the exact family was never made on the guest, it reuses another
      local family's setup (valid but different material); with no local setup at
      all it skips the decal instead of using an unsafe physics handle.
- [x] Grab attach checks the resolved object's template hash before using it,
      picks the nearest same-name spawn, and writes the sender's slot name into
      an empty puppet slot.

## Next

### 1. World sync (standby)

- [ ] Solve the client physics fighting the streamed transforms (jitter,
      cars/NPCs pushed around) before shipping `sync_world=1` again.
- [ ] Ambient spawn/despawn, world animation and destruction VFX for unmatched
      objects.
- [ ] Keep `spawn_authority`, `budget_zero` and `spawn_suppress` out of normal
      sessions until they are stable.

### 2. Remaining VFX

- [ ] Motion trails (`CreateMotionTrail` crashes at `0x10708320`).
- [ ] Vehicle dents on ambient vehicles that exist only on the host
      (`sync_world=0`); forwarded deforms already apply when the vehicle is
      present locally.
- [ ] WhipFist connecting tendril.

### 3. Online validation (public testers)

- [ ] Collect public reports for the lobby list, ping, join, names, invite
      prompt and save transfer across different Steam accounts.

### 4. Polish

- [ ] Lobby feedback timing and result refresh.
- [x] Cleanup of superseded documentation and old experimental tools.
- [ ] Rebuild `dist\PrototypeCoop.zip` after every shipped change.

### 5. Optional graphics settings

- [ ] Add a `graphics.ini` bridge in the plugin for tested `rm_*` toggles
      (geo LOD bias, particle throttle, distortion, grain) before exposing them
      in the launcher.
- [ ] Keep PrototypeFix unmodified; its permissions require permission before
      redistributing or modifying the binary.

### 6. Crash reports

- [ ] Optional automatic upload would need a small serverless endpoint and a
      server-side token; do not embed a GitHub token in the public launcher.
- [ ] Keep the current local ZIP + prefilled issue flow until that exists.

### 7. Player count

- [x] Launcher Play tab polls Steam public co-op lobbies and shows members
      online. This is a matchmaking count, not installed/solo users.

### 7. Optional save-folder split

- [ ] If main-menu Continue/New Game should also use the solo save folder,
      switch the redirected save path back when no online session is active.
      The current gate only changes replication, not `saves=coop`.

## Rules

- Do not enable experimental engine/world features in the packaged `coop.ini`.
- Rebuild the zip after code or movie changes and smoke-test the lobby.
- Keep changes grouped in meaningful commits.
