# 📄 Product Requirements Document (PRD) Técnico

## cxEngine_Shared_Components_v1
**Versión:** 1.0  
**Fecha:** 20 de febrero de 2026  
**Estado:** Especificación para implementación  
**Dominio:** cxEngine / Componentes Compartidos  
**Stack referencia:** Agnóstico (n8n/Make/Zapier + LLM + SMS/WhatsApp + CRM)

---

## 1. Overview Técnico

### 1.1 Propósito del módulo
Proveer una capa de componentes reutilizables, agnósticos y contract-first que habiliten la implementación consistente de todos los Androids de ventas (Sleeping Beauty, Speed To Lead, Out Of Hours, Document Collection) sin duplicación de lógica ni acoplamiento a stack específico.

### 1.2 Arquitectura de alto nivel
```
[Android Specific Workflows]
           ↓
[cxEngine Shared Components Layer]
           ↓
[External Adapters: SMS, CRM, LLM, Vector Store]
```

### 1.3 Principios técnicos fundamentales
- **Agnosticismo de stack:** Interfaces, no implementaciones
- **Contract-first:** Todos los componentes comunican mediante schemas definidos
- **Stateless por diseño:** Cada interacción es independiente; el estado se externaliza
- **Composición sobre herencia:** Componentes pequeños y combinables
- **Observabilidad nativa:** Todo componente emite trazas estructuradas

---

## 2. Problem Statement (Técnico)

### 2.1 Problema actual
Sin una capa compartida bien definida, cada Android de ventas tiende a:
- **Duplicar lógica:** Normalización de leads, manejo de errores, logging
- **Acoplarse a stack:** Código específico de HighLevel, Twilio, etc.
- **Romper contratos:** Outputs inconsistentes entre Androids
- **Dificultar testing:** Sin interfaces claras, no hay mocks estables

### 2.2 Impacto técnico
- Tiempo de desarrollo multiplicado por número de Androids
- Bugs que se replican en múltiples flujos
- Imposibilidad de reemplazar un adapter sin reescribir lógica
- Auditoría fragmentada y difícil de correlacionar

---

## 3. Technical Objectives

### 3.1 Objetivos SMART

| ID | Objetivo | Métrica | Target |
|----|----------|---------|--------|
| **TO-1** | Reducir duplicación de código entre Androids | Reuse rate de componentes | ≥ 80% |
| **TO-2** | Habilitar swap de adapters sin cambios en lógica | Adapter abstraction level | 100% interfaces |
| **TO-3** | Garantizar consistencia de contratos cross-Android | Schema validation success | 100% |
| **TO-4** | Proveer observabilidad unificada | Trace correlation rate | ≥ 95% |
| **TO-5** | Minimizar tiempo de onboarding para nuevos Androids | Time to first Android | < 2 días |

### 3.2 Requerimientos de calidad (NFRs)

| NFR | Descripción | Target |
|-----|-------------|--------|
| **NFR-1** | Disponibilidad de componentes | 99.9% uptime |
| **NFR-2** | Latencia agregada por capa shared | < 100ms por componente |
| **NFR-3** | Throughput sostenible | 200 interacciones/minuto |
| **NFR-4** | Data retention para audit | 90 días mínimo |
| **NFR-5** | Error handling consistente | 100% de errores capturados y logueados |

---

## 4. Scope Técnico

### 4.1 In Scope

| Componente | Descripción | Responsabilidad |
|------------|-------------|-----------------|
| **Lead Schema v1** | Estructura canónica de lead para todos los Androids | Definición de contrato |
| **ConversationState Manager** | Gestión de estado conversacional externalizado (cache/DB) | Estado transitorio |
| **Channel Adapter Interface** | Interfaz unificada para SMS/WhatsApp/Email | Abstracción de canal |
| **CRM Adapter Interface** | Interfaz para HighLevel, Salesforce, HubSpot, etc. | Abstracción de CRM |
| **LLM Prompt Templates Base** | Plantillas base para clasificación, calificación, nurturing | Consistencia de prompts |
| **Error Handling & Retry Policy** | Política unificada de reintentos y fallbacks | Resiliencia |
| **Trace & Audit Schema** | Estructura para logging y correlación de trazas | Observabilidad |
| **Compliance Guardrails Hooks** | Puntos de extensión para validación TCPA/GDPR | Compliance transversal |
| **Metrics Emission Interface** | Interface para emitir métricas estandarizadas | Analytics |

### 4.2 Out of Scope

| Componente | Razón de exclusión |
|------------|-------------------|
| Implementaciones concretas de adapters | Cada deployment elige su stack |
| UI/Dashboard para monitoreo | Capa de presentación separada |
| Machine Learning training | Solo inferencia en v1 |
| Gestión de usuarios/roles | Fuera de scope de Androids |
| Billing/Payments integration | Responsabilidad del socio, no del sistema |

---

## 5. Functional Requirements

### 5.1 FR-1: Lead Schema v1 (Contrato Canónico)

**ID:** FR-1  
**Prioridad:** CRÍTICA  
**Descripción:** Definir una estructura única de lead que todos los Androids consuman y produzcan.

**User Story:**  
Como desarrollador de un Android, quiero una estructura de lead estandarizada para que mi flujo sea compatible con los demás componentes sin transformaciones ad-hoc.

**Acceptance Criteria:**
- [ ] Schema definido en JSON Schema Draft-07
- [ ] Campos obligatorios: `lead_id`, `phone`, `source`, `status`, `created_at`
- [ ] Campos opcionales: `email`, `name`, `custom_fields`, `metadata`
- [ ] `status` enum: `new`, `contacted`, `qualified`, `disqualified`, `converted`, `dead`
- [ ] `source` enum: `db_reactivation`, `fresh_optin`, `out_of_hours`, `document_request`
- [ ] `custom_fields` permite extensión por dominio sin romper schema
- [ ] Versionado explícito: `schema_version: "v1"`

