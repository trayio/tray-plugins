# Changelog

All notable changes to the tray-workflows plugin are documented here.
This project follows Semantic Versioning.

## [2.0.1] — 2026-08-19

### Fixed

- **Callable-workflow test-fires now use the real runtime envelope.** `trigger_workflow` on a callable workflow takes its input fields flat — `data: {<your fields>}`, surfacing at `$.steps.trigger.<field>` — exactly the shape a real `call-workflow` step produces. Previously, jsonpaths that passed a test-fire could return `undefined` when the workflow was invoked for real. Legacy wrapped payloads are still accepted, and the `tray-gotchas` skill reflects the corrected shape.
- **Unknown operations are rejected at write time.** `add_workflow_steps` / `update_workflow_steps` fail immediately when a step names an operation the connector doesn't have, instead of saving a step that only fails at runtime.
- **`$.auth.` jsonpaths validate cleanly.** The workflow validator accepts `$.auth.*` references for injecting authentication values into connector fields, instead of flagging them as unknown roots.
- Removed a misleading step count and dead cursor fields from `get_workflow_execution` / `get_workflow_step_detail`.

### Added

- **Step-level pagination for large executions.** `get_workflow_execution` and `get_workflow_step_detail` can now page through executions with many steps (`step_before_id` / `step_after_id` inputs, `next_step_cursor` output) and report how many steps were truncated, instead of silently cutting off.

## [2.0.0] — 2026-07-08

**Breaking change:** your Tray workspace is no longer set in plugin config — it's chosen at sign-in.

### Changed

- **No more workspace config.** The `workspaceId` option is removed; you select your workspace during OAuth sign-in (it comes from the grant), not from config. Existing `workspaceId` values are ignored — remove them. If you previously pinned a workspace, re-authenticate and pick it at the consent screen.
- **`TRAY_MCP_URL` override** — point the plugin at a different MCP server URL via an environment variable.

### Removed

- The `set-workspace` skill and the workspace-config setup step.

## [1.0.3] — 2026-06-22

### Added

- **Codex packaging** for `tray-workflows` — the plugin is now installable in the [OpenAI Codex CLI](https://developers.openai.com/codex) (`codex plugin marketplace add trayio/tray-plugins`) alongside Claude Code. Same workflow tooling and skills, repackaged for Codex conventions (`AGENTS.md`, `.codex-plugin` manifest, TOML subagent). No change to the plugin's behaviour in Claude Code.

## [1.0.2] — 2026-06-03

Initial public release of **tray-workflows**, the Tray plugin for Claude Code — part of Tray Headless.

### Added

- Build, run, and validate Tray workflows from natural language, inside Claude Code.
- Connector research — discover any connector's operations, authentication, and required fields on demand.
- Built-in validation — every change is checked against Tray's structural rules before it runs.
- In-session run and debug — trigger a workflow, inspect executions, and drill into per-step input and output.
- OAuth2 authentication (PKCE) — sign in once in the browser; no API token to manage.
