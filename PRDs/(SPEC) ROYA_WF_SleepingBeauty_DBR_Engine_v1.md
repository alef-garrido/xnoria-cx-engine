# 📄 Product Requirements Document (PRD) Técnico

## WF_SleepingBeauty_DBR_Engine_v1
**Versión:** 1.0  
**Fecha:** 20 de febrero de 2026  
**Estado:** Especificación para implementación  
**Dominio:** ROYA / Sleeping Beauty Android (DBR)  
**Stack referencia:** Agnóstico (n8n/Make/Zapier + LLM + SMS/WhatsApp + CRM)

---

## 1. Overview Técnico

### 1.1 Propósito del sistema
Motor de reactivación de bases de datos inactivas (DBR) que transforma leads "muertos" en oportunidades calificadas mediante comunicación automatizada, inteligente y compliant, siguiendo el patrón "Prince Charming Kiss" → Conversación → Calificación → Cita.

### 1.2 Arquitectura de alto nivel
```
CSV Leads Inactivos → Ingestion → Prince Charming Kiss → Conversation Loop → Qualification → Appointment → CRM Sync
                              ↓
                    [ROYA Shared Components]
                    - Lead Schema v1
                    - ChannelAdapter
                    - ComplianceGuardrails
                    - TraceEmitter
                    - MetricsEmitter
```

### 1.3 Principios técnicos fundamentales
- **Reactivación, no spam:** Cada mensaje tiene propósito de re-engagement, no broadcast
- **Conversación, no monólogo:** El sistema espera y procesa respuestas antes de continuar
- **Calificación estructurada:** Lead scoring basado en señales explícitas, no intuición
- **Compliance-first:** TCPA/GDPR/opt-out enforcement antes de cada envío
- **Auditabilidad completa:** Toda interacción deja traza correlacionable

---

## 2. Problem Statement (Técnico)

### 2.1 Problema actual
Las bases de datos de leads inactivos representan valor no realizado debido a:
- **Fricción manual:** Reactivación requiere tiempo humano por lead
- **Inconsistencia:** Mensajes ad-hoc sin estrategia conversacional
- **Riesgo compliance:** Envíos masivos sin validación de consentimiento
- **Pérdida de señal:** Respuestas de leads no se capturan ni procesan sistemáticamente

### 2.2 Impacto técnico
- ROI negativo en campañas de reactivación manual
- Imposibilidad de escalar más allá de cientos de leads
- Exposición legal por envíos no compliant
- Datos de respuesta no estructurados → no accionables

---

## 3. Technical Objectives

### 3.1 Objetivos SMART

| ID | Objetivo | Métrica | Target |
|----|----------|---------|--------|
| **TO-1** | Reactivar leads inactivos con tasa de respuesta medible | Response rate (reply / sent) | ≥ 15% |
| **TO-2** | Calificar leads reactivados con criterio estructurado | Qualification rate (qualified / replied) | ≥ 40% |
| **TO-3** | Agendar citas sin intervención humana para leads calificados | Appointment conversion (appt / qualified) | ≥ 60% |
| **TO-4** | Mantener compliance 100% en todos los envíos | Compliance adherence rate | 100% |
| **TO-5** | Procesar 1000 leads/hora sin degradación | Throughput sustained | ≥ 1000 leads/hr |

### 3.2 Requerimientos de calidad (NFRs)

| NFR | Descripción | Target |
|-----|-------------|--------|
| **NFR-1** | Disponibilidad del flujo de reactivación | 99.5% uptime |
| **NFR-2** | Latencia por interacción conversacional | < 30 segundos (reply → next message) |
| **NFR-3** | Tolerancia a fallos de canal (SMS/WhatsApp) | Retry + fallback + DLQ |
| **NFR-4** | Retención de trazas para auditoría | 90 días mínimo |
| **NFR-5** | Escalabilidad horizontal por volumen de leads | Linear scaling con shards |

---

## 4. Scope Técnico

### 4.1 In Scope

| Componente | Descripción | Responsabilidad |
|------------|-------------|-----------------|
| **Lead Ingestion** | Carga y validación de CSV con leads inactivos | Transformación |
| **Prince Charming Kiss** | Mensaje inicial de 2 frases diseñado para provocar respuesta | Comunicación |
| **Conversation Loop** | Manejo de réplicas del lead con lógica de calificación progresiva | Decisión |
| **Qualification Engine** | Evaluación estructurada de interés, viabilidad y timing | Clasificación |
| **Appointment Scheduler** | Agendamiento automático en calendario del socio | Integración |
| **CRM Sync** | Actualización de estado del lead en CRM externo | Sincronización |
| **Compliance Enforcement** | Validación TCPA/GDPR/opt-out antes de cada envío | Seguridad legal |
| **Metrics & Logging** | Emisión de métricas de reactivación y trazas auditables | Observabilidad |

### 4.2 Out of Scope

| Componente | Razón de exclusión |
|------------|-------------------|
| Generación de contenido creativo para campañas | Fuera de scope de automatización |
| Gestión de listas de opt-out centralizadas | Responsabilidad del socio / shared component |
| Integración con múltiples CRMs simultáneos | V1: un CRM por instancia |
| A/B testing de mensajes iniciales | Optimización post-v1 |
| Soporte para canales voice/llamada | Solo SMS/WhatsApp en v1 |
| Machine Learning para scoring predictivo | Reglas explícitas en v1 |

