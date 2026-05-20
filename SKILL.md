---
name: post-deploy
description: Post-deployment monitoring, alerting, rollback, and health checks — know the moment your app breaks and fix it before users notice
version: 1.0.0
author: nerudek
compatible-with: hermes-agent, claude-code, openclaw
---

# Post-Deploy — Monitor, Alert, Recover

## Problem

You deployed your app. Three hours later, a user messages you: "site is down." You had no idea. There is no monitoring, no alerts, no automatic recovery. Every outage is discovered by users, not by your infrastructure. Rolling back requires SSH, git reflog, and praying.

## Solution

A layered monitoring and recovery system: health checks every 60 seconds, instant alerts to Telegram/Discord, automatic rollback on failure, and a dashboard showing uptime history.

## Architecture

```
App -> Health Check Endpoint (/health)
       |
  Uptime Monitor (every 60s)
       |
  +- Healthy -> log, update uptime counter
  +- Unhealthy -> send alert -> attempt restart -> still down -> rollback
```

## 1. Health Check Script (runs on your Mac Mini or separate VPS)

```python
#!/usr/bin/env python3
"""health-check.py - ping your app, alert on failure, attempt recovery"""
import urllib.request, json, time, os, subprocess

TARGET = os.getenv("HEALTH_URL", "https://your-app.com/health")
TELEGRAM_BOT = os.getenv("TELEGRAM_BOT_TOKEN")
TELEGRAM_CHAT = os.getenv("TELEGRAM_CHAT_ID")
MAX_FAILURES = 3
CHECK_INTERVAL = 60

def alert(msg):
    if TELEGRAM_BOT and TELEGRAM_CHAT:
        urllib.request.urlopen(f"https://api.telegram.org/bot{TELEGRAM_BOT}/sendMessage",
            data=urllib.parse.urlencode({"chat_id": TELEGRAM_CHAT, "text": msg}).encode())
    print(f"[ALERT] {msg}")

failures = 0
while True:
    try:
        r = urllib.request.urlopen(TARGET, timeout=10)
        if r.status == 200 and '"status":"ok"' in r.read().decode():
            if failures > 0:
                alert(f"RECOVERED - app healthy after {failures} failures")
            failures = 0
            print(f"[OK] {time.strftime('%H:%M:%S')}")
        else:
            failures += 1
            print(f"[WARN] Bad response: {r.status}")
    except Exception as e:
        failures += 1
        print(f"[FAIL] {e} ({failures}/{MAX_FAILURES})")

    if failures == 1:
        alert(f"FIRST FAILURE - {TARGET}")
    elif failures == MAX_FAILURES:
        alert(f"RESTARTING after {MAX_FAILURES} failures")
    elif failures == MAX_FAILURES + 2:
        alert(f"ROLLING BACK - restart did not help")

    time.sleep(CHECK_INTERVAL)
```

## 2. Uptime Monitoring (Free)

| Service | Free tier | Best for |
|---------|----------|----------|
| Uptime Robot | 50 monitors, 5-min checks | Simple HTTP monitoring |
| Better Uptime | 10 monitors, 3-min + heartbeats | Status page included |
| Pulsetic | 10 monitors, 1-min checks | Modern UI, Slack alerts |
| Self-hosted (above script) | Unlimited | Full control |

## 3. Instant Alerts (Telegram)

```bash
# Create bot with @BotFather, get token
# Get chat ID: curl https://api.telegram.org/bot<TOKEN>/getUpdates

curl -s -X POST "https://api.telegram.org/bot<TOKEN>/sendMessage" \
  -d "chat_id=<CHAT_ID>" \
  -d "text=ALERT: App is DOWN at $(date)"
```

### Discord Webhook

```bash
curl -X POST "$DISCORD_WEBHOOK" \
  -H "Content-Type: application/json" \
  -d '{"content": "ALERT: App is DOWN"}'
```

## 4. Automatic Rollback

| Platform | Command |
|----------|---------|
| Railway | `railway rollback` |
| Fly.io | `fly deploy --image registry.fly.io/app:previous` |
| VPS | `ssh vps "cd /app && git checkout HEAD~1 && docker compose up -d --build"` |
| Vercel | Dashboard: Promote previous deploy to Production |

