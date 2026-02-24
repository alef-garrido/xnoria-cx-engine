# 📄 Product Requirements Document (PRD) Técnico

## cxEngine_Compliance_Guardrails_v1
**Versión:** 1.0  
**Fecha:** 20 de febrero de 2026  
**Estado:** Especificación para implementación  
**Dominio:** cxEngine / Compliance Transversal  
**Stack referencia:** Agnóstico (n8n/Make/Zapier + LLM + SMS/WhatsApp + CRM + Storage)

---

## 1. Overview Técnico

### 1.1 Propósito del módulo
Proveer una capa de guardrails de compliance transversal a todos los Androids de cxEngine, garantizando cumplimiento automático con regulaciones de comunicación (TCPA, GDPR, LGPD, etc.), manejo ético de datos y protección reputacional, sin intervención humana ni duplicación de lógica por Android.

### 1.2 Arquitectura de alto nivel
```
[Android Workflows]
       ↓
[Compliance Guardrails Layer] ←→ [Regulatory Config Store]
       ↓
[Channel Adapters] → [External Systems]
```

### 1.3 Principios técnicos fundamentales
- **Fail-safe por diseño:** Ante duda de compliance, bloquear acción (no arriesgar)
- **Configuración sobre código:** Reglas por región/nicho, no hardcodeadas
- **Auditoría nativa:** Todo bloqueo/permiso deja traza justificada
- **Consentimiento primero:** Verificación pre-send obligatoria, no post-hoc
- **Separación de concerns:** Compliance no decide negocio, solo valida límites

---

## 2. Problem Statement (Técnico)

### 2.1 Problema actual
Los sistemas de automatización de ventas enfrentan riesgos legales y reputacionales debido a:
- **Consentimiento reactivo:** Validación de TCPA/GDPR después del envío, no antes
- **Reglas fragmentadas:** Cada Android implementa su propia lógica de compliance
- **Opt-out inconsistente:** Listas de baja no sincronizadas entre canales/Androids
- **PII expuesta:** Datos sensibles en logs sin masking adecuado
- **Retención indefinida:** Datos almacenados más allá de lo permitido por regulación

### 2.2 Impacto técnico
- Exposición legal: multas TCPA hasta $1,500 por mensaje no compliant
- Riesgo reputacional: bloqueo de números, daño a marca
- Costo operativo: revisión manual de compliance por campaña
- Fragmentación: imposible auditar compliance cross-Android
- Escalabilidad limitada: cada nuevo nicho/región requiere re-implementación

---

## 3. Technical Objectives

### 3.1 Objetivos SMART

| ID | Objetivo | Métrica | Target |
|----|----------|---------|--------|
| **TO-1** | Validar consentimiento pre-send en 100% de envíos | Pre-send verification rate | 100% |
| **TO-2** | Procesar opt-outs en <60 segundos desde recepción | Opt-out processing latency P95 | ≤ 60s |
| **TO-3** | Mantener listas de compliance sincronizadas cross-Android | Sync consistency rate | ≥ 99.9% |
| **TO-4** | Masking automático de PII en 100% de logs | PII masking coverage | 100% |
| **TO-5** | Escalar a 10 regiones con configuración, no código | Time to add region | < 4 horas |

### 3.2 Requerimientos de calidad (NFRs)

| NFR | Descripción | Target |
|-----|-------------|--------|
| **NFR-1** | Disponibilidad del servicio de compliance | 99.99% uptime |
| **NFR-2** | Latencia de validación pre-send | P95 ≤ 50ms |
| **NFR-3** | Consistencia de listas de opt-out | Strong consistency (≤ 1s propagation) |
| **NFR-4** | Retención de trazas de compliance | 7 años (regulatorio) |
| **NFR-5** | Inmutabilidad de registros de auditoría | Write-once, append-only |

---

## 4. Scope Técnico

### 4.1 In Scope

| Componente | Descripción | Responsabilidad |
|------------|-------------|-----------------|
| **Consent Registry** | Almacén centralizado de consentimientos por lead/canal/tipo | Estado |
| **Opt-Out Manager** | Gestión unificada de bajas cross-canal con propagación inmediata | Estado |
| **Pre-Send Validator** | Validación obligatoria antes de cada envío (consent, frecuencia, horario) | Decisión |
| **Frequency Limiter** | Control de límites de mensajes por lead/canal/ventana temporal | Estado |
| **Time Window Validator** | Validación de horarios permitidos por región y tipo de mensaje | Decisión |
| **PII Masker** | Masking automático de datos sensibles en logs y trazas | Transformación |
| **Retention Scheduler** | Programación y ejecución de políticas de retención/eliminación | Estado |
| **Audit Logger** | Registro inmutable de todas las decisiones de compliance | Observabilidad |
| **Regulatory Config Store** | Configuración de reglas por región/nicho/tipo de documento | Configuración |

### 4.2 Out of Scope

| Componente | Razón de exclusión |
|------------|-------------------|
| Generación de documentos legales (Términos, Privacidad) | Fuera de scope de automatización |
| Gestión de disputas legales o reclamos | Proceso humano/legal separado |
| Verificación de identidad (KYC/AML) | Integración con proveedores externos en v2 |
| Encriptación de datos en reposo | Responsabilidad de infraestructura del socio |
| Auditorías externas o certificaciones | Proceso post-implementación |

---

## 5. Functional Requirements

### 5.1 FR-1: Consent Registry & Verification

**ID:** FR-1  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe almacenar y verificar consentimientos explícitos para cada lead, canal y tipo de comunicación.

**User Story:**  
Como sistema de compliance, quiero verificar que existe consentimiento explícito antes de cada envío, para cumplir con TCPA/GDPR y proteger al negocio de multas.

**Acceptance Criteria:**
- [ ] Almacenamiento de consentimientos con: `lead_id`, `channel`, `consent_type`, `timestamp`, `source`, `expiry`
- [ ] Verificación pre-send: `has_valid_consent(lead_id, channel, consent_type) → boolean`
- [ ] Soporte para múltiples tipos de consentimiento: `marketing`, `transactional`, `document_handling`, `sensitive_data`
- [ ] Expiración automática de consentimientos según configuración por región
- [ ] Auditoría de cada verificación: quién, cuándo, qué tipo, resultado
- [ ] API idempotente para registrar consentimiento: mismos inputs = mismo registro

**Pseudocódigo de verificación:**
```
FUNCTION has_valid_consent(lead_id: STRING, channel: ENUM, consent_type: ENUM, context: OBJECT) → ConsentVerification:
    // Buscar consentimiento activo
    consent = CONSENT_REGISTRY.find_active(
        lead_id: lead_id,
        channel: channel,
        consent_type: consent_type,
        region: context.region
    )
    
    IF consent == NULL:
        RETURN {
            valid: false,
            reason: "no_consent_recorded",
            action: "block_and_request_consent"
        }
    
    // Verificar expiración
    IF consent.expiry AND NOW() > consent.expiry:
        RETURN {
            valid: false,
            reason: "consent_expired",
            action: "block_and_refresh_consent"
        }
    
    // Verificar revocación (opt-out)
    IF OPT_OUT_MANAGER.is_opted_out(lead_id, channel):
        RETURN {
            valid: false,
            reason: "consent_revoked_via_optout",
            action: "block_no_retry"
        }
    
    // Consentimiento válido
    AUDIT_LOGGER.log_consent_check({
        lead_id: lead_id,
        channel: channel,
        consent_type: consent_type,
        result: "valid",
        consent_id: consent.id,
        timestamp: NOW()
    })
    
    RETURN {valid: true, reason: "consent_verified", consent_id: consent.id}
```

