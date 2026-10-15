# Updating IINACT_KR for a new game patch

This is the runbook for bringing the Korean build up to date after a patch.
A normal patch is data only: opcode tables, package bumps and version strings.
No source change is needed unless upstream changed code that touches the Korean wiring listed in [CLAUDE.md](../CLAUDE.md).

Three repositories move together:

| Repository | What it ships | Remote to push |
| --- | --- | --- |
| `IINACT_KR` (this repo) | The Dalamud plugin and the `repo.kr.json` registry | `origin` = aetherfl0w/IINACT_KR |
| `machina` (submodule, branch `dalamud`) | Network opcode tables, including `Korean.txt` | `mine` = aetherfl0w/machina |
| `../iinact_kr_dist` | The `FFXIV_ACT_Plugin` zip served at iinact.hareen.io | `origin` = aetherfl0w/iinact_kr_dist |

## 1. Check what landed

```bash
git fetch upstream
git -C machina fetch origin      # ravahn/machina
git -C machina fetch marzent     # marzent/machina

git log --oneline HEAD..upstream/main
git -C machina log --oneline HEAD..origin/master
git -C machina log --oneline HEAD..marzent/dalamud

cat ~/.fexiv/programs/ffxiv/game/ffxivgame.ver
curl -s https://raw.githubusercontent.com/dal4kr/Dalamud.Resources/refs/heads/main/Dalamud/VersionInfo.json
gh release view --repo ravahn/FFXIV_ACT_Plugin --json tagName
curl -s https://iinact.hareen.io/version
```

Go ahead only when all of these hold for the Korean game version:

- `upstream/main` has a `Korean` block for that version in `OverlayPlugin.Core/resources/opcodes.jsonc`.
- ravahn's `Korean.txt` was updated for it.
- dal4kr's `SupportedGameVer` matches it, because CI builds against that Dalamud.

The same version string does not mean the same opcodes.
Opcodes are shuffled per build, so the Korean and Global tables differ even when both clients report the same `ffxivgame.ver`.

Cross-check the two Korean tables before merging.
`opcodes.jsonc` is decimal and `Korean.txt` is hex, and `ActorMove` and `ActorSetPos` exist in both:

```bash
git -C machina show origin/master:Machina.FFXIV/Headers/Opcodes/Korean.txt | grep -i '^ActorMove\|^ActorSetPos'
git show upstream/main:OverlayPlugin.Core/resources/opcodes.jsonc | sed -n '/"Korean"/,/^    }/p' | grep -A1 '"ActorMove"\|"ActorSetPos"'
```

## 2. Merge machina

```bash
cd machina
git merge origin/master
git merge marzent/dalamud
```

Expected conflicts and how to resolve them:

- `Global.txt`: ravahn and marzent differ only by a trailing newline. Take marzent's.
- Any other region table, such as `Chinese.txt`: take whichever side is newer, which is normally ravahn's. Diff the two sides before choosing.
- `Machina.FFXIV.csproj`: keep marzent's Dalamud-adapted file, which targets `net10.0` and drops the Deucalion content block, with ravahn's version number. Taking ravahn's file breaks the build with `NU1201`.

After merging, confirm:

```bash
diff <(git show origin/master:Machina.FFXIV/Headers/Opcodes/Korean.txt) Machina.FFXIV/Headers/Opcodes/Korean.txt
grep -c RegionLocked Machina.FFXIV/Headers/Opcodes/OpcodeManager.cs   # must be 2
grep -n '<TargetFramework>' Machina.FFXIV/Machina.FFXIV.csproj         # must be net10.0
```

## 3. Merge upstream

```bash
cd ..
git merge upstream/main
```

Expected conflicts:

