# Core Connectors — Detailed Examples

Complete JSON examples for every core connector. Refer to SKILL.md for the quick reference.

## Triggers

### Manual Trigger
```json
"trigger": {
  "title": "Manual Trigger",
  "connector": {"name": "noop", "version": "1.1"},
  "operation": "trigger",
  "output_schema": {},
  "error_handling": {},
  "properties": {}
}
```

### Callable Trigger — Asynchronous
```json
"trigger": {
  "title": "Callable Trigger",
  "connector": {"name": "callable-trigger", "version": "2.0"},
  "operation": "trigger",
  "output_schema": {},
  "error_handling": {},
  "properties": {}
}
```

### Callable Trigger — Synchronous
Must pair with `callable-workflow-response` step.
```json
"trigger": {
  "title": "Callable Trigger and Respond",
  "connector": {"name": "callable-trigger", "version": "2.0"},
  "operation": "trigger_and_respond",
  "output_schema": {},
  "error_handling": {},
  "properties": {}
}
```

### Scheduled Trigger (example: daily at 9am UK)
All non-manual, non-callable, non-agent triggers need `public_url`.
```json
"trigger": {
  "title": "Daily Schedule",
  "connector": {"name": "scheduled", "version": "3.5"},
  "operation": "daily",
  "output_schema": {},
  "error_handling": {},
  "properties": {
    "public_url": {"type": "jsonpath", "value": "$.env.public_url"},
    "hour": {"type": "string", "value": "9"},
    "minute": {"type": "string", "value": "0"},
    "tz": {"type": "string", "value": "Europe/London"}
  }
}
```

## Boolean Condition

### boolean_condition — Compare values
```json
"boolean-condition-1": {
  "title": "Check ARR Above 100k",
  "connector": {"name": "boolean-condition", "version": "2.3"},
  "operation": "boolean_condition",
  "output_schema": {},
  "error_handling": {},
  "properties": {
    "conditions": {
      "type": "array",
      "value": [{
        "type": "object",
        "value": {
          "value1": {"type": "jsonpath", "value": "$.steps.loop-1.value.Current_ARR__c"},
          "comparison_type": {"type": "string", "value": ">"},
          "value2": {"type": "number", "value": 100000},
          "is_case_sensitive": {"type": "boolean", "value": false}
        }
      }]
    },
    "strictness": {"type": "string", "value": "All"}
  }
}
```

### property_exists — Check JsonPath exists
```json
"properties": {
  "property": {"type": "jsonpath", "value": "$.steps.trigger.body.user.email"}
}
```

## Loop

Operations: `loop_array`, `loop_object`, `loop_forever`

Output: `value` (current item), `count`, `index` (0-based), `is_first`, `is_last`

```json
"loop-1": {
  "title": "Loop Through Records",
  "connector": {"name": "loop", "version": "1.3"},
  "operation": "loop_array",
  "output_schema": {},
  "error_handling": {},
  "properties": {
    "array": {"type": "jsonpath", "value": "$.steps.salesforce-1.records"}
  }
}
```

## Delay

Properties: `time_unit` ("minutes" or "seconds"), `delay_value` (integer, max 600)

```json
"delay-1": {
  "title": "Wait 30 seconds",
  "connector": {"name": "delay", "version": "1.0"},
  "operation": "delay",
  "output_schema": {},
  "error_handling": {},
  "properties": {
    "time_unit": {"type": "string", "value": "seconds"},
    "delay_value": {"type": "integer", "value": 30}
  }
}
```

## HTTP Client

Operations: `get_request`, `post_request`, `put_request`, `patch_request`, `delete_request`

```json
"http-client-1": {
  "title": "GET API Call",
  "connector": {"name": "http-client", "version": "5.6"},
  "operation": "get_request",
  "output_schema": {},
  "error_handling": {},
  "properties": {
    "url": {"type": "string", "value": "https://api.example.com/v1/accounts"},
    "queries": {
      "type": "array",
      "value": [
        {"type": "object", "value": {
          "key": {"type": "string", "value": "limit"},
          "value": {"type": "string", "value": "100"}
        }}
      ]
    },
    "status_code": {
      "type": "object",
      "value": {"range": {"type": "object", "value": {
        "from": {"type": "integer", "value": 200},
        "to": {"type": "integer", "value": 299}
      }}}
    },
    "parse_response": {"type": "string", "value": "json"}
  }
}
```

### HTTP Client — POST with Body and Custom Headers

Use this pattern for APIs that need a JSON body and token-based auth via header (http-client has no built-in bearer auth option).