**ConsentRecord Schema:**
```
OBJECT ConsentRecord:
    consent_id: STRING (UUID)
    lead_id: STRING
    channel: ENUM ["sms", "whatsapp", "email", "voice"]
    consent_type: ENUM ["marketing", "transactional", "document_handling", "sensitive_data"]
    granted_at: ISO-8601
    source: ENUM ["web_form", "verbal_recorded", "document_signed", "import_legacy"]
    expiry: ISO-8601 OR NULL
    region: STRING (ISO country code)
    metadata: OBJECT
        ip_address: STRING OR NULL
        user_agent: STRING OR NULL
        form_version: STRING OR NULL
        recording_id: STRING OR NULL
    revoked_at: ISO-8601 OR NULL
    revoked_reason: STRING OR NULL
```

---

### 5.2 FR-2: Opt-Out Manager & Cross-Channel Sync

**ID:** FR-2  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe gestionar solicitudes de opt-out y propagarlas inmediatamente a todos los canales y Androids.

**User Story:**  
Como operador de compliance, quiero que una solicitud de baja se procese en <60 segundos y bloquee futuros envíos en todos los canales, para cumplir con regulaciones y proteger la reputación.

**Acceptance Criteria:**
- [ ] Detección de keywords de opt-out por idioma/región: `config.optout_keywords`
- [ ] Procesamiento de opt-out en <60 segundos desde recepción
- [ ] Propagación cross-channel: opt-out en SMS bloquea WhatsApp/email del mismo lead
- [ ] Confirmación obligatoria de opt-out al usuario (mensaje estándar por canal)
- [ ] Lista de opt-out centralizada con búsqueda O(1)
- [ ] Soporte para re-opt-in explícito (proceso separado, no automático)
- [ ] Auditoría de cada opt-out: lead, canal, timestamp, keyword detectada

**Pseudocódigo de procesamiento:**
```
FUNCTION process_optout_request(raw_message: OBJECT, channel: ENUM) → OptOutResult:
    // Normalizar mensaje y detectar keyword
    normalized = normalize_message(raw_message)
    keyword = DETECT_OPTOUT_KEYWORD(normalized.body, CONFIG.optout_keywords)
    
    IF keyword == NULL:
        RETURN {detected: false, reason: "no_keyword_matched"}
    
    // Extraer lead_id del mensaje
    lead_id = EXTRACT_LEAD_IDENTIFIER(raw_message, channel)
    IF lead_id == NULL:
        LOG: "optout_orphan", channel: channel, raw: raw_message
        RETURN {detected: true, error: "lead_not_identified"}
    
    // Registrar opt-out centralizado
    optout_record = {
        optout_id: GENERATE_UUID(),
        lead_id: lead_id,
        channel: channel,  // canal donde se solicitó
        keyword_detected: keyword,
        requested_at: NOW(),
        processed_at: NOW(),
        confirmation_sent: false
    }
    
    // Agregar a lista central (propagación inmediata)
    OPT_OUT_REGISTRY.add(optout_record)
    
    // Propagar a otros canales del mismo lead (si aplica)
    IF CONFIG.cross_channel_optout:
        other_channels = GET_LEAD_CHANNELS(lead_id) - {channel}
        FOR EACH ch IN other_channels:
            OPT_OUT_REGISTRY.add({
                lead_id: lead_id,
                channel: ch,
                source: "propagated_from_" + channel,
                requested_at: optout_record.requested_at,
                processed_at: NOW()
            })
    
    // Enviar confirmación obligatoria
    confirmation = CONFIG.optout_confirmation_templates[channel]
    CHANNEL_ADAPTER.send_message(lead_id, channel, {type: "text", body: confirmation})
    UPDATE optout_record.confirmation_sent = true
    
    // Actualizar consentimientos relacionados
    CONSENT_REGISTRY.revoke_related(lead_id, channel)
    
    // Auditoría
    AUDIT_LOGGER.log_optout_processed(optout_record)
    
    LOG: "optout_processed", lead_id: lead_id, channels_affected: 1 + other_channels.length
    EMIT_METRIC: "optout_processed", value: 1, tags: {channel: channel, region: CONFIG.region}
    
    RETURN {
        detected: true,
        optout_id: optout_record.optout_id,
        channels_blocked: [channel] + other_channels,
        confirmation_sent: true
    }
```

---

### 5.3 FR-3: Pre-Send Validation Gateway

**ID:** FR-3  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe validar cada intento de envío contra reglas de compliance antes de permitir la ejecución.

**User Story:**  
Como gateway de compliance, quiero interceptar cada intento de envío y validar consentimiento, frecuencia, horario y contenido, para bloquear proactivamente envíos no compliant.

**Acceptance Criteria:**
- [ ] Validación obligatoria pre-send para todos los Androids/canales
- [ ] Reglas evaluadas en orden: consent → opt-out → frequency → time_window → content
- [ ] Resultado estructurado: `allowed`, `blocked`, `requires_review` con razón y rule_id
- [ ] Logging de cada validación (éxito y fallo) para auditoría
- [ ] Fallback seguro: si validación falla → bloquear (fail-safe)
- [ ] Métricas de bloqueos por regla para optimización

**ValidationRule Schema:**
```
OBJECT ComplianceRule:
    rule_id: STRING (unique identifier)
    rule_type: ENUM ["consent_check", "optout_check", "frequency_limit", "time_window", "content_filter"]
    region: STRING OR NULL (NULL = global)
    channel: ENUM OR NULL (NULL = all channels)
    parameters: OBJECT (rule-specific config)
    action_on_violation: ENUM ["block", "warn_and_allow", "queue_for_review"]
    error_message: STRING (human-readable explanation)
```

