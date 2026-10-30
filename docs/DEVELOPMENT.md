# Claims AI Agent — Development and Evaluation

[Back to the project README](../README.md)

## Shared infrastructure

The claims agent builds on the Prompt Opinion Google ADK examples. `shared/app_factory.py` constructs the A2A application and agent card. `shared/middleware.py` validates agent API keys and bridges FHIR metadata. `shared/fhir_hook.py` extracts that context into session state, where `shared/tools/fhir.py` reads it when executing tools.

The orchestrator delegates to the healthcare and general agents in-process through ADK `AgentTool`; it does not need to call their separate HTTP endpoints. The claims agent separately connects to the companion MCP service.

## Configuration

Create `.env` yourself; no `.env.example` is currently tracked. Keep credentials out of version control.

| Variable | Purpose |
|---|---|
| `GOOGLE_API_KEY` | Gemini access |
| `GOOGLE_GENAI_USE_VERTEXAI` | Set `FALSE` for the API-key configuration shown in the README |
| `API_KEYS` | Comma-separated valid agent API keys |
| `API_KEY_PRIMARY`, `API_KEY_SECONDARY` | Optional alternative key slots supported by the middleware |
| `CLAIMS_MCP_URL` | Companion MCP endpoint; agent default is `http://localhost:5000/mcp` |
| `PO_PLATFORM_BASE_URL` | Workspace base URL used in the advertised FHIR extension URI |
| `HEALTHCARE_AGENT_URL` | Claims service URL advertised in the agent card |
| `GENERAL_AGENT_URL` | General agent's advertised URL |
| `ORCHESTRATOR_URL` | Orchestrator's advertised URL |
| `LOG_FULL_PAYLOAD` | Set `false` to disable full request-body logging |
| `LOG_HOOK_RAW_OBJECTS` | Optional callback debugging; leave disabled for normal use |

The claims agent requires an API key. The general example is configured as public. The agent-card discovery endpoint remains public even when the agent requires a key.

## FHIR request metadata

Prompt Opinion sends patient context with an A2A message. The values below are placeholders, not working credentials:

```json
{
  "jsonrpc": "2.0",
  "id": "demo-request",
  "method": "message/send",
  "params": {
    "message": {
      "messageId": "demo-message",
      "role": "user",
      "parts": [{"kind": "text", "text": "Code this test encounter and cite the evidence."}],
      "metadata": {
        "https://your-workspace.example/schemas/a2a/v1/fhir-context": {
          "fhirUrl": "https://your-test-fhir-server.example/r4",
          "fhirToken": "your-short-lived-test-token",
          "patientId": "your-test-patient-id"
        }
      }
    }
  }
}
```

Send requests to the agent's root endpoint with `Content-Type: application/json` and the configured `X-API-Key`. Align the metadata URI with your workspace. The FHIR hook stores credentials in session state for tool calls rather than adding them to the model instructions.

The companion MCP tools read FHIR context from HTTP headers. Supplying metadata to the agent does not, by itself, prove those headers are forwarded to MCP. Verify the integration when using the MCP server's patient-age checks; without patient context, those checks can be skipped.

## Running all agents

```bash
python -m pip install -r requirements-dev.txt
honcho start
```

The `Procfile` starts claims/healthcare on 8001, general on 8002, and the orchestrator on 8003. Install the main requirements first.

For Docker, create `.env` before starting:

```bash
docker compose up --build
```

The companion MCP/Chroma services are separate dependencies. Inside a container, `localhost` refers to that container: set `CLAIMS_MCP_URL` to an address reachable from the agent container. Stop the agents with `docker compose down`. Run either the local servers or Compose to avoid port conflicts.

## Running evaluations

The evaluation scripts call live model/FHIR services. They are not an offline unit-test suite. Their case patient IDs must exist in your authorized test environment.

Add these settings to `.env`:

```dotenv
EVAL_API_KEY=your-local-agent-key
EVAL_FHIR_URL=https://your-test-fhir-server.example/r4
EVAL_FHIR_TOKEN=your-short-lived-test-token
```

`HEALTHCARE_AGENT_URL` selects the agent endpoint. For supported token-refresh configurations, inspect `eval/runner.py` for `EVAL_FHIR_TOKEN_URL`, `EVAL_FHIR_REFRESH_TOKEN`, and client credential variables. The refresh path imports `requests`, so install it if needed:

```bash
python -m pip install requests
```

From the repository root:

```bash
python eval/run_evals.py --case tamera_preventive_v1 --verbose
python eval/run_evals.py --output eval/local-results.json
python eval/chaos_run.py --iterations 5 --sleep 5 --output eval/local-stability.json
```

The scorer compares parsed outputs with the case definitions. Inspect precision, recall, F1, modifier accuracy, prohibited-code violations, latency, and parsing/infrastructure failures together. Changing the prompt, model, rules, FHIR data, or case suite changes what a result measures. Earlier four-case reports and later six-case submissions are not interchangeable.

`refresh_token.sh` is environment-specific; inspect its configuration before relying on automatic refresh. The FHIR middleware smoke script is at `scripts/test_fhir_hook.sh`.

## A2A compatibility

The checked-in app factory includes `AgentCardV1` and `AgentExtensionV1` adapters around the installed SDK types, including `supportedInterfaces`, security-scheme serialization, and extension parameters. A method/role shim normalizes the Prompt Opinion request forms expected by this implementation.

These are implementation-specific compatibility layers. Check the installed SDK and platform behavior before removing them or changing the advertised card. This document does not assert compatibility with every future A2A SDK version.

## Deployment

The repository includes a Dockerfile for managed container hosting such as Google Cloud Run. Choose `AGENT_MODULE` and `PORT` for the service being deployed. Configure Gemini and agent API keys in the host's secret/environment settings, publish the service URL in its agent card, and register that URL with Prompt Opinion.

Validate discovery, authentication, FHIR access, MCP reachability, and a test encounter after deployment. Hosting and model usage depend on your provider account and pricing; no free-tier guarantee is assumed.
