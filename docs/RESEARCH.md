# RESEARCH — Prototype (2009) and the feasibility of a co-op mode

**Date:** 2026-11-10  
**Target game:** Prototype (2009), Steam version (App ID 10150, build 19788432)  
**Scope:** build identification, prior work, review of the `pablopenna/prototype-decomp` project, technical feasibility.  
**Rules followed:** the Steam installation was not modified, no tools were installed and no code was written during this phase. All local analysis was read-only (listings, hashes, PE headers, strings and DNS queries).

> **Report convention:**  
> **[FACT]** = directly checked in the files, tools or a cited source.  
> **[HYPOTHESIS]** = reasonable technical inference, not yet verified at runtime.  
> **[UNVERIFIED]** = could not be checked with the available means.

---

## 0. Executive summary

1. **The engine is identified:** Prototype uses Radical Entertainment's **Titanium Engine**, with the **Pure3D** renderer, its own **ravenphysics**, Scaleform GFx, Lua 5.1, Bink, Direct3D 9 and **Visual Studio 2005 SP1 (MSVC 8.0)**. `detect_engine` did not recognise it because its signature database does not include Titanium; that does not mean the engine is unknown.
2. **The build is the Steam patch from 2025-09-29** (DLL PE timestamp 2025-08-14). The DLL matches the reference version used by the decompilation projects. The EXE is only a loader; the public package never modifies or replaces it.
3. **No public co-op mod for Prototype is known.** The Co-op Anything directory (400 entries, 252 games) has no entry for Prototype or Titanium; GitHub, Nexus and ModDB show no co-op project. What exists are fixes, visual mods, a debug menu and non-co-op ports/recompilations.
4. **The important discovery:** the retail binary **retains much of the multiplayer Radical cancelled**. There are RTTI classes and code for `proto::CoOpBehaviour`, `CoOpMessage`, partner respawn, pairing volumes, input synchronisation (`SyncObject`), `TimeSync`, `OnlineManager`, `DesyncHunter`, `OLSPeer`, lobbies/sessions and a Quazal layer. The hostname `prototypepclive.quazal.net` is also present (today it resolves to `127.0.0.1`). So the project does not start from zero, but the original online service is dead and the session/UI flow is unproven.
5. **Feasibility:** high for a "see and move" prototype with two clients; medium for shared combat in the open world; low-medium for the full campaign; low for "reviving the original online mode" as-is, because that requires rebuilding the Quazal backend.
6. **Recommended architecture:** reuse the deterministic simulation and the existing synchronisation layer, replace the Quazal transport with a **direct session shim (Steam relay/UDP)** and add a host/join flow. Start with two players in free roam and validate determinism with `Replay`/`DesyncHunt` before touching the campaign.
7. **`pablopenna`'s project is useful as methodology, not as a complete base:** 28 verified functions (main loop, `SimulationTick`, `SimulationDriver`, `Time`, strings, options). It contains no network work. The `Arefaris/prototype_decomp` decompilation is more relevant for co-op: it is much more advanced (53.48% of functions) and does reconstruct the online layer.

---

## 1. Phase 1 — Build identification

### 1.1 MCP tool results

**`detect_engine` on `<Steam library>\steamapps\common\Prototype`:**

- 124 files scanned.
- Executables: `prototypef.exe`, `vcredist_x86.exe`, `DirectX\DXSETUP.exe`.
- `best_guess`: `null`.
- `all_matches`: empty.
- `hooking_tools`: "unknown engine".
- Projects using the same engine: none.

**[FACT]** The tool does not identify the engine. That is **not evidence that there is no engine**; its signature database simply does not cover Titanium/Pure3D. Identification was completed with PE headers, strings and public documentation.

**`game_identity` on `prototypef.exe`:**

| Field | Value |
|---|---|
| File | `prototypef.exe` |
| Steam App ID | `10150` |
| Steam build ID | `19788432` |
| Steam name | `Prototype` |
| Manifest | `appmanifest_10150.acf` |
| Architecture | x86 (32-bit), PE32 |
| PE timestamp | `1755193331` = 2025-08-14 |
| Sections | `.text`, `.rdata`, `.data`, `.rsrc` |
| File version | 1.0.0.1 |
| Company / product | Activision / Prototype |

