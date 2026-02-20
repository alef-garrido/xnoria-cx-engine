# 📄 Product Requirements Document (PRD) Técnico

## ROYA_Analytics_Metrics_v1
**Versión:** 1.0  
**Fecha:** 20 de febrero de 2026  
**Estado:** Especificación para implementación  
**Dominio:** ROYA / Analytics & Metrics Framework  
**Stack referencia:** Agnóstico (n8n/Make/Zapier + LLM + SMS/WhatsApp + CRM + BI)

---

## 1. Overview Técnico

### 1.1 Propósito del módulo
Proveer un framework unificado de medición, análisis y optimización para todos los Androids de ROYA, permitiendo visibilidad en tiempo real del performance operativo, atribución de revenue y mejora continua basada en datos, sin acoplamiento a stack específico de BI.

### 1.2 Arquitectura de alto nivel
```
[Android Workflows]
       ↓
[Metrics Emission Layer] → [Metrics Collector]
       ↓                           ↓
[Event Bus] → [Aggregation Engine] → [Storage: Time-Series DB]
                                     ↓
[Query API] → [Dashboards / Alerts / Optimization Engine]
```

### 1.3 Principios técnicos fundamentales
- **Metric-first design:** Cada Android emite métricas como parte de su flujo normal, no como post-procesamiento
- **Agnosticismo de backend:** Métricas estructuradas, independientes de Prometheus/Datadog/BigQuery
- **Atribución clara:** Cada métrica incluye contexto suficiente para correlacionar con revenue
- **Privacidad por diseño:** PII nunca en métricas, solo identificadores anonimizados
- **Escalabilidad horizontal:** Métricas emitidas asíncronamente, sin bloquear flujos principales

---

## 2. Problem Statement (Técnico)

### 2.1 Problema actual
Los sistemas de automatización de ventas carecen de visibilidad accionable debido a:
- **Métricas fragmentadas:** Cada Android mide cosas diferentes, imposible comparar performance
- **Atribución opaca:** No se puede correlacionar actividad de Android con revenue generado
- **Latencia de insight:** Datos disponibles horas/días después, no en tiempo real para optimización
- **Privacidad vs. utilidad:** Dilema entre medir bien y proteger datos de leads
- **Costo de instrumentación:** Cada nuevo Android requiere re-implementar logging/metrics

### 2.2 Impacto técnico
- Imposibilidad de optimizar Androids basándose en datos reales
- Decisiones de escalamiento basadas en intuición, no evidencia
- Dificultad para demostrar ROI a socios/comercial
- Riesgo de sobre-optimizar métricas vanidosas vs. métricas de negocio
- Costo marginal alto para agregar nuevos Androids al ecosistema

---

## 3. Technical Objectives

### 3.1 Objetivos SMART

| ID | Objetivo | Métrica | Target |
|----|----------|---------|--------|
| **TO-1** | Emitir métricas estructuradas desde todos los Androids | Metrics coverage rate | 100% de Androids |
| **TO-2** | Correlacionar actividad de Android con revenue atribuido | Attribution accuracy | ≥ 95% |
| **TO-3** | Proveer insights en tiempo real para optimización | Insight latency P95 | ≤ 5 minutos |
| **TO-4** | Mantener privacidad de leads en todas las métricas | PII leakage rate | 0% |
| **TO-5** | Escalar a 10M eventos/mes sin degradación | Throughput sustained | ≥ 10M events/month |

### 3.2 Requerimientos de calidad (NFRs)

| NFR | Descripción | Target |
|-----|-------------|--------|
| **NFR-1** | Disponibilidad del servicio de métricas | 99.9% uptime |
| **NFR-2** | Latencia de emisión de métricas | P95 ≤ 100ms (async) |
| **NFR-3** | Consistencia de agregación | Strong consistency para métricas de revenue |
| **NFR-4** | Retención de datos crudos | 90 días para debugging, 7 años para compliance |
| **NFR-5** | Privacidad por diseño | PII nunca en métricas, solo hashes/anonimizados |

---

## 4. Scope Técnico

### 4.1 In Scope

| Componente | Descripción | Responsabilidad |
|------------|-------------|-----------------|
| **Metrics Emitter Interface** | Interface unificada para emitir métricas desde cualquier Android | Contrato |
| **Event Schema Registry** | Definición de eventos estandarizados por tipo de Android | Especificación |
| **Metrics Collector** | Agregador asíncrono de métricas emitidas | Infraestructura |
| **Aggregation Engine** | Cálculo de métricas derivadas: tasas, promedios, percentiles | Procesamiento |
| **Attribution Mapper** | Correlación de eventos de Android con revenue en CRM | Integración |
| **Query API** | Endpoint para consultar métricas agregadas por filtros | API |
| **Alerting Engine** | Detección de anomalías y triggers de alerta configurables | Monitoreo |
| **Optimization Feedback Loop** | Sugerencias de ajuste basadas en patrones detectados | ML/Reglas |
| **Privacy Guard** | Masking/anonimización automática de datos sensibles en métricas | Seguridad |

### 4.2 Out of Scope

| Componente | Razón de exclusión |
|------------|-------------------|
| Dashboards visuales pre-construidos | Capa de presentación; framework provee API, no UI |
| Machine Learning avanzado para predicción | V2; v1 se enfoca en métricas descriptivas y diagnósticas |
| Integración con herramientas de BI específicas | Agnóstico; el framework emite datos, no los visualiza |
| Gestión de usuarios/roles para acceso a métricas | Responsabilidad del sistema de identidad del socio |
| Exportación masiva de datos crudos para auditoría externa | Proceso separado; framework provee API para consultas |

---

## 5. Functional Requirements

### 5.1 FR-1: Metrics Emitter Interface (Unificada)

**ID:** FR-1  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe proveer una interface simple y consistente para que cualquier Android emita métricas estructuradas.

**User Story:**  
Como desarrollador de un Android, quiero emitir métricas con una llamada simple y estandarizada, para que mis datos sean consistentes con el resto del ecosistema sin esfuerzo adicional.

**Acceptance Criteria:**
- [ ] Interface con métodos: `emit_counter()`, `emit_gauge()`, `emit_histogram()`, `emit_business_metric()`
- [ ] Todas las métricas incluyen tags obligatorios: `android_id`, `partner_id`, `region`, `version`
- [ ] Emisión asíncrona: no bloquea el flujo principal del Android
- [ ] Fallback graceful: si el collector no responde, métrica se loguea localmente para retry
- [ ] Soporte para métricas de negocio con atribución: `revenue_attributed`, `conversion_value`
- [ ] Validación de schema en emisión: métricas mal formadas son rechazadas con error claro

**Pseudocódigo de interface:**
```
INTERFACE MetricsEmitter:
    // Métricas de contador: eventos discretos
    METHOD emit_counter(
        name: STRING,           // e.g., "android.event.lead_contacted"
        value: INTEGER,         // típicamente 1
        tags: OBJECT,           // {android: "sleeping_beauty", channel: "sms", ...}
        timestamp: ISO-8601 OR NULL  // default: NOW()
    ) → EmitResult

    // Métricas de gauge: estado actual (puede subir/bajar)
    METHOD emit_gauge(
        name: STRING,           // e.g., "android.queue.pending_leads"
        value: FLOAT,
        tags: OBJECT,
        timestamp: ISO-8601 OR NULL
    ) → EmitResult

    // Métricas de histograma: distribuciones (latencia, tamaño, etc.)
    METHOD emit_histogram(
        name: STRING,           // e.g., "android.latency.decision_ms"
        value: FLOAT,           // valor observado
        tags: OBJECT,
        timestamp: ISO-8601 OR NULL,
        sample_rate: FLOAT OR NULL  // default: 1.0 (100%)
    ) → EmitResult

    // Métricas de negocio: con atribución de revenue
    METHOD emit_business_metric(
        name: STRING,           // e.g., "android.revenue.attributed"
        value: FLOAT,           // monto en moneda base
        currency: STRING,       // e.g., "USD", "MXN"
        tags: OBJECT,           // debe incluir lead_hash (anonimizado)
        attribution_window_hours: INTEGER,  // ventana para atribuir revenue
        timestamp: ISO-8601 OR NULL
    ) → EmitResult

    // Flush forzado (para shutdown limpio o testing)
    METHOD flush(timeout_ms: INTEGER) → FlushResult
```

