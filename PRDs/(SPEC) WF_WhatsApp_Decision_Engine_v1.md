
**Workflow:** WF_WhatsApp_Decision_Engine_v1
**Versión:** 1.0  
**Fecha:** 18 de febrero de 2026  
**Estado:** Especificación para implementación  
**Dominio:** WhatsApp Decision Engine (Agnóstico)  
**Stack referencia:** n8n + WhatsApp Business API + LLM Provider + Vector Store

---

## 1. Overview Técnico

### 1.1 Propósito del sistema
Extensión multicanal del CX Engine Pro que procesa mensajes entrantes de WhatsApp mediante los mismos contratos de decisión que email, manteniendo consistencia arquitectónica y auditabilidad.

### 1.2 Arquitectura de alto nivel
```
WhatsApp Message → WhatsApp Adapter → Normalización → Decision Engine (LLM) → Routing → Response Contract → WhatsApp Adapter
```

### 1.3 Principios técnicos fundamentales
- **Contrato-unificado:** Mismo Decision Contract v1 que email
- **Canal-agnóstico:** La decisión no conoce el canal, solo el adapter
- **Consistencia de output:** Mismo Response Contract v1 estructurado
- **Separación de concerns:** Adapter ≠ Decisión ≠ Ejecución
- **Auditable:** Toda decisión deja traza completa con canal identificado

---

## 2. Problem Statement (Técnico)

### 2.1 Problema actual
La arquitectura actual solo soporta email como canal de entrada, lo que limita:
- **Cobertura de usuarios:** WhatsApp es el canal preferido en mercados LATAM
- **Velocidad de respuesta:** Expectativa de respuesta en minutos, no horas
- **Tipos de interacción:** WhatsApp permite multimedia, ubicación, botones
- **Caso de uso DEMO:** No se puede mostrar multicanalidad real a leads

### 2.2 Impacto técnico
- Arquitectura acoplada a email (webhook, triggers, adapters)
- Contracts no validados para otros canales
- Sin estrategia de normalización cross-channel
- Riesgo de duplicación de lógica de decisión por canal

---

## 3. Technical Objectives

### 3.1 Objetivos SMART

| ID | Objetivo | Métrica | Target |
|----|----------|---------|--------|
| **TO-1** | Procesar mensajes WhatsApp con misma decisión que email | Decision consistency rate | ≥ 98% |
| **TO-2** | Mantener latencia aceptable para chat síncrono | P95 latency | < 3 segundos |
| **TO-3** | Soportar multimedia (imágenes, audio, ubicación) | Media type coverage | 4 tipos mínimos |
| **TO-4** | Adapter reutilizable para otros canales (Telegram, SMS) | Adapter abstraction level | 80% código común |
| **TO-5** | Mantener auditabilidad cross-channel | Traceability coverage | 100% |

### 3.2 Requerimientos de calidad (NFRs)

| NFR | Descripción | Target |
|-----|-------------|--------|
| **NFR-1** | Disponibilidad | 99.5% uptime |
| **NFR-2** | Latencia P95 | < 3 segundos (chat es síncrono) |
| **NFR-3** | Throughput | 50 mensajes/minuto |
| **NFR-4** | Data retention | 90 días de logs |
| **NFR-5** | Error rate | < 1% fallos |
| **NFR-6** | WhatsApp API rate limits | Respetar límites oficiales |

---

## 4. Scope Técnico

### 4.1 In Scope

| Componente | Descripción | Responsabilidad |
|------------|-------------|-----------------|
| **WhatsApp Ingestion** | Webhook oficial WhatsApp Business API | Trigger/Adapter |
| **Message Normalization** | Convierte WhatsApp payload a Input Contract v1 | Transformación |
| **Media Handling** | Procesa imágenes, audio, ubicación, documentos | Transformación |
| **Session Management** | Maneja hilos conversacionales (thread_id) | Estado |
| **Decision Engine** | Mismo LLM skill que email (reutilizado) | LLM Skill |
| **Context Grounding** | Mismo vector store que email (reutilizado) | RAG Tool |
| **Decision Routing** | Mismos IF nodes que email (reutilizado) | Orquestación |
| **Response Wrapper** | Mismo Response Contract v1 (reutilizado) | Transformación |
| **WhatsApp Delivery** | Envía respuesta vía WhatsApp Business API | Integración |
| **Audit Logging** | Registra toda decisión con trace_id + canal | Observabilidad |

