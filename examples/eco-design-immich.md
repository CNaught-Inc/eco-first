# Eco-Design: Building a Self-Hosted Photo Manager

> **Reference project:** [immich-app/immich](https://github.com/immich-app/immich) (~60k stars)
> **Premise:** You're building a similar self-hosted Google Photos alternative — photo/video backup, AI-powered search, face recognition, and a web + mobile interface.

---

## What Immich Actually Chose vs. Eco-First Alternatives

### 1. ML Pipeline — Always-On vs. On-Demand

**What Immich chose:**
- A dedicated **always-on ML container** (Python FastAPI + ONNX Runtime) running 24/7
- Every upload triggers: CLIP embedding (smart search), face detection, face recognition, duplicate detection
- Models are ~200-500 MB each, loaded into RAM on first inference, then unloaded after 300s idle TTL
- The ML container idles most of the time after the initial library import

**Eco-first alternative:**
The ML container is the largest idle resource. The eco-first version would:

1. **Scale ML to zero** — use a sidecar or serverless function that spins up only when the job queue has ML work. Cold start adds ~5-10s but saves hours of idle compute daily.
2. **Batch ML processing** — instead of processing each photo immediately, queue uploads and run ML in batches (e.g., every 30 minutes or when queue reaches 50 items). Batching amortizes model load time.
3. **Offer a "lite" mode** — skip CLIP embeddings entirely for users who don't need smart search. Face detection alone is lighter.
4. **Use smaller models by default** — `buffalo_s` instead of `buffalo_l` for face recognition. The quality difference is marginal for personal libraries.

**Impact:** Eliminates hours of idle compute daily. Batching reduces model load/unload cycles. Lite mode cuts ML compute per photo by ~50%.

*Reference: I3 — NRDC Data Center Efficiency Assessment; A1 — AWS Well-Architected Sustainability Pillar*

---

### 2. Thumbnail Generation — Eager vs. Lazy

**What Immich chose:**
- **Three thumbnails generated per upload**: preview (1440px), small (250px WebP), blurred thumbhash
- All generated eagerly on upload via background jobs
- Video thumbnails extracted via FFmpeg scene detection

**Eco-first alternative:**
The thumbhash (tiny blurred placeholder) is brilliant — adopt it. But the 1440px preview may never be viewed for most photos. The eco-first version would:

1. **Generate only the small thumbnail + thumbhash on upload** — these cover the timeline view
2. **Lazy-generate the 1440px preview** on first click — cache it afterward
3. **Use WebP for previews** (not JPEG) — 25-35% smaller at equivalent quality

Most users scroll past 80%+ of their photos without clicking. Lazy preview generation avoids processing the majority.

**Impact:** Reduces per-upload processing by ~40% (skip preview generation). WebP previews save 25-35% storage per preview.

*Reference: C4 — Google Developers Web Fundamentals; A1 — AWS Well-Architected Sustainability Pillar*

---

### 3. Storage Architecture — Originals Forever

**What Immich chose:**
- Originals stored at full quality forever with no automatic tiering
- Thumbnails and transcoded video add 10-20% overhead beyond originals
- Archive feature hides photos from timeline but does **not** reduce storage
- No built-in compression or quality reduction for older assets
- Duplicate detection suggests duplicates but doesn't auto-resolve them

**Eco-first alternative:**
1. **Tiered storage** — after 1 year, offer to compress originals to 80% quality JPEG/HEIF (user opt-in). A 10 MB photo at 80% quality is ~3-4 MB with negligible visual difference.
2. **Auto-resolve exact duplicates** — file-hash matches are definitively duplicates. Auto-delete with a 30-day undo window instead of requiring manual review.
3. **Cold storage tier** — move originals older than 2 years to a cheaper/lower-energy storage backend (S3 Glacier, local cold disk).
4. **Global deduplication** — currently per-library only. Cross-library dedup would catch duplicates from shared albums.

**Impact:** Tiered storage could reduce total storage by 30-50% for mature libraries. Auto-dedup prevents the most obvious waste.

*Reference: P3 — AWS Well-Architected Sustainability Pillar; A4 — Industry consensus*

---

### 4. Docker Image Footprint

**What Immich chose:**
- 4 required containers totaling **~4+ GB** of Docker images
- Server: `node:22-bookworm-slim` + FFmpeg + Sharp/libvips (~1.8 GB)
- ML: Python + ONNX Runtime (~1.8 GB CPU; GPU variants much larger)
- Custom PostgreSQL with vector extensions (~500 MB)
- 6 ML image variants built in CI (CPU, CUDA, ROCm, OpenVINO, ARM NN, RKNN)

**Eco-first alternative:**
1. **Alpine for the server** — `bookworm-slim` is decent but Alpine would be smaller. The FFmpeg + libvips dependencies are the bulk.
2. **Single-binary ML service** — rewrite the ML service in a compiled language (Rust/Go with ONNX bindings) to eliminate the Python runtime overhead. Or use `onnxruntime-web` in the Node.js server to consolidate to one container.
3. **Build only the variants users request** — don't build 6 ML variants × 2 architectures = 12 images on every release. Use a matrix strategy that only builds requested targets.
4. **Consolidate containers** — the server already runs the worker in-process. Adding ML inference (for CPU mode) would reduce to 3 containers.

**Impact:** 50-90% reduction in image sizes. Fewer containers = less orchestration overhead and idle memory.

*Reference: I2 — Docker best practices (https://docs.docker.com/build/building/best-practices/)*

---

### 5. Processor Architecture

**What Immich chose:**
- Server container runs on `node:22-bookworm-slim` — x86_64 by default
- ML container built for 6 architectures including ARM NN and RKNN (ARM support exists but isn't the default)
- Docker Compose defaults to the platform's native architecture

**Eco-first alternative:**
ARM-based processors use up to 60% less energy per core. Immich is well-positioned for ARM since:
- Node.js has full ARM64 support
- ONNX Runtime supports ARM64 (and ARM NN for acceleration)
- PostgreSQL with pgvecto.rs works on ARM64

The eco-first version would:
1. **Default documentation to ARM** for self-hosters on cloud instances — recommend AWS Graviton, Azure Cobalt, or GCP Axion
2. **Publish ARM64 as the recommended tag** on Docker Hub (currently available but not promoted)
3. **Test ARM NN as default ML backend** on ARM platforms — it's already supported but not the default

**Impact:** Up to 60% less energy per core. 20-40% cost savings. For an always-on service with 4 containers, this compounds significantly.

*Reference: I7 — ARM Newsroom; AWS Graviton documentation*

---

### 6. Video Transcoding and Delivery

**What Immich chose:**
- Full video transcoding for web compatibility (h264 default, also hevc, vp9, av1)
- HDR-to-SDR tonemapping for thumbnails
- Hardware acceleration supported but optional
- CPU transcoding is the default (extremely energy-intensive)

**Eco-first alternative:**
1. **Transcode on-demand, not on upload** — most videos in a personal library are never rewatched. Only transcode when a user tries to play a video that the browser can't handle natively.
2. **Skip transcoding for already-compatible formats** — if the source is already h264/mp4, serve the original instead of re-encoding. Most modern phone videos are already web-compatible.
3. **Default to hardware acceleration** — detect available GPU/NPU on startup and auto-enable. CPU transcoding uses 10-50x more energy per video-minute than GPU.
4. **AV1 as default output** — AV1 is 30-50% smaller than h264 at equivalent quality. Encoding is slower but the storage and transfer savings compound over time.

**Impact:** Skipping transcoding for compatible formats eliminates the most wasteful processing. GPU encoding uses 10-50x less energy than CPU. GIF→MP4 conversion yields 94-95% file size reduction for animated content.

*Reference: I3 — AWS Well-Architected Sustainability Pillar; C10 — Web Almanac 2024; The Shift Project*

---

### 7. Frontend — What Immich Gets Right

**What Immich chose:**
- **SvelteKit** — compiles away the framework runtime (smaller than React/Vue)
- **Virtual scrolling** with `TimelineManager` — only renders visible thumbnails
- **Progressive loading** — thumbhash → small thumbnail → preview → original
- **Service worker** for preloading adjacent assets and canceling out-of-view requests
- **WebP for small thumbnails** — modern, efficient format

**Eco-first assessment:**
This is one of the best frontend implementations from a sustainability perspective. The progressive loading pattern (thumbhash → thumbnail → preview → original) is exactly what eco-design recommends. Virtual scrolling prevents loading content the user never sees (pattern P2). The service worker intelligently prefetches without wasteful speculation.

**One improvement:** Offer a "low-data mode" for mobile that skips the preview step (thumbhash → thumbnail → original on tap) and reduces prefetch radius.

*Reference: P2 — Nielsen Norman Group research on scrolling behavior; C5 — webpack/Vite documentation*

---

## Summary: Eco-First Photo Manager Architecture

If building a self-hosted photo manager from scratch with sustainability as a design constraint:

| Decision | Immich's Choice | Eco-First Choice | Difference |
|----------|----------------|-----------------|------------|
| ML processing | Always-on container, per-photo | Scale-to-zero, batched processing | Eliminates idle compute |
| Thumbnails | 3 sizes generated eagerly | Small + thumbhash only; lazy preview | 40% less per-upload processing |
| Preview format | JPEG (1440px) | WebP (1440px) | 25-35% smaller previews |
| Storage lifecycle | Originals forever, no tiering | Tiered: hot → warm → cold over time | 30-50% less storage long-term |
| Duplicate handling | Suggest, require manual review | Auto-resolve exact hash matches | Immediate storage savings |
| Video transcoding | Eager, CPU default | On-demand, GPU default, skip compatible | 10-50x less energy per video |
| Processor architecture | x86_64 default | ARM64 recommended for cloud | Up to 60% less energy per core |
| Docker footprint | 4+ GB across 4 containers | 3 containers, <2 GB total | 50%+ smaller |
| Default ML model | buffalo_l (large) | buffalo_s (small) | Faster, less memory |

### What Immich Gets Right

These patterns should be adopted as-is:
- ✓ **Progressive image loading** (thumbhash → thumbnail → preview → original)
- ✓ **Virtual scrolling** — only renders visible items
- ✓ **SvelteKit** — minimal framework runtime
- ✓ **WebP for thumbnails** — modern efficient format
- ✓ **BullMQ job queue** — efficient background processing
- ✓ **Service worker** with intelligent prefetch/cancel
- ✓ **CLIP-based duplicate detection** — catches near-duplicates, not just exact matches
- ✓ **Configurable concurrency** per job type
- ✓ **30-day trash retention** with auto-cleanup

---

*Generated by [eco-first](https://github.com/CNaught-Inc/eco-first) — sustainability-first design for Claude Code*
