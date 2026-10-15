# IINACT_KR

A fork of marzent/IINACT modified for the Korean FFXIV client.
It is the parsing plugin used with [fexiv](https://github.com/hareeen/fexiv), a macOS launcher for FFXIV KR.
The `~/.fexiv/resources/dalamud/core/` path in the csproj files is the Dalamud install that fexiv manages, which is dal4kr's Korean Dalamud.

## Structure

- **IINACT_KR** (this repo): `origin` = aetherfl0w/IINACT_KR, `upstream` = marzent/IINACT
- **machina** (submodule): fork at aetherfl0w/machina, branch `dalamud`
    - remote `origin` = ravahn/machina (original)
    - remote `marzent` = marzent/machina (Dalamud port)
    - remote `mine` = aetherfl0w/machina (our fork)
- **../iinact_kr_dist**: serves the `FFXIV_ACT_Plugin` zip at iinact.hareen.io

Branches prefixed `u/` track upstream: `u/main` mirrors `upstream/main`, and `u/fix/*` branches carry changes meant for upstream pull requests.

## Updating for a new patch

Follow [docs/updating.md](docs/updating.md).
It covers the machina and upstream merges, the expected conflicts, the release workflow and the ACT plugin distribution.

## Korean-specific changes

Keep these when resolving conflicts:

- `IINACT/Plugin.cs`: `GameRegion.Korean` hardcoded with `RegionLocked`, and `ZoneDownHookManager` started after `GetFfxivPlugin()` returns
- `IINACT/Network/ZoneDownHookManager.cs`: version constants and unscrambler are loaded for the Global region only, everything else uses the dynamic fallback
- `FetchDependencies/FetchDependencies.cs`: `Region` enum with the Korean download URLs at iinact.hareen.io
- `machina/Machina.FFXIV/Headers/Opcodes/OpcodeManager.cs`: the `RegionLocked` property
- `machina/Machina.FFXIV/Headers/Opcodes/Korean.txt` and the `Korean` block of `OverlayPlugin.Core/resources/opcodes.jsonc`: Korean opcodes, taken from ravahn and marzent
- `.github/workflows/*.yml`, `repo.kr.json`, `.github/release-notices.kr.md`: our release pipeline and plugin registry

## Gotchas

- `gh` resolves this checkout to marzent/IINACT because of the `upstream` remote. Pass `--repo aetherfl0w/IINACT_KR`.
- Never edit `repo.json`. It is owned by upstream's CI, and `repo.kr.json` is owned by ours.
- The same `ffxivgame.ver` on the Korean and Global clients does not mean the same opcodes.