**EmitResult Schema:**
```
OBJECT EmitResult:
    success: BOOLEAN
    metric_id: STRING OR NULL  // ID asignado por collector si éxito
    error: STRING OR NULL      // Razón si falló
    retry_scheduled: BOOLEAN   // Si falló pero se programó retry
```

---

### 5.2 FR-2: Event Schema Registry (Estándar por Android)

**ID:** FR-2  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe definir un registro de eventos estandarizados que cada Android debe emitir para garantizar consistencia cross-Android.

**User Story:**  
Como analista, quiero que todos los Androids emitan los mismos tipos de eventos con la misma estructura, para poder comparar performance y agregar datos sin transformaciones ad-hoc.

**Acceptance Criteria:**
- [ ] Schema definido para eventos base: `android.started`, `android.lead_processed`, `android.decision_made`, `android.action_executed`, `android.error_occurred`
- [ ] Schema extendido por tipo de Android: `sleeping_beauty.*`, `speed_to_lead.*`, etc.
- [ ] Validación de schema en tiempo de emisión: eventos mal formados son rechazados
- [ ] Versionado de schemas: `event_schema_version` en cada evento
- [ ] Documentación automática de schemas para descubrimiento

**Base Event Schema (ejemplo):**
```
OBJECT BaseEvent:
    event_id: STRING (UUID)
    event_type: STRING  // e.g., "android.lead_processed"
    event_schema_version: STRING  // e.g., "v1"
    timestamp: ISO-8601
    android_id: STRING  // e.g., "sleeping_beauty"
    partner_id: STRING
    region: STRING
    lead_hash: STRING  // SHA256(lead_id + salt), nunca PII raw
    meta: OBJECT  // event-specific fields
```

**Sleeping Beauty Extended Schema:**
```
OBJECT SB_LeadProcessedEvent EXTENDS BaseEvent:
    event_type: "sleeping_beauty.lead_processed"
    meta: OBJECT
        batch_id: STRING
        kiss_sent: BOOLEAN
        reply_received: BOOLEAN
        qualification_result: ENUM ["qualified", "unqualified", "needs_more_info"]
        appointment_scheduled: BOOLEAN
        processing_time_ms: INTEGER
        followup_count: INTEGER
```

**Pseudocódigo de registro:**
```
CLASS EventSchemaRegistry:
    // Carga schemas desde configuración
    schemas: Map<STRING, EventSchema>  // event_type → schema

    METHOD validate_event(event: OBJECT) → ValidationResult:
        schema = schemas.get(event.event_type)
        IF schema == NULL:
            RETURN {valid: false, error: "unknown_event_type"}
        
        // Validar campos obligatorios
        FOR EACH required_field IN schema.required_fields:
            IF event[required_field] == NULL:
                RETURN {valid: false, error: "missing_required_field: " + required_field}
        
        // Validar tipos de campos
        FOR EACH field, expected_type IN schema.field_types:
            IF event[field] != NULL AND TYPE_OF(event[field]) != expected_type:
                RETURN {valid: false, error: "type_mismatch: " + field}
        
        // Validar enums
        FOR EACH field, allowed_values IN schema.enum_fields:
            IF event[field] != NULL AND event[field] NOT IN allowed_values:
                RETURN {valid: false, error: "invalid_enum_value: " + field}
        
        RETURN {valid: true}

    METHOD register_schema(event_type: STRING, schema: EventSchema) → Void:
        // Solo admin puede registrar schemas
        IF NOT AUTH.is_admin():
            THROW PermissionError
        
        schemas[event_type] = schema
        LOG: "schema_registered", event_type: event_type, version: schema.version
```

---

### 5.3 FR-3: Metrics Collector & Aggregation Engine

**ID:** FR-3  
**Prioridad:** ALTA  
**Descripción:** El sistema debe colectar métricas emitidas y calcular métricas derivadas (tasas, promedios, percentiles) para análisis.

**User Story:**  
Como sistema de análisis, quiero agregar métricas crudas en ventanas de tiempo configurables para producir KPIs accionables sin sobrecargar los Androids con lógica de agregación.

**Acceptance Criteria:**
- [ ] Colecta asíncrona vía queue (Kafka/PubSub/Redis Streams)
- [ ] Agregación en ventanas configurables: 1min, 5min, 1h, 24h
- [ ] Cálculo de métricas derivadas: `response_rate`, `qualification_rate`, `conversion_rate`, `avg_latency`
- [ ] Soporte para agrupación por tags: `by android`, `by partner`, `by region`, `by channel`
- [ ] Retención configurable: datos crudos 90 días, agregados 7 años
- [ ] Exportación a formatos estándar: Prometheus, OpenTelemetry, BigQuery

**Pseudocódigo de agregación:**
```
CLASS AggregationEngine:
    // Configuración de ventanas de agregación
    windows: [60, 300, 3600, 86400]  // segundos

    FUNCTION process_raw_metric(raw: RawMetric) → Void:
        // Validar schema
        IF NOT SchemaRegistry.validate(raw):
            LOG: "invalid_metric", error: SchemaRegistry.last_error
            RETURN
        
        // Enqueue para procesamiento asíncrono
        QUEUE.enqueue("metrics_aggregation", {
            metric: raw,
            processed_at: NOW()
        })

    FUNCTION aggregate_window(window_seconds: INTEGER, filters: OBJECT) → AggregatedMetrics:
        // Consultar datos crudos en ventana
        raw_data = STORAGE.query_raw_metrics({
            time_range: {start: NOW() - window_seconds, end: NOW()},
            filters: filters
        })
        
        // Calcular métricas derivadas
        aggregated = {}
        
        // Contadores: suma por grupo
        FOR EACH counter_metric IN raw_data.counters:
            key = BUILD_GROUP_KEY(counter_metric.tags, filters.group_by)
            aggregated.counters[key] = (aggregated.counters[key] OR 0) + counter_metric.value
        
        // Gauges: promedio por grupo
        FOR EACH gauge_metric IN raw_data.gauges:
            key = BUILD_GROUP_KEY(gauge_metric.tags, filters.group_by)
            aggregated.gauges[key] = CALCULATE_WEIGHTED_AVG(
                aggregated.gauges[key], 
                gauge_metric.value, 
                gauge_metric.weight OR 1
            )
        
        // Histogramas: percentiles por grupo
        FOR EACH hist_metric IN raw_data.histograms:
            key = BUILD_GROUP_KEY(hist_metric.tags, filters.group_by)
            aggregated.histograms[key] = APPEND_TO_HISTOGRAM(
                aggregated.histograms[key] OR [], 
                hist_metric.value
            )
        
        // Calcular tasas (derivadas de contadores)
        IF filters.include_rates:
            aggregated.rates = CALCULATE_RATES(aggregated.counters, window_seconds)
        
        RETURN {
            window_seconds: window_seconds,
            generated_at: NOW(),
            filters: filters,
            metrics: aggregated
        }
```

