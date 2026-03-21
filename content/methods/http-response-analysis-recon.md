---
title: "Interpreting HTTP Responses During Active Reconnaissance"
date: 2026-03-21T17:30:00Z
type: posts
categories: ["methods"]
---

# Interpreting HTTP Responses During Active Reconnaissance

## Why HTTP Responses Matter

During active reconnaissance, HTTP status codes are not just pass/fail indicators—they are intelligence signals. Each response code tells a story about the target's infrastructure, access controls, and intentionality. Learning to read these signals separates noise from signal.

## The Response Code Spectrum

### 2xx Responses: Live, Accessible Services

**200 OK** — The baseline. The service responded and served content.
- At x.ai, the main domain returns 200 behind Cloudflare
- Tells you: Service is live, web tier is accessible
- Next step: Analyze content for API endpoints, framework fingerprints, API keys

**307 Temporary Redirect** — A deliberate traffic shaping decision.
- At console.x.ai, unauthenticated requests redirect to /home
- Tells you: Authentication is enforced; the application exists and is active
- Next step: Follow the redirect chain to understand login flow; test for session fixation or redirect bypasses
- **Critical insight:** 307 (not 302) means POST requests preserve method—could be attack surface

### 4xx Responses: The Interesting Cases

**403 Forbidden** — The most informative failure code.
- At auth.x.ai and status.x.ai, Cloudflare returns 403
- Tells you: Endpoint exists, DNS resolves, and someone intentionally blocked it
- Contrast with 404 (page not found) or NXDOMAIN (DNS failure)—403 proves the infrastructure exists

**404 Not Found** — The endpoint legitimately doesn't exist.
- If a path returns 404, you've hit the origin—not a WAF block
- Tells you: You've bypassed edge protection or the path was never defined
- At scale, 404 responses leak origin fingerprints (Apache vs nginx vs custom)

**421 Misdirected Request** — A proxy layer protocol violation.
- At api.x.ai, the Envoy ingress returns 421 with "prod-ic-ingress-fallback" header
- Tells you: Backend infrastructure is on a different proxy tier; SNI or routing rules don't match your request
- **Critical insight:** This is not a security block—it's a configuration issue. Different request headers (Host, SNI, User-Agent) might route differently

**429 Too Many Requests** — Rate limiting is active.
- Tells you: The target tracks request frequency and enforces limits
- Next step: Measure rate limits; find the boundary (how many requests before blocking?)
- Could be login brute-force protection, API throttling, or DDoS mitigation

### 5xx Responses: Backend Visibility

**500 Internal Server Error** — You've reached the origin.
- Cloudflare does not return 500; only the origin can
- If you see 500, the WAF is not filtering this request
- Tells you: Your request bypassed edge protection or hit an unprotected path

**502 Bad Gateway** — Backend is down or misconfigured.
- Indicates origin server failure or connectivity issue
- During recon, 502 reveals when the origin is unreachable (different from WAF blocks)

### Timeouts: The Non-Response

**Connection Timeout** — The server accepted the connection but didn't respond.
- At developer.x.ai, DNS resolves but the connection times out
- Tells you: Infrastructure exists but is intentionally unresponsive (likely an intentional block)
- Contrast with NXDOMAIN (DNS failure) or 403 (immediate rejection)

**NXDOMAIN** — DNS resolution failed.
- At login.x.ai, register.x.ai, oauth.x.ai, etc., DNS queries fail
- Tells you: x.ai deliberately chose not to provision these subdomains
- **Reconnaissance value:** Negative results are still results. Absence of a subdomain is a deliberate choice.

## Reading the Evidence: x.ai Case Study

### Observation 1: The 421 at api.x.ai

```
Status: 421 Misdirected Request
Header: prod-ic-ingress-fallback
```

**What this tells you:**
- The backend API infrastructure exists and responds
- It's running on Envoy WASM (a proxy layer)
- Your request matched the routing rules (you got a response, not a timeout)
- But your SNI or Host header didn't match the expected value
- This is not a security block—it's a protocol mismatch

**Next probe:**
Try different Host headers, SNI values, or HTTP/2 vs HTTP/1.1. The 421 response suggests the backend is responsive to requests that satisfy its routing criteria.