### 4.2 Out of Scope

| Componente | Razón de exclusión |
|------------|-------------------|
| WhatsApp Business UI | Capa de presentación (separada) |
| Broadcast/mass messaging | Solo 1:1 en v1 |
| Catálogos/tiendas WhatsApp | Complejidad comercial |
| Payment integration | Fuera de scope CX |
| Multi-agent handoff | Solo escalación a humano externo |
| WhatsApp Groups | Solo mensajes directos en v1 |

---

## 5. Functional Requirements

### 5.1 FR-1: WhatsApp Message Ingestion

**ID:** FR-1  
**Prioridad:** HIGH  
**Descripción:** El sistema debe capturar mensajes entrantes desde WhatsApp Business API webhook.

**User Story:**  
Como sistema, quiero recibir mensajes de WhatsApp en tiempo real y convertirlos a formato estandarizado para procesamiento consistente con email.

**Acceptance Criteria:**
- [ ] Webhook configurado con WhatsApp Cloud API (Meta)
- [ ] Verificación de firma HMAC para seguridad
- [ ] Extrae: `from` (número), `message_body`, `timestamp`, `message_id`
- [ ] Detecta tipo de mensaje: `text`, `image`, `audio`, `location`, `document`
- [ ] Maneja mensajes reaccionados/editados (ignora en v1)
- [ ] Output conforme a Input Contract v1 extendido

**Input Contract v1 (WhatsApp Extension):**
```json
{
  "input_id": "string (WhatsApp message_id)",
  "channel": "whatsapp",
  "message": "string (texto transcrito o contenido)",
  "context_domain": "string",
  "metadata": {
    "from": "string (phone_number)",
    "timestamp": "ISO-8601",
    "original_message_id": "string",
    "message_type": "text|image|audio|location|document",
    "thread_id": "string (session_identifier)",
    "media_url": "string (optional)",
    "mime_type": "string (optional)"
  }
}
```

---

### 5.2 FR-2: Message Type Normalization

**ID:** FR-2  
**Prioridad:** HIGH  
**Descripción:** El sistema debe normalizar diferentes tipos de mensaje WhatsApp a texto procesable.

**User Story:**  
Como motor de decisiones, necesito que todos los tipos de mensaje (texto, audio, imagen) se conviertan a representación textual para clasificación consistente.

**Acceptance Criteria:**
- [ ] **Texto:** Pasa sin modificación
- [ ] **Imagen:** Extrae descripción/caption si existe, sino marca como `IMAGE_NO_CONTEXT`
- [ ] **Audio:** Transcribe a texto (speech-to-text) antes de decisión
- [ ] **Ubicación:** Convierte a `LOCATION: lat, long` + contexto inverso si disponible
- [ ] **Documento:** Extrae metadata (nombre, tipo), contenido en v2
- [ ] Tipos no soportados → `REJECT_REQUEST` con razón clara

**Pseudocódigo de normalización:**
```
FUNCTION normalize_whatsapp_message(raw_payload):
    SWITCH raw_payload.message_type:
        CASE "text":
            RETURN raw_payload.text.body
            
        CASE "image":
            IF raw_payload.image.caption EXISTS:
                RETURN "IMAGEN: " + raw_payload.image.caption
            ELSE:
                RETURN "IMAGEN_SIN_DESCRIPCION"
                
        CASE "audio":
            audio_url = raw_payload.audio.url
            transcript = CALL speech_to_text_service(audio_url)
            RETURN "AUDIO_TRANSCRITO: " + transcript
            
        CASE "location":
            lat = raw_payload.location.latitude
            long = raw_payload.location.longitude
            RETURN "UBICACION: " + lat + ", " + long
            
        CASE "document":
            filename = raw_payload.document.filename
            mime_type = raw_payload.document.mime_type
            RETURN "DOCUMENTO: " + filename + " (" + mime_type + ")"
            
        DEFAULT:
            RETURN NULL (trigger REJECT_REQUEST)
```

