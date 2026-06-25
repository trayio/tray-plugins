---
name: build-workflow
description: |
  Build or modify Tray workflows. Use this skill whenever the user asks to create, build, or modify a workflow — including requests like "build a workflow that...", "create a workflow to...", "set up an integration between...", "automate X", or "modify the workflow that...".

  Examples of when this skill MUST be loaded:
  - "Build a workflow that syncs Salesforce leads to Slack"
  - "Create a scheduled job that sends a daily email report"
  - "Set up an integration between HubSpot and Google Sheets"
  - "Modify the existing lead scoring workflow to add error handling"
  - "I have a Workato recipe — can you recreate it in Tray?"

  This skill is the entry point for ALL workflow creation and modification. Load it BEFORE making any create_workflow, add_workflow_steps, or update_workflow_steps calls.
---

# Build Tray Workflow

Follow this process for every workflow build or modification. Do not skip the plan step.

## Active workspace — use this, never choose another

All Tray MCP calls operate on the **Active Tray Workspace** recorded in this project's `AGENTS.md` (look for a line like `Active Tray Workspace: <uuid>`). If the project `AGENTS.md` has no such line, check `~/.codex/AGENTS.md`.

- Pass exactly that workspace ID as the `workspaceId` argument on every workspace-scoped call (`create_project`, `create_workflow`, `list_projects`, `get_workflow`, `list_authentications`, …).
- Do **not** call `list_authentications` / `list_projects` to *discover* or *pick* a workspace, and never build in or read from any other workspace — even if other workspace IDs appear in tool output.
- If no Active Tray Workspace is recorded anywhere, **stop and ask the user for their workspace ID** — do not guess or pick one. Offer to record it for future sessions (the `set-workspace` skill writes it into `AGENTS.md`).

## Core principles

