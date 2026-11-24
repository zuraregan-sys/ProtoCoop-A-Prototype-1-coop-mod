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

### Verify files

Press **Verify files** on the Install tab after an antivirus warning, a failed
install or a partial update. It checks:

- the supported game version (the base menu movie inside `art.rcf`);
- the mod ASI, XInput proxy and both menu hashes;
- the Steam API DLL and its required exports;
- PrototypeFix presence and `PreferLooseFiles = true`;
- `coop.ini` presence.

It repairs only the missing or damaged mod/proxy/menu/Steam API files and
enables `PreferLooseFiles` when needed. PrototypeFix itself is third-party and
is never installed or replaced; if it is missing, the report points to Nexus.
Saves, `coop.ini`, logs and an existing PrototypeFix install are kept.

### Uninstall

Close the game, open **Install**, and press **Uninstall**. The launcher:

- removes `prototype_coop.asi`, `coop_steam_api.dll` and the XInput proxy;
- restores the original menu movies from the backups made at install time, or
  removes the loose movies so the game uses `art.rcf` again;
- restores a backed-up `steam_appid.txt` if the installer removed one.

Saves, `coop.ini`, logs and PrototypeFix are **not** touched. Nothing else in
the game folder is changed, and the launcher can reinstall the mod afterwards.

### Versions and changelog

The **launcher** and the **mod** are versioned separately. The launcher footer
shows both versions, and the **Changelog** tab lists every update from the
early public test through the current build.

### Updates

The **Changelog** tab has **Check for updates**, and the launcher also checks
once on startup. Next to the button you can choose **Stable** (default) or
**Beta**. Stable ignores GitHub prereleases; Beta includes them, but a stable
release still wins when the numeric version is the same.

If the launcher, the mod, or both have a newer version, a popup offers:

- **View changes** - read the release changelog.
- **Download package** - save a rollback copy of the current package, download
  the official ZIP, verify its SHA-256 and extract it into the launcher data
  folder (`Documents\Prototype Coop\updates\PrototypeCoop-<tag>` normally,
  `<launcher>\data\updates\...` in portable mode), with an instructions file,
  then open that folder.
- **Skip this version** - hide this release on future startups. **Check for
  updates** still shows it, and a newer release is offered normally.
- **Close** - keep playing; the launcher reminds you on the next start.

To apply it, close the launcher, copy the files from that folder over your
launcher folder and open the new `PrototypeCoopLauncher.exe`. If the mod also
changed, press **Install** in the new launcher. Nothing restarts itself, runs
hidden, or needs GitHub. Your saves, `coop.ini` and PrototypeFix are untouched.

The launcher saves the current package in the launcher data folder
(`Documents\Prototype Coop\rollbacks` normally, `<launcher>\data\rollbacks` in
portable mode) on startup and before downloading an update. **Rollback** on the
Changelog tab lists those copies. Rollback is manual: close the launcher, copy
the chosen folder over the launcher folder, open the old
`PrototypeCoopLauncher.exe` and press **Install** to restore the older mod.
Saves, `coop.ini`, logs and PrototypeFix are not touched.

Automatic update checks require **v0.1.7 or newer**. Older launchers must
update manually once; after that, the Changelog tab handles future updates.

### Folders, saves and portable mode

The launcher remembers the selected game folder in `launcher.ini`, so you do
not need to browse for it on every launch. The **Play** tab has buttons to open
the game folder, the co-op saves in `Documents\Prototype Coop`, and the game
logs (`coop.log` when present, otherwise the launcher log).

**Back up saves** copies the solo and co-op save files (`profile.bin` and
`slot-*.bin`) into the launcher data folder under `save-backups`. The launcher
also performs this backup automatically before the first co-op install, so the
existing campaign is safe while the mod seeds its co-op save folder. Saves are
only copied, never changed; the newest five backups are kept.

**Portable mode** (Cog dialog) stores updates, rollback copies, crash reports
and save backups in `data` next to the launcher instead of
`Documents\Prototype Coop`. Game saves stay in `Documents\Prototype Coop`
because the game controls that path. `launcher.ini` already lives next to the
launcher, so the whole folder can be moved together.

### Feedback

