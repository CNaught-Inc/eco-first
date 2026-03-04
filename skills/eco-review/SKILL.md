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
- **Suppression file:** Check for `.eco-ignore` in the project root. Each line contains a pattern ID (e.g., `I7`, `C8`) with optional comment after `#`. Skip suppressed patterns in Steps 2-4. List them at the end of the report.

### Step 2: Match Against Patterns

Compare findings against each pattern in the pattern set below. Record every match with the specific file and line where the anti-pattern appears.

### Step 3: Look Up Cloud Regions

If cloud region configuration is found, look up the region in the carbon intensity reference in `data/carbon-intensity.md`. Note the grid carbon intensity and identify lower-carbon alternatives in the same provider.

### Step 4: Produce the Report

Generate the output report using the format specified below.

---

## Output Format

```
## Eco-Review: [Project Name]

> **[Cø] Powered by CNaught** — carbon-aware code intelligence

### TL;DR

1. **[ID]:** [problem] — [fix] ([effort])
2. **[ID]:** [problem] — [fix] ([effort])
3. **[ID]:** [problem] — [fix] ([effort])

### Top Recommendations

1. **[Pattern ID]: [Description]** [Severity]
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

### Suppressed Patterns
[Pattern IDs from .eco-ignore, if any. Omit this section if no .eco-ignore exists.]

---
*[Cø] Powered by [CNaught](https://www.cnaught.com) — carbon-aware code intelligence*
```

---

## Pattern Set: Infrastructure

### I1: High-carbon cloud region

