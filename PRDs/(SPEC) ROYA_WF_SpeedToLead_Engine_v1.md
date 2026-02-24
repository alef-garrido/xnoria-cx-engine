# 📄 Product Requirements Document (PRD) Técnico

## WF_SpeedToLead_Engine_v1
**Versión:** 1.0  
**Fecha:** 20 de febrero de 2026  
**Estado:** Especificación para implementación  
**Dominio:** cxEngine / Speed To Lead Android  
**Stack referencia:** Agnóstico (n8n/Make/Zapier + LLM + SMS/WhatsApp + CRM)

---

## 1. Overview Técnico

### 1.1 Propósito del sistema
Motor de respuesta inmediata a leads frescos que garantiza contacto en <60 segundos desde el opt-in, maximizando la probabilidad de conversión mediante calificación estructurada, nurturing contextual y routing inteligente al equipo de ventas.

### 1.2 Arquitectura de alto nivel
```
Lead Opt-In (Webhook) → Validation → Immediate Acknowledgment → Qualification Loop → Routing → CRM Sync
                              ↓
                    [cxEngine Shared Components]
                    - Lead Schema v1
                    - ChannelAdapter
                    - ComplianceGuardrails
                    - TraceEmitter
                    - MetricsEmitter
```

### 1.3 Principios técnicos fundamentales
- **Velocidad-first:** Cada milisegundo cuenta; arquitectura optimizada para latencia mínima
- **Consentimiento explícito:** Validación TCPA/GDPR antes del primer envío (no después)
- **Calificación progresiva:** Preguntas secuenciales que no abruman al lead
- **Routing determinista:** Decisión → acción sin ambigüedad para el equipo de ventas
- **Auditabilidad en tiempo real:** Toda interacción deja traza correlacionable

---

## 2. Problem Statement (Técnico)

### 2.1 Problema actual
Los leads frescos representan la mayor oportunidad de conversión, pero se pierden debido a:
- **Latencia humana:** Respuesta manual típica: 5–30 minutos → interés decae 400x
- **Fricción de handoff:** Formulario → CRM → asignación → contacto tiene múltiples puntos de falla
- **Calificación inconsistente:** Criterios subjetivos generan leads mal enrutados
- **Compliance reactivo:** Validación de consentimiento después del envío → riesgo legal

### 2.2 Impacto técnico
- Tasa de conversión de leads frescos < 5% en promedio
- Costo por lead adquirido no recuperado por falta de seguimiento
- Exposición legal por envíos sin consentimiento verificado
- Datos de interacción no estructurados → no accionables para optimización

---

## 3. Technical Objectives

### 3.1 Objetivos SMART

| ID | Objetivo | Métrica | Target |
|----|----------|---------|--------|
| **TO-1** | Contactar leads frescos en <60 segundos | P95 latency (opt-in → first message) | ≤ 60s |
| **TO-2** | Calificar leads con criterio estructurado | Qualification accuracy (vs human review) | ≥ 85% |
| **TO-3** | Enrutar leads calificados al vendedor correcto | Routing accuracy (right rep, right time) | ≥ 95% |
| **TO-4** | Mantener compliance 100% en todos los envíos | Consent verification rate pre-send | 100% |
| **TO-5** | Escalar a 500 leads/minuto sin degradación | Throughput sustained | ≥ 500 leads/min |

### 3.2 Requerimientos de calidad (NFRs)

| NFR | Descripción | Target |
|-----|-------------|--------|
| **NFR-1** | Disponibilidad del flujo de respuesta | 99.9% uptime |
| **NFR-2** | Latencia end-to-end (opt-in → first message) | P95 ≤ 60s, P50 ≤ 15s |
| **NFR-3** | Tolerancia a picos de tráfico (burst handling) | 10x baseline sin cola |
| **NFR-4** | Retención de trazas para auditoría | 90 días mínimo |
| **NFR-5** | Consistencia de routing cross-channel | ≥ 99% same-lead correlation |

---

## 4. Scope Técnico

### 4.1 In Scope

| Componente | Descripción | Responsabilidad |
|------------|-------------|-----------------|
| **Lead Ingestion** | Captura de webhook de opt-in (form, landing page, API) | Trigger/Adapter |
| **Consent Validation** | Verificación explícita de consentimiento TCPA/GDPR pre-send | Compliance |
| **Immediate Acknowledgment** | Primer mensaje de confirmación en <60s | Comunicación |
| **Qualification Loop** | Conversación estructurada de 1–3 preguntas para calificar | Decisión |
| **Lead Scoring** | Puntuación numérica basada en señales explícitas | Clasificación |
| **Intelligent Routing** | Asignación a vendedor por territorio, especialidad, carga | Integración |
| **CRM Sync** | Actualización en tiempo real del estado del lead | Sincronización |
| **Metrics & Logging** | Emisión de métricas de velocidad y conversión | Observabilidad |

### 4.2 Out of Scope

| Componente | Razón de exclusión |
|------------|-------------------|
| Generación de contenido creativo para campañas | Fuera de scope de automatización |
| Gestión de listas de opt-out centralizadas | Responsabilidad del socio / shared component |
| Integración con múltiples CRMs simultáneos | V1: un CRM por instancia |
| A/B testing de mensajes de acknowledgment | Optimización post-v1 |
| Soporte para canales voice/llamada | Solo SMS/WhatsApp en v1 |
| Machine Learning para scoring predictivo | Reglas explícitas en v1 |
| Handoff a chat humano en tiempo real | Solo routing a queue en v1 |

---

## 5. Functional Requirements

