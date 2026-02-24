# cxEngine Analytics & Metrics Framework v1

This document defines how metrics are emitted and tracked across all cxEngine engines.

## 1. Canonical Event Schema (`MetricsEntry_v1`)

```typescript
OBJECT MetricsEntry_v1:
    REQUIRED:
        schema_version: STRING ("v1")
        timestamp: ISO-8601
        engine: ENUM ["sleeping_beauty", "speed_to_lead", "document_collection", "out_of_hours"]
        event_type: STRING (e.g., "lead_ingested", "message_sent", "reply_received", "qualified")
        lead_id: STRING (UUID)
    
    OPTIONAL:
        niche: STRING
        channel: ENUM ["sms", "whatsapp", "email"]
        region: STRING
        latency_ms: INTEGER
        score: INTEGER
        metadata: OBJECT
```

## 2. Shared Metrics Node (n8n Blueprint)

All workflows should include a "Emit Metric" node (Code node) before final completion or after critical milestones.

```javascript
// Emit Metric Node Logic
const lead = $json;
const event = "message_sent"; // Configurable per node call

const metric = {
  schema_version: "v1",
  timestamp: new Date().toISOString(),
  engine: "sleeping_beauty", // Must be set per workflow
  event_type: event,
  lead_id: lead.lead_id,
  niche: lead.metadata.niche,
  channel: lead.channel,
  latency_ms: Date.now() - new Date(lead.created_at).getTime()
};

// In production, this sends to a database (InfluxDB, Prometheus) or an observability tool
return { json: metric };
```

## 3. Key Performance Indicators (KPIs)

| Engine | KPI | Target |
|---|---|---|
| Sleeping Beauty | Response Rate | > 15% |
| Speed to Lead | Acknowledgment Latency | < 60s |
| Document Collection | Completeness Rate | > 80% |
| Overall | ROI (Converted Leads) | Tracked in CRM |
