# Eco-Design: Building a Workflow Automation Platform

> **Reference project:** [n8n-io/n8n](https://github.com/n8n-io/n8n) (~55k stars)
> **Premise:** You're building a similar visual workflow automation tool — a node-based editor, 400+ integrations, webhook/polling/cron triggers, and self-hostable deployment.

---

## What n8n Actually Chose vs. Eco-First Alternatives

### 1. Always-On Compute — The Fundamental Problem

**What n8n chose:**
- n8n **must run 24/7** — there is no scale-to-zero capability
- Even with zero active workflows, the server stays running to listen for webhooks, fire cron triggers, and poll external services
- Production requires 3+ always-on services: n8n main instance + Redis + PostgreSQL
- Community attempts to run on Cloud Run (scale-to-zero) break polling and cron triggers

**Eco-first alternative:**
This is the hardest problem and the biggest sustainability gap. A true eco-first workflow platform would:

1. **Separate the trigger layer from the execution layer** — a lightweight "trigger receiver" (serverless function or edge worker) accepts webhooks and writes to a queue. The execution engine scales from zero only when there's work.
2. **Replace polling triggers with push adapters** — for services without webhooks, run a shared polling service that checks multiple accounts/integrations in one process, not one n8n instance per user.
3. **Defer cron triggers to a cloud scheduler** — AWS EventBridge, GCP Cloud Scheduler, or Cloudflare Cron Triggers can fire webhooks at scheduled times without a persistent process.
4. **Hibernate between executions** — checkpoint the engine state to disk, shut down after 5 minutes idle, restore on next trigger.

**Impact:** Eliminates idle compute. A workflow that runs once per day wastes 23 hours and 59 minutes of always-on server time.

*Reference: I3 — NRDC Data Center Efficiency Assessment; AWS Well-Architected Sustainability Pillar*

---

### 2. Polling Triggers — Continuous Waste

**What n8n chose:**
- Many integrations use polling triggers (Gmail, Google Sheets, Airtable, RSS, IMAP)
- A Gmail poll trigger at 1-minute intervals generates **1,440 API calls/day** even when no new emails arrive
- Each poll fires a complete workflow check cycle regardless of whether data changed
- Polling is per-workflow — 10 Gmail triggers = 14,400 API calls/day

**Eco-first alternative:**
1. **Prefer webhooks and surface this in the UI** — when a user selects Gmail Trigger, show: "Webhook mode (recommended, instant, lower resource usage)" vs "Polling mode (checks every N minutes)". Many services support both.
2. **Smart polling with exponential backoff** — if 10 consecutive polls find nothing, increase the interval from 1 minute to 5, then 15. Reset on activity.
3. **Shared polling** — multiple workflows polling the same service could share a connection.
4. **Long-polling / SSE where supported** — Gmail supports push notifications via Google Pub/Sub. IMAP supports IDLE. Use these instead of interval polling.

**Impact:** Exponential backoff alone reduces poll count by 50-80% for low-activity sources. Webhook preference eliminates polling entirely for supported services.

*Reference: C1 — GSF Green Software Patterns (https://patterns.greensoftware.foundation/)*

---

### 3. Execution Data Growth

**What n8n chose:**
- Built-in execution pruning (14-day default, 10K max — this is good)
- However, community reports of PostgreSQL reaching **tens of gigabytes within weeks** under moderate load
- Binary data (file attachments) processed in workflows compounds the problem
- SQLite mode has no auto-vacuum — pruned data doesn't reclaim disk space

**Eco-first alternative:**
The defaults are reasonable — n8n is ahead of many projects here. Improvements:

1. **Separate binary data storage** — don't store file attachments in the execution table. Use a file-based store with independent TTLs.
2. **Auto-vacuum for SQLite** — enable `DB_SQLITE_VACUUM_ON_STARTUP=true` by default.
3. **Tiered retention** — keep full execution data for 24 hours, then retain only metadata (status, duration, error message) for 14 days, then delete.
4. **Execution data compression** — gzip execution data in the database. JSON compresses 60-80%.

**Impact:** Tiered retention could reduce database size by 70-80%.

*Reference: P3 — AWS Well-Architected Sustainability Pillar*

---

### 4. Monolithic Integration Bundle

**What n8n chose:**
- All **400+ integrations** bundled into every deployment via the `nodes-base` package
- No tree-shaking or selective loading — a user who needs 5 nodes ships with 400+
- Frontend loads configuration UIs for all nodes (8 GB build memory required)
- The `nodes-base` package is the largest in the monorepo

**Eco-first alternative:**
1. **Selective node loading** — let self-hosters specify which integrations to include: `N8N_NODES=gmail,slack,postgres,http`. Only load those.
2. **Lazy-load node UIs in the frontend** — when a user drags a Gmail node onto the canvas, dynamically import its configuration UI. Don't ship all 400+ UIs upfront.
3. **Plugin marketplace as the primary model** — ship a minimal core and let users install integrations on demand. This is how n8n's "community nodes" already work — apply the same pattern to core nodes.

**Impact:** Selective loading could reduce the Docker image by 30-50% and the frontend bundle dramatically. The 8 GB build memory requirement would drop significantly.

*Reference: C5 — webpack/Vite documentation; I2 — Docker best practices*

---

### 5. Docker Image

**What n8n chose:**
- `node:XX-alpine` base (good choice)
- Multi-stage build (good)
- ~400-500 MB compressed
- Bundles graphicsmagick, full-icu, git, openssh into every image
- No "slim" variant for users who don't need image processing or git

**Eco-first alternative:**
1. **Slim variant** — `n8n:slim` without graphicsmagick, git, openssh. Most workflow users don't need image processing or git.
2. **Remove full-icu if not needed** — adds ~30 MB for non-Latin locale support. Offer as an option.
3. **Target <200 MB** for the slim variant.

**Impact:** 50% smaller image. Faster pulls, less registry storage.

*Reference: I2 — Docker best practices (https://docs.docker.com/build/building/best-practices/)*

---

### 6. Queue Architecture — What n8n Gets Right

**What n8n chose:**
- **BullMQ** + Redis for job queuing in production
- Workers are stateless and horizontally scalable
- Main instance handles triggers; workers handle execution
- Health check endpoints for orchestration

**Eco-first assessment:**
Well-designed. BullMQ is efficient, Redis is lightweight, and the separation of triggers from execution is correct. The gap: workers can't auto-scale without external orchestration.

**One improvement:** Add a `/metrics` endpoint with queue depth so orchestrators can scale workers to zero when the queue is empty.

*Reference: A1 — AWS Well-Architected Sustainability Pillar*

---

## Summary: Eco-First Workflow Automation Architecture

| Decision | n8n's Choice | Eco-First Choice | Difference |
|----------|-------------|-----------------|------------|
| Compute model | Always-on 24/7 | Trigger receiver + scale-to-zero execution | Eliminates 23+ hrs idle compute |
| Polling triggers | Fixed-interval per-workflow | Exponential backoff + webhook preference | 50-80% fewer API calls |
| Integration bundle | All 400+ in every deployment | Selective loading, plugin marketplace | 30-50% smaller image/bundle |
| Docker image | 400-500 MB with extras | <200 MB slim variant | 50%+ smaller |
| Execution retention | 14 days full data (good defaults) | Tiered: full → metadata → delete | 70-80% less DB storage |
| Frontend bundle | 8 GB build, all node UIs loaded | Lazy-loaded node UIs | Dramatically smaller bundle |
| Queue system | BullMQ + Redis (well-designed) | Same, add auto-scale signals | Workers scale to zero |

### What n8n Gets Right

- ✓ **Built-in execution pruning** — enabled by default with sane limits
- ✓ **Alpine Docker base** — lightweight foundation
- ✓ **BullMQ queue architecture** — efficient, scalable job processing
- ✓ **Worker separation** — stateless workers that scale independently
- ✓ **Turbo build caching** — avoids redundant monorepo rebuilds
- ✓ **CI run eligibility checks** — intelligent filtering before running jobs
- ✓ **Per-workflow retention overrides** — granular control over data lifecycle

---

*Generated by [eco-first](https://github.com/CNaught-Inc/eco-first) — sustainability-first design for Claude Code*