---

### 5.3 FR-3: Session & Thread Management

**ID:** FR-3  
**Prioridad:** MEDIUM  
**Descripción:** El sistema debe mantener contexto conversacional por hilo de WhatsApp.

**User Story:**  
Como sistema de CX, necesito identificar mensajes del mismo usuario en secuencia para mantener coherencia sin caer en chatbot conversacional.

**Acceptance Criteria:**
- [ ] `thread_id` generado por combinación `from + timestamp_window`
- [ ] Ventana de sesión: 24 horas desde último mensaje
- [ ] Session data almacenada en cache (Redis/memory)
- [ ] Session NO afecta decisión (stateless por diseño)
- [ ] Session SOLO para logging y trazabilidad
- [ ] Session expira automáticamente después de ventana

**Pseudocódigo de session management:**
```
FUNCTION get_or_create_thread(phone_number, timestamp):
    cache_key = "whatsapp_thread:" + phone_number
    existing_session = CACHE.get(cache_key)
    
    IF existing_session EXISTS AND 
       (timestamp - existing_session.created_at) < 24_HOURS:
        RETURN existing_session.thread_id
    ELSE:
        new_thread_id = GENERATE_UUID()
        CACHE.set(cache_key, {
            thread_id: new_thread_id,
            created_at: timestamp,
            phone_number: phone_number
        }, ttl: 24_HOURS)
        RETURN new_thread_id
```

---

### 5.4 FR-5: Decision Engine Reuse (Critical)

**ID:** FR-5  
**Prioridad:** HIGH  
**Descripción:** El sistema debe reutilizar el mismo Decision Engine que email sin modificaciones.

**User Story:**  
Como arquitecto, quiero que la lógica de decisión sea idéntica entre canales para garantizar consistencia y reducir mantenimiento.

**Acceptance Criteria:**
- [ ] Mismo skill LLM (`decision_intake_email_v1` renombrado a `decision_intake_v1`)
- [ ] Mismo Decision Contract v1 output
- [ ] Mismo grounding via vector store
- [ ] Mismo system prompt (sin referencias a canal)
- [ ] El canal es metadata, no input de decisión
- [ ] Tests de email deben pasar con input WhatsApp normalizado

**Principio arquitectónico:**
> "El Decision Engine no sabe si procesa email o WhatsApp.  
> Solo recibe Input Contract v1 y emite Decision Contract v1.  
> El adapter es el único que conoce el canal."

---

### 5.5 FR-6: WhatsApp Response Delivery

**ID:** FR-6  
**Prioridad:** HIGH  
**Descripción:** El sistema debe enviar respuestas estructuradas vía WhatsApp Business API.

**User Story:**  
Como usuario final, quiero recibir respuestas claras y apropiadas para WhatsApp (formato, longitud, multimedia si aplica).

**Acceptance Criteria:**
- [ ] Respuestas de texto: Máximo 1024 caracteres por mensaje
- [ ] Respuestas largas: Segmentar en múltiples mensajes
- [ ] Soporte para formato WhatsApp (negritas, cursivas, listas)
- [ ] Botones interactivos para `REQUEST_MORE_INFO` (v2)
- [ ] Confirmación de entrega (webhook de estado)
- [ ] Manejo de fallos de envío (reintentos, dead letter)