---

## 5. Functional Requirements

### 5.1 FR-1: Lead Ingestion & Validation

**ID:** FR-1  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe cargar, validar y normalizar leads desde CSV para procesamiento.

**User Story:**  
Como operador, quiero cargar un archivo CSV con leads inactivos y que el sistema valide, deduplique y prepare cada lead para reactivación sin intervención manual.

**Acceptance Criteria:**
- [ ] Soporte para CSV con columnas: `phone`, `name`, `last_contact_date`, `source`, `custom_fields`
- [ ] Validación de formato phone (E.164) y rechazo de inválidos
- [ ] Deduplicación por `phone` dentro del mismo batch y contra CRM existente
- [ ] Normalización a `Lead_v1` schema de ROYA Shared Components
- [ ] Asignación de `status: "new"` y `source: "db_reactivation"`
- [ ] Logging de leads rechazados con razón clara
- [ ] Throughput: ≥ 1000 leads/minuto en ingestión

**Pseudocódigo de validación:**
```
FUNCTION ingest_leads(csv_file: File) → ARRAY<Lead_v1>:
    valid_leads = []
    rejected = []
    
    FOR EACH row IN csv_file:
        // Validación básica
        IF NOT is_valid_e164(row.phone):
            rejected.append({phone: row.phone, reason: "invalid_format"})
            CONTINUE
        
        // Deduplicación intra-batch
        IF row.phone IN valid_leads.phones:
            rejected.append({phone: row.phone, reason: "duplicate_in_batch"})
            CONTINUE
        
        // Verificación contra CRM (opcional, configurable)
        IF CONFIG.check_crm_exists AND crm_adapter.lead_exists(row.phone):
            rejected.append({phone: row.phone, reason: "already_in_crm"})
            CONTINUE
        
        // Normalización a Lead_v1
        lead = {
            lead_id: GENERATE_UUID(),
            phone: row.phone,
            name: row.name OR NULL,
            source: "db_reactivation",
            status: "new",
            created_at: NOW(),
            schema_version: "v1",
            metadata: {
                last_contact_date: row.last_contact_date,
                original_source: row.source,
                custom_fields: row.custom_fields OR {}
            }
        }
        valid_leads.append(lead)
    
    LOG: "ingestion_complete", total: csv_file.rows, valid: valid_leads.count, rejected: rejected.count
    RETURN valid_leads
```

---

### 5.2 FR-2: Prince Charming Kiss Delivery

**ID:** FR-2  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe enviar el mensaje inicial de reactivación ("Prince Charming Kiss") diseñado para maximizar respuesta.

**User Story:**  
Como sistema de reactivación, quiero enviar un mensaje inicial breve y personalizado que provoque respuesta del lead inactivo, para iniciar la conversación de calificación.

**Acceptance Criteria:**
- [ ] Mensaje máximo 2 frases, ≤ 160 caracteres para SMS
- [ ] Personalización mínima: `{name}` o genérico si no disponible
- [ ] Template configurable por nicho: `{niche_kiss_template}`
- [ ] Envío vía ChannelAdapter (SMS/WhatsApp) con compliance check previo
- [ ] Actualización de lead a `status: "contacted"` tras envío exitoso
- [ ] Registro de timestamp y provider_message_id para trazabilidad
- [ ] Manejo de fallos de envío con retry policy de Shared Components

**Pseudocódigo de envío:**
```
FUNCTION send_prince_charming_kiss(lead: Lead_v1, template: OBJECT) → DeliveryResult:
    // Compliance check obligatorio antes de enviar
    compliance = compliance_guardrails.pre_send_check(
        lead = lead,
        content = {type: "text", body: "template_preview"},
        context = {campaign: "sleeping_beauty", region: CONFIG.region}
    )
    
    IF compliance.result == "blocked":
        LOG: "kiss_blocked", lead_id: lead.lead_id, reason: compliance.reason
        UPDATE lead.status = "disqualified"
        RETURN {status: "blocked", reason: compliance.reason}
    
    // Construcción del mensaje
    name = lead.name OR "amigo"
    message_body = template.text
        .replace("{name}", name)
        .replace("{niche}", template.niche_context)
        .trim()
    
    // Validación de longitud para SMS
    IF CONFIG.channel == "sms" AND message_body.length > 160:
        message_body = message_body.substring(0, 157) + "..."
    
    // Envío vía adapter
    delivery = channel_adapter.send_message(
        lead = lead,
        content = {
            type: "text",
            body: message_body
        }
    )
    
    // Actualización de estado
    IF delivery.status == "sent" OR delivery.status == "queued":
        UPDATE lead.status = "contacted"
        UPDATE lead.metadata.last_kiss_sent_at = NOW()
        UPDATE lead.metadata.kiss_message_id = delivery.provider_message_id
        EMIT_METRIC: "kiss_sent", tags: {niche: template.niche, channel: CONFIG.channel}
    
    LOG: "kiss_delivery", lead_id: lead.lead_id, status: delivery.status
    RETURN delivery
```

**Template base (ejemplo nicho clínico):**
```
{
  "niche": "clinicas",
  "text": "Hola {name}, somos [Clínica]. ¿Te gustaría recibir información sobre nuestros servicios? Responde SÍ para continuar.",
  "expected_reply_keywords": ["sí", "si", "yes", "información", "info"],
  "negative_keywords": ["no", "stop", "basta", "eliminar"]
}
```

