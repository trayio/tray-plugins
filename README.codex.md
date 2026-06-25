# Tray.ai for Codex

Build, modify, and manage [Tray.ai](https://tray.ai) workflows using natural language, right inside the [OpenAI Codex CLI](https://developers.openai.com/codex).

This is the Codex packaging of Tray.ai's **[`tray-workflows`](https://github.com/trayio/tray-plugins)** workflow tooling (also available as a [Claude Code plugin](https://github.com/trayio/tray-plugins)). It connects Codex to your Tray workspace via the hosted **`tray` MCP server**, so you can describe an integration in plain English — *"sync new Salesforce leads to Slack"* — and have it planned, built, validated, and test-fired without leaving your terminal.

> **⚠️ The tooling acts in your Tray workspace as you.** Once you sign in, it operates with **your Tray identity and permissions**. It can create, modify, and **delete** projects, workflows, and authentications, and — with your explicit go-ahead — run workflows that have real side effects. Every action is taken as you and only ever against the workspace you've configured.

## What you get

- **Workflow builder** — plan, build, and validate complete Tray workflows from a natural-language description (the `build-workflow` skill)
- **Connector research** — discover any connector's operations, authentication, and required fields on demand (the `tray-researcher` subagent + `research-connector` skill)
- **Built-in validation** — every change is checked against Tray's structural rules so workflows are correct before you run them
- **Run & debug in-session** — trigger workflows, inspect executions, and drill into per-step input/output

It's the same Tray domain knowledge as the Claude Code plugin, repackaged for Codex's conventions (`config.toml` MCP servers, `AGENTS.md`, skills, and subagents).

## Requirements

- [Codex CLI](https://developers.openai.com/codex) installed and on your `PATH`
- A [Tray.ai](https://tray.ai) workspace and your **Workspace ID**

## Installation

> **Requires** a Codex build with the plugin marketplace (`codex plugin --help` lists `marketplace`). On older builds, use the manual option below.

### Option A — Codex plugin marketplace (recommended)

```bash
codex plugin marketplace add trayio/tray-plugins
codex            # then run /plugins to install "tray-workflows"
```

This bundles and registers the skills, the `tray-researcher` subagent, and the remote `tray` MCP server — no scripts, no files copied into your home directory by us. Because the Tray server is **remote (HTTP) and OAuth-based**, after install run `codex mcp login tray` (see [Authorize](#authorize)) and record your workspace ID (see [Set your workspace](#set-your-workspace)).

### Option B — Manual (older Codex builds)

1. **Add the MCP server.** Merge the block from [`examples/config.toml`](./examples/config.toml) into `~/.codex/config.toml`:

   ```toml
   [mcp_servers.tray]
   url = "https://api.tray.io/mcp"
   ```

   If `codex mcp list` doesn't show `tray`, your Codex build may gate HTTP MCP behind a flag — add `[features]\nexperimental_use_rmcp_client = true` and restart. (Try without it first.)

2. **Install the skills** — copy `plugins/codex/tray-workflows/skills/*` into `~/.agents/skills/` (global) or `.agents/skills/` in your project.

3. **Install the subagent** — copy `plugins/codex/tray-workflows/agents/tray-researcher.toml` into `~/.codex/agents/`.

4. **Add the instructions** — copy the `tray-workflows` block from [`AGENTS.md`](./AGENTS.md) into your project `AGENTS.md` (or `~/.codex/AGENTS.md`) and fill in your workspace ID.

## Authorize

The Tray MCP server uses OAuth2 (PKCE) — sign in once:

```bash
codex mcp login tray
```

Your browser opens to authorize. After that, Codex manages tokens automatically. There are no API tokens to create or manage manually.

## Set your workspace

Codex has no per-plugin config substitution, so the active workspace is recorded as a line in `AGENTS.md`:

```
Active Tray Workspace: <your-workspace-uuid>
```

Find the UUID in the Tray app URL: `https://app.tray.io/workspaces/<ID>/...`.

Ask Codex to run the **`set-workspace`** skill (it updates the `AGENTS.md` block), or edit the line by hand. Project `AGENTS.md` overrides the global `~/.codex/AGENTS.md`, so different repos can target different workspaces.

## Getting started

Start a new Codex session in a project that has the Tray instructions, then describe what you want:

```
Build a workflow that syncs new Salesforce leads to Slack
```

Codex will load the `build-workflow` skill, plan the workflow, research the connectors it needs (via the `tray-researcher` subagent), build the steps, and validate the result before anything runs.

## What's in the Codex bundle

This repo serves both the Claude Code plugin (`.claude-plugin/`, `plugins/tray-workflows/`)
and the Codex bundle below. Each tool reads only its own files.

```
tray-plugins/
├── AGENTS.md                         # Tray instructions block (copy into ~/.codex/AGENTS.md)
├── examples/config.toml              # manual MCP server snippet (Option B)
├── .agents/plugins/marketplace.json  # Codex marketplace catalog (Option A)
└── plugins/codex/tray-workflows/
    ├── .codex-plugin/plugin.json     # Codex plugin manifest
    ├── .mcp.json                     # plugin MCP server definition
    ├── agents/tray-researcher.toml   # connector-research subagent
    └── skills/
        ├── build-workflow/           # entry point for all workflow builds
        ├── research-connector/       # connector discovery via DDL
        ├── tray-connectors/          # core connector names/versions + type-wrapping
        ├── tray-patterns/            # structure: branches, loops, callables, scheduling
        ├── tray-gotchas/             # error debugging + edge cases
        └── set-workspace/            # record the active workspace in AGENTS.md
```

## Notes & caveats

- **Same Tray knowledge, Codex packaging.** The Tray workflow logic mirrors the Claude Code plugin; the Codex packaging (config, skills format, subagent) is adapted to Codex conventions and verified by structure, not against every Codex release.
- **Codex versions vary.** Remote/HTTP MCP support, skills, subagents, and the plugin marketplace have all landed at different points in Codex's history. Option A targets builds with the plugin marketplace; Option B (manual) adapts to older builds. See the inline comments in `examples/config.toml`.
- **No code runs locally.** This bundle ships no executable code — only declarative config, skills, and an MCP server URL. The `tray` MCP server is hosted by Tray; there's nothing to build or run.

## Credits & license

Tray workflow knowledge and skills are shared with the **[trayio/tray-plugins](https://github.com/trayio/tray-plugins)** Claude Code plugin by Tray.ai.

[Apache-2.0](./LICENSE)
