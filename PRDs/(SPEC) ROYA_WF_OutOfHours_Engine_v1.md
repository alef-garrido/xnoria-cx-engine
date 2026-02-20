# 📄 Product Requirements Document (PRD) Técnico

## WF_OutOfHours_Engine_v1
**Versión:** 1.0  
**Fecha:** 20 de febrero de 2026  
**Estado:** Especificación para implementación  
**Dominio:** ROYA / Out Of Hours Android  
**Stack referencia:** Agnóstico (n8n/Make/Zapier + LLM + SMS/WhatsApp + CRM)

---

## 1. Overview Técnico

### 1.1 Propósito del sistema
Motor de gestión inteligente de leads que contactan fuera del horario laboral, diseñado para mantener el engagement mediante nurturing contextual, establecer expectativas claras de seguimiento y garantizar handoff eficiente al equipo de ventas en el próximo ventana disponible, sin intervención humana en tiempo real.

### 1.2 Arquitectura de alto nivel
```
Lead Contact (Fuera de Horas) → Time Detection → Acknowledgment + Nurturing → Queue for Handoff → CRM Sync → Next-Day Routing
                              ↓
                    [ROYA Shared Components]
                    - Lead Schema v1
                    - ChannelAdapter
                    - ComplianceGuardrails
                    - TraceEmitter
                    - MetricsEmitter
```

### 1.3 Principios técnicos fundamentales
- **Time-aware:** Detección precisa de horario laboral por timezone del lead o configuración del socio
- **Nurturing, no venta:** Comunicación empática que mantiene interés sin presionar fuera de horas
- **Expectativa clara:** El lead sabe cuándo será contactado, reduciendo ansiedad y abandono
- **Handoff determinista:** Lead calificado y encolado para seguimiento en próximo ventana laboral
- **Compliance horario:** Respeto automático a regulaciones de contacto por región y hora

---

## 2. Problem Statement (Técnico)

### 2.1 Problema actual
Los leads que contactan fuera de horario laboral representan una oportunidad perdida debido a:
- **Respuesta tardía:** Sin acknowledgment inmediato, el lead busca competencia o pierde interés
- **Fricción de handoff:** Información del lead no se captura ni prioriza para el siguiente día
- **Compliance reactivo:** Envíos fuera de horario permitido generan riesgo legal en regiones reguladas
- **Pérdida de señal:** Contexto de la interacción nocturna no se preserva para el equipo diurno

### 2.2 Impacto técnico
- Tasa de abandono de leads fuera de horas: ≥ 60% en promedio
- Costo de adquisición no recuperado por falta de seguimiento oportuno
- Exposición legal por contactos en horarios no permitidos (TCPA, GDPR)
- Datos de interacción nocturna no estructurados → no accionables para optimización

---

## 3. Technical Objectives

### 3.1 Objetivos SMART

| ID | Objetivo | Métrica | Target |
|----|----------|---------|--------|
| **TO-1** | Detectar contacto fuera de horas con precisión timezone-aware | Detection accuracy | ≥ 99% |
| **TO-2** | Enviar acknowledgment en <30 segundos desde detección | P95 latency (detect → ack) | ≤ 30s |
| **TO-3** | Mantener engagement del lead hasta handoff | Lead retention rate (next-day contact) | ≥ 70% |
| **TO-4** | Garantizar compliance horario 100% en todos los envíos | Compliance adherence rate | 100% |
| **TO-5** | Escalar a 200 leads/hora fuera de horario sin degradación | Throughput sustained | ≥ 200 leads/hr |

### 3.2 Requerimientos de calidad (NFRs)

| NFR | Descripción | Target |
|-----|-------------|--------|
| **NFR-1** | Disponibilidad del flujo fuera de horas | 99.5% uptime |
| **NFR-2** | Latencia acknowledgment | P95 ≤ 30s |
| **NFR-3** | Tolerancia a fallos de configuración horaria | Fallback a configuración global |
| **NFR-4** | Retención de trazas para auditoría | 90 días mínimo |
| **NFR-5** | Consistencia cross-timezone | ≥ 99% correcta detección |

---

## 4. Scope Técnico

### 4.1 In Scope

| Componente | Descripción | Responsabilidad |
|------------|-------------|-----------------|
| **Time Detection** | Determina si contacto ocurre dentro/fuera de horario laboral | Trigger/Logic |
| **Acknowledgment Delivery** | Mensaje inicial que confirma recepción y establece expectativa | Comunicación |
| **Nurturing Loop** | Respuestas empáticas a preguntas frecuentes sin venta activa | Decisión |
| **Queue Management** | Encolado de lead para handoff en próximo ventana laboral | Estado |
| **Handoff Scheduler** | Programación automática de seguimiento para equipo de ventas | Integración |
| **CRM Sync** | Actualización de estado del lead con contexto nocturno | Sincronización |
| **Compliance Enforcement** | Validación de horario permitido por región antes de cada envío | Seguridad legal |
| **Metrics & Logging** | Emisión de métricas de engagement y handoff success | Observabilidad |

### 4.2 Out of Scope

| Componente | Razón de exclusión |
|------------|-------------------|
| Venta activa o calificación profunda fuera de horas | Fuera de propósito del Android |
| Integración con múltiples CRMs simultáneos | V1: un CRM por instancia |
| A/B testing de mensajes de acknowledgment | Optimización post-v1 |
| Soporte para canales voice/llamada | Solo SMS/WhatsApp en v1 |
| Machine Learning para predicción de horario óptimo | Reglas explícitas en v1 |
| Handoff a chat humano en tiempo real | Solo queue para next-day en v1 |

---

## 5. Functional Requirements

### 5.1 FR-1: Time Detection & Business Hours Validation

**ID:** FR-1  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe determinar si un contacto ocurre dentro o fuera del horario laboral configurado, considerando timezone del lead o configuración del socio.

**User Story:**  
Como sistema de gestión fuera de horas, quiero detectar con precisión si un lead contacta fuera de horario laboral para activar el flujo de nurturing apropiado sin intervención humana.