---

### 5.3 FR-3: Conversation Loop & Reply Handling

**ID:** FR-3  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe capturar, normalizar y procesar respuestas del lead para continuar la conversación de calificación.

**User Story:**  
Como motor conversacional, quiero recibir y entender las respuestas del lead para avanzar en la calificación sin intervención humana, manteniendo contexto y propósito.

**Acceptance Criteria:**
- [ ] Webhook/listener para capturar replies de SMS/WhatsApp
- [ ] Normalización de reply a `NormalizedMessage` de Shared Components
- [ ] Correlación reply → lead original vía `phone` o `thread_id`
- [ ] Detección de intención básica: `positive`, `negative`, `question`, `other`
- [ ] Actualización de `ConversationState` con último intercambio
- [ ] Trigger de siguiente paso: calificación, más preguntas, o escalación
- [ ] Timeout de conversación: si no hay reply en 48h → marcar como `dead`

**Pseudocódigo de manejo de reply:**
```
FUNCTION handle_incoming_reply(raw_payload: OBJECT) → ProcessingResult:
    // Normalización
    message = channel_adapter.parse_incoming(raw_payload)
    IF message == NULL:
        RETURN {status: "parse_failed", reason: "invalid_payload"}
    
    // Correlación con lead
    lead = crm_adapter.get_lead_by_phone(message.from)
    IF lead == NULL:
        LOG: "orphan_reply", phone: message.from
        RETURN {status: "orphan", reason: "lead_not_found"}
    
    // Verificación de estado válido para conversación
    IF lead.status NOT IN ["contacted", "qualifying"]:
        LOG: "unexpected_reply", lead_id: lead.lead_id, status: lead.status
        RETURN {status: "ignored", reason: "lead_not_in_conversation_state"}
    
    // Actualización de estado conversacional
    convo_state = conversation_state_manager.get_state(lead.lead_id) OR {stage: "initial"}
    convo_state.last_reply_at = message.timestamp
    convo_state.reply_count = (convo_state.reply_count OR 0) + 1
    conversation_state_manager.set_state(lead.lead_id, convo_state, ttl: 48_HOURS)
    
    // Clasificación básica de intención de reply
    intent = classify_reply_intent(message.body, template_keywords)
    // classify_reply_intent returns: "positive" | "negative" | "question" | "other"
    
    // Routing según intención
    SWITCH intent:
        CASE "positive":
            RETURN trigger_qualification_flow(lead, message, convo_state)
        CASE "negative":
            RETURN handle_negative_reply(lead, message)
        CASE "question":
            RETURN handle_question_reply(lead, message, convo_state)
        CASE "other":
            RETURN handle_ambiguous_reply(lead, message, convo_state)
    
    LOG: "reply_processed", lead_id: lead.lead_id, intent: intent
    RETURN {status: "processed", next_action: determined_action}
```

---

### 5.4 FR-4: Qualification Engine (Structured Scoring)

**ID:** FR-4  
**Prioridad:** ALTA  
**Descripción:** El sistema debe evaluar respuestas del lead mediante criterios estructurados para determinar viabilidad de venta.

**User Story:**  
Como sistema de calificación, quiero evaluar señales explícitas del lead (interés, timing, presupuesto, necesidad) para clasificarlo como qualified/unqualified sin subjetividad humana.

**Acceptance Criteria:**
- [ ] Criterios de calificación configurables por nicho: `qualification_rules`
- [ ] Evaluación de señales: interés explícito, timing, presupuesto, autoridad
- [ ] Puntuación numérica 0-100 con thresholds: `qualified ≥ 70`, `unqualified ≤ 30`, `needs_more_info: 31-69`
- [ ] Preguntas de calificación secuenciales (máx. 3) para obtener datos faltantes
- [ ] Justificación estructurada de la decisión: `reason_codes[]`
- [ ] Actualización de `lead.status` y emisión de métricas de calificación

**Pseudocódigo de calificación:**
```
FUNCTION qualify_lead(lead: Lead_v1, conversation_history: ARRAY<Message>) → QualificationResult:
    // Cargar reglas de calificación por nicho
    rules = load_qualification_rules(lead.metadata.niche)
    
    // Extraer señales de la conversación
    signals = extract_qualification_signals(conversation_history, rules.signal_keywords)
    // signals: {interest: bool, timing: enum, budget: enum, authority: bool, need: bool}
    
    // Calcular score
    score = 0
    IF signals.interest: score += 30
    IF signals.timing == "immediate": score += 25
    IF signals.timing == "soon": score += 15
    IF signals.budget == "confirmed": score += 20
    IF signals.budget == "estimated": score += 10
    IF signals.authority: score += 15
    IF signals.need: score += 10
    
    // Determinar resultado
    IF score >= rules.thresholds.qualified:
        result = "qualified"
        reason_codes = ["high_interest", "good_timing", "budget_confirmed"] // según señales
    ELSE IF score <= rules.thresholds.unqualified:
        result = "unqualified"
        reason_codes = ["low_interest", "no_timing", "no_budget"]
    ELSE:
        result = "needs_more_info"
        reason_codes = ["insufficient_data"]
        missing_questions = generate_followup_questions(signals, rules.question_bank)
    
    // Actualizar lead
    lead.status = result
    lead.metadata.qualification_score = score
    lead.metadata.qualification_reasons = reason_codes
    lead.metadata.qualified_at = result != "needs_more_info" ? NOW() : NULL
    
    // Emitir métrica
    EMIT_METRIC: "lead_qualified", value: 1, tags: {result: result, niche: lead.metadata.niche, score_bucket: score_bucket(score)}
    
    LOG: "qualification_complete", lead_id: lead.lead_id, score: score, result: result
    RETURN {
        result: result,
        score: score,
        reason_codes: reason_codes,
        missing_questions: missing_questions OR []
    }
```