**Lead Schema v1 (Pseudocódigo estructural):**
```
OBJECT Lead_v1:
    REQUIRED:
        lead_id: STRING (UUID or provider-specific ID)
        phone: STRING (E.164 format: +521234567890)
        source: ENUM ["db_reactivation", "fresh_optin", "out_of_hours", "document_request"]
        status: ENUM ["new", "contacted", "qualified", "disqualified", "converted", "dead"]
        created_at: ISO-8601 TIMESTAMP
        schema_version: STRING (fixed: "v1")
    
    OPTIONAL:
        email: STRING (validated format)
        name: STRING (full name)
        custom_fields: OBJECT (arbitrary key-value, domain-specific)
        metadata: OBJECT
            provider_lead_id: STRING
            campaign_id: STRING
            utm_params: OBJECT
            last_interaction_at: ISO-8601
            interaction_count: INTEGER
```

---

### 5.2 FR-2: ConversationState Manager (Stateless por Diseño)

**ID:** FR-2  
**Prioridad:** ALTA  
**Descripción:** Gestionar estado conversacional transitorio sin acoplar lógica de negocio a persistencia.

**User Story:**  
Como sistema de conversación, necesito recordar el contexto de una interacción en curso sin mantener estado en memoria, para ser escalable y tolerante a fallos.

**Acceptance Criteria:**
- [ ] Interface con métodos: `get_state(lead_id)`, `set_state(lead_id, state)`, `clear_state(lead_id)`
- [ ] TTL configurable por tipo de estado (default: 24 horas)
- [ ] Estado serializable a JSON
- [ ] Implementación reference: Redis o memoria volátil
- [ ] Fallo de cache NO bloquea el flujo (fallback a estado vacío)
- [ ] Logging de operaciones de estado para audit

**Pseudocódigo de interface:**
```
INTERFACE ConversationStateManager:
    METHOD get_state(lead_id: STRING) → OBJECT OR NULL
        RETURNS: Estado conversacional o NULL si no existe
        BEHAVIOR: 
            - Intenta obtener de cache (Redis/memory)
            - Si falla, retorna NULL sin error
            - Log: "state_read", lead_id, hit/miss
    
    METHOD set_state(lead_id: STRING, state: OBJECT, ttl: INTEGER) → BOOLEAN
        PARAMETERS:
            state: OBJECT con campos:
                thread_id: STRING
                last_message_at: ISO-8601
                conversation_stage: ENUM ["init", "qualifying", "scheduling", "closing"]
                context_snapshot: OBJECT (resumen de contexto relevante)
            ttl: INTEGER (segundos, default: 86400 = 24h)
        RETURNS: true si éxito, false si fallo (sin lanzar excepción)
        BEHAVIOR:
            - Serializa state a JSON
            - Almacena con TTL en cache
            - Log: "state_written", lead_id, success/fail
    
    METHOD clear_state(lead_id: STRING) → BOOLEAN
        RETURNS: true si existía y fue eliminado, false si no existía
        BEHAVIOR:
            - Elimina entrada de cache
            - Log: "state_cleared", lead_id
```

---

### 5.3 FR-3: Channel Adapter Interface (SMS/WhatsApp/Email)

**ID:** FR-3  
**Prioridad:** CRÍTICA  
**Descripción:** Abstraer la comunicación por canal para que la lógica de decisión sea agnóstica al medio.

**User Story:**  
Como motor de decisiones, quiero enviar y recibir mensajes sin saber si es SMS, WhatsApp o email, para poder cambiar de canal sin reescribir lógica.

**Acceptance Criteria:**
- [ ] Interface con métodos: `send_message(lead, content)`, `parse_incoming(raw_payload)`, `get_channel_type()`
- [ ] `send_message` retorna estado de entrega: `sent`, `queued`, `failed`
- [ ] `parse_incoming` normaliza cualquier payload a `NormalizedMessage`
- [ ] Soporte para multimedia: texto, imagen (metadata), audio (transcrito)
- [ ] Manejo de opt-out: método `is_opted_out(phone)` consulta lista de baja
- [ ] Rate limiting integrado: `can_send(phone)` verifica límites por canal

**NormalizedMessage Schema:**
```
OBJECT NormalizedMessage:
    message_id: STRING (único por canal)
    from: STRING (E.164 o email)
    to: STRING (E.164 o email)
    body: STRING (texto normalizado, sin formato)
    timestamp: ISO-8601
    channel: ENUM ["sms", "whatsapp", "email"]
    media: OBJECT OR NULL
        type: ENUM ["image", "audio", "document"]
        url: STRING (si aplica)
        transcript: STRING (si audio, después de STT)
        caption: STRING (si imagen, texto asociado)
    metadata: OBJECT (datos crudos del canal para debug)
```

**Pseudocódigo de interface:**
```
INTERFACE ChannelAdapter:
    METHOD send_message(lead: Lead_v1, content: OBJECT) → DeliveryResult
        PARAMETERS:
            content: OBJECT con:
                type: ENUM ["text", "template", "multimedia"]
                body: STRING (para type=text)
                template_name: STRING (para type=template, WhatsApp)
                media_url: STRING (para type=multimedia)
        RETURNS: OBJECT DeliveryResult
            status: ENUM ["sent", "queued", "failed"]
            provider_message_id: STRING OR NULL
            error: STRING OR NULL (si failed)
        BEHAVIOR:
            - Verifica opt-out antes de enviar
            - Aplica rate limiting del canal
            - Log: "message_sent", lead_id, channel, status
    
    METHOD parse_incoming(raw_payload: OBJECT) → NormalizedMessage OR NULL
        RETURNS: Mensaje normalizado o NULL si no parseable
        BEHAVIOR:
            - Extrae campos según formato del canal
            - Normaliza phone a E.164
            - Si audio, invoca STT service (async o sync según config)
            - Log: "message_parsed", channel, success/fail
    
    METHOD get_channel_type() → STRING
        RETURNS: "sms" | "whatsapp" | "email"
    
    METHOD is_opted_out(phone: STRING) → BOOLEAN
        RETURNS: true si el número está en lista de baja
        BEHAVIOR:
            - Consulta cache local o lista compartida
            - Log: "optout_check", phone, result
    
    METHOD can_send(phone: STRING) → BOOLEAN
        RETURNS: true si se puede enviar mensaje ahora
        BEHAVIOR:
            - Verifica rate limits del canal
            - Verifica ventana de respuesta (WhatsApp 24h)
            - Log: "rate_limit_check", phone, result
```