**Acceptance Criteria:**
- [ ] Configuración de horario laboral por socio: `business_hours` con días y rangos horarios
- [ ] Soporte para múltiples timezones: IANA timezone strings (e.g., `America/Mexico_City`)
- [ ] Detección basada en timezone del lead (si disponible) o fallback a timezone del socio
- [ ] Consideración de días festivos configurables: `holidays` list
- [ ] Output: `is_business_hours: boolean`, `next_available_window: ISO-8601`
- [ ] Logging de detección para auditoría y debugging
- [ ] Throughput: ≥ 200 detecciones/minuto

**Pseudocódigo de detección:**
```
FUNCTION is_out_of_hours(contact_timestamp: ISO-8601, lead_timezone: STRING OR NULL, config: OBJECT) → TimeDecision:
    // Determinar timezone a usar
    timezone = lead_timezone OR config.default_timezone OR "UTC"
    
    // Convertir timestamp a timezone local
    local_time = CONVERT_TO_TIMEZONE(contact_timestamp, timezone)
    
    // Verificar día festivo
    IF local_time.date IN config.holidays:
        RETURN {
            is_business_hours: false,
            reason: "holiday",
            next_available_window: NEXT_BUSINESS_DAY(config, timezone)
        }
    
    // Verificar día de la semana
    day_of_week = local_time.day_of_week  // 0=Sunday, 6=Saturday
    IF day_of_week NOT IN config.business_days:
        RETURN {
            is_business_hours: false,
            reason: "weekend",
            next_available_window: NEXT_BUSINESS_DAY(config, timezone)
        }
    
    // Verificar rango horario
    business_hours = config.business_hours[day_of_week]
    IF NOT business_hours:  // Día sin horario definido
        RETURN {
            is_business_hours: false,
            reason: "no_hours_configured",
            next_available_window: NEXT_BUSINESS_DAY(config, timezone)
        }
    
    current_time_minutes = local_time.hours * 60 + local_time.minutes
    start_minutes = PARSE_TIME(business_hours.start)
    end_minutes = PARSE_TIME(business_hours.end)
    
    IF current_time_minutes >= start_minutes AND current_time_minutes < end_minutes:
        RETURN {is_business_hours: true, reason: "within_hours"}
    ELSE:
        RETURN {
            is_business_hours: false,
            reason: current_time_minutes < start_minutes ? "before_hours" : "after_hours",
            next_available_window: CALCULATE_NEXT_WINDOW(local_time, business_hours, timezone)
        }
```

---

### 5.2 FR-2: Immediate Acknowledgment & Expectation Setting

**ID:** FR-2  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe enviar un mensaje de acknowledgment en <30 segundos que confirme recepción, establezca expectativa de seguimiento y mantenga engagement sin venta activa.

**User Story:**  
Como sistema de nurturing fuera de horas, quiero enviar un mensaje inicial empático y personalizado que confirme recepción y establezca expectativa clara de cuándo será contactado el lead, para mantener interés sin presionar.

**Acceptance Criteria:**
- [ ] Mensaje máximo 3 frases, ≤ 160 caracteres para SMS
- [ ] Personalización mínima: `{name}` o genérico si no disponible
- [ ] Template configurable por nicho: `{niche_ooo_template}`
- [ ] Inclusión obligatoria de `next_available_window` en mensaje
- [ ] Envío vía ChannelAdapter (SMS/WhatsApp) con compliance check previo
- [ ] Actualización de lead a `status: "contacted_ooo"` tras envío exitoso
- [ ] Registro de timestamp y provider_message_id para trazabilidad
- [ ] Latencia P95 desde detección hasta envío: ≤ 30 segundos

**Pseudocódigo de envío:**
```
FUNCTION send_ooo_acknowledgment(lead: Lead_v1, template: OBJECT, time_decision: TimeDecision) → DeliveryResult:
    // Compliance check obligatorio antes de enviar (pre-send)
    compliance = compliance_guardrails.pre_send_check(
        lead = lead,
        content = {type: "text", body: "template_preview"},
        context = {campaign: "out_of_hours", region: CONFIG.region, hour: NOW().hour}
    )
    
    IF compliance.result == "blocked":
        LOG: "ooo_ack_blocked", lead_id: lead.lead_id, reason: compliance.reason
        UPDATE lead.status = "disqualified"
        EMIT_METRIC: "ooo_ack_blocked_compliance", tags: {reason: compliance.reason}
        RETURN {status: "blocked", reason: compliance.reason}
    
    // Construcción del mensaje
    name = lead.name OR "amigo"
    next_window = FORMAT_WINDOW(time_decision.next_available_window, lead.timezone)
    
    message_body = template.text
        .replace("{name}", name)
        .replace("{niche}", template.niche_context)
        .replace("{next_window}", next_window)
        .replace("{brand}", template.brand_name)
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
        UPDATE lead.status = "contacted_ooo"
        UPDATE lead.metadata.ooo_ack_sent_at = NOW()
        UPDATE lead.metadata.ooo_message_id = delivery.provider_message_id
        UPDATE lead.metadata.expected_followup = time_decision.next_available_window
        UPDATE lead.metadata.ooo_latency_ms = latency
        EMIT_METRIC: "ooo_ack_sent", value: 1, tags: {niche: template.niche, channel: CONFIG.channel}
        EMIT_HISTOGRAM: "ooo_ack_latency", value: latency, tags: {channel: CONFIG.channel}
    
    LOG: "ooo_ack_delivery", lead_id: lead.lead_id, status: delivery.status, latency_ms: latency
    RETURN delivery
```

**Template base (ejemplo nicho clínico):**
```
{
  "niche": "clinicas",
  "text": "Hola {name}, gracias por contactar a {brand}. Nuestro horario es L-V 9am-7pm. Te contactaremos el {next_window}. ¿En qué podemos ayudarte?",
  "expected_next_step": "nurturing o handoff",
  "positive_keywords": ["cita", "urgente", "dolor", "información", "horario"],
  "negative_keywords": ["no", "stop", "baja", "eliminar"]
}
```