---

### 5.5 FR-5: Appointment Scheduling & CRM Sync

**ID:** FR-5  
**Prioridad:** ALTA  
**Descripción:** El sistema debe agendar citas automáticamente para leads calificados y sincronizar el estado con el CRM del socio.

**User Story:**  
Como sistema de cierre, quiero agendar una cita en el calendario del equipo de ventas y actualizar el CRM cuando un lead está calificado, para eliminar fricción manual y acelerar el handoff.

**Acceptance Criteria:**
- [ ] Integración con calendario del socio (Google Calendar, Calendly, CRM-native)
- [ ] Selección de slot disponible basado en configuración: `duration`, `buffer`, `business_hours`
- [ ] Generación de confirmación para el lead con detalles de la cita
- [ ] Actualización de `lead.status = "converted"` y campos de cita en CRM
- [ ] Notificación al equipo de ventas (email/Slack) con resumen del lead
- [ ] Manejo de fallos de agendamiento con fallback a "human scheduling queue"
- [ ] Trazabilidad completa: appointment_id correlacionado con trace_id

**Pseudocódigo de agendamiento:**
```
FUNCTION schedule_appointment_for_qualified_lead(lead: Lead_v1, qualification: QualificationResult) → ScheduleResult:
    // Obtener configuración de calendario del socio
    calendar_config = load_calendar_config(lead.metadata.partner_id)
    
    // Buscar slot disponible
    slot = calendar_adapter.find_available_slot(
        duration_minutes: calendar_config.default_duration,
        buffer_minutes: calendar_config.buffer,
        business_hours: calendar_config.hours,
        timezone: lead.metadata.timezone OR CONFIG.default_timezone,
        preferred_window: "next_3_business_days"
    )
    
    IF slot == NULL:
        // Fallback: enqueue para scheduling humano
        enqueue_for_human_scheduling(lead, qualification)
        LOG: "no_slot_available", lead_id: lead.lead_id
        RETURN {status: "queued_for_human", reason: "no_available_slot"}
    
    // Crear evento en calendario
    appointment = calendar_adapter.create_event(
        title: "Cita calificada - {lead.name}",
        start_time: slot.start,
        duration_minutes: calendar_config.default_duration,
        attendees: [lead.phone, calendar_config.sales_team_email],
        description: build_appointment_description(lead, qualification),
        metadata: {
            lead_id: lead.lead_id,
            qualification_score: qualification.score,
            source: "sleeping_beauty_android"
        }
    )
    
    // Confirmación al lead
    confirmation_message = build_confirmation_message(appointment, lead.name)
    channel_adapter.send_message(lead, {type: "text", body: confirmation_message})
    
    // Actualizar CRM
    crm_update = {
        status: "converted",
        appointment_id: appointment.id,
        appointment_time: appointment.start,
        qualification_score: qualification.score,
        last_interaction: NOW()
    }
    crm_adapter.update_lead(lead.lead_id, crm_update)
    
    // Notificar equipo de ventas
    IF calendar_config.notify_sales_team:
        notify_sales_team({
            lead: lead,
            appointment: appointment,
            qualification: qualification
        })
    
    // Emitir métrica de conversión
    EMIT_METRIC: "appointment_scheduled", value: 1, tags: {niche: lead.metadata.niche, channel: CONFIG.channel}
    
    LOG: "appointment_scheduled", lead_id: lead.lead_id, appointment_id: appointment.id
    RETURN {status: "scheduled", appointment_id: appointment.id, confirmation_sent: true}
```

---

### 5.6 FR-7: Compliance & Opt-Out Handling

**ID:** FR-7  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe respetar automáticamente opt-outs y regulaciones (TCPA/GDPR) en cada interacción.

**User Story:**  
Como operador en región regulada, quiero que el sistema detecte y procese solicitudes de opt-out inmediatamente, y bloquee futuros envíos a ese número, para cumplir con la ley y proteger la reputación.

**Acceptance Criteria:**
- [ ] Detección de keywords de opt-out: "STOP", "BAJA", "UNSUBSCRIBE", etc. (configurable por idioma)
- [ ] Procesamiento de opt-out en < 1 minuto desde recepción del mensaje
- [ ] Actualización de lista de opt-out centralizada (shared component)
- [ ] Bloqueo inmediato de futuros envíos a números opt-out
- [ ] Confirmación de opt-out al usuario (mensaje estándar)
- [ ] Logging de todos los eventos de opt-out para auditoría
- [ ] Soporte para re-opt-in explícito (vía proceso separado, no automático)