---

### 5.4 FR-4: Attribution Mapper (Revenue Correlation)

**ID:** FR-4  
**Prioridad:** ALTA  
**Descripción:** El sistema debe correlacionar eventos de Android con revenue generado en el CRM para atribución precisa.

**User Story:**  
Como operador de negocio, quiero saber cuánto revenue generó cada Android para cada socio, para justificar inversión y optimizar asignación de recursos.

**Acceptance Criteria:**
- [ ] Mapeo de `lead_hash` entre eventos de Android y registros de CRM
- [ ] Ventana de atribución configurable: revenue dentro de X horas/días del evento se atribuye
- [ ] Soporte para atribución multi-toque: primer toque, último toque, lineal
- [ ] Exclusión de revenue duplicado: mismo lead, misma venta, no contar dos veces
- [ ] Métricas de atribución: `revenue_attributed`, `roi_per_android`, `cost_per_conversion`
- [ ] Auditoría de atribución: trazabilidad de cómo se calculó cada monto

**Pseudocódigo de atribución:**
```
CLASS AttributionMapper:
    // Configuración de reglas de atribución
    attribution_config: OBJECT
        window_hours: INTEGER  // ventana para atribuir revenue
        model: ENUM ["first_touch", "last_touch", "linear", "time_decay"]
        exclude_duplicates: BOOLEAN

    FUNCTION attribute_revenue(
        android_event: AndroidEvent, 
        crm_conversions: ARRAY<ConversionRecord>
    ) → AttributionResult:
        // Filtrar conversiones en ventana de atribución
        eligible_conversions = crm_conversions.filter(conv => 
            conv.lead_hash == android_event.lead_hash AND
            conv.timestamp >= android_event.timestamp AND
            conv.timestamp <= android_event.timestamp + attribution_config.window_hours * 1_HOUR
        )
        
        IF eligible_conversions.length == 0:
            RETURN {attributed_revenue: 0, conversions: []}
        
        // Aplicar modelo de atribución
        SWITCH attribution_config.model:
            CASE "first_touch":
                // Solo atribuir si este fue el primer evento del Android para este lead
                IF android_event.is_first_android_touch:
                    total_revenue = SUM(eligible_conversions.map(c => c.amount))
                    RETURN {attributed_revenue: total_revenue, conversions: eligible_conversions}
                ELSE:
                    RETURN {attributed_revenue: 0, conversions: []}
            
            CASE "last_touch":
                // Solo atribuir si este fue el último evento antes de la conversión
                IF android_event.is_last_android_touch_before_conversion:
                    total_revenue = SUM(eligible_conversions.map(c => c.amount))
                    RETURN {attributed_revenue: total_revenue, conversions: eligible_conversions}
                ELSE:
                    RETURN {attributed_revenue: 0, conversions: []}
            
            CASE "linear":
                // Dividir revenue equitativamente entre todos los eventos del Android
                android_events_for_lead = GET_ANDROID_EVENTS(android_event.android_id, android_event.lead_hash)
                attribution_weight = 1.0 / android_events_for_lead.length
                total_revenue = SUM(eligible_conversions.map(c => c.amount * attribution_weight))
                RETURN {attributed_revenue: total_revenue, conversions: eligible_conversions, weight: attribution_weight}
            
            CASE "time_decay":
                // Ponderar por cercanía temporal a la conversión
                total_revenue = 0
                FOR EACH conv IN eligible_conversions:
                    time_diff_hours = (conv.timestamp - android_event.timestamp) / 1_HOUR
                    weight = EXP(-time_diff_hours / 24)  // decaimiento exponencial
                    total_revenue += conv.amount * weight
                RETURN {attributed_revenue: total_revenue, conversions: eligible_conversions, model: "time_decay"}
        
        // Default: no atribuir si modelo desconocido
        RETURN {attributed_revenue: 0, conversions: [], error: "unknown_attribution_model"}
```

---

### 5.5 FR-5: Query API & Filtering

**ID:** FR-5  
**Prioridad:** ALTA  
**Descripción:** El sistema debe proveer una API para consultar métricas agregadas con filtros flexibles.

**User Story:**  
Como analista o dashboard, quiero consultar métricas por Android, socio, región o período con una API simple, para construir visualizaciones o alertas sin acceso directo a la base de datos.

**Acceptance Criteria:**
- [ ] Endpoint REST/GraphQL para consultas: `GET /metrics/query`
- [ ] Filtros soportados: `android_id`, `partner_id`, `region`, `channel`, `time_range`, `event_type`
- [ ] Agrupación configurable: `group_by` tags para drill-down
- [ ] Paginación y límites para queries grandes
- [ ] Cache de queries frecuentes para performance
- [ ] Autenticación y autorización por partner/rol

**Query Request Schema:**
```
OBJECT MetricsQuery:
    time_range: OBJECT
        start: ISO-8601
        end: ISO-8601
    filters: OBJECT  // key-value pairs for exact match
        android_id: STRING OR NULL
        partner_id: STRING OR NULL
        region: STRING OR NULL
        channel: ENUM OR NULL
        event_type: STRING OR NULL
    group_by: ARRAY<STRING> OR NULL  // tags to group results by
    metrics: ARRAY<STRING>  // metric names to return
    aggregation_window: ENUM ["1m", "5m", "1h", "24h"]  // default: "1h"
    limit: INTEGER OR NULL  // max results, default: 1000
    offset: INTEGER OR NULL  // for pagination, default: 0
```

**Query Response Schema:**
```
OBJECT MetricsQueryResponse:
    query_id: STRING  // for tracing/caching
    generated_at: ISO-8601
    time_range: OBJECT
        start: ISO-8601
        end: ISO-8601
    filters_applied: OBJECT
    group_by: ARRAY<STRING> OR NULL
    data: ARRAY<OBJECT>  // one entry per group
        group_key: OBJECT  // values of group_by tags
        metrics: OBJECT
            counters: OBJECT  // name → value
            gauges: OBJECT
            histograms: OBJECT  // name → {p50, p90, p99, count}
            rates: OBJECT  // derived rates
    meta: OBJECT
        total_groups: INTEGER
        cached: BOOLEAN
        cache_ttl_seconds: INTEGER OR NULL
```

**Pseudocódigo de query:**
```
CLASS MetricsQueryAPI:
    FUNCTION execute_query(query: MetricsQuery, auth: AuthContext) → QueryResult:
        // Validar permisos
        IF NOT AUTH.can_query_metrics(auth.partner_id, query.filters):
            RETURN {error: "insufficient_permissions"}
        
        // Verificar cache
        cache_key = BUILD_CACHE_KEY(query, auth.partner_id)
        IF query.use_cache:
            cached = CACHE.get(cache_key)
            IF cached != NULL:
                RETURN {data: cached.data, meta: {cached: true, ...}}
        
        // Ejecutar query en aggregation engine
        results = AGGREGATION_ENGINE.query({
            time_range: query.time_range,
            filters: query.filters,
            group_by: query.group_by,
            metrics: query.metrics,
            window: query.aggregation_window
        })
        
        // Aplicar paginación
        paginated = APPLY_PAGINATION(results.data, query.limit, query.offset)
        
        // Guardar en cache si aplica
        IF query.cache_ttl_seconds > 0:
            CACHE.set(cache_key, {data: paginated, ...}, ttl: query.cache_ttl_seconds)
        
        RETURN {
            query_id: GENERATE_UUID(),
            generated_at: NOW(),
            time_range: query.time_range,
            filters_applied: query.filters,
            group_by: query.group_by,
            data: paginated,
            meta: {
                total_groups: results.data.length,
                cached: false,
                cache_ttl_seconds: query.cache_ttl_seconds
            }
        }
```

