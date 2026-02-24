# 📄 Product Requirements Document (PRD) Técnico

## WF_DocumentCollection_Engine_v1
**Versión:** 1.0  
**Fecha:** 20 de febrero de 2026  
**Estado:** Especificación para implementación  
**Dominio:** cxEngine / Document Collection Android  
**Stack referencia:** Agnóstico (n8n/Make/Zapier + LLM + SMS/WhatsApp + CRM + Storage)

---

## 1. Overview Técnico

### 1.1 Propósito del sistema
Motor de automatización para la solicitud, seguimiento y validación de documentación final en procesos de venta, diseñado para reducir fricción operativa, acelerar milestones de pago y garantizar cumplimiento regulatorio sin intervención humana repetitiva.

### 1.2 Arquitectura de alto nivel
```
Trigger de Milestone → Document Request → Follow-up Loop → Validation → CRM Sync → Payment Trigger
                              ↓
                    [cxEngine Shared Components]
                    - Lead Schema v1
                    - ChannelAdapter
                    - ComplianceGuardrails
                    - TraceEmitter
                    - MetricsEmitter
```

### 1.3 Principios técnicos fundamentales
- **Milestone-driven:** Cada solicitud está vinculada a un evento de negocio verificable
- **Follow-up inteligente:** Recordatorios contextuales, no spam genérico
- **Validación estructural:** Chequeo de completitud antes de marcar como "entregado"
- **Compliance documental:** Respeto a regulaciones de manejo de datos sensibles
- **Auditabilidad completa:** Trazabilidad de cada documento desde solicitud hasta aceptación

---

## 2. Problem Statement (Técnico)

### 2.1 Problema actual
La recolección de documentación final en procesos de venta representa un cuello de botella operativo debido a:
- **Seguimiento manual:** Equipos dedican horas a recordar leads que envíen documentos
- **Validación inconsistente:** Criterios subjetivos para determinar si un documento está "completo"
- **Pérdida de momentum:** Leads abandonan el proceso por fricción en la entrega
- **Riesgo de compliance:** Manejo inadecuado de datos sensibles (PII, financieros, legales)
- **Milestones no triggerados:** Pagos por envío/revisión se retrasan por falta de confirmación automática

### 2.2 Impacto técnico
- Tiempo promedio de cierre extendido 3-7 días por retrasos documentales
- Costo operativo de seguimiento manual: ≥ $15/lead en tiempo humano
- Tasa de abandono en etapa documental: ≥ 25% en promedio
- Exposición legal por manejo inadecuado de documentos sensibles
- Ingresos por milestones no reconocidos por falta de confirmación automática

---

## 3. Technical Objectives

### 3.1 Objetivos SMART

| ID | Objetivo | Métrica | Target |
|----|----------|---------|--------|
| **TO-1** | Reducir tiempo de recolección documental | Avg. collection time (trigger → accepted) | ≤ 48 horas |
| **TO-2** | Aumentar tasa de completitud documental | Completion rate (submitted / requested) | ≥ 85% |
| **TO-3** | Minimizar intervención humana en seguimiento | Human touch rate (touches / leads) | ≤ 10% |
| **TO-4** | Garantizar compliance en manejo de documentos | Compliance adherence rate | 100% |
| **TO-5** | Escalar a 300 solicitudes/hora sin degradación | Throughput sustained | ≥ 300 req/hr |

### 3.2 Requerimientos de calidad (NFRs)

| NFR | Descripción | Target |
|-----|-------------|--------|
| **NFR-1** | Disponibilidad del flujo de recolección | 99.5% uptime |
| **NFR-2** | Latencia de notificación (trigger → first request) | P95 ≤ 5 minutos |
| **NFR-3** | Tolerancia a fallos de upload/validación | Retry + fallback + DLQ |
| **NFR-4** | Retención de trazas para auditoría documental | 7 años (regulatorio) |
| **NFR-5** | Consistencia cross-document type | ≥ 99% validación correcta |

---

## 4. Scope Técnico

### 4.1 In Scope

| Componente | Descripción | Responsabilidad |
|------------|-------------|-----------------|
| **Milestone Trigger** | Detecta evento de negocio que requiere documentación | Trigger/Logic |
| **Document Request Builder** | Construye solicitud personalizada por tipo de documento | Comunicación |
| **Follow-up Scheduler** | Programa recordatorios contextuales con backoff inteligente | Estado |
| **Submission Handler** | Captura y normaliza documentos entrantes (link, archivo, texto) | Ingestión |
| **Validation Engine** | Evalúa completitud estructural de documentos contra reglas | Decisión |
| **CRM Sync** | Actualiza estado del lead y campos documentales en CRM externo | Sincronización |
| **Payment Trigger** | Emite señal para milestone payment cuando documento es aceptado | Integración |
| **Compliance Enforcement** | Validación de consentimiento, retención y masking de PII | Seguridad legal |
| **Metrics & Logging** | Emisión de métricas de completitud y tiempo de ciclo | Observabilidad |

### 4.2 Out of Scope

| Componente | Razón de exclusión |
|------------|-------------------|
| OCR o extracción de contenido de documentos | Fuera de scope de v1; validación estructural solamente |
| Firma electrónica o notariado | Integración con proveedores externos en v2 |
| Almacenamiento permanente de documentos | Solo metadata y referencias; storage externo del socio |
| Generación automática de documentos | Solo solicitud y validación, no creación |
| Soporte para canales voice/llamada | Solo SMS/WhatsApp/email en v1 |
| Machine Learning para detección de fraude | Reglas explícitas en v1 |

---

## 5. Functional Requirements

### 5.1 FR-1: Milestone Detection & Trigger

**ID:** FR-1  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe detectar eventos de negocio que requieren documentación y triggerar el flujo de recolección.

**User Story:**  
Como sistema de recolección documental, quiero detectar automáticamente cuando un lead alcanza un milestone que requiere documentación, para iniciar el proceso sin intervención humana.

**Acceptance Criteria:**
- [ ] Soporte para triggers desde CRM: `status_change`, `stage_advance`, `custom_event`
- [ ] Configuración de mapeo milestone → documento requerido: `milestone_config`
- [ ] Validación de prerequisitos antes de triggerar (ej: lead debe estar `qualified`)
- [ ] Generación de `document_request_id` único para trazabilidad
- [ ] Actualización de `lead.status` a `awaiting_documents`
- [ ] Logging del trigger para auditoría y debugging
- [ ] Throughput: ≥ 300 triggers/minuto

