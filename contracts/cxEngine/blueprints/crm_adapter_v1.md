# INTERFACE: CRMAdapter_v1

This interface defines the contract for interacting with Lead Management Systems (HighLevel, Salesforce, Hubspot) in the cxEngine v1 system.

## METHODS

### `get_lead_by_phone(phone: STRING) -> Lead_v1 | NULL`
- **Description**: Retrieves a lead record from the CRM using the phone number as the unique identifier.
- **Outputs**:
    - `Lead_v1`: The normalized lead object if found, else `NULL`.

### `update_lead(lead_id: STRING, update_data: OBJECT) -> SyncResult`
- **Description**: Updates fields on an existing lead record.
- **Inputs**:
    - `lead_id`: The identifier (internal or provider-specific).
    - `update_data`: Key-value pairs of fields to update (status, custom fields, tags).
- **Outputs**:
    - `SyncResult`: Object with `success` (boolean) and `error` (optional).

### `create_lead(lead: Lead_v1) -> SyncResult`
- **Description**: Creates a new lead record in the CRM.
- **Outputs**:
    - `SyncResult`: Object with `success` (boolean) and `provider_lead_id`.

## IMPLEMENTATION GUIDELINES
- Must ensure idempotency (e.g., deduplicating by phone).
- Must sync cxEngine internal status to CRM-specific stages/statuses.
- Must log all CRM interactions for auditability.