The **Feedback** tab has a single text box and a **Send** button. It sends the
message plus the launcher and mod versions to a private Discord channel through
a Cloudflare Worker. No SteamID, username, file path or hardware data is sent.
If sending is unavailable, **Copy** puts the text on the clipboard.
Feedback is not end-to-end encrypted: Cloudflare and Discord process the
message under their own privacy policies. Do not send passwords, addresses or
other sensitive information.

### Privacy and network access

The launcher has no analytics, telemetry or advertising and never uploads logs
by itself. It contacts:

- **GitHub** on startup to check releases and during install to download
  Steamworks.NET. GitHub sees your IP and the requested file.
- **Cloudflare and Discord** only when you press **Send** in Feedback.
  Cloudflare receives your IP; the Worker stores only a short-lived hash for
  rate limiting. Discord keeps the message in a private channel until it is
  deleted. Do not send personal information.
- **Steam** for the game, lobby and public player count, under Steam's policy.

Crash reports are written locally under the launcher data folder
(`Documents\Prototype Coop\crash-reports` normally, `data\crash-reports` in
portable mode) and are uploaded only if you attach one to a GitHub issue
yourself.

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

> **Joining feedback:** after choosing a host, the lobby shows `JOINING`. On
> success it shows `JOINED` with the host name and returns to the host list; on
> failure it shows `JOIN FAILED` and you can retry.

> **Invite feedback:** after inviting a friend, the host's waiting row shows
> `INVITED <name>`, `ALREADY INVITED`, `ALREADY IN LOBBY` or `LOBBY FULL`. The
> friend list marks the row `[INVITED]`. A declined invitation is remembered
> until the host invites that friend again, so the guest is not prompted every
> few seconds.

> **Co-op only starts from the Online lobby.** Continue and New Game on the
> main menu are single-player; they do not arm co-op, send player state or spawn
> a remote avatar. This works even if a lobby is still open.

### Kicking a guest

The host sees the guest's name in the lobby. That row is marked `[KICK]`;
select it and press A to remove the guest. The guest is asked to leave and sees
`KICKED`.

This is cooperative: Steam's matchmaking API in this build has no forced lobby
kick, so a modified client could ignore it.

All lobby text follows the game's language: English, Spanish, French and
Italian, plus German, Portuguese and Dutch when the game uses them.

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

- **Play** — launch, component status, port check, public-lobby player count, open game/save/log folders and back up saves.
- **Install** — choose the game folder (remembered), install/update, uninstall or verify files.
- **Game settings** — video, LOD, FOV, shadow resolution, FPS, VRAM and fixes.
- **Status** — copyable diagnostics: hashes, port owner and current settings.
- **Changelog** — version history, update checks, stable/beta channel and rollback.
- **Feedback** — send a message to the developer from the launcher.
- **About** — credits, docs and the report button.
- **Cog** — language (English, Spanish, French, Italian, German, Portuguese and Dutch), Dark/Light UI and portable mode. The update popup and Changelog tab follow the same language.

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
  lobbies, invites and the public-lobby count. GitHub is used to check for
  launcher/mod updates, download an update ZIP or the pinned Steamworks.NET API
  when needed, and open links you click. Downloads are verified by SHA-256.
  The update check sends no account or game data.
- **No executable patches.** The launcher adds the mod and input proxy and
  replaces the two menu movies with versions generated from your own `art.rcf`
  (originals are backed up). PrototypeFix is installed separately and is not
  copied or modified. It does not patch or replace `prototypef.exe`, and it
  ships no game assets or Steam DLLs.
- **Local files only.** Launcher settings stay next to the launcher; game
  settings and logs stay in the game folder. Updates, rollback copies, crash
  reports and save backups live under `Documents\Prototype Coop`, or under
  `data` next to the launcher in portable mode. Co-op saves stay in
  `Documents\Prototype Coop`. Delete them anytime.
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

`Documents\Prototype Coop\crash-reports` normally, or
`data\crash-reports` in portable mode.

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
solo folder is not overwritten. The launcher also offers **Back up saves** and
backs up the solo/co-op `profile.bin` and `slot-*.bin` files before the first
co-op install.

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