---

### 5.6 FR-6: Alerting Engine (Anomaly Detection)

**ID:** FR-6  
**Prioridad:** MEDIA  
**Descripción:** El sistema debe detectar anomalías en métricas y disparar alertas configurables.

**User Story:**  
Como operador, quiero ser notificado automáticamente cuando una métrica clave se desvía de lo esperado, para intervenir proactivamente antes de que impacte el negocio.

**Acceptance Criteria:**
- [ ] Definición de alertas vía configuración: `alert_rules.yaml`
- [ ] Condiciones soportadas: `threshold`, `rate_of_change`, `anomaly_detection`, `absence`
- [ ] Canales de notificación: email, Slack, webhook, SMS
- [ ] Deduplicación de alertas: no spam si misma condición persiste
- [ ] Silenciamiento temporal: `maintenance_window` para no alertar durante deploy
- [ ] Auditoría de alertas: qué se disparó, cuándo, a quién

**Alert Rule Schema:**
```
OBJECT AlertRule:
    rule_id: STRING
    name: STRING  // human-readable
    description: STRING
    enabled: BOOLEAN
    severity: ENUM ["info", "warning", "critical"]
    
    // Condición de disparo
    condition: OBJECT
        metric_name: STRING
        operator: ENUM ["gt", "lt", "eq", "neq", "rate_gt", "anomaly"]
        threshold: FLOAT OR NULL
        window_seconds: INTEGER  // ventana para evaluar
        group_by: ARRAY<STRING> OR NULL  // evaluar por grupo
    
    // Acción al disparar
    actions: ARRAY<OBJECT>
        type: ENUM ["email", "slack", "webhook", "sms"]
        recipients: ARRAY<STRING> OR NULL
        webhook_url: STRING OR NULL
        message_template: STRING OR NULL
    
    // Configuración de frecuencia
    cooldown_seconds: INTEGER  // no re-disparar en este período
    repeat_count: INTEGER OR NULL  // cuántas veces repetir antes de silenciar
    
    // Silenciamiento
    silence: OBJECT OR NULL
        start: ISO-8601 OR NULL
        end: ISO-8601 OR NULL
        reason: STRING OR NULL
```

**Pseudocódigo de evaluación:**
```
CLASS AlertingEngine:
    // Cargar reglas desde configuración
    rules: ARRAY<AlertRule>

    FUNCTION evaluate_rules(metrics: AggregatedMetrics) → ARRAY<AlertTriggered>:
        triggered = []
        
        FOR EACH rule IN rules:
            IF NOT rule.enabled OR IS_SILENCED(rule):
                CONTINUE
            
            // Obtener valor de métrica para evaluar
            metric_value = GET_METRIC_VALUE(
                metrics, 
                rule.condition.metric_name, 
                rule.condition.group_by
            )
            
            IF metric_value == NULL:
                CONTINUE  // métrica no disponible, no evaluar
            
            // Evaluar condición
            IF EVALUATE_CONDITION(metric_value, rule.condition):
                // Verificar cooldown
                IF IS_IN_COOLDOWN(rule.rule_id):
                    CONTINUE
                
                // Disparar alerta
                alert = {
                    rule_id: rule.rule_id,
                    triggered_at: NOW(),
                    metric_value: metric_value,
                    condition: rule.condition,
                    severity: rule.severity
                }
                
                // Ejecutar acciones
                FOR EACH action IN rule.actions:
                    SEND_NOTIFICATION(action, alert)
                
                // Registrar para cooldown
                RECORD_ALERT_TRIGGERED(rule.rule_id)
                
                triggered.append(alert)
        
        RETURN triggered

    FUNCTION EVALUATE_CONDITION(value: FLOAT, condition: OBJECT) → BOOLEAN:
        SWITCH condition.operator:
            CASE "gt": RETURN value > condition.threshold
            CASE "lt": RETURN value < condition.threshold
            CASE "eq": RETURN value == condition.threshold
            CASE "neq": RETURN value != condition.threshold
            CASE "rate_gt":
                // Calcular tasa de cambio vs ventana anterior
                prev_value = GET_PREVIOUS_WINDOW_VALUE(condition.metric_name, condition.window_seconds)
                IF prev_value == NULL: RETURN false
                rate = (value - prev_value) / prev_value
                RETURN rate > condition.threshold
            CASE "anomaly":
                // Detectar anomalía estadística (z-score, IQR, etc.)
                RETURN IS_STATISTICAL_ANOMALY(value, condition.metric_name, condition.window_seconds)
            DEFAULT: RETURN false
```

---

### 5.7 FR-7: Optimization Feedback Loop (Sugerencias)

**ID:** FR-7  
**Prioridad:** MEDIA  
**Descripción:** El sistema debe analizar patrones en métricas y sugerir ajustes configurables para mejorar performance.

**User Story:**  
Como optimizador de Androids, quiero recibir sugerencias basadas en datos sobre cómo ajustar parámetros (frecuencia, mensajes, thresholds) para mejorar resultados, sin tener que adivinar o hacer A/B testing manual.

**Acceptance Criteria:**
- [ ] Detección de patrones: correlaciones entre parámetros y métricas de resultado
- [ ] Sugerencias accionables: "aumentar frecuencia de follow-up en 20%", "cambiar threshold de calificación a 75"
- [ ] Explicación de sugerencia: por qué se recomienda, qué datos la respaldan
- [ ] Simulación de impacto: estimación de mejora esperada si se aplica sugerencia
- [ ] Aprobación humana requerida: sugerencias no se aplican automáticamente
- [ ] Historial de sugerencias: qué se recomendó, qué se aplicó, qué resultado tuvo

**Pseudocódigo de optimización:**
```
CLASS OptimizationEngine:
    // Reglas de optimización basadas en heurísticas
    optimization_rules: ARRAY<OptimizationRule>

    FUNCTION analyze_and_suggest(android_id: STRING, partner_id: STRING, time_window: OBJECT) → ARRAY<Suggestion>:
        suggestions = []
        
        // Obtener métricas relevantes
        metrics = QUERY_METRICS({
            android_id: android_id,
            partner_id: partner_id,
            time_range: time_window,
            metrics: ["response_rate", "qualification_rate", "conversion_rate", "opt_out_rate"]
        })
        
        // Aplicar reglas de optimización
        FOR EACH rule IN optimization_rules:
            IF rule.applies_to(android_id, metrics):
                suggestion = rule.generate_suggestion(android_id, metrics)
                IF suggestion != NULL:
                    // Calcular impacto estimado
                    suggestion.estimated_impact = ESTIMATE_IMPACT(suggestion, metrics)
                    // Agregar explicación
                    suggestion.explanation = rule.explain_why(metrics)
                    suggestions.append(suggestion)
        
        // Ordenar por impacto estimado
        suggestions.sort_by(estimated_impact DESC)
        
        RETURN suggestions

// Ejemplo de regla de optimización
CLASS FollowupFrequencyRule EXTENDS OptimizationRule:
    FUNCTION applies_to(android_id: STRING, metrics: OBJECT) → BOOLEAN:
        // Solo aplica a Androids con follow-up
        RETURN android_id IN ["sleeping_beauty", "out_of_hours"] AND
               metrics.response_rate < 0.20  // baja tasa de respuesta
    
    FUNCTION generate_suggestion(android_id: STRING, metrics: OBJECT) → Suggestion OR NULL:
        // Analizar correlación histórica: frecuencia vs response_rate
        historical = QUERY_HISTORICAL_DATA(android_id, "followup_frequency", "response_rate")
        
        IF historical.correlation > 0.5:  // correlación positiva fuerte
            RETURN {
                type: "parameter_adjustment",
                parameter: "followup_frequency",
                current_value: metrics.current_followup_frequency,
                suggested_value: metrics.current_followup_frequency * 1.2,  // +20%
                reason: "response_rate_below_threshold",
                confidence: historical.correlation
            }
        
        RETURN NULL
    
    FUNCTION explain_why(metrics: OBJECT) → STRING:
        RETURN "La tasa de respuesta actual (${metrics.response_rate*100}%) está por debajo del objetivo (20%). " +
               "Análisis histórico muestra que aumentar la frecuencia de follow-up en 20% correlaciona con " +
               "mejora en respuesta sin aumentar opt-outs significativamente."
```