### 5.1 FR-1: Lead Ingestion & Validation

**ID:** FR-1  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe capturar leads frescos desde webhook y validar datos mínimos para procesamiento.

**User Story:**  
Como sistema de velocidad al lead, quiero recibir leads frescos desde cualquier fuente (form, API, landing page) y validar que tengan datos mínimos para contacto inmediato sin intervención manual.

**Acceptance Criteria:**
- [ ] Soporte para webhook POST con payload JSON estandarizado
- [ ] Validación de campos obligatorios: `phone` (E.164), `source`, `timestamp`, `consent_flag`
- [ ] Rechazo inmediato de leads sin consentimiento explícito (`consent_flag: false`)
- [ ] Normalización a `Lead_v1` schema de cxEngine Shared Components
- [ ] Asignación de `status: "new"` y `source: "fresh_optin"`
- [ ] Logging de leads rechazados con razón clara para debugging
- [ ] Throughput: ≥ 500 leads/minuto en ingestión

**Pseudocódigo de validación:**
```
FUNCTION ingest_fresh_lead(webhook_payload: OBJECT) → Lead_v1 OR NULL:
    // Validación de consentimiento (crítico para compliance)
    IF NOT webhook_payload.consent_flag:
        LOG: "consent_missing", phone: webhook_payload.phone, source: webhook_payload.source
        EMIT_METRIC: "lead_rejected_consent", tags: {source: webhook_payload.source}
        RETURN NULL
    
    // Validación de formato phone
    IF NOT is_valid_e164(webhook_payload.phone):
        LOG: "invalid_phone_format", phone: webhook_payload.phone
        EMIT_METRIC: "lead_rejected_format", tags: {source: webhook_payload.source}
        RETURN NULL
    
    // Deduplicación contra CRM (opcional, configurable)
    IF CONFIG.check_crm_duplicate AND crm_adapter.lead_exists(webhook_payload.phone):
        LOG: "duplicate_lead", phone: webhook_payload.phone
        EMIT_METRIC: "lead_rejected_duplicate", tags: {source: webhook_payload.source}
        RETURN NULL
    
    // Normalización a Lead_v1
    lead = {
        lead_id: GENERATE_UUID(),
        phone: webhook_payload.phone,
        name: webhook_payload.name OR NULL,
        email: webhook_payload.email OR NULL,
        source: "fresh_optin",
        status: "new",
        created_at: NOW(),
        schema_version: "v1",
        meta: {
            original_source: webhook_payload.source,
            utm_params: webhook_payload.utm OR {},
            landing_page: webhook_payload.page_url OR NULL,
            consent_timestamp: webhook_payload.consent_timestamp,
            custom_fields: webhook_payload.custom_fields OR {}
        }
    }
    
    LOG: "lead_ingested", lead_id: lead.lead_id, source: lead.meta.original_source
    EMIT_METRIC: "lead_ingested", value: 1, tags: {source: lead.meta.original_source}
    RETURN lead
```

---

### 5.2 FR-2: Immediate Acknowledgment Delivery

**ID:** FR-2  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe enviar el primer mensaje de acknowledgment en <60 segundos para confirmar recepción y establecer expectativa.

**User Story:**  
Como sistema de velocidad al lead, quiero enviar un mensaje inicial breve y personalizado que confirme recepción y establezca expectativa de seguimiento, para mantener el interés del lead fresco.

**Acceptance Criteria:**
- [ ] Mensaje máximo 2 frases, ≤ 160 caracteres para SMS
- [ ] Personalización mínima: `{name}` o genérico si no disponible
- [ ] Template configurable por nicho: `{niche_ack_template}`
- [ ] Envío vía ChannelAdapter (SMS/WhatsApp) con compliance check previo
- [ ] Actualización de lead a `status: "contacted"` tras envío exitoso
- [ ] Registro de timestamp y provider_message_id para trazabilidad
- [ ] Latencia P95 desde ingestión hasta envío: ≤ 60 segundos

**Pseudocódigo de envío:**
```
FUNCTION send_immediate_acknowledgment(lead: Lead_v1, template: OBJECT) → DeliveryResult:
    // Compliance check obligatorio antes de enviar (pre-send)
    compliance = compliance_guardrails.pre_send_check(
        lead = lead,
        content = {type: "text", body: "template_preview"},
        context = {campaign: "speed_to_lead", region: CONFIG.region}
    )
    
    IF compliance.result == "blocked":
        LOG: "ack_blocked", lead_id: lead.lead_id, reason: compliance.reason
        UPDATE lead.status = "disqualified"
        EMIT_METRIC: "ack_blocked_compliance", tags: {reason: compliance.reason}
        RETURN {status: "blocked", reason: compliance.reason}
    
    // Construcción del mensaje
    name = lead.name OR "amigo"
    message_body = template.text
        .replace("{name}", name)
        .replace("{niche}", template.niche_context)
        .replace("{next_step}", template.expected_next_step)
        .trim()
    
    // Validación de longitud para SMS
    IF CONFIG.channel == "sms" AND message_body.length > 160:
        message_body = message_body.substring(0, 157) + "..."
    
    // Envío vía adapter con timeout estricto
    start_time = NOW()
    delivery = channel_adapter.send_message(
        lead = lead,
        content = {type: "text", body: message_body},
        timeout_ms: 5000  // 5 segundos máximo para no bloquear flujo
    )
    latency = NOW() - start_time
    
    // Actualización de estado y métricas
    IF delivery.status == "sent" OR delivery.status == "queued":
        UPDATE lead.status = "contacted"
        UPDATE lead.metadata.first_ack_sent_at = NOW()
        UPDATE lead.metadata.ack_message_id = delivery.provider_message_id
        UPDATE lead.metadata.ack_latency_ms = latency
        EMIT_METRIC: "ack_sent", value: 1, tags: {niche: template.niche, channel: CONFIG.channel}
        EMIT_HISTOGRAM: "ack_latency", value: latency, tags: {channel: CONFIG.channel}
    
    LOG: "ack_delivery", lead_id: lead.lead_id, status: delivery.status, latency_ms: latency
    RETURN delivery
```