**Pseudocódigo de delivery:**
```
FUNCTION send_whatsapp_response(response_contract, phone_number):
    action_type = response_contract.final_action.type
    payload = response_contract.final_action.payload
    
    SWITCH action_type:
        CASE "PROVIDE_INFO":
            message = FORMAT_FOR_WHATSAPP(payload.message)
            segments = SPLIT_IF_LONG(message, max_length: 1024)
            FOR EACH segment IN segments:
                WHATSAPP_API.send_text(phone_number, segment)
                
        CASE "REQUEST_MORE_INFO":
            message = "Necesito más información:\n"
            FOR EACH item IN payload.missing_information:
                message = message + "• " + item + "\n"
            WHATSAPP_API.send_text(phone_number, message)
            
        CASE "ESCALATE_HUMAN":
            IF payload.level == "CRITICAL":
                message = "⚠️ CASO CRÍTICO - Un agente te contactará en <5 min"
            ELSE:
                message = "Tu caso requiere atención humana. Te contactaremos pronto."
            WHATSAPP_API.send_text(phone_number, message)
            NOTIFY_HUMAN_TEAM(response_contract)
            
        CASE "REJECT_REQUEST":
            message = "Lo sentimos, no podemos ayudarte con esta solicitud."
            WHATSAPP_API.send_text(phone_number, message)
```

---

### 5.6 FR-7: Rate Limiting & Throttling

**ID:** FR-7  
**Prioridad:** MEDIUM  
**Descripción:** El sistema debe respetar límites de WhatsApp Business API.

**User Story:**  
Como operador, quiero que el sistema maneje automáticamente los límites de la API para evitar bloqueos o penalizaciones.

**Acceptance Criteria:**
- [ ] Límite oficial: 80 mensajes/segundo por número de teléfono
- [ ] Cola de salida con rate limiting
- [ ] Backoff exponencial en caso de 429 errors
- [ ] Alertas cuando se acerque a 80% del límite
- [ ] Logging de todos los intentos de envío

**Pseudocódigo de rate limiting:**
```
CLASS WhatsAppRateLimiter:
    max_per_second = 80
    current_window_count = 0
    window_start_time = NOW()
    
    FUNCTION can_send():
        IF NOW() - window_start_time > 1_SECOND:
            current_window_count = 0
            window_start_time = NOW()
            
        IF current_window_count < max_per_second:
            current_window_count = current_window_count + 1
            RETURN true
        ELSE:
            RETURN false
    
    FUNCTION send_with_retry(message):
        WHILE NOT can_send():
            WAIT(100ms)
        
        response = WHATSAPP_API.send(message)
        
        IF response.status == 429:
            WAIT(expponential_backoff())
            RETRY(send_with_retry, max_retries: 3)
```

---

### 5.7 FR-8: Error Handling & Fallback

**ID:** FR-8  
**Prioridad:** HIGH  
**Descripción:** El sistema debe manejar errores de WhatsApp API gracefulmente.

**User Story:**  
Como sistema de producción, quiero fallar de forma segura cuando WhatsApp API no está disponible, sin perder mensajes ni decisiones.

**Acceptance Criteria:**
- [ ] Webhook de WhatsApp: Acknowledge inmediatamente (200 OK)
- [ ] Procesamiento asíncrono después de acknowledge
- [ ] Dead Letter Queue para mensajes fallidos
- [ ] Reintentos automáticos (max 3) con backoff
- [ ] Alerta a equipo si DLQ > threshold
- [ ] Fallback a email si WhatsApp indisponible > 5 min

**Error Matrix:**
| Error Type | Código | Manejo |
|------------|--------|--------|
| Invalid webhook signature | 401 | Reject, log security event |
| WhatsApp API unavailable | 503 | Queue + retry, alert |
| Rate limit exceeded | 429 | Backoff + retry |
| Invalid phone number | 400 | Reject, log, notify |
| Message delivery failed | 500 | Retry 3x, then DLQ |
| Template not approved | 400 | Fallback to text, alert |

---

### 5.8 FR-9: Logging & Cross-Channel Audit

**ID:** FR-9  
**Prioridad:** HIGH  
**Descripción:** El sistema debe mantener traza completa unificada entre email y WhatsApp.

