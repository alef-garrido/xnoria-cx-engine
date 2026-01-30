# Decision Contract v1 — README

## Scope

This contract defines the **authoritative output structure** of the decision-making stage in the CX engine.

It represents the result of analyzing an incoming request (e.g. email) and contains:

* intent classification
* risk assessment
* missing information detection
* the primary decision to be taken by downstream systems

This contract is produced by AI decision skills and **consumed by orchestration layers** (n8n workflows, routing logic, escalation handlers).

---

## Guarantees

Consumers of this contract can rely on the following:

* The object is **complete and self-contained** for decision routing.
* `decision.primary_action` is always present and belongs to a known, finite set.
* Risk and escalation are **explicitly stated**, never inferred.
* Fields are **descriptive, not imperative** (no side effects implied).
* No natural language response generation is included.
* The structure is stable within the same major version (`v1`).

This contract is suitable for:

* deterministic branching
* audit trails
* debugging and replay
* versioned automation pipelines

---

## Non-Goals

This contract explicitly does **not**:

* Define how responses are worded or formatted.
* Contain channel-specific output (email, chat, voice, etc.).
* Perform validation or enforcement of business rules.
* Represent final API responses to external consumers.
* Replace workflow logic or escalation policies.
* Optimize for UI rendering or end-user readability.

Those concerns are handled by:

* response contracts
* templates
* orchestration workflows
* downstream services

