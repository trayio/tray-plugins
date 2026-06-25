---
name: tray-patterns
description: 'Load when structuring a workflow — branches, loops (the `_loop` wrapper), callable workflows (sync vs async + the `callable-workflow-response` pairing), scheduled triggers (the full `day_of_week` object), manual error handling (`error_handling: "manual"` + `update_workflow_structure`), and the mandatory version_id chaining across mutations.'
---

# Tray Workflow Patterns

Consult this when building workflows or debugging structure issues.

## Incremental Build Flow

Workflows are built step-by-step, not as monolithic JSON:

```
create_workflow(project_id, workflow_name, trigger={...})
  → workflow_id + version_id

add_workflow_steps(workflow_id, version_id, steps=[{...}, {...}, ...])
  → new version_id

update_workflow_steps(workflow_id, new_version_id, steps=[{step_name, properties, ...}, ...])
  → newer version_id
```

**CRITICAL: Version ID chaining** — Every mutating tool returns a `version_id`. You MUST pass it to the next call. Stale version_ids cause API errors (which include the latest version_id for recovery).

## Trigger Properties

For the canonical trigger connector table (names, versions, which triggers need `public_url`), see `tray-connectors`. The one trigger whose structure is worth codifying here is the scheduled `daily` operation — it has the richest required-field shape.

### Scheduled trigger — `daily` operation
```json
{
  "public_url": {"type": "jsonpath", "value": "$.env.public_url"},
  "hour": {"type": "string", "value": "9"},
  "minute": {"type": "string", "value": "0"},
  "tz": {"type": "string", "value": "Europe/London"},
  "day_of_week": {
    "type": "object",
    "value": {
      "monday": {"type": "boolean", "value": true},
      "tuesday": {"type": "boolean", "value": true},
      "wednesday": {"type": "boolean", "value": true},
      "thursday": {"type": "boolean", "value": true},
      "friday": {"type": "boolean", "value": true},
      "saturday": {"type": "boolean", "value": true},
      "sunday": {"type": "boolean", "value": true}
    }
  }
}
```
**`day_of_week` must be explicit** — always set all 7 days. Do not rely on defaults.

**Trigger output uses zero-indexed `month` and `day_of_week`** — January is `0`, December is `11`; Monday is `0`, Sunday is `6`. Branches that compare the trigger's day/month output against a literal day name will silently route wrong on production runs (the validator can't catch it because the jsonpaths are valid). For any date arithmetic prefer the Date Time Helper's `Get Date Property` operation over parsing the trigger payload directly.

## Workflow Structure Reference

When you call `get_workflow` (default `view: "structure"`), the returned JSON contains a `structure` array (the tree) and a `steps` index (per-step `{name, connector, version, operation, title, auth_uuid}`, no properties). Use `view: "step"` with `step_names: [...]` to fetch the full `{name, properties, metadata}` for specific steps. Understanding this structure helps when debugging or inspecting workflows.

### steps_structure Patterns

**Step execution order is determined by array position.** Do NOT use `"next"` keys — they cause validation failure.

### Normal Step
```json
{"name": "script-1", "type": "normal", "content": {}}
```

### Branch (boolean condition)
```json
{
  "name": "boolean-condition-1",
  "type": "branch",
  "content": {
    "true": [{"name": "script-1", "type": "normal", "content": {}}],
    "false": [{"name": "terminate-1", "type": "normal", "content": {}}]
  }
}
```

### Manual Error Handling
Two-step process:

**Step 1** — Enable error handling on the step using `error_handling: "manual"` on the step entry passed to `add_workflow_steps` or `update_workflow_steps`. This creates a branch with empty paths:
```json
{"name": "slack-1", "type": "branch", "content": {"error": [], "success": []}}
```

**Step 2** — Move steps into the error/success paths using `update_workflow_structure`. Example: moving `trigger-reply-1` into the error path:
```json
{"name": "slack-1", "type": "branch", "content": {
  "error": [{"name": "trigger-reply-1", "type": "normal", "content": {}}],
  "success": []
}}
```
Steps inside `error` run when the step fails. Steps inside `success` run when it succeeds. Steps after the branch in the top-level array run regardless of which path was taken.

