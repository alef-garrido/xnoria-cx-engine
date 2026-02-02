# System Architecture v1 — xnoria-cx-engine

Este documento describe **cómo interactúan técnicamente Antigravity, n8n y NotebookLM-MCP**, qué contratos existen y **dónde vive cada responsabilidad**.

---

## 1. Objetivo del sistema

Procesar inputs de CX (email inicialmente), **tomar decisiones estructuradas y auditables**, y ejecutar acciones externas **sin acoplar razonamiento, contexto y ejecución**.

---

## 2. Componentes y responsabilidades

### 2.1 n8n — Orchestration Layer

**Responsabilidades**

- Recepción de inputs externos (Webhook / Email)
    
- Orquestación del flujo
    
- Evaluación de decisiones (`IF`, routing)
    
- Ejecución de consumidores (Email, Slack, Ticketing)
    
- Construcción del `Response Contract`
    

**Artefactos**

- `workflows/*.json`
    
- Consumer adapters (Gmail, Send Email, Slack, etc.)
    

**No permitido**

- Interpretación semántica
    
- Evaluación de riesgo
    
- Generación de decisiones
    

---

### 2.2 Antigravity — Decision Engine

**Responsabilidades**

- Ejecutar skills MCP
    
- Clasificar intención
    
- Evaluar riesgo
    
- Aplicar constraints
    
- Emitir decisiones **estructuradas**
    

**Artefactos**

- `.agent/skills/decision_intake_email_v1`
    
- Decision logic encapsulada en el skill
    

**Contratos**

- Entrada: Decision Input Contract
    
- Salida: Decision Output Contract
    

---

### 2.3 NotebookLM-MCP — Context Provider

**Responsabilidades**

- Exponer conocimiento estático (read-only)
    
- Proveer contexto confiable al skill
    

**Artefactos**

- SOPs
    
- FAQs
    
- Políticas
    
- Documentación operativa
    

**No permitido**

- Tomar decisiones
    
- Ejecutar acciones
    
- Modificar flujos
    

---

## 3. Contratos del sistema

### 3.1 Decision Contract (Input → Antigravity)

Define **qué información puede usar el motor de decisión**.

- Vive en: `/schemas/decision_input_v1.json`
    
- Consumido solo por Antigravity
    
- Versionado
    

---

### 3.2 Decision Output Contract (Antigravity → n8n)

Define **qué decisiones puede emitir el motor**.

- Vive en: `/schemas/decision_output_v1.json`
    
- Nunca contiene texto libre
    
- Nunca ejecuta acciones
    

---

### 3.3 Response Contract v1 (n8n → consumidores)

Define **qué recibe cualquier sistema externo**.

Campos clave:

- `response_id`
    
- `timestamp`
    
- `channel`
    
- `final_action { type, payload }`
    
- `meta { version, trace_id }`
    

Este contrato:

- Es estable
    
- Es auditable
    
- Es desacoplado del LLM
    

---

## 4. Flujo de datos (secuencia exacta)

```csharp
[External Input]
      ↓
[n8n Webhook]
      ↓
[Build Decision Input]
      ↓
[Antigravity Skill Execution]
      ↓
[NotebookLM-MCP (internal)]
      ↓
[Decision Output JSON]
      ↓
[n8n IF Routing]
      ↓
[Set_* Nodes]
      ↓
[Set_Final_Action_Wrapper]
      ↓
[Respond to Webhook]
      ↓
[Consumer Adapter]
```

---

## 5. Normalización y encapsulación

- **Normalizar**: convertir múltiples caminos (`Set_*`) en una estructura común (`final_action`)
    
- **Encapsular**: esconder lógica interna antes de exponer respuesta
    
- **Serializar**: emitir solo JSON válido conforme a contrato
    

---

## 6. Principios técnicos del diseño

- Single Decision Source (Antigravity)
    
- Deterministic Execution (n8n)
    
- Read-only Knowledge (NotebookLM)
    
- Contract-first architecture
    
- Versioned outputs
    
- Debuggable at every step
    

---

## 7. Estado actual (freeze v1)

|Elemento|Estado|
|---|---|
|Decision Contract v1|Congelado|
|Response Contract v1|Congelado|
|Email Intake Workflow|Funcional|
|Critical Escalation|Validado|
|Consumer Adapter (Email)|En progreso|

---
