# Prototype Coop changelog

Current: Launcher 0.2.3 | Mod 0.1.10

## Launcher 0.2.3 / Mod 0.1.10 - 2026-11-26

- **Known issue: starting a new campaign from the ONLINE lobby is not ready.**
  The opening cinematic can be skipped and both players can be left without
  control. Start the campaign solo, play through the morgue until you escape
  the doctors, save, then host ONLINE -> PLAY and load that save in co-op.
- **Mission state is now host-authoritative.** While a co-op session is active
  the host publishes the scenario's state machines (the mission state and its
  objective/checkpoint sub-machines) and the guest applies them, so both run
  the same mission instead of drifting apart after a load. `mission_sync=1`
  (the default in generated configs) controls it; set it to 0 to keep each
  game's own mission state.
- **Sequences and cinematics now start on both players.** The host also
  publishes which NIS sequence each sequence player is loading or playing, and
  the guest starts the same sequence, so a cinematic that begins on the host
  is not skipped on the guest. The full camera/sequence data is still the
  game's own; this synchronizes the begin/load, not a re-timed playback.
- **Checkpoint saves reach the guest.** When the host writes a checkpoint or
  autosave, the changed co-op save is sent to the guest and stored, so both
  machines keep the host's checkpoint. `checkpoint_sync=1` (the default)
  controls it. Respawning still uses the save loaded at the start; the
  transferred checkpoint is what the guest loads next.
- **NEW campaign mission-state sync (intro not ready):** the guest opens the
  same difficulty screen as the host and both receive the same mission state;
  the opening cinematic/control issue above still applies. Scenario-spawned
  mission actors and markers match on both players;
  ambient world objects/traffic, scripted checkpoints and the full
  sequence/camera data of cinematics are still not replicated.
- **Mission-only world state:** list up to eight mission actor/marker spawn
  template hashes in `mission_world`; the host sends their positions and the
  guest applies them, while ambient traffic and everything else stays local.
  Verified with `5A310666` (host moved it 2 m, guest followed with 0 m delta).
- **Supported mission-sync scope:** scenario state machines, the ONLINE
  PLAY save load, NIS begin/load, checkpoint save transfer, scenario-spawned
  mission actors/markers and the explicit `mission_world` actor list. The
  ONLINE NEW mission state syncs, but the opening issue above still applies.
  Ambient
  world streaming/traffic, live in-memory respawn and the full NIS
  camera/sequence timing are documented as unsupported and are not claimed.
  A general actor-sync experiment (240 entries) crashed the guest with
  `C0000005`; a transform-guarded retry also crashed, so the general mode was
  removed and only the explicit mission whitelist is supported.

## Launcher 0.2.2 / Mod 0.1.9 - 2026-11-24

- Fixed the lobby's **PLAY** row for controllers: the game leaves the movie's
  CONTINUE entry disabled on that screen, so a gamepad could not confirm PLAY.
  NEW keeps the top row, the waiting line sits under it, and PLAY moved to the
  third row, which a pad reaches and confirms.
- The **chat panel now stays open after sending**. Enter sends, Esc closes, and
  the footer shows the Escape hint in the accent colour instead of small grey
  text. Holding the opening Y no longer starts the message with a stray `y`.
- The **in-game lobby, version card and chat** now ship the four languages the
  Steam build offers: English, Spanish, French and Italian. Other language
  codes fall back to English. The launcher keeps its seven-language UI.
- The **start is no longer cancelled after 15 seconds** while the save list is
  open; it now waits up to two minutes for the load, so browsing saves cannot
  leave two connected games without a session and without the other avatar.
- The version card no longer shows a peer name before a peer is connected on
  the UDP test transport.
- **SEARCH** with Steam unavailable now reports that instead of starting a
  search that cannot return anything.

## Launcher 0.2.1 / Mod 0.1.8 - 2026-11-24