**Pseudocódigo de manejo de opt-out:**
```
FUNCTION process_opt_out_request(message: NormalizedMessage, lead: Lead_v1) → OptOutResult:
    // Verificar si ya está en opt-out
    IF compliance_guardrails.is_opted_out(lead.phone):
        // Ya está opt-out, ignorar pero loguear
        LOG: "optout_duplicate", phone: lead.phone
        RETURN {status: "already_opted_out"}
    
    // Procesar opt-out
    compliance_guardrails.add_to_optout_list(lead.phone, {
        timestamp: message.timestamp,
        channel: message.channel,
        campaign: "sleeping_beauty",
        reason: "user_requested"
    })
    
    // Actualizar lead
    lead.status = "disqualified"
    lead.metadata.disqualified_reason = "opt_out_requested"
    lead.metadata.opted_out_at = message.timestamp
    crm_adapter.update_lead(lead.lead_id, {
        status: "disqualified",
        opt_out: true,
        opt_out_date: message.timestamp
    })
    
    // Confirmación al usuario (obligatorio por TCPA)
    confirmation = CONFIG.optout_confirmation_message[message.channel] OR "Has sido dado de baja. No recibirás más mensajes."
    channel_adapter.send_message(lead, {type: "text", body: confirmation})
    
    // Emitir métrica de compliance
    EMIT_METRIC: "opt_out_processed", value: 1, tags: {channel: message.channel, region: CONFIG.region}
    
    LOG: "opt_out_processed", lead_id: lead.lead_id, phone: lead.phone
    RETURN {status: "opted_out", confirmation_sent: true}
```

---

### 5.7 FR-8: Metrics Emission & Observability

**ID:** FR-8  
**Prioridad:** ALTA  
**Descripción:** El sistema debe emitir métricas estructuradas para medir performance de reactivación y permitir optimización.

**User Story:**  
Como analista, quiero métricas en tiempo real de respuesta, calificación y conversión por campaña, para identificar qué funciona y ajustar la estrategia.

**Acceptance Criteria:**
- [ ] Métricas de funnel: `kiss_sent`, `reply_received`, `lead_qualified`, `appointment_scheduled`
- [ ] Métricas de calidad: `response_rate`, `qualification_rate`, `conversion_rate`, `opt_out_rate`
- [ ] Segmentación por: `niche`, `channel`, `region`, `batch_id`, `template_version`
- [ ] Emisión asíncrona vía MetricsEmitter de Shared Components
- [ ] Correlación con trace_id para debugging end-to-end
- [ ] Dashboard-ready: formato compatible con Prometheus/Datadog

**Pseudocódigo de emisión de métricas:**
```
FUNCTION emit_dbr_metrics(lead: Lead_v1, event: ENUM, metadata: OBJECT):
    base_tags = {
        android: "sleeping_beauty",
        niche: lead.metadata.niche,
        channel: CONFIG.channel,
        region: CONFIG.region,
        batch_id: lead.metadata.batch_id
    }
    
    SWITCH event:
        CASE "kiss_sent":
            metrics_emitter.emit_counter("dbr_kiss_sent", 1, base_tags)
            
        CASE "reply_received":
            metrics_emitter.emit_counter("dbr_reply_received", 1, base_tags)
            // Calcular response rate si tenemos kiss_sent del mismo batch
            response_rate = calculate_response_rate(lead.metadata.batch_id)
            metrics_emitter.emit_gauge("dbr_response_rate", response_rate, base_tags)
            
        CASE "lead_qualified":
            metrics_emitter.emit_counter("dbr_lead_qualified", 1, {
                ...base_tags,
                qualification_result: metadata.result,
                score_bucket: score_bucket(metadata.score)
            })
            
        CASE "appointment_scheduled":
            metrics_emitter.emit_counter("dbr_appointment_scheduled", 1, base_tags)
            // Calcular conversión final
            conversion_rate = calculate_conversion_rate(lead.metadata.batch_id)
            metrics_emitter.emit_gauge("dbr_conversion_rate", conversion_rate, base_tags)
            
        CASE "opt_out_processed":
            metrics_emitter.emit_counter("dbr_opt_out", 1, base_tags)
```

---

## 6. Non-Functional Requirements

### 6.1 Performance

| Métrica | Target | Medición |
|---------|--------|----------|
| Latencia kiss → reply processing | < 30 segundos | P95 |
| Throughput de ingestión | ≥ 1000 leads/minuto | Sustained load |
| Tiempo de calificación | < 5 segundos por interacción | P95 |
| Agendamiento de cita | < 10 segundos desde calificación | P95 |

### 6.2 Reliability

| Métrica | Target | Estrategia |
|---------|--------|------------|
| Delivery rate de kisses | ≥ 98% | Retry + fallback channel |
| Reply capture rate | ≥ 99% | Webhook redundancy + polling fallback |
| CRM sync success | 100% | Idempotent updates + DLQ |
| Compliance enforcement | 100% | Pre-send check obligatorio, fail-safe |

### 6.3 Security

| Requisito | Implementación |
|-----------|----------------|
| PII en logs | Masking automático: phone → `+52***`, name → inicial |
| Credenciales de adapters | Secrets management, nunca en código |
| Webhook verification | HMAC signature para replies de WhatsApp |
| Opt-out list integrity | Write-once, append-only, audit trail |

### 6.4 Scalability

| Dimensión | Estrategia |
|-----------|------------|
| Horizontal | Stateless processing, queue-based ingestion |
| Multi-niche | Configuración por `niche_id`, reglas aisladas |
| Multi-partner | Aislamiento por `partner_id` en metadata |
| Multi-region | Compliance config por región, timezone-aware |