- `machina`: always conflicts. Resolve with `git add machina` after step 2.
- `IINACT/IINACT.json`: `ApplicableVersion` must be the Korean game version. Take upstream's value when it matches, otherwise set it by hand.
- `OverlayPlugin.Core/resources/opcodes.jsonc`: take upstream's file whole. It carries the Korean block.
- `FetchDependencies/FetchDependencies.cs`, `IINACT/Plugin.cs`: keep the Korean wiring and adopt upstream's other changes.

Never edit `repo.json`. It belongs to upstream's CI, and leaving it alone keeps syncs conflict-free.
`repo.kr.json` belongs to our CI.

Build before committing:

```bash
dotnet build IINACT/IINACT.csproj -c Release
```

## 4. Push

Push the submodule first, so the pointer in `main` always resolves:

```bash
git -C machina push mine dalamud
git push origin main
```

Do not rewrite history on a range that spans a merge from upstream.
`git filter-branch` over such a range also rewrites upstream's commits and leaves `main` with a duplicate copy of marzent's history.

## 5. Release

Pushing to `main` runs `.github/workflows/build.yml`.
It downloads dal4kr's Dalamud, verifies `hashes.json` against the manifest, and builds.
When the version in `IINACT/IINACT.csproj` differs from `repo.kr.json`, it publishes release `v<version>`, rewrites `repo.kr.json` and commits it to `main`.
Pull that commit afterwards with `git pull --ff-only origin main`.

`gh` resolves this checkout to marzent/IINACT because of the `upstream` remote, so always name the repository:

```bash
gh run list --repo aetherfl0w/IINACT_KR --limit 3
gh release view v<version> --repo aetherfl0w/IINACT_KR
gh workflow run build.yml --repo aetherfl0w/IINACT_KR   # manual run
```

For a Korean-only release between upstream versions, bump the fourth component of `<Version>` in `IINACT/IINACT.csproj`.
Dalamud only offers an update when the version goes up.

## 6. Update the ACT plugin distribution

Only needed when ravahn published a newer `FFXIV_ACT_Plugin`.

```bash
cd ../iinact_kr_dist
gh release download <ver> --repo ravahn/FFXIV_ACT_Plugin --pattern 'FFXIV_ACT_Plugin_<ver>.zip'
unzip -tq FFXIV_ACT_Plugin_<ver>.zip
shasum -a 256 FFXIV_ACT_Plugin_<ver>.zip   # compare with the asset digest on the release
git rm FFXIV_ACT_Plugin_<old>.zip
git add FFXIV_ACT_Plugin_<ver>.zip
git commit -m "feat: plugin <ver>"
git push origin main
```

Keep exactly one zip in the repo, because the version served at `/version` comes from its filename.
The push builds `ghcr.io/aetherfl0w/iinact-kr-dist`, tagged with the version and `latest`.
The host then has to pull the new image.
Verify with `curl https://iinact.hareen.io/version` and by hashing `https://iinact.hareen.io/download`.

IINACT re-downloads the plugin only when the served version is newer than the installed one, so a late deploy breaks nothing.

## 7. Verify in game

Reload the plugin, hit a striking dummy, then check:

- `~/.fexiv/resources/dalamud/log/dalamud.log` shows `GameRegion: "Korean"` and no `No resource opcodes for game region` warning.
- The newest `~/Documents/IINACT/Network_*.log` contains cast lines (`20|`) and ability lines (`21|`, `22|`).

## Troubleshooting

- **Overlay is empty but the logs look healthy.** The opcode tables do not match the running build. The network log then holds only memory-based lines such as `261|` and `00|`, with no `21|` or `22|`. Check the Korean tables for the current `ffxivgame.ver`.
- **Plugin fails to load with `Invalid handle`, `Cannot access a closed file` or `Stream does not support seeking`.** This is the hook-install race under Wine during the plugin download, fixed in `Plugin.cs` by starting `ZoneDownHookManager` after `GetFfxivPlugin()` returns. Make sure a merge did not move it back. Upstream PR: marzent/IINACT#177.
- **No CI run after a push.** Check that Actions is enabled on the repository and in the organization.
