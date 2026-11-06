# urlvet: Rate Limit Bypass via X-Forwarded-For Spoofing

| Field | Detail |
|---|---|
| **Target** | [urlvet](https://github.com/urlvet/urlvet) (url.vet), a phishing link scanner |
| **Advisory** | [GHSA-jvxq-3x7m-258w](https://github.com/urlvet/urlvet/security/advisories/GHSA-jvxq-3x7m-258w) |
| **CVE** | Pending (under review by GitHub) |
| **Weaknesses** | CWE-348 (Use of Less Trusted Source), CWE-307 (Improper Restriction of Excessive Authentication Attempts) |
| **Severity** | Medium, CVSS 3.1 5.3 (`AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:L`) |
| **Affected** | < 0.4.0 |
| **Fixed in** | 0.4.0 |
| **Credit** | Reporter, aazanisrar |

## Summary

urlvet's API rate limiter trusted the `X-Forwarded-For` and `X-Real-IP` headers from any client. Changing the header on each request gave every request a fresh rate-limit key, so the per-IP limit could be bypassed. That limit was also the only protection on the admin login endpoint.

This write-up covers how I found it, why it worked, how it was fixed, and what I took from it. The [official advisory](https://github.com/urlvet/urlvet/security/advisories/GHSA-jvxq-3x7m-258w) is the source of truth for the vulnerability itself.

## Methodology

I started with a short threat model of the app:

| Question | Answer |
|---|---|
| What does the system do? | Scans a URL and returns a verdict on whether it's a phishing link. |
| Where does data enter? | As the URL, plus the contents of the web page behind it. |
| What does it do with that data? | Caches, parses and scans it. |
| What does it trust that it shouldn't? |  That clients won't spoof their IP. |
| What's the blast radius if that trust is wrong? | Unlimited scans and unlimited password guesses on the admin login. |

My plan was to read the `analyzer` directory and map out the API endpoints. Along the way I read the risk-scoring logic, which checks the ICANN TLD and other criteria and produces a score out of 100.

## Finding the bug

1. **Rate limit middleware.** In the middleware folder I found the rate limiter. It keys requests on Gin's `ctx.ClientIP()`, which behaves much like reading `X-Forwarded-For` directly, with extra features layered on.
2. **Router setup.** I opened the router file looking for a trusted proxies setting. There wasn't one. Gin's default is to trust forwarding headers from every peer, so `ClientIP()` returns whatever the client puts in `X-Forwarded-For` or `X-Real-IP`.
3. **Testing in Burp.** I sent a request, changed the IP in the header to something else, and watched the remaining-requests counter reset. It worked.

## Root cause

```
Client ──(X-Forwarded-For: <anything>)──▶ Gin ClientIP() ──▶ rate-limit key
```

Because no trusted proxies were configured, Gin treated every peer as a trusted proxy. The "client IP" was therefore attacker-controlled input, and anything keyed on it was bypassable. The limit applied to every API route (20 requests a minute per IP), including `POST /api/v1/admin/login`.

## Proof of concept

Against a backend before 0.4.0 that clients can reach directly (same PoC as the advisory):

```bash
for i in $(seq 1 30); do
  curl -s -o /dev/null -w '%{http_code} ' \
    -H "X-Forwarded-For: 203.0.113.$i" \
    "http://<host>:8080/api/v1/length?url=https://example.com"
done
```

Every request returns `200` and `X-RateLimit-Remaining` stays at 19, because each one gets a new rate-limit key. Without the header, requests after the 20th return `429`. The same works against `POST /api/v1/admin/login`.

I only tested this against my own local instance, not against any hosted deployment.

## Impact

Anyone able to reach the backend could bypass the per-IP rate limit:

- **Unlimited scans.** Each scan makes outbound requests and uses headless Chrome on the server, so this allows resource exhaustion and abuse of the server as a fetcher.
- **Unlimited password guesses** on the admin login, which had no other limit.

No data is exposed, and this issue alone doesn't bypass authentication. Affected deployments are those where clients can reach the backend directly, or through a proxy that passes on `X-Forwarded-For` without replacing it. That's why it rates Medium (5.3).

## The fix (0.4.0)

My suggested fix was `SetTrustedProxies([]string{"127.0.0.1"})`. The maintainer explained why that wouldn't work for them: url.vet is self-hosted in many different setups (Cloudflare, nginx/Caddy, direct exposure), and a fixed proxy list would break most of them. Instead, 0.4.0:

- Believes forwarding headers only from loopback and private networks by default.
- Adds `TRUSTED_PROXIES` (IPs/CIDRs, `none`, or `all`) and `TRUSTED_PLATFORM` (reads a CDN's header, e.g. `cloudflare`, when the CDN connects to the backend directly).
- Logs a startup warning when neither is set.
- Adds a separate limit of 5 login attempts a minute per IP on the admin login.
- Documents each deployment type. If you use `TRUSTED_PLATFORM`, make sure only that platform can reach the backend (e.g. a firewall allowing only its IP ranges).

**Workaround if you can't upgrade:** make sure your reverse proxy *replaces* `X-Forwarded-For` rather than appending to it (nginx: `proxy_set_header X-Forwarded-For $remote_addr;`, and Caddy does this by default), and that the backend port isn't reachable from the internet.

## Disclosure timeline

| Date | Event |
|---|---|
| 28 Sep 2026 | Started researching urlvet |
| 17 Oct 2026 | Reported privately to the maintainer |
| 27 Oct 2026 | Maintainer confirmed the finding and agreed on the classification |
| 5 Nov 2026 | Fix released in v0.4.0 |
| 6 Nov 2026 | Advisory published |
| Pending | CVE assigned |

Thanks to Abhishek for a fast, professional and friendly disclosure process.

## Dead ends

I also looked at file browsing through the dev server and `env.js` being visible in the browser. Both turned out to be intended behavior (the Vite dev server, and public env vars meant for the client), so I dropped them.

## Takeaways

- Always check whether a finding is a dev-only feature or intended behavior before reporting it.
- My suggested fix (`SetTrustedProxies` with a fixed list) was right for one setup but wouldn't work for a self-hosted project with many deployment types. The maintainer's configurable approach fits all of them.

## References

- [Official advisory (GHSA-jvxq-3x7m-258w)](https://github.com/urlvet/urlvet/security/advisories/GHSA-jvxq-3x7m-258w)
- [urlvet repository](https://github.com/urlvet/urlvet)
