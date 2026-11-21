# DECOMP AUDIT — `Arefaris/prototype_decomp`

**Date:** 2026-11-10  
**Repository audited:** `https://github.com/Arefaris/prototype_decomp`  
**Reference commit:** `3009bcafda741c3659d48b745e1679845c8fd9da` ("progress: after the eleventh and twelfth crunch waves", 2026-11-08)  
**Goal:** determine how much of Prototype's original multiplayer is actually reusable.  
**Nature:** read-only audit. Nothing was compiled, injected or modified. The repository was downloaded to a temporary folder; no third-party code was copied into this project.

> **Labels:** **[FACT]** = checked in the repository/documentation. **[HYPOTHESIS]** = technical inference. **[UNVERIFIED]** = could not be checked; the reason is given.

---

## 0. Executive summary

1. **The decompilation is 53.482% of functions** (75,061 / 140,347) and **31.059%** of code bytes. It is a "matching" decompilation project, not a functional port.
2. **The online subsystem is much less reconstructed than the rest.** The worklist marks the network layer (`ConsoleManager`, `Channel`, `MultiTailQueue`, sync; ~270 functions) as **not done**, along with `OLSPeer`, `ConsoleManagerPeer`, `SyncManager` and `DesyncHunterOnline`. Without that layer, online does not work.
3. **The most reusable piece is `SyncObject`** (press/release input synchronisation), together with `DesyncHunter`/`DesyncChecker` and the `Replay` class. They are relatively self-contained and well documented, but **not wired**: there are no reconstructed callers of `StartSync`/`UpdateSync`, and `ApplySyncMessage` is `NONMATCHING` (62%).
4. **`CoOpBehaviour` is only partly reconstructed in source, but the binary has the full logic.** Ghidra recovered `Update` (0x104DB8E0), `OnMessage` (0x104DB980) and the destructor (0x104DB960); it is a small state machine for respawn/pairing/death. It remains the easiest piece to reimplement.
5. **The real transport abstraction is `ChannelManager`/`Channel`, not `NetSession`/`NetConnection`:** those names in `gamelink.h` are guesses. `OnlineManager+0x70` holds a `ChannelManager`; `ChannelManager::Open` creates a `MultiTailQueue` and a `Channel`. That layer is exactly what the decompilation marks as not reconstructed, but its interface is small and replaceable.
6. **`CoOpBehaviour` does not directly depend on `OnlineManager`** in the reconstructed code. Conceptually it could work locally/loopback if its methods are reconstructed and messages are injected, but **it does not work as-is** and provides no network by itself.
7. **License:** `prototype_decomp License v1`, source-only, **non-commercial**, no original files, no distribution of binaries containing the original program, attribution and the same license for forks. This project's rule also forbids copying third-party code: use it as reference, not as pasted code.
8. **Recommended next step:** map `ChannelManager`/`Channel`/`ConsoleManagerPeer`, decide whether to hook the session bootstrap `FUN_108BF1B0` or replace `ChannelManager` at `OnlineManager+0x70`, and test loopback. `CreateOnlineManager` (0x104A29E0) is **not** the online entry point: it is `SoundManager`.

---

## 0.b Addendum — Ghidra verification (2026-11-10)

After the document audit, the DLL was imported into Ghidra 12.1.4 and the key functions were decompiled. This **corrects** several assumptions from the original report:

| Original claim | Verified correction |
|---|---|
| `0x104A29E0` = `CreateOnlineManager` | False. It is the `SoundManager` constructor (800 bytes, `snd_*` registrations). The decompilation name is wrong. |
| `OnlineManager+0x70` = `ChannelManager*` (header) / `NetSession*` (gamelink) | It is a **`ChannelManager`**. `gamelink.h` wrongly calls it `NetSession`. |
| The abstraction is `NetSession`/`NetConnection` | It is **`ChannelManager`/`Channel`**; `ChannelManager::Open` creates a `MultiTailQueue` and a `Channel`. |
| `StartSync` had no callers | `FUN_108BF1B0` (session bootstrap) calls it. |
| `UpdateSync` had no callers | `SimulationTick` (0x10838BA0) calls it. |
| `CoOpBehaviour` had no logic | Ghidra recovered `Update`, `OnMessage` and the destructor; the logic is small and complete. |

**Real startup:**

- `EngineInitialize` (0x1085EFB0) → `FUN_108BF5E0` (0x1085F47E): initialises the online subsystem, creates `OnlineManager` (0x9C) and `OnlineSync` (0x144).
- `OLSPeer` ctor (0x108E7E80) → `FUN_108BF1B0`: host/client session bootstrap.
- `FUN_108BF1B0` creates `ChannelManager` (0x7C) at `OnlineManager+0x70`, `ConsoleManagerPeer` (0x8C) at `+0x74`, opens two channels at `+0x94`/`+0x98`, installs the `OnlineManager` as `SimulationDriver`, initialises `OnlineSync`, calls `StartSync` and creates `DesyncHunterOnline`.
- `OnlineManager::Slot4` (0x108BF780) is the network pump: updates the clock, sends the queue, receives from the channel and dispatches to `OnlineSync`.

**`CoOpBehaviour` recovered:**

- `Update` (0x104DB8E0): advances the three states and emits event 4.
- `OnMessage` (0x104DB980): type 2 and type `0x30` ids 0..4; lazily creates `RespawnState` (0xC), `PlayerDeadState` (8) and `PairingVolumeState` (0x10); sets/queries per-player bits and fires events.
- Size 0x18; fields at +0xC, +0x10, +0x14; the creator `0x10294380` allocates 0x18 and calls the constructor.

These corrections take priority over sections 3, 7, 8 and 11 when names conflict.

---

## 1. License and reuse conditions

### 1.1 What the license allows

**[FACT]** `LICENSE.md` allows using, copying, modifying and distributing the content **as source**, for private or public **non-commercial** purposes, under these conditions:

1. No commercial use (sale, bundle, paywall, advertising, commercial product).
2. No original game files or byte-for-byte extracted/derived material.
3. No distribution of binaries containing part of the original program; each user must build their own hybrid DLL from a legal copy.
4. Separately sold content stays disabled by default.
5. No DRM/protection circumvention.
6. Attribution to Radical Entertainment/Activision and the contributors.
7. Modified versions must be marked as modified.
8. No trademark rights.
9. The same license for forks/derivatives.
10. Clean provenance: no leaked code, confidential SDK or NDA material.

**[FACT]** `LEGAL.md` adds that the license **only covers what the contributors wrote**; it grants no rights over the original program, the game, its assets or its trademarks. It also warns that decompiling and publishing reconstructed code may infringe copyright depending on the jurisdiction, and that the license is not legal advice.

