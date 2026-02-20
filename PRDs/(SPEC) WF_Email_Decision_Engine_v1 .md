# Technical Product Requirements Document 
## WF_Email_Decision_Engine_v1
**Versión:** 1.0  
**Fecha:** 18 de febrero de 2026  
**Estado:** Especificación para implementación  
**Dominio:** Email Decision Engine (Agnóstico)  
**Stack referencia:** n8n + LLM Provider + Vector Store

---

## 1. Overview Técnico

### 1.1 Propósito del sistema
Motor de decisiones estructuradas para procesamiento automatizado de emails entrantes. Convierte inputs no estructurados en decisiones auditables y ejecutables.

### 1.2 Arquitectura de alto nivel
```
Email Input → Normalización → Decision Engine (LLM) → Routing → Response Contract → Consumer Adapter
```

### 1.3 Principios técnicos fundamentales
- **Determinismo:** Mismos inputs = mismos outputs
- **Contrato-first:** Todos los componentes comunican mediante schemas definidos
- **Separación de concerns:** Decisión ≠ Ejecución ≠ Contexto
- **Auditable:** Cada decisión deja traza completa
- **Escalable:** Diseño stateless, horizontalmente escalable

---

## 2. Problem Statement (Técnico)

### 2.1 Problema actual
Los sistemas de procesamiento de emails actuales sufren de:
- **Respuestas no estructuradas:** Outputs textuales libres imposibles de orquestar
- **Falta de trazabilidad:** Decisiones "mágicas" sin explicación auditables
- **Acoplamiento tight:** Lógica de negocio mezclada con ejecución
- **Fragilidad ante cambios:** Modificar un prompt rompe todo el sistema

### 2.2 Impacto técnico
- Imposibilidad de automatización determinista
- Alta dependencia de supervisión humana
- Dificultad para escalar a múltiples dominios
- Riesgo operativo por respuestas inconsistentes

---

## 3. Technical Objectives

### 3.1 Objetivos SMART

| ID | Objetivo | Métrica | Target |
|----|----------|---------|--------|
| **TO-1** | Procesar emails con decisión estructurada | Decision completeness rate | ≥ 95% |
| **TO-2** | Emitir outputs conforme a contrato | Schema validation success | 100% |
| **TO-3** | Mantener latencia aceptable | P95 latency | < 5 segundos |
| **TO-4** | Soportar múltiples dominios sin cambios de código | Domain instantiation time | < 1 hora |
| **TO-5** | Auditar todas las decisiones | Traceability coverage | 100% |

### 3.2 Requerimientos de calidad (NFRs)

| NFR | Descripción | Target |
|-----|-------------|--------|
| **NFR-1** | Disponibilidad | 99.5% uptime |
| **NFR-2** | Latencia P95 | < 5 segundos |
| **NFR-3** | Throughput | 100 emails/minuto |
| **NFR-4** | Data retention | 90 días de logs |
| **NFR-5** | Error rate | < 1% fallos |

---

## 4. Scope Técnico

### 4.1 In Scope

| Componente | Descripción | Responsabilidad |
|------------|-------------|-----------------|
| **Email Ingestion** | Captura emails desde proveedor (Gmail, Outlook, etc.) | Trigger/Adapter |
| **Input Normalization** | Convierte email crudo a Input Contract v1 | Transformación |
| **Decision Engine** | LLM clasifica intent, riesgo, acción | LLM Skill |
| **Context Grounding** | Vector store proporciona conocimiento read-only | RAG Tool |
| **Decision Routing** | IF nodes enrutan según primary_action | Orquestación |
| **Response Wrapper** | Envuelve en Response Contract v1 | Transformación |
| **Consumer Adapter** | Ejecuta acción final (email, Slack, CRM) | Integración |
| **Audit Logging** | Registra toda decisión con trace_id | Observabilidad |

### 4.2 Out of Scope

| Componente | Razón de exclusión |
|------------|-------------------|
| UI/Dashboard | Capa de presentación (separada) |
| Multi-channel (WhatsApp, SMS) | Solo email en v1 |
| Machine Learning training | Solo inferencia |
| Real-time analytics | Batch reporting |
| Multi-tenancy | Single tenant en v1 |
| Advanced NLP (sentiment, entities) | Solo clasificación básica |