---

## 7. Technical Specifications

### 7.1 Contratos Extendidos (sobre ROYA Shared Components)

#### QualificationResult Schema
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "QualificationResult",
  "type": "object",
  "required": ["result", "score", "reason_codes"],
  "properties": {
    "result": {"enum": ["qualified", "unqualified", "needs_more_info"]},
    "score": {"type": "integer", "minimum": 0, "maximum": 100},
    "reason_codes": {"type": "array", "items": {"type": "string"}},
    "missing_questions": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "question": {"type": "string"},
          "field": {"type": "string"},
          "required": {"type": "boolean"}
        }
      }
    }
  }
}
```

#### AppointmentDetails Schema
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "AppointmentDetails",
  "type": "object",
  "required": ["appointment_id", "start_time", "duration_minutes"],
  "properties": {
    "appointment_id": {"type": "string"},
    "start_time": {"type": "string", "format": "date-time"},
    "duration_minutes": {"type": "integer"},
    "timezone": {"type": "string"},
    "meeting_link": {"type": "string"},
    "notes": {"type": "string"}
  }
}
```

### 7.2 Configuración por Nicho (Ejemplo: Clínicas)

```yaml
# config/niches/clinicas_v1.yaml
niche_id: clinicas
channel: sms  # o whatsapp
language: es-MX

prince_charming_kiss:
  template: "Hola {name}, somos [Clínica]. ¿Te gustaría recibir información sobre nuestros servicios? Responde SÍ para continuar."
  positive_keywords: ["sí", "si", "yes", "información", "info", "servicios"]
  negative_keywords: ["no", "stop", "baja", "eliminar", "basta"]

qualification_rules:
  signal_keywords:
    interest: ["quiero", "necesito", "me interesa", "cita", "consulta"]
    timing_immediate: ["hoy", "ahora", "urgente", "lo antes posible"]
    timing_soon: ["esta semana", "pronto", "cuándo puedo"]
    budget_confirmed: ["presupuesto", "precio", "costo", "cuánto"]
    authority: ["yo decido", "soy el paciente", "mi familia"]
  
  thresholds:
    qualified: 70
    unqualified: 30
  
  question_bank:
    - field: "specialty_interest"
      question: "¿Qué tipo de consulta te interesa: general, especializada o prevención?"
    - field: "preferred_timing"
      question: "¿Prefieres cita en la mañana, tarde o fin de semana?"
    - field: "insurance"
      question: "¿Cuentas con seguro médico o pagarías particular?"

calendar_config:
  default_duration: 30
  buffer: 15
  business_hours:
    monday_friday: "09:00-19:00"
    saturday: "09:00-14:00"
    sunday: "closed"
  timezone: "America/Mexico_City"
  sales_team_email: "ventas@clinica.example.com"
  notify_sales_team: true

compliance:
  region: MX
  optout_keywords: ["STOP", "BAJA", "CANCELAR", "NO MÁS"]
  optout_confirmation: "Has sido dado de baja. Para reactivar, envía ALTA a este número."
  max_messages_per_day: 1
  allowed_hours: "09:00-20:00"
```

### 7.3 Flujo de Estado del Lead (State Machine)

```
[NEW] 
   ↓ (ingestion)
[NEW] → send_kiss() → [CONTACTED]
   ↓ (reply received)
[CONTACTED] → classify_reply()
   ├─ positive → start_qualification() → [QUALIFYING]
   ├─ negative → handle_negative() → [DISQUALIFIED]
   └─ question/other → ask_clarifying() → [QUALIFYING]

[QUALIFYING] → evaluate_signals()
   ├─ score ≥ 70 → schedule_appointment() → [CONVERTED]
   ├─ score ≤ 30 → disqualify() → [DISQUALIFIED]
   └─ 31-69 → ask_more_questions() → [QUALIFYING] (max 3 rounds)

[CONVERTED] → sync_crm() + notify_sales() → [CLOSED]

[DISQUALIFIED] → log_reason() → [CLOSED]

Timeouts:
- CONTACTED → no reply in 48h → [DEAD]
- QUALIFYING → no reply in 24h → [DEAD]
```

---

## 8. Implementation Notes

### 8.1 Stack Reference (n8n como ejemplo)

| Componente | Nodo n8n | Configuración |
|------------|----------|---------------|
| CSV Ingestion | Read Binary File + Code | Parse CSV, validate, dedupe |
| Kiss Delivery | HTTP Request (Twilio/Meta) + Compliance Check | Rate limiting, retry |
| Reply Listener | Webhook (WhatsApp) / Polling (SMS) | HMAC verify, parse |
| Qualification | AI Agent (LLM) + Rules Engine | Structured output, scoring |
| Scheduling | HTTP Request (Calendly/Google Calendar) | Timezone handling |
| CRM Sync | HTTP Request (HighLevel/Salesforce) | Idempotent upsert |
| Metrics | HTTP Request (Prometheus push) | Async, batched |
| Logging | Write to File / HTTP | Structured JSON, trace_id |

### 8.2 Alternative Stacks

| Stack | Ventajas | Consideraciones |
|-------|----------|-----------------|
| **n8n + Twilio** | Visual, rápido para prototipar | Twilio pricing por mensaje |
| **Make.com + Meta WhatsApp** | Buena UI, WhatsApp nativo | Menos flexible para lógica custom |
| **Custom Node.js** | Máximo control, performance | Requiere más desarrollo y ops |
| **Serverless (Lambda)** | Escalado automático | Cold starts, debugging complejo |