**Pseudocódigo de detección:**
```
FUNCTION detect_document_milestone(lead: Lead_v1, event: OBJECT, config: OBJECT) → DocumentRequest OR NULL:
    // Verificar que el lead está en estado válido
    IF lead.status NOT IN ["qualified", "converted", "pending_docs"]:
        LOG: "invalid_state_for_docs", lead_id: lead.lead_id, status: lead.status
        RETURN NULL
    
    // Buscar configuración de milestone
    milestone_config = FIND_MILESTONE_CONFIG(event.type, lead.metadata.niche)
    IF milestone_config == NULL:
        LOG: "no_milestone_config", event_type: event.type, niche: lead.metadata.niche
        RETURN NULL
    
    // Verificar prerequisitos
    IF NOT check_prerequisites(lead, milestone_config.prerequisites):
        LOG: "prerequisites_not_met", lead_id: lead.lead_id, missing: milestone_config.prerequisites
        RETURN NULL
    
    // Construir solicitud de documento
    request = {
        document_request_id: GENERATE_UUID(),
        lead_id: lead.lead_id,
        document_type: milestone_config.document_type,
        required_fields: milestone_config.required_fields,
        validation_rules: milestone_config.validation_rules,
        deadline: CALCULATE_DEADLINE(milestone_config.sla_hours),
        created_at: NOW(),
        status: "requested",
        metadata: {
            trigger_event: event.type,
            trigger_timestamp: event.timestamp,
            niche: lead.metadata.niche,
            partner_id: lead.metadata.partner_id
        }
    }
    
    // Actualizar lead
    lead.status = "awaiting_documents"
    lead.metadata.pending_document_request = request.document_request_id
    crm_adapter.update_lead(lead.lead_id, {
        status: "awaiting_documents",
        pending_document: request.document_type,
        document_deadline: request.deadline
    })
    
    LOG: "document_request_created", request_id: request.document_request_id, type: request.document_type
    EMIT_METRIC: "document_request_triggered", value: 1, tags: {type: request.document_type, niche: lead.metadata.niche}
    RETURN request
```

---

### 5.2 FR-2: Document Request Delivery

**ID:** FR-2  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe enviar la solicitud inicial de documento con instrucciones claras y enlace/método de entrega.

**User Story:**  
Como sistema de recolección, quiero enviar una solicitud de documento clara y personalizada que indique qué se necesita, cómo enviarlo y cuándo, para maximizar la tasa de completitud sin fricción.

**Acceptance Criteria:**
- [ ] Mensaje máximo 3 frases + enlace/método de entrega, ≤ 160 caracteres para SMS (sin enlace)
- [ ] Personalización: `{name}`, `{document_type}`, `{deadline}`, `{delivery_method}`
- [ ] Template configurable por tipo de documento: `{doc_request_templates}`
- [ ] Inclusión obligatoria de método de entrega: link de upload, email, WhatsApp, etc.
- [ ] Envío vía ChannelAdapter (SMS/WhatsApp/email) con compliance check previo
- [ ] Actualización de `request.status = "sent"` tras envío exitoso
- [ ] Registro de timestamp y provider_message_id para trazabilidad
- [ ] Latencia P95 desde trigger hasta envío: ≤ 5 minutos

**Pseudocódigo de envío:**
```
FUNCTION send_document_request(request: DocumentRequest, lead: Lead_v1, config: OBJECT) → DeliveryResult:
    // Compliance check obligatorio antes de enviar
    compliance = compliance_guardrails.pre_send_check(
        lead = lead,
        content = {type: "text", body: "template_preview"},
        context = {campaign: "document_collection", region: CONFIG.region}
    )
    
    IF compliance.result == "blocked":
        LOG: "doc_request_blocked", lead_id: lead.lead_id, reason: compliance.reason
        UPDATE request.status = "blocked"
        EMIT_METRIC: "doc_request_blocked_compliance", tags: {reason: compliance.reason}
        RETURN {status: "blocked", reason: compliance.reason}
    
    // Construcción del mensaje
    name = lead.name OR "estimado cliente"
    doc_name = config.document_names[request.document_type] OR request.document_type
    deadline = FORMAT_DEADLINE(request.deadline, lead.timezone)
    delivery_method = config.delivery_methods[request.document_type]
    
    message_body = config.request_template
        .replace("{name}", name)
        .replace("{document_name}", doc_name)
        .replace("{deadline}", deadline)
        .replace("{delivery_method}", delivery_method)
        .replace("{brand}", config.brand_name)
        .trim()
    
    // Validación de longitud para SMS
    IF CONFIG.channel == "sms" AND message_body.length > 160:
        message_body = message_body.substring(0, 157) + "..."
    
    // Envío vía adapter con timeout
    start_time = NOW()
    delivery = channel_adapter.send_message(
        lead = lead,
        content = {
            type: "text",
            body: message_body,
            attachment: delivery_method.link OR NULL  // si aplica
        },
        timeout_ms: 5000
    )
    latency = NOW() - start_time
    
    // Actualización de estado y métricas
    IF delivery.status == "sent" OR delivery.status == "queued":
        UPDATE request.status = "sent"
        UPDATE request.metadata.first_request_sent_at = NOW()
        UPDATE request.metadata.request_message_id = delivery.provider_message_id
        UPDATE request.metadata.request_latency_ms = latency
        EMIT_METRIC: "doc_request_sent", value: 1, tags: {type: request.document_type, channel: CONFIG.channel}
        EMIT_HISTOGRAM: "doc_request_latency", value: latency, tags: {channel: CONFIG.channel}
    
    LOG: "doc_request_delivery", request_id: request.document_request_id, status: delivery.status, latency_ms: latency
    RETURN delivery
```

**Template base (ejemplo: formulario DSAR):**
```
{
  "document_type": "dsar_form",
  "request_template": "Hola {name}, para continuar con tu solicitud en {brand}, necesitamos el formulario DSAR completado. Envíalo aquí: {delivery_method}. Fecha límite: {deadline}. ¿Necesitas ayuda?",
  "required_fields": ["full_name", "id_number", "signature", "date"],
  "validation_rules": {
    "signature": "must_be_present",
    "date": "must_be_within_30_days",
    "id_number": "must_match_format"
  },
  "delivery_methods": {
    "link": "https://secure.{brand}.com/upload/dsar",
    "email": "docs@{brand}.com",
    "whatsapp": "reply_with_photo"
  }
}
```

---

### 5.3 FR-3: Follow-up Scheduler & Contextual Reminders

**ID:** FR-3  
**Prioridad:** ALTA  
**Descripción:** El sistema debe programar y enviar recordatorios contextuales si el documento no es entregado en el plazo esperado.

**User Story:**  
Como sistema de seguimiento, quiero enviar recordatorios empáticos y útiles que aumenten la probabilidad de entrega sin ser intrusivos, adaptando el mensaje según el tiempo transcurrido y el tipo de documento.

**Acceptance Criteria:**
- [ ] Configuración de intervalos de follow-up por tipo de documento: `followup_schedule`
- [ ] Mensajes progresivamente más directos pero siempre empáticos
- [ ] Inclusión de "última oportunidad" antes de escalar a humano
- [ ] Respeto a límites de frecuencia por canal (compliance)
- [ ] Pausa automática si lead responde o entrega documento
- [ ] Actualización de `request.followup_count` y `request.last_followup_sent_at`
- [ ] Logging de cada follow-up para auditoría