**Template base (ejemplo nicho clínico):**
```
{
  "niche": "clinicas",
  "text": "Hola {name}, gracias por tu interés en [Clínica]. Un asesor te contactará en breve para agendar tu cita. ¿Prefieres mañana o esta semana?",
  "expected_next_step": "calificación de timing",
  "positive_keywords": ["mañana", "esta semana", "hoy", "pronto", "sí"],
  "negative_keywords": ["no", "stop", "basta", "eliminar", "luego"]
}
```

---

### 5.3 FR-3: Qualification Loop & Reply Handling

**ID:** FR-3  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe capturar y procesar respuestas del lead para avanzar en la calificación mediante preguntas secuenciales estructuradas.

**User Story:**  
Como motor de calificación, quiero recibir y entender las respuestas del lead fresco para evaluar viabilidad de venta mediante 1–3 preguntas clave sin intervención humana, manteniendo contexto y propósito.

**Acceptance Criteria:**
- [ ] Webhook/listener para capturar replies de SMS/WhatsApp
- [ ] Normalización de reply a `NormalizedMessage` de Shared Components
- [ ] Correlación reply → lead original vía `phone` o `thread_id`
- [ ] Detección de intención básica: `positive`, `negative`, `question`, `other`
- [ ] Actualización de `ConversationState` con último intercambio
- [ ] Generación de siguiente pregunta basada en reglas configurables
- [ ] Timeout de conversación: si no hay reply en 24h → marcar como `dead`

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
    convo_state = conversation_state_manager.get_state(lead.lead_id) OR {stage: "ack_received", question_index: 0}
    convo_state.last_reply_at = message.timestamp
    convo_state.reply_count = (convo_state.reply_count OR 0) + 1
    conversation_state_manager.set_state(lead.lead_id, convo_state, ttl: 24_HOURS)
    
    // Clasificación básica de intención de reply
    intent = classify_reply_intent(message.body, template_keywords)
    // classify_reply_intent returns: "positive" | "negative" | "question" | "other"
    
    // Routing según intención y etapa de calificación
    SWITCH intent:
        CASE "positive":
            IF convo_state.question_index < MAX_QUESTIONS:
                RETURN ask_next_qualification_question(lead, message, convo_state)
            ELSE:
                RETURN trigger_final_scoring(lead, convo_state)
        CASE "negative":
            RETURN handle_negative_reply(lead, message)
        CASE "question":
            RETURN handle_question_reply(lead, message, convo_state)
        CASE "other":
            IF convo_state.reply_count <= 2:
                RETURN ask_clarifying_question(lead, message)
            ELSE:
                RETURN escalate_to_human_queue(lead, message, reason: "ambiguous_after_2_attempts")
    
    LOG: "reply_processed", lead_id: lead.lead_id, intent: intent, stage: convo_state.stage
    RETURN {status: "processed", next_action: determined_action}
```

---

### 5.4 FR-4: Lead Scoring & Qualification Decision

**ID:** FR-4  
**Prioridad:** ALTA  
**Descripción:** El sistema debe evaluar respuestas del lead mediante criterios estructurados para determinar viabilidad de venta y asignar puntuación numérica.

**User Story:**  
Como sistema de calificación, quiero evaluar señales explícitas del lead (interés, timing, presupuesto, autoridad) para clasificarlo como qualified/unqualified con puntuación objetiva y justificada.

**Acceptance Criteria:**
- [ ] Criterios de calificación configurables por nicho: `qualification_rules`
- [ ] Evaluación de señales: interés explícito, timing, presupuesto, autoridad, necesidad
- [ ] Puntuación numérica 0-100 con thresholds configurables
- [ ] Justificación estructurada de la decisión: `reason_codes[]`
- [ ] Actualización de `lead.status` y emisión de métricas de calificación
- [ ] Soporte para re-calificación si lead responde después de descalificación inicial

**Pseudocódigo de scoring:**
```
FUNCTION score_and_qualify_lead(lead: Lead_v1, conversation_history: ARRAY<Message>) → QualificationResult:
    // Cargar reglas de calificación por nicho
    rules = load_qualification_rules(lead.metadata.niche)
    
    // Extraer señales de la conversación
    signals = extract_qualification_signals(conversation_history, rules.signal_keywords)
    // signals: {interest: bool, timing: enum, budget: enum, authority: bool, need: bool}
    
    // Calcular score ponderado
    score = 0
    weights = rules.weights  // Configurable por nicho
    
    IF signals.interest: score += weights.interest * 30
    IF signals.timing == "immediate": score += weights.timing * 25
    IF signals.timing == "soon": score += weights.timing * 15
    IF signals.budget == "confirmed": score += weights.budget * 20
    IF signals.budget == "estimated": score += weights.budget * 10
    IF signals.authority: score += weights.authority * 15
    IF signals.need: score += weights.need * 10
    
    // Aplicar bonuses/penalties configurables
    IF conversation_history.length >= 3: score += 5  // Engagement bonus
    IF signals.urgency_keywords_detected: score += 10
    
    // Determinar resultado basado en thresholds
    IF score >= rules.thresholds.qualified:
        result = "qualified"
        reason_codes = ["high_interest", "good_timing", "budget_confirmed"] // según señales
        next_action = "route_to_sales"
    ELSE IF score <= rules.thresholds.unqualified:
        result = "unqualified"
        reason_codes = ["low_interest", "no_timing", "no_budget"]
        next_action = "disqualify"
    ELSE:
        result = "needs_more_info"
        reason_codes = ["insufficient_data"]
        next_action = "ask_more_questions"
    
    // Actualizar lead
    lead.status = result
    lead.metadata.qualification_score = score
    lead.metadata.qualification_reasons = reason_codes
    lead.metadata.qualified_at = result != "needs_more_info" ? NOW() : NULL
    
    // Emitir métrica
    EMIT_METRIC: "lead_qualified", value: 1, tags: {
        result: result, 
        niche: lead.metadata.niche, 
        score_bucket: score_bucket(score),
        source: lead.meta.original_source
    }
    
    LOG: "qualification_complete", lead_id: lead.lead_id, score: score, result: result
    RETURN {
        result: result,
        score: score,
        reason_codes: reason_codes,
        next_action: next_action,
        signals: signals
    }