---

## 5. Functional Requirements

### 5.1 FR-1: Email Ingestion & Normalization

**ID:** FR-1  
**Prioridad:** HIGH  
**Descripción:** El sistema debe capturar emails entrantes y normalizarlos a Input Contract v1.

**User Story:**  
Como sistema, quiero recibir emails de cualquier proveedor y convertirlos a un formato estandarizado para procesamiento consistente.

**Acceptance Criteria:**
- [ ] Captura emails desde Gmail API, Outlook API, o webhook genérico
- [ ] Extrae: `from`, `subject`, `body` (texto plano), `timestamp`, `messageId`
- [ ] Normaliza body: elimina HTML, normaliza encoding
- [ ] Genera `input_id` único basado en `messageId`
- [ ] Asigna `channel: "email"` fijo
- [ ] Asigna `context_domain` parametrizable (default: "agnostic")
- [ ] Output conforme a Input Contract v1

**Input Contract v1:**
```json
{
  "input_id": "string (UUID or messageId-based)",
  "channel": "email",
  "message": "string (normalized text)",
  "context_domain": "string (e.g., 'clinic', 'travel', 'agnostic')",
  "metadata": {
    "from": "string",
    "subject": "string",
    "timestamp": "ISO-8601",
    "original_message_id": "string"
  }
}
```

---

### 5.2 FR-2: Context Retrieval & Grounding

**ID:** FR-2  
**Prioridad:** HIGH  
**Descripción:** El sistema debe recuperar contexto relevante antes de tomar decisiones.

**User Story:**  
Como motor de decisiones, necesito acceso a conocimiento estructurado (SOPs, políticas, FAQs) para tomar decisiones informadas y evitar alucinaciones.

**Acceptance Criteria:**
- [ ] Vector store indexado con documentos de dominio
- [ ] Embeddings generados con modelo consistente (text-embedding-3-small o equivalente)
- [ ] Chunking con tamaño configurable (default: 900 chars, overlap: 50)
- [ ] Búsqueda por similitud semántica (top_k: 4, threshold: 0.75)
- [ ] Namespace por dominio (e.g., `clinic_v1`, `travel_v1`)
- [ ] Contexto expuesto como tool MCP al Decision Engine
- [ ] Read-only: el vector store nunca genera texto final

**Technical Implementation:**
```yaml
vector_store:
  type: pinecone|weaviate|chroma
  index_name: "email_decision_engine"
  namespace: "{{ context_domain }}_v1"
  
embeddings:
  model: "text-embedding-3-small"
  dimensions: 1536
  
chunking:
  size: 900
  overlap: 50
  strategy: "recursive_character"
  
retrieval:
  top_k: 4
  score_threshold: 0.75
  max_tokens: 2000
```

---

### 5.3 FR-3: Intent Classification

**ID:** FR-3  
**Prioridad:** HIGH  
**Descripción:** El sistema debe clasificar la intención principal del mensaje en categorías predefinidas.

**User Story:**  
Como sistema de orquestación, necesito saber qué tipo de solicitud es para enrutarla correctamente sin intervención humana.

**Acceptance Criteria:**
- [ ] Clasifica exactamente UNA intención primaria por mensaje
- [ ] Categorías agnósticas: `ASK_INFO`, `REQUEST_ACTION`, `SCHEDULE`, `COMPLAINT`, `SUPPORT`, `UNKNOWN`
- [ ] Nivel de confianza: `low`, `medium`, `high`
- [ ] Justificación breve en `notes` (máx. 100 chars)
- [ ] Output conforme a Decision Contract v1

**Decision Contract v1 (extracto):**
```json
{
  "intent": {
    "primary": "ASK_INFO|REQUEST_ACTION|SCHEDULE|COMPLAINT|SUPPORT|UNKNOWN",
    "confidence": "low|medium|high",
    "notes": "string (≤100 chars)"
  }
}
```

---

### 5.4 FR-4: Risk & Urgency Detection

**ID:** FR-4  
**Prioridad:** HIGH  
**Descripción:** El sistema debe evaluar nivel de riesgo y detectar señales de urgencia o peligro.