**Pseudocódigo de scheduling:**
```
FUNCTION schedule_followups(request: DocumentRequest, config: OBJECT) → FollowupPlan:
    // Obtener configuración de follow-up por tipo de documento
    followup_config = config.followup_schedules[request.document_type]
    IF followup_config == NULL:
        LOG: "no_followup_config", doc_type: request.document_type
        RETURN {enabled: false}
    
    // Calcular tiempos de follow-up basados en deadline
    followups = []
    FOR EACH interval IN followup_config.intervals:
        // interval: {offset_hours: 24, message_template: "gentle", max_count: 3}
        send_time = request.created_at + interval.offset_hours * 1_HOUR
        IF send_time < request.deadline:  // No enviar follow-up después del deadline
            followups.append({
                scheduled_time: send_time,
                template: interval.message_template,
                max_attempts: interval.max_count,
                attempt: 0
            })
    
    // Guardar plan en request
    request.followup_plan = {
        enabled: true,
        next_followup: followups[0].scheduled_time IF followups.length > 0 ELSE NULL,
        total_planned: followups.length,
        templates: followups.map(f => f.template)
    }
    
    LOG: "followup_plan_created", request_id: request.document_request_id, count: followups.length
    RETURN {enabled: true, plan: request.followup_plan}

FUNCTION send_contextual_followup(request: DocumentRequest, lead: Lead_v1, config: OBJECT) → FollowupResult:
    // Seleccionar template según intento actual
    attempt = request.followup_count + 1
    template_key = config.followup_templates[request.document_type][attempt] OR "generic"
    template = config.templates[template_key]
    
    // Construir mensaje contextual
    name = lead.name OR "estimado cliente"
    doc_name = config.document_names[request.document_type]
    time_remaining = FORMAT_TIME_REMAINING(request.deadline, NOW())
    
    message_body = template.text
        .replace("{name}", name)
        .replace("{document_name}", doc_name)
        .replace("{time_remaining}", time_remaining)
        .replace("{delivery_method}", config.delivery_methods[request.document_type])
        .replace("{brand}", config.brand_name)
        .trim()
    
    // Compliance check
    IF NOT compliance_guardrails.can_send(lead.phone):
        LOG: "followup_blocked_rate_limit", request_id: request.document_request_id
        RETURN {status: "blocked", reason: "rate_limit"}
    
    // Envío
    delivery = channel_adapter.send_message(lead, {type: "text", body: message_body})
    
    IF delivery.status == "sent" OR delivery.status == "queued":
        UPDATE request.followup_count = request.followup_count + 1
        UPDATE request.last_followup_sent_at = NOW()
        EMIT_METRIC: "doc_followup_sent", value: 1, tags: {
            type: request.document_type,
            attempt: request.followup_count,
            channel: CONFIG.channel
        }
    
    LOG: "followup_sent", request_id: request.document_request_id, attempt: request.followup_count
    RETURN {status: "sent", attempt: request.followup_count}
```

---

### 5.4 FR-4: Submission Handling & Normalization

**ID:** FR-4  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe capturar documentos entrantes desde múltiples métodos de entrega y normalizarlos para validación.

**User Story:**  
Como sistema de ingestión documental, quiero recibir documentos desde enlaces de upload, email adjuntos, WhatsApp o respuesta de texto, y normalizarlos a una estructura común para validación consistente.

**Acceptance Criteria:**
- [ ] Soporte para métodos de entrega: `upload_link`, `email_attachment`, `whatsapp_media`, `text_response`
- [ ] Extracción de metadata: filename, mime_type, size, timestamp, sender
- [ ] Normalización a `DocumentSubmission` schema estandarizado
- [ ] Correlación submission → request → lead vía `document_request_id` o `lead_id`
- [ ] Detección de duplicados (mismo lead, mismo tipo de documento)
- [ ] Actualización de `request.status = "submitted"` y `lead.metadata.last_doc_submission_at`
- [ ] Logging de cada submission para auditoría

**DocumentSubmission Schema:**
```
OBJECT DocumentSubmission:
    submission_id: STRING (UUID)
    document_request_id: STRING (FK a DocumentRequest)
    lead_id: STRING (FK a Lead_v1)
    document_type: STRING
    delivery_method: ENUM ["upload_link", "email_attachment", "whatsapp_media", "text_response"]
    submitted_at: ISO-8601
    content_reference: OBJECT
        type: ENUM ["file_url", "email_message_id", "whatsapp_media_id", "text_body"]
        value: STRING
        metadata: OBJECT
            filename: STRING OR NULL
            mime_type: STRING OR NULL
            size_bytes: INTEGER OR NULL
            pages: INTEGER OR NULL  // si aplica
    status: ENUM ["received", "validating", "accepted", "rejected", "needs_clarification"]
    validation_result: ValidationResult OR NULL
```

**Pseudocódigo de normalización:**
```
FUNCTION normalize_document_submission(raw_input: OBJECT, config: OBJECT) → DocumentSubmission OR NULL:
    // Determinar método de entrega
    delivery_method = DETECT_DELIVERY_METHOD(raw_input)
    // returns: "upload_link" | "email_attachment" | "whatsapp_media" | "text_response"
    
    // Extraer referencia al contenido
    content_ref = EXTRACT_CONTENT_REFERENCE(raw_input, delivery_method)
    IF content_ref == NULL:
        LOG: "content_extraction_failed", input_type: delivery_method
        RETURN NULL
    
    // Correlacionar con request pendiente
    request = FIND_PENDING_REQUEST(
        lead_id: raw_input.lead_id OR raw_input.from_phone,
        document_type: raw_input.document_type_hint OR NULL,
        status: "sent" OR "followed_up"
    )
    IF request == NULL:
        LOG: "orphan_submission", lead_id: raw_input.lead_id, type: raw_input.document_type_hint
        RETURN NULL  // o enqueue para revisión humana
    
    // Construir submission normalizada
    submission = {
        submission_id: GENERATE_UUID(),
        document_request_id: request.document_request_id,
        lead_id: request.lead_id,
        document_type: request.document_type,
        delivery_method: delivery_method,
        submitted_at: NOW(),
        content_reference: content_ref,
        status: "received",
        validation_result: NULL
    }
    
    // Actualizar request
    request.status = "submitted"
    request.submission_id = submission.submission_id
    request.submitted_at = submission.submitted_at
    
    LOG: "submission_normalized", submission_id: submission.submission_id, method: delivery_method
    EMIT_METRIC: "doc_submission_received", value: 1, tags: {type: request.document_type, method: delivery_method}
    RETURN submission
```

---

### 5.5 FR-5: Structural Validation Engine

**ID:** FR-5  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe evaluar la completitud estructural de documentos contra reglas configurables, sin interpretar contenido.

**User Story:**  
Como sistema de validación, quiero verificar que un documento entregado cumple con requisitos estructurales mínimos (campos presentes, formato, fecha) para determinar si está listo para revisión humana o procesamiento posterior.

**Acceptance Criteria:**
- [ ] Reglas de validación configurables por tipo de documento: `validation_rules`
- [ ] Tipos de reglas soportados: `field_present`, `format_match`, `date_within_range`, `signature_detected`, `file_type_allowed`
- [ ] Evaluación determinista: mismos inputs = mismos resultados
- [ ] Justificación estructurada de rechazo: `rejection_reasons[]`
- [ ] Soporte para "necesita aclaración" vs "rechazado" vs "aceptado"
- [ ] Actualización de `request.status` y `submission.validation_result`
- [ ] Logging de validación para auditoría y mejora de reglas

**ValidationRule Schema:**
```
OBJECT ValidationRule:
    rule_id: STRING
    rule_type: ENUM ["field_present", "format_match", "date_within_range", "signature_detected", "file_type_allowed"]
    field_name: STRING OR NULL  // si aplica
    parameters: OBJECT  // específico por rule_type
        // Ejemplo format_match: {pattern: "^[A-Z]{2}\\d{6}$", error: "Formato inválido"}
        // Ejemplo date_within_range: {min_days_ago: 0, max_days_ago: 30, error: "Fecha fuera de rango"}
    error_message: STRING  // mensaje para el lead si falla
```

