# vscodium-base/

VSCodium upstream tree. Source of the build pipeline that produces `HypeProof Studio.app`.
See top-level [/CLAUDE.md](../CLAUDE.md) for the hard rules. This file adds build-specific detail.

## How changes flow

```
upstream VS Code source  ──┐
                           ├── prepare_vscode.sh  ──►  patched tree  ──►  build.sh  ──►  .app
patches/*.patch        ────┘                                ▲
                                                            │
product.json + jq overrides (in prepare_vscode.sh)  ────────┘
```

- **Never edit `vscode/` or `VSCode-*/` directly** — both are derived. Edits there will be lost on the next `prepare_src.sh`.
- To change app behavior: add a `patches/NN-<topic>.patch`. Naming follows existing prefix (`00-brand-...`, `00-build-...`).
- To change branding strings: extend the jq block inside `prepare_vscode.sh` that rewrites `product.json`.

## Scripts (run order)

| Script | Purpose | Time |
|---|---|---|
| `get_repo.sh` | Resolve VS Code commit hash | seconds |
| `prepare_src.sh` | Download VS Code source (~500 MB) | 1–3 min |
| `prepare_vscode.sh` | Apply patches + product.json overrides | < 1 min |
| `build.sh` | Full build → `.app` | **1–2 h, 10–20 GB** |
| `icons/build_icons.sh` | SVG → .icns/.ico/.png for all platforms | 1–2 min |

## Required env (override `utils.sh` defaults)

`utils.sh` defaults are VSCodium-flavored. Source `../hypeproof-studio.env` first so these win:

```
APP_NAME, APP_NAME_LC, BINARY_NAME, ASSETS_REPOSITORY, GH_REPO_PATH, ORG_NAME
```

Plus build-time:

```bash
export NODE_OPTIONS="--max-old-space-size=12288"
export VSCODE_PUBLISH_COUNTER=0     # skip sourcemap upload attempts
export GITHUB_TOKEN=...              # only if hitting GitHub rate limits
```

## Known failure modes

| Symptom | Fix |
|---|---|
| `prepare_src.sh` 401/404 | Set `GITHUB_TOKEN` (any PAT with public repo read) |
| `npm install` fails on node-gyp | `xcode-select --install`; verify `python3` resolves to 3.11 |
| OOM during build | Raise `NODE_OPTIONS` max-old-space-size; close other apps |
| `.app` shows VSCodium name | `hypeproof-studio.env` not sourced, or `prepare_vscode.sh` not re-run after env change |

## product.json overrides (Phase 2)

Keys to override via jq in `prepare_vscode.sh`:
`nameShort`, `nameLong`, `applicationName`, `dataFolderName`,
`darwinBundleIdentifier` (→ `ai.hypeproof.studio`), `urlProtocol` (→ `hypeproof-studio`),
`win32DirName`, `win32NameVersion`, `win32MutexName`.

## Don't touch

- `patches/*.patch` from upstream VSCodium without reading the patch first — most are load-bearing (branding removal, build fixes, electron pin).
- `npmrc` — controls private registry behavior for build.
- Files under `VSCode-darwin-arm64/` — build output, regenerated every build.