**User Story:**  
Como sistema de seguridad, necesito identificar casos críticos que requieren escalación humana inmediata para proteger al negocio y al usuario.

**Acceptance Criteria:**
- [ ] Niveles de riesgo: `NONE`, `LOW`, `MEDIUM`, `HIGH`, `CRITICAL`
- [ ] Detecta señales específicas según dominio (ej: síntomas médicos, lenguaje legal)
- [ ] Lista explícita de `signals` detectados
- [ ] User states: `CLEAR_INTENT`, `UNCLEAR_INTENT`, `EMOTIONAL`, `RISK_FLAG`, `MEDICAL_INQUIRY`, etc.
- [ ] Riesgo ≥ HIGH bloquea automatización
- [ ] Output conforme a Decision Contract v1

**Risk Schema:**
```json
{
  "risk": {
    "level": "NONE|LOW|MEDIUM|HIGH|CRITICAL",
    "signals": ["string"]
  },
  "user_states": [
    "CLEAR_INTENT|UNCLEAR_INTENT|EMOTIONAL|RISK_FLAG|MEDICAL_INQUIRY|..."
  ]
}
```

---

### 5.5 FR-5: Information Completeness Check

**ID:** FR-5  
**Prioridad:** MEDIUM  
**Descripción:** El sistema debe determinar si hay información suficiente para ejecutar una acción.

**User Story:**  
Como sistema de calidad, quiero detectar datos faltantes y solicitarlos explícitamente en lugar de adivinar o fallar silenciosamente.

**Acceptance Criteria:**
- [ ] Identifica campos requeridos según tipo de acción
- [ ] Lista EXACTAMENTE qué falta en `missing_information[]`
- [ ] No inventa valores por defecto
- [ ] Si falta info crítica → `primary_action: REQUEST_MORE_INFO`
- [ ] Output conforme a Decision Contract v1

**Missing Information Schema:**
```json
{
  "missing_information": [
    "especialidad médica",
    "fecha preferida",
    "nombre completo",
    "contacto"
  ]
}
```

---

### 5.6 FR-6: Decision Selection & Justification

**ID:** FR-6  
**Prioridad:** HIGH  
**Descripción:** El sistema debe seleccionar exactamente UNA acción primaria basada en reglas predefinidas.

**User Story:**  
Como motor de decisiones, necesito emitir una acción clara y justificada que el orquestador pueda ejecutar sin ambigüedad.

**Acceptance Criteria:**
- [ ] Selecciona exactamente UNA `primary_action`
- [ ] Acciones permitidas: `REQUEST_MORE_INFO`, `PROVIDE_INFO`, `LOG_AND_ROUTE`, `SCHEDULE_REQUEST`, `ESCALATE_HUMAN`, `REJECT_REQUEST`
- [ ] Justificación obligatoria en `reason` (máx. 200 chars)
- [ ] Constraints aplicados listados explícitamente
- [ ] Escalación requerida si `risk ≥ HIGH` o constraints médicos/legal
- [ ] Output conforme a Decision Contract v1

**Decision Schema:**
```json
{
  "decision": {
    "primary_action": "REQUEST_MORE_INFO|PROVIDE_INFO|LOG_AND_ROUTE|SCHEDULE_REQUEST|ESCALATE_HUMAN|REJECT_REQUEST",
    "secondary_actions": ["string"],
    "reason": "string (≤200 chars)"
  },
  "constraints_applied": ["C01", "C03", "C08"],
  "escalation": {
    "required": true|false,
    "reason": "string"
  }
}
```

---

### 5.7 FR-7: Action Orchestration

**ID:** FR-7  
**Prioridad:** HIGH  
**Descripción:** El sistema debe enrutar la decisión a través de IF nodes y construir el Response Contract.

**User Story:**  
Como orquestador, necesito evaluar la decisión estructurada y ejecutar la acción correspondiente de forma determinista.

**Acceptance Criteria:**
- [ ] IF nodes evalúan `decision.primary_action`
- [ ] Cada acción tiene su propio Set node con payload específico
- [ ] IF adicional evalúa `risk.level` para escalaciones críticas
- [ ] Todos los paths convergen en Set_Final_Action_Wrapper
- [ ] Output conforme a Response Contract v1