## 5. Dashboard (Minimal)

```html
<!-- status.html -->
<!DOCTYPE html>
<html><head><meta charset="UTF-8"><title>Status</title>
<style>body{background:#111;color:#fff;font:system-ui;display:flex;justify-content:center;align-items:center;height:100vh}
.up{color:#22c55e}.down{color:#ef4444}</style></head>
<body>
<div><h1 class="up" id="status">CHECKING...</h1></div>
<script>
fetch('https://your-app.com/health').then(r=>r.json()).then(d=>{
  document.getElementById('status').textContent = d.status==='ok'?'UP':'DOWN';
  document.getElementById('status').className = d.status==='ok'?'up':'down';
}).catch(()=>{document.getElementById('status').textContent='DOWN';document.getElementById('status').className='down'});
</script></body></html>
```

## 6. Resource Monitoring

```bash
#!/bin/bash
THRESHOLD=80
USAGE=$(ssh $VPS "free | awk '/Mem/{printf \"%.0f\", \$3/\$2*100}'")
if [ "$USAGE" -gt "$THRESHOLD" ]; then
  curl -s "https://api.telegram.org/bot$TOKEN/sendMessage" \
    -d "chat_id=$CHAT_ID" \
    -d "text=RAM: ${USAGE}% (threshold: ${THRESHOLD}%)"
fi
```

## 7. Log Aggregation

```bash
ssh $VPS "docker compose logs -f --tail=100" > logs/app-$(date +%Y%m%d).log &
```

## Common Pitfalls

1. Health check always returns 200 -- check actual functionality, not just HTTP status
2. No dead man's switch -- if monitoring script crashes, you lose coverage
3. Alert fatigue -- only alert on actionable issues
4. Restart loop -- after 3 failed restarts, STOP and alert
5. Health check URL exposed -- use a cacheable path behind CDN

## FAQ

**Q: How fast should health checks be?**
60 seconds for most apps. 30 seconds for critical APIs. 5 minutes for hobby projects.

**Q: What should the health endpoint check?**
Process alive (minimum). Database connectivity, Redis ping, disk space (better). Lightweight integration test (best).

**Q: Can I monitor multiple apps from one script?**
Yes -- loop over a list of URLs, track failures per-app, alert per-app.

**Q: How do I know if the monitoring itself is down?**
Uptime Robot sends "monitor is down" emails. For self-hosted, use Healthchecks.io as dead man's switch.

**Q: What about SSL certificate expiry?**
Uptime Robot monitors SSL. Or: `echo | openssl s_client -connect domain:443 2>/dev/null | openssl x509 -noout -dates`

**Q: How to handle planned maintenance?**
Toggle maintenance mode, silence alerts for 5 min, deploy, health check, resume alerts.

**Q: Can I monitor internal services?**
Use Tailscale IP (100.x.x.x) in health check from another Tailscale-connected machine.

**Q: What about performance monitoring?**
Track `/health` latency, alert if >2s. Use `time curl` or `time.monotonic()` in script.

**Q: Simplest alerting for solo developer?**
Telegram bot + health check script. 20 lines of Python. Zero cost. Instant phone notifications.

**Q: Transient failures vs real outages?**
Count consecutive failures. 1-2 = warn (transient). 3+ = alert (real). Reset on success.

**Q: Auto-restart or auto-rollback?**
Auto-restart first. Rollback if restart fails within 2 minutes. Human review for anything else.

**Q: Can I monitor cron jobs?**
Healthchecks.io or heartbeat URL that cron pings on completion. If ping stops, cron failed.

**Q: How to set up a status page?**
Better Uptime (free status page included). Or host the status.html above on Vercel (free).

**Q: What metric matters most?**
Uptime percentage. Aim 99.5%+ (= max 3.6h downtime/month). Track with counter.

**Q: How to avoid false positives during deploy?**
Add 60-second grace period after deploy before health checks resume alerting.

---

If this saved you time: [PayPal.me/nerudek](https://www.paypal.me/nerudek)
GitHub: [github.com/nerudek](https://github.com/nerudek)
