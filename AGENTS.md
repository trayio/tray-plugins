<!-- tray-workflows:begin -->
# Tray Headless for Codex

These instructions let Codex build, modify, run, and debug [Tray.ai](https://tray.ai)
workflows in natural language, via the **`tray` MCP server**.

> Copy this block into the `AGENTS.md` of any project where you want to build Tray
> workflows (or into `~/.codex/AGENTS.md` to make it global). Keep the
> `tray-workflows:begin`/`end` markers.

## The Tray MCP server

The `tray` server (configured in `~/.codex/config.toml`) exposes the workflow tools:
`list_connectors`, `list_connector_operations`, `call_connector`,
`list_authentications`, `list_service_environments`, `create_auth_collection`,
`create_project`, `create_workflow`, `add_workflow_steps`, `update_workflow_steps`,
`update_workflow_structure`, `update_workflow_metadata`, `get_workflow`,
`validate_workflow`, `trigger_workflow`, `get_workflow_execution`,
`get_workflow_step_detail`, and more. Codex namespaces them to the `tray` server.

If a Tray tool errors with an auth/permission failure, the user likely needs to
sign in: `codex mcp login tray`. **The server acts in the workspace as the signed-in
user** — it can create, modify, and delete projects, workflows, and auths, and run
workflows with real side effects. Always confirm before firing a workflow.

## How to build (load the skills)

This project ships skills under `plugins/codex/tray-workflows/skills/`. Load and follow
them — they carry the detailed, load-bearing rules. Do not improvise Tray internals.

- **`build-workflow`** — the entry point for ALL workflow creation/modification.
  Load it before any `create_workflow` / `add_workflow_steps` / `update_workflow_steps`
  call. It defines the mandatory plan → research → build → validate → test flow.
- **`research-connector`** — discover a connector's version, operations, auth, required
  fields, and DDL values before building a step.
- **`tray-connectors`** — core connector names/versions, the type-wrapper format,
  step-naming, trigger selection, and jsonpath shapes.
- **`tray-patterns`** — workflow structure: branches, loops (`_loop` wrapper), callable
  workflows, scheduled triggers, manual error handling, version_id chaining.
- **`tray-gotchas`** — error debugging and known-weird surfaces.

For connector research, prefer spawning the **`tray-researcher`** subagent
(`plugins/codex/tray-workflows/agents/tray-researcher.toml`) — it absorbs verbose schemas
and returns a typed, drop-in `properties` block. If subagents aren't available in your
Codex version, follow the `research-connector` skill inline.

## Non-negotiable rules (full detail in the skills)

- **Plan first.** Present a structured plan and get approval before any mutating call.
- **Build in Tray, not local files.** The source of truth is the workspace via MCP
  tools — never hand-edit workflow JSON locally.
- **Type-wrap every property value** — `{"type": "string", "value": "..."}`,
  `{"type": "jsonpath", "value": "$.steps...."}`, etc. Array items wrap too.
- **Chain `version_id`.** Every mutating tool returns a new one; the next call must use it.
- **Never guess field values.** DDL the picklist or ask the user — especially Salesforce
  `*__c` custom fields.
- **Ask explicit permission before `trigger_workflow`** — it fires against the live
  workspace, consuming quota and hitting real downstream systems.
<!-- tray-workflows:end -->