---

### 5.4 FR-4: CRM Adapter Interface (HighLevel, Salesforce, etc.)

**ID:** FR-4  
**Prioridad:** ALTA  
**Descripción:** Abstraer la interacción con CRMs para que los Androids sean agnósticos a la plataforma del socio.

**User Story:**  
Como sistema de ventas, quiero actualizar leads y agendar citas sin depender de si el socio usa HighLevel, Salesforce u otro CRM.

**Acceptance Criteria:**
- [ ] Interface con métodos: `upsert_lead(lead)`, `update_status(lead_id, status)`, `schedule_appointment(lead, details)`, `get_lead(lead_id)`
- [ ] `upsert_lead` es idempotente: mismo lead_id → mismo registro
- [ ] `schedule_appointment` retorna confirmación o error con razón
- [ ] Mapeo de `Lead_v1.status` a estados del CRM específico
- [ ] Fallback graceful si CRM está indisponible (queue + retry)
- [ ] Logging de todas las operaciones para reconciliación

**Pseudocódigo de interface:**
```
INTERFACE CRMAdapter:
    METHOD upsert_lead(lead: Lead_v1) → UpsertResult
        RETURNS: OBJECT UpsertResult
            success: BOOLEAN
            crm_lead_id: STRING OR NULL
            created: BOOLEAN (true si nuevo, false si actualizado)
            error: STRING OR NULL
        BEHAVIOR:
            - Mapea Lead_v1 a formato del CRM
            - Usa lead_id o phone como clave de búsqueda
            - Log: "lead_upsert", lead_id, success, created
    
    METHOD update_status(lead_id: STRING, new_status: ENUM, reason: STRING) → BOOLEAN
        RETURNS: true si éxito, false si fallo
        BEHAVIOR:
            - Mapea status cxEngine a estado del CRM
            - Registra reason en campo de notas del CRM
            - Log: "status_updated", lead_id, new_status
    
    METHOD schedule_appointment(lead: Lead_v1, details: OBJECT) → ScheduleResult
        PARAMETERS:
            details: OBJECT con:
                start_time: ISO-8601
                duration_minutes: INTEGER
                timezone: STRING (IANA)
                meeting_link: STRING OR NULL
                notes: STRING
        RETURNS: OBJECT ScheduleResult
            success: BOOLEAN
            appointment_id: STRING OR NULL
            calendar_link: STRING OR NULL
            error: STRING OR NULL
        BEHAVIOR:
            - Verifica disponibilidad en calendario del CRM
            - Crea evento y asigna al lead
            - Envía confirmación si configurado
            - Log: "appointment_scheduled", lead_id, success
    
    METHOD get_lead(lead_id: STRING) → Lead_v1 OR NULL
        RETURNS: Lead en formato cxEngine o NULL si no encontrado
        BEHAVIOR:
            - Consulta CRM por lead_id o phone
            - Mapea respuesta a Lead_v1
            - Log: "lead_retrieved", lead_id, found
    
    METHOD is_available() → BOOLEAN
        RETURNS: true si el CRM está respondiendo
        BEHAVIOR:
            - Health check ligero (sin carga)
            - Log: "crm_health_check", result
```

---

### 5.5 FR-5: LLM Prompt Templates Base

**ID:** FR-5  
**Prioridad:** ALTA  
**Descripción:** Proveer plantillas base de prompts para tareas comunes de los Androids, con placeholders para personalización por nicho.

**User Story:**  
Como implementador, quiero prompts probados y estructurados para clasificación, calificación y nurturing, para no reinventar la rueda en cada Android.

**Acceptance Criteria:**
- [ ] Plantillas en formato de pseudocódigo con placeholders `{variable}`
- [ ] Categorías cubiertas: `intent_classification`, `lead_qualification`, `objection_handling`, `appointment_scheduling`
- [ ] Cada plantilla incluye: system prompt, user prompt template, response schema
- [ ] Placeholders documentados: `{niche}`, `{offer}`, `{objections_list}`, `{calendar_link}`
- [ ] Response schema enforceable (JSON mode del LLM)
- [ ] Versionado de plantillas: `template_version: "v1"`

**Pseudocódigo de plantilla base (ejemplo: lead_qualification):**
```
TEMPLATE: lead_qualification_v1

SYSTEM_PROMPT:
"""
Eres un asistente de ventas para {niche}. Tu objetivo es calificar leads 
mediante conversación natural por SMS/WhatsApp.

REGLAS:
1. Sé breve: máximo 2 frases por mensaje
2. Haz una pregunta a la vez
3. Si el lead muestra interés real → agenda cita
4. Si el lead no es viable → descalifica amablemente
5. NUNCA inventes información sobre {offer}

CONTEXTO DEL NEGOCIO:
- Oferta principal: {offer}
- Precio base: {price_range}
- Objeciones comunes: {objections_list}
- Link para agendar: {calendar_link}

FORMATO DE SALIDA (JSON estricto):
{
  "qualification_result": "qualified|unqualified|needs_more_info",
  "confidence": "high|medium|low",
  "next_action": "schedule|ask_more|disqualify|escalate",
  "reason": "string (≤100 chars)",
  "missing_info": ["string"] (si needs_more_info)
}
"""

USER_PROMPT_TEMPLATE:
"""
Lead: {lead_name} ({phone})
Historial reciente: {conversation_summary}
Último mensaje del lead: "{last_message}"

Califica este lead según las reglas.
"""

RESPONSE_SCHEMA:
{
  "type": "object",
  "required": ["qualification_result", "confidence", "next_action", "reason"],
  "properties": {
    "qualification_result": {"enum": ["qualified", "unqualified", "needs_more_info"]},
    "confidence": {"enum": ["high", "medium", "low"]},
    "next_action": {"enum": ["schedule", "ask_more", "disqualify", "escalate"]},
    "reason": {"type": "string", "maxLength": 100},
    "missing_info": {"type": "array", "items": {"type": "string"}}
  }
}
```

