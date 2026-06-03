---
name: tray-connectors
description: Load when adding workflow steps, picking a trigger, or wrapping step properties — covers core connector names and versions (including the non-obvious `noop`/`scheduled`/`callable-trigger` trigger names), the type-wrapper format, connector-name-only step naming, and common jsonpath shapes (loop `value`, callable `response`).
disable-model-invocation: false
---

# Tray Core Connectors Reference

Consult this when building workflow steps with standard connectors.

For connectors not listed here, use `/tray-workflows:research-connector` to discover them via MCP tools.

## Quick Version Table

**Always verify versions with `list_connectors` — versions may vary by workspace.**

| Connector | Name | Version | Use Case |
|---|---|---|---|
| Manual Trigger | `noop` | `1.1` | Start manually |
| Scheduled Trigger | `scheduled` | `3.5` | Time-based triggers |
| Callable Trigger | `callable-trigger` | `2.0` | Called by other workflows |
| Boolean Condition | `boolean-condition` | `2.3` | Two-way branch |
| Loop | `loop` | `1.3` | Iterate arrays/objects |
| Break Loop | `break-loop` | `1.1` | Exit/skip iterations |
| Delay | `delay` | `1.0` | Pause execution |
| HTTP Client | `http-client` | `5.6` | REST/HTTP APIs |
| Script (JS) | `script` | `3.3` / `3.4` | Custom JavaScript |
| Python | `python-script` | `2.0` | Python transformations |
| Call Workflow | `call-workflow` | `2.0` | Invoke other workflows |
| Callable Response | `callable-workflow-response` | `1.0` | Return data to caller |
| Send Email | `send-email` | `4.1` | Email notifications |
| Terminate | `terminate` | `1.1` | Stop execution |

For detailed examples of each connector, see [core-connectors.md](core-connectors.md).

## Triggers

Every workflow starts with exactly one trigger step, always named `"trigger"`.

### Picking the trigger from the user's prompt

The user's wording maps to a specific trigger connector. Pick deliberately — `webhook` is the wrong default for form/submit/manual prompts, but it's the right answer for "Stripe sends events to us" / "incoming HTTP from system X".

| User says… | Use |
|---|---|
| "form", "submit a form", "form fields", "form submission", "fill in" | `form-trigger` |
| "manual", "press a button", "run on-demand", "trigger it myself", "kick it off" | `noop` (operation `trigger`) |
| "scheduled", "every day at", "weekly", "cron", "at 9am" | `scheduled` |
| "called by another workflow", "callable", "invoked from", "shared workflow" | `callable-trigger` |
| "agent tool", "the agent calls", "tool the agent uses" | `agent-tool-trigger` |
| "API endpoint", "expose an API", "request/response API" | `api-operation-trigger` |
| "Slack message arrives", "new Slack event" | `slack-trigger` |
| "external system POSTs to us", "incoming webhook from <SaaS>", "<service> sends an HTTP payload" | `webhook` |

`form-trigger` and `webhook` are not interchangeable — the rubric for form-shaped scenarios is explicit. A form trigger has typed, schema-driven fields the workflow can branch on; a webhook trigger's body is opaque JSON the workflow has to defensively unpack. Use `form-trigger` whenever the user mentions a form, even if the surrounding workflow looks "webhook-shaped" (POST-in, JSON-out).

**Trigger output paths are always declared in the trigger operation's `outputSchema.properties`.** When you research the trigger via `list_connector_operations`, look at `outputSchema.properties` — the keys at that level are the only valid first segment after the step name. Examples observed in current connectors: `webhook → body`, `form-trigger → result`, `callable-trigger → data`, `agent-tool-trigger → tool_input` / `static_data`. The build-time validator enforces these envelope keys; paths like `$.steps.trigger.<not-an-envelope-key>.foo` reject with "available keys: …". Do not invent paths or copy a shape from a different trigger type.

