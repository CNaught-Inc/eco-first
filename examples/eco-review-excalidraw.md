# Eco-Review: Excalidraw

> **Repository:** [excalidraw/excalidraw](https://github.com/excalidraw/excalidraw) (~90k stars)
> **What it is:** Browser-based collaborative whiteboard. Hand-drawn style diagrams, real-time multiplayer, export to PNG/SVG, embeddable React component.
> **Stack:** React, Vite/Rollup, Socket.IO, Firebase (Firestore + Storage), TypeScript, Yarn workspaces

> **[Cø] Powered by CNaught** — carbon-aware code intelligence

---

## TL;DR

1. **I1:** Firebase likely on us-central1 (413 gCO2/kWh) — move to europe-west6 or northam-northeast1 (architectural)
2. **C9:** Full CJK font families loaded for all users — subset to Latin, lazy-load CJK on demand (moderate)
3. **C5:** 2+ MiB initial JS bundle — lazy-load Mermaid, image processing, and CJK fonts (moderate)

---

## Top Recommendations

### 1. I1: High-carbon cloud region — Medium

**Found in:** Firebase configuration (collaboration backend)
**Details:** Excalidraw uses Firebase Firestore for persistent scene storage and Firebase Storage for binary files (images). Firebase defaults to `us-central1` (Iowa, **413 gCO2/kWh** — "High Carbon" tier). No evidence of region-specific Firebase configuration in public source or docs.

**Impact:** 2-10x reduction in carbon intensity. Moving from us-central1 (413) to europe-west6 (Zürich, 15) or northam-northeast1 (Montréal, 5) would be a 27-83x improvement.
**Effort:** Architectural — requires Firestore migration, latency testing for global users
**Source:** Google Cloud Region Carbon Data, 2024 (https://cloud.google.com/sustainability/region-carbon)

### 2. C5: Large frontend bundles — Medium

**Found in:** `excalidraw-app/vite.config.mts`, PWA service worker config
**Details:** The main JS bundle exceeds **2 MiB**, confirmed by the Workbox precache limit (`maximumFileSizeToCacheInBytes` had to be increased). The bundle ships all features upfront including:
- `@excalidraw/mermaid-to-excalidraw` — full Mermaid parser, used by a small fraction of users
- `pica` / `image-blob-reduce` — image processing, only needed on image import
- Multiple font families loaded at initialization (including CJK variants)

**Impact:** Varies widely. Common savings: 30-60% bundle size reduction from lazy-loading non-essential modules.
**Effort:** Moderate — dynamic imports for Mermaid converter, image processing libs, and CJK fonts
**Source:** webpack/Vite documentation; Web Almanac 2024

### 3. C4: Unoptimized images (export) — Medium

**Found in:** Export functionality (`@excalidraw/utils`)
**Details:** Raster export only supports **PNG** format. No WebP or AVIF export option. The PNG export embeds scene metadata as tEXt chunks for round-trip reimport, which adds payload. For a tool that exports millions of drawings, this is significant cumulative waste.

**Impact:** WebP is 25-35% smaller than PNG at equivalent quality. AVIF is 40-50% smaller.
**Effort:** Moderate — add WebP export option alongside PNG. Keep PNG for round-trip reimport use case.
**Source:** Google Developers Web Fundamentals; Web Almanac 2024

### 4. A4: Redundant data storage — Low

**Found in:** `LocalData` class, browser storage layer
**Details:** Excalidraw stores scene data in **both localStorage and IndexedDB** simultaneously. localStorage has a known 5 MB limit (Issue #8395) and the data is duplicated in IndexedDB. Binary files and library items go to IndexedDB via `LibraryIndexedDBAdapter`, but scene data exists in both stores.

**Impact:** Linear reduction in storage energy with deduplication. Eliminates the 5 MB localStorage overflow issue.
**Effort:** Moderate — migrate fully to IndexedDB (Issue #1280 already proposed this)
**Source:** Industry consensus

### 5. I4/I6: CI pipeline efficiency (unverified) — Medium

**Found in:** `.github/workflows/`, monorepo build configuration
**Details:** The monorepo has a strict sequential build chain: `common → math → element → excalidraw`. Could not confirm whether `actions/cache` is used for `node_modules`/Yarn cache, or whether path-based filtering skips builds for docs-only changes. The sequential build means even a trivial change may trigger a full rebuild of all 4 packages.

**Impact:** 20-60% reduction in CI minutes with path filtering; 30-80% with dependency caching
**Effort:** Quick fix — add `paths:` filters and `actions/cache` for Yarn store
**Source:** GitHub Actions caching docs; industry consensus

### 6. C9: Unsubsetted web fonts — High

**Found in:** Font loading at initialization, Virgil/Cascadia/CJK font families
**Details:** Excalidraw loads multiple custom font families (Virgil hand-drawn, Cascadia Code, and CJK variants) at application startup. The CJK hand-drawn font is especially large (1-5 MB) and is loaded for all users regardless of whether their document uses CJK characters. Fonts are served from esm.run CDN (good delivery), but are not subsetted to required character ranges.

**Impact:** 88-99% file size reduction from subsetting. Loading only Latin subsets initially and deferring CJK to on-demand could save 1-5 MB on initial load for the majority of users.
**Effort:** Moderate — subset Latin-only versions for default load, lazy-load CJK variants when CJK characters are detected in the document
**Source:** Paul Calvano, Web Font optimization 2024; Web Almanac 2024, HTTP Archive (https://almanac.httparchive.org/)

### 7. P4: Always-on background features — Low

**Found in:** PWA service worker, Socket.IO reconnection
**Details:** The PWA service worker precaches **2+ MiB** of assets on first visit, regardless of whether the user will return. Auto-reconnection via Socket.IO runs continuously during collaboration sessions with no user control. No "lite mode" or option to disable background sync for battery-conscious mobile users.

**Impact:** Device-level energy savings. Background activity is a top battery drain on mobile.
**Effort:** Moderate — selective precaching (core only on install, extras on-demand); add user toggle for background sync
**Source:** Android Developers battery optimization guides; Apple Energy Efficiency Guide

---

## Category Summary

### Infrastructure
- ✓ Multi-stage Docker builds for self-hosting (good base image practices)
- ✓ Hosted on Vercel (edge deployment with automatic CDN)
- ⚠ Firebase likely on us-central1 (413 gCO2/kWh) — see #1
- ⚠ CI path filtering and caching unverified — see #5

### Code
- ✓ **WebSockets via Socket.IO** for real-time collaboration (not polling)
- ✓ **ESM-only distribution** (v0.18.0+) — enables tree-shaking for consumers
- ✓ Fonts loaded from **esm.run CDN** — edge delivery of static assets
- ✓ `FirebaseSceneVersionCache` — prevents redundant cloud writes by comparing version numbers
- ✓ **Debounced saves** via `LocalData` class — not on every keystroke
- ⚠ Main JS bundle exceeds 2 MiB — see #2
- ⚠ PNG-only raster export — see #3
- ⚠ Full CJK font families loaded for all users without subsetting — see #6

### Product Design
- ✓ **Local-first architecture** — zero server energy for solo use (excellent)
- ✓ **End-to-end encryption** — server never sees plaintext (no unnecessary processing)
- ⚠ PWA precaches everything on first visit — see #7
- ⚠ No user control over background sync — see #7

### Architecture
- ✓ **Local-first by default** — IndexedDB + localStorage for solo mode, no server needed
- ✓ **CDN for fonts** — esm.run serves font assets from edge
- ✓ **Version caching** — `FirebaseSceneVersionCache` avoids redundant Firestore writes
- ⚠ Dual browser storage (localStorage + IndexedDB) — see #4
- No issues found: the collaboration architecture (Socket.IO relay + Firebase persistence) is well-designed

---

*Generated by [eco-first](https://github.com/CNaught-Inc/eco-first) — sustainability-first design for Claude Code*
