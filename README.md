<div align="center">

# Prototype Coop

**Online co-op for Prototype (2009) — plus a launcher, game settings and fixes.**

![Status](https://img.shields.io/badge/status-early%20public%20test-orange)
![Players](https://img.shields.io/badge/players-2-blue)
![License](https://img.shields.io/badge/license-MIT-green)

</div>

> ⚠️ **This is an early public test.** Expect crashes, desync, missing effects and
> unfinished features. Testing so far has been limited. Please report what
> breaks — that is how this project gets better.

> 🐞 **Found a bug or crash?** Please open an issue. Include the launcher Status
> report and your logs. Details are in [Bugs & crashes](#-bugs-crashes--logs).

---

## 🍎 Simple version

One player **hosts**. The other **joins**. The guest gets a remote Alex that
moves, fights and uses the same powers. The city itself is still simulated on
each PC, so pedestrians, traffic and ambient destruction are not shared yet.

No dedicated server, no account system, no install-time telemetry. Just Steam
lobbies and a two-player session.

---

## ✨ What you get

- **Two-player online co-op** through Steam.
- **A native launcher** that installs the mod and Steam API, and detects your
  separately installed PrototypeFix.
- **Game settings** in the launcher: video, LOD, FOV, FPS, VRAM and fixes.
- **A live lobby count** so you can see whether people are matchmaking.
- **Crash reports** with a one-click GitHub issue and a ZIP to attach.

---

## ✅ What works

| Feature | State |
|---|---|
| Two-player Steam co-op | ✅ Works |
| Host / guest lobby, search and invites | ✅ Works |
| Remote Alex model, movement and 67-joint pose | ✅ Works |
| Powers, armour, shield and disguise | ✅ Works(kinda) |
| Devastators (Tendril Barrage, Groundspike Graveyard, Critical Pain) | ✅ Replayed on the puppet |
| Damage, death and destruction | ✅ Health/death sync |
| Grabs, carries and throws | ✅ Attach/release and throw velocity |
| Hit particles, explosions, decals and ground cracks | ✅ Mostly; some materials differ |
| Co-op saves and host-to-guest save transfer | ✅ Works |
| Vehicle dents | ⚠️ Partial |
| WhipFist on the remote avatar | ⚠️ Arm/pose yes, connecting tendril weirdly |
| Shared NPCs, traffic and ambient world | ❌ Off; each PC uses its own |
| Motion trails | ❌ Disabled; they crash the puppet pose |
| Full campaign / missions | ❌ Not validated |
| 3+ players | ❌ Two-player only |

> 🧪 These results come from limited testing. If something in the “works” list
> fails for you, please report it with logs.

---

## 🧩 Mod compatibility

There is no guaranteed compatibility list beyond vanilla Prototype and
PrototypeFix 1.8.0. Test one mod at a time.

- **Likely compatible:** texture swaps and cosmetic loose-file mods that do not
  replace `xinput1_3.dll`, `binkw32.dll`, `prototypeenginef.dll`, the menu
  movies, or add another ASI loader.
- **Likely incompatible:** other co-op/online mods, controller/XInput proxies,
  menu mods, trainers, and mods that hook the same engine functions.

### Character model mods

- Model swaps that keep Alex's original skeleton and joint count are the
  safest.
- Models with a different skeleton, joint count or animation set can break the
  puppet pose.
- **Both players need the same model mod.** Models are not streamed: each PC
  builds the remote puppet from its own local files and resolves the host's
  model hash locally. If only one player has the mod, the other may see vanilla
  Alex or a fallback.

If the puppet T-poses, shows the wrong model, or the game crashes, remove the
model mod and report it with `coop.log`. The `model apply` and `pose` lines in
that log show whether the model hash and joint count matched.

---

## 📦 Requirements

- **Prototype (2009)** on Steam (App ID `10150`).
- **Windows** with the **.NET Framework 4.5 or newer** for the launcher.
  Windows 10/11 already include 4.8; on older Windows install the
  [.NET Framework 4.8 Runtime](https://dotnet.microsoft.com/en-us/download/dotnet-framework/net48).
- **PrototypeFix 1.8.0 by emoose**, installed separately from
  [Nexus Mods](https://www.nexusmods.com/prototype/mods/52), with
  `PreferLooseFiles = true`. It is **not** included in this ZIP.
- **Steam running and signed in.**
- Your own Steam copy of the game. This project does not support pirated copies.
- **Two players maximum** for now.
- TCP port **27015** free when the game starts. If another program uses it
  (commonly Apple Mobile Device Service / iTunes), the game can hang before its
  window appears. The launcher detects this and offers to stop that service.

---

## 🚀 Install

1. Download **[`PrototypeCoop.zip`](PrototypeCoop.zip)** and extract it to a
   normal folder. Do not run the launcher from inside the ZIP.
2. Install **PrototypeFix 1.8.0 by emoose** from
   [Nexus Mods](https://www.nexusmods.com/prototype/mods/52) into the game
   folder, and set `PreferLooseFiles = true` in `prototype_fix.ini`.
3. Run **`PrototypeCoopLauncher.exe`**.
4. Select the folder that contains the game executable
   (usually `Steam\steamapps\common\Prototype`).
5. Press **Install / Update**. The launcher detects PrototypeFix; it does not
   copy or replace it.
6. Start the game from the launcher, or from Steam after installing.

> Keep Steam open and signed in. The launcher needs it to obtain a compatible
> Steam API DLL.

The launcher does **not** modify or replace the game executable. It installs
the mod and builds the two menu movies from your own `art.rcf`. PrototypeFix is
a separate download from Nexus and is not bundled or redistributed.

### PrototypeFix load order

Install PrototypeFix first and let it install its own `binkw32.dll` and
`binkw32Hooked.dll`. The launcher never writes or replaces those files, or
`prototypeenginef.dll`.

The game loads our `xinput1_3.dll` proxy through its normal XInput import. The
proxy loads `prototype_coop.asi`, and loads `prototype_fix.asi` only if
PrototypeFix's own loader has not already loaded it. It checks the loaded
module list, so each ASI initializes once; `prototype_coop.asi` is a standard
ASI and works with either loader.

`PreferLooseFiles = true` in `prototype_fix.ini` is what makes the game use the
generated `art\hud` menu movies.

---

## 🎮 Start a co-op match

### Host

1. Open **ONLINE** in the main menu.
2. The mod publishes a **public co-op lobby**.
3. Wait for a guest, or use **INVITE** to invite a Steam friend.
4. Press **PLAY** to load a save, or **NEW** to start a new campaign.
5. Load the same save slot as the guest.

### Guest

1. Open **ONLINE**.
2. Press **SEARCH** and choose a host, **or** accept an invite.
3. Wait for the host to press **PLAY** or **NEW**.
4. Load the same save slot.

> **Starting a new save?** Ideally, do it after Alex Mercer escapes the
> biopsy.

> **Co-op only starts from the Online lobby.** Continue and New Game on the
> main menu are single-player; they do not arm co-op, send player state or spawn
> a remote avatar. This works even if a lobby is still open.

Both players need their own Steam account. Two instances on one PC share one
Steam account, so they cannot find each other through Steam.

---

## 🧠 How it works

- The **launcher** installs a plugin and an input proxy, fetches a compatible
  Steam API DLL, and generates the ONLINE/lobby movies from your own game
  files. It requires PrototypeFix, which you install separately from Nexus.
- The **host** creates a Steam lobby. The mod sends the host player’s movement,
  pose, powers and actions over Steam networking.
- The **guest** rebuilds the remote avatar and replays the received effects,
  attacks, grabs and decals locally.
- There is **no dedicated server** and no shared world simulation yet. Each PC
  keeps its own city, traffic and NPCs.

---

## ⚙️ Launcher

- **Play** — launch, component status, port check and the public-lobby player count.
- **Install** — choose the game folder and install/update.
- **Game settings** — video, LOD, FOV, shadow resolution, FPS, VRAM and fixes.
- **Status** — copyable diagnostics: hashes, port owner and current settings.
- **About** — credits, docs and the report button.
- **Cog** — language and Dark/Light UI. Dark mode is the default.

The player count shows **players inside public co-op lobbies**, not every
installation or solo player. It needs Steam running and refreshes once a minute.
The count pauses while the game is running, so the launcher never opens a Steam
session during gameplay.

---

## 🔒 Safety & privacy

- **No telemetry, accounts or analytics.** The mod does not collect, sell or
  send usage data, and there is no backend server.
- **No automatic uploads.** Logs and crash ZIPs stay on your PC. The launcher
  only opens a prefilled GitHub issue when you choose to report. Paths and
  17-digit Steam IDs are redacted, but logs may still contain your Steam name,
  so review the ZIP before attaching it.
- **Network access is limited to Steam and GitHub.** Steam handles the game,
  lobbies, invites and the public-lobby count. GitHub is used only to download
  the pinned Steamworks.NET API when no compatible local copy exists; the ZIP
  and DLL are verified by SHA-256. Nexus Mods and GitHub links open only when
  you click them.
- **No executable patches.** The launcher adds the mod and input proxy and
  replaces the two menu movies with versions generated from your own `art.rcf`
  (originals are backed up). PrototypeFix is installed separately and is not
  copied or modified. It does not patch or replace `prototypef.exe`, and it
  ships no game assets or Steam DLLs.
- **Local files only.** Launcher settings stay next to the launcher; game
  settings and logs stay in the game folder. Co-op saves and crash reports live
  under `Documents\Prototype Coop`. Delete them anytime.
- **No drivers or background services.** Nothing is added to Windows startup.
  If TCP port 27015 is occupied by Apple Mobile Device Service, the launcher
  asks before stopping it; that service returns after a reboot.
- **Unsigned build.** Windows SmartScreen or antivirus software may warn about
  the launcher or plugin because they are unsigned. Do not disable your
  antivirus. See [Verify the download](#verify-the-download) below.

---

## Verify the download

`SHA256SUMS.txt` contains the SHA-256 of `PrototypeCoop.zip`. To check it in
PowerShell:

```powershell
Get-FileHash .\PrototypeCoop.zip -Algorithm SHA256
```

The `Hash` value must match the hash in [`SHA256SUMS.txt`](SHA256SUMS.txt). If it
does not match, delete the file and download it again from the official release;
do not run it.

The launcher and plugins are unsigned, so Windows SmartScreen or antivirus
software may warn about them. Do not disable your antivirus. If the ZIP hash
matches and you believe a warning is a false positive, [open an
issue](../../issues) with:

- the exact flagged file name, for example `PrototypeCoopLauncher.exe` or
  `payload\mod\prototype_coop.asi`;
- that file's SHA-256;
- the ZIP hash;
- your antivirus name/version and the detection name.

---

## 🐞 Bugs, crashes & logs

> **Please report every bug and crash, even small ones.** Early reports are the
> only way the rough edges get fixed.

The issue tracker is:

[Open the issue tracker](../../issues)

### Before you report

1. Reproduce the problem if you can.
2. Open the launcher **Status** tab and press **Copy**.
3. Collect:
   - `coop.log` from the game folder;
   - `install.log` from the launcher folder;
   - the crash ZIP, if one was created.
4. Open an issue with:
   - host or guest;
   - what you did, step by step;
   - what you expected and what happened;
   - your logs or the copied Status report.

### Crashes

When the game is started from the launcher and exits with an error, the launcher
creates a ZIP in:

`Documents\Prototype Coop\crash-reports`

It then offers to open a GitHub issue. The ZIP is **not uploaded
automatically**; review it and attach it to the issue. User paths and
17-digit Steam IDs are redacted, but logs may still contain a Steam name.

> **Again: please log errors and crashes.** A short report with logs is much
> more useful than “it doesn’t work”. If the game closes silently, check the
> crash-reports folder and attach the newest ZIP.

---

## 🧩 Known issues

- **World sync is off.** NPCs, traffic and ambient destruction are local to
  each PC and can differ.
- **Motion trails are off.** Recreating them crashes the puppet pose.
- **The WhipFist tendril** is working partially.
- **Some decals use the guest’s local material** when the exact shader family
  was never created locally.
- **Vehicle dents** only replay when the same vehicle exists on the guest.
- **Full campaign co-op** has not been validated fill but it """""should""""" work.
- **Cross-account join** needs more public testing.

> 🐞 These are exactly the things we need logs for. If you hit one, please report
> it instead of assuming it is already known.

---

## ❓ FAQ

**Does it touch my solo saves?**  
The mod uses a separate `Documents\Prototype Coop` save folder. Your original
solo folder is not overwritten.

**Can three or more players join?**  
No. The current target is two players.

**Does it work offline?**  
No. Steam must be running for the lobby, invites and networking.

**Do I need to install anything else?**  
Yes. Install **PrototypeFix 1.8.0 by emoose** from
[Nexus Mods](https://www.nexusmods.com/prototype/mods/52) and enable
`PreferLooseFiles = true`. The launcher installs the mod and Steam API and
detects PrototypeFix, but does not include or redistribute it.

**Will it work with a pirated copy?**  
No. Use a legitimate Steam copy.

**Where are the technical details?**  
The research and reverse-engineering notes are in [`docs/`](docs/README.md).

---

## 🙏 Credits

This project would not exist without the people and projects below.

- **PrototypeFix 1.8.0** - **emoose**  
  https://www.nexusmods.com/prototype/mods/52  
  Required third-party mod. Not bundled or redistributed; install it from
  Nexus and follow the author's permissions.
- **Co-op Anything** — co-op research directory and test-harness tooling.
- **Arefaris/prototype_decomp** and **pablopenna/prototype-decomp** — decompilation
  research references. No third-party source code is copied into this project.
- **Steamworks.NET** — rlabrecque  
  https://github.com/rlabrecque/Steamworks.NET/releases  
  The Steam API DLL is obtained at install time and is not bundled.
- **MinHook 1.3.4 / HDE32 / HDE64** — Tsuda Kageyu / Vyacheslav Patkov  
  BSD 2-Clause; see `THIRD_PARTY_LICENSES.txt`.
- **Valve / Steam** for Steamworks.
- **Activision / Radical Entertainment** for Prototype (2009).

Prototype and all related assets are property of Activision / Radical
Entertainment. This is an unofficial fan project and is not endorsed by them.

---

## 📜 License

Original code and documentation in this repository are released under the
**MIT License** — see [`LICENSE`](LICENSE).

The MIT license does **not** cover Prototype, its assets or trademarks,
PrototypeFix (emoose), Valve’s Steam API or MinHook/HDE. See
[`THIRD_PARTY.md`](THIRD_PARTY.md) for the full list and their own terms.

No game assets, Steam DLLs or executable patches are redistributed. The menu
movies are generated locally from the user’s own `art.rcf`.