If `list_connectors(connector_names: [...])` returns "found N/M" with form-trigger / callable-trigger / agent-tool-trigger missing, those are trigger-only connectors — the REST endpoint excludes them. Re-call with `is_trigger: true` to see them.

### Internal trigger connectors (always verify versions with `list_connectors`)

These are the built-in trigger connectors. **Names are NOT guessable** — e.g. "Manual" is `noop`, not `manual`; "Scheduled" is `scheduled`, not `schedule`:

| UI label | `connector_name` | Version | Typical operation | Notes |
|---|---|---|---|---|
| Manual | `noop` | `1.1` | `trigger` | No public_url; no auth; empty properties |
| Scheduled | `scheduled` | `3.5` | `daily`, `every_n_minutes`, etc. | Requires `public_url`. `daily` also requires `hour`, `minute`, `tz`, and all 7 `day_of_week` keys |
| Webhook | `webhook` | `2.3` | `fire_and_forget` (no response), `webhook` (validate-and-respond — runs sync validation then auto-200), `webhook_with_response` (await workflow and return its response body) | No `public_url` needed (uses trigger URL instead). For `webhook_with_response`, pair with a `trigger-reply` step. |
| Callable | `callable-trigger` | `2.0` | `trigger` (async), `trigger_and_respond` (sync, pair with `callable-workflow-response`) | No `public_url`. Output also exposes `#calling_workflow` / `#calling_execution` / `#calling_execution_log_url` siblings of `data` for caller diagnostics. |
| Form | `form-trigger` | (verify) | — | No `public_url` |
| Email | `email-trigger` | (verify) | — | Requires `public_url` |
| Alert | `alerting-trigger` | `1.1` | — | Used by Tray Alerting to fire workflows on platform alerts. |
| API Operation | `api-operation-trigger` | `1.0` | `fire_and_forget`, `publish_subscribe`, `request_response`, `validate` | No `public_url` |
| Agent tool | `agent-tool-trigger` | `1.0` | — | No `public_url`; description of the containing workflow IS the tool description the agent sees |
| Agent (chat) | `agent-trigger` | (verify) | — | No `public_url` |
| Slack trigger | `slack-trigger` | (verify) | — | No `public_url` |

### public_url requirement

**Source of truth: the operation's `inputSchema.required`.** `public_url` is required when the trigger's operation schema declares it required — don't guess from the connector name.

Confirmed from schemas:

| Requires `public_url` | Does NOT require `public_url` |
|---|---|
| `scheduled` (all operations) | `noop`, `callable-trigger` |
| `email-trigger` | `webhook` (URL auto-generated by Tray, schema has no `public_url` property) |
| | `slack-trigger`, `form-trigger`, `api-operation-trigger` |
| | `agent-trigger`, `agent-tool-trigger` |

When `public_url` is required, set it as:

```json
"public_url": {"type": "jsonpath", "value": "$.env.public_url"}
```

For unfamiliar triggers, call `list_connector_operations` with the `operation_name` — check whether `public_url` appears in `inputSchema.required`. The MCP server's per-operation schema validation (`validateStepProperties`) enforces this automatically; you don't need to track the list yourself.

## Constructing Step Properties

**CRITICAL: Field names come from inputSchema, not from guessing.**

1. Use `list_connector_operations` to get the operation's `inputSchema`
2. Use the exact field names from `inputSchema.properties`
3. Only include fields that exist in the schema
4. Wrap values according to their type

**Never make up field names.** If a field isn't in the schema, it doesn't exist.

## Property Type Mapping

All property values must be wrapped:

| Type | Format |
|---|---|
| String | `{"type": "string", "value": "text"}` |
| Integer | `{"type": "integer", "value": 123}` |
| Number | `{"type": "number", "value": 3.14}` |
| Boolean | `{"type": "boolean", "value": true}` |
| Array | `{"type": "array", "value": [...]}` |
| Object | `{"type": "object", "value": {...}}` |
| JsonPath | `{"type": "jsonpath", "value": "$.steps.step-1.field"}` |