---

### 5.6 FR-6: Error Handling & Retry Policy Unificada

**ID:** FR-6  
**Prioridad:** ALTA  
**Descripción:** Definir política consistente de manejo de errores y reintentos para todos los componentes.

**User Story:**  
Como operador, quiero que los fallos temporales se manejen automáticamente y los permanentes se logueen claramente, para mantener el sistema estable sin intervención manual constante.

**Acceptance Criteria:**
- [ ] Clasificación de errores: `TRANSIENT`, `PERMANENT`, `BUSINESS_RULE`
- [ ] Política de reintentos: backoff exponencial (base: 1s, max: 60s, max_attempts: 3)
- [ ] Dead Letter Queue para errores permanentes después de reintentos
- [ ] Alertas configurables por tipo de error y frecuencia
- [ ] Logging estructurado con contexto suficiente para debug
- [ ] Métricas de error emitidas para monitoreo

**Pseudocódigo de política de errores:**
```
CLASS ErrorPolicy:
    ENUM ErrorType: TRANSIENT, PERMANENT, BUSINESS_RULE
    
    METHOD classify_error(error: OBJECT) → ErrorType
        RULES:
            IF error.code IN [429, 500, 502, 503, 504]: RETURN TRANSIENT
            IF error.code IN [400, 401, 403, 404]: RETURN PERMANENT
            IF error.reason IN ["opt_out", "invalid_lead", "out_of_scope"]: RETURN BUSINESS_RULE
            DEFAULT: RETURN PERMANENT
    
    METHOD should_retry(error_type: ErrorType, attempt: INTEGER) → BOOLEAN
        IF error_type != TRANSIENT: RETURN false
        IF attempt >= 3: RETURN false
        RETURN true
    
    METHOD get_backoff_delay(attempt: INTEGER) → INTEGER (milliseconds)
        // Backoff exponencial con jitter
        base = 1000
        max = 60000
        delay = MIN(base * (2 ^ (attempt - 1)), max)
        jitter = RANDOM(0, delay * 0.1)
        RETURN delay + jitter
    
    METHOD handle_error(context: OBJECT, error: OBJECT) → ActionResult
        error_type = classify_error(error)
        attempt = context.retry_count OR 0
        
        IF should_retry(error_type, attempt):
            delay = get_backoff_delay(attempt + 1)
            LOG: "retry_scheduled", context, error_type, attempt, delay
            SCHEDULE_RETRY(context, delay)
            RETURN { action: "retry", delay: delay }
        
        ELSE IF error_type == BUSINESS_RULE:
            LOG: "business_rule_violation", context, error.reason
            RETURN { action: "skip", reason: error.reason }
        
        ELSE:
            LOG: "permanent_error", context, error
            SEND_TO_DLQ(context, error)
            IF alert_threshold_exceeded(error_type):
                TRIGGER_ALERT(error_type, context)
            RETURN { action: "fail", error: error }
```

---

### 5.7 FR-7: Trace & Audit Schema (Observabilidad Unificada)

**ID:** FR-7  
**Prioridad:** ALTA  
**Descripción:** Definir estructura de logging y trazas para correlacionar eventos cross-Android y cross-componente.

**User Story:**  
Como auditor o operador, quiero ver el flujo completo de un lead a través de todos los componentes, para debuggear, optimizar y cumplir con compliance.

**Acceptance Criteria:**
- [ ] Schema de traza con campos obligatorios: `trace_id`, `span_id`, `timestamp`, `component`, `event_type`, `lead_id`
- [ ] `trace_id` único por interacción de lead, propagado entre componentes
- [ ] `span_id` único por operación dentro de una traza
- [ ] Contexto enriquecido: `metadata`, `input_snapshot`, `output_snapshot`, `error` (si aplica)
- [ ] Exportable a formatos estándar: JSON, OpenTelemetry
- [ ] Retención configurable: default 90 días

**Trace Schema v1 (Pseudocódigo estructural):**
```
OBJECT TraceEntry_v1:
    REQUIRED:
        trace_id: STRING (UUID, mismo para toda la interacción del lead)
        span_id: STRING (UUID, único por operación)
        parent_span_id: STRING OR NULL (para tracing jerárquico)
        timestamp: ISO-8601 (cuando ocurrió el evento)
        component: STRING (ej: "channel_adapter", "crm_adapter", "llm_classifier")
        event_type: ENUM ["input_received", "decision_made", "action_executed", "error_occurred", "state_updated"]
        lead_id: STRING (referencia al Lead_v1)
    
    OPTIONAL:
        channel: ENUM ["sms", "whatsapp", "email"] (si aplica)
        android: ENUM ["sleeping_beauty", "speed_to_lead", "out_of_hours", "document_collection"]
        input_snapshot: OBJECT (copia del input, sin PII sensible)
        output_snapshot: OBJECT (copia del output)
        error: OBJECT
            code: STRING
            message: STRING
            type: ENUM ["transient", "permanent", "business_rule"]
        metadata: OBJECT (datos adicionales para contexto)
        duration_ms: INTEGER (tiempo de la operación)
```