### 1.2 Implications for this project

- **We cannot copy code from that repository into ProtoCoop.** Besides the license, this audit's explicit rule forbids it. We may read it, understand it and reimplement independently.
- Reusing its source under the license would make the resulting project a **source-only, non-commercial derivative without distributable binaries**, incompatible with a normal installable mod if the binary contains original code.
- **The hybrid DLL, an installer and game files cannot be distributed.** Each user would have to build from their own copy.
- **`pablopenna/prototype-decomp` has no declared license** (`license: null`). Without express permission, its code must not be reused; it is useful only as methodology.
- **`ProtoDroid`/`PrototypeRecomp`** are not co-op and may contain game files; they are not a safe source to copy.

**Operational legal recommendation:** use both repositories only as documentation of behaviour, names, layouts and addresses; write your own code; do not copy files or fragments; do not distribute hybrid binaries.

---

## 2. Global decompilation state

| Metric | Value |
|---|---:|
| Matching functions | 75,061 / 140,347 (**53.482%**) |
| Matching code bytes | 3,028,396 / 9,750,615 (**31.059%**) |
| `engine` module | 16,905 functions, 699,609 bytes |
| `proto` module | 35,945 functions, 1,332,738 bytes |
| Hybrid state | Boots and reaches free roam; **no online validation** |
| Target | `PROTOPC_2025`, DLL `d331be20…537071` |
| Compiler | Visual Studio 2005 SP1 (MSVC 8.0) |

**[FACT]** The project distinguishes:

- **match**: the code compiles to the same binary as the original.
- **EQUIVALENT**: same operations in the same order, but not counted as a match.
- **NONMATCHING**: body inside `#ifdef NONMATCHING`, not compiled in the matching build.
- **declared/mapped**: only address and name in `symbols.txt`; no reconstructed body.

This matters: **"match" does not mean "functionally connected"**. Some matching functions have no reconstructed callers, and entire classes exist only as declarations.

---

## 3. Dependency map

```
                         [proto::CreateOnlineManager 0x104A29E0]
                                      |
                                      v
                         [engine::OnlineManager 0x108BF160]
                          +0x70  session / ChannelManager  -----> [NetSession] --Open(kind=4)--> [NetConnection]
                          +0x74  ConsoleManagerPeer                    |                         |
                          +0x78  inner SimulationDriver                |                         +--> Send / Receive
                          +0x80  stepsThisFrame                        |                         |
                          +0x84  clock                                  +--> [GameLinkManager] --Open(kind=6)--> ...
                                                                        |
                                                                        +--> [SyncObject::StartSync 0x10833380]
                                                                              |
                                                                              +--> [SyncObject / ApplySyncMessage]
                                                                              +--> [Replay / ReplaySimDriver]

 [OLS singleton 0x108441E0] --> [OLSGame] --> [RequestGameState / RequestTransition] --> [OnlineSync 0x108C7750]
        |                          |                                                          |
        |                          +--> [OLSUserList]                                        +--> [SyncCommands]
        +--> [OLSStorage] / [OLSProfileProto] / [OLSPeer] / [SyncManager] / [DesyncHunterOnline]
                                                        ^
                                                        |
                                      [ConsoleManager / Channel / MultiTailQueue]
                                             (NOT reconstructed, ~270 functions)

 [ISimulationDriver::Slot7] --> [OnlineClock::GetTime 0x108C7340]   (NOT reconstructed)
 [ReplaySimDriver] --> [DesyncHunter] --> [DesyncChecker]
 [CoOpBehaviour] --> [BehaviourController] (message dispatch); listens to types 0x30, 2 and 4
```

### 3.1 Key finding: the session pointer at `OnlineManager+0x70`

**[FACT]** `onlinemanager.h` calls `+0x70` `ChannelManager* channels` (a guessed name). However, `gamelink.cpp` defines a view of `OnlineManager` where `+0x70` is `NetSession* session`, and `GameLinkManager::Connect()` does:

- `session = gOnlineManager->session;`
- `connection = session->Open(6, -1, 0x2000);`

So **the real transport object is at `OnlineManager+0x70`**, and the header's `channels` name is a guess. `SyncObject::StartSync` does the same through `SyncOwner->session->Open(4, ...)`.

This makes `OnlineManager+0x70` the most promising place to install a custom `NetSession`, provided the creation and lifetime are understood.

### 3.2 Dependency summary

| Node | Depends on | Used by |
|---|---|---|
| `OnlineManager` | `ISimulationDriver`, `SimulationDriverTimeData`, `SimulationDriver* inner`, `ChannelManager`, `ConsoleManagerPeer`, `NetSession` (+0x70) | `CreateOnlineManager`; `GameLinkManager` (reads the session); driver vtable |
| `OLS` | `gOLS`, `ConsoleManager::Callback`, `OLSGame`, `OLSUserList`, `OLSProfile`, `OLSStorage` | `OLSGame`, `ExceptionManager`, `PauseState`, `OLSStorage`, `OnlineGameDBHelper` |
| `SyncObject` | `NetSession`, `NetConnection`, `NetSystem`, `gSyncOwner->machines`, `Replay` | `SyncObjectDefault::Press/Release`; `ReplaySimDriver`; no reconstructed `StartSync`/`UpdateSync` callers |
| `OnlineSync` / `OnlineClock` | no implementation | `ISimulationDriver::Slot7`, `RequestTransition`, `CancelStateRequest`, `StartNextRequest` |
| `DesyncHunter` | `DesyncChecker`, `gDesyncSettings` | `ReplaySimDriver`; state transitions |
| `CoOpBehaviour` | `Behaviour`, `BehaviourController`, `CoOpMessage`, `RespawnState`/`PlayerDeadState`/`PairingVolumeState` | `BehaviourController` (update/messages); `MetaBehaviourCreatorImpl<CoOpBehaviour>` |
| `Replay` | `ReplaySimDriver`, `DesyncHunter`, `RequestGameState`, `PauseState` | `Replay_*` scripts; simulation/real-time tasks |
| `NetSession`/`NetConnection` | sockets, `NetSystem` | `SyncObject`, `GameLinkManager` |

---

## 4. Subsystem audit

### 4.1 `OnlineManager`

**Files:** `src/code/engine/online/onlinemanager.h`, `onlinemanager.cpp`; worklist `engine/online/onlinemanager.cpp` and `sn-ei`.