---

### 5.3 FR-3: Nurturing Loop & FAQ Handling

**ID:** FR-3  
**Prioridad:** ALTA  
**Descripción:** El sistema debe responder preguntas frecuentes del lead fuera de horas mediante respuestas predefinidas y empáticas, sin iniciar proceso de venta ni calificación profunda.

**User Story:**  
Como sistema de nurturing, quiero responder preguntas comunes del lead fuera de horas con información útil y empática, manteniendo engagement sin comprometer al equipo de ventas ni violar compliance horario.

**Acceptance Criteria:**
- [ ] Catálogo de FAQs configurables por nicho: `faq_responses`
- [ ] Detección de intención básica: `faq_match`, `general_question`, `urgent_signal`, `other`
- [ ] Respuestas predefinidas para FAQs (no generadas por LLM en tiempo real)
- [ ] Detección de señales de urgencia que requieren escalación inmediata (ej: emergencia médica)
- [ ] Actualización de `ConversationState` con último intercambio
- [ ] Timeout de conversación: si no hay reply en 12h → marcar como `queued_for_handoff`
- [ ] Logging de todas las interacciones para auditoría y optimización

**Pseudocódigo de manejo de nurturing:**
```
FUNCTION handle_ooo_nurturing(message: NormalizedMessage, lead: Lead_v1, config: OBJECT) → NurturingResult:
    // Clasificación básica de intención
    intent = classify_ooo_intent(message.body, config.faq_keywords)
    // classify_ooo_intent returns: "faq_match" | "general_question" | "urgent_signal" | "other"
    
    SWITCH intent:
        CASE "faq_match":
            // Buscar respuesta predefinida
            faq = FIND_BEST_FAQ_MATCH(message.body, config.faq_responses)
            IF faq:
                response = faq.template
                    .replace("{brand}", config.brand_name)
                    .replace("{next_window}", FORMAT_WINDOW(lead.metadata.expected_followup, lead.timezone))
                channel_adapter.send_message(lead, {type: "text", body: response})
                LOG: "faq_answered", lead_id: lead.lead_id, faq_id: faq.id
                RETURN {status: "faq_responded", faq_id: faq.id}
            ELSE:
                // Fallback a respuesta genérica
                RETURN send_generic_nurturing(lead, config)
                
        CASE "urgent_signal":
            // Detectar si es emergencia real que requiere escalación inmediata
            IF is_true_emergency(message.body, config.urgent_keywords):
                // Escalación inmediata a humano, incluso fuera de horas
                enqueue_for_immediate_escalation(lead, message, reason: "emergency_detected")
                LOG: "emergency_escalated", lead_id: lead.lead_id
                RETURN {status: "escalated_emergency"}
            ELSE:
                // No es emergencia real, tratar como pregunta general
                RETURN send_generic_nurturing(lead, config)
                
        CASE "general_question":
            // Respuesta empática genérica
            RETURN send_generic_nurturing(lead, config)
            
        CASE "other":
            // Respuesta de mantenimiento de engagement
            RETURN send_engagement_nurturing(lead, config)
    
    LOG: "nurturing_processed", lead_id: lead.lead_id, intent: intent
    RETURN {status: "nurtured", next_action: "await_handoff"}

FUNCTION send_generic_nurturing(lead: Lead_v1, config: OBJECT) → NurturingResult:
    response = config.generic_nurturing_template
        .replace("{name}", lead.name OR "amigo")
        .replace("{next_window}", FORMAT_WINDOW(lead.metadata.expected_followup, lead.timezone))
        .replace("{brand}", config.brand_name)
    
    channel_adapter.send_message(lead, {type: "text", body: response})
    LOG: "generic_nurturing_sent", lead_id: lead.lead_id
    RETURN {status: "generic_nurturing_sent"}
```

---

### 5.4 FR-5: Queue Management & Handoff Scheduling

**ID:** FR-5  
**Prioridad:** ALTA  
**Descripción:** El sistema debe encolar leads para handoff al equipo de ventas en el próxima ventana laboral disponible, con priorización basada en señales de urgencia detectadas.

**User Story:**  
Como sistema de handoff, quiero encolar leads fuera de horas con prioridad adecuada y programar seguimiento automático para el equipo de ventas, para garantizar que ningún lead se pierda y el contexto nocturno se preserve.

**Acceptance Criteria:**
- [ ] Queue con priorización: `urgent`, `normal`, `low` basado en señales detectadas
- [ ] Programación automática de handoff para próxima ventana laboral configurable
- [ ] Actualización de `lead.status = "queued_for_handoff"` con metadata de contexto nocturno
- [ ] Notificación al equipo de ventas (email/Slack) al inicio del próximo día laboral
- [ ] Manejo de fallos de queue con fallback a CRM-native task
- [ ] Trazabilidad completa: queue_id correlacionado con trace_id