**Pseudocódigo de validación:**
```
FUNCTION validate_pre_send(lead_id: STRING, channel: ENUM, content: OBJECT, context: OBJECT) → ValidationDecision:
    // Cargar reglas aplicables
    rules = LOAD_APPLICABLE_RULES(context.region, channel, content.type)
    
    // Evaluar reglas en orden de prioridad
    FOR EACH rule IN rules ORDER BY priority:
        result = EVALUATE_RULE(rule, lead_id, channel, content, context)
        
        IF result.violated:
            // Registrar violación
            AUDIT_LOGGER.log_rule_violation({
                rule_id: rule.rule_id,
                lead_id: lead_id,
                channel: channel,
                content_hash: HASH(content),
                timestamp: NOW(),
                reason: rule.error_message
            })
            
            // Aplicar acción configurada
            SWITCH rule.action_on_violation:
                CASE "block":
                    EMIT_METRIC: "compliance_blocked", value: 1, tags: {rule: rule.rule_id, region: context.region}
                    RETURN {
                        allowed: false,
                        reason: rule.error_message,
                        rule_id: rule.rule_id,
                        action: "block_no_retry"
                    }
                CASE "warn_and_allow":
                    EMIT_METRIC: "compliance_warning", value: 1, tags: {rule: rule.rule_id}
                    // Continuar evaluación
                CASE "queue_for_review":
                    ENQUEUE_FOR_REVIEW(lead_id, content, rule)
                    RETURN {
                        allowed: false,
                        reason: rule.error_message,
                        rule_id: rule.rule_id,
                        action: "queue_for_human_review"
                    }
    
    // Todas las reglas pasaron
    AUDIT_LOGGER.log_validation_passed({
        lead_id: lead_id,
        channel: channel,
        rules_evaluated: rules.length,
        timestamp: NOW()
    })
    
    RETURN {allowed: true, reason: "all_checks_passed", rules_evaluated: rules.length}

// Ejemplo de evaluación de regla de frecuencia
FUNCTION evaluate_frequency_limit(rule: ComplianceRule, lead_id: STRING, channel: ENUM) → RuleEvaluation:
    window = rule.parameters.time_window_hours
    max_messages = rule.parameters.max_per_window
    
    count = COUNT_RECENT_MESSAGES(lead_id, channel, window: window)
    
    IF count >= max_messages:
        RETURN {
            violated: true,
            details: {current_count: count, limit: max_messages, window: window}
        }
    ELSE:
        RETURN {violated: false}
```

---

### 5.4 FR-4: Frequency & Rate Limiting Engine

**ID:** FR-4  
**Prioridad:** ALTA  
**Descripción:** El sistema debe controlar límites de frecuencia de mensajes por lead, canal y ventana temporal para evitar spam y cumplir regulaciones.

**User Story:**  
Como sistema de rate limiting, quiero controlar cuántos mensajes puede recibir un lead en una ventana dada, para cumplir con TCPA/GDPR y mantener engagement sin fatiga.

**Acceptance Criteria:**
- [ ] Configuración de límites por: región, canal, tipo de mensaje, nicho
- [ ] Conteo en tiempo real con ventana deslizante (sliding window)
- [ ] Excepciones configurables: mensajes transaccionales, respuestas a opt-in reciente
- [ ] Reset automático de contadores al finalizar ventana
- [ ] Métricas de límites alcanzados para optimización de campañas
- [ ] Logging de cada verificación de frecuencia

**Pseudocódigo de rate limiting:**
```
CLASS FrequencyLimiter:
    // Configuración por contexto
    config: FrequencyConfig  // loaded from Regulatory Config Store
    
    FUNCTION can_send(lead_id: STRING, channel: ENUM, message_type: ENUM, context: OBJECT) → RateLimitDecision:
        // Obtener configuración aplicable
        limit_config = FIND_LIMIT_CONFIG(
            region: context.region,
            channel: channel,
            message_type: message_type,
            niche: context.niche
        )
        
        IF limit_config == NULL:
            // Sin configuración = usar defaults conservadores
            limit_config = DEFAULT_LIMITS[channel]
        
        // Calcular ventana actual
        window_start = NOW() - limit_config.window_duration
        window_key = BUILD_WINDOW_KEY(lead_id, channel, window_start)
        
        // Obtener contador actual (cache-first)
        current_count = CACHE.get(window_key) OR QUERY_MESSAGE_COUNT(lead_id, channel, window_start, NOW())
        
        // Verificar contra límite
        IF current_count >= limit_config.max_messages:
            // Calcular cuándo se resetea
            reset_time = CALCULATE_WINDOW_RESET(window_start, limit_config.window_duration)
            
            AUDIT_LOGGER.log_rate_limit_hit({
                lead_id: lead_id,
                channel: channel,
                current_count: current_count,
                limit: limit_config.max_messages,
                reset_at: reset_time
            })
            
            RETURN {
                allowed: false,
                reason: "frequency_limit_exceeded",
                retry_after_seconds: (reset_time - NOW()).seconds,
                current_count: current_count,
                limit: limit_config.max_messages
            }
        
        // Permitir envío y actualizar contador
        CACHE.increment(window_key, ttl: limit_config.window_duration)
        EMIT_METRIC: "message_sent", value: 1, tags: {channel: channel, type: message_type}
        
        RETURN {allowed: true, remaining: limit_config.max_messages - current_count - 1}
```

---

### 5.5 FR-5: Time Window Validator (Horarios Permitidos)

**ID:** FR-5  
**Prioridad:** ALTA  
**Descripción:** El sistema debe validar que los envíos ocurran dentro de horarios permitidos por región y tipo de mensaje.

**User Story:**  
Como sistema de validación horaria, quiero bloquear envíos fuera de ventanas permitidas por regulación local, para cumplir con TCPA (US), LGPD (BR), y otras normas de contacto.

**Acceptance Criteria:**
- [ ] Configuración de horarios permitidos por región: `allowed_contact_hours[region][day_of_week]`
- [ ] Detección de timezone del lead o fallback a región
- [ ] Consideración de días festivos por región
- [ ] Excepciones configurables: respuestas a opt-in reciente (<24h), mensajes transaccionales
- [ ] Queue automático para envío en próxima ventana permitida (no reintentar inmediatamente)
- [ ] Logging de bloqueos por horario para auditoría

**Pseudocódigo de validación horaria:**
```
FUNCTION is_within_allowed_hours(lead_id: STRING, region: STRING, message_type: ENUM) → TimeWindowDecision:
    // Determinar timezone
    timezone = GET_LEAD_TIMEZONE(lead_id) OR CONFIG.default_timezones[region]
    
    // Obtener configuración de horarios
    hours_config = CONFIG.allowed_contact_hours[region]
    IF hours_config == NULL:
        LOG: "no_hours_config", region: region
        RETURN {allowed: false, reason: "no_hours_configured_for_region"}
    
    // Verificar día festivo
    IF IS_HOLIDAY(NOW(), region, timezone):
        IF hours_config.holidays == "closed":
            next_window = FIND_NEXT_BUSINESS_DAY(NOW(), region, timezone)
            RETURN {
                allowed: false,
                reason: "holiday_no_contact",
                next_allowed_at: next_window
            }
    
    // Obtener horario para día actual
    day_of_week = GET_DAY_OF_WEEK(NOW(), timezone)
    day_hours = hours_config[day_of_week]
    
    IF day_hours == NULL OR day_hours == "closed":
        next_window = FIND_NEXT_ALLOWED_DAY(NOW(), region, timezone, hours_config)
        RETURN {
            allowed: false,
            reason: "day_not_allowed",
            next_allowed_at: next_window
        }
    
    // Verificar hora actual
    current_time = GET_TIME_IN_TIMEZONE(NOW(), timezone)
    start_time = PARSE_TIME(day_hours.start)
    end_time = PARSE_TIME(day_hours.end)
    
    IF current_time >= start_time AND current_time < end_time:
        RETURN {allowed: true, reason: "within_allowed_hours"}
    ELSE:
        // Calcular próxima ventana
        IF current_time < start_time:
            next_window = SET_TIME(NOW(), start_time, timezone)
        ELSE:
            next_window = FIND_NEXT_DAY_WINDOW(NOW(), region, timezone, hours_config)
        
        RETURN {
            allowed: false,
            reason: current_time < start_time ? "before_allowed_hours" : "after_allowed_hours",
            next_allowed_at: next_window
        }
```