**Pseudocódigo de validación:**
```
FUNCTION validate_document_structure(submission: DocumentSubmission, config: OBJECT) → ValidationResult:
    // Cargar reglas de validación por tipo de documento
    rules = config.validation_rules[submission.document_type]
    IF rules == NULL OR rules.length == 0:
        // Sin reglas = aceptar por defecto (configurable)
        RETURN {
            status: "accepted",
            passed_rules: [],
            failed_rules: [],
            needs_clarification: false,
            notes: "No validation rules configured for this document type"
        }
    
    // Evaluar cada regla
    passed = []
    failed = []
    clarification_needed = []
    
    FOR EACH rule IN rules:
        result = EVALUATE_RULE(submission, rule, config)
        // EVALUATE_RULE returns: {passed: boolean, error: STRING OR NULL, needs_clarification: boolean}
        
        IF result.passed:
            passed.append(rule.rule_id)
        ELSE IF result.needs_clarification:
            clarification_needed.append({rule_id: rule.rule_id, message: rule.error_message})
        ELSE:
            failed.append({rule_id: rule.rule_id, message: rule.error_message})
    
    // Determinar resultado final
    IF failed.length > 0:
        status = "rejected"
        next_action = "request_resubmission"
    ELSE IF clarification_needed.length > 0:
        status = "needs_clarification"
        next_action = "ask_for_clarification"
    ELSE:
        status = "accepted"
        next_action = "trigger_milestone_payment"
    
    // Construir resultado
    result = {
        status: status,
        passed_rules: passed,
        failed_rules: failed,
        clarification_needed: clarification_needed,
        next_action: next_action,
        validated_at: NOW(),
        notes: BUILD_VALIDATION_NOTES(passed, failed, clarification_needed)
    }
    
    // Actualizar submission y request
    submission.validation_result = result
    submission.status = status
    request.status = status  // propagar a request
    
    LOG: "validation_complete", submission_id: submission.submission_id, status: status
    EMIT_METRIC: "doc_validation_completed", value: 1, tags: {
        type: submission.document_type,
        result: status,
        rule_count: rules.length
    }
    
    RETURN result
```

---

### 5.6 FR-6: CRM Sync & Milestone Payment Trigger

**ID:** FR-6  
**Prioridad:** ALTA  
**Descripción:** El sistema debe sincronizar el estado documental con el CRM y triggerar pagos de milestone cuando un documento es aceptado.

**User Story:**  
Como sistema de integración, quiero actualizar el CRM con el estado de documentación y emitir señales de pago de milestone cuando un documento es aceptado, para cerrar el ciclo de valor sin intervención humana.

**Acceptance Criteria:**
- [ ] Actualización de campos documentales en CRM: `document_status`, `submission_date`, `validation_result`
- [ ] Trigger de payment webhook cuando `validation_result.status == "accepted"`
- [ ] Payload de payment trigger con: `lead_id`, `document_type`, `milestone_amount`, `trace_id`
- [ ] Idempotencia: mismos inputs no generan pagos duplicados
- [ ] Fallback graceful si CRM o payment endpoint está indisponible (queue + retry)
- [ ] Logging de todas las operaciones de sync y payment para reconciliación

**Pseudocódigo de sync y payment:**
```
FUNCTION sync_and_trigger_milestone(submission: DocumentSubmission, request: DocumentRequest, config: OBJECT) → SyncResult:
    // Preparar actualización de CRM
    crm_update = {
        document_status: submission.status,
        document_submission_date: submission.submitted_at,
        document_validation_result: submission.validation_result.status,
        document_validation_notes: submission.validation_result.notes,
        last_document_interaction: NOW()
    }
    
    // Si aceptado, agregar campos de milestone
    IF submission.validation_result.status == "accepted":
        milestone_config = config.milestone_payments[request.document_type]
        IF milestone_config:
            crm_update.milestone_triggered = true
            crm_update.milestone_amount = milestone_config.amount
            crm_update.milestone_currency = milestone_config.currency
    
    // Actualizar CRM
    crm_result = crm_adapter.update_lead(request.lead_id, crm_update)
    
    // Si aceptado y hay payment configurado, triggerar webhook
    payment_result = NULL
    IF submission.validation_result.status == "accepted" AND config.milestone_payments[request.document_type]:
        payment_payload = {
            trace_id: GENERATE_UUID(),  // nuevo trace para payment
            lead_id: request.lead_id,
            document_request_id: request.document_request_id,
            submission_id: submission.submission_id,
            document_type: request.document_type,
            milestone: {
                name: config.milestone_payments[request.document_type].name,
                amount: config.milestone_payments[request.document_type].amount,
                currency: config.milestone_payments[request.document_type].currency
            },
            metadata: {
                validated_at: submission.validation_result.validated_at,
                partner_id: request.metadata.partner_id,
                niche: request.metadata.niche
            }
        }
        
        payment_result = payment_adapter.trigger_milestone(payment_payload, {
            endpoint: config.milestone_payments[request.document_type].webhook_url,
            api_key: config.payment_api_key,
            timeout_ms: 10000
        })
    
    // Actualizar request final
    request.status = submission.validation_result.status
    request.completed_at = NOW() IF submission.validation_result.status IN ["accepted", "rejected"] ELSE NULL
    request.payment_triggered = payment_result?.success OR FALSE
    
    // Emitir métricas
    EMIT_METRIC: "doc_crm_synced", value: 1, tags: {type: request.document_type, status: submission.status}
    IF payment_result?.success:
        EMIT_METRIC: "milestone_payment_triggered", value: 1, tags: {
            type: request.document_type,
            amount: config.milestone_payments[request.document_type].amount
        }
    
    LOG: "sync_and_payment_complete", request_id: request.document_request_id, crm_success: crm_result.success, payment_success: payment_result?.success
    RETURN {
        crm_synced: crm_result.success,
        payment_triggered: payment_result?.success OR FALSE,
        errors: [crm_result.error] + [payment_result?.error] FILTER non-null
    }
```

---

### 5.7 FR-7: Compliance & Data Handling

**ID:** FR-7  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe respetar regulaciones de manejo de datos sensibles (GDPR, CCPA, sectoriales) en cada interacción documental.

**User Story:**  
Como operador en región regulada, quiero que el sistema maneje documentos sensibles con consentimiento explícito, retención limitada y masking de PII en logs, para cumplir con la ley y proteger la reputación.

**Acceptance Criteria:**
- [ ] Verificación de consentimiento para manejo de datos sensibles antes de solicitar documento
- [ ] Configuración de retención por tipo de documento: `retention_policy`
- [ ] Masking automático de PII en logs: nombres, IDs, firmas → tokens o hashes
- [ ] Encriptación en tránsito y en reposo para referencias a documentos
- [ ] Derecho a olvido: endpoint para eliminar metadata de lead/documento
- [ ] Logging de todos los accesos a metadata documental para auditoría
- [ ] Alertas configurables cuando se acerque a límite de retención