```json
"http-client-1": {
  "title": "POST to API",
  "connector": {"name": "http-client", "version": "5.6"},
  "operation": "post_request",
  "output_schema": {},
  "error_handling": {},
  "properties": {
    "url": {"type": "string", "value": "https://api.example.com/v1/records"},
    "headers": {
      "type": "array",
      "value": [
        {"type": "object", "value": {
          "key": {"type": "string", "value": "Authorization"},
          "value": {"type": "string", "value": "Bearer <token-or-jsonpath>"}
        }},
        {"type": "object", "value": {
          "key": {"type": "string", "value": "Content-Type"},
          "value": {"type": "string", "value": "application/json"}
        }}
      ]
    },
    "body": {
      "type": "object",
      "value": {
        "raw": {"type": "string", "value": "{\"name\": \"example\", \"status\": \"active\"}"}
      }
    },
    "status_code": {
      "type": "object",
      "value": {"range": {"type": "object", "value": {
        "from": {"type": "integer", "value": 200},
        "to": {"type": "integer", "value": 299}
      }}}
    },
    "parse_response": {"type": "string", "value": "json"}
  }
}
```

**Key points:**
- **Body**: Use `{raw: {"type": "string", "value": "<json-string>"}}` — the JSON payload is a string inside the `raw` wrapper, not a structured object
- **Auth-as-header**: http-client has no built-in bearer token option. Pass auth via `headers` array with `Authorization: Bearer ...`
- **Headers**: Array of `{key, value}` objects
- **parse_response**: String `"json"`, not a boolean

## Script (JavaScript)

Operations: `execute`, `execute_async`

Available libraries (no import needed): lodash (`_`), moment-timezone, crypto, Buffer, URL. `request` is also available but deprecated — prefer `http-client` steps for new work.

```json
"script-1": {
  "title": "Process Data",
  "connector": {"name": "script", "version": "3.4"},
  "operation": "execute",
  "output_schema": {},
  "error_handling": {},
  "properties": {
    "variables": {
      "type": "array",
      "value": [
        {"type": "object", "value": {
          "name": {"type": "string", "value": "account"},
          "value": {"type": "jsonpath", "value": "$.steps.loop-1.value"}
        }}
      ]
    },
    "script": {"type": "string", "value": "exports.step = function(input) {\n  const account = input.account;\n  return {\n    name: account.Name,\n    processed: true\n  };\n};"}
  }
}
```

## Python

```json
"python-script-1": {
  "title": "Calculate Full Name",
  "connector": {"name": "python-script", "version": "2.0"},
  "operation": "execute",
  "output_schema": {},
  "error_handling": {},
  "properties": {
    "variables": {
      "type": "array",
      "value": [
        {"type": "object", "value": {
          "name": {"type": "string", "value": "first"},
          "value": {"type": "jsonpath", "value": "$.steps.loop-1.value.FirstName"}
        }},
        {"type": "object", "value": {
          "name": {"type": "string", "value": "last"},
          "value": {"type": "jsonpath", "value": "$.steps.loop-1.value.LastName"}
        }}
      ]
    },
    "script": {"type": "string", "value": "def executeScript(input):\n    full = f\"{input.get('first','').strip()} {input.get('last','').strip()}\".strip()\n    return { 'fullName': full }"}
  }
}
```

## Call Workflow

Operations: `fire_and_forget`, `fire_and_wait_for_response`

```json
"call-workflow-1": {
  "title": "Call Processing Workflow",
  "connector": {"name": "call-workflow", "version": "2.0"},
  "operation": "fire_and_wait_for_response",
  "output_schema": {},
  "error_handling": {},
  "properties": {
    "workflow_id": {"type": "string", "value": "<target-workflow-id>"},
    "trigger_input": {
      "type": "object",
      "value": {
        "accountId": {"type": "jsonpath", "value": "$.steps.loop-1.value.Id"},
        "source": {"type": "string", "value": "caller-workflow"}
      }
    }
  }
}
```

## Callable Response

```json
"callable-workflow-response-1": {
  "title": "Return Result",
  "connector": {"name": "callable-workflow-response", "version": "1.0"},
  "operation": "response",
  "output_schema": {},
  "error_handling": {},
  "properties": {
    "response": {
      "type": "object",
      "value": {
        "status": {"type": "string", "value": "success"},
        "result": {"type": "jsonpath", "value": "$.steps.script-1.result"}
      }
    }
  }
}
```

## Send Email

No auth required. Sender is auto-set to `@traymail.io` — use `reply_to` for replies.

```json
"send-email-1": {
  "title": "Send Notification",
  "connector": {"name": "send-email", "version": "4.1"},
  "operation": "send_text_email",
  "output_schema": {},
  "error_handling": {},
  "properties": {
    "to": {
      "type": "array",
      "value": [
        {"type": "object", "value": {
          "email": {"type": "string", "value": "recipient@example.com"},
          "name": {"type": "string", "value": "Recipient Name"}
        }}
      ]
    },
    "subject": {"type": "string", "value": "Workflow Notification"},
    "content": {"type": "string", "value": "<h2>Report</h2><p>Workflow completed.</p>"}
  }
}
```

## Terminate

Operations: `terminate_run` (success), `fail_run` (failure with message)

```json
"terminate-1": {
  "title": "Fail with Message",
  "connector": {"name": "terminate", "version": "1.1"},
  "operation": "fail_run",
  "output_schema": {},
  "error_handling": {},
  "properties": {
    "message": {"type": "string", "value": "Validation failed: missing required data"}
  }
}
```
