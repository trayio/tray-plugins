---
name: research-connector
description: Systematic process for researching unknown Tray connectors via DDL discovery. Use when you need to discover a connector's version, operations, auth, required fields, and DDL-resolved values before building a workflow step.
---

# Research Connector

This skill is the **fallback path** for direct connector research from the parent agent — useful for trivial lookups (single small operation, no DDL fields, no polymorphism) or for ad-hoc exploration. For real workflow builds, prefer delegating to the Researcher subagent (see `build-workflow#step-2-research-connectors`); the subagent absorbs verbose schemas and returns a step-level prescriptive result, keeping the parent's context lean.

If you do follow this process directly, the schemas you see from `list_connector_operations` are **compacted by default** — `advanced` fields stripped, polymorphic branches replaced with `_collapsed` markers carrying `expand: ['<propName>']` hints. Echo a hint verbatim into the next call's `expand` array when you actually need a collapsed branch; otherwise leave it collapsed. Pass `include_advanced: true` only when an advanced field is genuinely required, and `include_output_schema: false` for terminal steps where nothing downstream consumes the result.

Follow this process exactly. Do NOT skip steps or guess field values.

## Stop Conditions

Immediately stop and ask the user if:
- DDL operations don't reveal field values/options
- Field names contain ambiguous terms (status, active, priority, etc.)
- Custom fields are referenced (__c suffix fields)
- Authentication fails or is missing
- Sample queries return errors

## Step 1: Discover Connector

Call `list_connectors` with the connector name. Record exact `name`, `version`, and `service` details. When multiple versions exist, use the latest.

## Step 2: Check Authentication

Use the connector's **service name** (from Step 1's `connector.service.name` — e.g. `google-sheets`, `slack`, `salesforce`) to filter authentications. **Always prefer `service_name` over keyword `search`** — auth display names are user-supplied (e.g. an auth named "Sheets" backs `service.name === "google-sheets"`) and keyword search by name routinely misses the right auth.

```
list_authentications(service_name: "<connector.service.name>")
```

If 0 matches, fall back to a no-filter call to see every auth in the workspace. Don't reach for keyword `search` — it's the same fragile path. Present existing auths to the user — reuse if suitable. Only create new auth if none exist or user requests it.

**To create new auth** (parameter names are unintuitive — follow exactly):
1. Call `list_service_environments` with `service_name` (string, e.g., `"google-gmail"`) and `service_version` (integer from `list_connectors` → `service.version`) — NOT `service_id`
2. Call `create_auth_collection` with `service` (UUID from step 1's response), `service_environment_id` (from step 1's `id` field), and `scopes` — NOT `connector_name`/`connector_version`
3. Send the auth URL to the user, then call `check_auth_completion`

| Parameter | Source | Common mistake |
|---|---|---|
| `list_service_environments.service_name` | `list_connectors` → `service.name` | Using `service_id` instead |
| `list_service_environments.service_version` | `list_connectors` → `service.version` (integer) | Passing as string |
| `create_auth_collection.service` | `list_service_environments` → `service` (UUID) | Using `connector_name` |
| `create_auth_collection.service_environment_id` | `list_service_environments` → `id` | Omitting entirely |

## Step 3: List Operations

Call `list_connector_operations` for the discovered connector. The response is compacted by default — read carefully:

- `inputSchema.required` — mandatory fields
- `inputSchema.properties` — field types and constraints
- `hasDynamicOutput` — needed for `call_connector` calls
- `lookup` objects on individual fields — these reveal private DDL operations
- `_collapsed` markers — a property whose body is replaced by a string of the form `"oneOf with N variants (...); pass expand: ['<name>'] to retrieve in full"` (or `"array items: object with N properties; …"`). If the workflow needs that branch, re-call with the named property in the `expand` array. If it doesn't, leave it collapsed.

## Step 4: Execute DDL Operations

For fields with `lookup` objects, extract the private DDL operation name from `lookup.body.message` and call it via `call_connector`.

Key points:
- Private DDL operations are NOT listed in `list_connector_operations` but ARE callable
- Determine required inputs from `lookup.body.step_settings`
- Private DDL ops usually use `has_dynamic_output: false`
- Common naming patterns: `private_list_*`, `*_ddl`, `list_*_options`, `get_*_values`
- Some DDL operations chain: get objects → get fields for object → get values for field

## Step 5: Discover Dynamic Output Schemas (conditional)

**Do NOT blindly test every operation with `call_connector`.** The tool is gated server-side to protect production auths. Only call it in these cases:

- **Operation is a DDL** (`operationType === 'private'` or referenced as `lookup.body.message` from another op): call it to discover field values (handled in Step 4).
- **Operation has `hasDynamicOutput: true`**: call it with `has_dynamic_output: true` so the server returns `returnOutputSchema: true`. This reveals the real output shape, which you need to build correct jsonpaths.
- **Otherwise**: DO NOT call. Present the `inputSchema` (and `outputSchema` if static) to the user in the summary and move on. Regular mutating operations (create_record, send_message, etc.) can have real side effects with production auths and are billable per call — the user will test them in their own way.

If a user explicitly asks you to call a non-DDL, non-dynamic-output operation (e.g., "try this raw_http_request"), pass `allow_unsafe: true`. Never set `allow_unsafe` silently or in a loop.

## Step 6: Ask User for Unknown Values

For any field values not discoverable via DDL, ask the user explicitly. Examples:
- "What status value indicates 'active' records in your system?"
- "Which custom fields should I include?"
- "What are the valid priority levels in your instance?"

## Pre-Build Checklist

Before proceeding to workflow creation, verify:
- [ ] Exact connector version discovered (not guessed)
- [ ] Authentication confirmed working
- [ ] All required fields identified from `inputSchema.required`
- [ ] Lookup field values discovered via DDL operations
- [ ] User provided values for anything DDL couldn't reveal
- [ ] Output schema confirmed (via `hasDynamicOutput` discovery or the static `outputSchema`) — do NOT test-fire mutating operations
- [ ] Correct `has_dynamic_output` value noted for each operation