| Aspect | State |
|---|---|
| Constructor `0x108BF160` | Reconstructed; initialises `+0x70`, `+0x74`, `+0x78`, `clock`, `+0x94` |
| Destructor `0x108BF880` | Reconstructed; calls `Shutdown()` |
| Slots `1, 2, 3, 5, 9` | Reconstructed; delegate to `inner` or update the clock |
| `Slot4` `0x108BF780` | **Declared/mapped; no body** |
| `UpdateClock` `0x108BF520` | **Declared/mapped; no body** |
| `Shutdown` `0x108BF690` | **Declared/mapped; no body** |
| `ChannelManager` | Only `operator new/delete`; no layout/methods |
| `ConsoleManagerPeer` | Only `operator new`; no body |
| `CreateOnlineManager` `0x104A29E0` | **Declared; no body**; its call is in `protomain.cpp` (startup) |

**Dependencies:** `ISimulationDriver`, `SimulationDriverTimeData`, `SimulationDriver* inner`, `ChannelManager`, `ConsoleManagerPeer`, `NetSession` at `+0x70`.  
**Callers:** `proto::CreateOnlineManager` (startup); `GameLinkManager` reads `+0x70`; slots are invoked through the simulation vtable.  
**Verdict:** **partial**. It is a driver wrapper, not a complete online manager. Without `Slot4`, `UpdateClock`, `Shutdown`, `ChannelManager` and the session, it cannot be used as a network system.

**Counting note:** the unit's initial worklist says "2 match, 0 nonmatching"; a later pass (`sn-ei`) lists constructor, destructor and slots. There is no single final per-unit count, so this is treated as partial state, not a closed figure.

### 4.2 `OLS` / `OLSPeer` / session and lobby

**Files:** `src/code/engine/online/ols.{h,cpp}`, `olsgame.{h,cpp}`, `olsuser.cpp`, `olsprofile.{h,cpp}`, `olsstorage.{h,cpp}`, `olsfriend.*`, `olswrappers.cpp`, `onlinescripting.cpp`; `src/code/proto/olsproto.*`, `olsgameproto.*`.

| Class | State |
|---|---|
| `OLS` (singleton `0x108441E0`) | **Only `Get()` reconstructed.** The layout exists partially in `ols.h`; `Update()`, callbacks and singleton creation are missing. |
| `OLSGame` | Many methods reconstructed (`TryRequestState`, `Slot2`, `Slot3`, `Slot6`, `Slot9`, `SetSetting`, `RequestState`). **Missing `OnMessage` 0x10887FC0, slots 4/5/7/8 and `OnFinished` 0x108880C0.** |
| `OLSUserList` (`engine::OLSUser`) | Mostly reconstructed; `AddUser`/`OnFlags`/`OnValue` NONMATCHING; **`Start` 0x108878B0 open**; `OLSUserProto` not reconstructed. |
| `OLSProfile` | Partly reconstructed; `OLSProfileProto` yes; `OLSUserProto`/`OLSGameProto` partial. |
| `OLSStorage` | **Almost complete** (constructor, queue, messages, `ReadLine` EQUIVALENT); platform operations are pure virtual. |
| `OLSPeer` | **Not reconstructed** (0x108E7E80–0x108E9CF0); only script wrappers. |
| `SyncManager` / `DesyncHunterOnline` | **Not reconstructed**; depend on `ConsoleManager`, which is also missing. |
| `ConsoleManager` | **Not reconstructed**; base for `OLSPeer`, `SyncManager`, `DesyncHunterOnline`. |

**Dependencies:** `OLS` is the callback hub; `OLSGame` uses `RequestGameState`/`RequestTransition`; `OLSStorage` implements the global file queue; `OLSPeer` and `SyncManager` hang off `ConsoleManager`.  
**Verdict:** **no functional lobby/session.** UI/profile/storage pieces exist, but the OLS core and peers are not reconstructed.

### 4.3 `SyncObject`

**File:** `src/code/engine/system/syncobject.{h,cpp}`; worklist `sn-fr`.

| Function | State |
|---|---|
| `SyncObject` ctor/dtor, `OnPress`, `OnRelease`, `Reset`, `ResetIf` | Reconstructed; clear behaviour |
| `SyncObjectDefault` ctor, `Press`, `Release`, `SetFactory`, getters | Reconstructed |
| `SyncObjectFactoryDefault` ctor/dtor, `GetObject`, `Update`, `IndexOf` | Reconstructed |
| `InitSyncFactories`, `StartSync` `0x10833380`, `StopSync` `0x108333D0` | Reconstructed; `StartSync` opens `NetSession` with kind **4** |
| `RegisterSyncFactory` / `UnregisterSyncFactory` | Reconstructed |
| `ApplySyncMessage` `0x10833450` | **NONMATCHING (62%)**; same behaviour, not compiled in the matching build |
| `SetSyncEnabled` / `GetSyncEnabled` | Reconstructed |
| `RecordedInputsReady` | Reconstructed |
| `SendSyncMessage` `0x10833790` | Reconstructed; sends through `gSyncConnection` or applies locally |
| `UpdateSync` `0x108336F0` | Reconstructed; drains `gSyncConnection->Receive` |
| `LoadRecordedInputs` / `SaveRecordedInputs` | **Not reconstructed**; remain in the engine |

**Dependencies:** `NetSession`/`NetConnection`, `NetSystem`, `gSyncOwner`, `gSyncConnection`, `gReplay`.  
**Callers:** `SyncObjectDefault::Press/Release` → `OnPress/OnRelease` → `SendSyncMessage`; `ReplaySimDriver::Slot4` → `RecordedInputsReady`; **no reconstructed `StartSync`/`UpdateSync` callers**.  
**Verdict:** **the best reusable piece.** It is self-contained and clear, but using it live requires implementing the transport and calling `StartSync`/`UpdateSync` from a loop. `ApplySyncMessage` can be reimplemented functionally even if it does not match byte-for-byte.

### 4.4 `TimeSync` / `OnlineClock` / `OnlineSync`

| Element | State |
|---|---|
| `TimeSync::Callback` (RTTI 0x111DC228) | Only the type; **no reconstructed `TimeSync` class** |
| `OnlineClock::GetTime` `0x108C7340` | **Declared in `gamestate.cpp`; no body** |
| `OnlineSync` (0x108C7750 and methods 0x108C7B50, 0x108C7BE0, 0x108C7820, 0x108C7860, 0x108C77C0) | **Declared; no body** |
| `ISimulationDriver::Slot7` | Reconstructed; calls `gOnlineClock->GetTime(&flag)` |
| `RequestTransition`, `IsMessageSent`, `CancelStateRequest`, `StartNextRequest` | Reconstructed; call `Unknown108C7750()->...` |

