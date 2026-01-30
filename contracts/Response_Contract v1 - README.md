# Response Contract v1

## Purpose

The **Response Contract** defines the **only object shape exposed outside the workflow**.

It is the contract consumed by:

* APIs
* frontends
* notification systems
* external automations
* logging/monitoring sinks (non-internal)

It is produced **after** decision-making and orchestration are complete.

---

## Design Principles

* **Single responsibility**: represent the outcome, not the reasoning
* **Stable**: safe for long-lived consumers
* **Action-oriented**: easy to interpret without context
* **Minimal**: no internal diagnostics
* **Serializable**: JSON-safe, no expressions or runtime artifacts

---

## Response Contract v1 — Shape

```json
{
  "response_id": "string",
  "timestamp": "ISO-8601 string",
  "channel": "email | chat | webhook",

  "final_action": {
    "type": "PROVIDE_INFO | REQUEST_MORE_INFO | ESCALATE_HUMAN | REJECT_REQUEST | SCHEDULE_REQUEST",

    "payload": {
      "message": "string",
      "missing_information": ["string"],
      "level": "LOW | MEDIUM | HIGH | CRITICAL",
      "reason": "string",
      "signals": ["string"]
    }
  },

  "meta": {
    "version": "v1",
    "trace_id": "string"
  }
}
```

---

## Field Semantics (Precise)

### `response_id`

* Unique identifier for this response instance
* Generated at workflow execution time
* Used for tracing and reconciliation

---

### `timestamp`

* Time the response was generated
* ISO-8601 format
* Required for auditability

---

### `channel`

* The delivery channel this response targets
* **Not** the input channel
* Used by consumers to select formatting or transport

---

### `final_action`

The **only required actionable block**.

#### `type`

The canonical action the consumer must perform.

This is a **command**, not a classification.

Consumers must never infer behavior from anything else.

---

#### `payload`

The payload is **polymorphic**.
Only fields relevant to the `type` are populated.

| Action Type       | Required Payload Fields         |
| ----------------- | ------------------------------- |
| PROVIDE_INFO      | `message`                       |
| REQUEST_MORE_INFO | `missing_information`, `reason` |
| ESCALATE_HUMAN    | `level`, `reason`, `signals`    |
| REJECT_REQUEST    | `reason`                        |
| SCHEDULE_REQUEST  | `message`                       |

Unused fields **must be omitted**, not null.

---

### `meta`

Non-functional metadata.

#### `version`

* Response contract version
* Enables forward compatibility

#### `trace_id`

* Correlates this response to:

  * decision output
  * workflow execution
  * logs

---

## Explicit Guarantees

This contract guarantees:

* One and only one `final_action`
* No internal decision logic leakage
* No ambiguity for consumers
* Backward compatibility within `v1`
* Deterministic consumption

---

## Explicit Non-Goals

This contract does **not**:

* Explain *why* a decision was made
* Include confidence scores
* Include risk models
* Include constraints or policies
* Perform localization
* Contain raw user input

Those belong to **Decision Contracts** or **internal logs**.

---

## Mapping from Decision → Response (Conceptual)

| Decision Output         | Response Output              |
| ----------------------- | ---------------------------- |
| decision.primary_action | final_action.type            |
| missing_information     | payload.missing_information  |
| escalation.reason       | payload.reason               |
| risk.signals            | payload.signals              |
| —                       | response_id, timestamp, meta |

---

## Why this works for your current n8n design

* Your existing `Set_*` nodes already produce `type + payload`
* You only need **one final normalization node**
* `Respond to Webhook` can return **exactly this object**
* Debug data stays internal until the last step

---

