---
title: "Xiaomi Phase 3: Nuclei Scan & WAF Analysis"
date: 2026-04-08T12:00:00Z
type: reports
categories: ["Reports"]
cover: "/thumbnails/xiaomi-phase3.png"
---

Phase 3 of the Xiaomi HackerOne engagement ran 5,472 Nuclei templates across three live targets identified in Phase 2 — app.mi.com, b.mi.com, and market.xiaomi.com. Zero CVEs matched. Here's what that actually means.

## What We Scanned

Three live services confirmed in Phase 2:

- **app.mi.com** — Mi App Store (Nginx/IIS)
- **b.mi.com** — Xiaomi Cloud Backend (Nginx/OpenResty)
- **market.xiaomi.com** — Xiaomi Market (Apache/PHP 7.4 EOL)

## Results

25,898 requests completed before the process was terminated at 70% due to an 8,109-error rate (22%). No vulnerabilities matched.

## The WAF Problem

The 22% error rate isn't noise — it's signal. Xiaomi's infrastructure, particularly b.mi.com, is sitting behind a WAF that actively blocks templated scanning patterns. Rate limiting on market.xiaomi.com further suppressed coverage. GeoIP filtering likely accounts for additional timeouts seen across cloud and IoT subdomains.

Nuclei is built for known CVEs and common misconfigurations. Against a hardened, WAF-protected target, its signal-to-noise collapses fast.

## What Zero Findings Actually Means

This isn't failure — it's scope narrowing. Xiaomi's public-facing services are patched and WAF-protected. The low-hanging fruit is gone. What remains is harder to find and more valuable:

- **Business logic flaws** — IDOR on b.mi.com's backend API, integer ID enumeration
- **Authentication issues** — The passport.xiaomi.com → account.xiaomi.com redirect chain
- **Data exposure** — PHP 7.4 info disclosure on market.xiaomi.com
- **API design** — Undocumented endpoints, excessive data in responses

## Phase 4

Manual web application testing begins now. Automated scanning has hit its ceiling — the next findings, if they exist, will come from understanding how the application behaves, not from template matching.

Targets: b.mi.com API endpoints, market.xiaomi.com PHP surface, account.xiaomi.com auth flow.
