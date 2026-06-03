# Tray.ai for Claude Code

Build, modify, and manage [Tray.ai](https://tray.ai) workflows using natural language, right inside Claude Code.

The `tray-workflows` plugin connects Claude Code to your Tray workspace so you can describe an integration in plain English — *"sync new Salesforce leads to Slack"* — and have it planned, built, validated, and test-fired without leaving your terminal.

> **⚠️ The plugin acts in your Tray workspace as you.** Once you sign in, the plugin operates with **your Tray identity and permissions**. It can create, modify, and **delete** projects, workflows, and authentications, and — with your explicit go-ahead — run workflows that have real side effects. Every action is taken as you and only ever against the workspace you've configured.

## What you get

- **Workflow builder** — plan, build, and validate complete Tray workflows from a natural-language description
- **Connector research** — discover any connector's operations, authentication, and required fields on demand
- **Built-in validation** — every change is checked against Tray's structural rules so workflows are correct before you run them
- **Run & debug in-session** — trigger workflows, inspect executions, and drill into per-step input/output without switching tools

## Requirements

- [Claude Code](https://claude.com/claude-code) installed
- A [Tray.ai](https://tray.ai) workspace and your **Workspace ID**

## Installation

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

This installs the plugin and prompts you for configuration.

**3. Provide your Workspace ID**

You'll be prompted for your **Tray Workspace ID** during installation. To find it:

1. Go to your Tray workspace in your browser
2. Look at the URL: `https://app.tray.io/workspaces/<YOUR-WORKSPACE-ID>/...`
3. Copy the UUID from the URL (the part after `/workspaces/`)

That's your Workspace ID.

**4. Reload plugins**

After installation completes, run:

```
/reload-plugins
```

This loads the plugin into your current session.

**5. Authorize on first use**

The first time you use a Tray command, your browser will open to complete OAuth2 authorization. After authorizing, return to Claude Code — you're ready to build.

Authentication uses OAuth2 with PKCE for secure, automatic token management. No API tokens to create or manage manually.

That's it — the plugin connects to Tray's hosted service automatically. There's nothing to build or run locally.

## Getting started

Once installed, just describe what you want to build:

```
Build a workflow that syncs new Salesforce leads to Slack
```

Claude Code will plan the workflow, research the connectors it needs, build the steps, and validate the result.

## Switching workspace

The Workspace ID you enter at install is your **default** — every build targets it automatically.

To build in a **different** workspace for a specific project, run:

```
/tray-workflows:set-workspace <your-workspace-id>
```

This saves a per-project override in that project's local Claude Code settings; your default and other projects are unchanged. Run `/reload-plugins` afterwards to apply it.

## Documentation

For detailed guides, troubleshooting, and advanced usage, see the [Tray Headless for Claude Code documentation](https://tray.ai/documentation/platform/tray-headless/headless-for-claude-code).

## License

[Apache-2.0](./LICENSE)