**Pseudocódigo de compliance documental:**
```
FUNCTION verify_document_compliance(lead: Lead_v1, document_type: STRING, config: OBJECT) → ComplianceDecision:
    // Verificar consentimiento específico para tipo de documento
    consent_config = config.document_consents[document_type]
    IF consent_config.requires_explicit_consent:
        has_consent = compliance_guardrails.has_document_consent(lead.phone, document_type)
        IF NOT has_consent:
            RETURN {
                allowed: false,
                reason: "missing_explicit_consent",
                action: "request_consent_first"
            }
    
    // Verificar que el tipo de documento está permitido para la región
    region_config = config.regional_restrictions[CONFIG.region]
    IF region_config AND document_type IN region_config.prohibited_documents:
        RETURN {
            allowed: false,
            reason: "document_prohibited_in_region",
            action: "escalate_to_legal"
        }
    
    // Verificar límites de frecuencia de solicitud
    recent_requests = COUNT_RECENT_REQUESTS(lead.lead_id, document_type, window: "30_days")
    IF recent_requests >= config.max_requests_per_period[document_type]:
        RETURN {
            allowed: false,
            reason: "request_frequency_limit_exceeded",
            action: "wait_for_cooling_period"
        }
    
    // Consentimiento válido, permitir continuación
    RETURN {allowed: true, reason: "compliance_checks_passed"}

FUNCTION apply_data_retention(request: DocumentRequest, config: OBJECT) → RetentionResult:
    // Obtener política de retención por tipo de documento
    retention_policy = config.retention_policies[request.document_type]
    IF retention_policy == NULL:
        // Default: 90 días para metadata
        retention_policy = {metadata_retention_days: 90, content_retention_days: 0}
    
    // Calcular fecha de expiración
    expiry_date = request.completed_at OR request.created_at + retention_policy.metadata_retention_days * 1_DAY
    
    // Programar tarea de limpieza (si el sistema lo soporta)
    IF config.supports_scheduled_cleanup:
        SCHEDULE_METADATA_CLEANUP(
            request_id: request.document_request_id,
            execute_at: expiry_date,
            action: retention_policy.cleanup_action  // "delete", "anonymize", "archive"
        )
    
    // Actualizar request con metadata de retención
    request.retention_metadata = {
        policy_applied: retention_policy,
        expiry_date: expiry_date,
        cleanup_scheduled: config.supports_scheduled_cleanup
    }
    
    LOG: "retention_policy_applied", request_id: request.document_request_id, expiry: expiry_date
    RETURN {scheduled: true, expiry_date: expiry_date}
```

---

### 5.8 FR-8: Metrics Emission & Observability

**ID:** FR-8  
**Prioridad:** ALTA  
**Descripción:** El sistema debe emitir métricas estructuradas para medir performance de recolección documental y permitir optimización.

**User Story:**  
Como analista, quiero métricas en tiempo real de solicitud, entrega, validación y pago por tipo de documento, para identificar cuellos de botella y ajustar la estrategia de recolección.

**Acceptance Criteria:**
- [ ] Métricas de funnel: `doc_request_triggered`, `doc_request_sent`, `doc_submission_received`, `doc_validated`, `milestone_payment_triggered`
- [ ] Métricas de calidad: `completion_rate`, `validation_pass_rate`, `avg_collection_time`, `followup_effectiveness`
- [ ] Segmentación por: `document_type`, `niche`, `channel`, `region`, `partner_id`
- [ ] Emisión asíncrona vía MetricsEmitter de Shared Components
- [ ] Correlación con trace_id para debugging end-to-end
- [ ] Dashboard-ready: formato compatible con Prometheus/Datadog

**Pseudocódigo de emisión de métricas:**
```
FUNCTION emit_doc_collection_metrics(request: DocumentRequest, event: ENUM, meta OBJECT):
    base_tags = {
        android: "document_collection",
        document_type: request.document_type,
        niche: request.metadata.niche,
        channel: CONFIG.channel,
        region: CONFIG.region,
        partner_id: request.metadata.partner_id
    }
    
    SWITCH event:
        CASE "doc_request_triggered":
            metrics_emitter.emit_counter("doc_request_triggered", 1, base_tags)
            
        CASE "doc_request_sent":
            metrics_emitter.emit_counter("doc_request_sent", 1, base_tags)
            IF meta.request_latency_ms:
                metrics_emitter.emit_histogram("doc_request_latency", meta.request_latency_ms, base_tags)
            
        CASE "doc_submission_received":
            metrics_emitter.emit_counter("doc_submission_received", 1, {
                ...base_tags,
                delivery_method: meta.delivery_method
            })
            // Calcular tiempo de respuesta si tenemos request_sent
            IF request.metadata.first_request_sent_at:
                response_time = NOW() - request.metadata.first_request_sent_at
                metrics_emitter.emit_histogram("doc_response_time", response_time, base_tags)
            
        CASE "doc_validated":
            metrics_emitter.emit_counter("doc_validation_completed", 1, {
                ...base_tags,
                validation_result: meta.validation_result.status
            })
            // Calcular completion rate por tipo de documento
            completion_rate = calculate_completion_rate(request.document_type, time_window: "7d")
            metrics_emitter.emit_gauge("doc_completion_rate", completion_rate, base_tags)
            
        CASE "milestone_payment_triggered":
            metrics_emitter.emit_counter("milestone_payment_triggered", 1, {
                ...base_tags,
                amount: meta.milestone_amount
            })
            // Calcular revenue attribution
            revenue_rate = calculate_revenue_per_request(request.document_type, time_window: "30d")
            metrics_emitter.emit_gauge("doc_revenue_per_request", revenue_rate, base_tags)
            
        CASE "doc_rejected":
            metrics_emitter.emit_counter("doc_validation_rejected", 1, {
                ...base_tags,
                rejection_reasons: meta.rejection_reasons
            })
```

---

## 6. Non-Functional Requirements

### 6.1 Performance

| Métrica | Target | Medición |
|---------|--------|----------|
| Latencia trigger → first request | P95 ≤ 5 minutos | End-to-end tracing |
| Throughput de solicitudes | ≥ 300 requests/minuto | Sustained load |
| Tiempo de validación estructural | < 2 segundos por documento | P95 |
| CRM sync + payment trigger | < 5 segundos desde aceptación | P95 |

### 6.2 Reliability

| Métrica | Target | Estrategia |
|---------|--------|------------|
| Request delivery rate | ≥ 98% | Retry + fallback channel |
| Submission capture rate | ≥ 99.5% | Multi-method ingestion + correlation |
| Validation accuracy | ≥ 95% vs human review | Rule-based + fallback to human |
| Payment trigger success | 100% | Idempotent + DLQ + reconciliation |

### 6.3 Security

| Requisito | Implementación |
|-----------|----------------|
| PII en logs | Masking automático: nombres → hash, IDs → token |
| Credenciales de adapters | Secrets management, nunca en código |
| Encriptación de referencias | TLS 1.2+ en tránsito, encryption at rest |
| Consent data integrity | Write-once, append-only, audit trail |
| Derecho a olvido | Endpoint seguro para eliminación de metadata |

### 6.4 Scalability

| Dimensión | Estrategia |
|-----------|------------|
| Horizontal | Stateless processing, queue-based ingestion |
| Multi-document-type | Configuración por `document_type`, reglas aisladas |
| Multi-partner | Aislamiento por `partner_id` en metadata |
| Multi-region | Compliance config por región, retention policies |

---

## 7. Technical Specifications

### 7.1 Contratos Extendidos (sobre cxEngine Shared Components)