**Dependencies:** `OnlineClock` is used by the simulation driver; `OnlineSync` by game state transitions.  
**Verdict:** **a central piece is missing.** Without `OnlineClock`/`OnlineSync`, online state transitions and the network clock do not exist in the decompilation. A plugin can hook/no-op these points, but there is no reusable code.

### 4.5 `DesyncHunter` / `DesyncChecker`

**Files:** `src/code/engine/system/desynchunter.{h,cpp}`, `desyncchecker.cpp`; worklist `sn-bj`.

| Function | State |
|---|---|
| `DesyncHunter` ctor/dtor, `Start`, `Stop`, `Update` | Reconstructed |
| `SetInstance`, `GetInstance`, `UpdateInstance`, `StartInstance`, `StopInstance` | Reconstructed |
| `PushMode`, `PopMode`, `GetMode`, `RecordThread` | Reconstructed |
| `DesyncChecker` ctor/dtor, `Append`, `Reset`, `GetValue` | Reconstructed |
| `DesyncChecker::Verify` `0x108BFBD0` | **Empty body in source**; unclear whether the original is also empty **[UNVERIFIED]** |
| `DesyncPrintf`, `DesyncChecksum`, variable registration, `FormatRandomState` | Reconstructed |
| `gDesyncSettings` | **Declared extern; not defined in source** |
| `DesyncHunterOnline` | **Not reconstructed**; depends on `ConsoleManager` |

**Dependencies:** `DesyncChecker` uses `content::NamedMetaType::Find("GameObject")` and `math::gRandom`; `DesyncHunter` is used by `ReplaySimDriver` and state transitions.  
**Verdict:** **very useful for QA/determinism**, but incomplete: empty `Verify` and undefined settings. It is not a ready desync detector.

### 4.6 `NetMessage` and the transport layer

**Files:** `src/code/engine/netmessage.{h,cpp}`, `gamelink.{h,cpp}`, `networksockets.cpp`; worklist `sn-w`, `sn-dn`, `sn-bj`.

| Element | State |
|---|---|
| `NetMessageSent`, `NetMessageAllocator`, `NetMessageAllocatorDynamic` | Declared; only base constructor `0x10853BA0` reconstructed |
| `NetMessagePusher` / `NetMessagePuller` | **Not reconstructed**; bases of `SyncManager`/`MultiTailQueue` |
| `NetSession` `Open` 0x108C31D0 / `Close` 0x108C3670 | **Declared; no body** |
| `NetConnection` `Send` 0x108C3F90 / `Receive` 0x108C4030 / `IsReady` 0x108C40C0 | **Declared; no body** |
| `NetSystem` / `GetNetSystem` 0x108C4260 | Declared/mapped; no body |
| `NetLink` / `NetPacket` | Declared; used by `GameLinkManager` |
| `GameLinkManager` | Mostly reconstructed; constructor NONMATCHING; uses `NetSession::Open(6,...)` and `NetConnection::Send/Receive` |
| `networksockets.cpp` | Reconstructed Winsock layer (handles, bind/connect/send/receive, async DNS); **`GetLocalNetworkInfo` 0x1093F920 not done** |
| `ConsoleManager`/`Channel`/`MultiTailQueue` | **Not reconstructed** (~270 functions, 0x108C2F60–0x108C5870 and 0x108C7210–0x108C8D20) |

**Dependencies:** `SyncObject` and `GameLinkManager` use `NetSession`/`NetConnection`; `GameLinkManager` opens kind **6**, `SyncObject` opens kind **4**.  
**Verdict:** **the transport interface exists, the implementation does not.** `networksockets` is a good low-level base, but it does not replace `NetSession`.

### 4.7 `CoOpBehaviour` / `CoOpMessage`

**Files:** `src/code/proto/coopbehaviour.{h,cpp}`, `coopstateinstances.cpp`, `src/code/proto/messages/coopmessages.h`; worklist `sn-by`.

| Function | State |
|---|---|
| `GetClassInfo` `0x104DAE80` | Reconstructed |
| Constructor `0x104DAEC0` | Reconstructed; listens to types **0x30**, **2** and **4** (`Listen`/`Listen2`) |
| `Slot1` `0x104DAF20` | Reconstructed (empty) |
| `GetMembers` `0x104DB780` | Reconstructed; three members: `respawnPendingState`, `partnerDeadState`, `pairingVolumeState` |
| `Update` `0x104DB8E0` | **Not reconstructed** (0x5A bytes) |
| `OnMessage` `0x104DB980` | **Not reconstructed** (0x183 bytes) |
| Destructor `0x104DB960` | **Not reconstructed** (0x16 bytes) |
| `RespawnState` / `PlayerDeadState` / `PairingVolumeState` | `ClassInfo`/`GetMembers` reconstructed, but **no reflected members or recovered layout** |
| `CoOpMessage` base + `GetPlayerDeathState` + `GetRespawnState` | Reconstructed |
| `CoOpMessagePlayerDead`, `SetRespawnState`, `SetPairingVolumeState` | Names seen in binary strings in the earlier audit; **no source or symbols in this repo** |
| `MetaBehaviourCreatorImpl<CoOpBehaviour>` `0x10294380` / `RegisterMetaBehaviourCreator` `0x1029BA60` | Explicit instantiations; the registration call site is not visible in reconstructed source |

**Dependencies:** `engine::Behaviour` (messages), `BehaviourController` (dispatch/update), `CoOpMessage`, respawn/pairing states.  
**Callers:** `BehaviourController` calls `Update`/`OnMessage`; the behaviour is created by `MetaBehaviourCreatorImpl<CoOpBehaviour>`.  
**Verdict:** **a shell, not functional.** Without `Update`, `OnMessage`, destructor, layouts and messages, the respawn/pairing logic cannot run.

### 4.8 `Replay` / `ReplaySimDriver`

**Files:** `src/code/engine/online/replay.{h,cpp}`, `replaysimdriver.cpp`; worklist `sn-w`.

| Function | State |
|---|---|
| `Replay` ctor/dtor, `SetState`, `SetMode`, `StartDriver`, `Stop`, `Record`, `Play`, `PlayDesync`, `Resume`, `Reset`, `Quit`, scripts | Reconstructed; one `EQUIVALENT` in `Update` |
| `ReplaySimDriver` ctor/dtor, `Slot1/2/3/4/5/6/9`, `Stop`, `AddListener`, `RemoveListener`, `NotifyListeners`, `PollListeners` | Reconstructed |
| `ReplaySimDriver::Start` `0x10900960` | **Not reconstructed** |
| `ReplaySimDriver::Query` `0x109004B0` | **Not reconstructed** |
| `VerifyChecksum` `0x10900690` / `RecordChecksum` `0x109007C0` | **Not reconstructed** (the header says "left to the engine") |
| `LoadRecordedInputs` / `SaveRecordedInputs` | **Not reconstructed**; the driver calls them |