**Pseudocódigo de encolado:**
```
FUNCTION enqueue_for_handoff(lead: Lead_v1, conversation_context: OBJECT, config: OBJECT) → QueueResult:
    // Determinar prioridad basada en señales
    priority = determine_handoff_priority(conversation_context, config.priority_rules)
    // priority: "urgent" | "normal" | "low"
    
    // Calcular ventana de handoff
    handoff_window = CALCULATE_HANDOFF_WINDOW(
        current_time: NOW(),
        business_hours: config.business_hours,
        timezone: lead.timezone OR config.default_timezone,
        priority: priority  // urgent puede handoffear en próxima ventana, incluso si es mismo día
    )
    
    // Construir payload de handoff
    handoff_payload = {
        lead_id: lead.lead_id,
        priority: priority,
        scheduled_handoff_time: handoff_window,
        context_summary: build_context_summary(conversation_context),
        ooo_interaction_count: lead.metadata.ooo_interaction_count OR 0,
        urgent_signals: extract_urgent_signals(conversation_context),
        faqs_answered: extract_faq_ids(conversation_context),
        next_recommended_action: config.handoff_recommendations[priority]
    }
    
    // Encolar en sistema de queue
    queue_result = handoff_queue.enqueue(handoff_payload, {
        priority: priority,
        delay_until: handoff_window,
        retry_policy: config.queue_retry_policy
    })
    
    // Actualizar lead
    lead.status = "queued_for_handoff"
    lead.metadata.queue_id = queue_result.queue_id
    lead.metadata.scheduled_handoff = handoff_window
    lead.metadata.handoff_priority = priority
    crm_adapter.update_lead(lead.lead_id, {
        status: "queued_for_handoff",
        handoff_priority: priority,
        scheduled_followup: handoff_window,
        ooo_context: handoff_payload.context_summary
    })
    
    // Emitir métrica
    EMIT_METRIC: "lead_queued_for_handoff", value: 1, tags: {
        niche: lead.metadata.niche,
        priority: priority,
        source: lead.meta.original_source
    }
    
    LOG: "lead_queued", lead_id: lead.lead_id, queue_id: queue_result.queue_id, priority: priority
    RETURN {status: "queued", queue_id: queue_result.queue_id, scheduled_handoff: handoff_window}
```

---

### 5.5 FR-6: CRM Sync & Context Preservation

**ID:** FR-6  
**Prioridad:** ALTA  
**Descripción:** El sistema debe sincronizar el estado del lead y el contexto de la interacción nocturna con el CRM del socio para preservar información crítica para el handoff.

**User Story:**  
Como sistema de sincronización, quiero actualizar el CRM con el estado del lead y el contexto de la interacción fuera de horas, para que el equipo de ventas tenga toda la información necesaria al momento del handoff.

**Acceptance Criteria:**
- [ ] Actualización de `lead.status` a `queued_for_handoff` con timestamp
- [ ] Inclusión de contexto estructurado: preguntas respondidas, señales de urgencia, FAQs consultadas
- [ ] Campo dedicado para `ooo_context` en CRM (JSON o texto estructurado)
- [ ] Idempotencia: actualizaciones múltiples no crean duplicados
- [ ] Fallback graceful si CRM está indisponible (queue local + retry)
- [ ] Logging de todas las operaciones de sync para reconciliación

**Pseudocódigo de sincronización:**
```
FUNCTION sync_ooo_context_to_crm(lead: Lead_v1, interaction_context: OBJECT) → SyncResult:
    // Construir payload de actualización
    crm_update = {
        status: "queued_for_handoff",
        last_interaction: NOW(),
        channel: CONFIG.channel,
        ooo_metadata: {
            ack_sent_at: lead.metadata.ooo_ack_sent_at,
            interaction_count: lead.metadata.ooo_interaction_count,
            faqs_answered: interaction_context.faqs_answered OR [],
            urgent_signals: interaction_context.urgent_signals OR [],
            expected_followup: lead.metadata.expected_followup,
            priority: lead.metadata.handoff_priority
        },
        notes: build_crm_notes(interaction_context, lead)
    }
    
    // Intentar actualización en CRM
    start_time = NOW()
    sync_result = crm_adapter.update_lead(lead.lead_id, crm_update)
    latency = NOW() - start_time
    
    IF sync_result.success:
        LOG: "crm_sync_success", lead_id: lead.lead_id, latency_ms: latency
        EMIT_METRIC: "crm_sync_success", value: 1, tags: {source: "out_of_hours"}
        RETURN {status: "synced", latency_ms: latency}
    ELSE:
        // Fallback: enqueue para retry
        enqueue_crm_retry(lead.lead_id, crm_update, {
            max_retries: CONFIG.crm_sync_max_retries,
            backoff_strategy: "exponential"
        })
        LOG: "crm_sync_failed", lead_id: lead.lead_id, error: sync_result.error
        EMIT_METRIC: "crm_sync_failed", value: 1, tags: {source: "out_of_hours"}
        RETURN {status: "queued_for_retry", error: sync_result.error}
```

---

### 5.6 FR-7: Compliance & Hour-Based Enforcement

**ID:** FR-7  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe respetar automáticamente regulaciones de contacto por horario (TCPA, GDPR) y región, bloqueando envíos fuera de ventanas permitidas.

**User Story:**  
Como operador en región regulada, quiero que el sistema verifique automáticamente si un envío está permitido por horario y región antes de cada mensaje, para cumplir con la ley y proteger la reputación.

**Acceptance Criteria:**
- [ ] Configuración de ventanas permitidas por región: `allowed_contact_hours`
- [ ] Bloqueo automático de envíos fuera de ventana permitida, incluso si lead responde
- [ ] Logging de todos los intentos de envío bloqueados por compliance horario
- [ ] Fallback a queue para envío en próxima ventana permitida (no reintentar inmediatamente)
- [ ] Confirmación de opt-out procesada incluso fuera de horas (excepción compliance)
- [ ] Alertas configurables cuando se acerque a límite de envíos por ventana

**Pseudocódigo de enforcement horario:**
```
FUNCTION verify_hourly_compliance(lead: Lead_v1, content: OBJECT, context: OBJECT) → ComplianceDecision:
    // Obtener configuración de compliance por región
    compliance_config = compliance_guardrails.get_compliance_config(context.region)
    
    // Verificar si el tipo de mensaje está exento (ej: opt-out confirmation)
    IF content.type == "optout_confirmation":
        RETURN {allowed: true, reason: "exempt_optout_confirmation"}
    
    // Verificar ventana de contacto permitida
    current_time = NOW()
    timezone = lead.timezone OR compliance_config.default_timezone
    local_time = CONVERT_TO_TIMEZONE(current_time, timezone)
    
    allowed_hours = compliance_config.allowed_contact_hours[local_time.day_of_week]
    IF NOT allowed_hours:
        RETURN {
            allowed: false,
            reason: "no_allowed_hours_configured",
            next_allowed_window: CALCULATE_NEXT_ALLOWED_WINDOW(local_time, compliance_config)
        }
    
    current_minutes = local_time.hours * 60 + local_time.minutes
    start_minutes = PARSE_TIME(allowed_hours.start)
    end_minutes = PARSE_TIME(allowed_hours.end)
    
    IF current_minutes >= start_minutes AND current_minutes < end_minutes:
        RETURN {allowed: true, reason: "within_allowed_hours"}
    ELSE:
        RETURN {
            allowed: false,
            reason: current_minutes < start_minutes ? "before_allowed_hours" : "after_allowed_hours",
            next_allowed_window: CALCULATE_NEXT_ALLOWED_WINDOW(local_time, compliance_config)
        }
```