#### DocumentRequest Schema
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "DocumentRequest",
  "type": "object",
  "required": [
    "document_request_id", "lead_id", "document_type", 
    "required_fields", "deadline", "created_at", "status"
  ],
  "properties": {
    "document_request_id": {"type": "string", "format": "uuid"},
    "lead_id": {"type": "string"},
    "document_type": {"type": "string"},
    "required_fields": {
      "type": "array",
      "items": {"type": "string"}
    },
    "validation_rules": {
      "type": "array",
      "items": {"$ref": "#/definitions/ValidationRule"}
    },
    "deadline": {"type": "string", "format": "date-time"},
    "created_at": {"type": "string", "format": "date-time"},
    "status": {
      "enum": ["requested", "sent", "followed_up", "submitted", "validating", "accepted", "rejected", "needs_clarification", "blocked"]
    },
    "metadata": {
      "type": "object",
      "properties": {
        "trigger_event": {"type": "string"},
        "trigger_timestamp": {"type": "string", "format": "date-time"},
        "niche": {"type": "string"},
        "partner_id": {"type": "string"},
        "first_request_sent_at": {"type": "string", "format": "date-time"},
        "followup_count": {"type": "integer", "minimum": 0},
        "last_followup_sent_at": {"type": "string", "format": "date-time"},
        "submission_id": {"type": "string"},
        "submitted_at": {"type": "string", "format": "date-time"},
        "completed_at": {"type": "string", "format": "date-time"},
        "payment_triggered": {"type": "boolean"},
        "retention_metadata": {"$ref": "#/definitions/RetentionMetadata"}
      }
    }
  },
  "definitions": {
    "ValidationRule": {
      "type": "object",
      "required": ["rule_id", "rule_type"],
      "properties": {
        "rule_id": {"type": "string"},
        "rule_type": {"enum": ["field_present", "format_match", "date_within_range", "signature_detected", "file_type_allowed"]},
        "field_name": {"type": "string"},
        "parameters": {"type": "object"},
        "error_message": {"type": "string"}
      }
    },
    "RetentionMetadata": {
      "type": "object",
      "properties": {
        "policy_applied": {"type": "object"},
        "expiry_date": {"type": "string", "format": "date-time"},
        "cleanup_scheduled": {"type": "boolean"}
      }
    }
  }
}
```

#### DocumentSubmission Schema
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "DocumentSubmission",
  "type": "object",
  "required": [
    "submission_id", "document_request_id", "lead_id", 
    "document_type", "delivery_method", "submitted_at", "status"
  ],
  "properties": {
    "submission_id": {"type": "string", "format": "uuid"},
    "document_request_id": {"type": "string"},
    "lead_id": {"type": "string"},
    "document_type": {"type": "string"},
    "delivery_method": {
      "enum": ["upload_link", "email_attachment", "whatsapp_media", "text_response"]
    },
    "submitted_at": {"type": "string", "format": "date-time"},
    "content_reference": {
      "type": "object",
      "required": ["type", "value"],
      "properties": {
        "type": {"enum": ["file_url", "email_message_id", "whatsapp_media_id", "text_body"]},
        "value": {"type": "string"},
        "metadata": {
          "type": "object",
          "properties": {
            "filename": {"type": "string"},
            "mime_type": {"type": "string"},
            "size_bytes": {"type": "integer"},
            "pages": {"type": "integer"}
          }
        }
      }
    },
    "status": {
      "enum": ["received", "validating", "accepted", "rejected", "needs_clarification"]
    },
    "validation_result": {
      "type": "object",
      "properties": {
        "status": {"enum": ["accepted", "rejected", "needs_clarification"]},
        "passed_rules": {"type": "array", "items": {"type": "string"}},
        "failed_rules": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "rule_id": {"type": "string"},
              "message": {"type": "string"}
            }
          }
        },
        "clarification_needed": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "rule_id": {"type": "string"},
              "message": {"type": "string"}
            }
          }
        },
        "next_action": {"enum": ["trigger_milestone_payment", "request_resubmission", "ask_for_clarification"]},
        "validated_at": {"type": "string", "format": "date-time"},
        "notes": {"type": "string"}
      }
    }
  }
}
```

### 7.2 Configuración por Tipo de Documento (Ejemplo: DSAR Form)

```yaml
# config/document_types/dsar_form_v1.yaml
document_type: dsar_form
display_name: "Formulario DSAR (Derecho de Acceso)"
niche_applicability: ["legal", "financial", "healthcare"]

request_template:
  text: "Hola {name}, para continuar con tu solicitud en {brand}, necesitamos el formulario DSAR completado. Envíalo aquí: {delivery_method}. Fecha límite: {deadline}. ¿Necesitas ayuda?"
  max_length_sms: 160
  variables: ["name", "document_name", "deadline", "delivery_method", "brand"]

required_fields:
  - full_name
  - id_number
  - signature
  - date
  - request_type

validation_rules:
  - rule_id: "signature_present"
    rule_type: "field_present"
    field_name: "signature"
    error_message: "Falta la firma. Por favor firma el formulario."
    
  - rule_id: "date_within_30_days"
    rule_type: "date_within_range"
    field_name: "date"
    parameters:
      min_days_ago: 0
      max_days_ago: 30
    error_message: "La fecha debe ser de los últimos 30 días."
    
  - rule_id: "id_format_mx"
    rule_type: "format_match"
    field_name: "id_number"
    parameters:
      pattern: "^[A-Z]{4}\\d{6}[A-Z\\d]{3}$"  # CURP format example
      case_sensitive: false
    error_message: "El número de identificación no tiene el formato válido."

delivery_methods:
  upload_link: "https://secure.{brand}.com/upload/dsar"
  email: "docs@{brand}.com"
  whatsapp: "reply_with_photo"

followup_schedule:
  intervals:
    - offset_hours: 24
      message_template: "gentle_reminder"
      max_count: 2
    - offset_hours: 72
      message_template: "final_reminder"
      max_count: 1
  pause_on_response: true
  max_total_followups: 3

followup_templates:
  gentle_reminder:
    text: "Hola {name}, solo te recordamos que necesitamos el formulario DSAR para continuar. Puedes enviarlo aquí: {delivery_method}. ¡Gracias!"
  final_reminder:
    text: "Hola {name}, última oportunidad para enviar el formulario DSAR antes de {time_remaining}. Si necesitas ayuda, responde a este mensaje."

milestone_payment:
  enabled: true
  name: "DSAR Form Submission"
  amount: 125.00
  currency: "USD"
  webhook_url: "https://partner.{brand}.com/webhooks/milestone"
  idempotency_key_field: "document_request_id"

compliance:
  requires_explicit_consent: true
  consent_type: "document_handling_sensitive"
  retention_policy:
    metadata_retention_days: 2555  # 7 años para legal
    content_retention_days: 0  # No almacenamos contenido, solo referencia
    cleanup_action: "delete_metadata"
  regional_restrictions:
    EU:
      prohibited: false
      additional_requirements: ["gdpr_article_15_notice"]
    MX:
      prohibited: false
      additional_requirements: ["aviso_privacidad_mx"]
```

### 7.3 Flujo de Estado del Documento (State Machine)

```
[TRIGGERED] 
   ↓ (milestone detected)
[REQUESTED] → send_request() → [SENT]
   ↓ (follow-up scheduled)
[SENT] → wait_for_submission()
   ├─ submission received → [SUBMITTED]
   ├─ follow-up sent → [FOLLOWED_UP] (count++)
   └─ deadline passed → [OVERDUE] → escalate_to_human()

[SUBMITTED] → validate_structure()
   ├─ validation passed → [ACCEPTED] → trigger_milestone_payment()
   ├─ validation failed → [REJECTED] → request_resubmission()
   └─ needs clarification → [NEEDS_CLARIFICATION] → ask_for_clarification()

[ACCEPTED] → sync_crm() + trigger_payment() → [COMPLETED]

[REJECTED] → log_reasons() → [CLOSED]

[NEEDS_CLARIFICATION] → send_clarification_request()
   ↓ (lead responds)
[NEEDS_CLARIFICATION] → re-validate() → [ACCEPTED] | [REJECTED]

Timeouts:
- SENT → no submission in deadline → [OVERDUE]
- NEEDS_CLARIFICATION → no response in 48h → [REJECTED]
```