**User Story:**  
Como auditor, necesito ver todas las decisiones (email + WhatsApp) en un solo lugar para análisis y compliance.

**Acceptance Criteria:**
- [ ] Mismo schema de log que email
- [ ] Campo `channel` diferencia origen
- [ ] `trace_id` único por decisión (no por canal)
- [ ] Dashboard unificado de métricas
- [ ] Exportable para análisis cross-channel
- [ ] Retención mínima: 90 días

**Audit Schema (Extended):**
```json
{
  "audit_entry": {
    "trace_id": "uuid",
    "channel": "email|whatsapp",
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
      "vector_store_namespace": "string",
      "whatsapp_thread_id": "string",
      "whatsapp_phone_number": "string"
    }
  }
}
```

---

## 6. Non-Functional Requirements

### 6.1 Performance

| Métrica | Target | Medición |
|---------|--------|----------|
| Latencia P50 | < 1.5 segundos | Webhook a respuesta |
| Latencia P95 | < 3 segundos | Percentil 95 |
| Throughput | 50 mensajes/minuto | Sustained load |
| Error rate | < 1% | Fallos / total |

### 6.2 Reliability

| Métrica | Target | Estrategia |
|---------|--------|------------|
| Uptime | 99.5% | Redundancia, monitoring |
| Message durability | 100% | Queue + DLQ |
| Recovery time | < 5 min | Auto-healing, alerts |

### 6.3 Security

| Requisito | Implementación |
|-----------|----------------|
| Webhook verification | HMAC signature validation |
| Data encryption | TLS 1.2+, encryption at rest |
| Phone number masking | En logs, mostrar últimos 4 dígitos |
| PII handling | GDPR/LOPD compliance |
| Access control | API keys, OAuth2 |

### 6.4 Scalability

| Dimensión | Estrategia |
|-----------|------------|
| Horizontal | Stateless design, load balancing |
| Multi-phone | Multiple WhatsApp numbers support |
| Multi-domain | Namespace isolation en vector store |
| Cross-channel | Same decision engine, different adapters |

---

## 7. Technical Specifications

### 7.1 Input/Output Contracts

#### Input Contract v1 (WhatsApp Extension)
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Input Contract v1 (WhatsApp)",
  "type": "object",
  "required": ["input_id", "channel", "message", "context_domain"],
  "properties": {
    "input_id": {"type": "string"},
    "channel": {"type": "string", "enum": ["email", "whatsapp"]},
    "message": {"type": "string"},
    "context_domain": {"type": "string"},
    "metadata": {
      "type": "object",
      "properties": {
        "from": {"type": "string"},
        "timestamp": {"type": "string", "format": "date-time"},
        "original_message_id": {"type": "string"},
        "message_type": {"type": "string", "enum": ["text", "image", "audio", "location", "document"]},
        "thread_id": {"type": "string"},
        "media_url": {"type": "string"},
        "mime_type": {"type": "string"}
      }
    }
  }
}
```

#### Decision Contract v1 (Reutilizado)
> **Sin cambios** — Mismo schema que email  
> Path: `/contracts/decision_contract_v1.json`

#### Response Contract v1 (Reutilizado)
> **Sin cambios** — Mismo schema que email  
> Path: `/contracts/response_contract_v1.json`

---

### 7.2 WhatsApp Adapter Interface

**Pseudocódigo de adapter pattern:**
```
INTERFACE ChannelAdapter:
    FUNCTION ingest(raw_payload) → InputContract
    FUNCTION normalize(raw_message) → string
    FUNCTION deliver(response_contract, recipient) → DeliveryStatus
    FUNCTION get_channel_type() → string