**Inside an error path, reference the failure via `$.errors.<failed-step-name>.<field>`.** Tray exposes the failed step's error context under `$.errors`, keyed by the step name that errored. The fields available under `$.errors.<failed-step>` mirror the failed step's `output` schema (in Tray's snake_case naming) — for an http-client failure, that's `$.errors.http-client-1.message`, `$.errors.http-client-1.status_code`, `$.errors.http-client-1.headers`, `$.errors.http-client-1.response`. The bare form `$.errors.message` does NOT resolve at runtime — Tray treats `errors` as a namespace and needs the failing step's name as the next segment. This is how Slack/email/log steps inside an error branch pull the actual error message; otherwise the alert renders blank.

### Loop (CRITICAL: must use `_loop` wrapper)
```json
{
  "name": "loop-1",
  "type": "loop",
  "content": {
    "_loop": [
      {"name": "script-1", "type": "normal", "content": {}},
      {"name": "boolean-condition-1", "type": "branch", "content": {"true": [], "false": []}}
    ]
  }
}
```

**Common mistake**: Using a direct array for loop content instead of `{"_loop": [...]}`.

## Callable Workflow Dependencies

Create in this order:
1. Callable workflows with no dependencies
2. Callable workflows that call other callables
3. Non-callable workflows (callers)

Store each `workflow_id` returned by `create_workflow` — needed for `call-workflow` steps.

## Step Output References (JsonPath)

Build jsonpaths from the **actual output schema of the referenced step**:

```
$.steps.<step-name>.<path-from-step-output-schema>
$.steps.loop-1.value.<field>              — current loop iteration item
$.steps.call-workflow-1.response.<field>  — callable workflow response
$.env.<variable>                          — environment variable
```

**Field names with spaces or special characters use bracket-quoted segments.** Plain dot syntax breaks on whitespace, hyphens, or non-ASCII — for those, switch to `['<key>']` notation. Examples: `$.steps.trigger.body['Primary User Email']`, `$.steps.script-1.result['user-id']`, `$['My Variable']`. Mix freely with dot syntax across a path. The same rule applies inside brace-interpolated strings: `"text": {"type": "string", "value": "Hi {$.steps.trigger.body['First Name']}"}`.

**Every step-output jsonpath starts with `$.steps.<step-name>.`** — including the trigger, which is always referenced as `$.steps.trigger.<field>`. Writing `$.trigger.body.type` (missing the `.steps.` prefix) builds and validates cleanly but resolves to undefined at runtime — every branch short-circuits silently. Top-level segments after `$.` are always one of: `steps`, `env`, `config`. Never write `$.trigger.…`, `$.body.…`, `$.<connector>.…`, etc. The build-time validator hard-rejects these (`jsonpath_invalid_root`).

**Trigger envelopes come from `outputSchema.properties`.** For the trigger step specifically, the segment immediately after `$.steps.trigger.` MUST be one of the keys declared in the trigger operation's `outputSchema.properties`. Confirmed envelopes: `webhook → body`, `form-trigger → result`, `callable-trigger → data`, `agent-tool-trigger → tool_input` / `static_data`. The validator rejects unknown envelope keys with "available keys: …" — do not flatten (`$.steps.trigger.<field>`) and do not invent new envelope keys. See `tray-connectors` for the canonical trigger table.

Different step types have different output shapes. Always check the referenced step's output structure rather than assuming a universal pattern. The `call_connector` tool response may wrap data differently than the workflow runtime — verify the runtime shape.

## Inspecting and Modifying Existing Workflows

Use `get_workflow` to fetch current live workflow state. Default `view: "structure"` returns the tree + step index without properties — cheap and usually enough. Use `view: "step"` with `step_names: [...]` to fetch full `{properties, metadata}` for specific steps before editing.

To modify a step's properties, use `update_workflow_steps` with the **complete** properties object on the step entry (it replaces, not merges). Pass multiple entries to fix several steps in one round-trip.