---

## 8. Implementation Notes

### 8.1 Stack Reference (n8n como ejemplo)

| Componente | Nodo n8n | Configuración |
|------------|----------|---------------|
| Milestone Trigger | Webhook / CRM Trigger | Listen for status_change events |
| Request Builder | Code (JavaScript) + Template Engine | Merge variables, validate length |
| Follow-up Scheduler | Schedule Trigger + Code | Calculate intervals, pause on response |
| Submission Handler | Webhook / Email Trigger + Code | Normalize multi-method submissions |
| Validation Engine | Code (rule evaluation) | Deterministic rule execution |
| CRM Sync | HTTP Request (HighLevel/Salesforce) | Idempotent upsert, field mapping |
| Payment Trigger | HTTP Request (partner webhook) | Idempotency key, retry policy |
| Compliance Guard | Code + Shared Component | Consent check, retention scheduling |
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
# Document Collection Engine Config
DOC_DEFAULT_NICHE=legal
DOC_MAX_FOLLOWUP_ROUNDS=3
DOC_SUBMISSION_TIMEOUT_HOURS=168  # 7 días default
DOC_VALIDATION_TIMEOUT_MS=2000
DOC_REQUEST_TIMEOUT_MS=5000

# Delivery Methods
DELIVERY_UPLOAD_BASE_URL=https://secure.{brand}.com/upload
DELIVERY_EMAIL_DOCS_ADDRESS=docs@{brand}.com
DELIVERY_WHATSAPP_ENABLED=true

# Channel Config
CHANNEL_PROVIDER=twilio|meta_whatsapp|sendgrid
CHANNEL_PHONE_NUMBER=+52...
CHANNEL_API_KEY=xxx

# CRM Config
CRM_PROVIDER=highlevel|salesforce|hubspot
CRM_API_KEY=xxx
CRM_WEBHOOK_SECRET=xxx
CRM_DOC_FIELDS_PREFIX=doc_

# Payment Config
PAYMENT_WEBHOOK_ENABLED=true
PAYMENT_API_KEY=xxx
PAYMENT_IDEMPOTENCY_ENABLED=true

# Compliance
COMPLIANCE_REGION=MX|US|EU
COMPLIANCE_CONSENT_REQUIRED=true
COMPLIANCE_RETENTION_CONFIG_PATH=./config/retention_policies.yaml
COMPLIANCE_PII_MASKING=true

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
| **TC-DC-01** | Trigger de milestone válido | Lead `qualified` + event `contract_signed` | DocumentRequest creado, status `requested` |
| **TC-DC-02** | Request delivery exitoso | DocumentRequest en `sent` | Mensaje enviado, lead notificado |
| **TC-DC-03** | Follow-up programado | Request sin submission en 24h | Follow-up enviado, count incrementado |
| **TC-DC-04** | Submission vía upload link | POST a upload endpoint con file | DocumentSubmission creado, status `received` |
| **TC-DC-05** | Validación estructural exitosa | Submission con todos los campos requeridos | Status `accepted`, payment triggered |
| **TC-DC-06** | Validación fallida (firma faltante) | Submission sin campo signature | Status `rejected`, reason logged |
| **TC-DC-07** | Necesita aclaración | Submission con fecha fuera de rango | Status `needs_clarification`, message sent |
| **TC-DC-08** | Compliance block (sin consentimiento) | Lead sin consent para DSAR | Request bloqueado, log de compliance |
| **TC-DC-09** | Payment trigger idempotente | Mismo request procesado 2 veces | Solo 1 payment triggered |
| **TC-DC-10** | Retención programada | Request accepted con política 7 años | Cleanup scheduled para expiry_date |

### 9.2 Test Execution

```bash
# Unit tests (componentes individuales)
npm test -- unit/milestone-trigger/
npm test -- unit/validation-engine/
npm test -- unit/compliance-check/

# Integration tests (flujos end-to-end)
npm test -- integration/request-to-submission/
npm test -- integration/validation-to-payment/

# Compliance tests (regulaciones)
npm test -- compliance/consent-verification/
npm test -- compliance/retention-policy/

# Load tests (escalabilidad)
npm test -- load/doc-requests-300-per-min/
npm test -- load/concurrent-validations-100/

# Idempotency tests (pagos)
npm test -- idempotency/payment-trigger/
npm test -- idempotency/crm-sync/
```

---

## 10. Success Metrics (Técnicos y de Negocio)

| Métrica | Fórmula | Target | Frecuencia |
|---------|---------|--------|------------|
| Request delivery rate | sent / triggered | ≥ 98% | Diario |
| Submission completion rate | submitted / sent | ≥ 70% | Por tipo de documento |
| Validation pass rate | accepted / submitted | ≥ 85% | Por tipo de documento |
| Avg. collection time | mean(submitted_at - requested_at) | ≤ 48 horas | Por tipo de documento |
| Milestone payment success | payments_triggered / accepted | ≥ 99.9% | Diario |
| Compliance adherence | compliant_requests / total | 100% | Diario |
| Follow-up effectiveness | submissions_after_followup / followups_sent | ≥ 25% | Por template |
| Human escalation rate | escalated / total_requests | ≤ 10% | Semanal |

---

## 11. Deployment Checklist

### Pre-Deployment
- [ ] Configuración de tipos de documento validada (`config/document_types/{type}_v1.yaml`)
- [ ] ChannelAdapter configurado y testeado (SMS/WhatsApp/email)
- [ ] CRMAdapter configurado con campos documentales mapeados
- [ ] PaymentAdapter configurado con webhook endpoint y API key
- [ ] Compliance guardrails cargados con consent y retention policies
- [ ] Upload endpoint seguro configurado (si aplica)
- [ ] Métricas backend configurado y recibiendo datos
- [ ] Logging estructurado habilitado con masking de PII
- [ ] Tests de idempotencia de pago pasando
- [ ] Plan de rollback documentado para fallos de payment trigger

### Post-Deployment
- [ ] Monitorizar request delivery rate por primeras 100 solicitudes
- [ ] Validar que compliance blocks se aplican correctamente
- [ ] Confirmar que payments se triggeran solo una vez por request
- [ ] Verificar que CRM recibe campos documentales actualizados
- [ ] Revisar logs para detectar PII no masked
- [ ] Documentar cualquier desviación de targets de completitud

---

## 12. Version History

| Versión | Fecha | Cambios | Autor |
|---------|-------|---------|-------|
| 1.0 | 2026-02-20 | Especificación inicial de Document Collection Engine | O.A. Perez Garrido |

---

## 13. Appendices

### Appendix A: Template Library por Tipo de Documento