---

### 5.6 FR-6: PII Masker & Data Protection

**ID:** FR-6  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe aplicar masking automático a datos personales sensibles en logs, trazas y métricas.

**User Story:**  
Como sistema de protección de datos, quiero enmascarar automáticamente PII en todos los registros, para cumplir con GDPR/CCPA y proteger la privacidad de los leads.

**Acceptance Criteria:**
- [ ] Detección automática de campos PII: phone, email, name, id_number, address
- [ ] Masking configurable por tipo: `hash`, `tokenize`, `partial_mask`, `redact`
- [ ] Aplicación transparente: componentes no necesitan modificar código para usar masking
- [ ] Capacidad de des-masking controlada para auditorías autorizadas (role-based)
- [ ] Logging de accesos a datos no masked para auditoría de auditoría
- [ ] Configuración por región: algunos países requieren masking más estricto

**Pseudocódigo de masking:**
```
CLASS PIIMasker:
    // Configuración de masking por campo y región
    masking_rules: OBJECT  // loaded from Regulatory Config Store
    
    FUNCTION mask_payload(payload: OBJECT, context: OBJECT, access_level: ENUM) → MaskedObject:
        // Determinar nivel de masking
        masking_level = DETERMINE_MASKING_LEVEL(context.region, access_level)
        
        // Recorrer payload y aplicar masking
        RETURN MASK_FIELDS(payload, masking_rules[masking_level])
    
    FUNCTION MASK_FIELDS(obj: ANY, rules: OBJECT) → ANY:
        IF obj IS OBJECT:
            result = {}
            FOR EACH key, value IN obj:
                IF key IN rules.fields_to_mask:
                    result[key] = APPLY_MASK(value, rules.masking_strategy[key])
                ELSE IF value IS OBJECT OR value IS ARRAY:
                    result[key] = MASK_FIELDS(value, rules)  // recurse
                ELSE:
                    result[key] = value
            RETURN result
        ELSE IF obj IS ARRAY:
            RETURN obj.map(item => MASK_FIELDS(item, rules))
        ELSE:
            RETURN obj  // primitive, no masking needed
    
    FUNCTION APPLY_MASK(value: STRING, strategy: ENUM) → STRING:
        SWITCH strategy:
            CASE "hash":
                RETURN SHA256(value).substring(0, 16) + "..."
            CASE "tokenize":
                RETURN TOKEN_REGISTRY.get_or_create(value)  // reversible with key
            CASE "partial_mask_phone":
                // +521234567890 → +52***567890
                RETURN value.substring(0, 4) + "***" + value.substring(-6)
            CASE "partial_mask_email":
                // user@example.com → u***@***.com
                parts = value.split("@")
                RETURN parts[0][0] + "***@" + "***." + parts[1].split(".")[-1]
            CASE "redact":
                RETURN "[REDACTED]"
            DEFAULT:
                RETURN value
```

---

### 5.7 FR-7: Retention Scheduler & Data Lifecycle

**ID:** FR-7  
**Prioridad:** ALTA  
**Descripción:** El sistema debe gestionar el ciclo de vida de datos según políticas de retención por región y tipo de dato.

**User Story:**  
Como sistema de gestión de datos, quiero eliminar o anonimizar automáticamente datos que exceden su período de retención permitido, para cumplir con regulaciones de privacidad y minimizar riesgo.

**Acceptance Criteria:**
- [ ] Configuración de políticas de retención por: región, tipo de dato, tipo de lead
- [ ] Programación automática de tareas de limpieza (cron-like)
- [ ] Acciones soportadas: `delete`, `anonymize`, `archive_cold_storage`
- [ ] Notificación pre-eliminación para datos críticos (configurable)
- [ ] Registro inmutable de eliminaciones para auditoría regulatoria
- [ ] Capacidad de "derecho al olvido": eliminación inmediata bajo solicitud

**Pseudocódigo de gestión de retención:**
```
CLASS RetentionManager:
    // Políticas cargadas desde configuración
    policies: RetentionPolicyConfig
    
    FUNCTION schedule_retention(record: DataRecord, context: OBJECT) → RetentionSchedule:
        // Determinar política aplicable
        policy = FIND_RETENTION_POLICY(
            data_type: record.type,
            region: context.region,
            niche: context.niche,
            sensitivity: record.sensitivity_level
        )
        
        IF policy == NULL:
            // Default conservador
            policy = DEFAULT_RETENTION_POLICIES[record.type]
        
        // Calcular fecha de expiración
        expiry_date = CALCULATE_EXPIRY(record.created_at, policy.retention_period)
        
        // Programar tarea de limpieza
        cleanup_task = {
            task_id: GENERATE_UUID(),
            record_id: record.id,
            data_type: record.type,
            action: policy.cleanup_action,  // delete | anonymize | archive
            scheduled_at: expiry_date,
            created_at: NOW(),
            status: "scheduled"
        }
        
        SCHEDULER.register_task(cleanup_task)
        
        // Registrar para auditoría
        AUDIT_LOGGER.log_retention_scheduled({
            record_id: record.id,
            policy_id: policy.id,
            expiry_date: expiry_date,
            action: policy.cleanup_action
        })
        
        RETURN {
            scheduled: true,
            task_id: cleanup_task.task_id,
            expiry_date: expiry_date,
            action: policy.cleanup_action
        }
    
    FUNCTION execute_cleanup(task: CleanupTask) → CleanupResult:
        // Verificar que la tarea aún es válida
        record = DATA_STORE.get_record(task.record_id)
        IF record == NULL OR record.already_processed:
            RETURN {status: "skipped", reason: "record_not_found_or_processed"}
        
        // Ejecutar acción configurada
        SWITCH task.action:
            CASE "delete":
                DATA_STORE.delete_record(task.record_id)
                LOG: "record_deleted", record_id: task.record_id
            CASE "anonymize":
                anonymized = ANONYMIZE_RECORD(record)
                DATA_STORE.update_record(task.record_id, anonymized)
                LOG: "record_anonymized", record_id: task.record_id
            CASE "archive_cold_storage":
                ARCHIVE_SERVICE.move_to_cold_storage(task.record_id)
                DATA_STORE.mark_archived(task.record_id)
                LOG: "record_archived", record_id: task.record_id
        
        // Actualizar estado de tarea
        task.status = "completed"
        task.completed_at = NOW()
        SCHEDULER.update_task(task)
        
        // Auditoría final
        AUDIT_LOGGER.log_cleanup_completed({
            task_id: task.task_id,
            record_id: task.record_id,
            action: task.action,
            completed_at: task.completed_at
        })
        
        EMIT_METRIC: "retention_cleanup_executed", value: 1, tags: {action: task.action, data_type: task.data_type}
        
        RETURN {status: "completed", action: task.action}
```

---

### 5.8 FR-8: Audit Logger & Compliance Reporting

**ID:** FR-8  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe mantener un registro inmutable de todas las decisiones de compliance para auditoría regulatoria y análisis.