**Routing Logic:**
```javascript
// IF_Escalate
if (decision.primary_action === "ESCALATE_HUMAN") {
  if (risk.level === "CRITICAL") {
    // Set_Critical_Escalation
  } else {
    // Set_Normal_Escalation
  }
}

// IF_Provide_Info
if (decision.primary_action === "PROVIDE_INFO") {
  // Set_Provide_Info
}

// IF_Request_More_Info
if (decision.primary_action === "REQUEST_MORE_INFO") {
  // Set_Request_More_Info
}

// IF_Reject_Request
if (decision.primary_action === "REJECT_REQUEST") {
  // Set_Rejection
}
```

---

### 5.8 FR-8: Response Generation & Delivery

**ID:** FR-8  
**Prioridad:** MEDIUM  
**Descripción:** El sistema debe generar respuesta estructurada y entregarla al consumidor final.

**User Story:**  
Como sistema de salida, necesito emitir un contrato estable que consumidores externos puedan interpretar sin ambigüedad.

**Acceptance Criteria:**
- [ ] Wrapper construye Response Contract v1 completo
- [ ] Campos obligatorios: `response_id`, `timestamp`, `channel`, `final_action`, `meta`
- [ ] `final_action.type` mapea 1:1 con `decision.primary_action`
- [ ] `final_action.payload` específico por tipo de acción
- [ ] `meta.trace_id` correlaciona con `input_id`
- [ ] Entrega vía Consumer Adapter configurado (email, Slack, webhook)
- [ ] HTTP 200 + JSON en modo demo

**Response Contract v1:**
```json
{
  "response_id": "uuid",
  "timestamp": "ISO-8601",
  "channel": "email|chat|webhook",
  "final_action": {
    "type": "PROVIDE_INFO|REQUEST_MORE_INFO|ESCALATE_HUMAN|REJECT_REQUEST|SCHEDULE_REQUEST",
    "payload": {
      // Campos específicos según type
      "message": "string",
      "missing_information": ["string"],
      "level": "LOW|MEDIUM|HIGH|CRITICAL",
      "reason": "string",
      "signals": ["string"]
    }
  },
  "meta": {
    "version": "v1",
    "trace_id": "string"
  }
}
```

---

### 5.9 FR-9: Logging & Audit Trail

**ID:** FR-9  
**Prioridad:** HIGH  
**Descripción:** El sistema debe mantener traza completa de inputs, decisiones y acciones.

**User Story:**  
Como auditor, necesito revisar cualquier decisión pasada para mejorar el sistema y cumplir con compliance.

**Acceptance Criteria:**
- [ ] Log completo de input original
- [ ] Log de Decision Contract v1 emitido
- [ ] Log de Response Contract v1 entregado
- [ ] Timestamps ISO-8601 en todos los eventos
- [ ] Correlación mediante `trace_id`
- [ ] Retención mínima: 90 días
- [ ] Exportable para análisis

**Audit Schema:**
```json
{
  "audit_entry": {
    "trace_id": "uuid",
    "timestamp_ingest": "ISO-8601",
    "timestamp_decision": "ISO-8601",
    "timestamp_response": "ISO-8601",
    "input_snapshot": { /* Input Contract v1 */ },
    "decision_snapshot": { /* Decision Contract v1 */ },
    "response_snapshot": { /* Response Contract v1 */ },
    "execution_metadata": {
      "workflow_id": "string",
      "execution_id": "string",
      "llm_model": "string",
      "vector_store_namespace": "string"
    }
  }
}
```

---

## 6. Non-Functional Requirements

### 6.1 Performance

| Métrica | Target | Medición |
|---------|--------|----------|
| Latencia P50 | < 2 segundos | Desde email hasta response |
| Latencia P95 | < 5 segundos | Percentil 95 |
| Throughput | 100 emails/minuto | Sustained load |
| Error rate | < 1% | Fallos / total |

### 6.2 Reliability

| Métrica | Target | Estrategia |
|---------|--------|------------|
| Uptime | 99.5% | Redundancia, monitoring |
| Data durability | 100% | Persistencia en logs |
| Recovery time | < 5 min | Auto-healing, alerts |

### 6.3 Security