## Referencing Step Output

A property value is one of three things:
- A **static value** (string, integer, etc.) — hardcoded at build time
- A **jsonpath reference** — a single value from another step's output, supplied via the jsonpath wrapper
- A **string with brace-interpolated jsonpaths** — Tray substitutes `{$.steps.x.y}` segments inside string values at runtime

### Correct: Reference a single value from another step
```json
"channel": {"type": "jsonpath", "value": "$.steps.script-1.result.channel_id"}
"text": {"type": "jsonpath", "value": "$.steps.script-1.result.message"}
```

### Correct: Brace interpolation inside a string (single-brace, with `$`)

The runtime resolves `{$.steps.<name>.<path>}` substrings inside any `{type:"string", value:"..."}` property. Use this for "static prefix + one or more dynamic values" — it skips the cost of an extra `script` step purely for concatenation.

```json
"text": {"type": "string", "value": "Found {$.steps.salesforce-1.totalSize} opportunities for {$.steps.trigger.body.account_name}."}
"instructions": {"type": "string", "value": "Summarise the following customer note:\n\n{$.steps.slack-2.response.body}"}
```

Rules:
- Use single braces, not double — `{$.steps.x.y}`, NOT `{{$.steps.x.y}}`.
- The path must start with `$.steps.`, `$.env.`, or `$.config.` — same root rules as a regular jsonpath wrapper.
- Multiple braces in one string are fine.
- For a *single* dynamic value (no static prefix/suffix), prefer `{type:"jsonpath", value:"$.steps.x.y"}` — it preserves the source type (object/array/number) instead of stringifying.

### WRONG: Template syntax (does NOT work in Tray)
```json
"text": {"type": "string", "value": "Found {{salesforce-1.total}} records"}        // double-brace handlebars
"text": {"type": "string", "value": "Found $.steps.salesforce-1.total records"}     // raw $.steps with no braces
"value": {"type": "string", "value": "{{moment(trigger.datetime).toISOString()}}"}  // expressions
"text": {"type": "string", "value": "{{#each records}}{{Name}}{{/each}}"}           // handlebars iteration
```
These are treated as literal strings.

### When to use a script step instead

Brace interpolation handles "static + one or more values" cleanly. Reach for a `script` step when:
- You need to **transform** a value (date format, case change, lookup, conditional logic)
- You're **iterating** over an array to build a string (e.g. join records into a Slack message)
- You need to **compute** something from multiple sources (e.g. sum a list of totals)

When a step needs a formatted string built from multiple sources by transformation or iteration, use a **script step** to compose it:

1. Add a `script` step before the step that needs the composed value
2. Reference input data via jsonpath in the script's `variables`:
   ```json
   "variables": {"type": "array", "value": [
     {"type": "object", "value": {
       "name": {"type": "string", "value": "records"},
       "value": {"type": "jsonpath", "value": "$.steps.salesforce-1.records"}
     }},
     {"type": "object", "value": {
       "name": {"type": "string", "value": "total"},
       "value": {"type": "jsonpath", "value": "$.steps.salesforce-1.total"}
     }}
   ]}
   ```
3. Build the string in JavaScript:
   ```json
   "code": {"type": "string", "value": "exports.step = function(input) {\n  let msg = `Found ${input.total} opportunities:\\n`;\n  input.records.forEach(r => {\n    msg += `• ${r.Name} (${r.StageName}) - $${r.Amount}\\n`;\n  });\n  return msg;\n}"}
   ```
4. Reference the script output in the downstream step:
   ```json
   "text": {"type": "jsonpath", "value": "$.steps.script-1.result"}
   ```

