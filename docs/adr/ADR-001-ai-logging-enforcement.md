# ADR-001: AI Logging Enforcement ("Pinky Promise" Model)

**Date:** 2025-11-03  
**Status:** Proposed  
**Related Systems:** n8n Workflows, Queue Manager, AI Proxy, PostgreSQL (AI Compliance DB)

---

## Context
To ensure traceability and compliance for all AI-related workflows, every AI call must include workflow metadata.
The n8n Community Edition lacks fine-grained RBAC and cannot enforce mandatory logging blocks.
Manual "pinky promise" compliance relies on developer discipline, which is insufficient for audit-grade observability.

## Decision
Introduce enforcement at orchestration level via Queue Manager workflow:
- Business and AI flows must include `$workflow.name` and `$run.id` in AI prompts.
- Queue Manager validates completion reports for these identifiers.
- If missing, the item still completes but logs a **policy violation** record.
- Violations are visible in observability dashboards and can trigger alerts.

## Rationale
- Enforces accountability without disrupting workflow UX.
- Maintains segregation of duties: enforcement is outside business workflows.
- Enables correlation of AI prompt logs with n8n execution logs for full traceability.

## Consequences
- Business flows remain lightweight and unchanged.
- Enforcement logic centralized in Queue Manager.
- Occasional false positives possible during early rollout.
- Future upgrade path: automated strict-mode (block if missing metadata).

## References
- [AI Gateway Proxy design](./ADR-000-ai-gateway-proxy.md)
- [Compliance DB schema](../schema/ai_compliance.sql)
