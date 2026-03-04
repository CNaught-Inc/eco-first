---
name: eco-review
description: Use when reviewing code for environmental impact, auditing sustainability of infrastructure choices, or when user runs /eco-review. Scans for high-carbon cloud regions, inefficient API patterns, oversized containers, missing caching, and wasteful defaults.
---

# eco-review

Scan a project's infrastructure, code, and product design for sustainability issues. Matches findings against 27 known anti-patterns across infrastructure, code, product design, and architecture. Produces a ranked report with impact estimates, effort levels, and cited sources.

## Audit Process

### Step 1: Scan the Project

Search the codebase for:

- **Infrastructure files:** Terraform (`.tf`), CloudFormation (`template.yaml`/`.json`), Pulumi, Dockerfiles, docker-compose, CI configs (`.github/workflows/`, `.gitlab-ci.yml`, `Jenkinsfile`)
- **Code patterns:** API endpoints, payload handling, compression middleware, caching headers, image tags, frontend bundle configs (webpack, vite, rollup), font files (`@font-face`, Google Fonts URLs), video/GIF tags, third-party script includes
- **Database patterns:** ORM configuration (Prisma, Django ORM, ActiveRecord, TypeORM, Sequelize), query patterns, logging configuration
- **Product design:** Default settings, notification systems, content loading patterns, data retention configs, background feature toggles

### Step 2: Match Against Patterns

Compare findings against each pattern in the pattern set below. Record every match with the specific file and line where the anti-pattern appears.

### Step 3: Look Up Cloud Regions

If cloud region configuration is found, look up the region in the Carbon Intensity Reference Table below. Note the grid carbon intensity and identify lower-carbon alternatives in the same provider.

### Step 4: Produce the Report

Generate the output report using the format specified below.

---

## Output Format

```
## Eco-Review: [Project Name]

> **[Cø] Powered by CNaught** — carbon-aware code intelligence

### Top Recommendations

1. **[Pattern ID]: [Description]**
   Found in: `[file:line]`
   Impact: [range from pattern]
   Effort: [quick fix / moderate / architectural]
   Technical risk: [from pattern definition]
   Usability risk: [from pattern definition]
   Source: [verbatim citation from pattern]

2. ...

### Category Summary

#### Infrastructure
- ✓ [Good practice found]
- ⚠ [Recommendation — see #N above]

#### Code
- ✓ [Good practice found]
- ⚠ [Recommendation — see #N above]

#### Product Design
- ✓ [Good practice found]
- ⚠ [Recommendation — see #N above]

#### Architecture
- ✓ [Good practice found]
- ⚠ [Recommendation — see #N above]

---
*[Cø] Powered by [CNaught](https://www.cnaught.com) — carbon-aware code intelligence*
```

---

## Pattern Set: Infrastructure

### I1: High-carbon cloud region