---

### 5.8 FR-8: Privacy Guard (PII Protection in Metrics)

**ID:** FR-8  
**Prioridad:** CRÍTICA  
**Descripción:** El sistema debe garantizar que ninguna PII (información personal identificable) llegue a métricas o logs de analytics.

**User Story:**  
Como oficial de privacidad, quiero que el framework de métricas anonimice automáticamente cualquier dato sensible, para cumplir con GDPR/CCPA sin sacrificar utilidad analítica.

**Acceptance Criteria:**
- [ ] Detección automática de campos PII: `phone`, `email`, `name`, `id_number`, `address`
- [ ] Hashing irreversible con salt por partner: `lead_hash = SHA256(lead_id + partner_salt)`
- [ ] Agregación mínima: métricas nunca exponen datos de menos de K leads (K-anonymity configurable)
- [ ] Masking en logs de debugging: PII reemplazada por `[REDACTED]` o token
- [ ] Auditoría de acceso a métricas con PII potencial: logging de quién consultó qué
- [ ] Derecho al olvido: endpoint para eliminar métricas asociadas a lead específico

**Pseudocódigo de privacidad:**
```
CLASS PrivacyGuard:
    // Configuración de privacidad
    config: OBJECT
        min_group_size: INTEGER  // K-anonymity: no exponer grupos < K
        hash_salt_by_partner: Map<STRING, STRING>  // partner_id → salt
        pii_fields: ARRAY<STRING>  // campos a detectar y proteger

    FUNCTION anonymize_lead_data(lead: OBJECT, partner_id: STRING) → AnonymizedLead:
        // Generar hash irreversible
        salt = config.hash_salt_by_partner[partner_id]
        lead_hash = SHA256(lead.lead_id + salt)
        
        // Eliminar/mask PII
        anonymized = {}
        FOR EACH key, value IN lead:
            IF key IN config.pii_fields:
                // Nunca incluir PII raw en métricas
                CONTINUE
            ELSE:
                anonymized[key] = value
        
        // Agregar identificador anonimizado
        anonymized.lead_hash = lead_hash
        
        RETURN anonymized

    FUNCTION enforce_k_anonymity(metrics: ARRAY<MetricGroup>, min_k: INTEGER) → ARRAY<MetricGroup>:
        filtered = []
        
        FOR EACH group IN metrics:
            // Contar leads únicos en grupo
            unique_leads = COUNT_UNIQUE(group.lead_hashes)
            
            IF unique_leads >= min_k:
                // Grupo suficientemente grande, incluir
                filtered.append(group)
            ELSE:
                // Grupo muy pequeño, excluir para proteger privacidad
                LOG: "group_excluded_k_anonymity", group_key: group.key, size: unique_leads
                // Opcional: agregar a grupo "other" si aplica
        
        RETURN filtered

    FUNCTION redact_pii_in_logs(log_entry: OBJECT) → RedactedLog:
        redacted = DEEP_COPY(log_entry)
        
        // Recorrer y redactar campos PII
        REDACT_FIELDS(redacted, config.pii_fields, "[REDACTED]")
        
        RETURN redacted
```

---

## 6. Non-Functional Requirements

### 6.1 Performance

| Métrica | Target | Medición |
|---------|--------|----------|
| Latencia de emisión de métricas | P95 ≤ 100ms (async) | End-to-end tracing |
| Throughput de colecta | ≥ 10K eventos/segundo | Load testing |
| Latencia de query agregada | P95 ≤ 2 segundos | Query API monitoring |
| Tiempo de atribución | ≤ 5 minutos desde conversión en CRM | Attribution pipeline |

### 6.2 Reliability

| Métrica | Target | Estrategia |
|---------|--------|------------|
| Disponibilidad del collector | 99.9% | Multi-AZ, health checks |
| Durabilidad de métricas crudas | 100% | Replication + backup |
| Consistencia de agregación | Strong para revenue, eventual para otras | Write-ahead log + replay |
| Recuperación tras fallo | < 1 minuto | Auto-healing + circuit breakers |

### 6.3 Security

| Requisito | Implementación |
|-----------|----------------|
| PII en métricas | Hashing irreversible + K-anonymity + masking |
| Acceso a métricas | RBAC: partner_id isolation + role-based permissions |
| Integridad de métricas | Firmas digitales para métricas de revenue |
| Encriptación en tránsito | TLS 1.3 para todas las comunicaciones |
| Gestión de salts | Secrets manager, rotación automática |

### 6.4 Scalability

| Dimensión | Estrategia |
|-----------|------------|
| Horizontal | Stateless emission, partitioned aggregation by time/partner |
| Multi-partner | Aislamiento por `partner_id` en todas las capas |
| Multi-region | Replicación asíncrona de métricas agregadas |
| Growth | Time-series DB con retención configurable + cold storage |

---

## 7. Technical Specifications

### 7.1 Contratos de Interfaz (Pseudocódigo)

```
// === Metrics Emitter Interface ===

INTERFACE MetricsEmitter:
    METHOD emit_counter(name: STRING, value: INTEGER, tags: OBJECT, timestamp: ISO-8601 OR NULL) → EmitResult
    METHOD emit_gauge(name: STRING, value: FLOAT, tags: OBJECT, timestamp: ISO-8601 OR NULL) → EmitResult
    METHOD emit_histogram(name: STRING, value: FLOAT, tags: OBJECT, timestamp: ISO-8601 OR NULL, sample_rate: FLOAT OR NULL) → EmitResult
    METHOD emit_business_metric(name: STRING, value: FLOAT, currency: STRING, tags: OBJECT, attribution_window_hours: INTEGER, timestamp: ISO-8601 OR NULL) → EmitResult
    METHOD flush(timeout_ms: INTEGER) → FlushResult

// === Query API Interface ===

INTERFACE MetricsQueryAPI:
    METHOD query(query: MetricsQuery, auth: AuthContext) → MetricsQueryResponse
    METHOD list_available_metrics(partner_id: STRING) → ARRAY<MetricDefinition>
    METHOD export(query: MetricsQuery, format: ENUM["csv", "json", "parquet"], auth: AuthContext) → ExportResult

// === Attribution Interface ===

INTERFACE AttributionMapper:
    METHOD attribute_revenue(android_event: AndroidEvent, crm_conversions: ARRAY<ConversionRecord>) → AttributionResult
    METHOD get_attribution_report(partner_id: STRING, time_range: OBJECT, model: ENUM) → AttributionReport

// === Alerting Interface ===

INTERFACE AlertingEngine:
    METHOD register_rule(rule: AlertRule) → RegisterResult
    METHOD evaluate_and_trigger(metrics: AggregatedMetrics) → ARRAY<AlertTriggered>
    METHOD silence_rule(rule_id: STRING, duration_seconds: INTEGER, reason: STRING) → Void

// === Optimization Interface ===

INTERFACE OptimizationEngine:
    METHOD analyze_and_suggest(android_id: STRING, partner_id: STRING, time_window: OBJECT) → ARRAY<Suggestion>
    METHOD apply_suggestion(suggestion_id: STRING, applied_by: STRING) → ApplyResult
    METHOD get_suggestion_history(android_id: STRING) → ARRAY<SuggestionHistory>
```

