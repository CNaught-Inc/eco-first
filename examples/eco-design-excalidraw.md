# Eco-Design: Building a Collaborative Whiteboard

> **Reference project:** [excalidraw/excalidraw](https://github.com/excalidraw/excalidraw) (~90k stars)
> **Premise:** You're building a similar collaborative drawing tool — a browser-based whiteboard with real-time multiplayer, export to PNG/SVG, and an embeddable React component.

---

## What Excalidraw Actually Chose vs. Eco-First Alternatives

### 1. Real-Time Collaboration

**What Excalidraw chose:**
- Socket.IO (WebSockets) for real-time collaboration — good choice over polling
- Firebase Firestore for persistent scene storage
- Firebase Storage for binary files (images)
- End-to-end encryption with room-specific keys

**Eco-first alternative:**
WebSockets are the right call (pattern C1). The eco-first concern is the Firebase backend. Firebase's default region is `us-central1` (Iowa, **413 gCO2/kWh** — "High Carbon"). Excalidraw doesn't appear to configure a specific Firebase region.

**Recommendation:** If building this today, choose a Firebase region in the Green Zone — `europe-west6` (Zürich, 15 gCO2/kWh) or configure the Firestore instance in `northam-northeast1` (Montréal, 5 gCO2/kWh). That's a **27-83x** reduction in grid carbon intensity for the persistence layer.

**What Excalidraw does well:** The `FirebaseSceneVersionCache` prevents redundant writes by comparing version numbers before saving. This is an excellent pattern — adopt it.

*Reference: I1 — Google Cloud Region Carbon Data, 2024*

---

### 2. Frontend Bundle Size

**What Excalidraw chose:**
- Vite + Rollup for bundling
- ESM-only distribution (as of v0.18.0) — enables tree-shaking
- Code splitting via dynamic imports
- Main JS bundle exceeds **2 MiB** (confirmed by Workbox precache limit issues)
- PWA with service worker precaching

**Eco-first alternative:**
2 MiB is heavy for a first load. The eco-first approach:

1. **Lazy-load the Mermaid converter** — `@excalidraw/mermaid-to-excalidraw` pulls in the full Mermaid parser. Most users never use it. Dynamic import on first use.
2. **Lazy-load image processing** — `pica` and `image-blob-reduce` are only needed when a user imports an image. Defer.
3. **Font subsetting** — the CJK hand-drawn font can be very large. Load it only when CJK characters are detected, not on initial load.
4. **Target a 500 KB initial bundle** — the core drawing engine (roughjs + canvas) is the minimum. Everything else can be deferred.

**Impact:** 30-60% bundle size reduction from lazy-loading non-essential modules. Every kilobyte saved is multiplied by millions of monthly users.

*Reference: C5 — webpack/Vite documentation; Web Almanac 2024*

---

### 3. Image Export Formats

**What Excalidraw chose:**
- PNG export (with scene metadata embedded as tEXt chunks for round-trip reimport)
- SVG export
- No WebP or AVIF export options

**Eco-first alternative:**
Add **WebP export** as the default for raster exports. WebP is 25-35% smaller than PNG at equivalent quality. AVIF would be 40-50% smaller but has slower encoding (higher CPU cost during export — trade-off).

The PNG metadata round-trip is clever, but the eco-first version would offer: "Export as WebP (smaller) or PNG (supports reimport)?" — letting users choose.

**Impact:** WebP is 25-35% smaller than PNG. For a tool that exports millions of drawings, this reduces storage and transfer energy significantly.

*Reference: C4 — Google Developers Web Fundamentals; Web Almanac 2024*

---

### 4. Font Loading and Subsetting

**What Excalidraw chose:**
- Multiple custom font families: Virgil (hand-drawn), Cascadia Code, CJK hand-drawn variants
- All fonts loaded at application startup regardless of document content
- Fonts served from esm.run CDN (good delivery)
- No subsetting — full Unicode ranges loaded for each font

**Eco-first alternative:**
83% of websites use custom fonts, and subsetting yields 88-99% file size reduction. The eco-first version would:

1. **Subset to Latin by default** — the Virgil and Cascadia fonts only need Latin + common symbols for the majority of users. Subset to `unicode-range: U+0000-00FF, U+0131, U+0152-0153, U+02BB-02BC, U+2000-206F`.
2. **Lazy-load CJK fonts** — detect when CJK characters are typed or pasted into the canvas, then load the CJK font variant on demand. This saves 1-5 MB for users who never use CJK.
3. **Self-host subsetted fonts** — instead of loading from esm.run, bundle subsetted woff2 files for maximum control and cache efficiency.

**Impact:** 88-99% file size reduction from subsetting. Most users would save 1-5 MB on initial load by deferring CJK.

*Reference: C9 — Paul Calvano, Web Font optimization 2024; Web Almanac 2024*

---

### 5. Static Asset Delivery

**What Excalidraw chose:**
- Hosted on **Vercel** (edge deployment — good for latency)
- Fonts loaded from **esm.run CDN** (good)
- Self-hosters must manually configure asset paths

**Eco-first alternative:**
Vercel's edge deployment is excellent for CDN (pattern A2 — satisfied). The eco-first gap is for **self-hosters**: the Docker image serves everything from origin. The eco-first version would include a default nginx config with CDN-friendly headers and document Cloudflare/CloudFront setup.

For fonts, the CDN approach is correct. The improvement: **serve only the fonts the user's document needs**. If a drawing uses only the default hand-drawn font, don't preload the CJK variant.

**Impact:** Reduces origin server load and long-haul network transit. CDN hit rates of 90%+ are common.

*Reference: A2 — Cloudflare documentation*

---

### 6. Data Storage Architecture

**What Excalidraw chose:**
- **Local-first:** localStorage + IndexedDB for solo use (no server needed — excellent)
- **Cloud:** Firebase Firestore + Storage for collaboration
- Dual local storage: localStorage for scene data, IndexedDB for binaries and library items
- Debounced saves (not on every keystroke — good)

**Eco-first alternative:**
The local-first architecture is the best possible sustainability choice for solo use — zero server energy. The eco-first version would:

1. **Consolidate to IndexedDB only** — localStorage has a 5 MB limit (known issue #8395) and duplicates data that's also in IndexedDB (pattern A4 — redundant storage). Migrating fully to IndexedDB eliminates the redundancy.
2. **Add data lifecycle for cloud storage** — collaborative drawings in Firebase persist indefinitely. Add TTLs for abandoned rooms (e.g., no activity for 30 days → archive to cold storage, 90 days → delete).

**Impact:** Eliminating localStorage/IndexedDB duplication reduces browser storage. Cloud TTLs prevent unbounded Firebase growth.

*Reference: A4 — Industry consensus; P3 — AWS Well-Architected Sustainability Pillar*

---

### 7. Background Features

**What Excalidraw chose:**
- PWA service worker precaches 2+ MiB of assets for offline use
- Auto-reconnection via Socket.IO on network drops
- No user toggle for background sync behavior

**Eco-first alternative:**
The PWA precaching is a double-edged sword: great for repeat visitors (assets served from cache), wasteful for one-time visitors (2+ MiB cached and never used again). The eco-first version would:

1. **Cache selectively** — precache only the core drawing engine on install. Load collaboration, export, and library features on-demand.
2. **Add a "lite mode"** — users on mobile or low-bandwidth connections can opt out of the service worker entirely.
3. **User control over sync** — let users pause auto-save during battery-critical situations (pattern P4).

**Impact:** Device-level energy savings. Background activity is a top battery drain on mobile.

*Reference: P4 — Android Developers battery optimization guides; Apple Energy Efficiency Guide*

---

### 8. CI/CD Pipeline

**What Excalidraw chose:**
- GitHub Actions for CI
- Multi-stage Docker builds (good for self-hosted images)
- Sequential package builds: common → math → element → excalidraw

**Eco-first alternative:**
The sequential build chain means a docs-only change may trigger a full rebuild of all 4 packages. The eco-first version would:

1. **Path-based workflow filtering** — skip the build for changes to `*.md`, `docs/`, and `examples/` (pattern I6)
2. **Cache the build chain** — use `actions/cache` for node_modules and intermediate build artifacts (pattern I4)
3. **Parallel package builds** where dependency graph allows (math and common have no shared deps — can build in parallel)

**Impact:** 20-60% reduction in CI minutes for a monorepo this size.

*Reference: I4, I6 — GitHub Actions caching docs; GitLab CI rules documentation*

---

## Summary: Eco-First Whiteboard Architecture

If building a collaborative whiteboard from scratch with sustainability as a design constraint:

| Decision | Excalidraw's Choice | Eco-First Choice | Difference |
|----------|-------------------|-----------------|------------|
| Real-time protocol | Socket.IO (WebSocket) | Same | No change needed |
| Cloud persistence | Firebase (us-central1 default) | Firebase in europe-west6 or northam-northeast1 | 27-83x lower carbon intensity |
| Bundle size | 2+ MiB initial load | <500 KB core + lazy loading | 60-75% smaller first load |
| Image export | PNG only (raster) | WebP default, PNG for round-trip | 25-35% smaller exports |
| Local storage | localStorage + IndexedDB | IndexedDB only | Eliminates redundancy |
| Cloud data lifecycle | Indefinite | TTLs for abandoned rooms | Prevents unbounded growth |
| Font subsetting | Full Unicode ranges loaded | Latin subset + on-demand CJK | 88-99% font file size reduction |
| Font delivery | All loaded at start | On-demand per document needs | Saves CJK font download (~1-5 MB) |
| CI | Full rebuild on every change | Path filtering + caching | 20-60% less CI compute |

### What Excalidraw Gets Right

These patterns should be adopted as-is:
- ✓ **Local-first architecture** — zero server energy for solo use
- ✓ **WebSockets over polling** — efficient real-time sync
- ✓ **Debounced saves** — avoids excessive writes
- ✓ **Version caching** — prevents redundant cloud writes
- ✓ **ESM distribution** — enables tree-shaking for consumers
- ✓ **CDN for fonts** — edge delivery of static assets

---

*Generated by [eco-first](https://github.com/CNaught-Inc/eco-first) — sustainability-first design for Claude Code*