**Dependencies:** `DesyncHunter`, `inner` SimulationDriver, `content::SaveStream`, `ReplayReader`.  
**Callers:** `Replay_*` scripts, `PauseState`, simulation/real-time tasks.  
**Verdict:** **the `Replay` class is advanced and useful as a determinism test base**, but the driver does not start or verify checksums yet. It is not ready to run a full replay.

### 4.9 Online conditions and tracks

**Files:** `src/code/engine/fight/onlineconditions.cpp`, `src/code/proto/fightitems/online/online_conditions.cpp`, `online_tracks.cpp`.

| Element | State |
|---|---|
| `OnlineNumPlayersCondition` | Reconstructed; `Evaluate` uses `controller::InputSystem::Get()->slotCount` (local players, **not OLS**) |
| `OnlinePlayerIndexCondition` | Reconstructed; `Evaluate` uses `GetControllingPlayer` and `CompareInts` |
| `OnlineTestForRespawnStateCondition` | Properties/constructor; **`Evaluate` 0x102650E0 not reconstructed** |
| `DistanceToPlayerCondition` | Properties/constructor; **`Evaluate` not reconstructed** |
| `OnlineTestForPlayerDeathCondition` | Properties/constructor; **`Evaluate` not reconstructed** |
| `OnlinePlayerTypeCondition` | Properties/constructor; **`Evaluate` not reconstructed** |
| `OnlineFlagPlayerDeathTrack` | Properties/constructor/factory and death `CreateAction` reconstructed |
| `OnlineSetRespawnStateTrack` / `OnlineSetPairingVolumeStateTrack` | Properties/constructor/factory; **`CreateAction` 0x10266390 / 0x102663E0 not reconstructed** |
| `OnlineSyncLocalWorldCellStreamingTrack` | Properties/constructor/factory; streaming action **not reconstructed** |

**Verdict:** **metadata and registration are reusable; behaviour is incomplete.** The two `engine/fight` conditions work locally; respawn/distance/death/type conditions and pairing/respawn actions are missing.

### 4.10 Socket layer

**[FACT]** `networksockets.cpp` reconstructs a full Winsock layer: handle table, `bind`/`connect`/`send`/`receive`, `select`, async DNS, options, startup/cleanup. The worklist notes it was compiled without `/arch`, with local Winsock/Win32 declarations and `#pragma alternatename`.

**Verdict:** **useful as a low-level transport base**, but it is not a session or a channel; it would have to be wrapped in `NetSession`/`NetConnection`.

---

## 5. Summary table of functions and state

> "Match" per the worklist; "Partial" = some functions match, others are missing; "No body" = declared/mapped, not reconstructed. Per-unit counts are not invented when the repo does not provide them.

| Subsystem | Key functions | Reconstructed | Matching | Missing | Verdict |
|---|---|---|---|---|---|
| `CreateOnlineManager` | 0x104A29E0 | No | Mapped | Body | Entry point (wrong name; it is `SoundManager`) |
| `OnlineManager` | 0x108BF160, 0x108BF880, slots | Partial | Ctor/dtor/slots 1,2,3,5,9 | `Slot4`, `UpdateClock`, `Shutdown`, `ChannelManager` | Partial wrapper |
| `OLS` | 0x108441E0 | Only `Get` | `Get` | Layout, `Update`, callbacks, creation | Core missing |
| `OLSGame` | 0x10887DA0 etc. | Partial | Several methods | `OnMessage`, slots 4/5/7/8, `OnFinished` | Partial |
| `OLSUserList` | 0x10886A70–0x108879EA | Partial | Most | `Start`, `OLSUserProto` | Partial |
| `OLSStorage` | 0x108C5950–0x108C6700 | Yes | Almost all; `ReadLine` EQUIVALENT | Platform virtuals | Reusable with platform |
| `OLSPeer` | 0x108E7E80–0x108E9CF0 | No | — | All | Blocked by `ConsoleManager` |
| `SyncManager` / `DesyncHunterOnline` | — | No | — | All | Blocked by `ConsoleManager` |
| `SyncObject` | 0x10833330–0x10833BC0 | Yes | Core; `ApplySyncMessage` NONMATCHING 62% | `Load/SaveRecordedInputs`, callers | **Best piece** |
| `TimeSync` | — | No | — | All | Missing |
| `OnlineClock` | 0x108C7340 | No | — | `GetTime` | Missing |
| `OnlineSync` | 0x108C7750 + methods | No | — | All | Missing |
| `DesyncHunter` | 0x108215E0 etc. | Yes | Methods | `DesyncHunterOnline` | Useful for QA |
| `DesyncChecker` | 0x108BFA80 etc. | Partial | Ctor/Append/Reset/checksums | Empty `Verify`; undefined `gDesyncSettings` | Partial |
| `NetMessage` | 0x10853BA0 | Only base ctor | Ctor | Allocators, pushers/pullers | Almost nothing |
| `CoOpBehaviour` | 0x104DAEC0 etc. | Partial | ClassInfo/ctor/Slot1/GetMembers | `Update`, `OnMessage`, dtor, layouts | Shell |
| `CoOpMessage` | base vftable 0x10DDDE10 | Partial | Base + 2 `Get*` messages | `PlayerDead`, `SetRespawnState`, `SetPairingVolumeState` | Partial |
| `Replay` | 0x108C27A0–0x108C2F60 | Yes | Almost all; 1 EQUIVALENT | — | Reusable |
| `ReplaySimDriver` | 0x10900260–0x10900EC0 | Partial | Ctor/dtor/slots/Stop/listeners | `Start`, `Query`, checksums, inputs | Incomplete |
| `NetSession`/`NetConnection` | 0x108C31D0/0x108C3F90/0x108C4030 | No | Mapped | Implementation | Transport seam |
| `ConsoleManager`/`Channel` | 0x108C2F60–0x108C5870 | No | — | ~270 functions | Main blocker |
| `networksockets` | 0x1093E5D0–0x1093FD20 | Yes | Most | `GetLocalNetworkInfo` | UDP/TCP base |
| Online conditions (`engine`) | 0x108B0840 etc. | Partial | 2 conditions | 3 `Evaluate` | Partial |
| Online tracks/actions | 0x10264E40–0x102689E0 | Partial | Properties/factories | `Evaluate`/actions | Partial |

---

## 6. Can `CoOpBehaviour` work without `OnlineManager`?

### 6.1 What the code says

**[FACT]**