| Requisito | Implementación |
|-----------|----------------|
| Data encryption | TLS 1.2+, encryption at rest |
| Access control | API keys, OAuth2 |
| PII handling | Masking en logs, retention policy |
| Compliance | GDPR-ready, audit trails |

### 6.4 Scalability

| Dimensión | Estrategia |
|-----------|------------|
| Horizontal | Stateless design, load balancing |
| Vertical | Resource scaling (CPU, memory) |
| Multi-domain | Namespace isolation en vector store |
| Multi-tenant | Future extension (v2) |

---

## 7. Technical Specifications

### 7.1 Input/Output Contracts

#### Input Contract v1
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Input Contract v1",
  "type": "object",
  "required": ["input_id", "channel", "message", "context_domain"],
  "properties": {
    "input_id": {"type": "string"},
    "channel": {"type": "string", "enum": ["email"]},
    "message": {"type": "string"},
    "context_domain": {"type": "string"},
    "metadata": {
      "type": "object",
      "properties": {
        "from": {"type": "string"},
        "subject": {"type": "string"},
        "timestamp": {"type": "string", "format": "date-time"},
        "original_message_id": {"type": "string"}
      }
    }
  }
}
```

#### Decision Contract v1
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Decision Contract v1",
  "type": "object",
  "required": [
    "input_id", "channel", "comprehension", "relevance",
    "intent", "user_states", "risk", "decision",
    "constraints_applied", "escalation"
  ],
  "properties": {
    "input_id": {"type": "string"},
    "channel": {"type": "string"},
    "comprehension": {
      "type": "object",
      "required": ["is_understandable", "notes"],
      "properties": {
        "is_understandable": {"type": "boolean"},
        "notes": {"type": "string"}
      }
    },
    "relevance": {
      "type": "object",
      "required": ["is_relevant", "notes"],
      "properties": {
        "is_relevant": {"type": "boolean"},
        "notes": {"type": "string"}
      }
    },
    "intent": {
      "type": "object",
      "required": ["primary", "confidence"],
      "properties": {
        "primary": {
          "type": "string",
          "enum": ["ASK_INFO", "REQUEST_ACTION", "SCHEDULE", "COMPLAINT", "SUPPORT", "UNKNOWN"]
        },
        "confidence": {
          "type": "string",
          "enum": ["low", "medium", "high"]
        }
      }
    },
    "user_states": {
      "type": "array",
      "items": {"type": "string"}
    },
    "risk": {
      "type": "object",
      "required": ["level", "signals"],
      "properties": {
        "level": {
          "type": "string",
          "enum": ["NONE", "LOW", "MEDIUM", "HIGH", "CRITICAL"]
        },
        "signals": {"type": "array", "items": {"type": "string"}}
      }
    },
    "missing_information": {"type": "array", "items": {"type": "string"}},
    "decision": {
      "type": "object",
      "required": ["primary_action", "reason"],
      "properties": {
        "primary_action": {
          "type": "string",
          "enum": [
            "REQUEST_MORE_INFO", "PROVIDE_INFO", "LOG_AND_ROUTE",
            "SCHEDULE_REQUEST", "ESCALATE_HUMAN", "REJECT_REQUEST"
          ]
        },
        "secondary_actions": {"type": "array", "items": {"type": "string"}},
        "reason": {"type": "string"}
      }
    },
    "constraints_applied": {
      "type": "array",
      "items": {"type": "string"},
      "minItems": 1
    },
    "escalation": {
      "type": "object",
      "required": ["required"],
      "properties": {
        "required": {"type": "boolean"},
        "reason": {"type": "string"}
      }
    }
  }
}
```