### 7.2 Configuración de Métricas Base (Ejemplo)

```yaml
# config/metrics/base_v1.yaml
metric_definitions:
  # Métricas de evento (counters)
  - name: "android.event.lead_processed"
    type: counter
    description: "Número de leads procesados por el Android"
    required_tags: ["android_id", "partner_id", "region", "channel"]
    optional_tags: ["niche", "batch_id"]
    
  - name: "android.event.decision_made"
    type: counter
    description: "Número de decisiones emitidas"
    required_tags: ["android_id", "partner_id", "decision_type"]
    
  - name: "android.event.action_executed"
    type: counter
    description: "Número de acciones ejecutadas (envíos, escalaciones, etc.)"
    required_tags: ["android_id", "partner_id", "action_type"]
    
  # Métricas de estado (gauges)
  - name: "android.state.pending_leads"
    type: gauge
    description: "Número de leads en cola pendientes de procesamiento"
    required_tags: ["android_id", "partner_id"]
    
  # Métricas de performance (histograms)
  - name: "android.latency.decision_ms"
    type: histogram
    description: "Tiempo de toma de decisión en milisegundos"
    required_tags: ["android_id", "partner_id"]
    buckets: [10, 50, 100, 250, 500, 1000, 2500, 5000]
    
  # Métricas de negocio (business metrics)
  - name: "android.revenue.attributed"
    type: business_metric
    description: "Revenue atribuido a actividades del Android"
    required_tags: ["android_id", "partner_id", "lead_hash"]
    currency: "USD"  # o configurable por partner
    attribution_window_hours: 72  # default

# Configuración de agregación
aggregation:
  windows: [60, 300, 3600, 86400]  # 1min, 5min, 1h, 24h
  retention:
    raw_metrics_days: 90
    aggregated_metrics_years: 7
  k_anonymity_min_group: 10  # no exponer grupos < 10 leads

# Configuración de atribución
attribution:
  default_model: "last_touch"
  supported_models: ["first_touch", "last_touch", "linear", "time_decay"]
  crm_sync_interval_minutes: 5  # frecuencia de sincronización con CRM

# Configuración de alertas base
alerting:
  default_cooldown_seconds: 300  # 5 minutos entre alertas del mismo tipo
  default_severity_thresholds:
    warning: {response_rate: 0.10, opt_out_rate: 0.05}
    critical: {response_rate: 0.05, opt_out_rate: 0.10}
  notification_channels:
    email: {enabled: true, default_recipients: ["ops@partner.com"]}
    slack: {enabled: false, webhook_url: null}
    webhook: {enabled: false, url: null}
```

### 7.3 Flujo de Emisión de Métricas (Secuencia)

```
[Android ejecuta acción]
       ↓
[MetricsEmitter.emit_*()]
       ↓
[Validación de schema + tags obligatorios]
       ↓
[Anonimización de lead_id → lead_hash]
       ↓
[Enqueue asíncrono a Metrics Collector]
       ↓
[Android continúa sin esperar]
       ↓
[Metrics Collector procesa batch]
       ↓
[Aggregation Engine calcula derivados]
       ↓
[Storage: Time-Series DB]
       ↓
[Query API / Alerting / Optimization leen]
```

---

## 8. Implementation Notes

### 8.1 Stack Reference (n8n como ejemplo)

| Componente | Implementación en n8n | Notas |
|------------|----------------------|-------|
| Metrics Emitter | Function node + HTTP Request a collector | Async, no bloquea flujo |
| Event Schema Validation | Code node + JSON Schema library | Validación en emisión |
| Metrics Collector | External service (Prometheus/InfluxDB) | n8n solo emite, no almacena |
| Aggregation Engine | External (Grafana/ClickHouse) | Procesamiento fuera de n8n |
| Attribution Mapper | HTTP Request + CRM sync | Correlación vía lead_hash |
| Query API | External REST API (FastAPI/Node) | n8n consume, no sirve |
| Alerting Engine | External (Grafana Alerting/PagerDuty) | n8n puede recibir webhooks |
| Privacy Guard | Function node + hashing library | Masking antes de emitir |

### 8.2 Alternative Stacks

| Stack | Ventajas | Consideraciones |
|-------|----------|-----------------|
| **n8n + Prometheus** | Open source, maduro, buen ecosistema | Requiere infra adicional |
| **n8n + Datadog** | SaaS, fácil de usar, alerting integrado | Costo por volumen de métricas |
| **Custom Node.js + ClickHouse** | Máximo control, performance para grandes volúmenes | Requiere más desarrollo |
| **Serverless (Lambda + Timestream)** | Escalado automático, pago por uso | Cold starts, debugging complejo |

### 8.3 Environment Variables

```bash
# Metrics Emitter Config
METRICS_EMITTER_ENABLED=true
METRICS_EMITTER_ASYNC=true
METRICS_EMITTER_BATCH_SIZE=100
METRICS_EMITTER_FLUSH_INTERVAL_SECONDS=30

# Collector Config
METRICS_COLLECTOR_URL=https://metrics-collector.roya.internal
METRICS_COLLECTOR_API_KEY=xxx
METRICS_COLLECTOR_TIMEOUT_MS=5000

# Aggregation Config
AGGREGATION_WINDOWS_SECONDS=60,300,3600,86400
AGGREGATION_RETENTION_RAW_DAYS=90
AGGREGATION_RETENTION_AGGREGATED_YEARS=7
AGGREGATION_K_ANONYMITY_MIN=10

# Attribution Config
ATTRIBUTION_ENABLED=true
ATTRIBUTION_CRM_SYNC_URL=https://crm-sync.roya.internal
ATTRIBUTION_DEFAULT_MODEL=last_touch
ATTRIBUTION_WINDOW_HOURS=72

# Query API Config
QUERY_API_URL=https://metrics-api.roya.internal
QUERY_API_CACHE_TTL_SECONDS=300
QUERY_API_MAX_RESULTS=10000

# Alerting Config
ALERTING_ENABLED=true
ALERTING_EVALUATION_INTERVAL_SECONDS=60
ALERTING_DEFAULT_COOLDOWN_SECONDS=300

# Privacy Config
PRIVACY_HASH_SALT_BY_PARTNER_PATH=/secrets/hash_salts.json
PRIVACY_PII_FIELDS=phone,email,name,id_number,address
PRIVACY_MIN_GROUP_SIZE=10

# Optimization Config
OPTIMIZATION_ENABLED=true
OPTIMIZATION_SUGGESTION_COOLDOWN_HOURS=24
OPTIMIZATION_REQUIRE_HUMAN_APPROVAL=true
```

---

## 9. Testing Strategy

### 9.1 Test Cases