---

### 5.7 FR-8: Metrics Emission & Observability

**ID:** FR-8  
**Prioridad:** ALTA  
**Descripción:** El sistema debe emitir métricas estructuradas para medir performance de nurturing fuera de horas y handoff success, permitiendo optimización.

**User Story:**  
Como analista, quiero métricas en tiempo real de acknowledgment, engagement y handoff success por nicho y región, para identificar cuellos de botella y ajustar la estrategia de nurturing.

**Acceptance Criteria:**
- [ ] Métricas de funnel: `ooo_contact_detected`, `ack_sent`, `nurturing_response`, `lead_queued`, `handoff_completed`
- [ ] Métricas de calidad: `ack_latency_p95`, `engagement_rate`, `handoff_success_rate`, `compliance_block_rate`
- [ ] Segmentación por: `niche`, `channel`, `region`, `timezone`, `priority`
- [ ] Emisión asíncrona vía MetricsEmitter de Shared Components
- [ ] Correlación con trace_id para debugging end-to-end
- [ ] Dashboard-ready: formato compatible con Prometheus/Datadog

**Pseudocódigo de emisión de métricas:**
```
FUNCTION emit_ooo_metrics(lead: Lead_v1, event: ENUM, meta OBJECT):
    base_tags = {
        android: "out_of_hours",
        niche: lead.metadata.niche,
        channel: CONFIG.channel,
        region: CONFIG.region,
        timezone: lead.timezone OR CONFIG.default_timezone
    }
    
    SWITCH event:
        CASE "ooo_contact_detected":
            metrics_emitter.emit_counter("ooo_contact_detected", 1, base_tags)
            
        CASE "ack_sent":
            metrics_emitter.emit_counter("ooo_ack_sent", 1, base_tags)
            // Emitir latencia crítica
            IF meta.ack_latency_ms:
                metrics_emitter.emit_histogram("ooo_ack_latency", meta.ack_latency_ms, base_tags)
            
        CASE "nurturing_response":
            metrics_emitter.emit_counter("ooo_nurturing_sent", 1, {
                ...base_tags,
                response_type: meta.response_type  // faq, generic, engagement
            })
            
        CASE "lead_queued":
            metrics_emitter.emit_counter("ooo_lead_queued", 1, {
                ...base_tags,
                priority: meta.priority
            })
            // Calcular engagement rate si tenemos ack_sent del mismo timezone
            engagement_rate = calculate_engagement_rate(base_tags.timezone, time_window: "24h")
            metrics_emitter.emit_gauge("ooo_engagement_rate", engagement_rate, base_tags)
            
        CASE "handoff_completed":
            metrics_emitter.emit_counter("ooo_handoff_completed", 1, base_tags)
            // Calcular handoff success rate
            handoff_rate = calculate_handoff_success_rate(base_tags.niche, time_window: "7d")
            metrics_emitter.emit_gauge("ooo_handoff_success_rate", handoff_rate, base_tags)
            
        CASE "compliance_blocked":
            metrics_emitter.emit_counter("ooo_compliance_blocked", 1, base_tags)
```

---

## 6. Non-Functional Requirements

### 6.1 Performance

| Métrica | Target | Medición |
|---------|--------|----------|
| Latencia detección → acknowledgment | P95 ≤ 30s | End-to-end tracing |
| Throughput de detección | ≥ 200 leads/minuto | Sustained load |
| Tiempo de encolado | < 2 segundos desde decisión | P95 |
| CRM sync latency | < 3 segundos desde queue | P95 |

### 6.2 Reliability

| Métrica | Target | Estrategia |
|---------|--------|------------|
| Ack delivery rate | ≥ 98% | Retry + fallback channel |
| FAQ match accuracy | ≥ 90% | Keyword-based matching + fallback |
| Queue durability | 100% | Persistent queue + DLQ |
| Compliance enforcement | 100% | Pre-send check obligatorio, fail-safe |

### 6.3 Security

| Requisito | Implementación |
|-----------|----------------|
| PII en logs | Masking automático: phone → `+52***`, email → `u***@***` |
| Credenciales de adapters | Secrets management, nunca en código |
| Webhook verification | HMAC signature para replies de WhatsApp |
| Timezone data integrity | Validación IANA timezone strings |

### 6.4 Scalability

| Dimensión | Estrategia |
|-----------|------------|
| Horizontal | Stateless processing, queue-based ingestion |
| Multi-niche | Configuración por `niche_id`, reglas aisladas |
| Multi-partner | Aislamiento por `partner_id` en metadata |
| Multi-timezone | Timezone-aware detection, config por región |

---

## 7. Technical Specifications

### 7.1 Contratos Extendidos (sobre ROYA Shared Components)

#### TimeDecision Schema
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "TimeDecision",
  "type": "object",
  "required": ["is_business_hours", "reason"],
  "properties": {
    "is_business_hours": {"type": "boolean"},
    "reason": {
      "type": "string",
      "enum": [
        "within_hours",
        "before_hours",
        "after_hours",
        "weekend",
        "holiday",
        "no_hours_configured"
      ]
    },
    "next_available_window": {
      "type": "string",
      "format": "date-time",
      "description": "Próxima ventana laboral disponible (ISO-8601)"
    }
  }
}
```

#### OOOContext Schema (para CRM sync)
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "OOOContext",
  "type": "object",
  "properties": {
    "ack_sent_at": {"type": "string", "format": "date-time"},
    "interaction_count": {"type": "integer", "minimum": 0},
    "faqs_answered": {
      "type": "array",
      "items": {"type": "string"}
    },
    "urgent_signals": {
      "type": "array",
      "items": {"type": "string"}
    },
    "expected_followup": {"type": "string", "format": "date-time"},
    "priority": {"enum": ["urgent", "normal", "low"]},
    "conversation_summary": {"type": "string"}
  }
}
```