CLASS WhatsAppAdapter IMPLEMENTS ChannelAdapter:
    FUNCTION ingest(raw_payload):
        RETURN {
            input_id: raw_payload.messages[0].id,
            channel: "whatsapp",
            message: normalize(raw_payload.messages[0]),
            context_domain: CONFIG.default_domain,
            metadata: {
                from: raw_payload.contacts[0].wa_id,
                timestamp: raw_payload.messages[0].timestamp,
                original_message_id: raw_payload.messages[0].id,
                message_type: raw_payload.messages[0].type,
                thread_id: get_or_create_thread(raw_payload.contacts[0].wa_id),
                media_url: extract_media_url(raw_payload),
                mime_type: extract_mime_type(raw_payload)
            }
        }
    
    FUNCTION normalize(raw_message):
        // Ver FR-2 para lógica completa
        RETURN normalized_text
    
    FUNCTION deliver(response_contract, recipient):
        // Ver FR-6 para lógica completa
        RETURN delivery_status
    
    FUNCTION get_channel_type():
        RETURN "whatsapp"

CLASS EmailAdapter IMPLEMENTS ChannelAdapter:
    // Implementación existente para email
```

---

### 7.3 Workflow Architecture (n8n)

**Pseudocódigo de flujo:**
```
WORKFLOW WF_WhatsApp_Decision_Engine_v1:
    
    NODE WhatsApp_Webhook:
        TYPE: Webhook
        METHOD: POST
        PATH: /whatsapp/incoming
        VERIFY: HMAC_signature
        
    NODE Set_Input_Schema:
        TYPE: Code (JavaScript)
        INPUT: WhatsApp_Webhook output
        TRANSFORM: CALL WhatsAppAdapter.ingest()
        OUTPUT: Input Contract v1
        
    NODE AI_Agent_Decision:
        TYPE: AI Agent (LLM)
        INPUT: Set_Input_Schema output
        MODEL: gemini-1.5-pro-latest
        PROMPT: decision_intake_v1 (mismo que email)
        RESPONSE_FORMAT: JSON
        OUTPUT: Decision Contract v1
        
    NODE Set_Action:
        TYPE: Set
        EXTRACT: decision.primary_action
        
    NODE IF_Escalate, IF_Provide_Info, IF_Request_More_Info, IF_Reject_Request:
        TYPE: IF
        EVALUATE: decision.primary_action
        // Mismos nodos que workflow de email
        
    NODE Set_*_Action:
        TYPE: Set
        BUILD: final_action payload
        // Mismos nodos que workflow de email
        
    NODE Set_Final_Action_Wrapper:
        TYPE: Set
        BUILD: Response Contract v1
        // Mismo nodo que workflow de email
        
    NODE WhatsApp_Response_Delivery:
        TYPE: HTTP Request (WhatsApp API)
        INPUT: Set_Final_Action_Wrapper output
        CALL: WhatsAppAdapter.deliver()
        
    NODE Respond_to_Webhook:
        TYPE: Respond to Webhook
        STATUS: 200 OK
        // Acknowledge inmediato a WhatsApp
```

---

### 7.4 Environment Variables

```bash
# WhatsApp Business API
WHATSAPP_ENABLED=true
WHATSAPP_API_VERSION=v18.0
WHATSAPP_PHONE_NUMBER_ID=xxx
WHATSAPP_ACCESS_TOKEN=xxx
WHATSAPP_VERIFY_TOKEN=xxx
WHATSAPP_WEBHOOK_SECRET=xxx

# Speech-to-Text (para audio)
STT_PROVIDER=google|azure|openai
STT_API_KEY=xxx
STT_LANGUAGE=es-MX

# Rate Limiting
WHATSAPP_RATE_LIMIT_PER_SECOND=80
WHATSAPP_RETRY_MAX_ATTEMPTS=3
WHATSAPP_RETRY_BACKOFF_BASE=1000ms

# Session Management
SESSION_CACHE_TTL=24h
SESSION_CACHE_PROVIDER=redis|memory