| ID | Descripción | Input | Expected Output |
|----|-------------|-------|-----------------|
| **TC-AM-01** | Emisión de counter válida | `emit_counter("android.event.lead_processed", 1, {android: "sb"})` | Métrica aceptada, lead_hash generado |
| **TC-AM-02** | Emisión con PII raw | `emit_counter(..., tags: {phone: "+521234567890"})` | Rechazada o phone automáticamente masked |
| **TC-AM-03** | Agregación en ventana 5min | 100 eventos en 5 minutos | Counter agregado = 100, tasa calculada |
| **TC-AM-04** | Atribución last_touch | Android event + CRM conversion en ventana | Revenue atribuido al Android correcto |
| **TC-AM-05** | Query con filtros | `group_by: ["android_id"], filter: {partner_id: "acme"}` | Resultados agrupados por Android, solo partner acme |
| **TC-AM-06** | Alerta por threshold bajo | response_rate < 0.10 por 10 minutos | Alerta disparada, notificación enviada |
| **TC-AM-07** | Sugerencia de optimización | response_rate baja + correlación histórica | Sugerencia: "aumentar follow-up frequency" |
| **TC-AM-08** | K-anonymity enforcement | Grupo con 5 leads (min_k=10) | Grupo excluido de resultados de query |
| **TC-AM-09** | Derecho al olvido | Solicitud de eliminar métricas de lead_hash X | Métricas asociadas eliminadas/anonimizadas |
| **TC-AM-10** | Atribución multi-toque | Mismo lead, 3 eventos Android, 1 conversión | Revenue distribuido según modelo configurado |

### 9.2 Test Execution

```bash
# Unit tests (componentes individuales)
npm test -- unit/metrics-emitter/
npm test -- unit/privacy-guard/
npm test -- unit/attribution-mapper/

# Integration tests (flujos end-to-end)
npm test -- integration/emit-to-aggregate/
npm test -- integration/attribution-end-to-end/
npm test -- integration/query-api-auth/

# Privacy tests (compliance)
npm test -- privacy/pii-detection/
npm test -- privacy/k-anonymity/
npm test -- privacy/right-to-be-forgotten/

# Performance tests (escalabilidad)
npm test -- perf/emit-throughput-10k-per-sec/
npm test -- perf/query-latency-large-dataset/
npm test -- perf/aggregation-window-calculation/

# Attribution tests (precisión)
npm test -- attribution/first-touch-model/
npm test -- attribution/last-touch-model/
npm test -- attribution/linear-model/
npm test -- attribution/time-decay-model/
```

---

## 10. Success Metrics (Técnicos y de Negocio)

| Métrica | Fórmula | Target | Frecuencia |
|---------|---------|--------|------------|
| Metrics emission success rate | successful_emits / total_emits | ≥ 99.9% | Diario |
| Attribution accuracy | attributed_revenue / actual_revenue | ≥ 95% | Semanal |
| Query latency P95 | percentile_95(query_time) | ≤ 2s | Diario |
| Alert precision | true_alerts / total_alerts | ≥ 90% | Semanal |
| Optimization adoption rate | applied_suggestions / total_suggestions | ≥ 30% | Mensual |
| Privacy compliance | pii_leaks / total_metrics | 0% | Auditoría mensual |
| Partner isolation | cross_partner_data_leaks | 0 | Continuo |
| ROI visibility | partners_with_attribution_dashboard / total_partners | ≥ 80% | Trimestral |

---

## 11. Deployment Checklist

### Pre-Deployment
- [ ] Configuración de métricas base validada (`config/metrics/base_v1.yaml`)
- [ ] Privacy Guard con hashing de lead_id y salts por partner
- [ ] Metrics Collector desplegado y saludable (Prometheus/InfluxDB/etc.)
- [ ] Attribution Mapper con sincronización CRM configurada
- [ ] Query API con autenticación y aislamiento por partner_id
- [ ] Alerting Engine con reglas base y canales de notificación
- [ ] Optimization Engine con reglas heurísticas iniciales
- [ ] Tests de privacidad pasando (0 PII en métricas de prueba)
- [ ] Tests de atribución pasando (revenue correctamente mapeado)
- [ ] Plan de rollback documentado para fallos de emisión/agregación

### Post-Deployment
- [ ] Monitorizar metrics emission success rate por primeras 1000 emisiones
- [ ] Validar que lead_hash es consistente para mismo lead_id + partner
- [ ] Confirmar que queries respetan aislamiento por partner_id
- [ ] Verificar que alertas se disparan en condiciones de prueba
- [ ] Revisar logs para detectar cualquier PII no masked
- [ ] Documentar cualquier desviación de targets de performance/privacidad

---

## 12. Version History

| Versión | Fecha | Cambios | Autor |
|---------|-------|---------|-------|
| 1.0 | 2026-02-20 | Especificación inicial de Analytics & Metrics Framework | O.A. Perez Garrido |

---

## 13. Appendices

### Appendix A: Métricas Base por Android

| Android | Métricas Obligatorias | Métricas de Negocio | Métricas de Performance |
|---------|----------------------|---------------------|------------------------|
| **Sleeping Beauty** | `lead_processed`, `kiss_sent`, `reply_received`, `qualification_result`, `appointment_scheduled` | `revenue_attributed`, `conversion_rate`, `roi_per_batch` | `decision_latency_ms`, `followup_response_time` |
| **Speed To Lead** | `lead_ingested`, `ack_sent`, `reply_received`, `qualification_result`, `lead_routed` | `revenue_attributed`, `conversion_rate_vs_manual`, `speed_advantage_minutes` | `ack_latency_ms`, `routing_accuracy` |
| **Out Of Hours** | `ooo_contact_detected`, `ack_sent`, `nurturing_sent`, `lead_queued`, `handoff_completed` | `revenue_attributed`, `retention_rate_vs_no_response`, `handoff_success_rate` | `ack_latency_ms`, `queue_wait_time` |
| **Document Collection** | `doc_request_triggered`, `doc_sent`, `doc_validated`, `milestone_payment_triggered` | `revenue_per_document`, `completion_rate`, `time_to_milestone` | `validation_latency_ms`, `followup_effectiveness` |

### Appendix B: Tags Obligatorios por Capa

```yaml
# Tags que DEBEN incluirse en toda métrica emitida
required_tags:
  # Identificación del Android
  - android_id: STRING  # e.g., "sleeping_beauty"
  - android_version: STRING  # e.g., "v1.0"
  
  # Identificación del socio/instancia
  - partner_id: STRING  # ID único del socio
  - partner_tier: ENUM ["starter", "growth", "enterprise"]  # para segmentación
  
  # Contexto geográfico/regulatorio
  - region: STRING  # e.g., "MX", "US", "EU"
  
  # Canal de interacción
  - channel: ENUM ["sms", "whatsapp", "email"]

# Tags opcionales recomendados
recommended_tags:
  - niche: STRING  # e.g., "clinicas", "viajes", "solar"
  - campaign_id: STRING  # para agrupar por campaña
  - batch_id: STRING  # para tracking de procesamiento por lotes
  - user_segment: ENUM ["new", "returning", "vip"]  # si aplica
```

### Appendix C: Modelos de Atribución Detallados

| Modelo | Fórmula | Cuándo usar | Pros | Contras |
|--------|---------|-------------|------|---------|
| **First Touch** | Revenue 100% al primer evento del Android | Cuando el primer contacto es crítico (ej: reactivación) | Simple, fácil de explicar | Ignora interacciones posteriores |
| **Last Touch** | Revenue 100% al último evento antes de conversión | Cuando el cierre es lo que importa (ej: speed to lead) | Alineado con venta final | Subestima nurturing temprano |
| **Linear** | Revenue dividido equitativamente entre N eventos | Cuando todas las interacciones aportan valor similar | Balanceado, justo | Puede diluir impacto de eventos clave |
| **Time Decay** | Weight = exp(-hours_since_event / decay_constant) | Cuando cercanía temporal a conversión importa más | Realista para ciclos de venta cortos | Requiere calibrar decay_constant |

