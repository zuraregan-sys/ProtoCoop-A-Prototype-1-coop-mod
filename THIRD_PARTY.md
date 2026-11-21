# Third-party components

The public package is built to avoid redistributing game assets, Valve binaries
and unrelated mods. This is the current inventory.

## Project license

Original code and documentation in this repository are MIT-licensed (see
`LICENSE`). That license does **not** cover the components below, Prototype
(2009), its code/assets or its trademarks. PrototypeFix is **not** bundled or
redistributed; users install it separately from Nexus. MinHook/HDE are linked
into the plugin with their BSD notice; the Steam API is obtained at install
time and is not stored in the repository or ZIP.

## Required, not distributed

**PrototypeFix 1.8.0** — emoose
https://www.nexusmods.com/prototype/mods/52

The mod needs PrototypeFix's `PreferLooseFiles = true` to load the generated
menu movies. PrototypeFix is **not** bundled, copied or modified by this
project; users install it separately from Nexus and must follow the author's
permissions. The launcher only detects it and checks `PreferLooseFiles`.
Credit is in `README.md`, `CREDITS.txt`, `LEEME.txt` and the launcher.

## Compiled into the plugin

**MinHook 1.3.4 + HDE32/HDE64**
Tsuda Kageyu / Vyacheslav Patkov — BSD 2-Clause.

The plugin links MinHook for its engine hooks. The BSD notice is shipped in
`THIRD_PARTY_LICENSES.txt` in every package. Replacing it with an in-house hook
layer is a separate task if the project must contain no third-party source at
all.

## Fetched at install time (not in the package)

**Steam API (`steam_api.dll`)** — Valve / Steamworks.NET standalone 2025.164.1
https://github.com/rlabrecque/Steamworks.NET/releases

The launcher first looks for a compatible DLL in the user's Steam library. If
none is found it downloads the pinned Steamworks.NET standalone ZIP and verifies
both the ZIP and the extracted DLL by SHA-256 before copying it as
`coop_steam_api.dll`. No Valve binary is stored in the repository or the ZIP.

## Not distributed

- PrototypeFix (`prototype_fix.asi`, `prototype_fix.ini`) and its binkw32
  loader files: required, but installed separately by the user from Nexus.
- Game movies (`protomainmenu.gfx`, `protodevmenu.gfx`): generated locally from
  the user's own `art.rcf` using the patch data in `payload\patches`.
- Game executable patches and third-party delta/launcher tools: not
  distributed. The game is launched through Steam.
- Steamworks DLLs, decompilation sources and other mods.

## Research references

`docs/DECOMP-AUDIT.md` and `docs/RESEARCH.md` cite the decompilation projects
and other mods that were read for research. The audit records that no
third-party source was copied into this project; all plugin code is original.
This inventory and the research documents are not legal advice; confirm
redistribution permissions and the relevant licenses before publishing.