**User Story:**  
Como auditor o regulador, quiero acceder a un registro completo e inmutable de todas las decisiones de compliance, para verificar cumplimiento y investigar incidentes.

**Acceptance Criteria:**
- [ ] Registro estructurado de: consent checks, opt-outs, pre-send validations, rate limits, time blocks, PII access
- [ ] Inmutabilidad: registros no pueden ser modificados o eliminados (append-only)
- [ ] Búsqueda eficiente por: lead_id, timestamp, rule_id, region, outcome
- [ ] Exportación en formatos estándar: JSON, CSV, PDF para auditorías externas
- [ ] Retención configurable por tipo de registro (mínimo 7 años para regulatorio)
- [ ] Acceso role-based: solo roles autorizados pueden consultar registros completos

**AuditEntry Schema:**
```
OBJECT AuditEntry:
    entry_id: STRING (UUID)
    timestamp: ISO-8601
    event_type: ENUM ["consent_check", "optout_processed", "pre_send_validation", "rate_limit_hit", "time_window_block", "pii_access", "retention_cleanup"]
    lead_id: STRING OR NULL  // masked if PII
    channel: ENUM OR NULL
    region: STRING
    rule_id: STRING OR NULL
    outcome: ENUM ["allowed", "blocked", "warning", "queued_for_review"]
    reason: STRING OR NULL
    metadata: OBJECT  // event-specific details
    actor: OBJECT  // who/what triggered the event
        type: ENUM ["android", "human", "system", "api"]
        identifier: STRING  // android_id, user_id, system_name
    integrity: OBJECT
        hash: STRING  // SHA256 of entry content
        previous_hash: STRING  // for chain verification
        signature: STRING  // cryptographic signature
```

**Pseudocódigo de logging:**
```
CLASS AuditLogger:
    // Backend de almacenamiento (append-only)
    storage: ImmutableLogStore
    
    FUNCTION log_event(entry: AuditEntry) → LogResult:
        // Calcular integridad
        entry.integrity.hash = COMPUTE_HASH(entry)
        entry.integrity.previous_hash = storage.get_latest_hash()
        entry.integrity.signature = SIGN_WITH_PRIVATE_KEY(entry.integrity.hash)
        
        // Escribir en storage append-only
        storage.append(entry)
        
        // Indexar para búsqueda (separado del log inmutable)
        SEARCH_INDEX.index({
            entry_id: entry.entry_id,
            lead_id: entry.lead_id,  // already masked
            event_type: entry.event_type,
            timestamp: entry.timestamp,
            region: entry.region,
            outcome: entry.outcome
        })
        
        // Emitir métrica
        EMIT_METRIC: "audit_event_logged", value: 1, tags: {event_type: entry.event_type, outcome: entry.outcome}
        
        RETURN {logged: true, entry_id: entry.entry_id}
    
    FUNCTION query_audit_log(filters: AuditQueryFilters, access_level: ENUM) → QueryResult:
        // Validar permisos de acceso
        IF NOT ACCESS_CONTROL.can_query(filters, access_level):
            RETURN {error: "insufficient_permissions"}
        
        // Ejecutar consulta en índice (no en log inmutable)
        results = SEARCH_INDEX.query(filters)
        
        // Aplicar masking adicional si acceso limitado
        IF access_level != "full_audit":
            results = APPLY_QUERY_MASKING(results, access_level)
        
        RETURN {results: results, count: results.length}
    
    FUNCTION export_for_audit(query: AuditQueryFilters, format: ENUM, requester: OBJECT) → ExportResult:
        // Registrar solicitud de exportación
        AUDIT_LOGGER.log_event({
            event_type: "audit_export_requested",
            requester: requester,
            query_hash: COMPUTE_HASH(query),
            timestamp: NOW()
        })
        
        // Ejecutar consulta y formatear
        raw_results = query_audit_log(query, access_level: "full_audit")
        formatted = FORMAT_FOR_EXPORT(raw_results, format)
        
        // Firmar exportación
        signed_export = SIGN_EXPORT(formatted, requester.id)
        
        RETURN {
            export_id: GENERATE_UUID(),
            format: format,
            record_count: raw_results.count,
            signed_data: signed_export,
            expires_at: NOW() + 24_HOURS  // links temporales
        }
```

---

## 6. Non-Functional Requirements

### 6.1 Performance

| Métrica | Target | Medición |
|---------|--------|----------|
| Latencia de validación pre-send | P95 ≤ 50ms | End-to-end tracing |
| Propagación de opt-out cross-channel | ≤ 1 segundo | Sync monitoring |
| Búsqueda en audit log (1M entries) | P95 ≤ 200ms | Query performance |
| Throughput de validaciones | ≥ 1000/segundo | Load testing |

### 6.2 Reliability

| Métrica | Target | Estrategia |
|---------|--------|------------|
| Disponibilidad del servicio | 99.99% | Multi-AZ, health checks |
| Consistencia de listas de opt-out | Strong (≤1s propagation) | Distributed cache + pub/sub |
| Durabilidad de audit logs | 100% | Append-only storage + replication |
| Recuperación tras fallo | < 30 segundos | Auto-healing + circuit breakers |

### 6.3 Security

| Requisito | Implementación |
|-----------|----------------|
| Inmutabilidad de audit logs | Cryptographic chaining + write-once storage |
| Acceso a PII | Role-based access control + masking by default |
| Firmas digitales | Private key signing for audit integrity |
| Encriptación en tránsito | TLS 1.3 para todas las comunicaciones |
| Gestión de secretos | Secrets manager, nunca en código/config estático |

### 6.4 Scalability

| Dimensión | Estrategia |
|-----------|------------|
| Horizontal | Stateless validation, distributed cache for state |
| Multi-región | Config store replicado, validación local-first |
| Multi-Android | Shared service, no per-Android deployment |
| Audit log growth | Partitioning by time + cold storage for old data |

---

## 7. Technical Specifications

### 7.1 Contratos de Interfaz (Pseudocódigo)

