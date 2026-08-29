# Independent Security Research

Independent vulnerability research on self-hosted, open-source applications.
Each target is approached with no prior knowledge of existing vulnerabilities —
hypotheses are formed from source review and testing, not from public CVEs or writeups.

All findings are disclosed privately to maintainers prior to public documentation.
Details are published only after a fix has been released, or the maintainer has
been notified and given reasonable time to respond.

## Methodology

For each target: map the attack surface, identify trust boundaries, form specific
testable hypotheses, verify against source code and live testing, document results
including negative findings.

## Targets

| # | Target | Type | Result |
|---|--------|------|--------|
| 1 | VortexPanel | Self-hosted server control panel | PHP web shell scanner bypass (false negative on hardcoded reverse shell payloads) — reported, fixed in v3.4.11 |
| 2 | web-portal | Self-hosted dashboard |No findings — Stored XSS sound via admin dashboard embed widget |
| 3 | Gitea | Git hosting platform | No findings — SSRF/webhook allow list confirmed sound across proxy and direct paths |
| 4 | WebSSH | Web-based SSH client | No findings — auth and session handling confirmed sound; protocol-level testing via custom SSH mimicker |
| 5 | Keyper | Self-hosted credential manager | No findings — encryption and access control confirmed sound; app does not parse or interpret stored content |
| 6 | Coupon browser extension | Chrome extension backend | No findings — race condition testing (50 concurrent requests) confirmed atomic DB operations with proper locking |

## Certifications

(ISC)² Certified in Cybersecurity (CC)
Actively pursuing (ISC)² CPE credit through documented independent research (see individual write-ups for logged hours).
