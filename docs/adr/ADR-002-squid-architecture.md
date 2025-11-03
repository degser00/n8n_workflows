# ADR-002: SQUiD Architecture — Single-Queue Unified Integration & Dispatch

- **Status:** Accepted
- **Date:** 2025-11-03
- **Deciders:** Platform Eng, Workflow Eng
- **Supersedes / Depends on:** ADR-000 (AI Gateway Proxy), ADR-001 (AI Logging Enforcement “Pinky Promise”)

---

## Context

We need a clear execution architecture for n8n workflows that:
- Separates **event ingestion**, **orchestration**, **business logic**, and **audit logging**.
- Enforces least-privilege access to data stores.
- Produces an authoritative, chronological audit trail for AI prompts/responses tied to **workflow name** and **run id**.
- Works today on a **single-worker** constraint (local AI model capacity), with a path to scale later.

This ADR captures the **SQUiD** model (Single-Queue Unified Integration & Dispatch): lightweight webhooks push events → a queue controls dispatch → a single job-start manager triggers business flows → a job-end manager closes out jobs and enforces AI logging. The AI Logs DB is only reachable by the **Gateway Proxy** (per ADR-000/001); n8n never has credentials for it.

---

## Decision

Adopt **SQUiD** as the core execution model:

1. **Lightweight Webhook Ingestors**  
   - Minimal validation.  
   - Forward events to the **Queue Recorder** (append to Queue DB).  
   - Do **not** run business logic.

2. **Queue (Control Plane)**
   - **Queue Recorder**: creates queue items with full payload + metadata. Immediately triggers **Queue Job Start Manager**.  
   - **Queue Job Start Manager**: picks the next `pending` job, marks it `started`, and triggers the mapped **Business Logic Workflow**.  
     - Capped to **one concurrent business job** by design.  
     - Also invoked by **cron** as fallback.  
   - **Queue Job End Manager**: called by **Business Logic** on completion to mark `done` or `failed` (with error code). Triggers the **AI Gateway Proxy** to write the workflow name + run id into the AI logs (ADR-001 chain).  
     - (Optional wiring) The Job Start Manager’s workflow graph can fan out to business flows that all converge into the Job End Manager node.

3. **Business Logic Workflows**
   - Contain domain logic only; have access **only** to the **Business DB**.  
   - On finish, they call the **Queue Job End Manager** with `{job_id, status, error_code?, workflow_name, run_id}`.

4. **AI Logging Enforcement**
   - On `job_end`, the End Manager sends **workflow_name + run_id** (and any required linkage) to the **AI Gateway Proxy**, which persists to the **Logs DB**.  
   - **n8n has no credentials** for Logs DB. Network isolation can be added later.

5. **No automatic retries** in v1.  
   - Failures are terminal.  
   - If Job Start Manager starts and detects a job marked `started` while **no business workflow is actually running** (i.e., it’s the only active orchestrator), it reclassifies the job as `failed` with an **orphaned/exclusive-violation** error.

---

## Architecture Overview

```text
[Slack/Trello/... Webhooks]
           │
           ▼
  (Lightweight Ingestor Workflows)
           │
           ▼
   [Queue Recorder Workflow] ──► [Queue DB]
           │                          ▲
           ├──────── triggers ────────┘
           ▼
 [Queue Job Start Manager] ── picks next pending, mark started ──► triggers Business Flow
           │                                                       (Business DB only)
           │                              ┌───────────────────────────────┐
           │                              ▼                               │
           │                   calls AI Gateway Proxy ───────► [Logs DB]  │
           │                              ▲                               │
           │                              └───────────────────────────────┘
           │
           ▼
   [Queue Job End Manager] ── mark done/failed ──► [Queue DB]
           │
           ├─► call AI Gateway Proxy ──► [Logs DB] (n8n has no creds)
           │
           └─► triggers next job ───────► [Queue Job Start Manager]

```

**Security boundaries**
- **Business flows → Business DB** only.  
- **Queue workflows → Queue DB** only.  
- **Logs DB** accessible **only** by **AI Gateway Proxy**.

---

## Data Model (Queue DB)

**Table: `queue_jobs`**
- `id` (UUID, PK)
- `created_at` (ts)
- `updated_at` (ts)
- `origin_source` (enum/string) — e.g., `slack:webhook-xyz`, `trello:board-abc`
- `origin_workflow` (string) — name/id of ingestor workflow
- `tenant_id` (string) — optional, for multi-tenant
- `env` (string) — e.g., `dev`, `staging`, `prod`
- `payload` (jsonb) — full event payload (immutable)
- `payload_hash` (string) — for dedupe
- `dedupe_key` (string, nullable)
- `status` (enum) — `pending` | `started` | `done` | `failed`
- `error_code` (string, nullable)
- `error_message` (string, nullable)
- `started_at` (ts, nullable)
- `finished_at` (ts, nullable)
- `dispatched_workflow` (string) — business workflow name
- `dispatched_run_id` (string, nullable) — set at job_end by business flow
- `attempt` (int, default 0) — reserved; no retries in v1
- `metadata` (jsonb) — freeform

**Status transitions (v1)**  
`pending → started → (done | failed)`  
No `retrying`; failures are terminal.

**Initial error codes (suggested)**
- `ORPHANED_JOB`: job marked `started` but no corresponding running business flow.
- `DISPATCH_MAP_MISSING`: no business workflow mapping for item.
- `BUSINESS_FLOW_ERROR`: business flow reported failure.
- `INVALID_PAYLOAD`: ingestor validation found malformed event.

---

## Workflow Contracts