- **Anti-pattern:** Deploying to a high-carbon-intensity region without considering alternatives
- **Look for:** `region` in `.tf`, `template.yaml`, `pulumi`, docker-compose, or cloud CLI configs; match value against High Carbon tier in `data/carbon-intensity.md`
- **Recommendation:** Migrate to a lower-carbon region in the same provider. See carbon intensity table.
- **Impact:** 2-10x reduction in carbon intensity depending on region pair
- **Severity:** Medium
- **Effort:** Architectural — requires migration plan, latency testing, compliance review
- **Technical risk:** Data residency/compliance violations, migration downtime, cross-region replication complexity
- **Usability risk:** Higher latency for users near original region
- **Skip if:** Region is already in the Green or Moderate tier; compliance/data residency mandates a specific region
- **Tools:** `aws configure get region`, `terraform show | grep region` to identify current region
- **Source:** Google Cloud Region Carbon Data, 2024 (https://cloud.google.com/sustainability/region-carbon)

### I2: Oversized container images

- **Anti-pattern:** Docker images using full OS base (ubuntu, debian) or including build tools in production
- **Look for:** `FROM` in Dockerfiles using `ubuntu`/`debian`/`centos`/`fedora` (not `alpine`/`distroless`/`slim`); build tools (`build-essential`, `gcc`, `make`) in final stage
- **Recommendation:** Use alpine or distroless base images. Multi-stage builds to exclude build deps.
- **Impact:** 50-90% reduction in image size, reducing transfer, storage, and cold start energy
- **Severity:** Medium
- **Effort:** Moderate — Dockerfile refactor, test for compatibility
- **Technical risk:** Missing system libraries in minimal images, debugging difficulty without shell tools
- **Usability risk:** None
- **Skip if:** Already using alpine, distroless, or slim base images; multi-stage build excludes build tools from final image
- **Tools:** `dive <image>` for layer analysis
- **Source:** Docker best practices (https://docs.docker.com/build/building/best-practices/)

### I3: Always-on compute for bursty workloads

- **Anti-pattern:** Running 24/7 instances for workloads with variable or low utilization
- **Look for:** `aws_instance`, `google_compute_instance`, or `azurerm_virtual_machine` in IaC without autoscaling groups; always-on containers in docker-compose with no scaling config
- **Recommendation:** Serverless (Lambda, Cloud Functions) or spot/preemptible instances for fault-tolerant work
- **Impact:** Pay-per-use eliminates idle energy waste. Typical server utilization is 12-18%.
- **Severity:** Medium
- **Effort:** Architectural — requires workload analysis, may need code changes
- **Technical risk:** Cold start latency (100ms–10s), state management complexity, execution time limits
- **Usability risk:** Noticeable delay on first request after idle period
- **Skip if:** Workload is latency-critical with consistent high utilization (>60%); already using autoscaling or serverless
- **Source:** NRDC Data Center Efficiency Assessment, 2014; AWS Well-Architected Sustainability Pillar

### I4: No build caching in CI/CD

- **Anti-pattern:** CI pipeline rebuilds everything from scratch on every run
- **Look for:** CI configs (`.github/workflows/*.yml`, `.gitlab-ci.yml`, `Jenkinsfile`) missing `cache:` or `actions/cache@` steps; Dockerfiles without `--mount=type=cache`
- **Recommendation:** Cache dependencies, Docker layers, build artifacts between runs
- **Impact:** 30-80% reduction in CI compute time (varies by project)
- **Severity:** High
- **Effort:** Quick fix — CI config change
- **Technical risk:** Stale cache causing flaky builds, cache invalidation bugs
- **Usability risk:** None
- **Skip if:** CI already caches dependencies and Docker layers; project has <1 min build time
- **Tools:** `grep -r 'actions/cache\|cache:' .github/workflows/` to check for existing cache config
- **Source:** GitHub Actions caching docs; industry consensus

### I5: Over-provisioned compute

- **Anti-pattern:** Instance types much larger than workload requires
- **Look for:** `instance_type` in IaC with `xlarge`/`2xlarge`/`4xlarge` sizes; `resources.requests` in Kubernetes with high CPU/memory limits relative to typical web workloads
- **Recommendation:** Right-size based on actual utilization metrics. Start small, scale up.
- **Impact:** Linear reduction in energy — half the cores = roughly half the power draw
- **Severity:** Medium
- **Effort:** Moderate — requires monitoring data to right-size accurately
- **Technical risk:** Under-provisioning causes performance degradation under load spikes
- **Usability risk:** Slower response times during peak usage if sized too aggressively
- **Skip if:** Utilization monitoring shows >60% average CPU usage; workload is compute-intensive (ML training, video encoding)
- **Source:** AWS Well-Architected Sustainability Pillar

### I6: Redundant CI runs

- **Anti-pattern:** Running full test suite on every commit regardless of what changed
- **Look for:** CI configs without `paths:` or `paths-ignore:` filters; `on: push` with no path constraints in GitHub Actions; missing `only:changes:` in GitLab CI
- **Recommendation:** Path-based filtering, skip CI for docs-only changes, parallelise test shards
- **Impact:** 20-60% reduction in CI minutes for monorepos (varies by change distribution)
- **Severity:** High
- **Effort:** Quick fix — CI config change
- **Technical risk:** Path filters may miss cross-cutting changes, allowing broken code through
- **Usability risk:** None
- **Skip if:** Single-package repo where all changes affect the build; CI already has path-based filtering
- **Source:** Industry consensus; GitLab CI rules documentation

### I7: x86 instances when ARM is viable

- **Anti-pattern:** Deploying on x86 (Intel/AMD) instances when the workload runs on ARM without modification
- **Look for:** `instance_type` matching x86 families (`m[0-9]i`, `c[0-9]i`, `t[0-9].`, `r[0-9]i` on AWS); `machine_type` with `n2-`/`c2-` on GCP; absence of `arm64`/`graviton`/`cobalt`/`axion` in IaC
- **Recommendation:** Use ARM-based instances (AWS Graviton, Azure Cobalt, GCP Axion) for compatible workloads. Check IaC for instance type families (e.g., `m6i` → `m7g` on AWS).
- **Impact:** Up to 60% less energy per core. 20-40% cost savings on equivalent instance sizes.
- **Severity:** Medium
- **Effort:** Moderate — requires testing application and dependencies on ARM, updating IaC
- **Technical risk:** Native binary dependencies (e.g., compiled C extensions, some ML libraries) may lack ARM builds. Container images must be multi-arch or ARM-specific.
- **Usability risk:** None
- **Skip if:** Workload requires native x86 binaries (e.g., specific ML libraries, legacy compiled code); already using ARM/Graviton instances
- **Source:** ARM Newsroom (https://newsroom.arm.com/); AWS Graviton documentation (https://aws.amazon.com/ec2/graviton/)

---

## Pattern Set: Code

### C1: Polling instead of event-driven

- **Anti-pattern:** Clients polling an endpoint at fixed intervals for updates
- **Look for:** `setInterval` or `setTimeout` with fetch/axios calls; repeated GET requests to the same endpoint in client code; absence of WebSocket/SSE/EventSource usage
- **Recommendation:** Server-Sent Events (SSE), WebSockets, or webhooks
- **Impact:** Eliminates redundant requests. A 5-second poll interval = 720 wasted requests/hour per client when nothing changes.
- **Severity:** Medium
- **Effort:** Moderate — per-endpoint refactor
- **Technical risk:** WebSocket connection management, reconnection logic, firewall/proxy issues
- **Usability risk:** Missed updates if push connection drops silently
- **Skip if:** Polling interval is >60 seconds; endpoint is used for one-time checks (not continuous updates); long-polling is already in use
- **Tools:** Chrome DevTools Network panel to observe repeated identical requests
- **Source:** GSF Green Software Patterns (https://patterns.greensoftware.foundation/)

### C2: Uncompressed text resources

- **Anti-pattern:** Text resources (HTML, CSS, JS, JSON, API responses) served without compression
- **Look for:** Express/Koa without `compression` middleware; nginx without `gzip on;`; CDN config without compression enabled; missing `Content-Encoding` headers in response config
- **Recommendation:** Enable gzip or brotli compression at the server/CDN level for all text-based responses
- **Impact:** 60-80% reduction in payload size for text-based responses. ~30% of websites serve uncompressed text resources, wasting ~500KB per median page load.
- **Severity:** High
- **Effort:** Quick fix — server/middleware configuration
- **Technical risk:** CPU overhead for compression on high-throughput endpoints, incompatibility with streaming responses
- **Usability risk:** None
- **Skip if:** Compression already enabled at CDN/reverse proxy level; all responses are binary (images, video); API only serves small payloads (<1KB)
- **Tools:** `curl -sI -H 'Accept-Encoding: gzip' <url> | grep -i content-encoding` to verify compression
- **Source:** Web Almanac 2024, HTTP Archive (https://almanac.httparchive.org/)

### C3: Missing cache-control headers

- **Anti-pattern:** Static assets served without cache-control headers
- **Look for:** `express.static()` without `maxAge`/`setHeaders`; nginx `location` blocks for static files without `expires` or `add_header Cache-Control`; CDN config without caching rules
- **Recommendation:** Set appropriate Cache-Control headers (immutable for hashed assets, max-age for others)
- **Impact:** Eliminates redundant data transfer for repeat visitors. Static assets are typically 60-80% of page weight.
- **Severity:** High
- **Effort:** Quick fix — server/CDN configuration
- **Technical risk:** Stale content served after deploys if cache-busting strategy is wrong
- **Usability risk:** Users see outdated content until cache expires
- **Skip if:** Cache-Control headers already set; assets served through CDN with caching configured; all content is dynamic/personalized
- **Tools:** `curl -sI <url> | grep -i cache-control` to verify headers on static assets
- **Source:** Web Almanac 2024, HTTP Archive

### C4: Unoptimized images

- **Anti-pattern:** Serving PNG/JPEG when modern formats available, no responsive sizing, no lazy loading
- **Look for:** `<img>` tags with `.png`/`.jpg`/`.jpeg` without `srcset`; images missing `loading="lazy"`; no `<picture>` element with WebP/AVIF sources; image build pipeline without format conversion
- **Recommendation:** WebP/AVIF format, srcset for responsive sizes, loading="lazy" for below-fold images
- **Impact:** WebP is 25-35% smaller than JPEG at equivalent quality. AVIF is 40-50% smaller.
- **Severity:** Medium
- **Effort:** Moderate — build pipeline and markup changes
- **Technical risk:** Older browser incompatibility with WebP/AVIF, build pipeline complexity for multiple formats
- **Usability risk:** Quality degradation if compression too aggressive, layout shift from lazy-loaded images
- **Skip if:** Images already served as WebP/AVIF with srcset; image CDN (Cloudinary, imgix) handles format negotiation; only SVG/icon images used
- **Tools:** `npx sharp-cli`, `npx @squoosh/cli`
- **Source:** Google Developers Web Fundamentals; Web Almanac 2024

### C5: Large frontend bundles

- **Anti-pattern:** Shipping entire libraries when only parts are used, no code splitting
- **Look for:** Webpack/Vite/Rollup config without `splitChunks`/`manualChunks`; `import` of full libraries (`import _ from 'lodash'` vs `import get from 'lodash/get'`); single entry point with no dynamic `import()`
- **Recommendation:** Tree-shaking, code splitting, dynamic imports for route-based loading
- **Impact:** Varies widely. Common savings: 30-60% bundle size reduction from tree-shaking alone.
- **Severity:** Medium
- **Effort:** Moderate — build config and import changes
- **Technical risk:** Code splitting adds loading states, race conditions with shared dependencies
- **Usability risk:** Visible spinners/flashes for lazy-loaded chunks on slow connections
- **Skip if:** Bundle is already <100KB gzipped; no frontend or server-rendered only; already using code splitting and tree-shaking
- **Tools:** `npx webpack-bundle-analyzer`, `npx vite-bundle-visualizer`
- **Source:** webpack/Vite documentation; Web Almanac 2024

### C6: Verbose serialization at scale

- **Anti-pattern:** Using JSON for high-volume internal service communication
- **Look for:** Internal service-to-service calls using `JSON.stringify`/`JSON.parse` or `application/json`; gRPC `.proto` files absent in multi-service repos; high-throughput message queues with JSON payloads
- **Recommendation:** Consider Protocol Buffers, MessagePack, or similar compact formats for internal APIs
- **Impact:** 30-50% smaller payloads vs JSON. Most impactful for high-throughput internal services.
- **Severity:** Low
- **Effort:** Moderate to Architectural — requires schema definition, client/server changes
- **Technical risk:** Schema versioning complexity, harder debugging (binary payloads not human-readable)
- **Usability risk:** None (internal service change, not user-facing)
- **Skip if:** Single-service architecture; internal traffic is low-volume (<100 req/s); JSON overhead is negligible compared to payload content
- **Source:** Google Developers Protocol Buffers documentation

### C7: Dead code in frontend bundles

- **Anti-pattern:** Shipping JavaScript that is never executed by users. Unused exports, abandoned feature code, and over-inclusive library imports remain in production bundles.
- **Look for:** Bundle config without `sideEffects: false` in package.json; CommonJS `require()` instead of ESM `import`; large dependencies in `package.json` that could be replaced with lighter alternatives
- **Recommendation:** Audit bundles with coverage tools (Chrome DevTools Coverage, `webpack-bundle-analyzer`). Remove unused exports, replace large libraries with lighter alternatives, use tree-shaking-friendly ESM imports.
- **Impact:** 206KB median wasted JavaScript per mobile page load (44% of JS bytes unused). Reduces parse time, execution energy, and data transfer.
- **Severity:** Medium
- **Effort:** Moderate — requires bundle analysis, dependency audit, and testing after removal
- **Technical risk:** Removing code that appears unused but is dynamically referenced (e.g., via string-based imports or reflection). Side-effectful modules may break if tree-shaken.
- **Usability risk:** None if properly tested
- **Skip if:** No frontend bundle (server-rendered only); bundle already <50KB gzipped; tree-shaking and dead code elimination already configured
- **Tools:** `npx webpack-bundle-analyzer`, `npx depcheck`
- **Source:** Web Almanac 2024, HTTP Archive (https://almanac.httparchive.org/)

### C8: Excessive third-party scripts

- **Anti-pattern:** Loading numerous third-party scripts (analytics, ads, trackers, social widgets) that bloat page weight, block rendering, and consume device energy
- **Look for:** `<script src="` tags pointing to external domains; GTM/analytics snippets in HTML; multiple tracking pixels; third-party SDK imports in JS bundles
- **Recommendation:** Audit third-party scripts with `navigator.connection` or performance budgets. Remove unused trackers, consolidate analytics, defer non-critical scripts, use facade patterns for heavy embeds (e.g., YouTube, chat widgets).
- **Impact:** 94% of mobile pages load third-party resources; top sites average 66 third-party domains. On media sites, up to 70% of page electricity is consumed by ads and trackers.
- **Severity:** Medium
- **Effort:** Moderate — requires stakeholder buy-in to remove tracking scripts, may need consent management updates
- **Technical risk:** Removing analytics/tracking may break business reporting or A/B testing. Facade patterns require loading the real resource on interaction.
- **Usability risk:** Removing social widgets or live chat reduces convenience for some users
- **Skip if:** Backend-only service with no HTML output; only 1-2 essential third-party scripts (e.g., single analytics provider); all scripts already deferred/async
- **Tools:** `npx bundlephobia <package>`, Chrome DevTools Network panel third-party filter
- **Source:** Web Almanac 2024, HTTP Archive (https://almanac.httparchive.org/); The Shift Project (https://theshiftproject.org/)

### C9: Unsubsetted web fonts

- **Anti-pattern:** Loading full font files (containing thousands of glyphs for all languages) when the page only uses a subset of characters
- **Look for:** `@font-face` without `unicode-range`; Google Fonts URLs without `&text=` or `&subset=`; font files >100KB; `.woff2`/`.woff`/`.ttf` files in assets without subsetting pipeline
- **Recommendation:** Subset fonts to include only required character ranges (e.g., Latin). Use `unicode-range` in `@font-face` declarations. For Google Fonts, use the `&text=` or `&subset=` parameter. Self-host subsetted files for best control.
- **Impact:** 88-99% file size reduction from subsetting. 83% of sites use custom fonts, making this broadly applicable.
- **Severity:** High
- **Effort:** Quick fix to Moderate — subsetting tools are straightforward, but self-hosting adds build step
- **Technical risk:** Missing characters for user-generated content in unexpected languages. Font licensing may restrict subsetting.
- **Usability risk:** Missing glyphs display as boxes/fallback font for unsupported characters
- **Skip if:** Only system fonts used (no `@font-face` or external font loading); fonts already subsetted with `unicode-range`; icon fonts only (limited glyph set by design)
- **Tools:** `npx glyphhanger`, `npx subset-font`
- **Source:** Paul Calvano, Web Font optimization 2024; Web Almanac 2024, HTTP Archive (https://almanac.httparchive.org/)

### C10: Unoptimized video delivery

- **Anti-pattern:** Serving uncompressed video, using GIF for animations, autoplaying video without user intent, or missing `preload` attributes
- **Look for:** `.gif` files in assets or `<img src="*.gif">`; `<video>` tags without `preload` attribute; `autoplay` on non-hero videos; large video files (>5MB) without adaptive bitrate config
- **Recommendation:** Replace GIF with MP4/WebM (94-95% size reduction). Use `preload="none"` or `preload="metadata"` for non-hero videos. Serve adaptive bitrate (HLS/DASH) for longer content. Add `loading="lazy"` for below-fold video.
- **Impact:** GIF→MP4 conversion yields 94-95% file size reduction. 55% of videos lack preload attributes, causing unnecessary data transfer.
- **Severity:** Medium
- **Effort:** Quick fix to Architectural — replacing GIFs is simple; adaptive bitrate requires transcoding pipeline
- **Technical risk:** MP4/WebM browser compatibility (universal for modern browsers). Adaptive bitrate adds infrastructure complexity.
- **Usability risk:** Replacing autoplay with click-to-play changes user experience. Lower bitrate may reduce perceived quality.
- **Skip if:** No video or GIF content in the project; videos already use adaptive bitrate with proper preload; animated content uses CSS animations or Lottie instead of GIF
- **Tools:** `ffmpeg -i input.gif output.mp4` for GIF conversion
- **Source:** Web Almanac 2024, HTTP Archive (https://almanac.httparchive.org/); The Shift Project (https://theshiftproject.org/)

---

## Pattern Set: Product Design

### P1: Real-time-only sync

- **Anti-pattern:** All updates pushed in real-time with no option for batched delivery
- **Look for:** WebSocket/SSE connections for all notification types with no digest/batch alternative; notification settings lacking frequency controls; push notifications for every event without grouping
- **Recommendation:** Default to batched/digest mode for non-urgent content. Let users opt into real-time.
- **Impact:** Reduces server push frequency and client device wake-ups. Impact scales with user count.
- **Severity:** Low
- **Effort:** Moderate — requires feature design and UI for preference
- **Technical risk:** Batch scheduling complexity, ensuring urgent items still deliver promptly
- **Usability risk:** Users miss time-sensitive updates when defaulted to digest mode
- **Skip if:** All notifications are genuinely time-critical (e.g., security alerts, payment confirmations); user base is small (<100 users); digest/batch option already available
- **Source:** GSF Green Software Patterns

### P2: Infinite scroll without limits

- **Anti-pattern:** Loading content endlessly as user scrolls, with no pagination or virtualization
- **Look for:** Scroll event listeners triggering API calls for next page; `IntersectionObserver` loading content without upper bound; absence of `react-window`/`react-virtualized` or equivalent in list components
- **Recommendation:** Paginate, or use virtual scrolling to render only visible items
- **Impact:** Prevents speculative loading of content users never see. Typical user views <20% of infinite-scrolled content.
- **Severity:** Medium
- **Effort:** Moderate — UI refactor
- **Technical risk:** Pagination state management, virtual scrolling library complexity
- **Usability risk:** Extra clicks to navigate pages, less fluid browsing experience
- **Skip if:** Content list is bounded (e.g., <50 items total); virtual scrolling already implemented; content is paginated
- **Source:** Industry consensus; Nielsen Norman Group research on scrolling behavior

### P3: No data retention policy

- **Anti-pattern:** Storing all data indefinitely with no archival or deletion strategy
- **Look for:** Database tables without TTL columns or expiration logic; S3 buckets without lifecycle policies; no `EXPIREAT`/`TTL` in Redis usage; missing data retention docs or config
- **Recommendation:** Define TTLs for transient data, archive cold data to cheaper/lower-energy tiers
- **Impact:** Less stored data = less storage energy. S3 Glacier uses ~50% less energy than S3 Standard per GB.
- **Severity:** Medium
- **Effort:** Moderate — requires policy decisions and migration
- **Technical risk:** Data migration complexity, backup coordination, accidental deletion of needed data
- **Usability risk:** Users lose data they expected to keep, compliance conflicts with retention requirements
- **Skip if:** Data retention policies already defined and enforced; data volume is trivial (<1GB); regulatory requirements mandate indefinite retention
- **Source:** AWS Well-Architected Sustainability Pillar

### P4: Always-on background features

- **Anti-pattern:** Background sync, location tracking, or analytics running continuously with no user control
- **Look for:** Background `setInterval`/`requestAnimationFrame` in client code; service worker sync without user opt-in; `navigator.geolocation.watchPosition` without pause controls; analytics heartbeat pings
- **Recommendation:** Let users disable non-essential background activity. Default to off for battery-intensive features.
- **Impact:** Device-level energy savings. Background activity is a top battery drain on mobile.
- **Severity:** Low
- **Effort:** Moderate — feature flag and settings UI
- **Technical risk:** Feature flag complexity, ensuring critical background tasks (e.g., data sync) still run
- **Usability risk:** Users forget to re-enable features, miss functionality they relied on unconsciously
- **Skip if:** No client-side app (backend-only service); background features are user-controlled with opt-in; no mobile deployment
- **Source:** Android Developers battery optimization guides; Apple Energy Efficiency Guide

---

## Pattern Set: Architecture

### A1: Synchronous processing for non-urgent work

- **Anti-pattern:** Processing all tasks synchronously/in-request when some could be deferred
- **Look for:** Email sending, image processing, PDF generation, or webhook calls happening inside request handlers; absence of job queue libraries (Bull, BullMQ, Celery, Sidekiq, SQS) in dependencies
- **Recommendation:** Use async job queues (SQS, Bull, Celery) for non-urgent processing
- **Impact:** Enables batching (fewer cold starts, better utilization), smooths load peaks
- **Severity:** Medium
- **Effort:** Architectural — requires queue infrastructure and worker setup
- **Technical risk:** Queue monitoring complexity, dead letter handling, eventual consistency issues
- **Usability risk:** Delayed feedback — users don't see results immediately for deferred tasks
- **Skip if:** All processing is user-facing and latency-critical; application is simple CRUD with no background work; async queues already in use for heavy tasks
- **Source:** AWS Well-Architected Sustainability Pillar

### A2: No CDN for static assets

- **Anti-pattern:** Serving static files directly from origin server
- **Look for:** `express.static()`, `nginx` serving from local disk, or `whitenoise` without CDN in front; absence of CloudFront/Cloudflare/Fastly config; static assets served from same domain as API
- **Recommendation:** Use a CDN (CloudFront, Cloudflare, Fastly) to serve from edge locations
- **Impact:** Reduces origin server load and long-haul network transit. CDN hit rates of 90%+ are common for static content.
- **Severity:** High
- **Effort:** Quick fix to Moderate — depends on existing infra
- **Technical risk:** Cache invalidation on deploy, CDN configuration drift, origin failover complexity
- **Usability risk:** None
- **Skip if:** CDN already configured; no static assets (pure API service); assets served from object storage with built-in CDN (e.g., S3 + CloudFront)
- **Tools:** `curl -sI <url> | grep -i 'x-cache\|cf-cache\|x-cdn'` to detect CDN presence
- **Source:** Industry consensus; Cloudflare documentation

### A3: Monolithic scheduled jobs

- **Anti-pattern:** Large cron jobs that run on fixed schedules regardless of whether there's work
- **Look for:** Crontab entries, `node-cron`/`cron` in package.json, `schedule` in CI config, `@Scheduled` annotations, Celery Beat tasks, AWS EventBridge rules on fixed intervals
- **Recommendation:** Event-driven triggers — process when data arrives, not on a timer
- **Impact:** Eliminates wasted runs. A cron job that finds no work 80% of the time wastes 80% of its compute.
- **Severity:** Medium
- **Effort:** Architectural — requires event source and trigger infrastructure
- **Technical risk:** Event ordering guarantees, idempotency requirements, harder to test than cron
- **Usability risk:** None (backend processing change, not user-facing)
- **Skip if:** Jobs always have work to do (e.g., daily report generation); job is lightweight with <1 min runtime; already event-driven with cron as fallback
- **Source:** GSF Green Software Patterns

### A4: Redundant data storage

- **Anti-pattern:** Same data stored in multiple locations without deduplication strategy
- **Look for:** Same entity stored in both database and cache without invalidation strategy; file uploads stored in both local disk and object storage; denormalized copies without sync mechanism
- **Recommendation:** Single source of truth with caching/derived views. Deduplicate where possible.
- **Impact:** Linear reduction in storage energy with deduplication ratio
- **Severity:** Low
- **Effort:** Architectural — requires data architecture review
- **Technical risk:** Single point of failure, cache coherence complexity, migration risk for existing consumers
- **Usability risk:** None (data layer change, not user-facing)
- **Skip if:** Redundancy is intentional for fault tolerance (replicas, backups); data volume is trivial; caching layer has proper TTL/invalidation
- **Source:** Industry consensus

### A5: N+1 database queries

- **Anti-pattern:** ORM code that issues one query per related record instead of batch-loading (e.g., looping over results and lazy-loading associations)
- **Look for:** ORM relations (`hasMany`, `belongsTo`, `@ManyToOne`, `ForeignKey`) plus loops without `include`/`prefetch_related`/`joinedload`/`with`; `.map()` or `forEach` calling `.getRelated()` or accessing lazy-loaded properties
- **Recommendation:** Use eager loading (`include`, `prefetch_related`, `joinedload`, `with`) to batch-fetch related records in a single query. Enable ORM query logging in development to detect N+1 patterns.
- **Impact:** 80% faster at 1000 records with eager loading vs N+1. Reduces database round-trips from N+1 to 2, cutting network and CPU overhead proportionally.
- **Severity:** High
- **Effort:** Quick fix to Moderate — adding eager loading is simple per-query, but auditing an entire codebase takes time
- **Technical risk:** Over-eager loading can fetch more data than needed, increasing memory usage. Complex eager loading can generate slow JOINs.
- **Usability risk:** None
- **Skip if:** No ORM used (raw SQL or query builder only); all queries already use eager loading; data access is single-record lookups (no list queries with relations)
- **Tools:** Prisma: `log: ['query']`; Django: `django-debug-toolbar` or `nplusone`; Rails: `bullet` gem; SQLAlchemy: `echo=True`
- **Source:** PlanetScale blog (https://planetscale.com/); Azure Well-Architected Framework (https://learn.microsoft.com/en-us/azure/well-architected/)

### A6: Verbose logging in production

- **Anti-pattern:** Running DEBUG or TRACE log levels in production, logging full request/response bodies, or retaining high-volume logs indefinitely
- **Look for:** `LOG_LEVEL=debug` in `.env*` or docker-compose; logging middleware dumping full request bodies (`morgan('combined')` without filtering, `req.body` in logs); no log retention config in observability stack
- **Recommendation:** Set production log level to WARN or INFO. Avoid logging full payloads — use structured logging with relevant fields only. Set log retention policies (7-30 days for verbose logs, longer for errors).
- **Impact:** Debug logging produces 10-100x the volume of info-level logging. 30-50% reduction in observability infrastructure costs from right-sizing log levels and retention.
- **Severity:** High
- **Effort:** Quick fix — configuration change for log levels; moderate for retention policies
- **Technical risk:** Reducing log verbosity may make production debugging harder for intermittent issues. Retention policies may discard logs needed for incident review.
- **Usability risk:** None
- **Skip if:** Production log level already WARN or ERROR; structured logging already in use with appropriate field filtering; log retention policies already configured
- **Tools:** `grep -ri 'log_level\|LOG_LEVEL\|loglevel' .env* docker-compose*` to audit current log levels
- **Source:** ClickHouse blog (https://clickhouse.com/blog); Logz.io (https://logz.io/)

---

## Carbon Intensity by Cloud Region

Read `data/carbon-intensity.md` from the eco-first plugin directory.

---

## Ranking and Output Rules

- Always sort recommendations by estimated impact (highest first)
- Use ranges, never point estimates
- Reproduce citations verbatim from pattern definitions
- Include effort annotation on every recommendation
- Include both risk fields on every recommendation. If a risk category genuinely doesn't apply, write "None" — don't omit the field.
- Include the Severity label (High, Medium, or Low) on every recommendation
- Exclude device-dependent items (dark mode, etc.) from default output
- If no issues found in a category, say so briefly — don't pad the output

### Severity Legend

- **High** — significant waste, typically a quick or moderate fix (I4, I6, C2, C3, C9, A2, A5, A6)
- **Medium** — moderate waste or effort required (I1, I2, I3, I5, I7, C1, C4, C5, C7, C8, C10, P2, P3, A1, A3)
- **Low** — marginal impact or architectural scope (C6, P1, P4, A4)

### TL;DR Formatting

The TL;DR section appears before Top Recommendations. Include the top 3 findings by impact, formatted as:
`**[ID]:** [one-line problem] — [one-line fix] ([effort])`
