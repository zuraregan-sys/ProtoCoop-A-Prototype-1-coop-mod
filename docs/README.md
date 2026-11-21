# Prototype Co-op — research documentation

This directory is the public English edition of the project's research notes.
The original Spanish working notes were used during development; these editions
preserve the technical findings, addresses and source references.

| Document | Contents |
|---|---|
| [RESEARCH.md](RESEARCH.md) | Build identification, prior co-op work, decompilation review, feasibility and recommended architecture. |
| [DECOMP-AUDIT.md](DECOMP-AUDIT.md) | Read-only audit of `Arefaris/prototype_decomp`: online subsystem state, reusable pieces, blockers and entry points. |
| [WORLD-SYNC.md](WORLD-SYNC.md) | World-sync experiments: original netcode, snapshots, health, object→template mapping, freezing, determinism and render-only proxies. |
| [ENGINE-NOTES.md](ENGINE-NOTES.md) | Engine findings, verified addresses, puppet/pose/effects/damage/grab implementation notes. |
| [PLAN.md](PLAN.md) | Early transport-seam design (historical). |
| [ENGINE-COOP-ROADMAP.md](ENGINE-COOP-ROADMAP.md) | Exploratory path for reviving the engine's native online layer. |
| [TODO.md](TODO.md) | Current status and remaining work. |

## Scope and legal notes

- These documents describe reverse engineering and modding research for
  **Prototype (2009)**. They do not contain game code, game assets or leaked
  material, and they do not grant rights over the game.
- The decompilation projects referenced here have their own licenses. No code
  from them was copied into this project; see [DECOMP-AUDIT.md](DECOMP-AUDIT.md)
  and `../THIRD_PARTY.md`.
- The public package does not bypass Steam DRM or Family Sharing. The game must
  start normally through Steam; the launcher only checks TCP port 27015.
- Addresses are for the `PROTOPC_2025` build (`prototypeenginef.dll` SHA-256
  `d331be20…537071`). A game update can invalidate them.
- Prototype and related assets are property of Activision / Radical
  Entertainment. This is an unofficial fan project.