### 7.2 Configuración por Nicho (Ejemplo: Clínicas)

```yaml
# config/niches/clinicas_ooo_v1.yaml
niche_id: clinicas
channel: sms  # o whatsapp
language: es-MX

out_of_hours_acknowledgment:
  template: "Hola {name}, gracias por contactar a {brand}. Nuestro horario es L-V 9am-7pm. Te contactaremos el {next_window}. ¿En qué podemos ayudarte?"
  next_window_format: "lunes a las 9:00 AM"  # formato legible
  brand_name: "Clínica Salud"

faq_responses:
  - id: "horarios"
    keywords: ["horario", "abren", "cierran", "atención", "hora"]
    template: "Atendemos de lunes a viernes de 9am a 7pm, sábados 9am a 2pm. Te contactaremos el {next_window} para agendar tu cita."
  
  - id: "ubicacion"
    keywords: ["dónde", "ubicación", "dirección", "llegar"]
    template: "Estamos en [Dirección]. Te enviaremos un mapa y detalles de llegada cuando te contactemos el {next_window}."
  
  - id: "seguros"
    keywords: ["seguro", "aseguradora", "cobertura", "plan"]
    template: "Aceptamos múltiples aseguradoras. Nuestro equipo te confirmará cobertura específica el {next_window}."

urgent_keywords:
  - "emergencia"
  - "urgente"
  - "dolor intenso"
  - "sangrado"
  - "desmayo"
  - "no puedo respirar"

priority_rules:
  urgent_signals: ["emergencia", "dolor intenso", "sangrado"]
  normal_signals: ["cita", "información", "horario"]
  low_signals: ["gracias", "ok", "entendido"]

business_hours:
  timezone: "America/Mexico_City"
  monday_friday: {start: "09:00", end: "19:00"}
  saturday: {start: "09:00", end: "14:00"}
  sunday: null  # cerrado
  holidays: ["2026-01-01", "2026-02-05", "2026-03-21"]  # lista configurable

compliance:
  region: MX
  allowed_contact_hours:
    monday_friday: {start: "08:00", end: "21:00"}
    saturday: {start: "09:00", end: "20:00"}
    sunday: null
  optout_keywords: ["STOP", "BAJA", "CANCELAR", "NO MÁS"]
  optout_confirmation: "Has sido dado de baja. Para reactivar, envía ALTA a este número."
  max_messages_per_ooo_session: 3
```

### 7.3 Flujo de Estado del Lead (State Machine)

```
[NEW] 
   ↓ (contacto detectado)
[NEW] → is_out_of_hours()?
   ├─ true → send_ooo_ack() → [CONTACTED_OOO]
   └─ false → [HANDOFF_TO_SPEED_TO_LEAD]  // fuera de scope de este Android

[CONTACTED_OOO] → handle_reply()
   ├─ faq_match → send_faq_response() → [CONTACTED_OOO]
   ├─ urgent_signal → is_true_emergency()?
   │   ├─ true → escalate_immediately() → [ESCALATED_EMERGENCY]
   │   └─ false → send_generic_nurturing() → [CONTACTED_OOO]
   ├─ general_question → send_generic_nurturing() → [CONTACTED_OOO]
   └─ other/no_reply → timeout(12h) → [QUEUED_FOR_HANDOFF]

[QUEUED_FOR_HANDOFF] → wait_for_business_hours()
   ↓ (próxima ventana laboral)
[QUEUED_FOR_HANDOFF] → trigger_handoff_notification() → [HANDED_OFF]

[ESCALATED_EMERGENCY] → notify_human_immediately() → [CLOSED]

[HANDED_OFF] → sync_to_crm() + notify_sales_team() → [CLOSED]

Timeouts:
- CONTACTED_OOO → no reply in 12h → [QUEUED_FOR_HANDOFF]
- QUEUED_FOR_HANDOFF → business_hours reached → [HANDED_OFF]
```

---

## 8. Implementation Notes

### 8.1 Stack Reference (n8n como ejemplo)

| Componente | Nodo n8n | Configuración |
|------------|----------|---------------|
| Time Detection | Code (JavaScript) + Luxon/Date-fns | Timezone-aware logic |
| Ack Delivery | HTTP Request (Twilio/Meta) + Compliance Check | Rate limiting, retry, timeout 5s |
| FAQ Matching | Code (keyword matching) + Fallback | Configurable keyword sets |
| Queue Management | Redis node / Code + Delay node | Priority queue, TTL |
| CRM Sync | HTTP Request (HighLevel/Salesforce) | Idempotent upsert, fallback |
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
# Out Of Hours Engine Config
OOO_DEFAULT_NICHE=clinicas
OOO_MAX_NURTURING_ROUNDS=3
OOO_REPLY_TIMEOUT_HOURS=12
OOO_HANDOFF_PRIORITY_URGENT=true
OOO_ACK_TIMEOUT_MS=5000

# Timezone & Business Hours
DEFAULT_TIMEZONE=America/Mexico_City
BUSINESS_HOURS_CONFIG_PATH=./config/business_hours.yaml
HOLIDAYS_LIST_URL=https://.../holidays

# Channel Config
CHANNEL_PROVIDER=twilio|meta_whatsapp
CHANNEL_PHONE_NUMBER=+52...
CHANNEL_API_KEY=xxx

# CRM Config
CRM_PROVIDER=highlevel|salesforce|hubspot
CRM_API_KEY=xxx
CRM_WEBHOOK_SECRET=xxx
CRM_OOO_CONTEXT_FIELD=ooo_metadata

