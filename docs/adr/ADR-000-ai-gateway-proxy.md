# ADR-002: AI Gateway Proxy Design

**Date:** 2025-11-03  
**Status:** Proposed  
**Related Systems:** n8n Workflows, Techstack (Docker), PostgreSQL (AI Compliance DB)

---

## Context

The automation platform executes workflows that use local and contextual AI models.
Some prompts must go to **Ollama** (local, stateless) while others require **Open WebUI** (contextual, memory-based).
Compliance requires that all AI calls — regardless of backend — be logged centrally with 90-day retention, without relying on users or workflows to handle logging manually.

The current n8n Community Edition lacks granular RBAC and cannot enforce logging blocks.
To maintain **segregation of duties**, logging and routing must be enforced *outside* of n8n workflows.
The solution must be transparent to existing AI nodes and require no UX or node changes.

---

## Decision

Implement a centralized **AI Gateway Proxy** that sits between n8n and AI backends:

```
n8n → AI Gateway Proxy → (Ollama / Open WebUI) → PostgreSQL Logs
```

### Key Design Points

- **Framework:** FastAPI (Python) for simplicity and async request handling.
- **Deployment:** Docker container within the Techstack alongside Ollama, Open WebUI, and Postgres.
- **Routing Logic:**
  - Requests containing context types `"job_parse"` or `"fit_assessment"` → **Ollama**
  - All other requests → **Open WebUI**
  - (Routing rules configurable via YAML or environment variables)
- **Logging:**
  - Each request and response logged in `ai_prompts_log` table:
    - timestamp, model, source (from API key), prompt, response, duration, tokens, context (JSON)
  - Async writes to minimize latency impact.
- **Retention:**
  - Automatic deletion of entries older than 90 days.
  - Cleanup performed daily via pg_cron or scheduled n8n workflow.
- **Auth:**
  - Internal-only access in v1 (restricted Docker network).
  - Future version may add API keys per credential for source identification.
- **Response handling:**
  - Proxy transparently forwards responses to clients, preserving headers and streaming (SSE) behavior.
  - No transformation of payloads.
- **Error Handling:**
  - All HTTP status codes and errors are mirrored to the client.
  - Proxy logs exceptions separately for observability.
- **PII Scrubbing:**
  - Optional placeholder middleware to redact sensitive data fields (TBD).

---

## Example Request Flow

1. n8n AI node sends a standard POST request to the proxy (no workflow changes).
2. Proxy assigns a unique `request_id` and inserts the prompt into Postgres.
3. Routing logic selects destination (Ollama or Open WebUI).
4. Proxy forwards the same JSON body and waits for response.
5. Response is streamed or returned to n8n unchanged.
6. Proxy updates the same record with the model output, latency, and status.

---

## Example Table Schema (PostgreSQL)

```sql
CREATE TABLE ai_prompts_log (
    id SERIAL PRIMARY KEY,
    timestamp TIMESTAMP DEFAULT NOW(),
    source TEXT,
    model TEXT,
    prompt TEXT,
    response TEXT,
    tokens_in INT,
    tokens_out INT,
    duration_ms INT,
    context JSONB,
    error TEXT
);

-- Retention (90 days)
CREATE EXTENSION IF NOT EXISTS pg_cron;
SELECT cron.schedule('ai_log_cleanup', '0 3 * * *', $$
  DELETE FROM ai_prompts_log WHERE timestamp < NOW() - INTERVAL '90 days';
$$);
```

---

## Example Routing Configuration (routes.yml)

```yaml
routes:
  - match:
      context_type: ["job_parse", "fit_assessment"]
    forward_to: "http://ollama:11434/api/generate"
  - match:
      default: true
    forward_to: "http://openwebui:5000/api/chat"
```

---

## Example Docker Service

```yaml
services:
  ai-gateway:
    build: ./ai-gateway
    container_name: ai-gateway
    ports:
      - "8080:8080"
    environment:
      - POSTGRES_DSN=postgresql://user:pass@postgres/ai_observability
      - OLLAMA_URL=http://ollama:11434
      - OPENWEBUI_URL=http://openwebui:5000
      - ENABLE_LOGGING=true
      - ROUTE_CONFIG=/config/routes.yml
    volumes:
      - ./ai-gateway/config:/config
    depends_on:
      - postgres
      - ollama
      - openwebui
```

---

## Rationale

- **Segregation of Duties:** Logging enforcement resides outside of n8n, preventing tampering or bypass.
- **Transparency:** n8n workflows and AI nodes operate identically to direct API calls.
- **Scalability:** Routing and logging are centralized and can extend to future AI backends.
- **Compliance:** Enables audit-grade observability for AI activity with retention and traceability.

---

## Consequences

- Introduces one extra network hop (~10–20 ms latency per call).
- Proxy must be maintained and monitored as a production service.
- Additional database writes for every AI request (negligible at small scale).
- Future updates may add authentication and rate limiting per AI client key.

---

## Future Enhancements

- Add per-credential API keys and authentication.
- Expose Grafana/NocoDB dashboards for AI usage metrics.
- Implement automated anomaly detection (e.g., unusual prompt volume).
- Add configurable PII redaction layer.

---

## References

- [ADR-001](./ADR-001-ai-logging-enforcement.md): AI Logging Enforcement (“Pinky Promise”) Model  
- `docs/schema/ai_compliance.sql` – Database schema reference  
- [FastAPI StreamingResponse docs](https://fastapi.tiangolo.com/advanced/streaming-response/)
