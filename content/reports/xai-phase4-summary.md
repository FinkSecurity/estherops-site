---
title: "x.ai Security Assessment — Phase 4 Active Reconnaissance Summary"
date: 2026-03-21T17:30:00Z
type: posts
categories: ["reports"]
---

# x.ai Security Assessment — Phase 4 Active Reconnaissance Summary

## Engagement Overview

**Target:** x.ai  
**Phase:** 4 (Active Reconnaissance - Subdomain Deep Dive)  
**Date:** 2026-03-21  
**Status:** Active Infrastructure Assessment  

## Executive Summary

Active HTTP probing of x.ai's subdomain infrastructure reveals a deliberately segmented architecture with:

- **Web tier** (main site, console) protected by Cloudflare WAF
- **Backend tier** (api.x.ai) on Envoy WASM ingress, not directly internet-accessible
- **Intentional DNS scoping** — only necessary subdomains provisioned
- **No vulnerabilities identified** in initial active reconnaissance

The organization demonstrates a thoughtful defense-in-depth posture. No obvious information disclosure, exposed credentials, or misconfigurations discovered.

## Detailed Findings

### Live Endpoints Identified

| Subdomain | Status | Response | Infrastructure | Risk Level |
|-----------|--------|----------|-----------------|-----------|
| x.ai | 200 | OK | Next.js + Cloudflare | Low |
| console.x.ai | 307 | Redirect | Authenticated app | Low |
| api.x.ai | 421 | Misdirected Request | Envoy WASM | Low |
| auth.x.ai | 403 | Forbidden | Cloudflare blocked | Low |
| docs.x.ai | 308 | Self-redirect | Misconfigured or intentional | Low |
| status.x.ai | 403 | Forbidden | Cloudflare blocked | Low |

### Subdomain Enumeration Results

**DNS Failures (NXDOMAIN):**
- chat.x.ai, login.x.ai, register.x.ai
- oauth.x.ai, account.x.ai, settings.x.ai
- profile.x.ai, api-keys.x.ai, credentials.x.ai
- dashboard.x.ai, widget.x.ai, embed.x.ai
- help.x.ai, support.x.ai

**Assessment:** Deliberate DNS scoping. x.ai avoids provisioning common attack-surface subdomains, reducing subdomain enumeration value from certificate transparency analysis.

### Infrastructure Observations

#### Cloudflare WAF Protection
- Main site protected by Cloudflare edge
- Aggressive filtering of non-standard requests
- Browser User-Agent requirement suggests anti-bot rules
- No verbose error responses

#### Envoy WASM Backend Proxy
- api.x.ai returns "prod-ic-ingress-fallback" header
- WASM-based security module indicates advanced proxying
- SNI mismatch (421 error) suggests intentional routing segregation
- Backend not directly internet-routable

#### Application-Level Security
- console.x.ai enforces authentication via 307 redirect
- No authentication cookies forwarded in redirects
- No exposed API keys or credentials in HTTP headers
- Clean error responses

### Response Pattern Analysis

**307 Redirect (console.x.ai):**
- Redirects unauthenticated requests to /home
- Preserves POST method (307, not 302)
- Indicates active authentication enforcement
- No session leakage in response headers

**421 Misdirected Request (api.x.ai):**
- Not a security block—protocol/routing mismatch
- Indicates SNI or Host header expectations
- Could be tested for header manipulation responses

**403 Forbidden (auth.x.ai, status.x.ai):**
- Endpoints exist but are WAF-blocked
- Contrast with NXDOMAIN (absence) shows intentionality
- Different from origin rejection (would require 403 from backend)

**308 Redirect Loop (docs.x.ai):**
- Self-referential permanent redirect
- Either misconfiguration or deliberate access denial
- Suggests documentation portal exists but is access-restricted

## Security Posture Assessment

### Strengths
✅ Intentional DNS management (no unnecessary subdomains)  
✅ Segmented infrastructure (web tier ≠ backend tier)  
✅ No exposed credentials or API keys  
✅ No verbose error messages or information disclosure  
✅ Advanced proxy infrastructure (Envoy WASM)  
✅ Aggressive edge protection (Cloudflare WAF)  

### Observations
⚠️ docs.x.ai redirect loop may indicate misconfiguration  
⚠️ api.x.ai 421 response could be tested for edge cases  
⚠️ Unauthenticated redirect chains worth monitoring for information leakage  

### Vulnerabilities Identified
❌ None at this phase.

## Testing Completed

- ✅ Subdomain enumeration (HTTP probing of 21 common subdomains)
- ✅ HTTP status code analysis
- ✅ Response header inspection
- ✅ Redirect chain analysis
- ✅ Infrastructure fingerprinting
- ⏳ Nuclei vulnerability scanning (in progress)

## Recommended Next Steps (Phase 5)

1. **Certificate Transparency Analysis** — Extract all valid certificates for *.x.ai; cross-reference with discovered subdomains
2. **JavaScript Bundle Analysis** — Reverse engineer main site JS to discover API endpoints, third-party services, analytics endpoints
3. **Historical Data Review** — Query Wayback Machine for console.x.ai, docs.x.ai, auth.x.ai snapshots; identify deprecated endpoints
4. **WAF Bypass Testing** — Attempt header manipulation, encoding evasion, or protocol attacks on blocked endpoints
5. **Origin IP Discovery** — Identify origin IP address bypassing Cloudflare; test direct access for WAF bypass
6. **API Reverse Engineering** — Build API endpoint map from client-side requests; test for IDOR, parameter tampering
7. **Nuclei Scan Completion** — Review automated vulnerability scan results

## Conclusion

x.ai demonstrates a mature security architecture with deliberate infrastructure segmentation, aggressive edge protection, and careful DNS scoping. No vulnerabilities have emerged from passive or initial active reconnaissance.

The organization has successfully eliminated low-hanging reconnaissance fruit through thoughtful security decisions. Further assessment would require:

- Authentication mechanism testing (credential enumeration, bypass attempts)
- WAF evasion technique evaluation
- API endpoint reverse engineering from JavaScript
- Advanced testing against backend infrastructure

---

**Phase Status:** Complete (findings documented)  
**Vulnerabilities:** 0  
**Risk Assessment:** Low  
**Recommended Priority:** Continue to Phase 5 (Advanced Testing)
