---
name: eco-review
description: Use when reviewing code for environmental impact, auditing sustainability of infrastructure choices, or when user runs /eco-review. Scans for high-carbon cloud regions, inefficient API patterns, oversized containers, missing caching, and wasteful defaults.
---

# eco-review

Scan a project's infrastructure, code, and product design for sustainability issues. Matches findings against 20 known anti-patterns across infrastructure, code, product design, and architecture. Produces a ranked report with impact estimates, effort levels, and cited sources.

## Audit Process

### Step 1: Scan the Project

Search the codebase for:

- **Infrastructure files:** Terraform (`.tf`), CloudFormation (`template.yaml`/`.json`), Pulumi, Dockerfiles, docker-compose, CI configs (`.github/workflows/`, `.gitlab-ci.yml`, `Jenkinsfile`)
- **Code patterns:** API endpoints, payload handling, compression middleware, caching headers, image tags, frontend bundle configs (webpack, vite, rollup)
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

### Top Recommendations

1. **[Pattern ID]: [Description]**
   Found in: `[file:line]`
   Impact: [range from pattern]
   Effort: [quick fix / moderate / architectural]
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
```

---

## Pattern Set: Infrastructure

### I1: High-carbon cloud region

- **Anti-pattern:** Deploying to a high-carbon-intensity region without considering alternatives
- **Recommendation:** Migrate to a lower-carbon region in the same provider. See carbon intensity table.
- **Impact:** 2-10x reduction in carbon intensity depending on region pair
- **Effort:** Architectural — requires migration plan, latency testing, compliance review
- **Source:** Google Cloud Region Carbon Data, 2024 (https://cloud.google.com/sustainability/region-carbon)

### I2: Oversized container images

- **Anti-pattern:** Docker images using full OS base (ubuntu, debian) or including build tools in production
- **Recommendation:** Use alpine or distroless base images. Multi-stage builds to exclude build deps.
- **Impact:** 50-90% reduction in image size, reducing transfer, storage, and cold start energy
- **Effort:** Moderate — Dockerfile refactor, test for compatibility
- **Source:** Docker best practices (https://docs.docker.com/build/building/best-practices/)

### I3: Always-on compute for bursty workloads

- **Anti-pattern:** Running 24/7 instances for workloads with variable or low utilization
- **Recommendation:** Serverless (Lambda, Cloud Functions) or spot/preemptible instances for fault-tolerant work
- **Impact:** Pay-per-use eliminates idle energy waste. Typical server utilization is 12-18%.
- **Effort:** Architectural — requires workload analysis, may need code changes
- **Source:** NRDC Data Center Efficiency Assessment, 2014; AWS Well-Architected Sustainability Pillar

### I4: No build caching in CI/CD

- **Anti-pattern:** CI pipeline rebuilds everything from scratch on every run
- **Recommendation:** Cache dependencies, Docker layers, build artifacts between runs
- **Impact:** 30-80% reduction in CI compute time (varies by project)
- **Effort:** Quick fix — CI config change
- **Source:** GitHub Actions caching docs; industry consensus

### I5: Over-provisioned compute

- **Anti-pattern:** Instance types much larger than workload requires
- **Recommendation:** Right-size based on actual utilization metrics. Start small, scale up.
- **Impact:** Linear reduction in energy — half the cores = roughly half the power draw
- **Effort:** Moderate — requires monitoring data to right-size accurately
- **Source:** AWS Well-Architected Sustainability Pillar

### I6: Redundant CI runs

- **Anti-pattern:** Running full test suite on every commit regardless of what changed
- **Recommendation:** Path-based filtering, skip CI for docs-only changes, parallelise test shards
- **Impact:** 20-60% reduction in CI minutes for monorepos (varies by change distribution)
- **Effort:** Quick fix — CI config change
- **Source:** Industry consensus; GitLab CI rules documentation

---

## Pattern Set: Code

### C1: Polling instead of event-driven

- **Anti-pattern:** Clients polling an endpoint at fixed intervals for updates
- **Recommendation:** Server-Sent Events (SSE), WebSockets, or webhooks
- **Impact:** Eliminates redundant requests. A 5-second poll interval = 720 wasted requests/hour per client when nothing changes.
- **Effort:** Moderate — per-endpoint refactor
- **Source:** GSF Green Software Patterns (https://patterns.greensoftware.foundation/)

### C2: Uncompressed API payloads

- **Anti-pattern:** API responses served without compression
- **Recommendation:** Enable gzip or brotli compression at the server/CDN level
- **Impact:** 60-80% reduction in payload size for text-based responses (JSON, HTML, CSS, JS)
- **Effort:** Quick fix — server/middleware configuration
- **Source:** Web Almanac 2024, HTTP Archive (https://almanac.httparchive.org/)

### C3: Missing cache-control headers

- **Anti-pattern:** Static assets served without cache-control headers
- **Recommendation:** Set appropriate Cache-Control headers (immutable for hashed assets, max-age for others)
- **Impact:** Eliminates redundant data transfer for repeat visitors. Static assets are typically 60-80% of page weight.
- **Effort:** Quick fix — server/CDN configuration
- **Source:** Web Almanac 2024, HTTP Archive

### C4: Unoptimized images

- **Anti-pattern:** Serving PNG/JPEG when modern formats available, no responsive sizing, no lazy loading
- **Recommendation:** WebP/AVIF format, srcset for responsive sizes, loading="lazy" for below-fold images
- **Impact:** WebP is 25-35% smaller than JPEG at equivalent quality. AVIF is 40-50% smaller.
- **Effort:** Moderate — build pipeline and markup changes
- **Source:** Google Developers Web Fundamentals; Web Almanac 2024

### C5: Large frontend bundles

- **Anti-pattern:** Shipping entire libraries when only parts are used, no code splitting
- **Recommendation:** Tree-shaking, code splitting, dynamic imports for route-based loading
- **Impact:** Varies widely. Common savings: 30-60% bundle size reduction from tree-shaking alone.
- **Effort:** Moderate — build config and import changes
- **Source:** webpack/Vite documentation; Web Almanac 2024

### C6: Verbose serialization at scale

- **Anti-pattern:** Using JSON for high-volume internal service communication
- **Recommendation:** Consider Protocol Buffers, MessagePack, or similar compact formats for internal APIs
- **Impact:** 30-50% smaller payloads vs JSON. Most impactful for high-throughput internal services.
- **Effort:** Moderate to Architectural — requires schema definition, client/server changes
- **Source:** Google Developers Protocol Buffers documentation

---

## Pattern Set: Product Design

### P1: Real-time-only sync

- **Anti-pattern:** All updates pushed in real-time with no option for batched delivery
- **Recommendation:** Default to batched/digest mode for non-urgent content. Let users opt into real-time.
- **Impact:** Reduces server push frequency and client device wake-ups. Impact scales with user count.
- **Effort:** Moderate — requires feature design and UI for preference
- **Source:** GSF Green Software Patterns

### P2: Infinite scroll without limits

- **Anti-pattern:** Loading content endlessly as user scrolls, with no pagination or virtualization
- **Recommendation:** Paginate, or use virtual scrolling to render only visible items
- **Impact:** Prevents speculative loading of content users never see. Typical user views <20% of infinite-scrolled content.
- **Effort:** Moderate — UI refactor
- **Source:** Industry consensus; Nielsen Norman Group research on scrolling behavior

### P3: No data retention policy

- **Anti-pattern:** Storing all data indefinitely with no archival or deletion strategy
- **Recommendation:** Define TTLs for transient data, archive cold data to cheaper/lower-energy tiers
- **Impact:** Less stored data = less storage energy. S3 Glacier uses ~50% less energy than S3 Standard per GB.
- **Effort:** Moderate — requires policy decisions and migration
- **Source:** AWS Well-Architected Sustainability Pillar

### P4: Always-on background features

- **Anti-pattern:** Background sync, location tracking, or analytics running continuously with no user control
- **Recommendation:** Let users disable non-essential background activity. Default to off for battery-intensive features.
- **Impact:** Device-level energy savings. Background activity is a top battery drain on mobile.
- **Effort:** Moderate — feature flag and settings UI
- **Source:** Android Developers battery optimization guides; Apple Energy Efficiency Guide

---

## Pattern Set: Architecture

### A1: Synchronous processing for non-urgent work

- **Anti-pattern:** Processing all tasks synchronously/in-request when some could be deferred
- **Recommendation:** Use async job queues (SQS, Bull, Celery) for non-urgent processing
- **Impact:** Enables batching (fewer cold starts, better utilization), smooths load peaks
- **Effort:** Architectural — requires queue infrastructure and worker setup
- **Source:** AWS Well-Architected Sustainability Pillar

### A2: No CDN for static assets

- **Anti-pattern:** Serving static files directly from origin server
- **Recommendation:** Use a CDN (CloudFront, Cloudflare, Fastly) to serve from edge locations
- **Impact:** Reduces origin server load and long-haul network transit. CDN hit rates of 90%+ are common for static content.
- **Effort:** Quick fix to Moderate — depends on existing infra
- **Source:** Industry consensus; Cloudflare documentation

### A3: Monolithic scheduled jobs

- **Anti-pattern:** Large cron jobs that run on fixed schedules regardless of whether there's work
- **Recommendation:** Event-driven triggers — process when data arrives, not on a timer
- **Impact:** Eliminates wasted runs. A cron job that finds no work 80% of the time wastes 80% of its compute.
- **Effort:** Architectural — requires event source and trigger infrastructure
- **Source:** GSF Green Software Patterns

### A4: Redundant data storage

- **Anti-pattern:** Same data stored in multiple locations without deduplication strategy
- **Recommendation:** Single source of truth with caching/derived views. Deduplicate where possible.
- **Impact:** Linear reduction in storage energy with deduplication ratio
- **Effort:** Architectural — requires data architecture review
- **Source:** Industry consensus

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
- Exclude device-dependent items (dark mode, etc.) from default output
- If no issues found in a category, say so briefly — don't pad the output