#### Response Contract v1
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Response Contract v1",
  "type": "object",
  "required": ["response_id", "timestamp", "channel", "final_action", "meta"],
  "properties": {
    "response_id": {"type": "string"},
    "timestamp": {"type": "string", "format": "date-time"},
    "channel": {
      "type": "string",
      "enum": ["email", "chat", "webhook"]
    },
    "final_action": {
      "type": "object",
      "required": ["type", "payload"],
      "properties": {
        "type": {
          "type": "string",
          "enum": [
            "PROVIDE_INFO", "REQUEST_MORE_INFO", "ESCALATE_HUMAN",
            "REJECT_REQUEST", "SCHEDULE_REQUEST"
          ]
        },
        "payload": {"type": "object"}
      }
    },
    "meta": {
      "type": "object",
      "required": ["version", "trace_id"],
      "properties": {
        "version": {"type": "string", "enum": ["v1"]},
        "trace_id": {"type": "string"}
      }
    }
  }
}
```

---

### 7.2 Error Handling

| Error Type | Código | Manejo |
|------------|--------|--------|
| Invalid input schema | 400 | Reject, log error |
| LLM timeout | 504 | Retry (max 2), then escalate |
| Vector store unavailable | 503 | Fallback to base context only |
| Schema validation failed | 422 | Reject, detailed error |
| Consumer adapter failure | 500 | Retry, alert, dead letter queue |

**Dead Letter Queue:**
```json
{
  "dlq_entry": {
    "original_input": { /* Input Contract v1 */ },
    "error_details": {
      "timestamp": "ISO-8601",
      "error_code": "string",
      "error_message": "string",
      "retry_count": 0
    },
    "routing_key": "failed.email.processing"
  }
}
```

---

### 7.3 Monitoring & Observability

| Métrica | Tipo | Frecuencia |
|---------|------|------------|
| Emails procesados | Counter | Real-time |
| Decision distribution | Histogram | Real-time |
| Latencia por etapa | Histogram | Real-time |
| Error rate | Gauge | Real-time |
| Vector store hits/misses | Counter | Real-time |

**Health Check Endpoint:**
```
GET /health
Response: 200 OK
{
  "status": "healthy",
  "components": {
    "llm_provider": "ok",
    "vector_store": "ok",
    "email_provider": "ok",
    "consumer_adapters": "ok"
  },
  "uptime": "2d 4h 32m",
  "version": "v1.0"
}
```

---

## 8. Implementation Notes

### 8.1 Stack Reference (n8n)

| Componente | Nodo n8n | Configuración |
|------------|----------|---------------|
| Email Trigger | Gmail Trigger | OAuth2, polling |
| Normalization | Code (JavaScript) | Extract + normalize |
| Decision Engine | AI Agent (Gemini) | Temperature: 0, JSON output |
| Vector Store | Pinecone | Namespace por dominio |
| Embeddings | OpenAI Embeddings | text-embedding-3-small |
| Routing | IF nodes | Evaluate primary_action |
| Response Wrapper | Set (JSON) | Build Response Contract |
| Consumer | Gmail/Slack/Webhook | Configurable per deployment |
| Logging | Write to DB/File | Structured JSON |

### 8.2 Alternative Stacks

| Stack | Orquestador | LLM | Vector Store | Notas |
|-------|-------------|-----|--------------|-------|
| **n8n** | n8n | Gemini/OpenAI | Pinecone | Stack actual |
| **Zapier** | Zapier | OpenAI | Weaviate | Menos flexible |
| **Make** | Make.com | Anthropic | Chroma | Similar a n8n |
| **Custom** | Node.js/Python | Any | Any | Máxima flexibilidad |

### 8.3 Environment Variables

```bash
# Email Provider
EMAIL_PROVIDER=gmail|outlook|sendgrid
EMAIL_OAUTH_CLIENT_ID=xxx
EMAIL_OAUTH_CLIENT_SECRET=xxx

# LLM Provider
LLM_PROVIDER=gemini|openai|anthropic
LLM_API_KEY=xxx
LLM_MODEL=gemini-1.5-pro-latest

# Vector Store
VECTOR_STORE=pinecone|weaviate|chroma
VECTOR_STORE_API_KEY=xxx
VECTOR_STORE_INDEX=email_decision_engine

# Consumer Adapters
CONSUMER_ADAPTERS=email,slack,webhook
SLACK_WEBHOOK_URL=xxx
WEBHOOK_RESPONSE_URL=xxx