# Compliance
COMPLIANCE_REGION=MX|US|EU
COMPLIANCE_ALLOWED_HOURS_CONFIG_PATH=./config/compliance_hours.yaml
COMPLIANCE_OPTOUT_LIST_URL=https://.../optout
COMPLIANCE_MAX_MESSAGES_PER_OOO_SESSION=3

# Queue & Handoff
QUEUE_PROVIDER=redis|sqs|in_memory
HANDOFF_NOTIFICATION_CHANNEL=email|slack|sms
HANDOFF_NOTIFICATION_RECIPIENTS=sales_team@example.com

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
| **TC-OOO-01** | Contacto dentro de horas | Timestamp dentro de business_hours | Handoff a SpeedToLead (fuera de scope) |
| **TC-OOO-02** | Contacto fuera de horas (weekday night) | Timestamp 22:00 weekday | Ack enviado, lead en `contacted_ooo` |
| **TC-OOO-03** | Contacto fin de semana | Timestamp Saturday 10am (si cerrado) | Ack enviado, next_window = Monday 9am |
| **TC-OOO-04** | FAQ match: horarios | Reply: "¿A qué hora abren?" | Respuesta de FAQ enviada, no venta |
| **TC-OOO-05** | Señal de urgencia real | Reply: "Tengo dolor en el pecho" | Escalación inmediata a humano |
| **TC-OOO-06** | Señal de urgencia falsa | Reply: "Necesito cita urgente mañana" | Nurturing genérico, no escalación |
| **TC-OOO-07** | Timeout sin reply | Lead en `contacted_ooo`, 12h sin reply | Lead en `queued_for_handoff` |
| **TC-OOO-08** | Compliance block por hora | Intento de envío a las 23:00 en MX | Envío bloqueado, queued para 8:00 |
| **TC-OOO-09** | Handoff scheduling | Lead queued, Monday 9:00 reached | Notificación a ventas, lead en `handed_off` |
| **TC-OOO-10** | CRM sync con contexto | Lead con 2 FAQs respondidas | CRM actualizado con ooo_context completo |

### 9.2 Test Execution

```bash
# Unit tests (componentes individuales)
npm test -- unit/time-detection/
npm test -- unit/faq-matching/
npm test -- unit/compliance-hours/
npm test -- unit/queue-management/

# Integration tests (flujos end-to-end)
npm test -- integration/ooo-ack-to-nurturing/
npm test -- integration/nurturing-to-handoff/

# Compliance tests (regulaciones)
npm test -- compliance/hourly-limits/
npm test -- compliance/optout-outside-hours/

# Timezone tests (multi-región)
npm test -- timezone/mexico-city/
npm test -- timezone/new-york/
npm test -- timezone/europe-madrid/

# Load tests (escalabilidad)
npm test -- load/ooo-ingestion-200-leads/
npm test -- load/concurrent-nurturing-50/
```

---

## 10. Success Metrics (Técnicos y de Negocio)

| Métrica | Fórmula | Target | Frecuencia |
|---------|---------|--------|------------|
| Ack latency P95 | percentile_95(ack_sent - detected) | ≤ 30s | Diario |
| FAQ match rate | faq_matched / nurturing_requests | ≥ 90% | Por nicho |
| Engagement rate | replies / acks_sent | ≥ 40% | Por timezone |
| Handoff success rate | handed_off / queued | ≥ 95% | Diario |
| Compliance adherence | compliant_sends / total_sends | 100% | Diario |
| Emergency detection accuracy | true_emergencies_escalated / urgent_signals | ≥ 99% | Semanal |
| CRM sync success | successful_syncs / updates | ≥ 99.9% | Diario |
| Lead retention (next-day) | leads_contacted_next_day / queued | ≥ 70% | Por nicho |

---

## 11. Deployment Checklist

### Pre-Deployment
- [ ] Configuración de nicho validada (`config/niches/{niche}_ooo_v1.yaml`)
- [ ] Business hours configurado por socio con timezone correcto
- [ ] Holidays list actualizada para región objetivo
- [ ] ChannelAdapter configurado y testeado (SMS/WhatsApp)
- [ ] CRMAdapter configurado con campo `ooo_context` disponible
- [ ] Compliance guardrails cargados con allowed_contact_hours por región
- [ ] Queue system configurado (Redis/SQS) con TTL adecuado
- [ ] Métricas backend configurado y recibiendo datos de latency
- [ ] Logging estructurado habilitado con trace_id propagation
- [ ] Tests de timezone pasando para regiones objetivo
- [ ] Plan de rollback documentado para fallos de handoff

### Post-Deployment
- [ ] Monitorizar ack latency P95 por primeras 100 contactos OOO
- [ ] Validar que compliance blocks se aplican en horarios correctos
- [ ] Confirmar que handoffs se disparan en próxima ventana laboral
- [ ] Verificar que CRM recibe ooo_context completo
- [ ] Revisar logs para detectar PII no masked
- [ ] Documentar cualquier desviación de targets de engagement

---

## 12. Version History

| Versión | Fecha | Cambios | Autor |
|---------|-------|---------|-------|
| 1.0 | 2026-02-20 | Especificación inicial de Out Of Hours Engine | O.A. Perez Garrido |

---

## 13. Appendices

### Appendix A: Template Library por Nicho

| Nicho | OOO Ack Template | FAQ Examples | Urgent Keywords |
|-------|-----------------|--------------|----------------|
| **Clinicas** | "Hola {name}, gracias por contactar a {brand}. Nuestro horario es L-V 9am-7pm. Te contactaremos el {next_window}. ¿En qué podemos ayudarte?" | horarios, ubicación, seguros | emergencia, dolor intenso, sangrado, desmayo |
| **Viajes** | "¡Hola {name}! {brand} está cerrado ahora. Te enviaremos opciones de viaje el {next_window}. ¿Ya tienes destino en mente?" | destinos, precios, documentos | viaje urgente, emergencia, cancelación |
| **Solar** | "Hola {name}, gracias por tu interés en {brand}. Nuestro equipo técnico está disponible L-V 8am-6pm. Te contactaremos el {next_window} para tu evaluación." | costos, instalación, incentivos | falla sistema, sin energía, emergencia eléctrica |
| **Legal** | "Hola {name}, {brand} recibe consultas L-V 9am-6pm. Tu mensaje ha sido recibido y te contactaremos el {next_window}. ¿Es un asunto urgente?" | tipos de caso, honorarios, proceso | demanda inminente, detención, plazo venciendo |