| Tipo de Documento | Request Template | Follow-up Templates | Validation Focus |
|------------------|-----------------|-------------------|-----------------|
| **DSAR Form** | "Hola {name}, para continuar con tu solicitud en {brand}, necesitamos el formulario DSAR completado. Envíalo aquí: {delivery_method}. Fecha límite: {deadline}." | gentle: "Recordatorio amable", final: "Última oportunidad" | Firma presente, fecha válida, formato ID |
| **Insurance Claim** | "Hola {name}, para procesar tu reclamación en {brand}, adjunta los documentos solicitados: {required_fields}. Usa este enlace: {delivery_method}." | gentle: "¿Necesitas ayuda con los documentos?", final: "Tu reclamación está pendiente de documentos" | Todos los campos requeridos, fechas coherentes |
| **Loan Application** | "Hola {name}, para avanzar con tu solicitud de préstamo, necesitamos: {required_fields}. Envíalos aquí: {delivery_method} antes de {deadline}." | gentle: "Tu préstamo está en espera de documentos", final: "Último día para enviar documentos" | Ingresos verificables, identificación vigente |
| **Service Contract** | "Hola {name}, para activar tu servicio en {brand}, firma y devuelve el contrato adjunto. Instrucciones: {delivery_method}." | gentle: "Recordatorio: contrato pendiente", final: "Activa tu servicio firmando hoy" | Firma detectada, fecha de firma reciente |

### Appendix B: Validation Rule Patterns

```yaml
# Ejemplo: patrones de reglas de validación estructural
validation_patterns:
  field_present:
    description: "Verifica que un campo específico esté presente y no vacío"
    parameters:
      field_name: STRING (required)
      allow_whitespace_only: BOOLEAN (default: false)
    error_template: "El campo {field_name} es requerido."
    
  format_match:
    description: "Verifica que un campo coincida con un patrón regex"
    parameters:
      field_name: STRING (required)
      pattern: STRING (regex, required)
      case_sensitive: BOOLEAN (default: true)
      error_message: STRING (required)
    error_template: "{error_message}"
    
  date_within_range:
    description: "Verifica que una fecha esté dentro de un rango relativo a hoy"
    parameters:
      field_name: STRING (required)
      min_days_ago: INTEGER (default: -infinity)
      max_days_ago: INTEGER (default: +infinity)
      error_message: STRING (required)
    error_template: "{error_message}"
    
  signature_detected:
    description: "Verifica que una firma esté presente en documento escaneado (metadata)"
    parameters:
      metadata_field: STRING (required, e.g., "has_signature_flag")
      expected_value: ANY (default: true)
    error_template: "Se requiere una firma válida en el documento."
    
  file_type_allowed:
    description: "Verifica que el tipo de archivo esté en lista permitida"
    parameters:
      allowed_mime_types: ARRAY<STRING> (required)
      error_message: STRING (required)
    error_template: "{error_message}"
```

### Appendix C: Error Codes & Fallbacks

| Código | Descripción | Fallback | Alerta |
|--------|-------------|----------|--------|
| `MILESTONE_NOT_CONFIGURED` | No hay configuración para el tipo de milestone | Loguear, ignorar request | Si > 5% de triggers sin config |
| `REQUEST_SEND_FAILED` | Fallo al enviar solicitud de documento | Retry 2x con backoff, luego marcar como `blocked` | Si > 2% de fallos en ventana de 5min |
| `SUBMISSION_ORPHAN` | Submission recibida sin request pendiente | Loguear, enqueue para revisión humana | Si > 1% de submissions huérfanas |
| `VALIDATION_RULE_ERROR` | Error evaluando regla de validación | Marcar como `needs_clarification`, loguear regla | Inmediata si regla crítica falla |
| `PAYMENT_TRIGGER_FAILED` | Fallo al triggerar webhook de pago | Retry 3x con backoff, luego DLQ + alerta | Inmediata si crítico para revenue |
| `COMPLIANCE_CONSENT_MISSING` | Lead sin consentimiento para tipo de documento | No enviar request, loguear, emitir métrica | Si > 10% de leads sin consentimiento |
| `RETENTION_POLICY_MISSING` | No hay política de retención para tipo de documento | Aplicar default (90 días metadata), loguear | Si config incompleta detectada |

### Appendix D: Handoff a Humanos (Escenarios)

| Escenario | Trigger | Acción de Handoff | Información Entregada |
|-----------|---------|------------------|----------------------|
| **Documento rechazado tras 2 intentos** | `validation_result.status == "rejected"` + followup_count >= 2 | Escalar a "Document Review Queue" | Submission metadata, validation failures, lead contact |
| **Necesita aclaración sin respuesta** | `status == "needs_clarification"` + 48h sin respuesta | Escalar a "Clarification Queue" | Original submission, clarification request, lead context |
| **Lead solicita ayuda humana explícita** | Reply contiene "hablar con persona" o similar | Transferencia directa a queue de soporte | Lead data + context of request + document status |
| **Fallo técnico repetido en validación** | 3 fallos consecutivos evaluando mismas reglas | Marcar como `needs_review`, notificar a dev | Error log, rule evaluation trace, raw submission |
| **Compliance exception requerida** | Documento prohibido en región pero lead insiste | Escalar a equipo legal/compliance | Full trace, regional config, lead consent history |

### Appendix E: Configuración Multi-Partner (Ejemplo)

```yaml
# config/partners/acme_legal_v1.yaml
partner_id: acme_legal
niche: legal
brand_name: "Acme Legal Services"

document_types_enabled:
  - dsar_form
  - client_intake_form
  - power_of_attorney

delivery_methods:
  dsar_form:
    upload_link: "https://secure.acmelegal.com/upload/dsar"
    email: "docs@acmelegal.com"
    whatsapp: "reply_with_photo"
  client_intake_form:
    upload_link: "https://secure.acmelegal.com/upload/intake"
    email: "newclient@acmelegal.com"

milestone_payments:
  dsar_form:
    enabled: true
    name: "DSAR Form Submission"
    amount: 125.00
    currency: "USD"
    webhook_url: "https://billing.acmelegal.com/webhooks/milestone"
  client_intake_form:
    enabled: true
    name: "Client Intake Completed"
    amount: 75.00
    currency: "USD"
    webhook_url: "https://billing.acmelegal.com/webhooks/milestone"

compliance:
  region: US
  consent_requirements:
    dsar_form: "gdpr_ccpa_explicit"
    client_intake_form: "general_terms_acceptance"
  retention_policies:
    dsar_form:
      metadata_retention_days: 2555  # 7 años legal
      content_retention_days: 0
      cleanup_action: "delete_metadata"
    client_intake_form:
      metadata_retention_days: 1825  # 5 años
      content_retention_days: 0
      cleanup_action: "delete_metadata"

notification_preferences:
  rep_notification_channel: "email"
  rep_notification_recipients: ["intake-team@acmelegal.com"]
  escalate_on_overdue: true
  escalate_on_rejection: false
```

---

## ✅ Aprobaciones

| Rol | Nombre | Fecha | Firma |
|-----|--------|-------|-------|
| Product Owner | | | |
| Tech Lead | | | |
| Compliance Officer | | | |
| Revenue Ops Lead | | | |

---

**Documento generado:** 20 de febrero de 2026  
**Próxima revisión:** 20 de marzo de 2026  
**Status:** Ready for Implementation

---

> **Nota final para el equipo:**  
> Este SPEC define el cuarto Android de cxEngine: el motor que convierte fricción documental en revenue automatizado, sin violar compliance ni perder momentum de venta.  
> Cada componente debe reutilizar cxEngine_Shared_Components_v1 — nada de duplicar lógica.  
> La clave del éxito no es la IA, es la **validación estructural confiable** + **compliance documental estricto** + **trigger de payment determinista**.  
> Si algo no encaja en este spec, pregunta: "¿Esto ayuda a recolectar documentos de forma compliant y medible?" antes de implementarlo.