# Observability
LOG_LEVEL=info|debug|error
LOG_RETENTION_DAYS=90
METRICS_ENDPOINT=prometheus|datadog
```

---

## 9. Testing Strategy

### 9.1 Test Cases

| ID | Descripción | Input | Expected Output |
|----|-------------|-------|-----------------|
| **TC-01** | Email informativo simple | "¿Horarios?" | PROVIDE_INFO |
| **TC-02** | Solicitud de cita incompleta | "Quiero cita" | REQUEST_MORE_INFO |
| **TC-03** | Riesgo médico crítico | "Dolor pecho" | ESCALATE_HUMAN (CRITICAL) |
| **TC-04** | Fuera de dominio | "Declaración impuestos" | REJECT_REQUEST |
| **TC-05** | Email ambiguo | "Hola" | REQUEST_MORE_INFO |
| **TC-06** | Solicitud accionable | "Necesito factura" | LOG_AND_ROUTE |

### 9.2 Test Execution

```bash
# Unit tests
npm test -- unit/

# Integration tests
npm test -- integration/

# End-to-end tests
npm test -- e2e/

# Load tests
npm test -- load/ --emails=1000
```

---

## 10. Success Metrics (Técnicos)

| Métrica | Fórmula | Target | Frecuencia |
|---------|---------|--------|------------|
| Decision completeness | decisions_valid / total | ≥ 95% | Diario |
| Schema validation rate | valid_schemas / total | 100% | Diario |
| P95 latency | percentile_95(latency) | < 5s | Diario |
| Error rate | errors / total | < 1% | Diario |
| Consumer satisfaction | survey_score | ≥ 4.0/5.0 | Semanal |

---

## 11. Deployment Checklist

### Pre-Deployment
- [ ] Input Contract v1 validado
- [ ] Decision Contract v1 validado
- [ ] Response Contract v1 validado
- [ ] Vector store indexado con documentos
- [ ] LLM provider configurado y testeado
- [ ] Consumer adapters configurados
- [ ] Monitoring y logging activos
- [ ] Health check endpoint funcionando
- [ ] Tests unitarios pasando (≥ 90% coverage)
- [ ] Tests de integración pasando

### Post-Deployment
- [ ] Monitorizar métricas por 24h
- [ ] Verificar logs sin errores críticos
- [ ] Validar throughput esperado
- [ ] Confirmar latencia dentro de targets
- [ ] Documentar cualquier anomalía

---

## 12. Version History

| Versión | Fecha | Cambios | Autor |
|---------|-------|---------|-------|
| 1.0 | 2026-02-18 | Especificación inicial | O.A. Perez Garrido |

---

## 13. Appendices

### Appendix A: Constraints Catalog

| ID | Descripción | Dominio |
|----|-------------|---------|
| C01 | No inventar información médica/legal/técnica | Todos |
| C02 | No responder por el usuario final | Todos |
| C03 | Si ambigüedad → REQUEST_MORE_INFO | Todos |
| C04 | Priorizar riesgo sobre velocidad | Todos |
| C05 | Si no entiendes → comprehension=false | Todos |
| C06 | SCHEDULE_REQUEST solo con datos completos | Todos |
| C07 | LOG_AND_ROUTE para solicitudes no-cita | Todos |
| C08 | Síntomas → ESCALATE_HUMAN | Clínica |
| C09 | Legal risk → ESCALATE_HUMAN | Legal |
| C10 | Financial risk → ESCALATE_HUMAN | Finanzas |

### Appendix B: Domain-Specific Extensions

| Dominio | User States adicionales | Risk Signals | Constraints |
|---------|------------------------|--------------|-------------|
| **Clinic** | MEDICAL_INQUIRY, ADMIN_REQUEST, APPOINTMENT_REQUEST | síntomas, dolor, sangrado | C08 |
| **Travel** | TRIP_INQUIRY, BOOKING_REQUEST, ITINERARY_CHANGE | presupuesto, fechas, destino | C11 |
| **Solar** | TECHNICAL_INQUIRY, QUOTE_REQUEST, INSTALLATION | consumo, ubicación, tech specs | C12 |

---

## ✅ Aprobaciones

| Rol | Nombre | Fecha | Firma |
|-----|--------|-------|-------|
| Product Owner | | | |
| Tech Lead | | | |
| QA Lead | | | |
| DevOps | | | |

---

**Documento generado:** 18 de febrero de 2026  
**Próxima revisión:** 18 de marzo de 2026  
**Status:** Ready for Implementation