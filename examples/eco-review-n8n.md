# Eco-Review: n8n

> **Repository:** [n8n-io/n8n](https://github.com/n8n-io/n8n) (~55k stars)
> **What it is:** Visual workflow automation platform. Node-based editor, 400+ integrations, webhook/polling/cron triggers, self-hostable.
> **Stack:** Vue 3, Express, BullMQ, Redis, PostgreSQL/SQLite, TypeScript, pnpm + Turbo monorepo

---

## Top Recommendations

### 1. I3: Always-on compute model

**Found in:** Core architecture, `docker-compose.yml`, worker configuration
**Details:** n8n **must run 24/7** — there is no scale-to-zero capability. Even with zero active workflows, the server stays running to listen for webhooks, fire cron triggers, and poll external services. Production requires 3+ always-on services: n8n main instance + Redis + PostgreSQL. Community attempts to run on Cloud Run (scale-to-zero) break polling and cron triggers because the architecture assumes a persistent process.

**Impact:** A workflow that runs once per day wastes 23 hours and 59 minutes of always-on server time. Separating the trigger layer from execution could eliminate most idle compute.
**Effort:** Architectural — would require fundamental redesign of trigger handling
**Source:** NRDC Data Center Efficiency Assessment, 2014; AWS Well-Architected Sustainability Pillar

### 2. C1: Polling triggers at fixed intervals

**Found in:** Trigger node implementations (Gmail, Google Sheets, Airtable, RSS, IMAP)
**Details:** Many integrations use polling triggers at fixed intervals. A Gmail poll trigger at 1-minute intervals generates **1,440 API calls/day** even when no new emails arrive. Each poll fires a complete workflow check cycle regardless of whether data changed. Polling is per-workflow — 10 Gmail triggers = 14,400 API calls/day. No exponential backoff when consecutive polls find nothing.

**Impact:** Exponential backoff alone reduces poll count by 50-80% for low-activity sources. Webhook preference eliminates polling entirely for supported services.
**Effort:** Moderate — add backoff logic to polling base class; surface webhook alternatives in UI
**Source:** GSF Green Software Patterns (https://patterns.greensoftware.foundation/)

### 3. C5: Monolithic integration bundle

**Found in:** `packages/nodes-base/`, frontend build configuration
**Details:** All **400+ integrations** are bundled into every deployment via the `nodes-base` package. No tree-shaking or selective loading — a user who needs 5 nodes ships with 400+. The frontend loads configuration UIs for all nodes, requiring **8 GB build memory**. The `nodes-base` package is the largest in the monorepo.

**Impact:** Selective loading (`N8N_NODES=gmail,slack,postgres`) could reduce Docker image by 30-50% and frontend bundle dramatically. The 8 GB build memory requirement would drop significantly.
**Effort:** Moderate — extend the existing "community nodes" plugin pattern to core nodes
**Source:** webpack/Vite documentation; Docker best practices

### 4. I2: Oversized Docker image

**Found in:** `docker/images/n8n/Dockerfile`
**Details:** The Docker image is **400-500 MB compressed**. It bundles graphicsmagick, full-icu (~30 MB for non-Latin locale support), git, and openssh into every image. Most workflow users don't need image processing or git operations. No "slim" variant is offered.

**Impact:** A slim variant without graphicsmagick, git, openssh could target <200 MB — 50%+ reduction. Faster pulls, less registry storage, less network transfer.
**Effort:** Quick fix — create an `n8n:slim` Dockerfile variant
**Source:** Docker best practices (https://docs.docker.com/build/building/best-practices/)

### 5. P3: Execution data growth

**Found in:** Execution data layer, SQLite/PostgreSQL storage
**Details:** n8n has built-in execution pruning (14-day default, 10K max — this is good). However, community reports indicate PostgreSQL reaching **tens of gigabytes within weeks** under moderate load. Binary data (file attachments) processed in workflows compounds the problem. SQLite mode has no auto-vacuum — pruned data doesn't reclaim disk space.

**Impact:** Tiered retention (full data 24h → metadata only 14d → delete) could reduce database size by 70-80%.
**Effort:** Moderate — add tiered retention and enable auto-vacuum by default
**Source:** AWS Well-Architected Sustainability Pillar

### 6. I1: No green-region guidance for self-hosters

**Found in:** Self-hosting documentation
**Details:** n8n is primarily self-hosted. No documentation recommends low-carbon cloud regions. Users deploying on AWS, GCP, or Azure have no guidance on choosing regions like us-west-2 (Oregon, 79 gCO2/kWh) over us-east-1 (Virginia, 283 gCO2/kWh). For an always-on service, region choice has outsized impact.

**Impact:** 2-5x carbon reduction by choosing a green region. Since n8n runs 24/7, this compounds significantly.
**Effort:** Quick fix — add region recommendations to self-hosting docs
**Source:** Google Cloud Region Carbon Data, 2024 (https://cloud.google.com/sustainability/region-carbon)

### 7. A5: N+1 database queries

**Found in:** TypeORM entity relations, workflow/execution loading
**Details:** n8n uses TypeORM with PostgreSQL. Workflow listings load workflow entities, then separately load associated tags, shared users, and execution statistics for each workflow. Execution history views lazy-load node execution data per-execution rather than batch-loading. The admin panel's user listing loads workspaces and credentials per-user. These are classic N+1 patterns in TypeORM's lazy relation loading.

**Impact:** 80% faster at scale with eager loading. For an instance with 100+ workflows, the workflow list page generates hundreds of queries instead of 2-3.
**Effort:** Moderate — add `relations` option to TypeORM `find` calls, or use `QueryBuilder` with `leftJoinAndSelect`
**Source:** PlanetScale blog (https://planetscale.com/); Azure Well-Architected Framework (https://learn.microsoft.com/en-us/azure/well-architected/)

### 8. A6: Verbose logging in production

**Found in:** Execution data storage, workflow logging configuration
**Details:** n8n stores full input/output data for every node execution by default. A 10-node workflow storing JSON payloads at each step generates significant log-equivalent data volume. The `EXECUTIONS_DATA_SAVE_ON_SUCCESS` and `EXECUTIONS_DATA_SAVE_ON_ERROR` flags default to true for all data. Combined with the 14-day retention, this produces substantial storage I/O.

**Impact:** Storing only error data by default (setting `EXECUTIONS_DATA_SAVE_ON_SUCCESS=none`) could reduce execution storage by 80%+ for successful runs. 30-50% reduction in observability infrastructure costs.
**Effort:** Quick fix — change default to save execution data only on error; let users opt into full logging per-workflow
**Source:** ClickHouse blog (https://clickhouse.com/blog); Logz.io (https://logz.io/)

---

## Category Summary

### Infrastructure
- ✓ **Alpine Docker base** (`node:XX-alpine`) — lightweight foundation
- ✓ **Multi-stage build** — separates build from runtime
- ✓ **Turbo build caching** — avoids redundant monorepo rebuilds in CI
- ✓ **CI run eligibility checks** — intelligent filtering before running jobs
- ⚠ Docker image 400-500 MB with bundled extras (graphicsmagick, git, full-icu) — see #4
- ⚠ No green-region self-hosting guidance — see #6

### Code
- ✓ **BullMQ** for job queuing — efficient, mature queue system
- ✓ **Webhook triggers available** for many services (when users choose them)
- ✓ **Per-workflow retention overrides** — granular control over data lifecycle
- ⚠ All 400+ integrations bundled monolithically — see #3
- ⚠ Polling triggers use fixed intervals with no backoff — see #2

### Product Design
- ✓ **Built-in execution pruning** — 14-day default with 10K max, enabled by default
- ✓ **Per-workflow retention** — users can set different retention per workflow
- ⚠ No UI nudge toward webhooks over polling when both are available
- ⚠ SQLite mode doesn't auto-vacuum after pruning — see #5

### Architecture
- ✓ **Worker separation** — stateless workers scale independently from the main instance
- ✓ **Health check endpoints** — enables orchestration and monitoring
- ✓ **BullMQ + Redis** — efficient pub/sub and job queue pattern
- ⚠ Always-on compute with no scale-to-zero — see #1
- ⚠ TypeORM lazy-loading causes N+1 queries on workflow/execution listings — see #7
- ⚠ Full execution data stored for every successful run by default — see #8
- ⚠ No `/metrics` endpoint for queue depth (would enable auto-scaling workers to zero)

---

*Generated by [eco-first](https://github.com/CNaught-Inc/eco-first) — sustainability-first design for Claude Code*
