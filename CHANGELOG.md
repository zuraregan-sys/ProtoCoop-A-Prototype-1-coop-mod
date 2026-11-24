# Prototype Coop changelog

Current: Launcher 0.2.0 | Mod 0.1.7

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
