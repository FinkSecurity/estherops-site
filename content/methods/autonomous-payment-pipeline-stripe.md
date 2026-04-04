---
title: "The Fink Security Autonomous Payment Pipeline: Stripe to ESTHER"
date: 2026-03-25T02:35:00Z
type: methods
cover: "/thumbnails/autonomous-payment-pipeline-stripe.png"
categories: ["Methods"]
---

# The Fink Security Autonomous Payment Pipeline: Stripe to ESTHER

## Overview

How does a customer paying for a security assessment automatically trigger reconnaissance work without human intervention? This post documents the payment-to-reconnaissance pipeline that powers Fink Security's autonomous workflow.

**Architecture:** Stripe webhook → task file → ESTHER polling → service handler → output delivery

---

## The Flow: Payment to Recon in 30 Seconds

### 1. Customer Checkout (Stripe)

Customer completes checkout for a security service:
- Selects service (e.g., "Breach & Credential Check" at $47)
- Provides target email/domain
- Completes payment (Stripe checkout.session.completed event)

**Example:** Client buys "Breach & Credential Check" for `carnyride79@gmail.com`

### 2. Webhook Trigger (finksecurity-notify handler)

Stripe sends webhook event to `https://finksecurity.com/webhook/stripe`

Handler validates signature:
```
Stripe-Signature: t=<timestamp>,v1=<hmac_sha256_signature>
```

Handler verifies against `STRIPE_WEBHOOK_SECRET` using HMAC-SHA256.

### 3. Task File Generation

Handler extracts payment metadata and creates task file at `~/tasks_pending/task_<job_id>.json`:

```json
{
  "task_metadata": {
    "job_id": "test-breach-001",
    "timestamp": "2026-03-25T00:00:00+00:00",
    "priority": 2,
    "payment_status": "verified"
  },
  "client_context": {
    "client_email": "carnyride79@gmail.com",
    "client_name": "Adam Fink",
    "service_name": "Breach & Credential Check",
    "authorized_services": ["hibp"]
  },
  "instruction_payload": {
    "action": "breach_credential_check",
    "target": "carnyride79@gmail.com",
    "delivery_path": "~/esther-lab/engagements/clients/test-breach-001/"
  }
}
```

**Key mapping:** Stripe Price ID → service name + authorized actions

### 4. ESTHER Polling (poll-tasks.py)

Polling script runs on 5-minute cron schedule:

```bash
*/5 * * * * python3 ~/poll-tasks.py >> ~/poll.log 2>&1
```

Script checks `~/tasks_pending/` for new task files.

On discovery:
- Parses task JSON
- Validates action is authorized
- Dispatches to appropriate handler

### 5. Service Execution (breach_credential_check handler)

Handler runs HIBP check:

```python
subprocess.run([
    'python3', 'hibp-check.py',
    'carnyride79@gmail.com',
    '--out', '~/esther-lab/engagements/clients/test-breach-001/'
])
```

HIBP API queries Have I Been Pwned for breach records.

**Results for carnyride79@gmail.com:**
- LinkedIn (2021) — High severity
- Twitter (2022) — Medium severity
- Twitch (2021) — Medium severity
- Adobe (2013) — High severity
- Yahoo (2013) — High severity

Output: `hibp-carnyride79_at_gmail_com.json` (2.8 KB)

### 6. Operator Notification (Telegram)

Poll-tasks.py calls `deliver_breach_findings()` after successful run:

```python
def deliver_breach_findings(task, hibp_output_path):
    # Read results
    # Format Telegram message
    # Call notify_operator()
```

Operator receives on Telegram:
```
✅ Breach Check Complete: Adam Fink

Client: Adam Fink
Email: carnyride79@gmail.com
Job ID: test-breach-001

📊 Findings: 5 breach(es) found

📋 Breaches:
  • LinkedIn (High)
  • Twitter (Medium)
  • Twitch (Medium)
  • Adobe (High)
  • Yahoo (High)

📁 JSON Output:
~/esther-lab/engagements/clients/test-breach-001/hibp-carnyride79_at_gmail_com.json

🚀 READY TO DELIVER — forward findings to client
```

### 7. Task Lifecycle Tracking

Task moves through states:
1. `~/tasks_pending/task_<id>.json` — discovered by poller
2. `~/tasks_pending/completed/task_<id>.json` — archived after execution
3. Delivery path populated: `~/esther-lab/engagements/clients/<id>/`

Poll.log tracks the entire pipeline:
```
[2026-03-25T00:05:12] 🔍 Polling ~/tasks_pending/
[2026-03-25T00:05:13] ✅ Found task: test-breach-001
[2026-03-25T00:05:14] 🚀 Running hibp-check for carnyride79@gmail.com
[2026-03-25T00:05:18] ✅ test-breach-001 breach check completed
[2026-03-25T00:05:19] ✅ Operator notified of breach findings
```

---

## Service Mapping: Price ID → Handler

```
price_1TDU... → Service Name → Authorized Actions
price_1TDUM2CcdZLsCGAaDPgcDPtL → Breach & Credential Check → [hibp]
price_1TDU... → Port Scan Assessment → [nmap]
price_1TDU... → DNS Reconnaissance → [dns-enum]
... 9 services total
```

Each price ID maps to a specific service with authorized handler(s).

---

## Why This Architecture?

**Problem:** How do you trigger automated security research without human bottlenecks?

**Traditional approach:**
1. Customer pays
2. Human receives notification
3. Human logs in
4. Human sets up task
5. Human monitors execution
6. Human delivers results
7. 24-48h delay minimum

**Autonomous approach:**
1. Customer pays
2. Task file auto-generated
3. Polling agent detects task
4. Handler executes autonomously
5. Results ready in ~5 minutes
6. Operator reviews and delivers
7. No human bottleneck

**Result:** From payment to findings in 5-10 minutes, not 24+ hours.

---

## Security Considerations

**Stripe webhook verification:**
- HMAC-SHA256 signature validation required
- Replayed events rejected by timestamp check
- Webhook secret never exposed in logs

**Task file integrity:**
- Task files written with restrictive permissions (user only)
- Job IDs generated from Stripe session ID (cannot be guessed)
- Payment status verified before dispatching

**Service authorization:**
- Each price ID maps to whitelist of authorized services
- Arbitrary actions cannot be triggered via payment
- Invalid actions rejected by dispatcher

**Output isolation:**
- Results written to client-specific directories
- No cross-client data leakage
- File paths generated from job_id (unique per payment)

---

## Production Readiness Checklist

- [x] Stripe webhook endpoint configured
- [x] HMAC signature validation working
- [x] Task file parsing and validation
- [x] Poll-tasks.py running on 5-minute cron
- [x] Service handlers (hibp-check.py) operational
- [x] Telegram notifications working
- [x] Delivery directories created and isolated
- [x] End-to-end test passed (test-breach-001)

---

## Deployment

1. Deploy handler.py to webhook endpoint
2. Set `STRIPE_WEBHOOK_SECRET` environment variable
3. Deploy poll-tasks.py to cron scheduler
4. Verify `TELEGRAM_BOT_TOKEN` and `OPERATOR_CHAT_ID` set
5. Monitor poll.log for task execution

---

## Next Steps

- Phase 6: Add additional service handlers (DNS enum, port scan, vulnerability assessment)
- Client dashboard: Real-time task status tracking
- Webhook retries: Handle transient failures gracefully
- Rate limiting: Prevent abuse via multiple rapid purchases

---

**Status:** Production (Phase 1 — Breach Check Handler)  
**Tested:** 2026-03-25 (end-to-end validation complete)  
**Ready for:** Multiple concurrent tasks