**Pseudocódigo de emisión de trazas:**
```
CLASS TraceEmitter:
    METHOD start_trace(lead_id: STRING, android: ENUM, channel: ENUM) → STRING
        RETURNS: trace_id (UUID)
        BEHAVIOR:
            - Genera nuevo trace_id
            - Emite entrada "trace_started"
            - Retorna trace_id para propagación
    
    METHOD emit_span(trace_id: STRING, event: TraceEntry_v1) → VOID
        BEHAVIOR:
            - Valida que event.trace_id == trace_id
            - Genera span_id si no presente
            - Serializa a JSON
            - Envía a sink configurado (stdout, file, remote)
            - Log local para fallback
    
    METHOD end_trace(trace_id: STRING, final_status: ENUM) → VOID
        BEHAVIOR:
            - Emite entrada "trace_ended" con status
            - Cierra contexto de traza
    
    METHOD with_trace_context(trace_id: STRING, callback: FUNCTION) → ANY
        // Helper para propagar trace_id en callbacks
        BEHAVIOR:
            - Establece trace_id en contexto thread-local / async context
            - Ejecuta callback
            - Limpia contexto
            - Retorna resultado de callback
```

---

### 5.8 FR-8: Compliance Guardrails Hooks (Puntos de Extensión)

**ID:** FR-8  
**Prioridad:** CRÍTICA  
**Descripción:** Proveer hooks para validar compliance (TCPA, GDPR, opt-out) antes de ejecutar acciones.

**User Story:**  
Como operador en región regulada, quiero que el sistema verifique automáticamente consentimiento y límites legales antes de enviar mensajes, para evitar multas y proteger la reputación.

**Acceptance Criteria:**
- [ ] Hooks definidos como interfaces: `pre_send_check(lead, content)`, `post_interaction_audit(lead, outcome)`
- [ ] `pre_send_check` retorna: `allowed`, `blocked`, `requires_review`
- [ ] Validaciones base: opt-out, frecuencia máxima, horario permitido, consentimiento explícito
- [ ] Configuración por región: `compliance_config.region` afecta reglas aplicadas
- [ ] Logging obligatorio de todas las decisiones de compliance
- [ ] Fallback seguro: si hook falla → bloquear acción (fail-safe)

**Pseudocódigo de hooks:**
```
INTERFACE ComplianceGuardrails:
    METHOD pre_send_check(lead: Lead_v1, content: OBJECT, context: OBJECT) → ComplianceDecision
        RETURNS: OBJECT ComplianceDecision
            result: ENUM ["allowed", "blocked", "requires_review"]
            reason: STRING (explicación humana)
            rule_id: STRING (identificador de regla aplicada)
        BEHAVIOR:
            // Reglas base (siempre aplican)
            IF is_opted_out(lead.phone): RETURN { blocked, "opt-out", "OPT_OUT_001" }
            IF exceeds_frequency_limit(lead.phone): RETURN { blocked, "rate_limit", "RATE_001" }
            IF outside_allowed_hours(lead.timezone): RETURN { blocked, "hours", "HOURS_001" }
            
            // Reglas por región (configurables)
            IF context.region == "US" AND NOT has_tcpa_consent(lead.phone):
                RETURN { blocked, "consent", "TCPA_001" }
            IF context.region == "EU" AND NOT has_gdpr_basis(lead.phone):
                RETURN { blocked, "consent", "GDPR_001" }
            
            // Default: permitir
            RETURN { allowed, "checks_passed", "OK_001" }
    
    METHOD post_interaction_audit(lead: Lead_v1, outcome: OBJECT, context: OBJECT) → VOID
        BEHAVIOR:
            // Registra interacción para reporting de compliance
            LOG: "compliance_audit", lead_id, outcome, context
            // Actualiza contadores de frecuencia
            UPDATE frequency_counter(lead.phone)
            // Si outcome fue escalación, notificar a equipo de compliance
            IF outcome.escalated: NOTIFY_COMPLIANCE_TEAM(lead, outcome)
    
    METHOD get_compliance_config(region: STRING) → OBJECT
        RETURNS: Configuración de reglas para la región
        BEHAVIOR:
            - Carga config desde fuente centralizada
            - Merge con defaults
            - Cachea con TTL corto
```

---

### 5.9 FR-9: Metrics Emission Interface (Analytics Unificado)

**ID:** FR-9  
**Prioridad:** MEDIA  
**Descripción:** Interface para emitir métricas estandarizadas que permitan medir performance cross-Android.

**User Story:**  
Como analista, quiero métricas consistentes de todos los Androids para comparar performance, identificar cuellos de botella y optimizar ROI.

**Acceptance Criteria:**
- [ ] Métricas base definidas: `lead_processed`, `decision_made`, `action_executed`, `error_occurred`, `conversion_tracked`
- [ ] Cada métrica incluye: `name`, `value`, `timestamp`, `tags` (android, channel, region, status)
- [ ] Formato compatible con Prometheus, Datadog, CloudWatch
- [ ] Emisión asíncrona para no bloquear flujo principal
- [ ] Sampling configurable para alto volumen
- [ ] Métricas de negocio: `response_rate`, `qualification_rate`, `appointment_rate`, `revenue_attributed`

**Pseudocódigo de interface:**
```
INTERFACE MetricsEmitter:
    METHOD emit_counter(name: STRING, value: INTEGER, tags: OBJECT) → VOID
        BEHAVIOR:
            - Valida que name esté en lista permitida
            - Agrega tags obligatorios: android, channel, version
            - Serializa a formato del backend configurado
            - Envía asíncronamente (no bloquea)
            - Log local para fallback
    
    METHOD emit_gauge(name: STRING, value: FLOAT, tags: OBJECT) → VOID
        // Para métricas de estado: queue_size, latency_p95, etc.
        BEHAVIOR: similar a emit_counter
    
    METHOD emit_histogram(name: STRING, value: FLOAT, tags: OBJECT) → VOID
        // Para distribuciones: latency, message_length, etc.
        BEHAVIOR: similar a emit_counter
    
    METHOD emit_business_metric(name: STRING, value: FLOAT, tags: OBJECT, unit: STRING) → VOID
        // Métricas de negocio: response_rate, conversion_rate, revenue
        PARAMETERS:
            unit: ENUM ["percent", "currency", "count", "duration"]
        BEHAVIOR:
            - Valida que name sea métrica de negocio registrada
            - Agrega contexto de atribución si disponible
            - Emite con prioridad alta (menos sampling)
    
    METHOD flush() → VOID
        // Fuerza envío de métricas pendientes (para shutdown limpio)
        BEHAVIOR:
            - Espera envío asíncrono pendiente (timeout: 5s)
            - Log: "metrics_flushed", count
```

