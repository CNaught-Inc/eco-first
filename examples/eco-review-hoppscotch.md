# Eco-Review: Hoppscotch

> **Repository:** [hoppscotch/hoppscotch](https://github.com/hoppscotch/hoppscotch) (~70k stars)
> **What it is:** Open-source API development platform. Test REST, GraphQL, WebSocket, SSE, MQTT APIs. Self-hostable with real-time collaboration.
> **Stack:** Vue 3, Vite, NestJS, GraphQL (Apollo), PostgreSQL, Redis, Caddy, Tauri (desktop)

> **[Cø] Powered by CNaught** — carbon-aware code intelligence

---

## TL;DR

1. **I6:** Full CI on every PR in a 10+ package monorepo — add path-based filtering (quick fix)
2. **I4:** No dependency caching in CI — add pnpm store caching (quick fix)
3. **A5:** Prisma ORM without eager loading audit — add `include` for nested data endpoints (moderate)

---

## Top Recommendations

### 1. I6: Redundant CI runs — High

**Found in:** `.github/workflows/`
**Details:** A 10+ package pnpm monorepo with no evidence of path-based workflow filtering. Every PR triggers full CI across all packages regardless of which one changed. A docs-only change rebuilds the backend, frontend, admin dashboard, CLI, and desktop app.

**Impact:** 20-60% reduction in CI minutes for monorepos (varies by change distribution)
**Effort:** Quick fix — add `paths:` filters to workflow triggers, or use `dorny/paths-filter` action
**Source:** Industry consensus; GitLab CI rules documentation

### 2. I4: No build caching in CI/CD — High

**Found in:** `.github/workflows/tests.yml`
**Details:** No evidence of `actions/cache` for the pnpm store or build artifacts. No incremental build tool (Turbo, Nx) configured. pnpm's content-addressable store helps locally, but without explicit CI caching, every run reinstalls and rebuilds from scratch.

**Impact:** 30-80% reduction in CI compute time (varies by project)
**Effort:** Quick fix — add pnpm store caching and consider Turborepo for incremental builds
**Source:** GitHub Actions caching docs; industry consensus

### 3. I2: Oversized container images (build resources) — Medium

**Found in:** `prod.Dockerfile`, self-hosting docs
**Details:** The build requires **4 CPU cores and 16 GB RAM** — extremely heavy for a web application. The AIO (all-in-one) Docker image is 211 MB compressed, which is reasonable, but the build resource requirement suggests the compilation pipeline is inefficient. Multi-stage build with Alpine is good practice, but the build itself is the bottleneck.

**Impact:** Reducing build memory from 16 GB to 4-8 GB would allow building on smaller CI runners and developer machines
**Effort:** Moderate — requires build pipeline optimization, potentially splitting builds
**Source:** Docker best practices (https://docs.docker.com/build/building/best-practices/)

### 4. A2: No CDN for static assets (self-hosted) — High

**Found in:** Self-hosted deployment configuration
**Details:** Self-hosted instances serve all static assets from the Caddy container with no CDN layer. No evidence of content-hashing for long-lived cache headers (though Caddy may handle this). The hosted hoppscotch.io likely uses a CDN, but self-hosters get origin-only delivery.

**Impact:** Reduces origin server load and long-haul network transit. CDN hit rates of 90%+ are common for static content.
**Effort:** Moderate — document CDN setup for self-hosters; configure Caddy cache headers for hashed assets
**Source:** Industry consensus; Cloudflare documentation

### 5. P3: No data retention policy — Medium

**Found in:** PostgreSQL data layer
**Details:** No evidence of automated data retention policies, TTL-based cleanup, or scheduled maintenance. API request history, collection data, and team activity persist indefinitely. Magic link tokens have a 24-hour expiry (good), but general data grows unbounded.

**Impact:** Less stored data = less storage energy. Bounded retention keeps the database lean.
**Effort:** Moderate — requires policy decisions and scheduled cleanup jobs
**Source:** AWS Well-Architected Sustainability Pillar

### 6. I5: Over-provisioned compute (AIO container) — Medium

**Found in:** `docker-compose.yml`, AIO container architecture
**Details:** The AIO container bundles the web app, API backend, admin dashboard, and Caddy reverse proxy in a single process. The admin dashboard consumes resources even if rarely accessed. Redis runs as a separate always-on service even when no GraphQL subscriptions are active.

**Impact:** Splitting services lets unused components (admin, Redis) scale independently or shut down when idle
**Effort:** Moderate — the split-mode compose already exists; make it the documented default
**Source:** AWS Well-Architected Sustainability Pillar

### 7. A5: N+1 database queries — High

**Found in:** Prisma ORM usage in NestJS backend
**Details:** Hoppscotch uses Prisma ORM with PostgreSQL. Prisma does not support automatic eager loading — related records (team members, collections within a team, environments per workspace) require explicit `include` statements. Without careful attention, listing endpoints that return nested data (teams with members, collections with requests) produce N+1 query patterns.

**Impact:** 80% faster at 1000 records with eager loading vs N+1. Reduces database round-trips from N+1 to 2 for nested data.
**Effort:** Quick fix to Moderate — add `include` clauses to Prisma queries for endpoints that return related data
**Source:** PlanetScale blog (https://planetscale.com/); Azure Well-Architected Framework (https://learn.microsoft.com/en-us/azure/well-architected/)

---

## Category Summary

### Infrastructure
- ✓ Multi-stage Docker build with Alpine base
- ✓ Caddy provides automatic HTTPS (reduces manual config overhead)
- ✓ pnpm workspaces for efficient dependency deduplication
- ⚠ No CI path filtering in a 10+ package monorepo — see #1
- ⚠ No CI build caching evidence — see #2
- ⚠ 16 GB RAM build requirement — see #3

### Code
- ✓ WebSocket-based GraphQL subscriptions (not polling) for real-time features
- ✓ Vite build tool (faster, less energy than Webpack)
- ✓ Tailwind CSS with PurgeCSS (strips unused styles)
- ✓ PWA with full offline support (eliminates repeated server loads after first visit)
- No issues found: compression and caching are handled by Caddy (likely adequate)

### Product Design
- ✓ Full offline PWA — zero server energy for returning users
- ⚠ No data retention policies for stored API history — see #5
- ⚠ Redis always-on even when no subscriptions active — see #6

### Architecture
- ✓ NestJS + GraphQL reduces over-fetching vs REST
- ✓ Redis Pub/Sub for real-time (efficient pattern)
- ⚠ Self-hosted has no CDN layer — see #4
- ⚠ AIO container bundles unused services — see #6
- ⚠ Prisma ORM queries may produce N+1 patterns for nested data — see #7
- No issues found: Tauri desktop app (Rust) is lightweight compared to Electron

---

*Generated by [eco-first](https://github.com/CNaught-Inc/eco-first) — sustainability-first design for Claude Code*
