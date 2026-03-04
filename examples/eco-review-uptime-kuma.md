# Eco-Review: Uptime Kuma

> **Repository:** [louislam/uptime-kuma](https://github.com/louislam/uptime-kuma) (~60k stars)
> **What it is:** Self-hosted monitoring tool. Checks HTTP, TCP, ping, DNS endpoints at configurable intervals and alerts on downtime.
> **Stack:** Node.js, Express, Vue 3, Socket.IO, SQLite, Vite

---

## Top Recommendations

### 1. I2: Oversized container images

**Found in:** `docker/debian-base.dockerfile`
**Details:** Default image uses `node:22-bookworm-slim` base and bundles Chromium (~200MB) and MariaDB server into the standard tag. Total image size: ~555 MB. A `-slim` tag exists without these extras but is not the default.

**Impact:** 50-90% reduction in image size by using the slim variant. The standard image ships ~500 MB of extras (Chromium, MariaDB) that most users don't need.
**Effort:** Quick fix — document slim as the recommended default; consider making it the default tag
**Source:** Docker best practices (https://docs.docker.com/build/building/best-practices/)

### 2. P3: No data retention policy (with caveats)

**Found in:** `src/components/settings/MonitorHistory.vue`, server-side database code
**Details:** Uptime Kuma *does* have a configurable `keepDataPeriodDays` setting (default: 180 days), which is better than most. However, at 60-second intervals, each monitor generates ~259,200 heartbeat records per 180 days. There is no automatic down-sampling (e.g., keep per-minute for 7 days, then aggregate to hourly). The SQLite `VACUUM` to reclaim space must be triggered manually and causes 5-10 minutes of monitoring downtime.

**Impact:** Down-sampling old data could reduce storage by 90%+ for data older than a week while preserving trend visibility.
**Effort:** Moderate — requires data aggregation logic and migration
**Source:** AWS Well-Architected Sustainability Pillar

### 3. C2: Uncompressed API payloads (partial)

**Found in:** `server/server.js`, `package.json`
**Details:** The `compression` npm package is installed and used as Express middleware, providing gzip compression. However, Brotli compression is not configured. Brotli achieves 15-25% better compression than gzip for text-based responses.

**Impact:** 15-25% additional payload size reduction for dashboard API responses and static assets
**Effort:** Quick fix — add `shrink-ray-current` or configure Vite to emit pre-compressed Brotli assets
**Source:** Web Almanac 2024, HTTP Archive (https://almanac.httparchive.org/)

### 4. C3: Missing cache-control headers (unconfirmed)

**Found in:** `server/server.js` (Express static middleware)
**Details:** Vite produces content-hashed filenames (e.g., `app-abc123.js`), which are ideal for immutable caching. However, the `express.static()` configuration could not be confirmed to set `max-age` or `immutable` headers. If missing, browsers revalidate every request — defeating the purpose of content hashing.

**Impact:** Eliminates redundant data transfer for repeat visitors. Static assets are typically 60-80% of page weight.
**Effort:** Quick fix — add `{ maxAge: '1y', immutable: true }` to `express.static()` options for hashed assets
**Source:** Web Almanac 2024, HTTP Archive

### 5. I5: Over-provisioned compute (deployment guidance)

**Found in:** `compose.yaml`
**Details:** The default `compose.yaml` specifies no CPU or memory limits (`mem_limit`, `cpus`). The standard image includes Chromium, which can consume significant memory. No documentation guides users on right-sizing based on monitor count.

**Impact:** Linear reduction in energy — setting appropriate resource limits prevents the container from consuming more than needed
**Effort:** Quick fix — add sensible `deploy.resources.limits` to compose.yaml and document sizing guidance
**Source:** AWS Well-Architected Sustainability Pillar

### 6. A3: Monolithic scheduled jobs (monitor batching)

**Found in:** `server/model/monitor.js`
**Details:** Each monitor runs its own independent `setTimeout` loop. With 50 monitors checking the same host at different intervals, there is no coalescing or batching. While `setTimeout` (vs `setInterval`) is the correct choice to prevent overlap, the per-monitor architecture means N monitors = N independent timers.

**Impact:** Batching monitors targeting the same host could reduce DNS lookups and connection overhead. Most impactful at scale (50+ monitors).
**Effort:** Architectural — requires a scheduler that groups monitors by target
**Source:** GSF Green Software Patterns

---

## Category Summary

### Infrastructure
- ✓ Multi-stage Docker build with separate `base2-slim` and `base2` stages
- ✓ Uses `setTimeout` over `setInterval` for monitor scheduling (prevents overlap)
- ✓ Slim Docker variant available (`2-slim` tag)
- ⚠ Default Docker image is 555 MB with bundled Chromium and MariaDB — see #1
- ⚠ No resource limits in default compose.yaml — see #5

### Code
- ✓ Socket.IO for real-time dashboard updates (avoids client polling)
- ✓ Gzip compression enabled via `compression` middleware
- ⚠ No Brotli compression — see #3
- ⚠ Cache-control headers for static assets unconfirmed — see #4

### Product Design
- ✓ Configurable data retention period (`keepDataPeriodDays`)
- ⚠ No automatic data down-sampling for old heartbeat records — see #2
- ⚠ Manual-only database vacuum with monitoring downtime

### Architecture
- ✓ WebSocket-first architecture eliminates frontend polling
- ✓ SQLite eliminates need for separate database server
- ⚠ Per-monitor independent timers with no batching — see #6
- No issues found: CDN is not applicable (self-hosted dashboard tool)

---

*Generated by [eco-first](https://github.com/CNaught-Inc/eco-first) — sustainability-first design for Claude Code*