---

## 6. Non-Functional Requirements

### 6.1 Performance

| Métrica | Target | Medición |
|---------|--------|----------|
| Latencia agregada por componente shared | < 100ms | P95 por operación |
| Throughput de componentes | 200 ops/minuto | Sustained load |
| Cache hit rate (ConversationState) | ≥ 90% | Métrica interna |
| Error handling overhead | < 10ms | Comparado con happy path |

### 6.2 Reliability

| Métrica | Target | Estrategia |
|---------|--------|------------|
| Disponibilidad de interfaces | 99.9% | Circuit breaker, fallbacks |
| Consistencia de contratos | 100% | Schema validation en CI |
| Tolerancia a fallos de adapters | Graceful degradation | Retry + DLQ + alertas |
| Recuperación tras fallo | < 1 minuto | Health checks + auto-restart |

### 6.3 Security

| Requisito | Implementación |
|-----------|----------------|
| PII en logs | Masking automático: phone → `+52***`, email → `u***@***` |
| Credenciales de adapters | Secrets management, nunca en código |
| Validación de input | Schema enforcement en todos los boundaries |
| Auditoría de decisiones | Trace schema obligatorio, inmutable |

### 6.4 Scalability

| Dimensión | Estrategia |
|-----------|------------|
| Horizontal | Componentes stateless, escalado por réplicas |
| Multi-tenant | Aislamiento por `partner_id` en metadata |
| Multi-región | Config por región en Compliance Guardrails |
| Multi-Android | Composición de componentes, no duplicación |

---

## 7. Technical Specifications

### 7.1 Contratos Canónicos (JSON Schema Draft-07)

#### Lead Schema v1
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://cxEngine.dev/schemas/lead_v1.json",
  "title": "Lead Schema v1",
  "type": "object",
  "required": ["lead_id", "phone", "source", "status", "created_at", "schema_version"],
  "properties": {
    "lead_id": {"type": "string", "format": "uuid"},
    "phone": {"type": "string", "pattern": "^\\+[1-9]\\d{1,14}$"},
    "source": {"enum": ["db_reactivation", "fresh_optin", "out_of_hours", "document_request"]},
    "status": {"enum": ["new", "contacted", "qualified", "disqualified", "converted", "dead"]},
    "created_at": {"type": "string", "format": "date-time"},
    "schema_version": {"const": "v1"},
    "email": {"type": "string", "format": "email"},
    "name": {"type": "string"},
    "custom_fields": {"type": "object", "additionalProperties": true},
    "metadata": {
      "type": "object",
      "properties": {
        "provider_lead_id": {"type": "string"},
        "campaign_id": {"type": "string"},
        "utm_params": {"type": "object"},
        "last_interaction_at": {"type": "string", "format": "date-time"},
        "interaction_count": {"type": "integer", "minimum": 0}
      }
    }
  }
}
```

#### NormalizedMessage Schema
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "NormalizedMessage Schema",
  "type": "object",
  "required": ["message_id", "from", "to", "body", "timestamp", "channel"],
  "properties": {
    "message_id": {"type": "string"},
    "from": {"type": "string"},
    "to": {"type": "string"},
    "body": {"type": "string"},
    "timestamp": {"type": "string", "format": "date-time"},
    "channel": {"enum": ["sms", "whatsapp", "email"]},
    "media": {
      "type": ["object", "null"],
      "properties": {
        "type": {"enum": ["image", "audio", "document"]},
        "url": {"type": "string"},
        "transcript": {"type": "string"},
        "caption": {"type": "string"}
      }
    },
    "metadata": {"type": "object", "additionalProperties": true}
  }
}
```

### 7.2 Interfaces en Pseudocódigo (Resumen Ejecutivo)

```
// === Componentes Principales ===

INTERFACE ConversationStateManager:
    get_state(lead_id) → OBJECT|NULL
    set_state(lead_id, state, ttl) → BOOLEAN
    clear_state(lead_id) → BOOLEAN

INTERFACE ChannelAdapter:
    send_message(lead, content) → DeliveryResult
    parse_incoming(raw_payload) → NormalizedMessage|NULL
    get_channel_type() → STRING
    is_opted_out(phone) → BOOLEAN
    can_send(phone) → BOOLEAN

INTERFACE CRMAdapter:
    upsert_lead(lead) → UpsertResult
    update_status(lead_id, status, reason) → BOOLEAN
    schedule_appointment(lead, details) → ScheduleResult
    get_lead(lead_id) → Lead_v1|NULL
    is_available() → BOOLEAN

INTERFACE ComplianceGuardrails:
    pre_send_check(lead, content, context) → ComplianceDecision
    post_interaction_audit(lead, outcome, context) → VOID
    get_compliance_config(region) → OBJECT

INTERFACE MetricsEmitter:
    emit_counter(name, value, tags) → VOID
    emit_gauge(name, value, tags) → VOID
    emit_histogram(name, value, tags) → VOID
    emit_business_metric(name, value, tags, unit) → VOID
    flush() → VOID

// === Utilidades ===

CLASS ErrorPolicy:
    classify_error(error) → ErrorType
    should_retry(error_type, attempt) → BOOLEAN
    get_backoff_delay(attempt) → INTEGER
    handle_error(context, error) → ActionResult

CLASS TraceEmitter:
    start_trace(lead_id, android, channel) → STRING
    emit_span(trace_id, event) → VOID
    end_trace(trace_id, final_status) → VOID
    with_trace_context(trace_id, callback) → ANY
```

### 7.3 Configuración por Ambiente (Environment Variables)