### 8.3 Environment Variables

```bash
# DBR Engine Config
DBR_DEFAULT_NICHE=clinicas
DBR_MAX_CONVERSATION_ROUNDS=3
DBR_REPLY_TIMEOUT_HOURS=48
DBR_QUALIFICATION_TIMEOUT_HOURS=24

# Channel Config
CHANNEL_PROVIDER=twilio|meta_whatsapp
CHANNEL_PHONE_NUMBER=+52...
CHANNEL_API_KEY=xxx

# Calendar Config
CALENDAR_PROVIDER=google|calendly|crm_native
CALENDAR_API_KEY=xxx
CALENDAR_SALES_TEAM_EMAIL=xxx

# CRM Config
CRM_PROVIDER=highlevel|salesforce|hubspot
CRM_API_KEY=xxx
CRM_WEBHOOK_SECRET=xxx

# Compliance
COMPLIANCE_REGION=MX|US|EU
COMPLIANCE_OPTOUT_LIST_URL=https://.../optout
COMPLIANCE_MAX_MESSAGES_PER_DAY=1

# Metrics
METRICS_BACKEND=prometheus|datadog
METRICS_BATCH_SIZE=100
METRICS_FLUSH_INTERVAL_SECONDS=30
```

---

## 9. Testing Strategy

### 9.1 Test Cases

| ID | Descripción | Input | Expected Output |
|----|-------------|-------|-----------------|
| **TC-SB-01** | Ingestión de CSV válido | CSV con 100 leads válidos | 100 leads en estado `new` |
| **TC-SB-02** | Kiss delivery exitoso | Lead en `new` | Lead en `contacted`, mensaje enviado |
| **TC-SB-03** | Reply positiva → calificación | Reply: "Sí, quiero info" | Lead en `qualifying`, pregunta de seguimiento |
| **TC-SB-04** | Calificación con score alto | Signals: interest+timing+budget | Lead en `converted`, cita agendada |
| **TC-SB-05** | Opt-out processing | Reply: "STOP" | Lead en `disqualified`, opt-out registrado |
| **TC-SB-06** | Timeout sin reply | Lead en `contacted`, 48h sin reply | Lead en `dead` |
| **TC-SB-07** | Compliance block pre-send | Lead en opt-out list | Kiss no enviado, log de bloqueo |
| **TC-SB-08** | CRM sync idempotente | Mismo lead actualizado 3 veces | CRM con última versión, sin duplicados |
| **TC-SB-09** | Métricas de funnel | Batch de 100 leads procesados | Métricas: sent=100, replied=15, qualified=6, appt=4 |
| **TC-SB-10** | Fallback de agendamiento | Sin slots disponibles | Lead enqueued para scheduling humano |

### 9.2 Test Execution

```bash
# Unit tests (componentes individuales)
npm test -- unit/ingestion/
npm test -- unit/qualification/
npm test -- unit/compliance/

# Integration tests (flujos end-to-end)
npm test -- integration/kiss-to-reply/
npm test -- integration/qualification-to-appointment/

# Compliance tests (regulaciones)
npm test -- compliance/optout-processing/
npm test -- compliance/tcpa-limits/

# Load tests (escalabilidad)
npm test -- load/ingestion-1000-leads/
npm test -- load/concurrent-conversations-100/

# Cross-niche tests (configuración)
npm test -- config/niche-clinicas/
npm test -- config/niche-travel/
```

---

## 10. Success Metrics (Técnicos y de Negocio)

| Métrica | Fórmula | Target | Frecuencia |
|---------|---------|--------|------------|
| Response rate | replies / kisses_sent | ≥ 15% | Por batch |
| Qualification rate | qualified / replies | ≥ 40% | Por batch |
| Appointment conversion | appointments / qualified | ≥ 60% | Por batch |
| End-to-end conversion | appointments / kisses_sent | ≥ 3.6% | Por batch |
| Compliance adherence | compliant_sends / total_sends | 100% | Diario |
| Opt-out rate | opt_outs / replies | < 5% | Por batch |
| CRM sync success | successful_syncs / updates | ≥ 99.9% | Diario |
| Avg. conversation length | messages / qualified_leads | ≤ 5 | Por batch |

---

## 11. Deployment Checklist

### Pre-Deployment
- [ ] Configuración de nicho validada (`config/niches/{niche}_v1.yaml`)
- [ ] ChannelAdapter configurado y testeado (SMS/WhatsApp)
- [ ] CRMAdapter configurado con credenciales y permisos
- [ ] CalendarAdapter configurado con acceso al calendario del socio
- [ ] Compliance guardrails cargados con reglas de región
- [ ] Opt-out list inicializada (vacía o importada)
- [ ] Métricas backend configurado y recibiendo datos
- [ ] Logging estructurado habilitado con trace_id propagation
- [ ] Tests de integración pasando (≥ 90% coverage)
- [ ] Plan de rollback documentado

### Post-Deployment
- [ ] Monitorizar response rate por primeras 100 kisses
- [ ] Validar que opt-outs se procesan en < 1 minuto
- [ ] Confirmar que citas se agendan en calendario correcto
- [ ] Verificar que CRM se actualiza sin duplicados
- [ ] Revisar logs para detectar PII no masked
- [ ] Documentar cualquier desviación de targets

