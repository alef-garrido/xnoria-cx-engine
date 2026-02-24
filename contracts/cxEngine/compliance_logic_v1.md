# cxEngine Compliance Guardrails logic

This document outlines the logic for legal and ethical enforcement (TCPA, GDPR, etc.) in the cxEngine v1 system.

## 1. Pre-Send Check Flow

Every engine MUST call `pre_send_check` before emitting any message.

```typescript
FUNCTION pre_send_check(lead: Lead_v1, context: OBJECT) -> ComplianceDecision:
    // 1. Check Global Opt-Out List
    IF is_opted_out(lead.phone):
        RETURN { result: "blocked", reason: "opt_out", rule_id: "COMPL_001" }

    // 2. Check Frequency Limits (Global or per channel)
    IF exceeds_frequency_limit(lead.phone, context.channel, CONFIG.max_daily):
        RETURN { result: "blocked", reason: "frequency_limit", rule_id: "COMPL_002" }

    // 3. Check Business Hours (Timezone Aware)
    IF is_outside_allowed_hours(lead.timezone OR lead.phone_region):
        RETURN { result: "blocked", reason: "outside_hours", rule_id: "COMPL_003" }

    // 4. Region-Specific Checks
    IF context.region == "US" AND NOT has_consent(lead, "TCPA"):
        RETURN { result: "blocked", reason: "missing_tcpa_consent", rule_id: "COMPL_004" }

    RETURN { result: "allowed", reason: "checks_passed", rule_id: "OK_001" }
```

## 2. Opt-Out Management

### Detection Keywords (Case-Insensitive)
- **English**: STOP, UNSUBSCRIBE, CANCEL, END, QUIT
- **Spanish**: BAJA, CANCELAR, DETENER, PARAR, NO MAS

### Execution Logic
Upon detecting a keyword:
1. Immediately add phone to `GlobalOptOut` storage.
2. Update CRM: Set `lead.status = "disqualified"` and `meta.opt_out = true`.
3. Send a single confirmation message (mandatory in many regions).
4. Block all future messages.

## 3. PII Masking Rules for Logs
- **Phones**: Mask all but last 4 digits (e.g., `+52*******1234`).
- **Names**: Mask or use initials for log entries.
- **Emails**: Mask local part partially (e.g., `j***@example.com`).