```

---

### 5.5 FR-5: Intelligent Routing & CRM Sync

**ID:** FR-5  
**Prioridad:** ALTA  
**Descripción:** El sistema debe enrutar leads calificados al vendedor correcto y sincronizar el estado con el CRM del socio en tiempo real.

**User Story:**  
Como sistema de routing, quiero asignar leads calificados al vendedor más adecuado basado en territorio, especialidad y carga actual, y actualizar el CRM inmediatamente para eliminar fricción manual.

**Acceptance Criteria:**
- [ ] Reglas de routing configurables: territorio, especialidad, carga de trabajo, seniority
- [ ] Selección de vendedor disponible con menor carga actual
- [ ] Actualización de `lead.status = "converted"` y campos de asignación en CRM
- [ ] Notificación al vendedor asignado (SMS/email/Slack) con resumen del lead
- [ ] Manejo de fallos de routing con fallback a queue general
- [ ] Trazabilidad completa: assignment_id correlacionado con trace_id

**Pseudocódigo de routing:**
```
FUNCTION route_qualified_lead(lead: Lead_v1, qualification: QualificationResult) → RoutingResult:
    // Obtener configuración de routing del socio
    routing_config = load_routing_config(lead.metadata.partner_id)
    
    // Filtrar vendedores elegibles por reglas
    eligible_reps = crm_adapter.get_sales_reps({
        territory: lead.meta.territory OR routing_config.default_territory,
        specialty: lead.meta.specialty_interest OR routing_config.default_specialty,
        status: "active",
        max_current_load: routing_config.max_leads_per_rep
    })
    
    IF eligible_reps.length == 0:
        // Fallback: enqueue para routing manual
        enqueue_for_manual_routing(lead, qualification)
        LOG: "no_eligible_rep", lead_id: lead.lead_id
        RETURN {status: "queued_for_manual", reason: "no_eligible_rep_available"}
    
    // Seleccionar rep con menor carga actual (load balancing)
    selected_rep = select_rep_with_lowest_load(eligible_reps)
    
    // Asignar lead en CRM
    assignment = crm_adapter.assign_lead(
        lead_id: lead.lead_id,
        rep_id: selected_rep.id,
        priority: qualification.score >= 80 ? "high" : "normal",
        notes: build_assignment_notes(lead, qualification),
        metadata: {
            qualification_score: qualification.score,
            source: "speed_to_lead_android",
            assigned_at: NOW()
        }
    )
    
    // Notificar al vendedor
    IF routing_config.notify_rep_immediately:
        notify_sales_rep({
            rep: selected_rep,
            lead: lead,
            qualification: qualification,
            channel: routing_config.rep_notification_channel  // sms/email/slack
        })
    
    // Actualizar lead
    lead.status = "converted"
    lead.metadata.assigned_rep_id = selected_rep.id
    lead.metadata.assigned_at = NOW()
    crm_adapter.update_lead(lead.lead_id, {
        status: "converted",
        assigned_rep: selected_rep.name,
        qualification_score: qualification.score,
        last_interaction: NOW()
    })
    
    // Emitir métrica de conversión
    EMIT_METRIC: "lead_routed", value: 1, tags: {
        niche: lead.metadata.niche, 
        rep_id: selected_rep.id,
        priority: assignment.priority
    }
    
    LOG: "lead_routed", lead_id: lead.lead_id, rep_id: selected_rep.id, assignment_id: assignment.id
    RETURN {status: "routed", rep_id: selected_rep.id, assignment_id: assignment.id}