- New **host/guest version check**: when the second player joins, both mods
  exchange their mod version and the network protocol before a session starts.
- New **native lobby version card** drawn by the mod (no extra GFx movie):
  shows YOU, PEER, both versions, protocol and state (`IN SYNC`,
  `VERSION MISMATCH`, `INCOMPATIBLE`, `LINKING...`).
- The host's **PLAY** / **NEW** now waits for the guest's version. A different
  protocol blocks the start; a different mod version blocks it by default
  (`version_gate=block`) or warns with `version_gate=warn` in `coop.ini`.
- `coop_version.txt` in the game folder records the local and peer versions,
  the protocol and the last state. The launcher **Status** report and crash ZIP
  include it.
- The mod version is now compiled from `VERSION`, so the version shown in the
  lobby and the version in the package cannot drift apart.
- The number nameplate and the version card share the D3D9 overlay; the version
  card works with nameplates disabled (`version_overlay=1` by default).
- The card now saves/restores every D3D9 state it touches (blend op, colour
  write, stencil, fill mode, vertex blend, clip planes, texture transform), so
  engine/GFx state cannot tint or hide it.
- New **live ping** row on the card, measured once a second over the co-op
  transport (green < 80 ms, amber < 150 ms, red above). The card now also shows
  during a session and **Ctrl+F12** shows/hides it.
- New **connection chat**: **Y** opens the chat at the top-right, **Enter**
  sends, **Esc** closes, and PageUp/PageDown or the arrows scroll the history.
  A receiver sees a five-second message bubble with
  `PRESS Y ON YOUR KEYBOARD TO CHAT`. Messages are memory-only and are cleared
  when the peer link ends or a player is kicked; they survive level loads.
  `chat=1` in `coop.ini` enables it.
- The version card and chat are localized in English, Spanish, French, German,
  Italian, Portuguese and Dutch, in the same dark/red HUD style.
- Fixed the `Direct3DCreate9Ex` trampoline pointer in the overlay hook: it was
  never assigned, which could break device creation on that path.

## Launcher 0.2.0 / Mod 0.1.7 - 2026-11-23

- New **Verify files** button on the Install tab: checks the game version, mod
  ASI, input proxy, Steam API, menu hashes, PrototypeFix/PreferLooseFiles and
  `coop.ini`, then repairs only the missing or damaged pieces. It reports what
  was repaired and what still needs attention.
- Launcher visual refresh: dark tactical surfaces, neon-red accents,
  angular cards, red active tab and aligned Play-tab folder/save actions.
- The launcher remembers the selected game folder in `launcher.ini`.
- Play-tab shortcuts to open the game folder, co-op saves
  (`Documents\Prototype Coop`) and logs, plus a **Back up saves** button.
- Save backups copy the solo and co-op `profile.bin` / `slot-*.bin` files into
  the launcher data folder; the first co-op install backs them up automatically.
- **Portable mode** in the Cog dialog stores updates, rollback copies, crash
  reports and save backups in `data` next to the launcher. Game saves remain in
  `Documents\Prototype Coop`.
- New **Uninstall** button on the Install tab, with a confirmation prompt.
- The launcher version is now independent from the mod version.
- New **Changelog** tab that lists every update.
- Automatic update check on startup and a **Check for updates** button on the
  Changelog tab, with **View changes**, **Download package**, **Skip this
  version** and **Close**.
- **Stable / Beta** update channel: stable ignores GitHub prereleases; beta
  includes them but prefers a stable release with the same numeric version.
- **Skip this version** stores the skipped tag in `launcher.ini`; later startup
  checks stay quiet until a different release appears, while a manual check
  still shows it.
- Manual update flow: the launcher saves a rollback copy of the current package,
  downloads the official GitHub ZIP, verifies its SHA-256 and extracts it into a
  visible `Documents\Prototype Coop\updates` folder. The user copies the files
  over the launcher folder. No hidden PowerShell, self-copy or automatic
  restart, so antivirus heuristics are not triggered by a self-updater.
