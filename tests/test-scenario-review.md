# Test Scenario: eco-review

## Fictional Project: "FeedApp"

A social media feed application with the following codebase characteristics.

### Infrastructure

**terraform/main.tf**
```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "m5.2xlarge"
}
```

**Dockerfile**
```dockerfile
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y nodejs npm build-essential python3
COPY . /app
WORKDIR /app
RUN npm install
CMD ["node", "server.js"]
```

### Code

**server.js**
```javascript
const express = require('express');
const app = express();

// No compression middleware

// Static files with no cache headers
app.use(express.static('public'));

// Polling endpoint
app.get('/api/notifications', (req, res) => {
  res.json(getNotifications(req.user));
});

// Client polls this every 3 seconds via setInterval
app.get('/api/feed/updates', (req, res) => {
  res.json(getUpdates(req.query.since));
});

app.get('/api/messages/check', (req, res) => {
  res.json(getUnreadCount(req.user));
});
```

**public/index.html**
```html
<img src="/images/hero-banner.png" />
<img src="/images/logo.png" />
<img src="/images/background.png" />
<!-- No lazy loading, no srcset, all PNG -->
```

### Product Design

- Real-time WebSocket notifications with no digest/batch option
- Infinite scroll on the main feed (`/feed`) — no pagination, no virtualization

---

## Expected eco-review Output

### Patterns That Should Fire

| Pattern | Severity | Found In | Why |
|---------|----------|----------|-----|
| I1: High-carbon cloud region | Medium | terraform/main.tf | us-east-1 = 323 gCO2/kWh |
| I2: Oversized container images | Medium | Dockerfile | FROM ubuntu:22.04 with build tools |
| I5: Over-provisioned compute | Medium | terraform/main.tf | m5.2xlarge for a web server |
| I7: x86 instances when ARM is viable | Medium | terraform/main.tf | m5.2xlarge is x86; Node.js runs on ARM/Graviton |
| C1: Polling instead of event-driven | Medium | server.js | Three endpoints polled via setInterval |
| C2: Uncompressed text resources | High | server.js | No compression middleware |
| C3: Missing cache-control headers | High | server.js | express.static with no cache config |
| C4: Unoptimized images | Medium | public/index.html | PNG, no lazy loading, no srcset |
| A2: No CDN for static assets | High | server.js | express.static('public') serving from origin, no CDN config |
| P1: Real-time-only sync | Low | Product design | WebSocket notifications, no digest option |
| P2: Infinite scroll without limits | Medium | Product design | /feed infinite scroll |

### Expected TL;DR (top 3 by impact)

1. **I1:** High-carbon region (us-east-1, 323 gCO2/kWh) — migrate to us-west-2 or ca-central-1 (architectural)
2. **I2:** Full OS container image — switch to alpine/distroless (moderate)
3. **C1:** Polling 3 endpoints every 3 seconds — use SSE or WebSockets (moderate)

### Expected Order (by impact, highest first)

1. **I1** — 2-10x reduction (region migration) [Medium]
2. **I2** — 50-90% image size reduction [Medium]
3. **C1** — eliminates 720 wasted requests/hour/client (x3 endpoints) [Medium]
4. **A2** — CDN hit rates 90%+, reduces origin load and network transit [High]
5. **C2** — 60-80% payload reduction [High]
6. **C4** — 25-50% image file size reduction [Medium]
7. **I7** — up to 60% less energy per core (ARM migration) [Medium]
8. **I5** — linear reduction (right-sizing) [Medium]
9. **C3** — eliminates redundant transfer for repeat visitors [High]
10. **P1** — scales with user count [Low]
11. **P2** — prevents speculative loading [Medium]

### Category Summary Should Contain

- **Infrastructure:** ⚠ I1, ⚠ I2, ⚠ I5, ⚠ I7 (no ✓ items — nothing good found)
- **Code:** ⚠ C1, ⚠ C2, ⚠ C3, ⚠ C4
- **Product Design:** ⚠ P1, ⚠ P2
- **Architecture:** ⚠ A2 (express.static serving from origin, no CDN)

### Suppression Behavior

If `.eco-ignore` contains `I1`, then I1 should appear in Suppressed Patterns, not in recommendations.
