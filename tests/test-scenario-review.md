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

| Pattern | Found In | Why |
|---------|----------|-----|
| I1: High-carbon cloud region | terraform/main.tf | us-east-1 = 323 gCO2/kWh |
| I2: Oversized container images | Dockerfile | FROM ubuntu:22.04 with build tools |
| I5: Over-provisioned compute | terraform/main.tf | m5.2xlarge for a web server |
| C1: Polling instead of event-driven | server.js | Three endpoints polled via setInterval |
| C2: Uncompressed API payloads | server.js | No compression middleware |
| C3: Missing cache-control headers | server.js | express.static with no cache config |
| C4: Unoptimized images | public/index.html | PNG, no lazy loading, no srcset |
| P1: Real-time-only sync | Product design | WebSocket notifications, no digest option |
| P2: Infinite scroll without limits | Product design | /feed infinite scroll |

### Expected Order (by impact, highest first)

1. **I1** — 2-10x reduction (region migration)
2. **I2** — 50-90% image size reduction
3. **C1** — eliminates 720 wasted requests/hour/client (x3 endpoints)
4. **C2** — 60-80% payload reduction
5. **C4** — 25-50% image file size reduction
6. **I5** — linear reduction (right-sizing)
7. **C3** — eliminates redundant transfer for repeat visitors
8. **P1** — scales with user count
9. **P2** — prevents speculative loading

### Category Summary Should Contain

- **Infrastructure:** ⚠ I1, ⚠ I2, ⚠ I5 (no ✓ items — nothing good found)
- **Code:** ⚠ C1, ⚠ C2, ⚠ C3, ⚠ C4
- **Product Design:** ⚠ P1, ⚠ P2
- **Architecture:** no issues identified (no cron jobs, no redundant storage visible)