- `CoOpBehaviour` **does not include or reference `OnlineManager` or `OLS`** in its reconstructed header/source.
- Its constructor only calls `engine::Behaviour` and registers listening for message types `0x30`, `2` and `4`.
- Its only members are three pointers to `RespawnState`, `PlayerDeadState` and `PairingVolumeState`.
- `BehaviourController` (which dispatches `Update` and `OnMessage`) is reconstructed and works.
- `OnlineNumPlayersCondition` and `OnlinePlayerIndexCondition` use `InputSystem`/`GetControllingPlayer`, **not** OLS.
- `CoOpMessage` is a type `0x30` message that travels through the `GameObject` message system.

### 6.2 What is missing

**[FACT]**

- `CoOpBehaviour::Update` (0x104DB8E0), `OnMessage` (0x104DB980) and the destructor (0x104DB960) are not reconstructed.
- `RespawnState`, `PlayerDeadState` and `PairingVolumeState` have an empty `GetMembers`: **their layout was not recovered**.
- `CoOpMessagePlayerDead`, `CoOpMessageSetRespawnState` and `CoOpMessageSetPairingVolumeState` are missing.
- There is no evidence of how the partner is created/assigned or where inter-player messages come from.

### 6.3 Conclusion

**[HYPOTHESIS]** `CoOpBehaviour` **does not need `OnlineManager` to exist or to dispatch local messages**; it could work in a loopback scenario if its methods are reconstructed, the layouts recovered and messages injected. But **it does not work as-is** and **provides no network**: cross-machine transmission would still be missing.

`OnlineManager` is needed for the network clock/session and driver stepping; `CoOpBehaviour` is game logic, not transport.

---

## 7. Is there an abstraction to replace Quazal?

### 7.1 Short answer

**Yes, at the `ChannelManager`/`Channel` level** (the `NetSession`/`NetConnection` names in `gamelink.h` are guesses).  
**No, at the full `OLS`/Quazal level.**

### 7.2 Detail

**[FACT]** Ghidra confirms that `OnlineManager+0x70` is a `ChannelManager` (0x7C bytes), created in `FUN_108BF1B0`. Its `Open` method (0x108C31D0) creates 0x28-byte entries, a `MultiTailQueue` (0x18) and a `Channel` (4). The returned value is used as the connection for `Send`/`Receive`.

**[FACT]** The observed interface is small:

- `ChannelManager::Open(kind, unknown, capacity)` → `Channel*`
- `Channel::Send(target, channel, data, length, unknown)`
- `Channel::Receive(buffer, capacity, &length, &flag, &source)`
- `Channel::IsReady()`
- `NetSystem` with `unknown20` as `target`

**[FACT]** Known uses:

- `StartSync` (0x10833380) opens kind **4** and stores the `Channel*` in `gSyncConnection`.
- `SendSyncMessage` uses `target = GetNetSystem()->unknown20`, `channel = type`, 8 bytes, flag 1.
- `UpdateSync` drains `Receive` and uses `source` as the machine mask.
- `GameLinkManager::Connect` opens kind **6**, channel `0xFF`; uses `Receive`/`Send`.

**[FACT]** The `ChannelManager`/`Channel` implementations **are not reconstructed**; the worklist marks `ConsoleManager`, `Channel` and `MultiTailQueue` as not done (~270 functions, 0x108C2F60–0x108C5870 and 0x108C7210–0x108C8D20).

### 7.3 Implication

**[HYPOTHESIS]** A custom `ChannelManager`/`Channel` can be written over UDP/Steam and installed at `OnlineManager+0x70` (or `ChannelManager::Open` intercepted) so `SyncObject` and `GameLinkManager` work without Quazal. That avoids rebuilding the Quazal backend and preserves `SyncObject`, `Replay` and `DesyncHunter`.

But it **does not avoid**:

- implementing the semantics of `kind` 4/6, `target`, `channel`, flags and `source`;
- creating/destroying the session and lobby (`ConsoleManagerPeer`/`OLSPeer`);
- replacing `OnlineClock`/`OnlineSync` for state transitions;
- reimplementing `CoOpBehaviour` and its messages;
- deciding whether `OLSGame`/`OLSUserList` are used or bypassed.

**Conclusion:** the `ChannelManager`/`Channel` seam is the most realistic way to **avoid Quazal**, not to avoid the entire online system.

---

## 8. Entry points for an ASI plugin

> Addresses use image base `0x10000000`; at runtime: `prototypeenginef.dll base + (addr - 0x10000000)`. Nothing was hooked; only candidates are identified.

| Address | Function | Potential use |
|---|---|---|
| `0x1085EFB0` | `EngineInitialize` (export) | Early plugin bootstrap; sees launch arguments |
| `0x1085EB20` | `EngineService` (export) | Main loop; already used by other projects |
| `0x108BF5E0` | Online subsystem init (called from `EngineInitialize` at `0x1085F47E`) | **Real startup hook**: creates `OnlineManager` and `OnlineSync` |
| `0x108E7E80` | `OLSPeer` constructor | Host/join entry; calls the session bootstrap |
| `0x108BF1B0` | Host/client session bootstrap | **Main hook**: creates `ChannelManager`, `ConsoleManagerPeer`, channels, `StartSync` and `DesyncHunterOnline` |
| `0x11318530` | `gOnlineManager` (global) | Manager access; `+0x70` is the `ChannelManager` |
| `0x108BF160` | `OnlineManager` constructor | Replace/wrap the online driver |
| `0x108BF780` | `OnlineManager::Slot4` | **Network pump**: clock, send, receive and dispatch |
| `0x108BF520` | `OnlineManager::UpdateClock` | Online clock |
| `0x108BF690` | `OnlineManager::Shutdown` | Session close |
| `0x108C31D0` | `ChannelManager::Open` | **Transport seam**: intercept kind 4/6 |
| `0x108C3F90` | `Channel::Send` | Send through a custom transport |
| `0x108C4030` | `Channel::Receive` | Receive through a custom transport |
| `0x108C40C0` | `Channel::IsReady` | Connection state |
| `0x10833380` | `StartSync` | Start input sync; opens kind 4 |
| `0x108333D0` | `StopSync` | Stop sync |
| `0x10833790` | `SendSyncMessage` | Send press/release |
| `0x108336F0` | `UpdateSync` | Drain sync messages (called from `SimulationTick`) |
| `0x10833450` | `ApplySyncMessage` | Apply sync message (NONMATCHING) |
| `0x108C7340` | `OnlineClock::GetTime` | Network clock (implemented in the binary) |
| `0x108C7750` | `OnlineSync` accessor | State-transition synchronisation |
| `0x108C7B50` / `0x108C7BE0` / `0x108C7820` | `OnlineSync` methods | Transition start/end/query |
| `0x104DAEC0` | `CoOpBehaviour` constructor | Create the behaviour on a GameObject |
| `0x104DB8E0` | `CoOpBehaviour::Update` | Per-frame logic; recovered with Ghidra |
| `0x104DB980` | `CoOpBehaviour::OnMessage` | Co-op message handling; recovered with Ghidra |
| `0x10294380` | `MetaBehaviourCreatorImpl<CoOpBehaviour>::CreateBehaviour` | Create an instance from the behaviour system |
| `0x1029BA60` | `RegisterMetaBehaviourCreator<CoOpBehaviour>` | Register the factory |
| `0x10821520`–`0x10821560` | `DesyncHunter` statics | Start/stop/update QA |
| `0x108216A0` | `DesyncHunter::Update` | Compare state |
| `0x108BFA80` | `DesyncChecker` ctor | Create checker |
| `0x108C2D30` | `Replay` ctor | Create replay |
| `0x10900960` | `ReplaySimDriver::Start` | Start recording/replay |
| `0x108441E0` | `OLS::Get` | Access the OLS singleton |
| `0x10887DA0` | `OLSGame::TryRequestState` | Session state flow |