# Observability
LOG_CHANNEL_WHATSAPP=true
METRICS_WHATSAPP_ENABLED=true
```

---

## 8. Implementation Notes

### 8.1 Stack Reference (n8n)

| Componente | Nodo n8n | Configuración |
|------------|----------|---------------|
| WhatsApp Trigger | Webhook | POST, HMAC verify |
| Normalization | Code (JavaScript) | Adapter pattern |
| Speech-to-Text | HTTP Request | External API call |
| Decision Engine | AI Agent (Gemini) | Temperature: 0, JSON output |
| Vector Store | Pinecone | Namespace por dominio |
| Routing | IF nodes | Evaluate primary_action |
| Response Wrapper | Set (JSON) | Build Response Contract |
| WhatsApp Delivery | HTTP Request | WhatsApp Cloud API |
| Session Cache | Redis/Code | Thread management |
| Logging | Write to DB/File | Structured JSON |

### 8.2 Alternative Stacks

| Stack | WhatsApp Adapter | Speech-to-Text | Notes |
|-------|-----------------|----------------|-------|
| **n8n + Cloud API** | HTTP Request | Google Cloud STT | Stack actual |
| **Twilio API** | Twilio Node | Twilio STT | Más caro, más features |
| **360dialog** | 360dialog API | External | Enterprise focus |
| **Custom** | Node.js/Python | Any | Máxima flexibilidad |

### 8.3 WhatsApp-Specific Considerations

| Consideración | Impacto | Mitigación |
|---------------|---------|------------|
| **Ventana de 24h** | Solo puedes responder dentro de 24h después del último mensaje del usuario | Diseñar flujos que cierren dentro de ventana |
| **Templates aprobados** | Mensajes proactivos requieren templates pre-aprobados | Usar solo para follow-ups, no para respuestas reactivas |
| **Costo por conversación** | WhatsApp cobra por conversación (24h) | Optimizar número de mensajes por conversación |
| **Calidad de número** | Baja calidad puede limitar envío | Monitorear phone number quality rating |
| **Multimedia limits** | Imágenes < 5MB, audio < 16MB | Validar antes de enviar |

---

## 9. Testing Strategy

### 9.1 Test Cases

| ID | Descripción | Input | Expected Output |
|----|-------------|-------|-----------------|
| **TC-WA-01** | Mensaje de texto simple | "¿Horarios?" | PROVIDE_INFO |
| **TC-WA-02** | Mensaje de audio | Audio: "Quiero cita" | REQUEST_MORE_INFO (transcrito) |
| **TC-WA-03** | Mensaje con imagen | Imagen + caption | PROVIDE_INFO (usa caption) |
| **TC-WA-04** | Ubicación enviada | Location payload | LOG_AND_ROUTE (coordenadas) |
| **TC-WA-05** | Riesgo médico | "Dolor pecho" | ESCALATE_HUMAN (CRITICAL) |
| **TC-WA-06** | Fuera de dominio | "Declaración impuestos" | REJECT_REQUEST |
| **TC-WA-07** | Sesión misma conversación | 2do mensaje en 24h | Mismo thread_id |
| **TC-WA-08** | Rate limit | 85 mensajes/segundo | Queue + throttle |
| **TC-WA-09** | Cross-channel consistency | Mismo mensaje email + WA | Misma decisión |
| **TC-WA-10** | Webhook security | Firma HMAC inválida | 401 Reject |

### 9.2 Test Execution

```bash
# Unit tests (adapter logic)
npm test -- unit/whatsapp-adapter/

# Integration tests (end-to-end)
npm test -- integration/whatsapp-decision/

# Cross-channel consistency tests
npm test -- e2e/cross-channel-consistency/

# Load tests (rate limiting)
npm test -- load/whatsapp-rate-limit/ --messages=1000

