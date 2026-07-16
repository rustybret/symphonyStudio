# Draft: SymphonyStudio Agentic Game Dev Initialization

## Goal

Extend the upstream Symphony + OpenCode + oh-my-opencode stack into a game-development oriented distribution that:

- Bootstraps core orchestration (Symphony Elixir) and agent runtime (OpenCode + oh-my-opencode) first.
- Uses an interactive, idempotent initializer to collect required credentials and enable/compose engine + tool “packs”.
- Produces a working baseline with no editor-coupled dependencies, then offers optional supporting tools.

## Current Repo Facts (from exploration)

- Core configs:
  - `elixir/WORKFLOW.md` (Symphony runtime contract + agent prompt)
  - `opencode.jsonc` (OpenCode config + MCP registry; toggled via `enabled`)
  - `.opencode/oh-my-opencode.jsonc` (agent models/categories/skills)
- MCP sources vendored under `integrations/mcp-servers/`.
- No existing interactive initializer/wizard; setup is doc + file editing driven.
- Symphony CLI enforces a "guardrails acknowledgement" flag before running (pattern to reuse for risky OAuth scope consent).
- Current defaults likely need changing:
  - `opencode.jsonc` has `figma_context_mcp` and `obs_mcp` enabled by default.
  - `elixir/WORKFLOW.md` `hooks.after_create` clones `https://github.com/openai/symphony` (not SymphonyStudio / user repo).

## Proposed Init UX (two-stage)

1) Stage A: “Core bring-up” (can run while user gathers credentials)
- Verify prerequisites (mise, elixir/erlang, opencode, node, python, uv).
- Install/verify oh-my-opencode plugin baseline.
- Ensure Symphony can start with a minimal workflow.

2) Stage B: “Profile selection + secrets”
- Ask for:
  - Linear API key + project slug
  - Google OAuth/API credentials (scope-dependent)
  - Minimal oh-my-opencode agent set for ultrawork
- Ask to choose:
  - Primary engine target (Unity/Godot/Unreal/Roblox/Android/iOS/Python)
  - Optional tool packs (MCP servers + supporting apps/plugins)

Important: keep prompts out of unattended runs. Orchestrated agent sessions (per `elixir/WORKFLOW.md`) should never require a human action; all interactive credential collection belongs in a separate init command run by the developer before starting Symphony.

Default policy decision: pin exact versions/commits for reproducibility; provide an explicit upgrade path rather than tracking "latest" by default.

Team workflow decision: initializer should create a branch + commit changes so teams can open a PR for review/merge.

## Pack Model (recommended)

- A “pack” is a declarative overlay that can be applied/removed:
  - toggles/extends `opencode.jsonc` MCP entries (enable flags + env var requirements)
  - updates `.opencode/oh-my-opencode.jsonc` (skills/categories/agents)
  - updates `elixir/WORKFLOW.md` front matter (hooks, workspace root, Codex policies)
  - optionally adds documentation snippets + post-install checks

Pack metadata should explicitly declare:

- `requires`: host apps (Unity/Unreal/etc), runtimes (node/python/uv), and secrets/env vars
- `mcp`: which MCP entries to enable + expected local URL/port for remote servers
- `healthcheck`: concrete validation steps (e.g., `opencode mcp debug <name>`; port reachability)
- `rollback`: exact files/flags to revert

## Engine Packs: Unity vs Unreal (v1 design)

Engine packs are special because they install into a user-owned game project, not into SymphonyStudio.

### Unity pack (selected for v1: IvanMurzak Unity-MCP)

- Installation options supported upstream:
  - OpenUPM: `openupm add com.ivanmurzak.unity.mcp` (scriptable)
  - Unitypackage installer from GitHub Releases (manual import)
  - (Avoid floating branches; pin the plugin version, e.g., `0.51.4` in `package.json`)
- Server lifecycle: Unity plugin auto-downloads the correct server binary from GitHub Releases and starts it.
- Transport: server runs `streamableHttp` by default; OpenCode should connect via HTTP.
- Critical detail: plugin port is deterministic from Unity project path (SHA256 mapped to 50000–59999) per upstream guidance; do not hardcode `8080`.
- SymphonyStudio/OpenCode config should enable a remote MCP entry pointing at `http://localhost:<computed_port>`.
  - Installer can compute the port from the project directory path without opening Unity.

### Unreal pack (recommended: UnrealGenAISupport)

- Installation is a project plugin: add the plugin into the Unreal project under `Plugins/GenerativeAISupport`.
  - Upstream docs recommend git submodule installation to that path.
- MCP connection model: MCP client launches a Python script from the plugin directory (`.../Content/Python/mcp_server.py`), with env vars `UNREAL_HOST`/`UNREAL_PORT`.
- SymphonyStudio/OpenCode config should enable a local MCP entry with `command: ["python", "<project>/Plugins/GenerativeAISupport/Content/Python/mcp_server.py"]`.
- Version pinning: prefer tag/commit in submodule to make installs deterministic.

### Installer implications

- The initializer should generate engine-specific instructions plus an automated "repo fetch" step:
  - Unity: write the exact UPM git URL (pinned), and configure `opencode.jsonc` for the expected port.
  - Unreal: add/update a submodule at the correct path (pinned), and configure the MCP command path.
- The initializer must treat engine installs as *validated but partially manual*:
  - Validate file layout (plugin exists) and ports/scripts are reachable only after the user opens the engine and starts/permits the server.

## Risks / Watchouts

- Secrets handling must avoid committing credentials; prefer env vars and/or local-only `.gitignore`d files.
- Editor-coupled MCPs should be “deferred install” with clear prerequisites (host app running, plugin installed, port reachable).
- Default config should be conservative: no enabled MCPs requiring credentials by default.
- Avoid conflating "engine choice" with "MCP choice": engine packs should primarily install/enable editor/plugin integration and project templates; MCP packs should be capability-based (capture, asset pipeline, PM/docs).
- Ensure idempotency: re-running init should detect existing values and offer merge/keep/overwrite behavior.

- Workspace isolation vs editor-coupled MCP: Symphony runs agents inside per-issue workspace clones. Unity/Unreal MCP servers generally operate on the project currently opened in the editor. If the editor is opened on the developer's original checkout instead of the workspace clone, MCP-driven edits will violate the "work only in provided repository copy" contract.
  - Decision (v1): treat engine MCP packs as *local interactive tooling only*; Symphony unattended runs must not depend on Unity/Unreal MCP.

## Open Decisions

- What does “imported to the user’s project” mean operationally? (config overlays vs language packages vs git submodules)
- Which Google APIs are actually required for v1 (Drive? Docs? Gmail? YouTube? Sheets?)
