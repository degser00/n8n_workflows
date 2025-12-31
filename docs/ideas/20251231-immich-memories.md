# Idea: Fully Automated Immich Stories Pipeline (n8n-based)

## Goal
Automatically generate photo stories (videos, GIFs, compilations) from Immich when enough meaningful photos are added for a given day, including “On This Day” memories.

---

## Core Principles
- Immich is the **source of truth**
- Fully automated, no manual triggers
- Modular, composable n8n workflows
- Deterministic and idempotent
- Analysis separated from rendering

---

## High-Level Architecture

### 1. Poller Workflow
**Type:** Scheduled (cron)

**Responsibilities:**
- Query Immich API for:
  - New photos since last run
  - Photos grouped by capture date
  - “On this day” buckets (1y, 2y, 5y ago, etc.)
- Produce candidate buckets:
  - `{ date, assetIds, count, reason }`

**Output:**
- List of day-buckets with metadata

---

### 2. Dispatcher Workflow
**Type:** Rule-based router

**Responsibilities:**
- Apply thresholds and guards:
  - Minimum photo count
  - Not already processed
  - Optional filters (screenshots, low quality dominance)
- Create queue jobs per bucket:
  - `analyze-day-bucket`

**Output:**
- Queue items with job metadata

---

### 3. Analysis Workflow
**Type:** Worker / consumer

**Responsibilities:**
- Analyze assets in a day-bucket:
  - Image quality
  - Face presence / quality
  - Uniqueness / burst reduction
  - Optional aesthetic or CLIP scoring
- Rank assets and select top X
- Decide what to generate:
  - Story video
  - GIF
  - Album only
  - Multiple outputs

**Output:**
- Persisted analysis result (JSON)
- New queue jobs:
  - `render-story-video`
  - `render-gif`
  - `create-album`
  - `tag-assets`

---

### 4. Rendering Workflows
**Type:** Specialized workers

**Responsibilities:**
- Consume render jobs
- Use analysis output only (no re-analysis)
- Generate media:
  - Story video (FFmpeg)
  - GIFs
- Deterministic rendering settings

**Output:**
- Media files ready for import

---

### 5. Import & Tag Workflow
**Responsibilities:**
- Upload generated media to Immich
- Tag assets:
  - `auto-story`
  - `YYYY-MM-DD`
  - `on-this-day` (if applicable)
- Optionally add to album:
  - “Auto Stories”

---

## Queue Design (Conceptual)
Each job includes:
- jobType
- dateBucket
- assetIds
- reason (new / anniversary)
- status
- attemptCount
- analysisRef (if applicable)

---

## Safety & Quality Guards
- Idempotency per date-bucket
- Retryable rendering (analysis cached)
- Read-only Immich API token
- Stateless workers

---

## Why This Works Well
- Photos are easier than videos
- Daily batching reduces noise
- Modular workflows scale independently
- Matches Google Photos–style Stories without vendor lock-in

---

## Next Step
Define **queue schema and persistence strategy** before implementing analysis logic.
