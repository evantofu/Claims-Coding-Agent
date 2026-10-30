# Claims AI Agent

Claims AI Agent is a Python prototype built with Google ADK and Gemini that reads FHIR clinical records, suggests CPT procedure codes, and checks billing requirements through MCP tools.

[Devpost project](https://devpost.com/software/claims-ai-agent) · [Watch the demo](https://www.youtube.com/watch?v=X0_MR93zi7k) · [Rules and retrieval backend](https://github.com/evantofu/Claims-Rules-MCP)

## How to use

The [demo video](https://www.youtube.com/watch?v=X0_MR93zi7k) shows the agent inside Prompt Opinion. Running the complete workflow requires a configured Prompt Opinion/FHIR environment and the companion MCP server.

### 1. Select a patient encounter

In a configured Prompt Opinion workspace, select a test patient and ask the agent to code the encounter. The platform supplies the patient's FHIR server, access token, and patient ID as request metadata.

### 2. Review the clinical documentation

The agent retrieves demographics and clinical notes, with tools for conditions, medications, observations, and coverage. It uses the documented services to propose procedure codes.

### 3. Check billing requirements

For each proposed code, the agent calls the companion MCP server for CMS baseline rules and available payer-policy evidence. The response can identify eligibility issues, modifier requirements, and missing documentation.

### 4. Inspect the coding summary

Review the proposed codes, modifiers, clinical evidence, removed codes, and documentation flags. This is a hackathon/research prototype; its output requires review before any real billing use.

## Prerequisites

- Python 3.11 or later and Git
- A Google API key with access to the configured Gemini model
- The [Claims Rules MCP server](https://github.com/evantofu/Claims-Rules-MCP) for billing checks
- Authorized access to a test FHIR environment for patient-record queries
- A Prompt Opinion workspace for the platform demonstration

## Installation

### 1. Clone and install

```bash
git clone https://github.com/evantofu/Claims-Coding-Agent.git
cd Claims-Coding-Agent
python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

On Windows, activate the environment with `.venv\Scripts\activate`.

### 2. Configure the environment

Create a `.env` file in the repository root. The repository currently does not include an `.env.example` file.

```dotenv
GOOGLE_API_KEY=your-google-api-key
GOOGLE_GENAI_USE_VERTEXAI=FALSE
API_KEYS=your-local-agent-key
CLAIMS_MCP_URL=http://localhost:5000/mcp
HEALTHCARE_AGENT_URL=http://localhost:8001
PO_PLATFORM_BASE_URL=http://localhost:5139
LOG_FULL_PAYLOAD=false
```

Use the same agent key in your caller's `X-API-Key` header. Replace the platform URL with your workspace URL. The MCP server defaults to port 5001 unless `PORT` is set; start it with `PORT=5000 npm start`, or update `CLAIMS_MCP_URL` to match.

### 3. Start the claims agent

From the repository root:

```bash
python -m uvicorn healthcare_agent.app:a2a_app --host 127.0.0.1 --port 8001
```

Check its public agent card:

```bash
curl http://localhost:8001/.well-known/agent-card.json
```

To connect from Prompt Opinion, expose the service through your deployment or development tunnel, set `HEALTHCARE_AGENT_URL` to that public URL, restart the agent, and register its agent-card URL and API key in the workspace.

### 4. Explore the local agent UI

Alternatively, run:

```bash
adk web .
```

Open [localhost:8000](http://localhost:8000). The local UI is useful for inspecting agents, but it does not automatically supply the FHIR metadata used by the complete platform workflow.

## Project structure

| Folder | Purpose |
|---|---|
| `healthcare_agent/` | Claims-coding agent and clinical-document tools |
| `general_agent/` | Supporting date/time and small ICD-10 reference example |
| `orchestrator/` | Delegation to the specialist agents |
| `shared/` | A2A app factory, API-key middleware, FHIR context, and shared tools |
| `eval/` | Golden cases, response parsing, scoring, and repeated-run evaluation |

The project extends the Prompt Opinion ADK examples. The other agents and shared infrastructure are retained alongside the claims-specific implementation.

## Evaluation

Six golden-case files cover preventive visits, Medicare wellness, pediatric care, chronic care, asthma, and wrong-code correction. The evaluation code measures code precision/recall/F1, modifier accuracy, prohibited-code violations, and latency. Repeated-run reports track variability and retries.

Historical reports in the repository cover different versions and case sets. They should not be treated as a current score or evidence of general clinical accuracy. See [Development and Evaluation](docs/DEVELOPMENT.md) to configure the live dependencies and regenerate results.

## Troubleshooting

- **401 Unauthorized:** set `API_KEYS` and send a matching `X-API-Key` header.
- **FHIR context missing:** use the configured platform or supply FHIR metadata in the A2A request; a patient ID alone does not grant access.
- **MCP connection fails:** confirm the companion server is running and its port matches `CLAIMS_MCP_URL`.
- **No payer citations:** start Chroma and ingest the relevant policy documents in the companion repository.
- **Rate limits or expired tokens:** reduce evaluation frequency and refresh your authorized test credentials.

## Development notes

The bundled rules cover a limited set of codes, payer evidence depends on the loaded documents, and the agent contains demo-specific instructions and payer fallbacks. Broader datasets need additional validation.

See [Development and Evaluation](docs/DEVELOPMENT.md) for configuration details, FHIR request metadata, Docker, compatibility shims, and evaluation commands.