```bash
# === Shared Components Config ===
cxEngine_SCHEMA_VERSION=v1
cxEngine_DEFAULT_TTL_SECONDS=86400
cxEngine_MAX_RETRY_ATTEMPTS=3
cxEngine_RETRY_BASE_DELAY_MS=1000
cxEngine_RETRY_MAX_DELAY_MS=60000

# === Cache/State ===
STATE_CACHE_PROVIDER=redis|memory
STATE_CACHE_TTL_SECONDS=86400
STATE_CACHE_URL=redis://localhost:6379

# === Compliance ===
COMPLIANCE_DEFAULT_REGION=MX
COMPLIANCE_OPT_OUT_LIST_URL=https://.../optout
COMPLIANCE_ALERT_THRESHOLD_PER_HOUR=10

# === Metrics ===
METRICS_BACKEND=prometheus|datadog|stdout
METRICS_SAMPLE_RATE=1.0  # 1.0 = 100%, 0.1 = 10%
METRICS_FLUSH_INTERVAL_SECONDS=30

# === Tracing ===
TRACE_SINK=stdout|file|otlp
TRACE_RETENTION_DAYS=90
TRACE_PII_MASKING=true

# === Adapters (placeholders, implementados por deployment) ===
CHANNEL_ADAPTER_IMPL=twilio|meta_whatsapp|sendgrid
CRM_ADAPTER_IMPL=highlevel|salesforce|hubspot
LLM_PROVIDER_IMPL=openai|anthropic|gemini
```

---

## 8. Implementation Notes

### 8.1 Stack Reference (n8n como ejemplo)

| Componente | Implementación en n8n | Notas |
|------------|----------------------|-------|
| Lead Schema | Validación en nodo Code + JSON Schema | Usar biblioteca `ajv` si disponible |
| ConversationState | Redis node + TTL | Fallback a memoria si Redis no disponible |
| ChannelAdapter | HTTP Request + lógica de parsing | Wrapper alrededor de nodos de canal |
| CRMAdapter | HTTP Request + mapeo de campos | Configurar por partner en variables |
| LLM Prompts | AI Agent node + system prompt template | Placeholders reemplazados en runtime |
| ErrorPolicy | Function node con lógica de retry | Integrar con nodo de error handling de n8n |
| TraceEmitter | Code node + salida a archivo/HTTP | Usar `trace_id` de $execution.id como base |
| ComplianceGuardrails | Function node + lista de opt-out | Cache local para performance |
| MetricsEmitter | HTTP Request a endpoint de métricas | Batch para reducir llamadas |

### 8.2 Stack Alternativos (Agnóstico)

| Stack | Ventajas | Consideraciones |
|-------|----------|-----------------|
| **n8n** | Visual, rápido para prototipar | Menos control fino en lógica compleja |
| **Make.com** | Similar a n8n, buena UI | Pricing por operación puede escalar |
| **Zapier** | Muy fácil de usar | Menos flexible para lógica custom |
| **Node.js/Python custom** | Máximo control, performance | Requiere más desarrollo y ops |
| **Serverless (Lambda/CF)** | Escalado automático, pago por uso | Cold starts, debugging más complejo |

### 8.3 Estrategia de Testing

```
// === Unit Tests (por componente) ===
TEST ConversationStateManager:
    - get_state returns NULL for unknown lead_id
    - set_state with TTL expires after specified time
    - clear_state removes entry and returns correct boolean

TEST ChannelAdapter (mock):
    - parse_incoming normalizes WhatsApp payload to NormalizedMessage
    - send_message respects opt-out and returns blocked status
    - can_send enforces rate limits correctly

TEST ComplianceGuardrails:
    - pre_send_check blocks opted-out numbers
    - pre_send_check allows valid consent scenarios
    - post_interaction_audit logs and updates counters

// === Integration Tests (componentes juntos) ===
TEST Lead flow through shared components:
    - Input: raw WhatsApp message
    - Process: parse → classify → compliance check → CRM upsert
    - Output: Lead_v1 with updated status + trace emitted

// === Contract Tests (schemas) ===
TEST Lead Schema validation:
    - Valid lead passes validation
    - Missing required field fails with clear error
    - Extra fields are allowed (open schema for custom_fields)

// === Performance Tests ===
TEST Component latency under load:
    - 100 concurrent leads processed
    - P95 latency < 100ms per component
    - No memory leaks after 1000 iterations
```

---

## 9. Success Metrics (Técnicos)

| Métrica | Fórmula | Target | Frecuencia |
|---------|---------|--------|------------|
| Component reuse rate | shared_components_used / total_components | ≥ 80% | Por Android |
| Adapter swap time | time_to_replace_adapter | < 4 horas | Por cambio |
| Schema validation success | valid_payloads / total_payloads | 100% | Diario |
| Trace correlation rate | traces_with_full_context / total_traces | ≥ 95% | Diario |
| Error handling coverage | handled_errors / total_errors | ≥ 99% | Diario |
| Compliance block accuracy | correct_blocks / total_blocks | ≥ 99.9% | Semanal |

---

## 10. Deployment Checklist

### Pre-Deployment
- [ ] Todos los schemas validados contra JSON Schema Draft-07
- [ ] Interfaces implementadas con mocks para testing
- [ ] Environment variables documentadas y con defaults seguros
- [ ] Logging estructurado habilitado en todos los componentes
- [ ] Health checks implementados para cada adapter
- [ ] DLQ configurado para errores permanentes
- [ ] Métricas base emitidas y visibles en dashboard
- [ ] Compliance rules cargadas y testeadas por región
- [ ] Trace propagation verificada end-to-end
- [ ] Documentación de integración para desarrolladores de Androids

### Post-Deployment
- [ ] Monitorizar error rates por componente por 24h
- [ ] Validar que traces se correlacionan correctamente
- [ ] Confirmar que compliance blocks se aplican como esperado
- [ ] Medir latencia agregada vs. targets
- [ ] Revisar logs para detectar PII no masked
- [ ] Documentar cualquier desviación del spec

