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