### 1.2 Hashes

| File | SHA-256 | State |
|---|---|---|
| `prototypeenginef.dll` | `d331be205bdde99897952ee88261ab7e63c5c65e36a4660e2a367886a3537071` | **[FACT]** matches the reference version used by `prototype-decomp` and `prototype_decomp`. |

The EXE carries the `.bind` section from SteamStub; the DLL is not wrapped and is what matters for Ghidra addresses and the decompilation. The public launcher only checks that TCP port 27015 is free before starting the game through Steam; a common conflict is `Apple Mobile Device Service` (iTunes/iPhone).

### 1.3 Engine, toolchain and middleware

| Aspect | Result | Evidence |
|---|---|---|
| Engine | **Titanium Engine** (Radical Entertainment) | PCGamingWiki; German Wikipedia; `Pure3D` strings; `code\pure3d` paths; RTTI `engine`, `pure3d`, `proto` |
| Renderer | **Pure3D / PDDI** on Direct3D 9 | `..\..\..\code\pure3d\pddi\src\dx9\...` strings; `d3d9.dll` import |
| Physics | **ravenphysics** (in-house, not Havok) | RTTI `ravenphysics`; solver/constraint strings |
| UI | **Scaleform GFx** | 77 exports, almost all `scaleformp3d`; Flash resource imports |
| Scripting | **Lua 5.1** | `scripts/frevents.lua`, `fre_GetLua*` strings |
| Video | **Bink** | `binkw32.dll`, `binkw32Hooked.dll` |
| Compiler | **Visual Studio 2005 SP1 / MSVC 8.0**, CRT `MSVCR80.dll`, `/MD` | `MSVCR80.dll` import; linker 8.0; decomp README |
| Architecture | **x86-32**, PE32, image base `0x10000000`, 14.2 MB `.text` | `prototypeenginef.dll` PE header |
| Build | Steam patch **2025-09-29**, build `19788432` | `game_identity`; PCGamingWiki/SteamDB |
| DRM | Original EXE has SteamStub; **the DLL is not wrapped** | `.bind` section in the original EXE; DLL has normal sections |
| Anti-cheat | **None detected** | No known service/anti-cheat in the binary |

**Main imports of `prototypeenginef.dll`:** `USER32`, `SHELL32`, `MSVCR80`, `WS2_32`, `iphlpapi`, `d3d9`, `SHLWAPI`, `WINMM`, `binkw32`, `RPCRT4`, `XINPUT1_3`, `KERNEL32`, `ADVAPI32`, `ole32`, `OLEAUT32`, `SETUPAPI`, `DSOUND`.

**Exports:** 77 total; only 6 are engine interface (`EngineInitialize`, `EngineService`, `EngineKeyboardService`, `EngineMouseService`, `EngineWindowFocus`, `EngineTerminate`); the rest are Scaleform. The EXE is a launcher that loads the DLL and calls those 6 exports.

### 1.4 Mods and patches present in the development install

| File | Purpose | Relevance |
|---|---|---|
| `prototype_fix.asi` | **PrototypeFix 1.8.0** by emoose | ASI plugin with CPU/HID/VRAM fixes, `PreferLooseFiles`, `UnlockDevMenu`, etc. |
| `binkw32Hooked.dll` | Bink proxy that loads ASI plugins | **[FACT]** a plugin-loading mechanism already exists without touching the EXE. |
| `prototype_fix.ini` | Configuration | `PreferLooseFiles = true`, `UnlockDevMenu = false`, `DInputKeyboardMouseOnly = true`. |

`PreferLooseFiles = true` is important: loose files (scripts, assets) take priority over `.rcf` archives. `UnlockDevMenu = false` means the development menu is unlockable but not active.

### 1.5 Installation layout

