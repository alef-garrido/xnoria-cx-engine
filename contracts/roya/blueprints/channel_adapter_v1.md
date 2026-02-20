# INTERFACE: ChannelAdapter_v1

This interface defines the contract for communication channels (SMS, WhatsApp, Email) in the ROYA v1 system.

## METHODS

### `send_message(lead: Lead_v1, content: MessageContent, options: OBJECT) -> DeliveryResult`
- **Description**: Sends a message to a lead via the configured channel.
- **Inputs**:
    - `lead`: The `Lead_v1` object.
    - `content`: An object containing `type` (text|media|template) and relevant data (`body`, `media_url`, etc.).
    - `options`: Optional parameters like `timeout_ms`, `priority`.
- **Outputs**:
    - `DeliveryResult`: Object with `status` (sent|queued|failed|blocked), `provider_message_id`, and `error` (optional).

### `parse_incoming(raw_payload: OBJECT) -> NormalizedMessage_v1`
- **Description**: Normalizes an incoming webhook/event payload from the provider into the canonical ROYA message format.
- **Inputs**:
    - `raw_payload`: The raw JSON/Object from the provider (Twilio, Meta, etc.).
- **Outputs**:
    - `NormalizedMessage_v1`: Standardized message object.

## IMPLEMENTATION GUIDELINES
- Must handle rate limits and retries internally according to `Shared_Components` policies.
- Must log all delivery attempts with `TraceEmitter`.
- Must emit metrics for successful/failed deliveries with `MetricsEmitter`.
