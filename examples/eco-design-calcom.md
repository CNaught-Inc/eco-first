# Eco-Design: Building a Scheduling Platform

> **Reference project:** [calcom/cal.com](https://github.com/calcom/cal.com) (~35k stars)
> **Premise:** You're building a similar open-source scheduling platform — booking pages, calendar integrations, reminders via email/SMS, webhooks, and a self-hostable Docker deployment.

---

## What Cal.com Actually Chose vs. Eco-First Alternatives

### 1. Background Job Architecture

**What Cal.com chose:**
- **GitHub Actions scheduled workflows as cron job runners** — not a dedicated job queue
- The webhook trigger workflow runs **every single minute**, hitting two regional endpoints (US + EU)
- Email, SMS, and WhatsApp reminders each run as separate cron workflows every 15 minutes
- Total: ~2,880 cron-triggered HTTP requests/day just for webhooks, plus ~288/day for each reminder type
- Each invocation spins up a fresh Ubuntu VM on GitHub Actions to make a single HTTP POST

**Eco-first alternative:**
This is the single biggest sustainability issue. Each GitHub Actions run provisions a VM, checks out code (or runs curl), executes one HTTP call, and tears down. The eco-first version would:

1. **Use a lightweight event-driven queue** — Bull/BullMQ (Redis-backed, already in their stack) or database-backed job queue (like Plausible's Oban). Process webhooks as events, not on a timer.
2. **Consolidate cron into one scheduler** — a single lightweight cron container (or cloud scheduler like AWS EventBridge) replaces 10 separate GitHub Actions workflows.
3. **Make reminders event-triggered** — schedule reminder jobs at booking creation time, not by polling every 15 minutes for due reminders.

**Impact:** Eliminates ~3,500+ daily VM spin-ups. A cron job that finds no work 80% of the time wastes 80% of its compute.

*Reference: A3 — GSF Green Software Patterns; A1 — AWS Well-Architected Sustainability Pillar*

---

### 2. Docker Image Size

**What Cal.com chose:**
- Base image: `node:20` (full Debian) — not Alpine or slim
- Multi-stage build with 3 stages (Builder, Builder-Two, Runner)
- Final image: **~5 GB** on Docker Hub
- Build requires **6-8 GB Node memory** (`MAX_OLD_SPACE_SIZE=6144-8192`)
- Includes diagnostic tools (`netcat`, `wget`) in production

**Eco-first alternative:**
5 GB is exceptionally large for a web application. The eco-first version would:

1. **Use `node:20-alpine`** as the base — cuts hundreds of MB from the base layer alone
2. **Strip diagnostic tools** from production (install only in a debug variant)
3. **Optimize the build** — the 6-8 GB memory requirement suggests the build graph could be optimized (split builds, reduce in-memory dependencies)
4. **Target <500 MB** final image — Plausible's comparable Elixir/Phoenix app ships at 58 MB

**Impact:** 50-90% reduction in image size. Every `docker pull` transfers less data. Registry storage drops proportionally. Cold starts are faster.

*Reference: I2 — Docker best practices (https://docs.docker.com/build/building/best-practices/)*

---

### 3. Image Optimization

**What Cal.com chose:**
- Next.js image optimization is **explicitly disabled**: `images: { unoptimized: true }`
- All images served as-is — no resizing, no WebP/AVIF conversion, no responsive sizing
- App store integration logos copied as static files without optimization

**Eco-first alternative:**
This is the easiest win with the biggest impact for a user-facing scheduling platform. The eco-first version would:

1. **Enable Next.js built-in image optimization** — remove `unoptimized: true`. Next.js automatically serves WebP, resizes to requested dimensions, and caches.
2. **Set `formats: ['image/avif', 'image/webp']`** in `next.config.ts` for maximum compression
3. **Use `next/image` with proper `sizes` prop** across all booking pages
4. **Optimize app store logos at build time** — these are known at build time, so pre-optimize them

**Impact:** WebP is 25-35% smaller than JPEG. AVIF is 40-50% smaller. For booking pages served to millions of visitors, this is massive.

*Reference: C4 — Google Developers Web Fundamentals; Web Almanac 2024*

---

### 4. Static Asset Delivery

**What Cal.com chose:**
- No `assetPrefix` configured — static assets served from origin
- When deployed on Vercel, Vercel's CDN handles edge delivery automatically
- Self-hosted deployments have **no CDN configuration** at all

**Eco-first alternative:**
The Vercel deployment gets CDN "for free," but self-hosters (a major use case for Cal.com) get nothing. The eco-first version would:

1. **Configure `assetPrefix`** to support CDN deployment out of the box
2. **Include a self-hosting guide with CDN setup** (Cloudflare free tier covers this)
3. **Set proper `Cache-Control` headers** for hashed static assets (`immutable, max-age=31536000`)

**Impact:** Reduces origin server load and long-haul network transit. CDN hit rates of 90%+ are common for static content.

*Reference: A2 — Industry consensus; Cloudflare documentation; C3 — Web Almanac 2024*

---

### 5. Dual-Region Deployment

**What Cal.com chose:**
- Dual-region deployment on Vercel (US + EU)
- Every cron job fires **twice** (once per region)
- Every deployment happens twice

**Eco-first alternative:**
Dual-region is legitimate for latency and compliance (GDPR). The eco-first version would:

1. **Run cron jobs from one region only** — webhook triggers don't need to fire from both US and EU. Route to the correct region based on the webhook target.
2. **Use Vercel's Edge Functions** for latency-sensitive paths (booking pages) but keep background processing in a single region.
3. **Choose the greener region as primary** — if US is `us-east-1` (323 gCO2/kWh), consider `us-west-2` (Oregon, 79 gCO2/kWh). If EU, `eu-north-1` (Stockholm, 3 gCO2/kWh) beats `eu-central-1` (Frankfurt, 276 gCO2/kWh).

**Impact:** Eliminating redundant cron execution halves the background compute. Choosing lower-carbon regions provides 2-10x carbon intensity reduction.

*Reference: I1 — Google Cloud Region Carbon Data, 2024*

---

### 6. CI/CD Pipeline

**What Cal.com chose:**
- **59 GitHub Actions workflows** — builds, tests, crons, releases, AI reviews, bundle analysis
- E2E tests run **8 shards of 4 workers** on 4-vCPU machines (32 vCPUs total per run)
- Path filtering exists in `pr.yml` (excludes `.md`, `docs/`, `.vscode/`) — this is good
- Turborepo caching enabled for builds — this is good
- 4 AI/Devin code review workflows add automated review overhead

**Eco-first alternative:**
Path filtering and Turborepo caching are already in place (credit where due). The eco-first improvements:

1. **Separate the 10 cron workflows** from CI entirely — move to a cloud scheduler, not GitHub Actions
2. **Cache E2E test shards** — if a shard's relevant code hasn't changed, skip it
3. **Gate AI review workflows** — run automated AI reviews only on substantive PRs (not typos or version bumps)
4. **Consolidate similar workflows** — 59 is excessive; many could be combined with conditional steps

**Impact:** 20-60% reduction in CI minutes. The 10 cron workflows alone are a significant steady-state cost.

*Reference: I4, I6 — GitHub Actions caching docs; GitLab CI rules documentation*

---

### 7. Data Retention

**What Cal.com chose:**
- Booking records retained **indefinitely** — no archival or deletion
- Users cannot delete past meeting history (GitHub Issue #18787)
- No TTL on sessions, audit logs, or webhook delivery records
- Monthly user downgrade cron exists, but no data cleanup cron

**Eco-first alternative:**
1. **Add a configurable booking retention period** — archive bookings older than X months to cold storage
2. **Auto-purge sessions and tokens** — expired sessions should be cleaned up on a schedule
3. **Webhook delivery logs** — keep for 30 days, then delete (they're only needed for debugging)
4. **Let users delete their own booking history** — both for privacy and storage

**Impact:** Less stored data = less storage energy. Keeps the active database lean, improving query performance and reducing backup sizes.

*Reference: P3 — AWS Well-Architected Sustainability Pillar*

---

## Summary: Eco-First Scheduling Architecture

If building an open-source scheduling platform from scratch with sustainability as a design constraint:

| Decision | Cal.com's Choice | Eco-First Choice | Difference |
|----------|-----------------|-----------------|------------|
| Background jobs | GitHub Actions cron (VM per invocation) | Event-driven queue (Bull/BullMQ) | Eliminates ~3,500 daily VM spin-ups |
| Docker image | 5 GB (node:20 full Debian) | <500 MB (node:20-alpine, optimized build) | 90% smaller image |
| Image optimization | Disabled (`unoptimized: true`) | Next.js built-in WebP/AVIF | 25-50% smaller images |
| Static assets (self-hosted) | Origin-only, no CDN config | CDN-ready with `assetPrefix` | 90%+ cache hit rate |
| Cron execution | Dual-region (fires twice) | Single-region for background work | 50% less cron compute |
| CI workflows | 59 workflows including 10 crons | Cloud scheduler for crons, consolidated CI | 20-60% less CI compute |
| Data retention | Indefinite, no deletion | Configurable TTLs, archival tiers | Bounded storage growth |
| Cloud regions | Vercel (AWS default regions) | Prefer low-carbon regions | 2-10x carbon reduction |

### What Cal.com Gets Right

These patterns should be adopted as-is:
- ✓ **Turborepo caching** — avoids redundant rebuilds across the monorepo
- ✓ **Path filtering in PR checks** — skips CI for docs and config-only changes
- ✓ **`optimizePackageImports`** for UI library tree-shaking
- ✓ **Multi-stage Docker build** — separates build dependencies from runtime
- ✓ **Webhook architecture** — event-driven notifications to external services (the concept is right, the implementation via cron is the issue)

---

*Generated by [eco-first](https://github.com/CNaught-Inc/eco-first) — sustainability-first design for Claude Code*
