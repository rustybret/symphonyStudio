# Symphony + oh-my-opencode Getting Started

This guide sets up a combined stack for:

- Symphony experimental orchestration (Elixir reference implementation)
- OpenCode runtime configuration (`opencode.jsonc`)
- oh-my-opencode orchestration profile (`.opencode/oh-my-opencode.jsonc`)
- Local MCP server source checkout under `integrations/mcp-servers/`

## 1) Prerequisites

Install these first:

- `mise` (recommended by Symphony Elixir)
- `elixir` and `erlang` via `mise`
- `opencode` CLI
- `bun` (for `bunx oh-my-opencode install`, optional but recommended)
- `node` + `npm` (for several MCP servers)
- `python` + `uv/uvx` (for Python/UV-based MCP servers)
- `git`

Environment variables you will need:

- `LINEAR_API_KEY` for Symphony tracker access
- `SYMPHONY_WORKSPACE_ROOT` (optional override for workspace root)
- `CODEX_BIN` (optional, if `codex` is not on `PATH`)

Useful MCP variables (set only when enabling those servers):

- `FIGMA_API_KEY`
- `MAGIC_21ST_API_KEY`
- `OBS_WEBSOCKET_URL` (default usually `ws://localhost:4455`)
- `OBS_WEBSOCKET_PASSWORD`

## 2) Symphony Experimental Setup

From repo root:

```bash
cd elixir
mise trust
mise install
mise exec -- mix setup
mise exec -- mix build
mise exec -- ./bin/symphony ./WORKFLOW.md
```

Notes:

- `elixir/WORKFLOW.md` is the runtime contract (YAML front matter + Markdown prompt body).
- Symphony defaults include safer Codex policies (`mcp_elicitations`, workspace sandbox defaults) when policy fields are omitted.
- Run with a custom workflow file path if needed:

```bash
./bin/symphony /absolute/path/to/WORKFLOW.md
```

## 3) OpenCode + oh-my-opencode Config Layout

This repository now includes:

- `opencode.jsonc` — project OpenCode config and MCP server registry
- `.opencode/oh-my-opencode.jsonc` — project oh-my-opencode agent/category profile

How OpenCode loads config (high-level):

1. remote defaults
2. global config (`~/.config/opencode/opencode.json`)
3. custom path overrides
4. project config (`opencode.json` / `opencode.jsonc`)
5. `.opencode` directory assets

## 4) Cloned MCP Sources

Repositories are cloned into:

`integrations/mcp-servers/`

Included:

- `everything-claude-code`
- `Figma-Context-MCP`
- `openfang`
- `unity-mcp`
- `magic-mcp`
- `npcpy`
- `ivanmurzak-unity-mcp`
- `UnrealGenAISupport`
- `studio-rust-mcp-server`
- `blender-mcp-vxai`
- `game-asset-mcp`
- `advanced-unity-mcp`
- `flstudio-mcp`
- `obs-mcp`

## 5) MCP Registration Strategy in This Repo

All requested MCP entries are pre-registered in `opencode.jsonc` under `mcp` and default to:

- `"enabled": false`

Why disabled by default:

- several servers require heavyweight local app/runtime coupling (Unity Editor, Unreal Editor, FL Studio, OBS, Roblox Studio)
- several require per-user secrets or GUI-side setup
- this prevents OpenCode startup failures while keeping all server definitions versioned and ready

Enable one server at a time by switching `enabled` to `true` for that server in `opencode.jsonc`.

Then verify with:

```bash
opencode mcp list
opencode mcp debug <server-name>
```

For OAuth-based remote MCPs (if you add them), use:

```bash
opencode mcp auth <server-name>
```

## 6) Practical Bring-Up Order

Use this order to avoid debugging too many moving parts at once:

1. Bring up Symphony Elixir and verify tracker/workflow behavior.
2. Start OpenCode with `opencode` and confirm project config is loaded.
3. Enable low-friction MCPs first:
   - `figma_context_mcp` (if key available)
   - `magic_mcp` (if key available)
   - `obs_mcp` (if OBS websocket is configured)
4. Enable editor-coupled MCPs only when their host app is already running and configured:
   - Unity (`unity_mcp_coplay`, `unity_mcp_ivanmurzak`, `advanced_unity_mcp`)
   - Unreal (`unreal_genai_support`)
   - FL Studio (`flstudio_mcp`)
   - Roblox (`roblox_studio_rust_mcp_server`)
5. For each server, run one small command through the client before enabling the next.

## 7) Server-Specific Notes

- `everything_claude_code`: this is a harness toolkit/plugin repo, not only a single MCP binary; keep disabled unless you build/point to a valid runnable entrypoint.
- `openfang`: Agent OS, not a drop-in MCP daemon by default; currently mapped as local command placeholder and disabled.
- `npcpy`: framework + CLI toolkit; not a packaged single MCP daemon by default.
- `unity_mcp_*`/`advanced_unity_mcp`: require Unity-side plugin/dashboard setup first.
- `unreal_genai_support`: requires Unreal plugin installation and Python bridge path tied to a real Unreal project.
- `roblox_studio_rust_mcp_server`: requires binary build/download and Roblox Studio plugin side.

## 8) Day-0 Validation Checklist

- `elixir/WORKFLOW.md` parses cleanly and Symphony starts.
- `LINEAR_API_KEY` is exported and tracker requests succeed.
- `opencode mcp list` shows configured MCP entries.
- At least one enabled MCP completes a successful debug call.
- oh-my-opencode profile is loaded from `.opencode/oh-my-opencode.jsonc`.

## 9) Recommended Next Hardening Steps

- Move sensitive env vars to your shell profile or a dedicated secrets manager.
- Keep only active MCPs enabled per project to control context footprint and tool noise.
- Add per-project `AGENTS.md` guidance for when each MCP should be used.
- Pin server versions for reproducibility after initial validation.
