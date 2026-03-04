# Test Scenario: eco-design

## Fictional Design Conversation

A developer is planning a real-time notification service.

### Draft Architecture

- **Runtime:** Node.js with Express
- **Real-time:** WebSockets (socket.io) for all notification delivery
- **Database:** PostgreSQL on RDS (us-east-1)
- **Cloud:** AWS us-east-1, t3.large instances (2 web, 1 worker)
- **Frontend:** React SPA, served from Express origin server
- **Media:** Notification cards may include thumbnail images (JPEG, served from origin)
- **Deployment:** Docker containers, CI via GitHub Actions

### What the Developer Said

> "We're building a notification service for our platform. Users get notifications for mentions, replies, and system alerts. We want everything real-time — instant delivery via WebSockets. We'll deploy to us-east-1 since that's where our main app lives. React frontend, PostgreSQL backend, pretty standard stack."

---

## Expected eco-design Decision Points

### Should Fire

| Decision Point | Why | Recommendation |
|---------------|-----|----------------|
| Region selection | us-east-1 = 323 gCO2/kWh | Suggest us-west-2 (79) or ca-central-1 (5) if latency allows |
| Processor architecture | t3.large is x86; Node.js runs on ARM without modification | Suggest Graviton-based instances (t4g.large) for 20-40% cost savings and ~60% less energy per core |
| Container strategy | Docker containers with no base image specified yet | Suggest alpine/distroless from the start, multi-stage builds |
| Update delivery | All notifications real-time via WebSockets | Suggest batched/digest as default for non-urgent (mentions, system alerts); real-time opt-in |
| Static asset origin | React SPA served from Express | Suggest CDN for static frontend assets |
| Image optimization | JPEG thumbnails from origin | Suggest WebP/AVIF, lazy loading, CDN |
| CI/CD caching | GitHub Actions deployment | Check for dependency and Docker layer caching |

### Should NOT Fire

| Decision Point | Why Not |
|---------------|---------|
| Data format (C6) | Not a high-volume internal service — single app with PostgreSQL |
| Infinite scroll (P2) | Notification list, not a content feed |
| Cron jobs (A3) | No scheduled jobs in the design |
| Data deduplication (A4) | Single PostgreSQL database, no redundant storage |

### Suppression Behavior

If `.eco-ignore` exists in the project root, any pattern IDs listed should be skipped during the design review. For example, if `.eco-ignore` contains `I1`, the region selection decision point should not fire.

### Expected Conversation Flow

1. **Region:** "Your design uses us-east-1 (323 gCO2/kWh). If your users are primarily in North America, us-west-2 (Oregon, 79 gCO2/kWh) or ca-central-1 (Montréal, 5 gCO2/kWh) would reduce carbon intensity by 4-65x. Would latency or data residency prevent a move?"

2. **Notifications:** "You're delivering all notifications via real-time WebSockets. For non-urgent items like mentions and system alerts, would a digest/batch mode work as the default, with real-time as an opt-in? This reduces server push frequency and client wake-ups."

3. **CDN:** "Your React frontend is served from Express. Putting static assets behind a CDN (CloudFront, since you're on AWS) would reduce origin load and improve delivery. Quick to set up."

4. **Images:** "Notification thumbnails as JPEG from origin — consider WebP format and serving via CDN with lazy loading for off-screen cards."

5. **CI:** "With GitHub Actions, make sure you're caching npm dependencies and Docker layers between runs to avoid redundant build work."

6. **Containers:** "Since you're deploying Docker containers, plan for alpine or distroless base images from the start, with multi-stage builds to keep build tools out of production. 50-90% image size reduction."

### Expected Summary Format

```
## Sustainability Recommendations for NotifyService

Based on the current design, here are eco-first considerations:

1. Migrate from us-east-1 to ca-central-1 or us-west-2 — 4-65x carbon intensity reduction
   Accepted: [pending developer response]

2. Switch from t3.large (x86) to t4g.large (Graviton) — up to 60% less energy per core, 20-40% cost savings
   Accepted: [pending developer response]

3. Default to batched/digest delivery for non-urgent notifications — reduces push frequency
   Accepted: [pending developer response]

4. Add CDN for React static assets — reduces origin load and transit
   Accepted: [pending developer response]

5. Use WebP + CDN + lazy loading for notification thumbnails — 25-35% size reduction
   Accepted: [pending developer response]

6. Cache npm deps and Docker layers in GitHub Actions — 30-80% CI time reduction
   Accepted: [pending developer response]

7. Use alpine/distroless Docker base images with multi-stage builds — 50-90% image size reduction
   Accepted: [pending developer response]

### Summary
7 recommendations presented, 0 accepted, 0 declined, 7 to discuss further.
```