- **Anti-pattern:** Deploying to a high-carbon-intensity region without considering alternatives
- **Recommendation:** Migrate to a lower-carbon region in the same provider. See carbon intensity table.
- **Impact:** 2-10x reduction in carbon intensity depending on region pair
- **Effort:** Architectural — requires migration plan, latency testing, compliance review
- **Technical risk:** Data residency/compliance violations, migration downtime, cross-region replication complexity
- **Usability risk:** Higher latency for users near original region
- **Source:** Google Cloud Region Carbon Data, 2024 (https://cloud.google.com/sustainability/region-carbon)

### I2: Oversized container images

- **Anti-pattern:** Docker images using full OS base (ubuntu, debian) or including build tools in production
- **Recommendation:** Use alpine or distroless base images. Multi-stage builds to exclude build deps.
- **Impact:** 50-90% reduction in image size, reducing transfer, storage, and cold start energy
- **Effort:** Moderate — Dockerfile refactor, test for compatibility
- **Technical risk:** Missing system libraries in minimal images, debugging difficulty without shell tools
- **Usability risk:** None
- **Source:** Docker best practices (https://docs.docker.com/build/building/best-practices/)

### I3: Always-on compute for bursty workloads

- **Anti-pattern:** Running 24/7 instances for workloads with variable or low utilization
- **Recommendation:** Serverless (Lambda, Cloud Functions) or spot/preemptible instances for fault-tolerant work
- **Impact:** Pay-per-use eliminates idle energy waste. Typical server utilization is 12-18%.
- **Effort:** Architectural — requires workload analysis, may need code changes
- **Technical risk:** Cold start latency (100ms–10s), state management complexity, execution time limits
- **Usability risk:** Noticeable delay on first request after idle period
- **Source:** NRDC Data Center Efficiency Assessment, 2014; AWS Well-Architected Sustainability Pillar

### I4: No build caching in CI/CD

- **Anti-pattern:** CI pipeline rebuilds everything from scratch on every run
- **Recommendation:** Cache dependencies, Docker layers, build artifacts between runs
- **Impact:** 30-80% reduction in CI compute time (varies by project)
- **Effort:** Quick fix — CI config change
- **Technical risk:** Stale cache causing flaky builds, cache invalidation bugs
- **Usability risk:** None
- **Source:** GitHub Actions caching docs; industry consensus

### I5: Over-provisioned compute

- **Anti-pattern:** Instance types much larger than workload requires
- **Recommendation:** Right-size based on actual utilization metrics. Start small, scale up.
- **Impact:** Linear reduction in energy — half the cores = roughly half the power draw
- **Effort:** Moderate — requires monitoring data to right-size accurately
- **Technical risk:** Under-provisioning causes performance degradation under load spikes
- **Usability risk:** Slower response times during peak usage if sized too aggressively
- **Source:** AWS Well-Architected Sustainability Pillar

### I6: Redundant CI runs

- **Anti-pattern:** Running full test suite on every commit regardless of what changed
- **Recommendation:** Path-based filtering, skip CI for docs-only changes, parallelise test shards
- **Impact:** 20-60% reduction in CI minutes for monorepos (varies by change distribution)
- **Effort:** Quick fix — CI config change
- **Technical risk:** Path filters may miss cross-cutting changes, allowing broken code through
- **Usability risk:** None
- **Source:** Industry consensus; GitLab CI rules documentation

### I7: x86 instances when ARM is viable

- **Anti-pattern:** Deploying on x86 (Intel/AMD) instances when the workload runs on ARM without modification
- **Recommendation:** Use ARM-based instances (AWS Graviton, Azure Cobalt, GCP Axion) for compatible workloads. Check IaC for instance type families (e.g., `m6i` → `m7g` on AWS).
- **Impact:** Up to 60% less energy per core. 20-40% cost savings on equivalent instance sizes.
- **Effort:** Moderate — requires testing application and dependencies on ARM, updating IaC
- **Technical risk:** Native binary dependencies (e.g., compiled C extensions, some ML libraries) may lack ARM builds. Container images must be multi-arch or ARM-specific.
- **Usability risk:** None
- **Source:** ARM Newsroom (https://newsroom.arm.com/); AWS Graviton documentation (https://aws.amazon.com/ec2/graviton/)

---

## Pattern Set: Code

### C1: Polling instead of event-driven

- **Anti-pattern:** Clients polling an endpoint at fixed intervals for updates
- **Recommendation:** Server-Sent Events (SSE), WebSockets, or webhooks
- **Impact:** Eliminates redundant requests. A 5-second poll interval = 720 wasted requests/hour per client when nothing changes.
- **Effort:** Moderate — per-endpoint refactor
- **Technical risk:** WebSocket connection management, reconnection logic, firewall/proxy issues
- **Usability risk:** Missed updates if push connection drops silently
- **Source:** GSF Green Software Patterns (https://patterns.greensoftware.foundation/)

### C2: Uncompressed text resources

- **Anti-pattern:** Text resources (HTML, CSS, JS, JSON, API responses) served without compression
- **Recommendation:** Enable gzip or brotli compression at the server/CDN level for all text-based responses
- **Impact:** 60-80% reduction in payload size for text-based responses. ~30% of websites serve uncompressed text resources, wasting ~500KB per median page load.
- **Effort:** Quick fix — server/middleware configuration
- **Technical risk:** CPU overhead for compression on high-throughput endpoints, incompatibility with streaming responses
- **Usability risk:** None
- **Source:** Web Almanac 2024, HTTP Archive (https://almanac.httparchive.org/)

### C3: Missing cache-control headers

- **Anti-pattern:** Static assets served without cache-control headers
- **Recommendation:** Set appropriate Cache-Control headers (immutable for hashed assets, max-age for others)
- **Impact:** Eliminates redundant data transfer for repeat visitors. Static assets are typically 60-80% of page weight.
- **Effort:** Quick fix — server/CDN configuration
- **Technical risk:** Stale content served after deploys if cache-busting strategy is wrong
- **Usability risk:** Users see outdated content until cache expires
- **Source:** Web Almanac 2024, HTTP Archive

### C4: Unoptimized images

- **Anti-pattern:** Serving PNG/JPEG when modern formats available, no responsive sizing, no lazy loading
- **Recommendation:** WebP/AVIF format, srcset for responsive sizes, loading="lazy" for below-fold images
- **Impact:** WebP is 25-35% smaller than JPEG at equivalent quality. AVIF is 40-50% smaller.
- **Effort:** Moderate — build pipeline and markup changes
- **Technical risk:** Older browser incompatibility with WebP/AVIF, build pipeline complexity for multiple formats
- **Usability risk:** Quality degradation if compression too aggressive, layout shift from lazy-loaded images
- **Source:** Google Developers Web Fundamentals; Web Almanac 2024

### C5: Large frontend bundles

- **Anti-pattern:** Shipping entire libraries when only parts are used, no code splitting
- **Recommendation:** Tree-shaking, code splitting, dynamic imports for route-based loading
- **Impact:** Varies widely. Common savings: 30-60% bundle size reduction from tree-shaking alone.
- **Effort:** Moderate — build config and import changes
- **Technical risk:** Code splitting adds loading states, race conditions with shared dependencies
- **Usability risk:** Visible spinners/flashes for lazy-loaded chunks on slow connections
- **Source:** webpack/Vite documentation; Web Almanac 2024

### C6: Verbose serialization at scale

- **Anti-pattern:** Using JSON for high-volume internal service communication
- **Recommendation:** Consider Protocol Buffers, MessagePack, or similar compact formats for internal APIs
- **Impact:** 30-50% smaller payloads vs JSON. Most impactful for high-throughput internal services.
- **Effort:** Moderate to Architectural — requires schema definition, client/server changes
- **Technical risk:** Schema versioning complexity, harder debugging (binary payloads not human-readable)
- **Usability risk:** None (internal service change, not user-facing)
- **Source:** Google Developers Protocol Buffers documentation

### C7: Dead code in frontend bundles

- **Anti-pattern:** Shipping JavaScript that is never executed by users. Unused exports, abandoned feature code, and over-inclusive library imports remain in production bundles.
- **Recommendation:** Audit bundles with coverage tools (Chrome DevTools Coverage, `webpack-bundle-analyzer`). Remove unused exports, replace large libraries with lighter alternatives, use tree-shaking-friendly ESM imports.
- **Impact:** 206KB median wasted JavaScript per mobile page load (44% of JS bytes unused). Reduces parse time, execution energy, and data transfer.
- **Effort:** Moderate — requires bundle analysis, dependency audit, and testing after removal
- **Technical risk:** Removing code that appears unused but is dynamically referenced (e.g., via string-based imports or reflection). Side-effectful modules may break if tree-shaken.
- **Usability risk:** None if properly tested
- **Source:** Web Almanac 2024, HTTP Archive (https://almanac.httparchive.org/)

### C8: Excessive third-party scripts

- **Anti-pattern:** Loading numerous third-party scripts (analytics, ads, trackers, social widgets) that bloat page weight, block rendering, and consume device energy
- **Recommendation:** Audit third-party scripts with `navigator.connection` or performance budgets. Remove unused trackers, consolidate analytics, defer non-critical scripts, use facade patterns for heavy embeds (e.g., YouTube, chat widgets).
- **Impact:** 94% of mobile pages load third-party resources; top sites average 66 third-party domains. On media sites, up to 70% of page electricity is consumed by ads and trackers.
- **Effort:** Moderate — requires stakeholder buy-in to remove tracking scripts, may need consent management updates
- **Technical risk:** Removing analytics/tracking may break business reporting or A/B testing. Facade patterns require loading the real resource on interaction.
- **Usability risk:** Removing social widgets or live chat reduces convenience for some users
- **Source:** Web Almanac 2024, HTTP Archive (https://almanac.httparchive.org/); The Shift Project (https://theshiftproject.org/)

### C9: Unsubsetted web fonts

- **Anti-pattern:** Loading full font files (containing thousands of glyphs for all languages) when the page only uses a subset of characters
- **Recommendation:** Subset fonts to include only required character ranges (e.g., Latin). Use `unicode-range` in `@font-face` declarations. For Google Fonts, use the `&text=` or `&subset=` parameter. Self-host subsetted files for best control.
- **Impact:** 88-99% file size reduction from subsetting. 83% of sites use custom fonts, making this broadly applicable.
- **Effort:** Quick fix to Moderate — subsetting tools are straightforward, but self-hosting adds build step
- **Technical risk:** Missing characters for user-generated content in unexpected languages. Font licensing may restrict subsetting.
- **Usability risk:** Missing glyphs display as boxes/fallback font for unsupported characters
- **Source:** Paul Calvano, Web Font optimization 2024; Web Almanac 2024, HTTP Archive (https://almanac.httparchive.org/)

### C10: Unoptimized video delivery

- **Anti-pattern:** Serving uncompressed video, using GIF for animations, autoplaying video without user intent, or missing `preload` attributes
- **Recommendation:** Replace GIF with MP4/WebM (94-95% size reduction). Use `preload="none"` or `preload="metadata"` for non-hero videos. Serve adaptive bitrate (HLS/DASH) for longer content. Add `loading="lazy"` for below-fold video.
- **Impact:** GIF→MP4 conversion yields 94-95% file size reduction. 55% of videos lack preload attributes, causing unnecessary data transfer.
- **Effort:** Quick fix to Architectural — replacing GIFs is simple; adaptive bitrate requires transcoding pipeline
- **Technical risk:** MP4/WebM browser compatibility (universal for modern browsers). Adaptive bitrate adds infrastructure complexity.
- **Usability risk:** Replacing autoplay with click-to-play changes user experience. Lower bitrate may reduce perceived quality.
- **Source:** Web Almanac 2024, HTTP Archive (https://almanac.httparchive.org/); The Shift Project (https://theshiftproject.org/)

---

## Pattern Set: Product Design

### P1: Real-time-only sync

- **Anti-pattern:** All updates pushed in real-time with no option for batched delivery
- **Recommendation:** Default to batched/digest mode for non-urgent content. Let users opt into real-time.
- **Impact:** Reduces server push frequency and client device wake-ups. Impact scales with user count.
- **Effort:** Moderate — requires feature design and UI for preference
- **Technical risk:** Batch scheduling complexity, ensuring urgent items still deliver promptly
- **Usability risk:** Users miss time-sensitive updates when defaulted to digest mode
- **Source:** GSF Green Software Patterns

### P2: Infinite scroll without limits

- **Anti-pattern:** Loading content endlessly as user scrolls, with no pagination or virtualization
- **Recommendation:** Paginate, or use virtual scrolling to render only visible items
- **Impact:** Prevents speculative loading of content users never see. Typical user views <20% of infinite-scrolled content.
- **Effort:** Moderate — UI refactor
- **Technical risk:** Pagination state management, virtual scrolling library complexity
- **Usability risk:** Extra clicks to navigate pages, less fluid browsing experience
- **Source:** Industry consensus; Nielsen Norman Group research on scrolling behavior

### P3: No data retention policy

- **Anti-pattern:** Storing all data indefinitely with no archival or deletion strategy
- **Recommendation:** Define TTLs for transient data, archive cold data to cheaper/lower-energy tiers
- **Impact:** Less stored data = less storage energy. S3 Glacier uses ~50% less energy than S3 Standard per GB.
- **Effort:** Moderate — requires policy decisions and migration
- **Technical risk:** Data migration complexity, backup coordination, accidental deletion of needed data
- **Usability risk:** Users lose data they expected to keep, compliance conflicts with retention requirements
- **Source:** AWS Well-Architected Sustainability Pillar

### P4: Always-on background features

- **Anti-pattern:** Background sync, location tracking, or analytics running continuously with no user control
- **Recommendation:** Let users disable non-essential background activity. Default to off for battery-intensive features.
- **Impact:** Device-level energy savings. Background activity is a top battery drain on mobile.
- **Effort:** Moderate — feature flag and settings UI
- **Technical risk:** Feature flag complexity, ensuring critical background tasks (e.g., data sync) still run
- **Usability risk:** Users forget to re-enable features, miss functionality they relied on unconsciously
- **Source:** Android Developers battery optimization guides; Apple Energy Efficiency Guide

---

## Pattern Set: Architecture

### A1: Synchronous processing for non-urgent work

- **Anti-pattern:** Processing all tasks synchronously/in-request when some could be deferred
- **Recommendation:** Use async job queues (SQS, Bull, Celery) for non-urgent processing
- **Impact:** Enables batching (fewer cold starts, better utilization), smooths load peaks
- **Effort:** Architectural — requires queue infrastructure and worker setup
- **Technical risk:** Queue monitoring complexity, dead letter handling, eventual consistency issues
- **Usability risk:** Delayed feedback — users don't see results immediately for deferred tasks
- **Source:** AWS Well-Architected Sustainability Pillar

### A2: No CDN for static assets

- **Anti-pattern:** Serving static files directly from origin server
- **Recommendation:** Use a CDN (CloudFront, Cloudflare, Fastly) to serve from edge locations
- **Impact:** Reduces origin server load and long-haul network transit. CDN hit rates of 90%+ are common for static content.
- **Effort:** Quick fix to Moderate — depends on existing infra
- **Technical risk:** Cache invalidation on deploy, CDN configuration drift, origin failover complexity
- **Usability risk:** None
- **Source:** Industry consensus; Cloudflare documentation

### A3: Monolithic scheduled jobs

- **Anti-pattern:** Large cron jobs that run on fixed schedules regardless of whether there's work
- **Recommendation:** Event-driven triggers — process when data arrives, not on a timer
- **Impact:** Eliminates wasted runs. A cron job that finds no work 80% of the time wastes 80% of its compute.
- **Effort:** Architectural — requires event source and trigger infrastructure
- **Technical risk:** Event ordering guarantees, idempotency requirements, harder to test than cron
- **Usability risk:** None (backend processing change, not user-facing)
- **Source:** GSF Green Software Patterns

### A4: Redundant data storage

- **Anti-pattern:** Same data stored in multiple locations without deduplication strategy
- **Recommendation:** Single source of truth with caching/derived views. Deduplicate where possible.
- **Impact:** Linear reduction in storage energy with deduplication ratio
- **Effort:** Architectural — requires data architecture review
- **Technical risk:** Single point of failure, cache coherence complexity, migration risk for existing consumers
- **Usability risk:** None (data layer change, not user-facing)
- **Source:** Industry consensus

### A5: N+1 database queries

- **Anti-pattern:** ORM code that issues one query per related record instead of batch-loading (e.g., looping over results and lazy-loading associations)
- **Recommendation:** Use eager loading (`include`, `prefetch_related`, `joinedload`, `with`) to batch-fetch related records in a single query. Enable ORM query logging in development to detect N+1 patterns.
- **Impact:** 80% faster at 1000 records with eager loading vs N+1. Reduces database round-trips from N+1 to 2, cutting network and CPU overhead proportionally.
- **Effort:** Quick fix to Moderate — adding eager loading is simple per-query, but auditing an entire codebase takes time
- **Technical risk:** Over-eager loading can fetch more data than needed, increasing memory usage. Complex eager loading can generate slow JOINs.
- **Usability risk:** None
- **Source:** PlanetScale blog (https://planetscale.com/); Azure Well-Architected Framework (https://learn.microsoft.com/en-us/azure/well-architected/)

### A6: Verbose logging in production

- **Anti-pattern:** Running DEBUG or TRACE log levels in production, logging full request/response bodies, or retaining high-volume logs indefinitely
- **Recommendation:** Set production log level to WARN or INFO. Avoid logging full payloads — use structured logging with relevant fields only. Set log retention policies (7-30 days for verbose logs, longer for errors).
- **Impact:** Debug logging produces 10-100x the volume of info-level logging. 30-50% reduction in observability infrastructure costs from right-sizing log levels and retention.
- **Effort:** Quick fix — configuration change for log levels; moderate for retention policies
- **Technical risk:** Reducing log verbosity may make production debugging harder for intermittent issues. Retention policies may discard logs needed for incident review.
- **Usability risk:** None
- **Source:** ClickHouse blog (https://clickhouse.com/blog); Logz.io (https://logz.io/)

---

## Carbon Intensity by Cloud Region (gCO2eq/kWh, annual average)

### Lowest Carbon (Green Zones — prefer these)

| Location | Grid gCO2/kWh | AWS | GCP | Azure |
|----------|--------------|-----|-----|-------|
| Stockholm, Sweden | 3 | eu-north-1 | europe-north2 | Sweden Central |
| Montréal, Canada | 5 | ca-central-1 | northam-northeast1 | Canada Central |
| Zürich, Switzerland | 15 | — | europe-west6 | Switzerland North |
| Paris, France | 16 | eu-west-3 | europe-west9 | France Central |
| Finland | 39 | — | europe-north1 | — |
| Toronto, Canada | 59 | — | northam-northeast2 | Canada East |
| São Paulo, Brazil | 67 | sa-east-1 | southam-east1 | Brazil South |
| Oregon, US | 79 | us-west-2 | us-west1 | West US 2 |

### Moderate Carbon

| Location | Grid gCO2/kWh | AWS | GCP | Azure |
|----------|--------------|-----|-----|-------|
| Madrid, Spain | 89 | — | europe-southwest1 | — |
| Belgium | 103 | — | europe-west1 | — |
| London, UK | 106 | eu-west-2 | europe-west2 | UK South |
| Los Angeles, US | 169 | — | us-west2 | West US |
| Ireland | 279 | eu-west-1 | — | North Europe |
| Frankfurt, Germany | 276 | eu-central-1 | europe-west3 | Germany West Central |

### High Carbon (Consider alternatives)

| Location | Grid gCO2/kWh | AWS | GCP | Azure |
|----------|--------------|-----|-----|-------|
| N. Virginia, US | 323 | us-east-1 | us-east4 | East US |
| Iowa, US | 413 | — | us-central1 | — |
| Tokyo, Japan | 453 | ap-northeast-1 | asia-northeast1 | Japan East |
| Sydney, Australia | 498 | ap-southeast-2 | australia-southeast1 | Australia East |
| Hong Kong | 505 | — | asia-east2 | — |
| S. Carolina, US | 576 | — | us-east1 | — |
| Mumbai, India | 679 | ap-south-1 | asia-south1 | Central India |

Source: Google Cloud Region Carbon Data, 2024
(https://cloud.google.com/sustainability/region-carbon)
Grid intensity = local grid average, not provider-specific.
Providers may purchase renewable energy credits — this table reflects grid reality.

---

## Ranking and Output Rules

- Always sort recommendations by estimated impact (highest first)
- Use ranges, never point estimates
- Reproduce citations verbatim from pattern definitions
- Include effort annotation on every recommendation
- Include both risk fields on every recommendation. If a risk category genuinely doesn't apply, write "None" — don't omit the field.
- Exclude device-dependent items (dark mode, etc.) from default output
- If no issues found in a category, say so briefly — don't pad the output
