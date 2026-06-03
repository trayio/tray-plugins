---
name: tray-gotchas
description: Load when debugging a Tray workflow error or working with known-weird surfaces — `raw_http_request` input format, `has_dynamic_output` rules, `trigger-reply`'s empty inputSchema, callable workflow sync/async pairing, jsonpath output-shape differences across step types, scheduled-trigger `day_of_week` explicit-keys rule, Salesforce custom (`*__c`) field picklist values, and stale `version_id` recovery.
disable-model-invocation: false
---

# Tray Workflow Gotchas

Consult this when debugging workflow errors or working with complex structures.

## Error Quick Reference

| Error | Likely Cause | Fix |
|---|---|---|
| "Exported structure validation failed" | Malformed step properties, loop missing `_loop` wrapper, `"next"` keys in steps_structure | Check property type-wrapping. Use `get_workflow` to inspect current state |
| "Connector version not found" | Wrong version | Use `list_connectors` to get exact version |
| "Connector unexpectedly failed" | Wrong `has_dynamic_output` | Check operation's `hasDynamicOutput` field |
| "Required key [body] not found" | `raw_http_request` missing body | Set body explicitly even for GET |
| Workflow hangs | Missing callable response in sync mode | Add `callable-workflow-response` step |
| trigger-reply "not configurable" | Empty inputSchema misleads researcher | Use known property structure — see Trigger Event Reply section below |
| Silent step failures | Property values not wrapped | Use `{"type": "string", "value": "..."}` format |
| Missing connector icon in UI | Step name includes operation (e.g., `slack-send-message-1`) | Use connector name only: `slack-1`, never include the operation name |
| Stale version_id error | Used an old version_id after a mutating operation | Use the version_id from the error response (or from the last successful operation) |
| `create_auth_collection` fails with "service parameter is required" | Passed `connector_name`/`connector_version` instead of service UUID | Call `list_service_environments` first to get the `service` UUID and `service_environment_id` |
| `list_service_environments` fails with "service_name parameter is required" | Passed `service_id` instead of `service_name` | Use `service_name` (string, e.g., `"google-gmail"`) and `service_version` (integer) |
| `update_workflow_steps` with `error_handling` returns 400 | Omitted `auth_id` on an authenticated step entry | Always include `auth_id` when updating steps that have authentication, even if just setting error_handling |
| String contains `{{` or template syntax | Used Handlebars/Mustache/moment.js — Tray has NO template engine | Use a script step to compose dynamic strings, then reference via jsonpath |
| Step output returns null/empty at runtime | JsonPath doesn't match the referenced step's actual output schema | Build jsonpaths from the referenced step's output schema — different step types have different shapes. Note: `call_connector` nests data under `output`, but the workflow runtime may not — always verify the runtime shape |
| Empty alert/email body inside an error branch | Used bare `$.errors.message` (Tray reads `errors` as a step name, needs the failed step's name next) or referenced the failed step's own output (empty on failure) | Use `$.errors.<failed-step-name>.<field>` — fields mirror the step's output schema in snake_case (e.g. `$.errors.http-client-1.message`, `$.errors.http-client-1.status_code`). The failed step's name is the second segment |
| Workflow "passed" but downstream jsonpaths return undefined at runtime | `data` passed to `trigger_workflow` wasn't shaped like `$.steps.trigger` (e.g. payload at root instead of nested under `body` for a webhook) | Shape `data` as the full trigger envelope: webhook → `{body:{...}, query:{...}, headers:{...}}`, callable-trigger → `{data:{...}}`, form-trigger → `{result:{...}}`, agent-tool-trigger → `{tool_input:{...}}`. Returned `fired_payload_shape` is the literal `$.steps.trigger` — verify jsonpaths against it |

## Runtime Debug Flow

When a workflow has run and you need to know why it succeeded or failed:

**Permission first.** `trigger_workflow` fires against the live Tray workspace — it executes every step, creates an execution log, consumes API quota, and runs every connector against whatever real system it's wired to. Always ask the user explicitly before firing, naming the side effects: "Ready to fire the workflow in your Tray workspace? This will create execution logs and consume API resources, and the Slack step will post to #demo-channel." Don't use vague language like "ready to test?" — be specific about what will happen.

1. Fire with `trigger_workflow(workflow_id, data)`. **Shape `data` exactly like `$.steps.trigger` should look at runtime** — i.e. the trigger's full envelope. Examples:
   - **webhook** → `data: {body: {<your fields>}, query: {}, headers: {}}` (method/path/body_url/etc. are optional; Tray fills them on a real fire)
   - **callable-trigger** → `data: {data: {<your fields>}}`
   - **form-trigger** → `data: {result: {<your fields>}}`
   - **agent-tool-trigger** → `data: {tool_input: {<your fields>}}` or `{static_data: {<your fields>}}`

   The tool validates `data` against the trigger operation's `outputSchema` (the runtime envelope mirrored by `$.steps.trigger` — not `inputSchema`, which is for trigger configuration like `add_raw_body`/`allow_cors`/`timeout`) and rejects missing required keys with the declared envelope. Unknown top-level keys surface as warnings — most commonly a sign that you put the payload at root instead of nesting it (e.g. `data: {timezone: 'X'}` for a webhook should have been `data: {body: {timezone: 'X'}}`). The returned `fired_payload_shape` is the literal `$.steps.trigger` — verify your jsonpaths against it.
2. Call `get_workflow_execution(workflow_id, execution_id)` (or `latest: true`). Read `failed_step_names` and scan `step_summary`. Always check `child_executions` — a parent can report success while a callable failed silently.
3. For each failed step, call `get_workflow_step_detail(workflow_id, execution_id, step_name)`. Step bodies are truncated to ~8 KB by default — bump `max_body_bytes` if the error message itself got cut.
4. Fix and re-fire. The same `trigger_workflow` returns a new `workflow_instance`.

If you need to find a recent execution to debug without firing, `list_workflow_executions(workflow_id)` returns a lightweight summary of the most recent runs.

The retry tiebreak in step-name resolution is verified against synthetic fixtures only — if a real retried step resolves to the wrong attempt, pass the explicit `step_id` from `step_summary` instead.

## Critical Structural Rules

### Loop Content Must Use `_loop` Wrapper
```json
// WRONG - causes validation failure
{"name": "loop-1", "type": "loop", "content": [steps...]}

// CORRECT
{"name": "loop-1", "type": "loop", "content": {"_loop": [steps...]}}
```

### Authentication on Steps
Use the `auth_id` parameter on each step entry passed to `add_workflow_steps` or `update_workflow_steps` — this sets `metadata.auth_uuid` on the step. Get the auth ID from `list_authentications`.

`get_workflow` (any view) surfaces step auths as `metadata.auth_uuid: "<uuid>"` — a bare string, ready to pass back as `auth_id` on `update_workflow_steps`. The `view: "auths"` shape returns the same UUIDs as a flat list per step, useful when you want to copy an existing auth onto a new step. (The write path wraps it as `{value: "<uuid>"}` internally; the read shape is bare.)

### Step Ordering Is Array Position, NOT `next` Pointers
```json
// WRONG - causes validation failure
[
  {"name": "trigger", "type": "normal", "content": {}, "next": "salesforce-1"},
  {"name": "salesforce-1", "type": "normal", "content": {}, "next": "script-1"}
]

// CORRECT - array order determines execution sequence
[
  {"name": "trigger", "type": "normal", "content": {}},
  {"name": "salesforce-1", "type": "normal", "content": {}},
  {"name": "script-1", "type": "normal", "content": {}}
]
```

### All properties must be type-wrapped

See `/tray-workflows:tray-connectors` for the canonical type table (`string`, `integer`, `number`, `boolean`, `array`, `object`, `jsonpath`) and the jsonpath examples.

### Object array items must also be wrapped

When the inputSchema declares `items.type: "object"` (or `items.oneOf: [...]`), each item in the array must be wrapped as `{"type":"object","value":{...}}`. A bare object inside an array round-trips through Tray's API but is unreadable by the workflow editor — the UI shows "field is required" / "value is required" on every nested property and the step does not fire at runtime. See `/tray-workflows:build-workflow#object-array-items-must-be-wrapped-too` for the full pattern. The MCP server validator rejects this on every mutation.

Common offenders to watch for:

- `salesforce.find_records.conditions` — items are `{field, operator, value}` objects.
- `salesforce.update_record.fields` — items are `{key, value}` pairs (note `key`, not `name`, despite the UI label).
- `agent-tool-trigger.tool_input` — input parameter array items.
- `google-sheets`/`airtable` column-mapping array fields.

### Trigger `public_url` requirement

See `/tray-workflows:tray-connectors` for the internal-trigger table and which triggers need `public_url`.

## `has_dynamic_output` Rules

- Check each operation's `hasDynamicOutput` from `list_connector_operations`
- DDL/private operations: usually `false`
- Data query operations: usually `true`
- **Exception**: `raw_http_request` always use `false` (fails with `true`)

### What `has_dynamic_output` Actually Does

When using `call_connector` for testing:
- **`has_dynamic_output: true`** returns `{output_schema: {...}}` — the shape of the response for Tray's UI schema inference, NOT the actual data
- **`has_dynamic_output: false`** returns `{output: {...}}` — the actual data from the connector

**For field discovery**, you often want the actual data, so use `false`. For example, when discovering Salesforce fields via `so_object_describe`, use `has_dynamic_output: false` to see the actual field list — or use a SOQL query instead.

## `raw_http_request` Input Format

- `url`: Must be object `{"endpoint": {"type": "string", "value": "/path"}}` or `{"full_url": ...}`
- `parse_response`: Must be string `"true"`, not boolean
- `body`: Required even for GET — use `{"none": {"type": "null", "value": null}}`
- `method`: String — GET, POST, PATCH, PUT, DELETE

## String Interpolation: brace-jsonpath only

Tray supports exactly ONE form of string interpolation: single-brace jsonpaths inside `{type:"string", value:"..."}` values. The runtime substitutes `{$.steps.<name>.<path>}` (and `{$.env.x}`, `{$.config.x}`) at execution time.

```json
// CORRECT — single-brace, with `$`
"text": {"type": "string", "value": "Found {$.steps.salesforce-1.totalSize} opportunities for {$.steps.trigger.body.account_name}."}

// CORRECT — single-value reference (preserves source type)
"text": {"type": "jsonpath", "value": "$.steps.script-1.result"}

// WRONG — handlebars, expressions, raw $.steps without braces
"value": "Found {{salesforce-1.total}} records"
"value": "{{moment(trigger.datetime).toISOString()}}"
"value": "Found $.steps.salesforce-1.total records"
```

**When to use which:**
- One value, no static text → `{type:"jsonpath", value:"$.steps.x.y"}` (preserves type)
- Static prefix/suffix + one or more values → `{type:"string", value:"...{$.steps.x.y}..."}` (interpolation)
- Transformation, iteration, or computation → `script` step

Reach for a `script` step ONLY when transformation/iteration/computation is needed. Don't add a script for plain concatenation — brace interpolation handles that. See `/tray-workflows:tray-connectors` for the full pattern.

## JsonPath Gotchas

- **Build jsonpaths from the referenced step's actual output schema**: Different step types have different output shapes. Always check the referenced step's output structure rather than assuming a universal pattern. Note: `call_connector` nests data under `output`, but the workflow runtime may structure it differently.
- **Loop values**: `$.steps.loop-1.value.fieldName` (not `$.steps.loop-1.fieldName`)
- **Call workflow responses**: `$.steps.call-workflow-1.response.fieldName` (not `$.steps.call-workflow-1.fieldName`)

## Callable Workflow Rules

- `trigger_and_respond` operation MUST have a `callable-workflow-response` step
- `trigger` operation must NOT have a response step
- Synchronous callable workflows that don't return a response will hang

### Trigger output shape — read it from `outputSchema.properties`

Trigger output paths are deterministic: the segment immediately after `$.steps.trigger.` MUST be one of the keys declared in the trigger operation's `outputSchema.properties`. No invention, no copying from a different trigger type. Confirmed envelopes:

| Trigger | First segment after `$.steps.trigger.` |
|---|---|
| `webhook` | `body` (plus `query`, `headers` siblings) |
| `form-trigger` | `result` |
| `callable-trigger` | `data` (input fields land under here regardless of sync vs async invocation) |
| `agent-tool-trigger` | `tool_input` or `static_data` |

The build-time validator (`validate_workflow` MCP tool) enforces this — paths like `$.steps.trigger.body.<x>` against a callable-trigger reject at build time with "available keys: data". If you see that error, the agent has used the wrong envelope key for the trigger type. The fix is always the same: read the trigger operation's `outputSchema.properties` and use one of those keys.

For unfamiliar trigger types (rare), call `list_connector_operations(connector_name: "<trigger>", operation_name: "<op>")` and look at the `outputSchema.properties` returned — never guess.

## Trigger Event Reply Steps (`trigger-reply`)

The `trigger-reply` connector has an empty `inputSchema`, so its property shape is **declared by the trigger operation's `replySchema`**, not by trigger-reply itself. Read it with `list_connector_operations(connector_name: "<trigger>", operation_name: "<op>")` and shape `properties` to match — never memorise.

The **shape rules split by trigger family** (verified 2026-05-21 via direct schema capture + round-trip persistence, `testing/local/RESEARCH-2026-05-21-replyschemas.md`):

| Trigger op                                  | Reply shape                                                 |
|---------------------------------------------|-------------------------------------------------------------|
| `webhook.webhook_with_response`             | flat `{status?, body?, headers?}`                            |
| `api-operation-trigger.request_response`    | `{response: {<discriminator>: {...}}}` — discriminated oneOf of 4 variants, dynamic inner |
| `api-operation-trigger.publish_subscribe`   | flat `{id?, eventType?, data}` per event                    |
| `agent-tool-trigger.request_response`       | `{response: {<discriminator>: {...}}}` — discriminated oneOf of 2 variants, static inner |
| `replySchema: null` (e.g. `webhook.fire_and_forget`, async `callable-trigger.trigger`, etc.) | no reply step allowed — `reply_step_unsupported_trigger` |

Tray's workflow API does NOT enforce reply shape at write-time — the validator (1.12.0+) is the only static safety net. Garbage replies persist silently and only surface at runtime as ambiguous errors. Build-time correctness via the validator is therefore load-bearing.

### Webhook — `webhook.webhook_with_response` (flat)

`replySchema` declares `{status, body, headers}` with `required: []` and `additionalProperties: false`. Persist flat under `properties`:

```jsonc
"properties": {
  "status":  {"type": "number", "value": 200},
  "body":    {"type": "object", "value": {"ok": {"type": "boolean", "value": true}}},
  "headers": {"type": "array", "value": [
    {"type": "object", "value": {
      "key":   {"type": "string", "value": "Content-Type"},
      "value": {"type": "string", "value": "application/json"}
    }}
  ]}
}
```

`required: []` means nothing is mandatory — return whichever subset of `{status, body, headers}` makes sense. `additionalProperties: false` means **`response`, `successBody`, `errorBody`, or any other extra top-level key is rejected**. Webhook is the trigger family where the no-envelope rule applies; the validator emits `reply_shape_mismatch` if you add one.

### API operation trigger — `request_response` (discriminated oneOf, dynamic inner)

`replySchema.properties.response` is a `oneOf` of **four variants**, each requiring a single discriminator key:

| Discriminator (in the reply) | Variant     | Inner schema source                                          |
|------------------------------|-------------|---------------------------------------------------------------|
| `successBody`                | Success     | `trigger.properties.response.success.value`                  |
| `internalErrorBody`          | Internal Error | `trigger.properties.response.internalError.value`         |
| `invalidInputBody`           | Invalid Input  | `trigger.properties.response.invalidInput.value`          |
| `forbiddenBody`              | Forbidden      | `trigger.properties.response.forbidden.value`             |

Persist as `{response: {<discriminator>: {...}}}`. Exactly **one** discriminator per reply (oneOf):

```jsonc
"properties": {
  "response": {"type": "object", "value": {
    "successBody": {"type": "object", "value": {
      "id":   {"type": "number", "value": 42},
      "name": {"type": "string", "value": "widget"}
    }}
  }}
}
```

The inner shape of each `<discriminator>` is **dynamic** — defined by the agent when configuring the APIM trigger. For example, if `trigger.properties.response.success.value = {type:"object", properties:{id:{type:"number"}, name:{type:"string"}}, required:["id","name"], additionalProperties:false}`, then the reply's `response.value.successBody.value` must contain `id` and `name`, both wrapped with their declared types, and nothing else.

**Critical: the DynamicSchema's `.value` is RAW JSON Schema, NOT type-wrapped.** This is a single exception to Tray's "wrap every property value in `{type, value}`" rule. The properties tagged `DynamicSchema` in the trigger's `inputSchema` (APIM's `request.body`, `request.headers`, `response.<variant>`; agent-tool-trigger's `body.tool_input`, `body.static_data`) take an outer `{type:"object", value:<schema>}` wrapper as normal, but the `<schema>` IS a raw JSON Schema fragment — not recursively wrapped. The agent must write:

```jsonc
"body": {
  "type": "object",
  "value": {
    "type": "object",
    "properties": {
      "timezone": {"type": "string", "title": "Timezone"}
    },
    "required": ["timezone"],
    "additionalProperties": false
  }
}
```

and NOT:

```jsonc
"body": {
  "type": "object",
  "value": {
    "type":       {"type": "string",  "value": "object"},               // ❌
    "properties": {"type": "object",  "value": {"timezone": {...}}},    // ❌
    "required":   {"type": "array",   "value": [{"type":"string","value":"timezone"}]},  // ❌
    "additionalProperties": {"type": "boolean", "value": false}         // ❌
  }
}
```

Tray's workflow API accepts the wrong form silently (it stores anything), but the **Tray UI rejects it as "Unexpected json schema format"** and the schema is unusable for both UI rendering and DynamicSchema-based runtime validation. The reply-shape validator (1.12.0+) catches this as `reply_shape_mismatch` pointing at the trigger's broken DynamicSchema path.

Inside the raw JSON Schema fragment, each property's `title` populates Tray's UI "Name" field and `description` populates "Description" — include both whenever there's user-facing copy, otherwise the schema renders with a blank Name next to the key.

**Build-time rules the validator enforces:**

- `response` is required (envelope check).
- Exactly one discriminator key under `response.value`. Two or more → `reply_shape_mismatch` (multiple discriminators). Zero → `reply_shape_mismatch` (missing discriminator) with the allowed-keys hint.
- Unknown keys under `response.value` (not one of the four discriminators) → rejected.
- Inner walk: each discriminator's `.value` is validated against the trigger's matching `response.<variantInputKey>.value` JSON Schema (variantInputKey = discriminator without the `Body` suffix). Unknown inner keys, missing required inner keys, and leaf type mismatches all reject.

**Don't leave `trigger.properties.response.success.value` empty** when you build the trigger — the dynamic inner walk silently accepts everything if the schema is empty, leaving the contract ambient. Even a minimal `{type:"object", properties:{}, additionalProperties:false}` is better than nothing.

### API operation trigger — `publish_subscribe` (flat per-event)

APIM's "Receive Events" operation — SSE-style streaming where each `trigger-reply` emits one event back to the caller. `replySchema` declares `{id?, eventType?, data}` per event with `required: ["data"]` and `additionalProperties: false`:

```jsonc
"properties": {
  "id":        {"type": "string", "value": "evt-1"},
  "eventType": {"type": "string", "value": "progress"},
  "data":      {"type": "string", "value": "{\"percent\":50}"}
}
```

`data` is required and typed `string` — serialize per-event payloads to strings (or use a `jsonpath` wrapper pointing at a script that produces a string). No `response` envelope, no discriminator — flat, like webhook.

### Agent tool trigger — `request_response` (discriminated oneOf, STATIC inner)

`replySchema.properties.response` is a `oneOf` of **two variants** — materially different from APIM:

| Discriminator | Variant | Inner schema                                              |
|---------------|---------|------------------------------------------------------------|
| `successBody` | Success | `type: ["object", "string"]`, `additionalProperties: true` — open: any object OR any string |
| `errorBody`   | Error   | `{type: "object", properties: {message: {type: "string"}}, required: ["message"], additionalProperties: false}` |

`successBody` is open — the agent contract is whatever the tool returns:

```jsonc
"properties": {
  "response": {"type": "object", "value": {
    "successBody": {"type": "object", "value": {
      "customer": {"type": "object", "value": {
        "id":    {"type": "string", "value": "u-1"},
        "name":  {"type": "string", "value": "Ada"},
        "email": {"type": "string", "value": "ada@example.com"}
      }}
    }}
  }}
}
```

Or as a string:

```jsonc
"properties": {
  "response": {"type": "object", "value": {
    "successBody": {"type": "string", "value": "lookup complete"}
  }}
}
```

`errorBody` is fixed:

```jsonc
"properties": {
  "response": {"type": "object", "value": {
    "errorBody": {"type": "object", "value": {
      "message": {"type": "string", "value": "customer not found"}
    }}
  }}
}
```

**Build-time rules the validator enforces:**

- `response` required.
- Exactly one of `successBody` / `errorBody` under `response.value`. APIM's `internalErrorBody` / `invalidInputBody` / `forbiddenBody` do NOT exist here — they're rejected as unknown discriminators.
- `errorBody` must be `{message: string}` exactly — missing `message`, extra keys, or wrong leaf type all reject.
- `successBody` accepts any object (with any keys) or any string. No inner walk.

### Key points

- Always read `replySchema` from `list_connector_operations` for the trigger op. Different trigger families, different shapes.
- "Flat with no envelope" is only true for webhook + publish_subscribe. APIM and agent-tool-trigger require the `response: {<discriminator>: {...}}` envelope.
- APIM has 4 discriminators with dynamic inner. agent-tool-trigger has 2 discriminators with static inner. Don't extrapolate one to the other.
- Webhook triggers split three ways: `fire_and_forget` returns 200 immediately (no reply step); `webhook` (validate-and-respond) auto-200s (no reply step); only `webhook_with_response` pairs with `trigger-reply`.

## Callable Workflow Response Step (`callable-workflow-response`)

Pairs with `callable-trigger.trigger_and_respond` (the sync flavour) to return a value to the caller workflow. Unlike `trigger-reply`, the response shape is declared by the **response connector's** `inputSchema`, not by the trigger's `replySchema` (the callable trigger's `replySchema` is `null`).

`callable-workflow-response.response.inputSchema` declares one top-level key:

```jsonc
{
  "type": "object",
  "properties": {
    "response": {
      "type": "object",
      "description": "The response data to send to the caller workflow",
      "additionalProperties": false
    }
  },
  "additionalProperties": false
}
```

So the persisted property shape is, again, flat — a single `response` key wrapped as an object:

```jsonc
"properties": {
  "response": {"type": "object", "value": {"ok": {"type": "boolean", "value": true}}}
}
```

The per-step validator covers this on every `add_workflow_steps` / `update_workflow_steps` call — the cross-step `validate_workflow` audit does not need extra logic here.

## Disabling Callable Workflows

Callable workflows can't be turned off directly. Tray prevents `update_workflow_metadata({enabled: false})` on a callable to avoid breaking other workflows that depend on it. To stop a callable from doing real work, add a `terminate` step at the very top of its structure (before the trigger's actual children) — every invocation then short-circuits without running anything else.

## Modifying Existing Workflows

Use `get_workflow` to fetch the current live state, then use `update_workflow_steps` to change step properties, `remove_workflow_step` / `add_workflow_steps` to add/remove steps, or `update_workflow_structure` to reorder steps or move them into/out of branch paths.

`update_workflow_steps` replaces the **entire** properties object for each targeted step — it is not a merge/patch. Always provide the complete properties per entry.

## Scheduled Trigger

For the full `daily`-operation property shape (including the 7-key `day_of_week` object), see `/tray-workflows:tray-patterns`. The MCP validator (`validate_workflow` / per-step pre-flight) rejects scheduled-`daily` triggers missing any of `hour` / `minute` / `tz` / `day_of_week`, or `day_of_week` missing any of the 7 day keys.

**Trigger output is zero-indexed for `month` and `day_of_week`** — January is `0`, Monday is `0`. Conditions that compare these against a 1-indexed literal will silently route wrong at runtime. Use the Date Time Helper's `Get Date Property` operation rather than parsing the trigger payload directly.

## Field Value Discovery

- UI display names often differ from API values (e.g., "Active" vs "4 - Current Customer (Active)")
- Custom fields have `__c` suffix
- Always use DDL operations or ask the user — never guess

### Salesforce custom fields (`*__c`)

Whenever the user references a Salesforce field whose name ends in `__c`, the field is **custom to that org** and its valid values are not knowable from the connector's static schema. This applies to picklist values, lookup IDs, and free-text constraints alike.

**Rule:** before setting any value on a `*__c` field, you MUST do one of these:

1. **DDL the picklist.** If the field is a picklist, call the Salesforce DDL operation that lists its valid values (e.g., `lookup_picklist_values_ddl` or the equivalent for the version) and use a returned value verbatim.
2. **Ask the user.** "Just to confirm: is `<value>` a valid value for `Priority__c` in your org? It's a custom field so I can't verify on my own."
3. **Surface the assumption explicitly.** If the user has already supplied the value (e.g. "set Priority__c to High"), still call it out in the build summary: "Built with `Priority__c = 'High'` — this assumes 'High' is one of your existing picklist values for that field. If it isn't, the update step will fail at runtime."

Building silently with an unverified `*__c` value — even one the user gave you — is wrong. The cost of a one-line confirmation is tiny; the cost of a workflow that fails on every record at runtime is large.