---

## 11. Version History

| Versión | Fecha | Cambios | Autor |
|---------|-------|---------|-------|
| 1.0 | 2026-02-20 | Especificación inicial de componentes compartidos | O.A. Perez Garrido |

---

## 12. Appendices

### Appendix A: Mapeo de Estados de Lead (Cross-Android)

| Estado cxEngine | Sleeping Beauty | Speed To Lead | Out Of Hours | Document Collection |
|-------------|----------------|---------------|--------------|-------------------|
| `new` | Lead cargado de CSV, no contactado | Lead fresco de formulario, no contactado | Lead fuera de horario, en cola | Lead solicitó docs, no enviado |
| `contacted` | Prince Charming Kiss enviado | Primer SMS de seguimiento enviado | Primer mensaje de nurturing enviado | Solicitud de docs enviada |
| `qualified` | Respondió + calificado para venta | Mostró interés alto + datos completos | Agendó para seguimiento en horario | Docs enviados + validados |
| `disqualified` | No respondió tras N intentos / no viable | No respondió / no interesado / datos inválidos | No respondió tras ventana / no viable | Docs inválidos / no completados |
| `converted` | Cita agendada + venta cerrada | Cita agendada + venta cerrada | Cita agendada + venta cerrada | Milestone completado + pago generado |
| `dead` | Opt-out explícito / bloqueo permanente | Opt-out / bloqueo por compliance | Opt-out / bloqueo por compliance | Opt-out / abandono definitivo |

### Appendix B: Placeholders para Prompt Templates

| Placeholder | Descripción | Ejemplo de valor |
|-------------|-------------|------------------|
| `{niche}` | Industria o vertical del socio | "clínicas dentales", "instaladores solares" |
| `{offer}` | Oferta principal que se promueve | "consulta gratuita", "cotización sin costo" |
| `{price_range}` | Rango de precios o inversión | "$500-$2000 MXN", "desde $15,000 USD" |
| `{objections_list}` | Objeciones comunes y respuestas | ["muy caro", "no tengo tiempo", "ya tengo proveedor"] |
| `{calendar_link}` | URL para agendar cita | "https://calendly.com/socio/30min" |
| `{brand_voice}` | Tono de comunicación | "profesional pero cercano", "energético y directo" |
| `{compliance_disclaimer}` | Texto legal requerido | "Msg&data rates may apply. Reply STOP to opt out." |

### Appendix C: Reglas de Compliance por Región (Base)

| Región | Regla | Descripción | Implementación |
|--------|-------|-------------|----------------|
| **MX** | HORARIO | Solo enviar 9am-9pm hora local | `outside_allowed_hours()` verifica timezone del lead |
| **MX** | FRECUENCIA | Máx. 3 mensajes/día sin respuesta | `exceeds_frequency_limit()` cuenta en cache |
| **US** | TCPA_CONSENT | Consentimiento explícito requerido | `has_tcpa_consent()` consulta tabla de consentimientos |
| **US** | OPT_OUT | "STOP" debe procesarse en <1 minuto | Webhook de opt-out actualiza lista en tiempo real |
| **EU** | GDPR_BASIS | Base legal para procesamiento | `has_gdpr_basis()` verifica consentimiento o legítimo interés |
| **EU** | DATA_MINIMIZATION | Solo datos necesarios | Masking automático en logs, retención limitada |
| **GLOBAL** | OPT_OUT_UNIVERSAL | Opt-out aplica a todos los Androids | Lista centralizada consultada por todos los adapters |

### Appendix D: Guía de Integración para Nuevos Androids

```
// === Pasos para crear un nuevo Android usando Shared Components ===

1. IMPORTAR DEPENDENCIAS:
   - Lead Schema v1
   - ChannelAdapter interface
   - CRMAdapter interface
   - ComplianceGuardrails hooks
   - TraceEmitter para observabilidad

2. DEFINIR LÓGICA ESPECÍFICA:
   - ¿Qué desencadena este Android? (CSV, webhook, schedule)
   - ¿Qué decisiones toma? (clasificación, calificación, routing)
   - ¿Qué acciones ejecuta? (enviar mensaje, actualizar CRM, agendar)

3. CONFIGURAR PROMPTS:
   - Seleccionar plantilla base de LLM Prompt Templates
   - Personalizar placeholders para el nicho
   - Validar response schema con el LLM provider

4. IMPLEMENTAR FLUJO:
   - Ingesta → Normalización → Decisión → Acción → Logging
   - Usar TraceEmitter para correlacionar eventos
   - Usar ErrorPolicy para manejo robusto de fallos

5. VALIDAR COMPLIANCE:
   - Integrar pre_send_check antes de cualquier envío
   - Configurar región para reglas aplicables
   - Testear escenarios de bloqueo y fallback

6. EMITIR MÉTRICAS:
   - Usar MetricsEmitter para KPIs de negocio
   - Incluir tags: android, channel, region, status
   - Configurar dashboard para monitoreo

7. DOCUMENTAR:
   - Inputs/Outputs esperados
   - Configuración requerida (env vars, secrets)
   - Casos de prueba representativos
```

---

## ✅ Aprobaciones

| Rol | Nombre | Fecha | Firma |
|-----|--------|-------|-------|
| Product Owner | | | |
| Tech Lead | | | |
| QA Lead | | | |
| Compliance Officer | | | |

---

**Documento generado:** 20 de febrero de 2026  
**Próxima revisión:** 20 de marzo de 2026  
**Status:** Ready for Implementation

---

> **Nota final para el equipo:**  
> Este SPEC define la "biblioteca estándar" de cxEngine.  
> Cada Android futuro debe construirse **sobre** estos componentes, no **al lado** de ellos.  
> La consistencia que logremos aquí es lo que permitirá escalar de 1 a 10 Androids sin multiplicar la complejidad.  
> Si algo no encaja en este spec, primero pregunta: "¿Debería ser compartido?" antes de crear algo nuevo.