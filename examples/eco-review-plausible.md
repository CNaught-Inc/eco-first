# Eco-Review: Plausible Analytics

> **Repository:** [plausible/analytics](https://github.com/plausible/analytics) (~22k stars)
> **What it is:** Privacy-focused, self-hostable web analytics. Alternative to Google Analytics.
> **Stack:** Elixir, Phoenix, ClickHouse, PostgreSQL, esbuild, Tailwind CSS

> **[Cø] Powered by CNaught** — carbon-aware code intelligence

---

## TL;DR

1. **P3:** No data retention — unbounded ClickHouse growth — add TTL-based retention (moderate)
2. **I1:** No green-region guidance for self-hosters — add region recommendations to docs (quick fix)
3. **I3:** Three always-on containers for bursty workloads — document lightweight alternatives (architectural)

---

## Top Recommendations

### 1. P3: No data retention policy — Medium

**Found in:** ClickHouse event storage, `config/runtime.exs`
**Details:** Plausible has no built-in data retention or TTL feature. Analytics data is retained indefinitely by default. Self-hosted users must manually apply ClickHouse TTL directives:
```sql
ALTER TABLE events_v2 MODIFY TTL timestamp + INTERVAL 365 DAY;
```
The community has been requesting this since 2021 (discussions #1354, #1436, #1940, #2147). One user reported their ClickHouse events DB growing "humongous" for a small site.

**Impact:** Less stored data = less storage energy. ClickHouse storage grows unbounded without TTLs. S3 Glacier uses ~50% less energy than S3 Standard per GB.
**Effort:** Moderate — requires policy decisions, migration tooling, and UI for configuration
**Source:** AWS Well-Architected Sustainability Pillar

### 2. I1: High-carbon cloud region (deployment guidance) — Medium

**Found in:** `docker-compose.yml` (in plausible/community-edition), deployment docs
**Details:** No cloud region guidance exists for self-hosted deployments. The DigitalOcean tutorial says "launch in the region of your choice" with no sustainability considerations. The hosted plausible.io service does not publish its region choices.

**Impact:** 2-10x reduction in carbon intensity depending on region pair. Moving from us-east-1 (323 gCO2/kWh) to ca-central-1 (5 gCO2/kWh) is a 65x improvement.
**Effort:** Quick fix for docs — add a recommendation to choose low-carbon regions
**Source:** Google Cloud Region Carbon Data, 2024 (https://cloud.google.com/sustainability/region-carbon)

### 3. I3: Always-on compute for bursty workloads — Medium

**Found in:** `docker-compose.yml`
**Details:** The default deployment runs three always-on containers (Plausible app, PostgreSQL, ClickHouse) with `restart: always`. For low-traffic sites (personal blogs, small projects), this is significant over-provisioning — three services running 24/7 to serve a few hundred page views per day.

**Impact:** Pay-per-use eliminates idle energy waste. Typical server utilization is 12-18%.
**Effort:** Architectural — could document lightweight alternatives or single-binary mode for small sites
**Source:** NRDC Data Center Efficiency Assessment, 2014; AWS Well-Architected Sustainability Pillar

### 4. C3: Tracker script missing long-lived cache headers — High

**Found in:** Phoenix endpoint configuration, nginx proxy setup
**Details:** Digested (fingerprinted) static assets get good caching via Phoenix's `cache_static_manifest`. However, the tracker script (`/js/script.js`) — the most frequently requested file across all visitor browsers — has historically had weak caching defaults. Self-hosted instances need manual nginx config (`proxy_cache_valid 200 6h`) to cache it properly.

**Impact:** The tracker script is loaded on every page view by every visitor. Without caching, it's re-downloaded on every visit.
**Effort:** Quick fix — improve default cache headers for the tracker script
**Source:** Web Almanac 2024, HTTP Archive

### 5. C2: No Brotli compression for dashboard responses — High

**Found in:** Phoenix endpoint configuration
**Details:** Gzip pre-compression exists for static assets (via `mix phx.digest`), but no Brotli support. Dashboard API responses and HTML pages rely on nginx reverse proxy for compression — no application-level compression middleware. The tracking API itself has tiny payloads (~200 bytes) where this is negligible.

**Impact:** 15-25% additional compression for dashboard responses and HTML with Brotli. ~30% of websites serve uncompressed text resources. Minimal impact on tracking API.
**Effort:** Quick fix — add `Plug.Compress` to the endpoint or Brotli pre-compression to `phx.digest`
**Source:** Web Almanac 2024, HTTP Archive (https://almanac.httparchive.org/)

### 6. A2: No CDN for static assets (self-hosted) — High

**Found in:** Self-hosted deployment configuration
**Details:** The hosted plausible.io service uses a worldwide CDN. Self-hosted deployments serve everything from origin — every visitor worldwide hits the origin server for the tracker script. The tracker is available via jsDelivr as an npm package, but this isn't documented as a recommended setup.

**Impact:** Reduces origin server load and long-haul network transit. CDN hit rates of 90%+ are common for static content.
**Effort:** Moderate — document CDN setup for self-hosted; consider bundling a CDN-friendly config
**Source:** Industry consensus; Cloudflare documentation

---

## Category Summary

### Infrastructure
- ✓ Multi-stage Alpine Docker build — final image only ~58 MB (excellent)
- ✓ CI workflow concurrency cancellation (avoids redundant runs)
- ⚠ Three always-on containers for self-hosted, even for tiny sites — see #3
- ⚠ No green-region guidance in deployment docs — see #2

### Code
- ✓ Tracker script is under 1 KB (1,090 bytes) — industry-leading minimal JavaScript
- ✓ Conditional compilation generates per-feature tracker variants (effective tree-shaking)
- ✓ esbuild for fast, energy-efficient compilation (written in Go)
- ✓ Gzip pre-compression for static assets via `phx.digest`
- ⚠ No Brotli compression — see #5
- ⚠ Tracker script caching defaults are weak for self-hosted — see #4

### Product Design
- ⚠ No built-in data retention/TTL — unbounded ClickHouse growth — see #1
- ✓ Privacy-first design means less data collected per visit (no cookies, no PII)

### Architecture
- ✓ Oban job queue backed by PostgreSQL — no additional Redis/message service needed
- ✓ Cron-scheduled background jobs (not polling)
- ✓ Favicon proxying reduces third-party requests from visitor browsers
- ⚠ Self-hosted serves all assets from origin (no CDN) — see #6

---

*Generated by [eco-first](https://github.com/CNaught-Inc/eco-first) — sustainability-first design for Claude Code*