- **Rollback** on the Changelog tab lists packages saved in
  `Documents\Prototype Coop\rollbacks`; restoring one is manual and keeps saves,
  `coop.ini`, logs and PrototypeFix.
- The Changelog/About tabs state that automatic update checks need v0.1.7 or
  newer and older builds must update manually once.
- New **Feedback** tab: one text box, Send and Copy. The message and the
  launcher/mod versions go to a private Discord channel through a Cloudflare
  Worker. No SteamID, username, paths or hardware data.
- Launcher localization: English, Spanish, French, Italian, German, Portuguese
  and Dutch. The update popup, Changelog, Feedback and Rollback tabs use the
  selected language; the Cog dialog has a language selector.
- Feedback privacy note and docs disclose that Cloudflare receives the IP and
  keeps only a short-lived hash for rate limiting, and that Discord retains the
  message until it is deleted.
- Uninstall removes the co-op plugin, Steam API copy and XInput proxy, and
  restores the original menu movies from the install backups.
- Uninstall is upgrade-safe: older ProtoCoop proxies, movies and plugins are
  removed instead of being restored as if they were original files.
- A foreign `xinput1_3.dll` is preserved, not deleted.
- Saves, `coop.ini`, logs, PrototypeFix, `binkw32.dll` and
  `prototypeenginef.dll` are left untouched.
- Added `--uninstall <game folder>` for automation and support.

## Mod 0.1.6 - 2026-11-22

- Published clean release.
- Guest join feedback: `JOINING`, `JOINED` and `JOIN FAILED`.
- Host invite feedback: `INVITED <name>`, `ALREADY INVITED`,
  `ALREADY IN LOBBY` and `LOBBY FULL`.
- Invitations now reach a guest that never opened its own INVITE list.
- Fixed invite prompts repeating every few seconds; a declined invitation now
  waits for a new invitation.
- Fixed a join race that could create a new lobby while a join was pending.
- Fixed a kicked guest immediately creating its own lobby again.
- Added a full-lobby guard and withdrew outstanding invites when the last slot
  filled.
- Fixed stale invite data surviving into a new lobby.
- Improved lobby focus handling and removed duplicate focus entries.
- Lobby text follows the game language: English, Spanish, French and Italian,
  plus German, Portuguese and Dutch.

## Mod 0.1.5 - 2026-11 (rolled into 0.1.6)

- Added guest join states and failure feedback.
- The host now sees the guest's name and can remove the guest with `[KICK]`.
- Added the cooperative kick packet and the kicked guest screen.
- Localized the new lobby rows.
- Fixed joined, kicked and focus-related movie states.

## Mod 0.1.4 - 2026-11-21

- First clean public release.
- PrototypeFix is no longer bundled or redistributed; the launcher detects it
  and links to Nexus instead.
- The launcher installs the mod and Steam API, rebuilds the ONLINE and lobby
  movies from the player's own `art.rcf`, and checks the result.
- Launcher tabs: Play, Install, Settings, Status and About.
- Game settings for resolution, LOD, FOV, shadows, FPS, VRAM and fixes.
- Status report, crash ZIP and GitHub issue helper.
- Public lobby count.
- Two-player Steam co-op with remote avatar, movement, pose, powers, effects,
  health and world sync.
- .NET Framework 4.5 or newer required by the launcher.

## Mod 0.1.0 - 0.1.3 - 2026-11 (early public test)

- Initial public test releases.
- Steam lobby, host/guest flow and the first remote avatar.
- Early movement, pose, power, effect and world sync.
- Early launcher and documentation.
- Known crashes, desync and missing effects; these releases were superseded.
- Old release tags were removed; v0.1.4 is the first clean release.

## Launcher versions before 0.2.0

- The launcher followed the mod version from 0.1.0 through 0.1.7.
- Launcher 0.2.0 is the first independently versioned launcher.
