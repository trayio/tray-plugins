# Tray.ai for Claude Code & Codex

Build, modify, and manage [Tray.ai](https://tray.ai) workflows using natural language, right inside Claude Code or the [OpenAI Codex CLI](https://developers.openai.com/codex).

The `tray-workflows` plugin connects your AI IDE to your Tray workspace so you can describe an integration in plain English — *"sync new Salesforce leads to Slack"* — and have it planned, built, validated, and test-fired without leaving your terminal.

> **⚠️ The plugin acts as you, in the Tray workspace you sign in to.** It operates with **your Tray identity and permissions**. It can create, modify, and **delete** projects, workflows, and authentications, and — with your explicit go-ahead — run workflows that have real side effects. Every action is taken as you, in that workspace.

## What you get

- **Workflow builder** — plan, build, and validate complete Tray workflows from a natural-language description
- **Connector research** — discover any connector's operations, authentication, and required fields on demand
- **Built-in validation** — every change is checked against Tray's structural rules so workflows are correct before you run them
- **Run & debug in-session** — trigger workflows, inspect executions, and drill into per-step input/output without switching tools

## Requirements

- [Claude Code](https://claude.com/claude-code), **or** the [OpenAI Codex CLI](https://developers.openai.com/codex) (a build with the plugin marketplace)
- A [Tray.ai](https://tray.ai) account

## Installation

### Claude Code

**1. Add the Tray plugin marketplace**

In Claude Code, run:

```
/plugin marketplace add trayio/tray-plugins
```

This adds the Tray plugin catalog to Claude Code.

**2. Install the tray-workflows plugin**

```
/plugin install tray-workflows@tray-plugins
```

This installs the plugin.

**3. Reload plugins**

After installation completes, run:

```
/reload-plugins
```

This loads the plugin into your current session.

**4. Authorize on first use**

The first time you use a Tray command, your browser will open to complete OAuth2 authorization. After authorizing, return to Claude Code — you're ready to build.

Authentication uses OAuth2 with PKCE for secure, automatic token management. No API tokens to create or manage manually.

That's it — the plugin connects to Tray's hosted service automatically. There's nothing to build or run locally.

### Codex

From a Codex build with the plugin marketplace:

**1. Add the marketplace and install**

```
codex plugin marketplace add trayio/tray-plugins
codex plugin add tray-workflows@tray-plugins
```

**2. Authorize** the remote Tray MCP server (opens your browser):

```
codex mcp login tray
```

For older Codex builds, the manual `config.toml` path, and troubleshooting, see **[README.codex.md](./README.codex.md)**.

## Getting started

Once installed, just describe what you want to build:

```
Build a workflow that syncs new Salesforce leads to Slack
```

Your AI IDE will plan the workflow, research the connectors it needs, build the steps, and validate the result.

## Documentation

For detailed guides, troubleshooting, and advanced usage, see the [Tray Headless for Claude Code documentation](https://tray.ai/documentation/platform/tray-headless/headless-for-claude-code).

## License

[Apache-2.0](./LICENSE)
