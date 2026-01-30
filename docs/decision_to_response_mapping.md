Below is a **clean, explicit Decision → Response mapping table** suitable for `docs/decision_to_response_mapping.md`.

No prose padding. This is an **operational reference**, not marketing.

---

# Decision → Response Mapping

This document defines how **Decision Contract outputs** are transformed into **Response Contract v1** objects.

The mapping is **deterministic**: the same decision input must always yield the same response shape.

---

## Mapping Overview

| Decision Field            | Used In Response                           | Notes                       |
| ------------------------- | ------------------------------------------ | --------------------------- |
| `input_id`                | `meta.trace_id`                            | Preserved for correlation   |
| `channel`                 | `channel`                                  | Passed through unchanged    |
| `decision.primary_action` | `final_action.type`                        | Canonical behavioral switch |
| `missing_information`     | `final_action.payload.missing_information` | Only for REQUEST_MORE_INFO  |
| `decision.reason`         | `final_action.payload.reason`              | Used where applicable       |
| `risk.level`              | `final_action.payload.level`               | Only for escalation         |
| `risk.signals`            | `final_action.payload.signals`             | Only for escalation         |
| —                         | `response_id`                              | Generated at response time  |
| —                         | `timestamp`                                | Generated at response time  |
| —                         | `meta.version`                             | Hardcoded (`v1`)            |

---

## Action-Specific Mapping

### 1. PROVIDE_INFO

**Trigger**

```json
decision.primary_action = "PROVIDE_INFO"
```

**Response**

```json
final_action.type = "PROVIDE_INFO"
```

**Payload**

| Field     | Source                              | Required |
| --------- | ----------------------------------- | -------- |
| `message` | Domain or channel-specific template | Yes      |

**Notes**

* No risk or escalation fields allowed
* Message generation may be deferred to a downstream renderer

---

### 2. REQUEST_MORE_INFO

**Trigger**

```json
decision.primary_action = "REQUEST_MORE_INFO"
```

**Response**

```json
final_action.type = "REQUEST_MORE_INFO"
```

**Payload**

| Field                 | Source                  | Required |
| --------------------- | ----------------------- | -------- |
| `missing_information` | `missing_information[]` | Yes      |
| `reason`              | `decision.reason`       | Optional |

**Notes**

* Payload must not include escalation or risk fields
* Used when intent is valid but insufficient data exists

---

### 3. ESCALATE_HUMAN

**Trigger**

```json
decision.primary_action = "ESCALATE_HUMAN"
```

**Response**

```json
final_action.type = "ESCALATE_HUMAN"
```

**Payload**

| Field     | Source                                   | Required |
| --------- | ---------------------------------------- | -------- |
| `level`   | `risk.level`                             | Yes      |
| `reason`  | `decision.reason` OR `escalation.reason` | Yes      |
| `signals` | `risk.signals[]`                         | Optional |

**Notes**

* No automated response content allowed
* Consumer must route to a human operator or emergency flow

---

### 4. REJECT_REQUEST

**Trigger**

```json
decision.primary_action = "REJECT_REQUEST"
```

**Response**

```json
final_action.type = "REJECT_REQUEST"
```

**Payload**

| Field    | Source            | Required |
| -------- | ----------------- | -------- |
| `reason` | `decision.reason` | Yes      |

**Notes**

* Used for out-of-scope or policy-blocked requests
* No retry or follow-up implied

---

### 5. SCHEDULE_REQUEST

**Trigger**

```json
decision.primary_action = "SCHEDULE_REQUEST"
```

**Response**

```json
final_action.type = "SCHEDULE_REQUEST"
```

**Payload**

| Field    | Source            | Required |
| -------- | ----------------- | -------- |
| `reason` | `decision.reason` | Optional |

**Notes**

* Does not perform scheduling
* Signals readiness for downstream booking system

---

## Invalid Mappings (Explicitly Forbidden)

| Condition                                            | Reason              |
| ---------------------------------------------------- | ------------------- |
| `risk.level = CRITICAL` + non-escalation action      | Safety violation    |
| Missing `missing_information[]` on REQUEST_MORE_INFO | Incomplete response |
| Payload fields not aligned with `final_action.type`  | Contract breach     |
| Natural language diagnosis or advice                 | Out of scope        |

---

## Contract Enforcement Rules

* **Exactly one** `final_action` per response
* `final_action.type` is mandatory and authoritative
* Consumers must **ignore all internal decision fields**
* Only fields defined in `response_contract_v1.json` may be exposed

---

## Versioning

This mapping applies to:

```
Decision Contract: v1
Response Contract: v1
```

Changes require:

* new mapping doc
* version bump
* backward compatibility review