### Appendix B: Timezone Handling Patterns

```yaml
# Ejemplo: conversión segura de timezone
timezone_utils:
  validate_timezone: |
    FUNCTION is_valid_iana_timezone(tz_string: STRING) → BOOLEAN:
        TRY:
            DateTime.fromISO("2026-01-01T00:00:00", {zone: tz_string})
            RETURN true
        CATCH:
            RETURN false
  
  convert_with_fallback: |
    FUNCTION convert_timestamp_safe(timestamp: ISO-8601, preferred_tz: STRING, fallback_tz: STRING) → DateTime:
        IF is_valid_iana_timezone(preferred_tz):
            RETURN DateTime.fromISO(timestamp, {zone: preferred_tz})
        ELSE:
            LOG: "invalid_timezone", preferred: preferred_tz, using: fallback_tz
            RETURN DateTime.fromISO(timestamp, {zone: fallback_tz})
  
  format_human_window: |
    FUNCTION format_next_window(dt: DateTime, locale: STRING) → STRING:
        // Ejemplo: "lunes a las 9:00 AM"
        day_name = dt.toLocaleString({weekday: "long"}, {locale})
        time_str = dt.toLocaleString({hour: "numeric", minute: "2-digit"}, {locale})
        RETURN `${day_name} a las ${time_str}`
```

### Appendix C: Error Codes & Fallbacks

| Código | Descripción | Fallback | Alerta |
|--------|-------------|----------|--------|
| `TIMEZONE_INVALID` | Timezone del lead no es IANA válido | Fallback a timezone del socio | Si > 5% de leads con timezone inválido |
| `BUSINESS_HOURS_MISSING` | No hay configuración de horario para día | Tratar como "fuera de horas" | Si configuración incompleta detectada |
| `FAQ_NO_MATCH` | Pregunta no coincide con FAQs configuradas | Respuesta genérica de nurturing | Si > 30% de preguntas sin match |
| `URGENT_FALSE_POSITIVE` | Señal de urgencia detectada pero no es emergencia | Nurturing genérico, no escalación | Si > 10% de falsos positivos |
| `QUEUE_FULL` | Queue de handoff alcanzó límite | Fallback a CRM-native task | Inmediata si queue > 90% capacity |
| `CRM_SYNC_FAILED` | Fallo al actualizar contexto OOO en CRM | Retry 3x, luego DLQ + alerta | Inmediata si crítico |
| `COMPLIANCE_HOUR_BLOCK` | Envío bloqueado por horario no permitido | Queue para próxima ventana permitida | Si bloqueos > 5% de intentos |

### Appendix D: Handoff a Humanos (Escenarios)

| Escenario | Trigger | Acción de Handoff | Información Entregada |
|-----------|---------|------------------|----------------------|
| **Emergencia real detectada** | `is_true_emergency() == true` | Notificación inmediata a humano (SMS/email) | Full trace, urgent signals, raw message, lead contact |
| **Lead con alta prioridad** | `priority == "urgent"` | Handoff en próxima ventana, notificación prioritaria | Lead data, ooo_context, FAQs answered, next recommended action |
| **Pregunta fuera de scope** | `intent == "other"` + 3 nurturing rounds | Escalar a "Clarification Queue" | Conversation history, last question, attempted FAQs |
| **Fallo técnico repetido** | 3 fallos consecutivos en mismo lead | Marcar como `needs_review` | Error log, retry history, raw payloads |
| **Opt-out durante OOO** | Reply contiene keyword de opt-out | Procesar opt-out inmediatamente (excepción compliance) | Opt-out timestamp, channel, confirmation sent |

### Appendix E: Configuración Multi-Timezone (Ejemplo)

```yaml
# config/timezones/global_v1.yaml
supported_timezones:
  - "America/Mexico_City"
  - "America/New_York"
  - "Europe/Madrid"
  - "Asia/Tokyo"

default_fallback: "UTC"

business_hours_by_region:
  MX:
    timezone: "America/Mexico_City"
    monday_friday: {start: "09:00", end: "19:00"}
    saturday: {start: "09:00", end: "14:00"}
    sunday: null
  
  US:
    timezone: "America/New_York"
    monday_friday: {start: "09:00", end: "17:00"}
    saturday: null
    sunday: null
  
  EU:
    timezone: "Europe/Madrid"
    monday_friday: {start: "09:00", end: "18:00"}
    saturday: null
    sunday: null

compliance_hours_by_region:
  MX:
    allowed_contact: {start: "08:00", end: "21:00"}
    quiet_hours: {start: "21:00", end: "08:00"}
  
  US:
    allowed_contact: {start: "08:00", end: "20:00"}  # TCPA guidelines
    quiet_hours: {start: "20:00", end: "08:00"}
  
  EU:
    allowed_contact: {start: "09:00", end: "20:00"}  # GDPR considerations
    quiet_hours: {start: "20:00", end: "09:00"}
```

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
> Este SPEC define el tercer Android de ROYA: el motor que convierte contactos fuera de horas en oportunidades preservadas, sin violar compliance ni perder engagement.  
> Cada componente debe reutilizar ROYA_Shared_Components_v1 — nada de duplicar lógica.  
> La clave del éxito no es la IA, es el **nurturing empático** + **compliance horario estricto** + **handoff determinista**.  
> Si algo no encaja en este spec, pregunta: "¿Esto ayuda a mantener leads fuera de horas de forma compliant y medible?" antes de implementarlo.