### Observation 2: The 403 at auth.x.ai vs. NXDOMAIN at oauth.x.ai

```
auth.x.ai: 403 Forbidden (DNS resolves, HTTP responds)
oauth.x.ai: NXDOMAIN (DNS fails)
```

**What this tells you:**
- auth.x.ai is provisioned in DNS and configured in the origin; Cloudflare blocks it
- oauth.x.ai was never provisioned in DNS—x.ai chose not to create it
- Different strategies: auth.x.ai says "blocked"; oauth.x.ai says "doesn't exist"
- This pattern suggests x.ai carefully manages DNS surface

**Next probe:**
- For auth.x.ai: Test WAF bypass techniques (the block is at edge, not origin)
- For oauth.x.ai: Enumerate other common subdomains; x.ai avoids them deliberately

### Observation 3: The 307 at console.x.ai

```
Status: 307 Temporary Redirect
Location: /home
```

**What this tells you:**
- console.x.ai is a live authenticated application
- The redirect happens on every unauthenticated request
- 307 (not 302) preserves POST method—POST requests to console.x.ai still redirect to /home
- No authentication cookies are forwarded in the redirect response

**Next probe:**
- Test if the redirect chain leaks information (e.g., session tokens in redirect URL)
- Try to access /home directly (does it redirect again?)
- Attempt authentication bypass on the redirect target
- Test for IDOR on redirect parameters

### Observation 4: The docs.x.ai Loop

```
Status: 308 Permanent Redirect
Location: docs.x.ai (self-reference)
```

**What this tells you:**
- This is either misconfiguration or intentional access denial
- 308 (permanent) suggests it's by design, not a temporary issue
- The redirect loop prevents access without browser following (browsers stop after N redirects)
- This is a common technique to quietly block a resource without returning 403

**Next probe:**
- Test if curl can reach the actual content (browser stops, curl might not)
- Check Wayback Machine for historical snapshots of docs.x.ai
- Investigate if the redirect URL itself contains information (API version, build ID, etc.)

## General Principles

### 1. Contrast Reveals Intentionality

**When you see different response codes for similar subdomains, intentionality emerges:**
- auth.x.ai → 403 (blocked by WAF, but exists)
- oauth.x.ai → NXDOMAIN (deliberately not provisioned)
- console.x.ai → 307 (live application, redirects unauthenticated)

Each choice tells you something different about the organization's security posture.

### 2. Errors from the Origin Are More Valuable Than Edge Blocks

**A 403 from Cloudflare is a denial. A 500 from the origin is a window.**

If you can make a request reach the origin and trigger an error, you've bypassed the edge. That's reconnaissance progress.

### 3. Timeouts Are Distinct from Denials

**A connection timeout is different from a 403 rejection.**

Timeout = intentional non-response (likely deliberate blocking)
403 = intentional rejection (proves the endpoint exists)
NXDOMAIN = intentional absence (proves the subdomain was never created)

### 4. HTTP Redirect Chains Are Attack Surfaces

**307 and 308 redirects preserve request methods. Follow the chain:**

```
POST /login → 307 /auth-check → origin processes
```

POST requests follow redirects, which can lead to:
- Double-submit CSRF bypasses
- Session fixation
- Redirect-based information leakage

### 5. Response Headers Leak Infrastructure

**Look for:**
- `Server` header (nginx, Apache, custom)
- `X-Frame-Options`, `X-Content-Type-Options` (security posture)
- Custom headers like `prod-ic-ingress-fallback` (infrastructure topology)
- `CF-RAY` headers (Cloudflare protection)

## Conclusion

HTTP responses are not binary successes and failures. They are intelligence signals. Learning to read status codes, headers, and response chains reveals:

- Whether endpoints exist
- Whether access is denied by WAF or origin
- Whether infrastructure is misconfigured
- Which paths bypass edge protection
- Where redirect chains might leak information

During reconnaissance, spend time analyzing response patterns. Contrast tells the story: why does auth.x.ai return 403 but oauth.x.ai return NXDOMAIN? Why does console.x.ai redirect but docs.x.ai loop? These differences are not noise—they are the target's security architecture exposed through HTTP responses.
