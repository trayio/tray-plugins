# Changelog

All notable changes to the tray-workflows plugin are documented here.
This project follows Semantic Versioning.

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