```
// === Compliance Guardrails Interface ===

INTERFACE ComplianceGuardrails:
    // Validación pre-send obligatoria
    METHOD validate_pre_send(
        lead_id: STRING,
        channel: ENUM,
        content: OBJECT,
        context: OBJECT
    ) → ValidationDecision
    
    // Verificación de consentimiento
    METHOD has_valid_consent(
        lead_id: STRING,
        channel: ENUM,
        consent_type: ENUM,
        context: OBJECT
    ) → ConsentVerification
    
    // Procesamiento de opt-out
    METHOD process_optout_request(
        raw_message: OBJECT,
        channel: ENUM
    ) → OptOutResult
    
    // Verificación de rate limiting
    METHOD can_send_with_rate_limit(
        lead_id: STRING,
        channel: ENUM,
        message_type: ENUM,
        context: OBJECT
    ) → RateLimitDecision
    
    // Verificación de ventana horaria
    METHOD is_within_allowed_hours(
        lead_id: STRING,
        region: STRING,
        message_type: ENUM
    ) → TimeWindowDecision
    
    // Masking de PII
    METHOD mask_payload(
        payload: OBJECT,
        context: OBJECT,
        access_level: ENUM
    ) → MaskedObject
    
    // Programación de retención
    METHOD schedule_retention(
        record: DataRecord,
        context: OBJECT
    ) → RetentionSchedule
    
    // Logging de auditoría
    METHOD log_compliance_event(
        event: AuditEntry
    ) → LogResult

// === Estructuras de Respuesta ===

OBJECT ValidationDecision:
    allowed: BOOLEAN
    reason: STRING OR NULL
    rule_id: STRING OR NULL
    action: ENUM ["proceed", "block", "queue_for_review"]
    retry_after_seconds: INTEGER OR NULL

OBJECT ConsentVerification:
    valid: BOOLEAN
    reason: STRING OR NULL
    consent_id: STRING OR NULL
    action: ENUM ["proceed", "block_and_request", "block_no_retry"]

OBJECT OptOutResult:
    detected: BOOLEAN
    optout_id: STRING OR NULL
    channels_blocked: ARRAY<ENUM> OR NULL
    confirmation_sent: BOOLEAN
    error: STRING OR NULL

OBJECT RateLimitDecision:
    allowed: BOOLEAN
    reason: STRING OR NULL
    retry_after_seconds: INTEGER OR NULL
    current_count: INTEGER
    limit: INTEGER

OBJECT TimeWindowDecision:
    allowed: BOOLEAN
    reason: STRING OR NULL
    next_allowed_at: ISO-8601 OR NULL
```

### 7.2 Configuración por Región (Ejemplo: México)

```yaml
# config/compliance/regions/MX_v1.yaml
region_code: MX
region_name: "México"
default_timezone: "America/Mexico_City"
language: es-MX

consent_requirements:
  marketing:
    requires_explicit: true
    expiry_days: 730  # 2 años
    source_allowed: ["web_form", "verbal_recorded", "document_signed"]
  transactional:
    requires_explicit: false  # implícito por relación comercial
    expiry_days: null  # no expira mientras exista relación
  document_handling:
    requires_explicit: true
    expiry_days: 2555  # 7 años para legal
    source_allowed: ["document_signed"]

optout_config:
  keywords: ["BAJA", "STOP", "CANCELAR", "NO MÁS", "ELIMINAR"]
  confirmation_template: "Has sido dado de baja. Para reactivar, envía ALTA a este número."
  cross_channel_propagation: true
  processing_sla_seconds: 60

frequency_limits:
  sms:
    marketing: {max_per_day: 1, max_per_week: 3, max_per_month: 10}
    transactional: {max_per_day: 5, max_per_week: 20, max_per_month: 50}
  whatsapp:
    marketing: {max_per_day: 1, max_per_week: 3, max_per_month: 10}
    transactional: {max_per_day: 10, max_per_week: 30, max_per_month: 100}
  email:
    marketing: {max_per_day: 2, max_per_week: 7, max_per_month: 30}
    transactional: {max_per_day: 10, max_per_week: 50, max_per_month: 200}

allowed_contact_hours:
  monday_friday: {start: "09:00", end: "20:00"}
  saturday: {start: "10:00", end: "18:00"}
  sunday: "closed"
  holidays: "closed"
  exceptions:
    transactional: {start: "08:00", end: "21:00"}  # más flexible
    emergency: {start: "00:00", end: "23:59"}  # siempre permitido

retention_policies:
  consent_records: {retention_days: 2555, action: "archive_cold_storage"}  # 7 años
  optout_records: {retention_days: 2555, action: "archive_cold_storage"}
  audit_logs: {retention_days: 2555, action: "archive_cold_storage"}
  message_logs: {retention_days: 365, action: "delete"}  # 1 año
  pii_in_cache: {retention_days: 30, action: "delete"}

pii_masking:
  phone: "partial_mask_phone"  # +52***567890
  email: "partial_mask_email"  # u***@***.com
  name: "hash"  # SHA256 prefix
  id_number: "redact"  # [REDACTED]
  address: "redact"
  access_levels:
    support: ["phone_partial", "email_partial"]
    compliance: ["full_with_audit"]
    system: ["full_internal"]

holidays:
  fixed:
    - "01-01"  # Año Nuevo
    - "02-05"  # Constitución
    - "03-21"  # Juárez
    - "05-01"  # Trabajo
    - "09-16"  # Independencia
    - "11-20"  # Revolución
    - "12-25"  # Navidad
  movable:
    # Cálculo de días festivos móviles (ej: primer lunes de mes)
    # Implementado en servicio externo
```

### 7.3 Flujo de Validación Pre-Send (Secuencia)

```
[Android intenta enviar mensaje]
       ↓
[ComplianceGuardrails.validate_pre_send()]
       ↓
[1. Consent Check] → ¿Tiene consentimiento válido? → NO → BLOCK + log
       ↓ SI
[2. Opt-Out Check] → ¿Está en lista de baja? → SÍ → BLOCK + log
       ↓ NO
[3. Frequency Check] → ¿Excede límite de frecuencia? → SÍ → BLOCK + retry_after
       ↓ NO
[4. Time Window Check] → ¿Horario permitido? → NO → QUEUE for next window
       ↓ SÍ
[5. Content Filter] → ¿Contenido permitido? → NO → BLOCK + log
       ↓ SÍ
[✅ ALLOWED] → Registrar validación exitosa → Retornar {allowed: true}
```

---

## 8. Implementation Notes

### 8.1 Stack Reference (n8n como ejemplo)

| Componente | Implementación en n8n | Notas |
|------------|----------------------|-------|
| Consent Registry | Redis node + HTTP API | Cache-first, DB fallback |
| Opt-Out Manager | Redis pub/sub + Code node | Propagación inmediata |
| Pre-Send Validator | Function node + Config store | Evaluación en <50ms |
| Frequency Limiter | Redis sorted sets (sliding window) | Conteo eficiente |
| Time Window Validator | Code node + Luxon/Date-fns | Timezone-aware |
| PII Masker | Function node + masking library | Transparente para componentes |
| Retention Scheduler | Cron trigger + Queue + Code | Tareas programadas |
| Audit Logger | HTTP Request → Immutable store | Append-only, signed |
| Regulatory Config | JSON files + Code loader | Hot-reload sin restart |

### 8.2 Alternative Stacks

| Stack | Ventajas | Consideraciones |
|-------|----------|-----------------|
| **n8n + Redis** | Visual, rápido para prototipar | Redis management overhead |
| **Custom Node.js + PostgreSQL** | Máximo control, ACID compliance | Requiere más desarrollo |
| **Serverless (Lambda + DynamoDB)** | Escalado automático, pago por uso | Cold starts, debugging complejo |
| **Compliance SaaS (OneTrust, etc.)** | Certificaciones incluidas, soporte | Costo, menos flexibilidad |

### 8.3 Environment Variables