**Note:** `0x104A29E0` is `SoundManager`, not online. `CoOpBehaviour::Update/OnMessage` were missing from the source but exist in the binary and were recovered with Ghidra.

---

## 9. Reusable and to-be-implemented subsystems

### 9.1 Reusable as reference or a functional piece

| Subsystem | Level | Why |
|---|---|---|
| `SyncObject` | **High** | Complete press/release logic, factories, reset and masks; only transport and callers are missing |
| `networksockets` | **High** | Complete Winsock layer; base for a UDP/TCP `NetSession` |
| `Replay` (manager) | **Medium-high** | All methods and scripts; the driver needed to start/record is missing |
| `OLSStorage` | **Medium** | Global file queue complete; platform operations missing |
| `DesyncHunter`/`Checker` | **Medium** | QA base; empty `Verify` and undefined settings |
| `BehaviourController` | **High** | Behaviour dispatch/update reconstructed; `CoOpBehaviour` can hang off it |
| `OLSUserList` / `OLSGame` | **Medium-low** | User/session structures; key methods missing |
| `OnlineNumPlayers` / `OnlinePlayerIndex` | **Medium-low** | Work locally without OLS; useful for simple conditions |
| `GameLinkManager` | **Medium** | `NetSession` kind 6, send/receive and packets; not co-op |
| `CoOpBehaviour` shell | **Low** | Only constructor/reflection; all logic missing |

### 9.2 To implement / reconstruct

1. **Network layer:** `ConsoleManager`, `Channel`, `MultiTailQueue`, `LoopbackConnection`, `ChannelConnection`, `NetMessagePusher/Puller` (~270 functions).
2. **Peers:** `OLSPeer`, `SyncManager`, `DesyncHunterOnline`, `NetSystemLink*`.
3. **Clock/state sync:** `TimeSync`, `OnlineClock`, `OnlineSync`, `OnlineManager::Slot4/UpdateClock/Shutdown`.
4. **CoOpBehaviour:** `Update`, `OnMessage`, destructor, layouts of `RespawnState`/`PlayerDeadState`/`PairingVolumeState`, messages `PlayerDead`/`SetRespawnState`/`SetPairingVolumeState`.
5. **Replay:** `ReplaySimDriver::Start`, `Query`, `VerifyChecksum`, `RecordChecksum`, `LoadRecordedInputs`/`SaveRecordedInputs`.
6. **Online conditions/actions:** `Evaluate` for respawn/distance/death/type and `CreateAction` for pairing/respawn/streaming.
7. **OLS/session:** complete `OLS`, singleton creation, platform profiles/users/game, lobby/matchmaking.
8. **UI/flow:** host/join, player list, connection states, error handling.

---

## 10. Technical difficulties

1. **The network layer is the main blocker.** The worklist calls it "not done (judged too heavy for the time)"; ~270 functions of `ConsoleManager`/`Channel`/sync. Without it, `OLSPeer`, `SyncManager` and `DesyncHunterOnline` cannot be reconstructed.
2. **Circular dependency:** `OLSPeer`/`SyncManager`/`DesyncHunterOnline` derive from `ConsoleManager`; if `ConsoleManager` is missing, those classes are blocked.
3. **`OLS` is almost entirely unreconstructed.** Only `Get()`; the layout is partial, singleton creation is not in the source and callbacks are missing.
4. **`TimeSync`/`OnlineClock`/`OnlineSync` are completely missing.** They are used by the simulation driver and state transitions; without them there is no online clock/state.
5. **`CoOpBehaviour` is a shell.** 0x5A + 0x183 + 0x16 bytes of logic, state layouts and several messages are missing. The respawn/pairing logic cannot be tested.
6. **`DesyncChecker::Verify` is empty** and `gDesyncSettings` is not defined in source; the desync detector is not operational.
7. **`ReplaySimDriver` does not start.** `Start`, `Query` and checksums are missing; `Load/SaveRecordedInputs` remain in the engine.
8. **"Match" is not "used".** Some functions match but have no reconstructed callers (`StartSync`, `UpdateSync`); the code can compile and never execute.
9. **Quazal is dead.** `prototypepclive.quazal.net` resolves to 127.0.0.1; there is no backend. Replacing it requires a `NetSession` shim or protocol emulation.
10. **The hybrid is not validated online.** It reaches free roam; there is no evidence of a two-client session or that the online path runs.
11. **Addresses are tied to the build.** `PROTOPC_2025` and the DLL hash; any update invalidates the hooks.
12. **License and distribution.** Source-only, non-commercial, no hybrid binaries; a mod containing reconstructed code cannot be packaged. This project's rule forbids copying third-party code.

---

## 11. Five next research goals (by priority)

### 1. Trace online startup and session creation — **resolved with Ghidra**

**Done:** `EngineInitialize` → `0x108BF5E0` initialises `OnlineManager` and `OnlineSync`; `OLSPeer` (0x108E7E80) → `0x108BF1B0` starts the host/client session. `0x104A29E0` is `SoundManager`, not online.  
**Pending:** confirm at runtime when `OLSPeer` is instantiated in single-player and whether `0x108BF1B0` is called without a session.

### 2. Reverse the `ChannelManager`/`Channel` seam