---

## 12. Version History

| Versión | Fecha | Cambios | Autor |
|---------|-------|---------|-------|
| 1.0 | 2026-02-20 | Especificación inicial de Sleeping Beauty DBR Engine | O.A. Perez Garrido |

---

## 13. Appendices

### Appendix A: Template Library por Nicho

| Nicho | Kiss Template | Positive Keywords | Qualification Focus |
|-------|--------------|-------------------|-------------------|
| **Clinicas** | "Hola {name}, somos [Clínica]. ¿Te gustaría recibir información sobre nuestros servicios? Responde SÍ para continuar." | ["sí", "info", "cita", "consulta"] | Especialidad, timing, seguro |
| **Viajes** | "Hola {name}, ¿sigues interesado en planear tu próximo viaje? Responde SÍ para que te enviemos opciones." | ["sí", "viaje", "destino", "cotizar"] | Destino, fechas, presupuesto |
| **Solar** | "Hola {name}, ¿te gustaría saber cuánto podrías ahorrar con paneles solares? Responde SÍ para una evaluación gratuita." | ["sí", "ahorro", "paneles", "cotización"] | Consumo, ubicación, tipo de inmueble |
| **Legal** | "Hola {name}, tenemos actualizaciones sobre [tema legal]. ¿Te interesa recibir información? Responde SÍ." | ["sí", "información", "caso", "asesoría"] | Tipo de caso, urgencia, representación |

### Appendix B: Signal Extraction Patterns

```yaml
# Ejemplo: extracción de señales desde texto de reply
signal_patterns:
  interest:
    - regex: "(quiero|necesito|me interesa|deseo|busco)"
    - keywords: ["cita", "consulta", "cotizar", "información"]
  
  timing_immediate:
    - regex: "(hoy|ahora|urgente|lo antes posible|inmediato)"
    - keywords: ["emergencia", "rápido", "ya"]
  
  timing_soon:
    - regex: "(esta semana|pronto|cuándo|próximos días)"
    - keywords: ["agendar", "programar", "disponibilidad"]
  
  budget_confirmed:
    - regex: "(presupuesto|precio|costo|cuánto|inversión)"
    - keywords: ["pago", "tarifa", "plan", "paquete"]
  
  authority:
    - regex: "(yo decido|soy el|mi familia|represento)"
    - keywords: ["dueño", "titular", "responsable"]
```

### Appendix C: Error Codes & Fallbacks

| Código | Descripción | Fallback | Alerta |
|--------|-------------|----------|--------|
| `KISS_SEND_FAILED` | Fallo al enviar mensaje inicial | Retry 2x con backoff, luego marcar lead como `dead` | Si > 5% de fallos en batch |
| `REPLY_PARSE_FAILED` | No se pudo normalizar reply | Loguear raw payload, ignorar mensaje | Si > 1% de fallos |
| `QUALIFICATION_TIMEOUT` | Lead no responde en ventana de calificación | Marcar como `dead`, emitir métrica | Ninguna (esperado) |
| `CALENDAR_NO_SLOT` | No hay slots disponibles en ventana | Enqueue para scheduling humano | Si > 20% de leads afectados |
| `CRM_SYNC_FAILED` | Fallo al actualizar CRM | Retry 3x, luego DLQ + alerta | Inmediata si crítico |
| `COMPLIANCE_BLOCK` | Envío bloqueado por reglas de compliance | No enviar, loguear razón | Si bloqueos > 10% de batch |

### Appendix D: Handoff a Humanos (Escenarios)

| Escenario | Trigger | Acción de Handoff | Información Entregada |
|-----------|---------|------------------|----------------------|
| **Lead calificado, sin slot** | `CALENDAR_NO_SLOT` | Enqueue en "Human Scheduling Queue" | Lead data, qualification score, preferred timing |
| **Pregunta fuera de scope** | `classify_reply() == "other"` + 3 intentos | Escalar a "Clarification Queue" | Conversation history, last question |
| **Señal de riesgo legal/médico** | `risk.level >= HIGH` en clasificación | Escalación inmediata a humano | Full trace, risk signals, raw message |
| **Solicitud de humano explícita** | Reply contiene "hablar con persona" | Transferencia directa a queue de ventas | Lead data + context of request |
| **Fallo técnico repetido** | 3 fallos consecutivos en mismo lead | Marcar como `needs_review` | Error log, retry history, raw payloads |

---

## ✅ Aprobaciones

| Rol | Nombre | Fecha | Firma |
|-----|--------|-------|-------|
| Product Owner | | | |
| Tech Lead | | | |
| Compliance Officer | | | |
| Sales Ops Lead | | | |

---

**Documento generado:** 20 de febrero de 2026  
**Próxima revisión:** 20 de marzo de 2026  
**Status:** Ready for Implementation

---

> **Nota final para el equipo:**  
> Este SPEC define el primer Android de ROYA: el motor de reactivación que convierte bases muertas en revenue.  
> Cada componente debe reutilizar ROYA_Shared_Components_v1 — nada de duplicar lógica.  
> La clave del éxito no es la IA, es la **consistencia conversacional** + **compliance estricto** + **métricas accionables**.  
> Si algo no encaja en este spec, pregunta: "¿Esto ayuda a reactivar leads de forma compliant y medible?" antes de implementarlo.