### 1) Webhook Ingestor → Queue Recorder
- **Input:** raw webhook event  
- **Output:** new `queue_jobs` row with populated metadata & payload; triggers Job Start Manager  
- **Validation:** minimal (shape, must-have fields); discard/ignore irrelevant events

### 2) Queue Recorder → Job Start Manager
- **Trigger:** synchronous (event) + **cron fallback** (e.g., every 1–5 min)
- **Selection:** Next `pending` by FIFO (or priority if added later)
- **Actions:**
  - Mark `started`, set `started_at`
  - Resolve **business workflow** mapping (by `origin_source`, payload content, etc.)
  - Trigger business workflow with `{job_id, payload, metadata}`

### 3) Business Workflow → Job End Manager
- **On completion:** call `job_end` with:
  ```json
  {
    "job_id": "...",
    "status": "done" | "failed",
    "error_code": "BUSINESS_FLOW_ERROR|...",
    "workflow_name": "process-xyz",
    "run_id": "<n8n run id>"
  }
  ```
- **Data access:** Business DB only.

### 4) Job End Manager
- **Actions:**
  - Update `queue_jobs` (`done`/`failed`, `finished_at`, store `workflow_name`, `run_id`, `error_code` if any)
  - Call **AI Gateway Proxy** with `{workflow_name, run_id, job_id, timestamps}` to append to AI Logs DB.
  - Trigger **Job Start Manager** again (pull next job).

---

## Triggering & Fallbacks

- **Primary triggers:**  
  - Queue Recorder → Job Start Manager (new item)  
  - Business Flow → Job End Manager (completion)
  - Job End Manager (completion)  →  Job Start Manager (automatically pick next queue item)

- **Fallback trigger:**  
  - **Cron** for Job Start Manager (e.g., every 2 minutes) to:
    - Drain stragglers
    - Detect & mark **orphaned** `started` jobs as `failed` (`ORPHANED_JOB`) if **no business flows are currently running** and the Start Manager is the sole orchestrator.

- **Convergence wiring (optional):**  
  - In n8n, the Start Manager node can branch to various business flows which then converge to a Job End node. This is allowed as long as the **call contract** to Job End Manager remains the same.

---

## Security & Compliance

- **Least privilege:**  
  - Business flows have **no access** to Queue DB or Logs DB.  
  - Queue workflows have **no access** to Business DB or Logs DB.  
  - Logs DB credentials exist **only** in the **AI Gateway Proxy**.

- **Network policy (future hardening):**  
  - Enforce namespace/VNet/policy isolation so that only the Proxy can reach Logs DB.

- **Authoritative audit chain (ADR-001 “Pinky Promise”):**  
  - For each AI prompt/response, the **next chronological record** in the Logs DB will include the **originating workflow name** and **run id** provided by Job End Manager.  
  - **Multi-worker caveat:** Until we scale, this chain is authoritative. When introducing multiple workers, we will **enforce `workflow_name` and `run_id` in the AI payload** itself; the Proxy will reject any AI call missing these fields (blocking the flow).

---

## Concurrency & Scaling

- **v1 constraint:** Job Start Manager ensures **one active business job at a time**, matching local AI capacity.
- **vNext:** Add a configurable **worker_count**; Start Manager dispatches up to `N` concurrent jobs. Or maintain one Start Manager but a distributed token/lease model in Queue DB.

---

## Observability

- **Queue metrics:** counts of `pending/started/done/failed`, age of oldest `pending`, time-in-state.  
- **Business metrics:** success/failure by workflow.  
- **Compliance metrics:** existence & order of AI log chain entries per job/run.

---

## Alternatives Considered

- **Direct webhook → business flow:** Rejected. Hard to enforce ordering, isolation, and audit chain.  
- **Retries in v1:** Rejected to keep failure modes explicit and observable. Retries can mask systemic issues; add later with backoff.

---

## Risks & Mitigations

- **Risk:** Orphaned jobs due to crashes.  
  - **Mitigation:** Cron fallback + orphan detection rule.

- **Risk:** Accidental cross-DB access.  
  - **Mitigation:** Separate credentials per workflow type; policy checks in CI.

- **Risk:** Log chain gaps under concurrency.  
  - **Mitigation:** Single-worker in v1; strict Proxy validation in vNext.

---

## Implementation Notes (n8n)

Create distinct workflows:

1. `wh-<source>-ingestor` (multiple)  
2. `queue-recorder`  
3. `queue-job-start-manager` (also a cron trigger)  
4. `queue-job-end-manager`  
5. `biz-<domain>-<action>` (multiple)  
6. External **AI Gateway Proxy** service (ADR-000/001)

**Mapping:** A central step (in Start Manager) resolves queue item → business workflow by `origin_source` + payload rules.

---

## Open Questions / Future Work

- Priority lanes in Queue (VIP/normal).  
- Backoff/retry policy and error taxonomy expansion.  
- Worker pool & leases for parallelism.  
- Dead-letter queue for repeated operational failures.  
- End-to-end tracing id that spans webhook → queue → business → proxy.
- Potential future split of **Job End Manager** into two specialized components:
  - **Output Manager** — handles business-specific result operations (posting outputs, notifications, etc.).
  - **Queue End Manager** — finalizes queue state and enforces AI logging chain.  
  This separation can improve modularity once more complex workflows evolve.


---

## Glossary

- **SQUiD:** Single-Queue Unified Integration & Dispatch (this architecture).  
- **Job Start Manager:** Orchestrator that picks next job and triggers business flow.  
- **Job End Manager:** Finalizer that updates queue status and enforces AI logging chain.  
- **AI Gateway Proxy:** The only service with credentials to the Logs DB (ADR-000/001).