```bash
# Compliance Service Config
COMPLIANCE_DEFAULT_REGION=MX
COMPLIANCE_STRICT_MODE=true  # fail-safe: block on any doubt
COMPLIANCE_CACHE_TTL_SECONDS=300

# Consent & Opt-Out
CONSENT_REGISTRY_URL=redis://consent-redis:6379
OPT_OUT_REGISTRY_URL=redis://optout-redis:6379
OPT_OUT_PROPAGATION_ENABLED=true

# Rate Limiting
RATE_LIMIT_CACHE_URL=redis://ratelimit-redis:6379
RATE_LIMIT_WINDOW_PRECISION_SECONDS=60

# Time Validation
TIMEZONE_FALLBACK=America/Mexico_City
HOLIDAYS_API_URL=https://.../holidays

# PII Masking
PII_MASKING_DEFAULT_STRATEGY=partial_mask
PII_ACCESS_AUDIT_ENABLED=true

# Retention
RETENTION_SCHEDULER_ENABLED=true
RETENTION_CLEANUP_BATCH_SIZE=100
RETENTION_COLD_STORAGE_URL=s3://compliance-archive

# Audit Logging
AUDIT_LOG_STORAGE_URL=https://immutable-log-store/append
AUDIT_LOG_PRIVATE_KEY_PATH=/secrets/audit-signing-key.pem
AUDIT_QUERY_INDEX_URL=elasticsearch://audit-index

# Security
COMPLIANCE_API_PRIVATE_KEY_PATH=/secrets/compliance-api-key.pem
COMPLIANCE_TLS_MIN_VERSION=1.3
```

---

## 9. Testing Strategy

### 9.1 Test Cases

| ID | Descripción | Input | Expected Output |
|----|-------------|-------|-----------------|
| **TC-CG-01** | Consentimiento válido pre-send | Lead con consent activo | `allowed: true` |
| **TC-CG-02** | Sin consentimiento | Lead sin consent registrado | `allowed: false`, reason: "no_consent" |
| **TC-CG-03** | Opt-out detectado | Mensaje con keyword "BAJA" | Opt-out procesado, confirmación enviada |
| **TC-CG-04** | Propagación cross-channel | Opt-out en SMS | WhatsApp/email del mismo lead bloqueados |
| **TC-CG-05** | Límite de frecuencia excedido | 4to mensaje marketing en día (límite: 3) | `allowed: false`, retry_after calculado |
| **TC-CG-06** | Envío fuera de horario | Mensaje a las 22:00 en MX | `allowed: false`, next_allowed_at = mañana 9:00 |
| **TC-CG-07** | PII masking en logs | Log con phone +521234567890 | Phone aparece como +52***567890 |
| **TC-CG-08** | Retención programada | Registro creado con política 365 días | Cleanup task scheduled para expiry_date |
| **TC-CG-09** | Auditoría inmutable | 1000 audit entries escritos | Hash chain válida, sin modificaciones posibles |
| **TC-CG-10** | Derecho al olvido | Solicitud de eliminación de lead | Datos eliminados/anonimizados en <24h |

### 9.2 Test Execution

```bash
# Unit tests (componentes individuales)
npm test -- unit/consent-registry/
npm test -- unit/optout-manager/
npm test -- unit/pre-send-validator/
npm test -- unit/pii-masker/

# Integration tests (flujos de compliance)
npm test -- integration/consent-to-send/
npm test -- integration/optout-propagation/
npm test -- integration/audit-chain-integrity/

# Compliance tests (regulaciones)
npm test -- compliance/tcpa-us/
npm test -- compliance/gdpr-eu/
npm test -- compliance/lgpd-br/
npm test -- compliance/lfpdppp-mx/

# Performance tests
npm test -- perf/pre-send-validation-p95/
npm test -- perf/optout-propagation-latency/
npm test -- perf/audit-log-query-1m-entries/

# Security tests
npm test -- security/audit-log-immutability/
npm test -- security/pii-access-control/
npm test -- security/signature-verification/
```

---

## 10. Success Metrics (Técnicos y de Negocio)

| Métrica | Fórmula | Target | Frecuencia |
|---------|---------|--------|------------|
| Pre-send validation success rate | valid_checks / total_checks | ≥ 99.9% | Diario |
| Opt-out processing latency P95 | percentile_95(processed_at - received_at) | ≤ 60s | Diario |
| Cross-channel sync consistency | synced_optouts / total_optouts | ≥ 99.9% | Diario |
| PII masking coverage | masked_fields / total_pii_fields | 100% | Auditoría semanal |
| Compliance block accuracy | correct_blocks / total_blocks | ≥ 99.5% | Semanal |
| Audit log query latency P95 | percentile_95(query_time) | ≤ 200ms | Diario |
| Retention cleanup success | successful_cleanups / scheduled_cleanups | ≥ 99.9% | Mensual |
| False positive rate (legit messages blocked) | false_blocks / total_blocks | ≤ 0.1% | Semanal |

---

## 11. Deployment Checklist

### Pre-Deployment
- [ ] Configuración de regiones validada (`config/compliance/regions/*.yaml`)
- [ ] Consent Registry inicializado con schema correcto
- [ ] Opt-Out Manager con propagación cross-channel testeada
- [ ] PII Masker con reglas por región cargadas
- [ ] Audit Logger con firma criptográfica configurada
- [ ] Retention Scheduler con políticas cargadas y timezone correcto
- [ ] Health checks implementados para todos los componentes
- [ ] Métricas de compliance expuestas y visibles en dashboard
- [ ] Tests de integración de compliance pasando (≥ 95% coverage)
- [ ] Plan de rollback documentado para fallos de validación

### Post-Deployment
- [ ] Monitorizar pre-send validation latency por primeras 1000 validaciones
- [ ] Validar que opt-outs se propagan cross-channel en <60s
- [ ] Confirmar que PII aparece masked en logs de todos los componentes
- [ ] Verificar que audit logs son inmutables (hash chain válida)
- [ ] Revisar métricas de false positives para ajustar reglas si necesario
- [ ] Documentar cualquier desviación de targets de compliance

---

## 12. Version History

| Versión | Fecha | Cambios | Autor |
|---------|-------|---------|-------|
| 1.0 | 2026-02-20 | Especificación inicial de Compliance Guardrails | O.A. Perez Garrido |

---

## 13. Appendices

### Appendix A: Regulatory Reference Matrix

| Regulación | Región | Consentimiento | Opt-Out | Horarios | Retención | Notas |
|------------|--------|---------------|---------|----------|-----------|-------|
| **TCPA** | US | Expreso para marketing | Inmediato, confirmado | 8am-9pm hora local | 4 años mínimo | $500-$1500 por violación |
| **GDPR** | EU | Explícito, granular | Derecho al olvido | Sin restricción horaria | "Mientras sea necesario" | Multas hasta 4% revenue global |
| **LGPD** | BR | Consentimiento libre | Gratuito, fácil | 8am-8pm hora local | Hasta cumplir finalidad | ANPD puede multar |
| **LFPDPPP** | MX | Consentimiento expreso | Mecanismo gratuito | 9am-8pm hora local | Mientras exista relación | INAI puede sancionar |
| **CASL** | CA | Consentimiento implícito/explícito | Incluye en cada mensaje | Sin restricción horaria | Mientras sea necesario | Multas hasta $10M CAD |

### Appendix B: Opt-Out Keyword Library por Idioma