**Ejemplo de configuración:**
```yaml
attribution_models:
  sleeping_beauty:
    default: "last_touch"  # la calificación final es lo que cuenta
    alternative: "linear"  # para análisis de nurturing
  
  speed_to_lead:
    default: "first_touch"  # la velocidad inicial es el valor
    alternative: "time_decay"  # para ciclos de venta largos
  
  document_collection:
    default: "linear"  # cada follow-up aporta al cierre
    alternative: "last_touch"  # el documento enviado es el trigger
```

### Appendix D: Alertas Base Recomendadas

```yaml
# config/alerts/base_v1.yaml
alert_rules:
  # Alerta crítica: tasa de respuesta muy baja
  - rule_id: "response_rate_critical"
    name: "Tasa de respuesta críticamente baja"
    severity: critical
    condition:
      metric_name: "android.metrics.response_rate"
      operator: lt
      threshold: 0.05  # <5%
      window_seconds: 3600  # evaluar última hora
      group_by: ["android_id", "partner_id"]
    actions:
      - type: email
        recipients: ["ops-alerts@partner.com", "support@roya.dev"]
        message_template: "ALERTA CRÍTICA: {partner_id}/{android_id} tiene response_rate de {value*100}% (threshold: 5%)"
      - type: slack
        webhook_url: "https://hooks.slack.com/xxx"
    cooldown_seconds: 1800  # 30 minutos entre alertas
  
  # Alerta warning: tasa de opt-out alta
  - rule_id: "opt_out_rate_warning"
    name: "Tasa de opt-out elevada"
    severity: warning
    condition:
      metric_name: "android.metrics.opt_out_rate"
      operator: gt
      threshold: 0.08  # >8%
      window_seconds: 3600
      group_by: ["android_id", "partner_id", "channel"]
    actions:
      - type: email
        recipients: ["compliance@partner.com"]
        message_template: "ALERTA: {partner_id}/{android_id}/{channel} tiene opt_out_rate de {value*100}% (threshold: 8%)"
    cooldown_seconds: 3600
  
  # Alerta info: volumen inusualmente alto
  - rule_id: "volume_anomaly"
    name: "Volumen de procesamiento anómalo"
    severity: info
    condition:
      metric_name: "android.event.lead_processed"
      operator: anomaly  # detección estadística
      window_seconds: 300  # evaluar cada 5 minutos
      group_by: ["android_id", "partner_id"]
    actions:
      - type: webhook
        webhook_url: "https://monitoring.partner.com/alerts"
    cooldown_seconds: 900
```

### Appendix E: Guía de Integración para Nuevos Androids

```
// === Pasos para instrumentar un nuevo Android con métricas ===

1. IMPORTAR MetricsEmitter:
   IMPORT MetricsEmitter FROM "@roya/analytics-metrics"

2. CONFIGURAR tags base en inicialización:
   const emitter = new MetricsEmitter({
     android_id: "my_new_android",
     android_version: "v1.0",
     partner_id: CONFIG.partner_id,
     region: CONFIG.region
   })

3. EMITIR métricas en puntos clave del flujo:
   // Al procesar un lead
   emitter.emit_counter("android.event.lead_processed", 1, {
     channel: lead.channel,
     niche: lead.niche
   })
   
   // Al tomar una decisión
   emitter.emit_counter("android.event.decision_made", 1, {
     decision_type: decision.primary_action,
     risk_level: decision.risk.level
   })
   
   // Al ejecutar una acción
   emitter.emit_counter("android.event.action_executed", 1, {
     action_type: final_action.type,
     channel: final_action.channel
   })
   
   // Para performance
   emitter.emit_histogram("android.latency.decision_ms", latency_ms, {
     // tags adicionales si aplica
   })
   
   // Para revenue (si aplica)
   if (revenue_generated > 0) {
     emitter.emit_business_metric("android.revenue.attributed", revenue_generated, "USD", {
       lead_hash: anonymize_lead_id(lead.id),  // NUNCA lead_id raw
       conversion_type: "sale"
     }, attribution_window_hours: 72)
   }

4. VALIDAR que todas las métricas incluyen tags obligatorios:
   // El emitter valida automáticamente, pero se puede pre-validar:
   if (!emitter.validate_tags({channel: lead.channel})) {
     logger.error("Missing required tags for metric emission")
   }

5. TESTEAR emisión en entorno de staging:
   // Verificar que métricas llegan al collector
   // Verificar que lead_hash es consistente
   // Verificar que no hay PII en logs/métricas

6. DOCUMENTAR métricas emitidas en README del Android:
   // Para que analistas sepan qué pueden consultar
```

### Appendix F: Configuración Multi-Partner (Ejemplo)

```yaml
# config/partners/acme_legal_metrics_v1.yaml
partner_id: acme_legal
partner_tier: enterprise

# Configuración específica de métricas
metrics:
  # Habilitar métricas de revenue para este partner
  revenue_tracking_enabled: true
  currency: "USD"
  attribution_model: "linear"  # prefieren ver contribución de todo el journey
  
  # Retención más larga por requisitos legales
  retention:
    raw_metrics_days: 365  # 1 año vs default 90 días
    aggregated_metrics_years: 10  # 10 años vs default 7
  
  # Alertas personalizadas
  alerting:
    custom_thresholds:
      response_rate: {warning: 0.15, critical: 0.08}  # más estricto que default
      opt_out_rate: {warning: 0.03, critical: 0.06}  # nicho legal, opt-outs más sensibles
    additional_recipients:
      critical: ["legal-compliance@acme.com", "ceo@acme.com"]
      warning: ["ops@acme.com"]
  
  # Privacidad reforzada
  privacy:
    k_anonymity_min: 25  # grupos mínimos más grandes por sensibilidad legal
    pii_fields_extended: ["case_number", "client_reference"]  # campos adicionales a proteger
    audit_all_queries: true  # loguear todas las consultas a métricas
  
  # Optimización habilitada con aprobación humana
  optimization:
    enabled: true
    require_human_approval: true
    approval_workflow: "legal_review_required"  # sugerencias pasan por revisión legal antes de aplicar

# Configuración de Query API para este partner
query_api:
  max_results_per_query: 50000  # más que default para análisis profundos
  export_enabled: true  # permitir exportación a sus sistemas de BI
  custom_dashboards: ["legal_compliance_dashboard", "revenue_attribution_by_case_type"]
```

---

## ✅ Aprobaciones

| Rol | Nombre | Fecha | Firma |
|-----|--------|-------|-------|
| Product Owner | | | |
| Tech Lead | | | |
| Data/Analytics Lead | | | |
| Compliance Officer | | | |
| Security Lead | | | |

---

**Documento generado:** 20 de febrero de 2026  
**Próxima revisión:** 20 de marzo de 2026  
**Status:** Ready for Implementation

---

> **Nota final para el equipo:**  
> Este SPEC define el framework de analytics que cierra el ciclo de ROYA: sin medición, no hay optimización; sin atribución, no hay ROI visible.  
> Cada Android DEBE emitir métricas — no es opcional.  
> La clave del éxito no es la complejidad del dashboard, es la **consistencia de emisión** + **privacidad por diseño** + **atribución accionable**.  
> Si algo no encaja en este spec, pregunta: "¿Esto ayuda a medir, atribuir u optimizar de forma compliant y escalable?" antes de implementarlo.  