**Question:** what is the semantics of `kind` 4/6, `target`, `channel`, `flag` and `source`? Can a UDP/Steam shim satisfy `SyncObject` and `GameLinkManager`?  
**Method:** `ChannelManager::Open` 0x108C31D0, `Channel::Send` 0x108C3F90, `Receive` 0x108C4030, `IsReady` 0x108C40C0, `GetNetSystem` 0x108C4260; compare with `GameLinkManager::Connect` (kind 6) and `StartSync` (kind 4).  
**Exit criterion:** interface specification and an 8-byte loopback test (`SyncMessage`).

### 3. Reconstruct `CoOpBehaviour` logic

**Question:** what do `Update` (0x104DB8E0), `OnMessage` (0x104DB980) and the destructor (0x104DB960) do? What layouts do the three states have? What messages do they build/consume?  
**Method:** Ghidra over 0x104DAD00–0x104DBC00; RTTI/strings for `CoOpMessage*`; cross-check with `BehaviourController` and `online_tracks`.  
**Exit criterion:** table of messages, states and transitions; definitive answer on whether it needs `OnlineManager`.

### 4. Complete the determinism harness (`Replay` + `DesyncHunter`)

**Question:** is the game deterministic between two instances with the same inputs? What diverges first?  
**Method:** complete `ReplaySimDriver::Start/Query/VerifyChecksum/RecordChecksum` and `Load/SaveRecordedInputs`; use `Replay_Record`, `Replay_Replay`, `Replay_RecordDesync`, `Replay_ReplayDesync`; two isolated instances.  
**Exit criterion:** a short recording that replays identically in two instances, or the first divergence point documented.

### 5. Decide the OLS/peer strategy: shim vs reimplementation

**Question:** is it better to implement `ConsoleManager`/`Channel`/`OLSPeer`/`SyncManager`, or is a `NetSession` shim enough and OLS bypassed?  
**Method:** audit `OLSPeer` 0x108E7E80–0x108E9CF0, `SyncManager`, `DesyncHunterOnline`, `NetSystemLink*`, `TimeSync`/`OnlineClock`/`OnlineSync`; estimate how many functions co-op actually uses.  
**Exit criterion:** documented architectural decision (low-level shim vs OLS reimplementation) and a minimum function list.

---

## 12. Limitations of this audit

- **The initial audit was repository-based; addendum 0.b did inspect the binary with Ghidra 12.1.4.** Addresses, states and original names come from `config/PROTOPC_2025/symbols.txt`, `classes.tsv`, `rtti.tsv`, `docs/engine_worklist.md` and the repository source; the addendum corrects names the binary disproves.
- **The project was not compiled or executed**; neither the hybrid nor a two-client session was tested.
- **Could not inspect** (not in the repository):
  - `ConsoleManager`, `Channel`, `MultiTailQueue`, `LoopbackConnection`, `ChannelConnection`.
  - `OLSPeer`, `SyncManager`, `DesyncHunterOnline`.
  - `OnlineClock`, `OnlineSync`, `TimeSync` (only RTTI/declarations).
  - `NetSession`/`NetConnection` (only declarations).
  - `CoOpBehaviour::Update`, `OnMessage`, destructor.
  - `ReplaySimDriver::Start`, `Query`, checksums.
  - The full `OLS` layout and singleton creation.
- **The full worklist was not read** (8,766 lines); relevant sections were read (`onlinemanager`, `onlinescripting`, `sn-t`, `sn-w`, `sn-bj`, `sn-by`, `sn-er`, `sn-dy`, `sn-fr`, `sn-ei`, `sn-dn`, `sn-fs`). Other sections may contain additional detail.
- **Empty `DesyncChecker::Verify`** may be a reconstruction stub or the real behaviour; not verified against the binary.
- **`NetSession`/`NetConnection` semantics** (target/channel/kind) are unknown; inferred from callers only.
- **The license is not legal advice.** Before reusing any fragment, review `LICENSE.md`/`LEGAL.md` and seek advice if distributing anything.

---

## Appendix — addresses and artifacts cited

| Artifact | Address / location |
|---|---|
| `proto::CreateOnlineManager` | 0x104A29E0 |
| `gOnlineManager` | 0x11318530 |
| `OnlineManager` ctor / dtor | 0x108BF160 / 0x108BF880 |
| `OnlineManager::Slot4` | 0x108BF780 |
| `OnlineManager::UpdateClock` | 0x108BF520 |
| `OnlineManager::Shutdown` | 0x108BF690 |
| `NetSession::Open` / `Close` | 0x108C31D0 / 0x108C3670 |
| `NetConnection::Send` / `Receive` / `IsReady` | 0x108C3F90 / 0x108C4030 / 0x108C40C0 |
| `GetNetSystem` | 0x108C4260 |
| `StartSync` / `StopSync` | 0x10833380 / 0x108333D0 |
| `SendSyncMessage` / `UpdateSync` | 0x10833790 / 0x108336F0 |
| `ApplySyncMessage` | 0x10833450 (NONMATCHING) |
| `OnlineClock::GetTime` | 0x108C7340 |
| `OnlineSync` accessor / methods | 0x108C7750 / 0x108C7B50 / 0x108C7BE0 / 0x108C7820 |
| `CoOpBehaviour` ctor / Update / OnMessage / dtor | 0x104DAEC0 / 0x104DB8E0 / 0x104DB980 / 0x104DB960 |
| `MetaBehaviourCreatorImpl<CoOpBehaviour>::CreateBehaviour` | 0x10294380 |
| `RegisterMetaBehaviourCreator<CoOpBehaviour>` | 0x1029BA60 |
| `DesyncHunter::Update` / `Start` / `Stop` | 0x108216A0 / 0x10821680 / 0x10821690 |
| `DesyncChecker` ctor | 0x108BFA80 |
| `Replay` ctor | 0x108C2D30 |
| `ReplaySimDriver::Start` / `Query` | 0x10900960 / 0x109004B0 |
| `ReplaySimDriver::VerifyChecksum` / `RecordChecksum` | 0x10900690 / 0x109007C0 |
| `OLS::Get` | 0x108441E0 |
| `OLSGame::TryRequestState` | 0x10887DA0 |
| `GameLinkManager::Connect` | 0x10853B40 |
| `EngineInitialize` / `EngineService` | 0x1085EFB0 / 0x1085EB20 |

**Main sources:** `README.md`, `LICENSE.md`, `LEGAL.md`, `CONTRIBUTING.md`, `docs/progress.md`, `docs/engine_worklist.md`, `docs/plan.md`, `config/PROTOPC_2025/symbols.txt`, `classes.tsv`, `rtti.tsv`, and the source under `src/code/engine/online/`, `src/code/engine/system/`, `src/code/proto/coop*`, `src/code/proto/fightitems/online/`.

> **License note:** this document is a research review. The quoted license terms and any code/addresses from the audited project remain under that project's license; nothing from it is relicensed here.