```

---

### 5.6 FR-7: Compliance & Consent Enforcement

**ID:** FR-7  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe verificar consentimiento explícito antes de cada envío y respetar automáticamente opt-outs y regulaciones (TCPA/GDPR).

**User Story:**  
Como operador en región regulada, quiero que el sistema verifique consentimiento explícito antes del primer envío y procese opt-outs inmediatamente, para cumplir con la ley y proteger la reputación.

**Acceptance Criteria:**
- [ ] Validación de `consent_flag` en ingestión (bloqueo si false)
- [ ] Registro de timestamp de consentimiento para auditoría
- [ ] Detección de keywords de opt-out: "STOP", "BAJA", "UNSUBSCRIBE", etc.
- [ ] Procesamiento de opt-out en < 1 minuto desde recepción
- [ ] Actualización de lista de opt-out centralizada (shared component)
- [ ] Bloqueo inmediato de futuros envíos a números opt-out
- [ ] Confirmación de opt-out al usuario (mensaje estándar)
- [ ] Logging de todos los eventos de consentimiento para auditoría

**Pseudocódigo de manejo de consentimiento:**
```
FUNCTION verify_consent_and_handle_optout(lead: Lead_v1, message: NormalizedMessage) → ConsentResult:
    // Verificar consentimiento en ingestión (pre-send check)
    IF NOT lead.meta.consent_timestamp:
        LOG: "consent_missing_at_ingest", lead_id: lead.lead_id
        RETURN {status: "blocked", reason: "no_consent_recorded"}
    
    // Verificar si ya está en opt-out
    IF compliance_guardrails.is_opted_out(lead.phone):
        LOG: "optout_already_active", phone: lead.phone
        RETURN {status: "blocked", reason: "already_opted_out"}
    
    // Detectar opt-out en mensaje entrante
    IF contains_optout_keyword(message.body, CONFIG.optout_keywords):
        // Procesar opt-out inmediatamente
        compliance_guardrails.add_to_optout_list(lead.phone, {
            timestamp: message.timestamp,
            channel: message.channel,
            campaign: "speed_to_lead",
            reason: "user_requested_via_reply"
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
        
        // Confirmación obligatoria por TCPA
        confirmation = CONFIG.optout_confirmation_message[message.channel] OR "Has sido dado de baja. No recibirás más mensajes."
        channel_adapter.send_message(lead, {type: "text", body: confirmation})
        
        // Emitir métrica de compliance
        EMIT_METRIC: "opt_out_processed", value: 1, tags: {channel: message.channel, region: CONFIG.region}
        
        LOG: "opt_out_processed", lead_id: lead.lead_id, phone: lead.phone
        RETURN {status: "opted_out", confirmation_sent: true}
    
    // Consentimiento válido, permitir continuación
    RETURN {status: "allowed", consent_verified: true}
```

---

### 5.7 FR-8: Metrics Emission & Observability

**ID:** FR-8  
**Prioridad:** ALTA  
**Descripción:** El sistema debe emitir métricas estructuradas para medir performance de velocidad y conversión, permitiendo optimización en tiempo real.

**User Story:**  
Como analista, quiero métricas en tiempo real de latencia, respuesta y conversión por fuente, para identificar cuellos de botella y ajustar la estrategia de velocidad.

**Acceptance Criteria:**
- [ ] Métricas de velocidad: `ingest_to_ack_latency`, `ack_to_reply_latency`, `total_conversion_time`
- [ ] Métricas de funnel: `lead_ingested`, `ack_sent`, `reply_received`, `lead_qualified`, `lead_routed`
- [ ] Métricas de calidad: `response_rate`, `qualification_rate`, `routing_accuracy`, `opt_out_rate`
- [ ] Segmentación por: `source`, `niche`, `channel`, `region`, `campaign_id`
- [ ] Emisión asíncrona vía MetricsEmitter de Shared Components
- [ ] Correlación con trace_id para debugging end-to-end
- [ ] Dashboard-ready: formato compatible con Prometheus/Datadog

**Pseudocódigo de emisión de métricas:**
```
FUNCTION emit_speed_to_lead_metrics(lead: Lead_v1, event: ENUM, meta OBJECT):
    base_tags = {
        android: "speed_to_lead",
        niche: lead.metadata.niche,
        channel: CONFIG.channel,
        region: CONFIG.region,
        source: lead.meta.original_source
    }
    
    SWITCH event:
        CASE "lead_ingested":
            metrics_emitter.emit_counter("stl_lead_ingested", 1, base_tags)
            
        CASE "ack_sent":
            metrics_emitter.emit_counter("stl_ack_sent", 1, base_tags)
            // Emitir latencia crítica
            IF meta.ack_latency_ms:
                metrics_emitter.emit_histogram("stl_ack_latency", meta.ack_latency_ms, base_tags)
            
        CASE "reply_received":
            metrics_emitter.emit_counter("stl_reply_received", 1, base_tags)
            // Calcular response rate si tenemos ack_sent del mismo source
            response_rate = calculate_response_rate(lead.meta.original_source, time_window: "1h")
            metrics_emitter.emit_gauge("stl_response_rate", response_rate, base_tags)
            
        CASE "lead_qualified":
            metrics_emitter.emit_counter("stl_lead_qualified", 1, {
                ...base_tags,
                qualification_result: meta.result,
                score_bucket: score_bucket(meta.score)
            })
            
        CASE "lead_routed":
            metrics_emitter.emit_counter("stl_lead_routed", 1, base_tags)
            // Calcular conversión final
            conversion_rate = calculate_conversion_rate(lead.meta.original_source, time_window: "24h")
            metrics_emitter.emit_gauge("stl_conversion_rate", conversion_rate, base_tags)
            
        CASE "opt_out_processed":
            metrics_emitter.emit_counter("stl_opt_out", 1, base_tags)
```

---

## 6. Non-Functional Requirements

### 6.1 Performance

| Métrica | Target | Medición |
|---------|--------|----------|
| Latencia ingestión → acknowledgment | P95 ≤ 60s, P50 ≤ 15s | End-to-end tracing |
| Throughput de ingestión | ≥ 500 leads/minuto | Sustained load |
| Tiempo de calificación | < 3 segundos por interacción | P95 |
| Routing + CRM sync | < 5 segundos desde calificación | P95 |

### 6.2 Reliability

| Métrica | Target | Estrategia |
|---------|--------|------------|
| Ack delivery rate | ≥ 99% | Retry + fallback channel |
| Reply capture rate | ≥ 99.5% | Webhook redundancy + polling fallback |
| CRM sync success | 100% | Idempotent updates + DLQ |
| Compliance enforcement | 100% | Pre-send check obligatorio, fail-safe |

### 6.3 Security

| Requisito | Implementación |
|-----------|----------------|
| PII en logs | Masking automático: phone → `+52***`, email → `u***@***` |
| Credenciales de adapters | Secrets management, nunca en código |
| Webhook verification | HMAC signature para replies de WhatsApp |
| Consent data integrity | Write-once, append-only, audit trail |

### 6.4 Scalability

| Dimensión | Estrategia |
|-----------|------------|
| Horizontal | Stateless processing, queue-based ingestion |
| Multi-niche | Configuración por `niche_id`, reglas aisladas |
| Multi-partner | Aislamiento por `partner_id` en metadata |
| Multi-region | Compliance config por región, timezone-aware |

---

## 7. Technical Specifications

### 7.1 Contratos Extendidos (sobre cxEngine Shared Components)

#### FreshLeadInput Schema (extensión de Lead_v1)
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "FreshLeadInput Schema",
  "type": "object",
  "required": ["phone", "source", "consent_flag", "timestamp"],
  "properties": {
    "phone": {"type": "string", "pattern": "^\\+[1-9]\\d{1,14}$"},
    "source": {"type": "string", "enum": ["web_form", "landing_page", "api", "chat_widget"]},
    "consent_flag": {"type": "boolean", "description": "Explicit TCPA/GDPR consent"},
    "timestamp": {"type": "string", "format": "date-time"},
    "name": {"type": "string"},
    "email": {"type": "string", "format": "email"},
    "utm_params": {"type": "object"},
    "page_url": {"type": "string"},
    "custom_fields": {"type": "object", "additionalProperties": true}
  }
}
```

#### QualificationResult Schema
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "QualificationResult",
  "type": "object",
  "required": ["result", "score", "reason_codes", "next_action"],
  "properties": {
    "result": {"enum": ["qualified", "unqualified", "needs_more_info"]},
    "score": {"type": "integer", "minimum": 0, "maximum": 100},
    "reason_codes": {"type": "array", "items": {"type": "string"}},
    "next_action": {"enum": ["route_to_sales", "disqualify", "ask_more_questions"]},
    "signals": {
      "type": "object",
      "properties": {
        "interest": {"type": "boolean"},
        "timing": {"enum": ["immediate", "soon", "later", "unknown"]},
        "budget": {"enum": ["confirmed", "estimated", "unknown"]},
        "authority": {"type": "boolean"},
        "need": {"type": "boolean"}
      }
    }
  }
}
```

### 7.2 Configuración por Nicho (Ejemplo: Clínicas)

```yaml
# config/niches/clinicas_speed_v1.yaml
niche_id: clinicas
channel: sms  # o whatsapp
language: es-MX

immediate_acknowledgment:
  template: "Hola {name}, gracias por tu interés en [Clínica]. Un asesor te contactará en breve para agendar tu cita. ¿Prefieres mañana o esta semana?"
  expected_next_step: "calificación de timing"
  positive_keywords: ["mañana", "esta semana", "hoy", "pronto", "sí"]
  negative_keywords: ["no", "stop", "baja", "eliminar", "basta"]

qualification_rules:
  signal_keywords:
    interest: ["quiero", "necesito", "me interesa", "cita", "consulta"]
    timing_immediate: ["hoy", "ahora", "urgente", "lo antes posible"]
    timing_soon: ["esta semana", "pronto", "cuándo puedo"]
    budget_confirmed: ["presupuesto", "precio", "costo", "cuánto"]
    authority: ["yo decido", "soy el paciente", "mi familia"]
  
  weights:
    interest: 1.0
    timing: 0.8
    budget: 0.7
    authority: 0.6
    need: 0.5
  
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

routing_config:
  rules:
    - field: "specialty_interest"
      map_to: "rep_specialty"
    - field: "location"
      map_to: "rep_territory"
  default_territory: "general"
  default_specialty: "general"
  max_leads_per_rep: 10
  notify_rep_immediately: true
  rep_notification_channel: "sms"

compliance:
  region: MX
  optout_keywords: ["STOP", "BAJA", "CANCELAR", "NO MÁS"]
  optout_confirmation: "Has sido dado de baja. Para reactivar, envía ALTA a este número."
  max_messages_per_day: 3
  allowed_hours: "08:00-21:00"
  consent_retention_days: 730  # 2 años para TCPA
```

### 7.3 Flujo de Estado del Lead (State Machine)

```
[NEW] 
   ↓ (ingestion + consent check)
[NEW] → send_ack() → [CONTACTED]
   ↓ (reply received)
[CONTACTED] → classify_reply()
   ├─ positive → ask_qualification_question() → [QUALIFYING]
   ├─ negative → handle_negative() → [DISQUALIFIED]
   └─ question/other → ask_clarifying() → [QUALIFYING]

[QUALIFYING] → evaluate_signals() + score()
   ├─ score ≥ 70 → route_to_sales() → [CONVERTED]
   ├─ score ≤ 30 → disqualify() → [DISQUALIFIED]
   └─ 31-69 → ask_more_questions() → [QUALIFYING] (max 3 rounds)

[CONVERTED] → sync_crm() + notify_rep() → [CLOSED]

[DISQUALIFIED] → log_reason() → [CLOSED]

Timeouts:
- CONTACTED → no reply in 24h → [DEAD]
- QUALIFYING → no reply in 12h → [DEAD]
```

---

## 8. Implementation Notes

### 8.1 Stack Reference (n8n como ejemplo)

| Componente | Nodo n8n | Configuración |
|------------|----------|---------------|
| Lead Ingestion | Webhook + Code | Parse JSON, validate consent, dedupe |
| Ack Delivery | HTTP Request (Twilio/Meta) + Compliance Check | Rate limiting, retry, timeout 5s |
| Reply Listener | Webhook (WhatsApp) / Polling (SMS) | HMAC verify, parse, correlate |
| Qualification | AI Agent (LLM) + Rules Engine | Structured output, scoring |
| Routing | Code + HTTP Request (CRM) | Load balancing, fallback queue |
| CRM Sync | HTTP Request (HighLevel/Salesforce) | Idempotent upsert, assignment |
| Metrics | HTTP Request (Prometheus push) | Async, batched, tagged |
| Logging | Write to File / HTTP | Structured JSON, trace_id propagation |

### 8.2 Alternative Stacks

| Stack | Ventajas | Consideraciones |
|-------|----------|-----------------|
| **n8n + Twilio** | Visual, rápido para prototipar | Twilio pricing por mensaje |
| **Make.com + Meta WhatsApp** | Buena UI, WhatsApp nativo | Menos flexible para lógica custom |
| **Custom Node.js** | Máximo control, performance | Requiere más desarrollo y ops |
| **Serverless (Lambda)** | Escalado automático | Cold starts, debugging complejo |

### 8.3 Environment Variables

```bash
# Speed To Lead Engine Config
STL_DEFAULT_NICHE=clinicas
STL_MAX_CONVERSATION_ROUNDS=3
STL_REPLY_TIMEOUT_HOURS=24
STL_QUALIFICATION_TIMEOUT_HOURS=12
STL_ACK_TIMEOUT_MS=5000

# Channel Config
CHANNEL_PROVIDER=twilio|meta_whatsapp
CHANNEL_PHONE_NUMBER=+52...
CHANNEL_API_KEY=xxx

# CRM Config
CRM_PROVIDER=highlevel|salesforce|hubspot
CRM_API_KEY=xxx
CRM_WEBHOOK_SECRET=xxx
CRM_ROUTING_ENABLED=true

# Compliance
COMPLIANCE_REGION=MX|US|EU
COMPLIANCE_CONSENT_REQUIRED=true
COMPLIANCE_OPTOUT_LIST_URL=https://.../optout
COMPLIANCE_MAX_MESSAGES_PER_DAY=3

# Metrics
METRICS_BACKEND=prometheus|datadog
METRICS_BATCH_SIZE=100
METRICS_FLUSH_INTERVAL_SECONDS=30
METRICS_EMIT_LATENCY=true
```

---

## 9. Testing Strategy

### 9.1 Test Cases

| ID | Descripción | Input | Expected Output |
|----|-------------|-------|-----------------|
| **TC-STL-01** | Ingestión de lead válido | Webhook con phone, consent_flag=true | Lead en estado `new` |
| **TC-STL-02** | Ack delivery exitoso | Lead en `new` | Lead en `contacted`, mensaje enviado en <60s |
| **TC-STL-03** | Reply positiva → calificación | Reply: "Sí, quiero cita mañana" | Lead en `qualifying`, pregunta de seguimiento |
| **TC-STL-04** | Calificación con score alto | Signals: interest+timing+budget | Lead en `converted`, asignado a rep |
| **TC-STL-05** | Opt-out processing | Reply: "STOP" | Lead en `disqualified`, opt-out registrado |
| **TC-STL-06** | Timeout sin reply | Lead en `contacted`, 24h sin reply | Lead en `dead` |
| **TC-STL-07** | Compliance block pre-send | Lead sin consent_flag | Ack no enviado, log de bloqueo |
| **TC-STL-08** | Routing con carga balanceada | 3 reps disponibles, 2 con carga alta | Lead asignado a rep con menor carga |
| **TC-STL-09** | Métricas de velocidad | Batch de 100 leads procesados | Métricas: ingested=100, ack_sent=99, avg_latency=12s |
| **TC-STL-10** | Fallback de routing | Sin reps elegibles disponibles | Lead enqueued para routing manual |

### 9.2 Test Execution

```bash
# Unit tests (componentes individuales)
npm test -- unit/ingestion/
npm test -- unit/qualification/
npm test -- unit/compliance/
npm test -- unit/routing/

# Integration tests (flujos end-to-end)
npm test -- integration/ack-to-reply/
npm test -- integration/qualification-to-routing/

# Compliance tests (regulaciones)
npm test -- compliance/consent-validation/
npm test -- compliance/optout-processing/

# Load tests (escalabilidad)
npm test -- load/ingestion-500-leads/
npm test -- load/concurrent-conversations-200/

# Latency tests (velocidad crítica)
npm test -- latency/ack-delivery-p95/
npm test -- latency/end-to-end-conversion/
```

---

## 10. Success Metrics (Técnicos y de Negocio)

| Métrica | Fórmula | Target | Frecuencia |
|---------|---------|--------|------------|
| Ack latency P95 | percentile_95(ack_sent - ingested) | ≤ 60s | Diario |
| Response rate | replies / acks_sent | ≥ 25% | Por fuente |
| Qualification rate | qualified / replies | ≥ 50% | Por fuente |
| Routing accuracy | correct_assignments / routed | ≥ 95% | Semanal |
| End-to-end conversion | routed / ingested | ≥ 12.5% | Por fuente |
| Compliance adherence | compliant_sends / total_sends | 100% | Diario |
| Opt-out rate | opt_outs / replies | < 3% | Por fuente |
| CRM sync success | successful_syncs / updates | ≥ 99.9% | Diario |

---

## 11. Deployment Checklist

### Pre-Deployment
- [ ] Configuración de nicho validada (`config/niches/{niche}_speed_v1.yaml`)
- [ ] ChannelAdapter configurado y testeado (SMS/WhatsApp)
- [ ] CRMAdapter configurado con credenciales y permisos de routing
- [ ] Compliance guardrails cargados con reglas de región y consentimiento
- [ ] Opt-out list inicializada (vacía o importada)
- [ ] Métricas backend configurado y recibiendo datos de latencia
- [ ] Logging estructurado habilitado con trace_id propagation
- [ ] Tests de latencia P95 pasando (<60s ack delivery)
- [ ] Tests de integración pasando (≥ 90% coverage)
- [ ] Plan de rollback documentado para fallos de routing

### Post-Deployment
- [ ] Monitorizar ack latency P95 por primeras 100 leads
- [ ] Validar que opt-outs se procesan en < 1 minuto
- [ ] Confirmar que routing asigna reps correctamente
- [ ] Verificar que CRM se actualiza sin duplicados
- [ ] Revisar logs para detectar PII no masked
- [ ] Documentar cualquier desviación de targets de velocidad

---

## 12. Version History

| Versión | Fecha | Cambios | Autor |
|---------|-------|---------|-------|
| 1.0 | 2026-02-20 | Especificación inicial de Speed To Lead Engine | O.A. Perez Garrido |

---

## 13. Appendices

### Appendix A: Template Library por Nicho

| Nicho | Ack Template | Positive Keywords | Qualification Focus |
|-------|-------------|-------------------|-------------------|
| **Clinicas** | "Hola {name}, gracias por tu interés en [Clínica]. Un asesor te contactará en breve para agendar tu cita. ¿Prefieres mañana o esta semana?" | ["mañana", "esta semana", "hoy", "sí"] | Especialidad, timing, seguro |
| **Viajes** | "Hola {name}, ¡genial que quieras viajar! Un asesor te enviará opciones personalizadas. ¿Ya tienes destino o fechas en mente?" | ["sí", "destino", "fechas", "cotizar"] | Destino, fechas, presupuesto |
| **Solar** | "Hola {name}, gracias por solicitar info solar. Un experto te contactará para una evaluación gratuita. ¿Tu consumo mensual es mayor a $1,000 MXN?" | ["sí", "consumo", "recibo", "evaluación"] | Consumo, ubicación, tipo inmueble |
| **Legal** | "Hola {name}, recibimos tu consulta. Un abogado te contactará para evaluar tu caso. ¿Es un asunto urgente que requiere atención inmediata?" | ["sí", "urgente", "inmediato", "caso"] | Tipo de caso, urgencia, representación |

### Appendix B: Signal Extraction Patterns

```yaml
# Ejemplo: extracción de señales desde texto de reply
signal_patterns:
  interest:
    - regex: "(quiero|necesito|me interesa|deseo|busco)"
    - keywords: ["cita", "consulta", "cotizar", "información"]
  
  timing_immediate:
    - regex: "(hoy|ahora|urgente|lo antes posible|inmediato)"
    - keywords: ["emergencia", "rápido", "ya", "mañana"]
  
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
| `ACK_SEND_FAILED` | Fallo al enviar acknowledgment | Retry 2x con backoff, luego marcar lead como `dead` | Si > 2% de fallos en ventana de 5min |
| `CONSENT_MISSING` | Lead sin consentimiento explícito | No enviar, loguear, emitir métrica de rechazo | Si > 5% de leads sin consentimiento |
| `REPLY_PARSE_FAILED` | No se pudo normalizar reply | Loguear raw payload, ignorar mensaje | Si > 1% de fallos |
| `QUALIFICATION_TIMEOUT` | Lead no responde en ventana de calificación | Marcar como `dead`, emitir métrica | Ninguna (esperado) |
| `ROUTING_NO_REP` | No hay reps elegibles disponibles | Enqueue para routing manual | Si > 10% de leads afectados |
| `CRM_SYNC_FAILED` | Fallo al actualizar CRM | Retry 3x, luego DLQ + alerta | Inmediata si crítico |
| `COMPLIANCE_BLOCK` | Envío bloqueado por reglas de compliance | No enviar, loguear razón | Si bloqueos > 2% de batch |

### Appendix D: Handoff a Humanos (Escenarios)

| Escenario | Trigger | Acción de Handoff | Información Entregada |
|-----------|---------|------------------|----------------------|
| **Lead calificado, sin rep disponible** | `ROUTING_NO_REP` | Enqueue en "Manual Routing Queue" | Lead data, qualification score, preferred timing |
| **Pregunta fuera de scope** | `classify_reply() == "other"` + 2 intentos | Escalar a "Clarification Queue" | Conversation history, last question |
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
> Este SPEC define el segundo Android de cxEngine: el motor de velocidad que convierte leads frescos en oportunidades calificadas antes de que la competencia pueda reaccionar.  
> Cada componente debe reutilizar cxEngine_Shared_Components_v1 — nada de duplicar lógica.  
> La clave del éxito no es la IA, es la **velocidad consistente** + **compliance estricto** + **routing inteligente**.  
> Si algo no encaja en este spec, pregunta: "¿Esto ayuda a contactar leads más rápido de forma compliant y medible?" antes de implementarlo.