- 11 `.rcf` files (~6.4 GB total): `00woi`, `01woi`, `02woi`, `03woi`, `00audio`, `01audio`, `cells`, `art`, `movies`, `scripts`, `shaders`.
- `scripts\frevents.lua` is compiled Lua bytecode, not readable source.
- Folders `art`, `audio`, `DirectX`, `Docs`, `scripts`.
- Files `proto_uv.dat`, `skuinfo.p3d`, `cementfiles.p3d`, `10150_install.vdf`.
- Saves in `Documents\Prototype\slot-*.bin`, outside the game folder. **They were not touched.**

---

## 2. Phase 2 — Co-op research

### 2.1 Co-op Anything search

| Query | Result |
|---|---|
| `Prototype` | 20 fuzzy matches, **none is Prototype (2009)** (games with "prototype" in the name or notes). |
| `Titanium` | **0 entries**. |
| `Scarface` | **0 entries**. |
| `Hulk` | **0 entries**. |
| `Quazal` | **0 entries**. |
| `Radical Entertainment` | **0 entries**. |

The full directory has **400 entries and 252 games**, but **no Prototype entry**. That is a research gap, not proof that nothing private exists.

### 2.2 Previous multiplayer attempts

**Public search:** GitHub (repositories and terms), Nexus Mods, ModDB, PCGamingWiki, GameFAQs, Co-Optimus, 2008-2009 press.

**Result:** **[FACT]** no public online co-op mod, abandoned project or co-op fork for Prototype (2009) was found. What exists:

- Texture, skin, ReShade, HUD, save and graphics mods.
- **PrototypeFix** (emoose): technical fixes, not co-op.
- **Debug Menu** (dron_3 / Goob's guide): unlocks the `RAD_RELEASE` development menu.
- **ProtoDroid / PrototypeRecomp** (Nazatric): static port/recompilation for Android, not co-op.
- **prototype-decomp** (pablopenna) and **prototype_decomp** (Arefaris): decompilation, not co-op.

**Historical context [FACT]:** Prototype had a multiplayer/co-op mode in development, but Radical cancelled it in 2008 to focus on the campaign. Contemporary press confirms it (Critical Hit, Co-Optimus, Wikipedia). The retail engine retains much of that code (see §3.3).

### 2.3 Reusable tools and projects

| Project | What it provides | Co-op? | Reuse |
|---|---|---|---|
| **PrototypeFix** (emoose) | ASI loader, startup/performance fixes, `PreferLooseFiles`, dev menu | No | **[FACT]** already used; base for plugin loading and loose-file access. |
| **Debug Menu** (dron_3) | Unlocks `RAD_RELEASE` / DEV MENU | No | Useful for testing and inspecting free-roam modes. |
| **prototype-decomp** (pablopenna) | 28 verified functions, XInput proxy, MinHook, Ghidra annotations, address map | No | Methodology, injection, main loop, `Time`, `SimulationDriver`. **No declared license.** |
| **prototype_decomp** (Arefaris) | Much more advanced "matching" decompilation; headers, RTTI, online subsystem source | No | **The most valuable source** for co-op; source-only, non-commercial license. |
| **ProtoDroid / PrototypeRecomp** (Nazatric) | Static recompilation to ARM64, analysis docs | No | Little use for x86; **caution:** the repo may contain game assets; do not copy. |
| **QuazalShark / PRUDP / RDV emulators** (other games) | Tools for Quazal Rendez-Vous protocols | No | Relevant only if reviving the original backend; a separate project. |

---

## 3. Phase 3 — Reverse engineering

### 3.1 `pablopenna/prototype-decomp`

**Repository facts [FACT]:**

- Created 2026-09-10, last push 2026-10-18, 13 stars, 73 files, ~205 KB, C++.
- Approach: **incremental decompilation through an injected DLL**, not byte matching. The truth criterion is behavioural equivalence in the game.
- State: **28 functions replaced and verified**.
- Ghidra: 103,091 functions auto-analysed, no PDB.
- Target: `prototypeenginef.dll` with hash `d331be20…`; the setup **aborts if the hash does not match**.
- Proxy: `xinput1_3.dll`; hooking: MinHook 1.3.4; toolchain: llvm-mingw i686, CMake/Ninja, Ghidra 12.1.2; development on Linux/Proton.
- No declared license (`license: null`).

**What is reconstructed and verified:**

| Subsystem | Functions | State |
|---|---|---|
| Engine entry point | `EngineService` | Verified in game |
| Game loop | `UpdateGameflow`, `IsEngineRunning`, `SimulationTick` | Verified |
| Simulation driver | `UpdateRealTime`, `AccumulateTicks`, `ConsumeTick`, `EndUpdate`, `Activate`, `Reset` | Verified |
| Time | `Time` (constructor, `Set`, `Add`, `Subtract`, etc.) | Verified |
| Launch options | `ParseCommandLineOptions`, `HasOption`, `GetOptionValue`, `CheckTimedExit` | Verified |
| Engine strings | `StrCpy`, `StrTok`, `StrTokR`, `StrNICmp`, `StrICmp`, `AsciiToLower` | Verified |

**What it does not have:**

- **No network, sessions, synchronisation, NPCs, combat, missions, streaming or UI.**
- It only covers startup and the loop; it is a learning project, not a port or a co-op base.
- No PDB; it has to deal with `__thiscall`, `__fastcall`, x87 and aggressive optimisations.
- Addresses are tied to the exact DLL hash. If Steam updates, the whole map is invalidated.
- Addresses target the DLL, not the EXE.

**What can be reused:**

1. **Injection infrastructure:** `xinput1_3.dll` proxy, `DllMain`, MinHook, thunks that preserve `ECX`/`EDX`, file logging, replacement verification at startup.
2. **Verified loop/time implementations**, useful as a base for a mod that needs to hook `SimulationTick` or `UpdateGameflow`.
3. **Address map and Ghidra annotations** (`notes/addresses.md`), including the EXE↔DLL interface.
4. **Test methodology:** scripted gamepad, `TimedExit`, windowed runs, behaviour comparison.
5. **Engine facts:** variable tick (~43/s), `SimulationDriver` at 30 Hz by default but the game calls `SetTickRate(0)`, pause on focus loss, etc.

**Co-op limitations:** 28 functions are nowhere near enough to synchronise an open world. Their value is methodological and infrastructural, not functional.

### 3.2 `Arefaris/prototype_decomp` — the truly relevant project

**Facts [FACT]:**

- "Matching" decompilation with **Visual Studio 2005 SP1**, same target (`PROTOPC_2025`) and **same DLL hash**.
- **75,061 / 140,347 functions** matching (**53.482%**) and **31.059%** of code bytes (2026-11-08).
- The hybrid DLL already boots the game and reaches free roam.
- It includes headers, class layouts, RTTI, symbols and reconstructed source for many subsystems.
- License: **source-only, non-commercial**, forks under the same license.

**Reconstructed online subsystem (partial):**

```
src/code/engine/online/
  onlinemanager.{h,cpp}      OnlineManager, ConsoleManagerPeer, ChannelManager
  ols.{h,cpp}                OLS: online services singleton
  olsgame / olsuser / olsfriend / olsprofile / olsstorage
  onlinelua.cpp              online_DeterminePlayerIndex
  onlinescripting.cpp        Lua wrappers for ConsoleManager
  replay.{h,cpp}             deterministic replay
  replaysimdriver.cpp        simulation driver for replay
  variablesimrate.{h,cpp}    simulation rate control

src/code/engine/system/
  networksockets.cpp         Winsock layer, async DNS, socket table
  syncobject.{h,cpp}         input synchronisation between machines
  desynchunter.{h,cpp}       desync detection
  desyncchecker.cpp          state checksums

src/code/engine/netmessage.h  NetMessageSent / NetMessageAllocator / Dynamic
src/code/engine/fight/onlineconditions.*
src/code/proto/fightitems/online/
  online_conditions.*        respawn, distance, player-type conditions
  online_tracks.*            death, respawn, pairing volume, cell streaming
src/code/proto/
  coopbehaviour.{h,cpp}      proto::CoOpBehaviour
  coopstateinstances.cpp     RespawnState, PlayerDeadState, PairingVolumeState
  messages/coopmessages.h    CoOpMessage and derived classes
```

**Real state:** the worklist says `engine/online/onlinemanager.cpp` has **only 2 matching functions and 55 "not placed" functions**. The structure and behaviour are documented, but it is not a complete, ready-to-use implementation. Treat that source as a map, not as a drop-in binary.

### 3.3 What the retail binary retains from the cancelled multiplayer

This is the most important feasibility evidence. **[FACT]** from strings, RTTI and reconstructed code:

**Co-op classes and messages:**

- `proto::CoOpBehaviour`
- `proto::CoOpMessage`, `CoOpMessageGetPlayerDeathState`, `CoOpMessageGetRespawnState`, `CoOpMessagePlayerDead`, `CoOpMessageSetRespawnState`, `CoOpMessageSetPairingVolumeState`
- `coopPairingProgressionAllowed`, `coopPairingProgressionDisallowed`
- `proto::RespawnState`, `proto::PlayerDeadState`, `proto::PairingVolumeState`
- `proto::fightitems::online::*` (respawn, death, distance, pairing volume, cell streaming conditions/tracks)

**Engine online infrastructure:**

- `engine::OnlineManager`, `ConsoleManagerPeer`, `ChannelManager`
- `engine::SyncManager`, `engine::SyncObject`, `SyncObjectFactory`, `SyncObjectDefault`
- `engine::DesyncHunter`, `DesyncChecker`
- `engine::NetMessagePusher`, `NetMessagePuller`, `NetMessageAllocator*`
- `engine::TimeSync`, `OLSPeer`, `OLS`, `OLSGame`, `OLSUser`, `OLSFriend`, `OLSProfile`, `OLSStorage`
- `engine::NetSystemLink`, `NetSystemLinkPublish`, `NetSystemLinkDiscovery`
- `engine::OnlineControllerManager`, `OnlineSettings`, `OnlineGameDBHelper`, `OnlineAdvert`
- `engine::Replay`, `ReplaySimDriver`, `VariableSimRate`

**Revealing strings:**

- `prototypepclive.quazal.net` (Quazal-based PC online service).
- `HostMatch`, `ols_IsConnectedToHost`, `LobbyConnected`, `LobbyDisconnected`, `PeerLost`.
- `Peer dead. Re-spawn in %d second(s)`.
- `No Multiplayer GUID supplied for Multiplayer Session.`
- `online_GetNumPlayers`, `online_DeterminePlayerIndex`, `onlinePlayerIndex`, `onlineNumPlayers`.
- `onlineSyncLocalWorldCellStreaming`, `onlineSetRespawnState`, `onlineFlagPlayerDeath`, `onlineSetPairingVolumeState`.

**Interpretation [HYPOTHESIS]:** the game had a real online/co-op mode, probably for two players (the code speaks of "partner" and "pairing volume"), with input, clock, respawn and cell-streaming synchronisation. The code remains in the retail binary, but the menu/session flow and the Quazal backend are not operational. **It is not proven that it can be enabled from the current game without writing code.**

### 3.4 Quazal and the service state

**[FACT]** `prototypepclive.quazal.net` resolves to `127.0.0.1` on this machine, with no `hosts` entry. `nslookup` returns the same. That is consistent with a retired or sinkholed domain.

**[HYPOTHESIS]** The original Rendez-Vous/Net-Z backend no longer exists. Even if the code were restored, there would be no server to connect to. Reviving it would require a Quazal-compatible backend (emulators exist for other games, e.g. Ghost Recon Future Soldier Research, but that is a separate protocol reverse-engineering project).

---

## 4. Phase 4 — Evaluation

### 4.1 Feasibility by goal

| Goal | Feasibility | Effort | Comment |
|---|---|---|---|
| **Two clients, see and move** (free roam, no shared NPCs) | **High** | Medium | Can be done with an overlay/host-authoritative layer without touching the simulation. The correct MVP. |
| **Free roam with combat and shared enemies** | **Medium** | High | Requires damage, AI, physics and streaming synchronisation. The original infrastructure helps but must be understood. |
| **Full co-op campaign** | **Low-medium** | Very high (months/years) | Missions, scripts, checkpoints, cinematics, world states, saves. Not a first objective. |
| **Revive the original online mode as-is (Quazal)** | **Low** | Very high | Dead backend, proprietary protocol, unproven UI/session. Only makes sense with a compatible Quazal backend. |
| **Port to another platform** | Out of scope | — | Not needed for PC co-op. |

**Conclusion:** the existence of `CoOpBehaviour`, `SyncObject`, `OnlineManager` and `DesyncHunter` changes the problem from "write netcode from scratch" to "rebuild the transport, session flow and UI, and verify determinism". It is still a large project, but there is a real base to work from.

### 4.2 Systems that would need synchronisation

| Priority | System | Difficulty | Notes |
|---|---|---|---|
| 1 | **Simulation clock and tick** | High | `SimulationDriver`, `OnlineManager`, `TimeSync`, `VariableSimRate`. Variable tick (~43/s) complicates determinism. |
| 2 | **Player inputs** | High | `SyncObject` counts presses/releases and compares them between machines. The central piece of the original design. |
| 3 | **Avatar transform / animation / parkour** | High | Position, rotation, movement state, animation, equipment visuals. |
| 4 | **Combat and damage** | Very high | Attacks, hit reactions, grabs, consume, projectiles, health, armour, regen. |
| 5 | **NPCs, AI and crowds** | Very high | Spawns, military/infected alerts, pathing, ownership. The open world has hundreds of actors. |
| 6 | **Physics and destruction** | High | `ravenphysics`, vehicles, collisions, debris. Cross-machine determinism is not guaranteed. |
| 7 | **World/cell streaming** | High | `onlineSyncLocalWorldCellStreaming`; each client must load the same cell and objects. |
| 8 | **Missions, scripts and events** | Very high | `fight`/`scenario`, objectives, checkpoints, dialogue, cinematics, mission conditions. |
| 9 | **Respawn / death / pairing** | Medium | `CoOpBehaviour` and messages already exist; the most "free" part. |
| 10 | **Saves, profile and upgrades** | Medium | Host-only or per player? The original design appears to keep per-player state. |
| 11 | **UI/HUD/pause/menus** | Medium | Need host/join, lobby, player list, connection states. |

### 4.3 Technical difficulties

1. **The original online layer is incomplete in the decompilation.** `OnlineManager` has only 2 matching functions; 55 are "not placed". The structure is there, but it is not code ready to compile and run.
2. **Quazal is dead.** `prototypepclive.quazal.net` → `127.0.0.1`. Without a backend, the session layer must be replaced or the protocol emulated.
3. **No demonstrated session UI/flow.** `art.rcf` contains `protoonlineoption.swf` and `PROTO_ONLINEOPTION`, but its function is unverified; it may be online advertisement configuration rather than a multiplayer menu. **[UNVERIFIED]**
4. **Fragile determinism.** Variable tick, x87 floating point, RNG, streaming order, AI and physics. `DesyncHunter` exists precisely because the original team expected desyncs.
5. **2025 build.** The DLL was recompiled in 2025 with VS2005. Any Steam update can invalidate addresses and hooks.
6. **32-bit and two instances.** Limited address space, shared memory, CPU/GPU load. Saves must be isolated, and the single-instance/port-27015 checks avoided.
7. **Huge codebase without PDB.** 140,347 functions; `__thiscall`/`__fastcall`; FPO; x87. Decompilation is a multi-year project.
8. **Existing mods.** PrototypeFix patches the EXE and DLL; a co-op plugin must coexist with its hooks and `PreferLooseFiles`.
9. **Legal.** Binaries, assets and extracted data cannot be distributed. Arefaris's decomp is source-only, non-commercial; pablopenna's has no declared license. Contact the authors before reusing code.

### 4.4 Additional tools needed

**Analysis / RE:**

- Ghidra 12.1.x + JDK 21 (persistent project; do not re-import every session).
- x32dbg / WinDbg for live debugging.
- `objdiff` / `reccmp` if following the "matching" path.
- `Process Monitor` / `API Monitor` for mutexes, windows, port 27015 and DLL loading.

**Injection / mod:**

- MinHook (already used by pablopenna) or Detours.
- `xinput1_3.dll` proxy (pablopenna's proven method) or an ASI plugin (a loader already exists with `binkw32Hooked.dll`).
- VS2005 SP1 or llvm-mingw i686 for x86-32 builds.

**Network:**

- **Steamworks SDK** (`ISteamNetworkingMessages` / relay) for connection without opening ports.
- **ENet** or **GameNetworkingSockets** as a UDP alternative.
- Wireshark for captures; QuazalShark/PRUDP only if attempting to revive Quazal.

**Testing:**

- Co-op Anything's `gameslot` / `gamepool` for isolated instances and leases.
- `flight` (traces, divergence, shrink), `chaos`, `fuzz`, `ownership`, `scenarios`, `bots`, `animcheck`, `inspect`, `hotreload`.
- `Replay`, `ReplayDesync` and `DesyncHunt`, already provided by the engine as launch options.
- Two game copies with private saves, windowed mode and `TimedExit` for automated tests.

### 4.5 Recommended multiplayer architecture

**Main recommendation: host-authoritative, reusing the existing deterministic synchronisation, with a replacement Quazal transport.**

```
                 ┌─────────────────────────────┐
                 │        Host (player 1)       │
                 │  authoritative simulation    │
                 │  OnlineManager + SyncObject  │
                 │  CoOpBehaviour + DesyncHunter│
                 └──────────────┬──────────────┘
                                │  custom transport
                                │  (Steam relay / UDP-ENet)
                 ┌──────────────┴──────────────┐
                 │        Client (player 2)     │
                 │  local simulation + input    │
                 │  prediction / interpolation  │
                 └─────────────────────────────┘
```

**Why:**

- The engine already has **synchronised inputs + deterministic simulation + desync detection**. Rewriting that from scratch would discard the most valuable asset.
- `OLSPeer`/`OLS`/Quazal is the dead layer. The practical move is to **replace it with a shim** implementing the same interfaces (`OLSSession`, `OLSGame`, `OLSUser`, callbacks) over a custom transport.
- If determinism fails between two PCs, plan B is **host-authoritative with NPC ownership** and a client-authoritative avatar, using snapshots and interpolation. That is what most co-op mods do and is more tolerant of a 2009 engine.

**Concrete decisions:**

1. **Two-player MVP**, not 4+. The original code appears partner-oriented ("partner", "pairing volume"); there is no evidence of how many players it supported. **[HYPOTHESIS]**
2. **Fixed simulation tick for tests** (e.g. 30 Hz) even though the retail game uses a variable tick, so determinism can be validated. In production, decide whether to keep or force a fixed tick.
3. **Steam transport** (`ISteamNetworkingMessages`) for NAT traversal and relay; direct LAN/UDP fallback.
4. **Minimal custom UI** (overlay or Scaleform) for host/join; do not depend on `protoonlineoption.swf` until its purpose is verified.
5. **Isolate saves and profiles per instance**; never touch the original installation or saves.

### 4.6 Proposed roadmap (before programming)

| Phase | Goal | Exit criterion |
|---|---|---|
| **0. Lab** | Two isolated instances, windowed, private saves; check for single-instance/port-27015 checks. | Two clients boot and reach the menu without touching the original installation. |
| **1. Online map** | Locate `OnlineManager`, `OLS`, `SyncManager`, `CoOpBehaviour` in Ghidra; identify entry points and states. | Class diagram and addresses; questions answered. |
| **2. Handshake** | Loopback session shim; two clients "see" each other without gameplay. | Host and client establish a session; connection log. |
| **3. Movement** | Synchronise input, clock, transform, animation and parkour. | Two avatars move consistently in the same area. |
| **4. Combat** | Damage, health, attacks, grabs, consume; NPC ownership. | Two players fight the same enemies without critical desync. |
| **5. World** | Cell streaming, vehicles, physics, ambient events. | Stable free roam for 10+ minutes. |
| **6. Story** | Missions, scripts, checkpoints, cinematics, saves. | One complete mission in co-op. |
| **7. Online** | Steam transport, host/join UI, reconnect, latency tolerance. | Two PCs play through Steam relay. |

**Critical decision gate:** in phases 2-3, use `Replay`/`DesyncHunt` to check whether two instances with the same inputs produce the same state. If not, switch to pure host-authoritative before investing in missions.

---

## 5. Conclusions

1. **There is no previous co-op mod to reuse.** Prototype is a gap in the Co-op Anything directory. The community has made fixes, textures and ports, but no co-op.
2. **The real asset is inside the binary:** Radical left `CoOpBehaviour`, respawn/pairing messages, `SyncObject`, `TimeSync`, `OnlineManager` and `DesyncHunter`. It is a cancelled multiplayer skeleton, not a complete implementation.
3. **The biggest obstacle is not basic netcode but the backend and session flow.** Quazal is dead (`prototypepclive.quazal.net` → `127.0.0.1`) and there is no evidence of a working multiplayer UI.
4. **`pablopenna/prototype-decomp` is useful for learning and injection infrastructure, not as a co-op base.** Its 28 functions cover the loop and time; they do not touch network, world or combat. Its lack of a license is a problem for code reuse.
5. **`Arefaris/prototype_decomp` is much more relevant:** 53.482% of functions, headers, RTTI and online subsystem source. Even so, `OnlineManager` is far from complete (2 matching functions, 55 not placed).
6. **Verdict:** Prototype co-op is **technically feasible but a serious reverse-engineering project**, not a weekend mod. The realistic path is: two-player free roam → shared combat → campaign, reusing the existing synchronisation and replacing Quazal with a custom transport.

---

## Appendix A — Local evidence collected

- `detect_engine`: 124 files, 4 executables, `best_guess: null`, 0 matches.
- `game_identity`: App ID 10150, build 19788432, PE32 x86, timestamp 2025-08-14.
- `prototypeenginef.dll` hash: `d331be20...537071` (matches the reference).
- DLL PE header: image base `0x10000000`, `.text` 14,213,988 bytes, 77 exports, imports `WS2_32`/`iphlpapi`/`d3d9`/`binkw32`/`XINPUT1_3`/`DSOUND`/`MSVCR80`.
- Strings/RTTI: `proto::CoOpBehaviour`, `CoOpMessage*`, `coopPairingProgression*`, `online_*`, `prototypepclive.quazal.net`, `HostMatch`, `Peer dead. Re-spawn in %d second(s)`.
- `art.rcf`: `PROTO_ONLINEOPTION`, `protoonlineoption.swf`, `OnlineAdvert`, `RAD_RELEASE`.
- DNS: `prototypepclive.quazal.net` → `127.0.0.1`; no `hosts` entry.
- `prototype_fix.ini`: `PreferLooseFiles = true`, `UnlockDevMenu = false`, `DInputKeyboardMouseOnly = true`.
- `prototype_fix.asi`: PrototypeFix 1.8.0 by emoose, exports `InitializeASI`.
- `prototype_fix.log`: confirms `PrototypeFix v1.8.0.0`, the ASI module and the engine DLL at its preferred base `0x10000000`.

## Appendix B — Main sources

- `pablopenna/prototype-decomp` — README, `STATE.md`, `notes/addresses.md`, commits and repository tree (GitHub, 2026-11-10).
- `Arefaris/prototype_decomp` — README, `docs/plan.md`, `docs/progress.md`, `docs/engine_worklist.md`, `src/code/engine/online/*`, `src/code/proto/coop*` (GitHub, 2026-11-10).
- `Nazatric/ProtoDroid` and `Nazatric/PrototypeRecomp` — README and tree (GitHub, 2026-11-10).
- Co-op Anything MCP: `list_docs`, `search_directory` (Prototype, Titanium, Scarface, Hulk, Quazal), `get_skill("coop-triage")`, `read_playbook("networking")`, `read_playbook("desync-hunting")`, `read_playbook("multi-instance-testing")`.
- PCGamingWiki — Prototype (Titanium engine, 2025 patch, PrototypeFix).
- SteamDB — build 19788432, patch 2025-09-29.
- Wikipedia (DE) — Titanium Engine; Wikipedia (IT) — cancelled multiplayer.
- Critical Hit (2008-03-31), Co-Optimus (2008-12-15), Engadget (2008-04-30) — multiplayer cancellation.
- Nexus Mods — PrototypeFix (emoose), Debug Menu (dron_3).
- ModDB — texture/skin mods; no co-op.