```yaml
optout_keywords:
  es:
    - "BAJA"
    - "STOP"
    - "CANCELAR"
    - "ELIMINAR"
    - "NO MÁS"
    - "SACAR"
    - "QUITAR"
    - "OPT OUT"
  
  en:
    - "STOP"
    - "UNSUBSCRIBE"
    - "CANCEL"
    - "REMOVE"
    - "DELETE"
    - "OPT OUT"
    - "NO MORE"
  
  pt:
    - "PARAR"
    - "CANCELAR"
    - "REMOVER"
    - "EXCLUIR"
    - "NÃO QUERO MAIS"
    - "OPT OUT"
  
  fr:
    - "STOP"
    - "ARRÊTER"
    - "ANNULER"
    - "SUPPRIMER"
    - "JE NE VEUX PLUS"
    - "DÉSINSCRIRE"
```

### Appendix C: Error Codes & Fallbacks

| Código | Descripción | Fallback | Alerta |
|--------|-------------|----------|--------|
| `CONSENT_NOT_FOUND` | No hay consentimiento registrado para lead/canal/tipo | Bloquear envío, loguear | Si > 5% de validaciones fallan por esto |
| `OPT_OUT_PROPAGATION_FAILED` | Fallo al propagar opt-out a otro canal | Reintentar 3x, luego alertar | Inmediata si crítico para compliance |
| `RATE_LIMIT_CONFIG_MISSING` | No hay configuración de límites para contexto | Usar defaults conservadores | Si configuración incompleta detectada |
| `TIMEZONE_RESOLUTION_FAILED` | No se pudo determinar timezone del lead | Fallback a timezone de región | Si > 10% de leads con timezone inválido |
| `PII_MASKING_ERROR` | Fallo al aplicar masking a campo PII | Redactar campo completo ([REDACTED]) | Inmediata si riesgo de exposición |
| `AUDIT_LOG_WRITE_FAILED` | Fallo al escribir en log inmutable | Queue local + retry, luego alertar | Inmediata si crítico para auditoría |
| `RETENTION_POLICY_MISSING` | No hay política para tipo de dato | Aplicar default más conservador | Si config incompleta detectada |

### Appendix D: Integration Guide para Androids

```
// === Cómo integrar Compliance Guardrails en un Android ===

// 1. Importar dependencia
IMPORT ComplianceGuardrails FROM "@cxEngine/compliance-guardrails"

// 2. Inicializar con configuración
const compliance = new ComplianceGuardrails({
  region: CONFIG.region,
  cache_url: process.env.COMPLIANCE_CACHE_URL,
  audit_log_url: process.env.AUDIT_LOG_URL
})

// 3. Validar antes de CADA envío (obligatorio)
async function send_message_safely(lead, channel, content) {
  // Validación pre-send
  const validation = await compliance.validate_pre_send(
    lead.id,
    channel,
    content,
    { niche: lead.niche, region: CONFIG.region }
  )
  
  if (!validation.allowed) {
    // No enviar, registrar razón
    logger.info("Send blocked by compliance", {
      lead_id: lead.id,
      reason: validation.reason,
      rule_id: validation.rule_id
    })
    
    // Acción configurada
    if (validation.action === "queue_for_review") {
      await enqueue_for_human_review(lead, content, validation)
    }
    
    return { sent: false, blocked_by: validation.rule_id }
  }
  
  // ✅ Permitido, proceder con envío
  const result = await channel_adapter.send(lead, channel, content)
  
  // Registrar envío exitoso para rate limiting
  await compliance.record_message_sent(lead.id, channel, content.type)
  
  return { sent: true, message_id: result.message_id }
}

// 4. Procesar respuestas (opt-out detection)
async function handle_incoming_message(raw_message, channel) {
  // Detectar y procesar opt-out automáticamente
  const optout_result = await compliance.process_optout_request(raw_message, channel)
  
  if (optout_result.detected) {
    logger.info("Opt-out processed", {
      lead_id: raw_message.lead_id,
      channels_blocked: optout_result.channels_blocked
    })
    return { handled: true, action: "opt_out_processed" }
  }
  
  // ... continuar con lógica normal del Android
}

// 5. Masking automático en logs
logger.info("Processing lead", {
  // El compliance masker se aplica automáticamente si está habilitado
  lead_id: compliance.mask_payload(lead.id, { region: CONFIG.region }),
  phone: compliance.mask_payload(lead.phone, { region: CONFIG.region })
})
```

### Appendix E: Configuración Multi-Partner (Ejemplo)

```yaml
# config/compliance/partners/acme_legal_v1.yaml
partner_id: acme_legal
region: US
niche: legal

consent_overrides:
  # Requisitos más estrictos para nicho legal
  document_handling:
    requires_explicit: true
    source_allowed: ["document_signed_only"]  # solo firmado, no web form
    expiry_days: 2555  # 7 años obligatorio

optout_config:
  # Confirmación personalizada para marca
  confirmation_template: "You have been unsubscribed from Acme Legal communications. Reply START to re-subscribe."
  # Propagación más agresiva para nicho regulado
  cross_channel_propagation: true
  include_related_services: true  # opt-out aplica a todos los servicios del partner

frequency_limits:
  # Límites más conservadores para legal
  sms:
    marketing: {max_per_day: 1, max_per_week: 2, max_per_month: 5}
    transactional: {max_per_day: 3, max_per_week: 10, max_per_month: 30}

pii_masking:
  # Masking más estricto para datos legales sensibles
  id_number: "redact"  # nunca mostrar, ni parcial
  address: "redact"
  name: "hash"  # no partial, hash completo
  access_levels:
    support: ["phone_redacted", "email_redacted"]  # soporte ve menos
    compliance: ["full_with_audit_trail"]  # compliance ve todo con auditoría

retention_policies:
  # Retención extendida para cumplimiento legal
  consent_records: {retention_days: 2555, action: "archive_cold_storage"}
  message_logs: {retention_days: 2555, action: "archive_cold_storage"}  # 7 años vs default 1 año
  audit_logs: {retention_days: 3650, action: "archive_cold_storage"}  # 10 años para legal

audit_config:
  # Auditoría más detallada para partner regulado
  log_all_validations: true  # no solo bloqueos
  include_content_hash: true  # hash del contenido para trazabilidad
  signer_key_rotation_days: 90  # rotación más frecuente de claves de firma
```

---

## ✅ Aprobaciones

| Rol | Nombre | Fecha | Firma |
|-----|--------|-------|-------|
| Product Owner | | | |
| Tech Lead | | | |
| Compliance Officer | | | |
| Legal Counsel | | | |
| Security Lead | | | |

---

**Documento generado:** 20 de febrero de 2026  
**Próxima revisión:** 20 de marzo de 2026  
**Status:** Ready for Implementation

---

> **Nota final para el equipo:**  
> Este SPEC define la capa de compliance transversal que protege a todos los Androids de cxEngine.  
> No es un "feature" opcional: es el sistema inmunológico del producto.  
> Cada Android DEBE integrar estos guardrails — no hay excepciones.  
> La clave del éxito no es la complejidad, es la **consistencia automática** + **fail-safe por diseño** + **auditoría nativa**.  
> Si algo no encaja en este spec, pregunta: "¿Esto protege al negocio y al usuario de riesgo regulatorio?" antes de implementarlo.