### Common jsonpath patterns
| What you need | JsonPath |
|---|---|
| Step output field | `$.steps.step-name.<field-from-step-output>` |
| Loop current value | `$.steps.loop-1.value.<field>` |
| Callable workflow response | `$.steps.call-workflow-1.response.<field>` |
| Environment variable | `$.env.variable_name` |

**Build jsonpaths from the actual output schema of the referenced step.** Different step types have different output shapes (e.g., script steps wrap return values in `result`, connector steps may expose fields directly). Always check the step's output structure rather than assuming a universal pattern.

## Step Naming

Format: `{connector-name}-{number}` — use the connector name ONLY, never the operation name.
- `slack-1` (correct) — NOT `slack-send-message-1`
- `salesforce-1` (correct) — NOT `salesforce-find-records-1`
- `http-client-1` (correct) — NOT `http-client-get-request-1`

Using the wrong step name causes the connector icon to be missing in the Tray UI.

Exception: the trigger step is always named `"trigger"`.

## Authentication on Steps

Pass `auth_id` on each step entry when adding or updating:
```
add_workflow_steps(..., steps=[{..., auth_id: "<uuid from list_authentications>"}])
update_workflow_steps(..., steps=[{..., auth_id: "<uuid from list_authentications>"}])
```
Get the auth UUID from `list_authentications` (the `id` field). Steps without auth needs (script, loop, boolean-condition, send-email) should omit `auth_id`.

## Helper Connectors

Helper connectors are no-auth Tray-built primitives for common transformations. The operation names are non-obvious — most prompts say "decode base64" / "lowercase" / "join", and the connector calls those `base64_encoderdecoder` / `lower_case` / `concatenate`. Look up the schema before guessing — `list_connector_operations` is the source of truth for every operation name and field shape.

### `text-helpers` (note the **plural**) — string transformations

Connector name: `text-helpers` (plural — `text-helper` does NOT exist). Latest observed version: `3.0`. No auth.

| User says… | Operation | Notes |
|---|---|---|
| "encode/decode base64" | `base64_encoderdecoder` | Single op covers both directions; set `operation: {type:"string", value:"encode"}` or `"decode"` inside properties |
| "lowercase" / "to lower case" | `lower_case` | Not `to_lower_case` |
| "uppercase" | `upper_case` | |
| "join", "concatenate", "combine strings" | `concatenate` | Not `join`. Takes `values` array + optional `separator` |
| "split a string" | `split` | |
| "replace text" | `replace` | |
| "convert to number / boolean / string" | `change_type` | The target type field is named `type2` (not `type` — `type` is reserved by the wrapper). Easy to miss without reading the schema |
| "trim whitespace" | `trim` | |

Output: scalar transformations land on `$.steps.text-helpers-N.result`. Verify on the operation's `outputSchema`.

### `data-mapper` — value/key remapping

Connector name: `data-mapper`. Latest observed version: `3.5`. No auth. Three operations, **three different output shapes** — pick the one that matches your intent and reference its output correctly:

| Operation | What it does | Output jsonpath shape |
|---|---|---|
| `map_keys` | Rename keys on an object using a `from → to` mapping | `$.steps.data-mapper-N.<new-key-name>` — mapped keys are emitted at the root, not nested under `result` |
| `map_one_value` | Translate a single scalar via a `from → to` mapping (with `default`) | `$.steps.data-mapper-N.result` |
| `map_values` | Walk an input array/object and translate values using `from → to` | `$.steps.data-mapper-N.output[<index>].<field>` (mapped values land under `output`, not `result`) |

The Researcher subagent occasionally reports `map_values` output as living at `$.steps.<name>` directly — that's wrong; it's under `.output`. The validator catches it, but you can pre-empt the round-trip by referencing `.output` from the start.

All three operations share the `mappings` array property — items are `{from, to}` objects (`change_type` uses `type2`, `data-mapper` uses `from`/`to`). Each mapping item must be `{type:"object", value:{from:{...}, to:{...}}}` per the object-array-items rule.
