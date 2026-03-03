---
name: eco-design
description: Use when planning architecture, brainstorming features, or reviewing a draft design for sustainability. Provides an eco-first checklist of considerations for infrastructure, code, and product design decisions.
---

# eco-design

Walk through sustainability considerations for an architecture or feature being planned. Unlike eco-review (which audits existing code), eco-design is forward-looking — it checks whether the design being discussed has considered greener alternatives at each decision point.

## Process

1. **Read project context** — the design doc, plan, or conversation so far
2. **Identify decision points** — which items from the checklist below are relevant to this design
3. **Present recommendations** — for each relevant decision point, present the eco-first alternative with its rationale and citation
4. **Discuss with developer** — ask whether they want to adopt each recommendation; accept pushback on compliance, latency, or cost constraints
5. **Summarize** — list accepted, declined, and deferred recommendations

## Decision Points Checklist

### Infrastructure Decisions

**Region selection**
If the design specifies a cloud region, look it up in the carbon intensity table below. If it's in the "High Carbon" tier, suggest a lower-carbon alternative in the same provider with comparable latency.
*Reference: I1 — Google Cloud Region Carbon Data, 2024*

**Compute model**
If the design uses always-on instances (EC2, GCE, VMs), check whether the workload is bursty or variable. If so, suggest serverless or spot/preemptible alternatives.
*Reference: I3 — NRDC Data Center Efficiency Assessment; AWS Well-Architected Sustainability Pillar*

**Container strategy**
If the design involves Docker containers, check the base image choice. Suggest alpine or distroless over full OS images. Suggest multi-stage builds if build tools would ship to production.
*Reference: I2 — Docker best practices*

**CI/CD configuration**
If CI/CD is part of the design, check for build caching and path-based filtering. Suggest both if not already planned.
*Reference: I4, I6 — GitHub Actions caching docs; GitLab CI rules documentation*

### Code and API Decisions

**Real-time communication**
If the design includes polling for updates, suggest SSE, WebSockets, or webhooks instead.
*Reference: C1 — GSF Green Software Patterns*

**Data format**
If the design involves high-volume internal service communication using JSON, suggest compact alternatives (Protocol Buffers, MessagePack) for internal APIs.
*Reference: C6 — Google Developers Protocol Buffers documentation*

**Static asset delivery**
Check whether the design addresses:
- Compression (gzip/brotli) — *Reference: C2*
- Cache-control headers — *Reference: C3*
- Image optimization (WebP/AVIF, srcset, lazy loading) — *Reference: C4*
- CDN for static files — *Reference: A2*

**Frontend bundle strategy**
If the design involves a web frontend, check for tree-shaking, code splitting, and dynamic imports.
*Reference: C5 — webpack/Vite documentation; Web Almanac 2024*

### Product Design Decisions

**Update delivery**
If the design includes notifications or real-time updates, suggest batched/digest mode as the default for non-urgent content, with real-time as an opt-in.
*Reference: P1 — GSF Green Software Patterns*

**Content loading**
If the design involves content feeds, suggest pagination or virtual scrolling over infinite scroll.
*Reference: P2 — Nielsen Norman Group research on scrolling behavior*

**Data lifecycle**
If the design stores user or transient data, suggest defining TTLs and archival policies from the start.
*Reference: P3 — AWS Well-Architected Sustainability Pillar*

**Background features**
If the design includes background sync, tracking, or analytics, suggest user-controllable toggles with battery-intensive features defaulting to off.
*Reference: P4 — Android Developers battery optimization guides; Apple Energy Efficiency Guide*

### Architecture Decisions

**Processing model**
If the design processes tasks synchronously that could be deferred, suggest async job queues.
*Reference: A1 — AWS Well-Architected Sustainability Pillar*

**Static asset origin**
If static files are served from the origin server, suggest a CDN.
*Reference: A2 — Cloudflare documentation*

**Job scheduling**
If the design uses cron-based scheduled jobs, suggest event-driven triggers where possible.
*Reference: A3 — GSF Green Software Patterns*

**Data storage**
If the same data is stored in multiple locations, suggest a single source of truth with caching/derived views.
*Reference: A4 — Industry consensus*

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

## Output Format

After walking through relevant decision points:

```
## Sustainability Recommendations for [Project Name]

Based on the current design, here are eco-first considerations:

1. [Recommendation] — [rationale with citation]
   Accepted: [yes/no/modified]

2. ...

### Summary
[X] recommendations accepted, [Y] declined, [Z] to discuss further.
```

## Interaction Rules

- Only raise decision points relevant to the current design — don't ask about Docker if there are no containers
- Present recommendations conversationally, not as a wall of text
- Accept pushback gracefully — note compliance, latency, and cost constraints
- Keep it concise — this adds to an existing planning flow, not a standalone session