# Security tests (webhook verification)
npm test -- security/webhook-hmac/
```

---

## 10. Success Metrics (Técnicos)

| Métrica | Fórmula | Target | Frecuencia |
|---------|---------|--------|------------|
| Decision consistency (email vs WA) | same_decision / total_cross_channel | ≥ 98% | Diario |
| WhatsApp delivery rate | delivered / attempted | ≥ 99% | Diario |
| P95 latency (WA vs email) | percentile_95(wa_latency) | < 3s | Diario |
| Speech-to-text accuracy | correct_transcripts / total_audio | ≥ 95% | Semanal |
| Rate limit violations | throttled_requests / total | < 0.1% | Diario |
| Session continuity | same_thread_messages / total | ≥ 90% | Diario |

---

## 11. Deployment Checklist

### Pre-Deployment
- [ ] WhatsApp Business API cuenta aprobada
- [ ] Número de teléfono verificado
- [ ] Webhook configurado y verificado (HMAC)
- [ ] Templates aprobados (si aplica)
- [ ] Speech-to-text API configurada
- [ ] Rate limiting configurado
- [ ] Session cache configurado (Redis)
- [ ] DLQ configurado para fallos
- [ ] Monitoring y alertas activos
- [ ] Tests de cross-channel consistency pasando
- [ ] Security audit (webhook verification)

### Post-Deployment
- [ ] Monitorizar métricas por 24h
- [ ] Verificar logs sin errores críticos
- [ ] Validar latencia dentro de targets
- [ ] Confirmar decision consistency con email
- [ ] Testear fallback scenarios
- [ ] Documentar cualquier anomalía

---

## 12. Version History

| Versión | Fecha | Cambios | Autor |
|---------|-------|---------|-------|
| 1.0 | 2026-02-18 | Especificación inicial WhatsApp | O.A. Perez Garrido |

---

## 13. Appendices

### Appendix A: WhatsApp Message Type Matrix

| Tipo | Soporte v1 | Normalización | Notas |
|------|------------|---------------|-------|
| Texto | ✅ | Directo | Sin cambios |
| Imagen | ✅ | Caption o `IMAGE_NO_CONTEXT` | Sin OCR en v1 |
| Audio | ✅ | Speech-to-text | Requiere STT API |
| Video | ❌ | N/A | Postponed to v2 |
| Documento | ✅ | Metadata only | Contenido en v2 |
| Ubicación | ✅ | Coordenadas | Reverse geocoding opcional |
| Contacto | ❌ | N/A | Postponed to v2 |
| Sticker | ❌ | N/A | Postponed to v2 |
| Reacción | ❌ | N/A | Ignorar en v1 |

### Appendix B: Cross-Channel Decision Consistency

**Principio:** La misma consulta debe producir la misma decisión sin importar el canal.

| Consulta | Email Decision | WhatsApp Decision | Consistency |
|----------|----------------|-------------------|-------------|
| "¿Horarios?" | PROVIDE_INFO | PROVIDE_INFO | ✅ |
| "Quiero cita" | REQUEST_MORE_INFO | REQUEST_MORE_INFO | ✅ |
| "Dolor pecho" | ESCALATE_HUMAN (CRITICAL) | ESCALATE_HUMAN (CRITICAL) | ✅ |
| "Declaración impuestos" | REJECT_REQUEST | REJECT_REQUEST | ✅ |

**Validación:** Tests automatizados deben verificar consistencia en cada deployment.

### Appendix C: WhatsApp Business API Limits

| Límite | Valor | Nota |
|--------|-------|------|
| Mensajes/segundo | 80 | Por número de teléfono |
| Ventana de respuesta | 24 horas | Desde último mensaje del usuario |
| Tamaño mensaje texto | 4096 caracteres | Recomendar < 1024 |
| Tamaño imagen | 5 MB | JPEG, PNG |
| Tamaño audio | 16 MB | AAC, MP3 |
| Tamaño documento | 100 MB | Cualquier tipo |
| Rate limit tier | Basado en calidad | Monitorear phone number quality rating |

### Appendix D: Security Considerations

| Riesgo | Mitigación |
|--------|------------|
| Webhook spoofing | HMAC signature verification obligatorio |
| Phone number exposure | Mask en logs (últimos 4 dígitos) |
| PII in messages | Encryption at rest, retention policy |
| API token leakage | Secrets management (no hardcoded) |
| Rate limit abuse | Throttling + IP whitelisting |
| Message injection | Input validation + sanitization |

---


**Status:** Ready for Implementation