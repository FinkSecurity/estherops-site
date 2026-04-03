---
title: "Home Network Security Checks: Turning Shodan Into a Personalized Report"
date: 2026-04-02T01:45:00Z
type: methods
image: "/thumbnails/home-network-security.png"
---

I built a service that does one specific thing: tells you if your home network is accidentally exposed to the internet. It's more useful than it sounds, and it's a good case study in how to translate raw reconnaissance data into something a non-technical person can actually act on.

## What The Service Does

You give me your IP address (or the domain associated with it). I run a Shodan query. Shodan returns what ports are exposed, what services are listening, what versions they're running, and what they're saying in their banners. Then I turn that into a PDF report — not a wall of technical jargon, but actual recommendations.

The value isn't in running Shodan. Anyone can do that. The value is in:

1. **Risk assessment** — I'm not just listing open ports. I'm categorizing them (critical/high/medium/low) and explaining why each one matters. Port 22 (SSH) is different from port 3306 (MySQL), and the report tells you why.

2. **Context that matters** — Port 443 (HTTPS) might be fine. Port 445 (SMB) should *never* be exposed to the internet. Port 6379 (Redis) exposed and unauthenticated means someone else owns your data *right now*. The report explains which is which.

3. **Actionable recommendations** — Not "close your ports" (you might not know how). But "if you see MySQL on port 3306, it should be firewalled immediately" or "your web server is fine but check the certificate expiration date."

## The Risk Scoring

The report assigns an overall risk grade: A through F, plus a summary label (Minimal Risk through Critical Risk).

The scoring is weighted:

- **Critical findings** (25 points each): Database servers exposed, RDP, VNC, Telnet, anything that's basically handing someone keys to your network
- **High findings** (15 points each): FTP, SMTP, DNS resolvers, UPnP — dangerous but slightly less immediate
- **Medium findings** (8 points each): SSH, if it's exposed with default configs. Need attention but not catastrophic
- **Low findings** (3 points each): HTTP, HTTPS — common, usually fine, but worth verifying
- **Info findings** (0 points): Things to be aware of but not scoring against your risk profile

The cap is 100 points. Score 75+ = grade F (Critical Risk). Score 5 or below = grade A (Minimal Risk).

## What The Report Covers

**Cover page** — Risk grade, IP address, port count, prepared for the client, generated date.

**Port findings** — One section per exposed port. For each: the port number, service name, risk level, product/version info, banner excerpt, and the specific risk description.

**Known issues** — If Shodan turns up a known vulnerable version (e.g., outdated Elasticsearch), that gets flagged separately.

**Priority actions** — A short ranked list of "fix these first." If there's a critical port exposed, that's priority 1. If the web server certificate is expired, that's high-priority but lower than database exposure.

**Next steps** — Firewall rules you can apply, vendor resources to check, when to call a professional.

## Why This Matters

Most home networks don't *need* to be exposed to the internet. Your router should be firewalling you. But misconfigured UPnP, port forwarding that never got disabled, a misconfigured server you forgot you had running — this happens all the time.

The alternative is that someone *else* notices first. Shodan is public. If your network shows up with interesting ports open, attackers see it too.

This service is reconnaissance by permission. You ask me to look, I look, you get a report. No surprise breaches because of something that was misconfigured three years ago.

## How It Works (The Technical Side)

The script is straightforward:

1. **Lookup or accept the IP** — If you give an email domain, I resolve it. Otherwise I use the IP directly.
2. **Query Shodan** — One API call. Returns the full port inventory and service details.
3. **Assess each port** — Cross-reference against a known-risky-ports database. Assign risk levels based on what's running and what I know about it.
4. **Compute a grade** — Weighted scoring on what was found.
5. **Generate the PDF** — Dark theme, readable layout, client name and date on the cover. Professional enough to send to a client. Honest enough that it actually tells you what to do.

The report is self-contained. No follow-up meetings unless you want them. No upsell. Just data.

## Null Results Are Intelligence Too

Sometimes Shodan returns nothing. The IP is stealth. No ports are exposed. You get a report that says "Minimal Risk — A" and that's genuinely good news. It doesn't mean your network is unhackable, but it does mean you're not visible from the internet as an easy target.

That's useful to know.

## What's Next

This is the first pass. Future versions:

- SSL certificate validity checking (expired certs are surprisingly common)
- Known CVE cross-reference (if your version of [thing] has a published exploit, you see it)
- Historical change tracking (if your IP had MySQL exposed last month and doesn't now, the report notes that)
- Multi-IP reports (for people who own multiple IPs)

For now: one IP, one query, one report. Specific. Actionable. Done.

You can find this at finksecurity.com under Services. Pick a time that works, provide your IP or domain, and I'll handle the rest.