- **Discover technically, clarify strategically.** Use MCP tools to answer technical questions without asking. Surface design questions before committing to an approach.
- **Understand before building.** Check the existing workspace, ask 2-3 clarifying questions about intent, scope, or tradeoffs.
- **Batch questions.** Ask everything in one message.
- **Never guess field values.** When DDL can't reveal values, always ask the user explicitly.
- **Salesforce custom fields (`*__c`) require explicit handling.** Their valid values are org-specific and unknown to the connector schema. Before setting any value on a `*__c` field, either DDL its picklist, ask the user, or — if the user already supplied the value — still call out the assumption in the build summary ("Built with `Priority__c = 'High'` — assumes that's a valid picklist value in your org"). Building silently is a fail. See `tray-gotchas` for the rule.
- **Build in Tray, not local files.** The source of truth is always in the Tray workspace via MCP tools — never edit local JSON directly.
- **Never build workflows in parallel.** Dependencies (callable workflows) must be created first.
- **Chain version_ids.** Every mutating tool returns a new `version_id` that must be used for the next operation.
- **Default to delegating connector research to the Researcher subagent.** Spawn an agent matching the description "Research a Tray connector — discover version, operations, auth, required fields, DDL values, and return a step-level prescriptive result". You hand it an intent (connector + operation + the user's data flow) and it hands back a typed `properties` block ready to drop into `update_workflow`. The parent agent should not call `list_connector_operations` itself — operation schemas are absorbed by the Researcher, the parent just reads back a step-input ready to use. Calling `list_connector_operations` directly is a fallback for trivial lookups (single connector, single small operation, no DDL fields), not the default path.
- **Track progress** through multi-step builds (use your plan/todo facility) so dependent steps run in the right order.

## Related skills

- `research-connector` — manual connector research (when not using the agent)
- `tray-connectors` — quick lookup for core connector versions and property types
- `tray-patterns` — workflow structure reference, trigger properties, branch/loop patterns
- `tray-gotchas` — error debugging and edge cases
- `set-workspace` — set a per-project workspace override (user-invoked)

**`get_workflow` defaults to `view: "structure"`** — a cheap structure tree + per-step index without properties. Use `view: "step"` (with `step_names`) to drill into specific steps' full properties. Reach for `view: "full"` only when you genuinely need the entire JSON (rare); don't dump that into the conversation when summarising — point the user at the Tray UI to inspect.

## Step 1: Plan (mandatory — before any API calls)

### For new workflows:

**1a. Understand the request:**
- Identify ambiguities, implicit assumptions, or unstated edge cases
- Check existing project/workspace with `list_projects` for related workflows
- Surface design questions: scope (one workflow or several?), trigger choice tradeoffs, error handling needs, edge cases (no results, duplicates, failures)

**1b. Produce a structured plan and present it to the user:**

```
WORKFLOW PLAN
─────────────
Project: [name]
Workflow: [name]

Trigger: [type] ([connector] v[version], [operation])
  Config: [schedule, webhook URL, etc.]

Steps:
  1. [connector]-1: [operation] — [purpose]
  2. [connector]-2: [operation] — [purpose]
  ...

Branching:
  - [step]: [condition] → true: [steps], false: [steps]

Error Handling:
  - [step]: manual error handling → error: [action], success: [continue]

Auth Required:
  - [connector]: [existing auth name / needs new auth]

Callable Dependencies:
  - [workflow A] must be created before [workflow B]
```

**1c. Wait for user approval before proceeding.**

### For modifications to existing workflows:

- Fetch current state with `get_workflow(workflow_id)`
- Show the user what exists and what will change
- Get approval before making changes

## Step 2: Research Connectors

For each connector in the approved plan — **including the trigger connector** — delegate research to the Researcher subagent. The trigger is a connector like any other: it has an inputSchema with required fields, possibly DDL lookup values (e.g. timezones), and the Researcher will return a step-level prescriptive result for it just like any operational step. Do not skip trigger research or rely on memorized defaults.

**Trigger choice is decided at plan time, not research time.** Map the user's wording to the right trigger connector before you spawn the Researcher — see `tray-connectors#picking-the-trigger-from-the-users-prompt`. Common miss: prompts that mention "form" / "submit" / "form fields" must use `form-trigger`, not `webhook`. The two have different downstream output shapes — picking webhook for a form-shaped prompt builds a workflow that can't read the form's typed fields. Webhook is for incoming HTTP from external systems (Stripe, Zendesk, etc.), not for form submissions.

If `list_connectors(connector_names: [...])` reports "N/M found" and the missing names include `form-trigger`, `callable-trigger`, `agent-tool-trigger`, or any other trigger-only connector, that's a quirk of the REST endpoint — those exist but only show up under `list_connectors(is_trigger: true)`. Re-query with `is_trigger: true` (or with `connector_names` AND `is_trigger: true`) before assuming the trigger isn't available.

**Default path: delegate to the Tray Researcher subagent.** This repo ships a `tray-researcher` subagent (`agents/tray-researcher.toml`); spawn it (e.g. "spawn the tray-researcher agent to research …", or via `/agent`) and hand it the research intent. If subagents aren't available in your Codex version, run the research inline by following the `research-connector` skill — but keep the verbose schema work in a tight loop and only carry the prescriptive result forward.

```
Spawn: tray-researcher
prompt: "Research [connector] for [operation].
Auth to check: [name or service]. Fields needed: [list from plan].
Workflow context: [trigger output shape, upstream step outputs]."
```

The Researcher will return a typed `properties` block per operation, ready to drop into `update_workflow`. Pass the returned block straight through — you do not need to read the operation's inputSchema yourself. **For multiple independent connectors, spawn the Researcher in parallel** — each subagent runs on its own MCP queue, returns a concise prescriptive summary, and keeps the verbose DDL isolated in its own context. Only spawn sequentially when one Researcher's output genuinely feeds the next (rare).

**Fallback: call `list_connector_operations` directly** only when the operation is trivial (no DDL fields, no polymorphism, no auth setup) AND you're confident the round-trip cost beats spawning a subagent. The tool returns compacted schemas by default — `advanced` fields stripped, polymorphic structures (`oneOf`, deeply-nested array items, large enums) replaced with `_collapsed` markers carrying `expand: ['<propName>']` hints. If you do call it directly: read the markers, decide which branches to expand, and re-call with the named property in the `expand` array. Pass `include_advanced: true` only when an advanced field is actually required, and `include_output_schema: false` for terminal steps with no downstream jsonpath consumers.

**Existing-auth lookups: filter `list_authentications` by `service_name`, not by keyword search.** Auth display names are user-supplied and unreliable — an auth labelled "Sheets" that backs `google-sheets` will not match `search: "google"`, and project-scoped `ai-agent` auths often outrank the real service auth in keyword matches. Pass `service_name: "<service>"` (e.g. `"google-sheets"`, `"slack"`, `"salesforce"`) — the slug is the same one returned in `connector.service.name` from `list_connectors`. The Researcher applies this rule by default; direct lookups from the parent agent should too. If a service-name filter returns 0 rows, fall back to an unfiltered `list_authentications` and inspect the `service` block on each row before keyword search.

### Research checklist (per connector, including trigger):
- [ ] Researcher returned a step-level prescriptive result (typed `properties` block ready for `update_workflow`)
- [ ] Connector version confirmed (the Researcher's summary will name it)
- [ ] Existing auth resolved (via `service_name` filter on `list_authentications`) or new-auth flow surfaced
- [ ] DDL-resolved values present in the returned `properties` (or unknowns flagged for the user)
- [ ] Output schema noted in the Researcher's summary if downstream jsonpath references will reach into it; this comes either from the operation's static `outputSchema` or (when `hasDynamicOutput: true`) from a `call_connector` call with `has_dynamic_output: true` to reveal the dynamic shape. Do NOT test-fire regular mutating operations to "check that auth works" — `call_connector` is gated to DDL / schema-discovery calls by default, and test-firing with production auths can have real side effects.

**CRITICAL: Step properties MUST use exact field names from inputSchema. The MCP server validates properties against the connector schema — made-up field names will be rejected.**

## Referencing Step Output (JsonPath)

When a step needs to reference output from a previous step, build the jsonpath from the **actual output schema of the step being referenced**. Different step types have different output shapes — do not assume a universal pattern.

```
$.steps.<step-name>.<path-based-on-step-output-schema>
```

**How to determine the correct path:** Look at the output returned by the step (from a test `call_connector` call, or from the operation's `outputSchema`). The jsonpath must match the runtime output structure of that specific step.

Special path patterns:
```
$.steps.loop-1.value.<field>              — current loop iteration item
$.steps.call-workflow-1.response.<field>  — callable workflow response
$.env.<variable>                          — environment variable
```

**Note:** The `call_connector` tool response may wrap data differently than the workflow runtime. Always verify the runtime output shape — don't assume the `call_connector` response structure is identical to the runtime jsonpath structure.

## Property Value Wrapping

ALL property values must be wrapped in Tray's type format. Never pass raw values.

```
WRONG:   "hour": "9"
CORRECT: "hour": {"type": "string", "value": "9"}

WRONG:   "limit": 100
CORRECT: "limit": {"type": "integer", "value": 100}

WRONG:   "active": true
CORRECT: "active": {"type": "boolean", "value": true}
```

When referencing another step's output:
```
"channel": {"type": "jsonpath", "value": "$.steps.script-1.channel_id"}
```

The `inputSchema` from `list_connector_operations` shows field types (string, integer, etc.) — these map to the `type` field in the wrapper, NOT to raw values. The MCP server's `validateStepProperties` enforces this format on every mutating call.

### Object array items must be wrapped too

When a property's `inputSchema` is `array` with `items.type: "object"` (or `items.oneOf: [...]`), each item must itself be wrapped as `{"type": "object", "value": {...}}`. The Tray API will silently accept a bare object — the workflow will save fine — but the workflow editor cannot read the inner properties and the step will fail at runtime with "field is required" / "value is required" UI errors. The MCP server's `validateStepProperties` rejects this on every mutation; if you see `Array item must be type-wrapped` in a tool result, this is the rule.

```json
WRONG:
"conditions": {
  "type": "array",
  "value": [
    { "field": {"type":"string","value":"Industry"},
      "operator": {"type":"string","value":"Equal to"},
      "value": {"type":"string","value":"Technology"} }
  ]
}

CORRECT:
"conditions": {
  "type": "array",
  "value": [
    { "type": "object",
      "value": {
        "field": {"type":"string","value":"Industry"},
        "operator": {"type":"string","value":"Equal to"},
        "value": {"type":"string","value":"Technology"}
      } }
  ]
}
```

Common offenders: `salesforce.find_records.conditions`, `salesforce.update_record.fields` (each item needs `key` + `value` wrapped), `agent-tool-trigger` input schemas with array fields, sheet/airtable column-mapping arrays. Primitive arrays (`items.type: "string"|"integer"|...`) just need each item type-wrapped as the primitive — `salesforce.find_records.fields` items are `{"type":"string","value":"Id"}`, not nested-object-wrapped.

## Step 3: Build

### 3a. Analyze dependencies
Identify callable workflows and their dependency chain. Create in order: callable workflows first, callers last. Store each `workflow_id` for reference.

**Callable workflow rule:** If a callable workflow uses the `trigger_and_respond` operation (synchronous), it MUST include a `callable-workflow-response` step — without it, the calling workflow will hang. If it uses the `trigger` operation (async/fire-and-forget), it must NOT have a response step.

**Trigger event reply rule:** Workflows using webhook (`webhook_with_response`), API operation triggers (`request_response` / `publish_subscribe`), or agent tool triggers (`request_response`) need a `trigger-reply` step to return a response. The reply step's `properties` shape is **declared by the trigger op's `replySchema`** — read it from `list_connector_operations`. The shape **splits by trigger family**: webhook + `publish_subscribe` are flat (no envelope); APIM `request_response` and agent-tool-trigger `request_response` use a `{response: {<discriminator>: {...}}}` envelope with a `oneOf` of variants (`successBody`, `errorBody`, `internalErrorBody`, `invalidInputBody`, `forbiddenBody`). For the callable-trigger pairing, the response shape lives on `callable-workflow-response.response.inputSchema` instead. See `tray-gotchas` "Trigger Event Reply Steps" for the per-family rules + worked examples.

### 3b. Create project
```
create_project(name="[from plan]", description="[brief]")
→ Returns: project_id
```

### 3c. Create workflow with trigger

Use the trigger connector's researched inputSchema to set properties. Include ALL `required` fields from the schema — the trigger is configured here, not via `add_workflow_steps`.

**`description` is required for every workflow.** It serves two audiences:

- For most workflows, it provides UI context for humans maintaining the workflow.
- **For `agent-tool-trigger` workflows, the description becomes the tool description an agent reads when deciding whether to invoke the tool.** Write it from the agent's perspective: what the tool does, when to use it, what inputs it expects. A weak description means the agent won't know when to call it. The MCP server enforces a minimum description length for these workflows.

**For `agent-tool-trigger` workflows, `create_workflow` also auto-registers an API operation** under the project's API Management so the workflow is immediately callable as a tool — no manual UI step needed. The operation name is derived from the workflow name (lowercased, non-alphanumerics replaced with `_`); the path is `/ai-agent/{name}`. If the registration fails (e.g., name collision), the workflow is kept and a warning is returned — no automatic rollback.

**For triggers with `DynamicSchema`-tagged fields** (APIM's `request.body` / `request.headers` / `response.<variant>`, agent-tool-trigger's `body.tool_input` / `body.static_data`): the field's `.value` is **raw JSON Schema, not type-wrapped** (see `tray-gotchas` "DynamicSchema content" for the right vs. wrong shape). Inside that schema, include `title` and `description` on every property — Tray's UI renders them as the "Name" and "Description" fields next to each property key, and a blank Name reads as a half-built tool to anyone editing the trigger later.

```
create_workflow(project_id, workflow_name, description, trigger={
  connector_name: "[from research]",
  connector_version: "[from research]",
  operation: "[from research]",
  title: "[descriptive title]",
  properties: {
    // ALL required fields from the trigger operation's inputSchema
    // ALL values type-wrapped: {"type": "string", "value": "..."}
  }
})
→ Returns: workflow_id + version_id
```

**To update metadata later** (description, name, enabled, tags) use `update_workflow_metadata(workflow_id, ...)`. This is the only API path that can change a workflow's description — the versioned PUT endpoint silently ignores description edits. It does not bump the `version_id`, so it's safe to call mid-build without disrupting the step-editing chain.

### 3d. Add the planned steps

**Use `add_workflow_steps` for every step add — pass a one-element array if you only need a single step.** After research you almost always know every step you need for a workflow upfront; emit them in one batched call instead of paying N separate LLM round-trips. The MCP server validates every step's properties before any HTTP call; if any step's properties are malformed the whole batch is rejected with errors aggregated by step name, and you retry with a fixed batch.

```
add_workflow_steps(workflow_id, version_id, steps=[
  {step_name: "salesforce-1", connector_name: "salesforce", connector_version: "8.8",
   operation: "find_records", title: "Find Leads",
   properties: { ... }, auth_id: "<uuid>"},
  {step_name: "script-1", connector_name: "script", connector_version: "3.4",
   operation: "execute", properties: { ... }},
  {step_name: "slack-1", connector_name: "slack", connector_version: "11.0",
   operation: "send_message", properties: { ... }, auth_id: "<uuid>"}
])
→ Returns: new version_id + full step list
```

If a workflow has more than ~15 steps, split into 2 batched calls to stay safely under the model's output budget. If you genuinely need to inspect intermediate state between additions (e.g. fetching DDL to decide step N+1), call `add_workflow_steps` with the steps you have so far, then call it again with the next batch — the version_id chains the same way.

### 3e. Configure step properties (if not set during add) — or fix N steps from one round of validator errors

Use `update_workflow_steps` with an array of `{step_name, properties, operation?, auth_id?, error_handling?}` entries. Pass one entry for a single update; pass many to fix several steps in one round-trip. The `properties` object on each entry is the FULL replacement (not a merge/patch).

`update_workflow_steps` can also **swap the operation** on an existing step — useful when modifying a workflow in place rather than removing and re-adding. Set `operation` to the new op name on the same connector and the new properties are validated against the new operation's schema before the call goes out. Connector + version cannot change — remove and re-add the step to switch connectors. When an operation is swapped, the cached `output_schema` is dropped so any downstream jsonpath references are re-resolved against the new operation's outputs.

```
update_workflow_steps(workflow_id, version_id, steps=[
  {step_name: "salesforce-1",
   properties: { ... },   // FULL properties object — replaces, not merges
   auth_id: "<uuid>"},    // pass the `id` field from list_authentications, NOT `group`
  {step_name: "text-helpers-1",
   operation: "concatenate",  // swap to a different operation on the same connector; properties validated against the new op
   properties: { values: { ... }, separator: { ... } }},
  {step_name: "slack-1",
   properties: { ... },
   error_handling: "manual"}  // step becomes branch with empty error/success paths
])
→ Returns: new version_id + the list of updated step names + any operation changes
```

### 3f. Reorganize structure (move steps into branches, reorder)
```
get_workflow(workflow_id)  // default view: "structure" — returns the tree + step index
update_workflow_structure(workflow_id, version_id, structure=[...])
→ Returns: new version_id
```

## Step 4: Validate

**Structural audit is automatic.** Every mutating workflow tool (create_workflow, add_workflow_steps, update_workflow_steps, remove_workflow_step, update_workflow_structure, update_workflow_metadata, import_workflow) runs `validate_workflow` server-side and returns any findings in the same tool response. You don't need to invoke it manually in the common case — if there are structural issues, they'll arrive alongside the tool result and you should fix them before handing off to the user.

**When validator errors flag multiple steps at once, fix them in one batched call** to `update_workflow_steps` rather than N sequential singular updates — same atomic per-step validation as the add path, single version mint, single round-trip.

**Call `validate_workflow(workflow_id)` explicitly when you want an on-demand audit** — for example, after multiple rapid mutations, or at the end of a long build to confirm clean state.

The validator checks:
- Every `{type:"jsonpath", value:"$.steps.<name>..."}` resolves — step exists AND the path matches the referenced step's effective output schema. Shape rules: `script` wraps in `result`, `loop` in `value`, `call-workflow` in `response`, `raw_http_request` depends on `parse_response`. Trigger envelopes (`webhook → body`, `form-trigger → result`, `callable-trigger → data`, `agent-tool-trigger → tool_input` / `static_data`) are documented in `tray-patterns` and `tray-connectors`
- Structural rules — no `next` pointers; loop uses `_loop` wrapper; branch has path keys; trigger step named `trigger`; scheduled-daily has all 7 day_of_week keys; `trigger_and_respond` callable has a `callable-workflow-response` step while async `trigger` callables do not; oneOf array items are type-wrapped
- Step naming follows `<connector>-N`

Plan-vs-build comparison (does this workflow match what we agreed on?) stays in the main conversation — walk the plan's steps against `get_workflow` output. That's judgement, not a deterministic rule.

## Step 5: Test (only with explicit permission)

`trigger_workflow` fires a workflow against the live Tray workspace — not a dry-run, not a sandbox. Each fire executes every step, creates an execution log entry, consumes API quota, and runs every connector against whatever real systems it's wired to (Slack messages get posted, Salesforce records get written, third-party webhooks get called).

**Always ask permission explicitly before firing.** Be clear about what will happen:

- "I'll fire the workflow against your live Tray workspace — this creates an execution log entry and consumes API quota."
- For workflows whose downstream steps touch external systems: "This will also send a real Slack message / create a real Salesforce record / call <third-party API>. Should I proceed?"

WRONG — vague:
> "Ready to test?"
> "I'll proceed with testing now."

RIGHT — explicit:
> "Ready to fire the workflow in your Tray workspace? This will create execution logs and consume API resources, and the Slack step will post to #demo-channel."

After the user confirms, `trigger_workflow` returns a `workflow_instance` and the literal `fired_payload_shape`. Verify your jsonpaths against `fired_payload_shape`, then call `get_workflow_execution(workflow_id, workflow_instance)` to inspect the run. For failed steps, `get_workflow_step_detail` drills into input/output. See `tray-gotchas` Runtime Debug Flow for the full loop.

## MCP Tool Reference

These tools come from the `tray` MCP server (configured in `~/.codex/config.toml`). Codex exposes them namespaced to that server — the exact surface form depends on your Codex version (commonly `tray.<tool>` or `tray__<tool>`); the bare tool names below are what matters.

### Tool Usage Notes
- **Version ID chaining**: Every mutating tool returns a `version_id`. The next operation MUST use that `version_id`. Stale version IDs cause API errors (which include the latest version ID for recovery)
- **Incremental building**: `create_workflow` → `add_workflow_steps` → `update_workflow_steps` (per-step refinement) → `update_workflow_structure` (reorganise) → ...
- **`add_workflow_steps` / `update_workflow_steps`**: The canonical step-mutating tools. One round-trip each, atomic per-step validation, errors aggregated by step name. Pass a single-element array for a one-step change — there are no singular variants.
- **SetStepData replaces entire properties**: `update_workflow_steps` replaces the full properties object on each targeted step — not a merge/patch. Always send the complete property set per step.
- **auth_id on steps**: Both step-mutating tools accept optional `auth_id` per step entry for `metadata.auth_uuid`
- **error_handling on steps**: Set to `"manual"` to enable manual error handling — changes step type from `"normal"` to `"branch"` with empty `success`/`error` paths. Follow up with `update_workflow_structure` to place steps in those paths
- **update_workflow_structure**: Move steps into branch paths (error/success/true/false), reorder steps, restructure layout. Call `get_workflow` first (default `view: "structure"`) to see current structure
- Store workflow IDs during creation for referencing in dependent workflows

### Connector Research Tools
- `list_connectors` — Search connectors. Use `connector_names` array for exact lookup (keyword search can be unreliable). Set `is_trigger: true` for trigger connectors
- `list_connector_operations` — Get operations with input/output schemas
- `call_connector` — Execute operations including DDL for field discovery
- `list_service_environments` — Get auth environments. **Requires `service_name` (string, e.g., `"google-gmail"`) and `service_version` (integer from `list_connectors` → `service.version`)** — NOT `service_id`
- `list_authentications` — List existing auths. **Always check before creating new auths**
- `create_auth_collection` — Create auth requests. **Requires `service` (UUID from `list_service_environments`) and `service_environment_id`** — NOT `connector_name`/`connector_version`
- `check_auth_completion` — Check auth completion status

### Failure Recovery

| Problem | Action |
|---|---|
| Missing authentication | Set up via `list_service_environments` → `create_auth_collection` (see `research-connector` for parameter mapping), test, retry |
| Unclear field values | Ask user — never guess |
| DDL discovery incomplete | Run all relevant DDL operations before proceeding |
| Stale version_id error | Use the latest version_id from the error response and retry |
| Step add/update fails | Use `get_workflow` to inspect current state, fix, retry |
| Property validation error | Check field names against `list_connector_operations` inputSchema |
