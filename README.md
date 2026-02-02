# xnoria-cx-engine

**xnoria-cx-engine** is a structured decision-making engine for Customer Experience (CX) tasks. It leverages n8n workflows and Model Context Protocol (MCP) skills to orchestrate email intake, classification, and downstream actions.

## Features

- **Context-Aware Decisions**: Uses a dual-context approach (Base Agnostic + Domain Specific) to evaluate requests.
- **Risk Assessment**: Automatically identifies emotional states, ambiguity, and medical/legal/safety risks.
- **Structured Outputs**: Produces deterministic JSON outputs designed to align with explicit decision and response schemas.
- **n8n Integration**: Integrates with n8n workflows to orchestrate the decision pipeline.

## Project Structure



```text
├── .agent/
│   └── skills/              # MCP skills (decision logic)
├── contracts/               # Decision & response contracts + versioning
├── workflows/               # n8n workflow definitions (JSON)
├── bin/                     # Local utility binaries
└── tests/                   # Test payloads and expected outputs
```

## Key Components

### Decision Intake Email Skill (`.agent/skills/decision_intake_email_v1`)
A core skill designed to classify emails, evaluate risk levels, and select primary actions without generating natural language responses.

### Decision Output Schema (`schemas/decision_output_v1.json`)
The source of truth for all decision outputs, ensuring consistency across all automation steps.

## Getting Started

### Prerequisites
- [n8n](https://n8n.io/) installed and running.
- MCP-compatible AI agent (like Antigravity) configured with access to the skills in this repository.

### Installation
1. Clone the repository:
   ```bash
   git clone git@github:alef-garrido/xnoria-cx-engine.git
   ```
2. Import the workflows from the `workflows/` directory into your n8n instance.
3. Configure your MCP-compatible AI agent (e.g. Antigravity) to load the skill located in `.agent/skills/decision_intake_email_v1`.

## System Dependencies (Critical)

This project relies on external systems that are NOT vendored in this repository.

### 1. notebooklm-mcp-server (REQUIRED)

The decision engine assumes the presence of a configured MCP tool that exposes
NotebookLM contexts to Antigravity skills.

Required contexts:

| Context | Canonical ID | Source |
|-------|-------------|--------|
| Base Context | Context_Agnostic_Base_v1.0 | /docs/Context_Agnostic_Base_v1.0.md |
| Clinic Context | Context_Clinic_Instance_v1.0 | /docs/Context_Clinic_Instance_v1.0.md |

These contexts must be:
- Ingested into NotebookLM
- Exposed via notebooklm-mcp-server
- Registered as an MCP tool in the Antigravity workspace

⚠️ The skill DOES NOT read files directly. Grounding occurs via MCP.

### 2. Gemini Workspace Configuration (Local)

The following file is expected to exist locally but MUST NOT be committed:

`/.agent/notebooks.json`

This file should contain the NotebookLM notebook IDs used by the MCP server.

Example (DO NOT COMMIT):

```json
{
  "notebooks": {
    "Context_Agnostic_Base_v1.0": "notebook_id_here",
    "Context_Clinic_Instance_v1.0": "notebook_id_here"
  }
}
```


## License
This project is licensed under the terms included in the [LICENSE](LICENSE) file.
