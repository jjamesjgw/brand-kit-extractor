# Research: $0-forever hosting, DNS, bot protection, analytics, and an SSRF-safe fetch proxy

**Product:** free, anonymous brand-kit tool. Static frontend plus one small serverless endpoint that fetches a third-party page's HTML, its linked CSS and a few small images (logo, favicon, og:image) for the user's browser to analyze. All uploads stay in the browser.
**Operator:** Justin James, individual, Rowlett, Texas. Tool lives at `<sub>.wethejamesphoto.com` (subdomain name TBD; written as `<sub>` below).
**Research date:** 2026-11-08. Every limit below was read from the vendor's live page on this date unless marked otherwise.
**Confidence tags:** [Certain] = read on a primary source today, or observed directly (live DNS/RDAP query). [Likely] = strong inference or consistent secondary sources. [Guessing] = gap-filling; verify before relying on it.

---

## 0. Bottom line

1. **Recommended stack:** Cloudflare **Pages** (static site) + **Pages Functions** (the fetch endpoint, same origin under `/api/*`) + **Turnstile** (bot check) on a dedicated Cloudflare account on the **Workers Free** plan with **no payment method on file**. GitHub (public repo) for source, CI, and Dependabot. Total cost: $0. When a free limit is hit, requests fail until the daily reset. Nothing is billed. [Certain] on the limit behavior; [Likely] that no card is needed (see 3.2).
2. **Why Pages and not Workers, although Cloudflare now steers new projects to Workers:** the DNS for wethejamesphoto.com is on Squarespace Domains, not Cloudflare. Cloudflare's own compatibility matrix says custom domains outside Cloudflare zones work on **Pages only**. Workers would require moving the photography domain's nameservers, and that domain also carries Google Workspace email. [Certain] (matrix); live DNS shows Google Workspace MX records.
3. **Daily capacity is set by the free quota of 100,000 Function invocations per account per day.** The quota resets at 00:00 UTC, which is 6:00 pm CST or 7:00 pm CDT in Rowlett. Static page views do not count against it. With the recommended endpoint design (one Turnstile-gated call for HTML+CSS, plus one call per image), one extraction costs about 5 invocations: **about 20,000 extractions/day in the typical case and about 11,000/day in the worst case.** When the quota runs out, the Function returns Cloudflare error 1027 until the reset. The static UI and the upload-based features keep working. [Certain] on the quota and error; the per-extraction numbers are arithmetic on the design in 3.12.
4. **SSRF:** Workers/Pages Functions cannot `fetch()` IP literals; Cloudflare's docs say requests can only be made to URLs, not IPs. Custom ports *are* honored on current compatibility dates. Workers have no DNS-lookup or connect-time hook, so there is **no way to pin a validated IP**: hostname-based rebinding cannot be fully closed in code. The endpoint must still enforce the guard in Section 1-D. The more important protection is product-level: never make the endpoint an open proxy (Section 1-E). [Certain] on IP literals and ports; [Likely] on no connect-time pinning.
5. **Biggest non-obvious risk:** reputation, not cost. If anyone can make `<sub>.wethejamesphoto.com/api/...` *render* attacker-chosen HTML or SVG, the subdomain can be used for phishing. Google Safe Browsing could then flag it, which can hurt the photography business on the same domain. Fix: proxied content is never served in a renderable form (Section 1-E). [Likely]

---

## 1. Concrete requirements for this product

IDs (INFRA-xx) are for the spec. Each requirement states its reason and source; sources are in Section 5.

### A. Platform and account

- **INFRA-01. Host on Cloudflare Pages with Pages Functions on the Workers Free plan.** Static assets come from the Pages project. The fetch endpoint is a Pages Function under `/api/*`, on the same origin as the UI. *Why:* static requests are "free and unlimited." Functions share the 100,000/day Workers Free quota. Pages is the only Cloudflare compute product that supports a custom domain whose DNS is not on Cloudflare. [Certain] (Pages Functions pricing; migration-guide matrix)
- **INFRA-02. Dedicated Cloudflare account, no payment method, never upgrade.** Create a Cloudflare account used only for this tool, and turn on 2FA. Do not add a card. Do not enable R2 (it requires a payment method) or the $5/mo Workers Paid plan. *Why:* the 100,000/day limit is **per account** and shared by every Worker and Pages Function on it. Without a paid plan, overages fail instead of billing. [Certain] (limits page: "Accounts on the Workers Free plan have a daily request limit of 100,000 requests"); [Likely] (R2 card gate, per community/Q&A sources)
- **INFRA-03. `_routes.json` includes only `/api/*`.** Everything else is a static asset and never invokes a Function. *Why:* only Function invocations count toward the quota. [Certain] (Pages Functions routing and pricing docs)
- **INFRA-04. Set Pages "Fail open".** Path: Settings > Runtime > Fail open / closed. When the Function quota is exhausted, static assets are still served, so the UI and upload features keep working. [Certain] (Pages Functions routing docs)
- **INFRA-05. Compatibility date ≥ 2024-09-02 and the `global_fetch_strictly_public` flag.** A recent date makes custom ports in URLs behave predictably (`allow_custom_ports` is the default from 2024-09-02); the guard must still reject non-default ports. `global_fetch_strictly_public` routes any fetch to the tool's own hostnames through Cloudflare's "front door" instead of directly to an origin. Use the deploy date as the compatibility date. [Certain] (compatibility-flags page)
- **INFRA-06. Keep each Function invocation under 10 ms CPU.** Network wait does not count. Stream bodies instead of buffering, parse HTML only with `HTMLRewriter`, and never base64-encode images on the server. Overruns return error 1102 for that request; they do not bill. [Certain] (limits page: "Waiting on network requests … does not count toward CPU time"; 1102 behavior)
- **INFRA-07. Stay at or below 50 subrequests per invocation, and count every redirect hop.** The Turnstile `siteverify` call also counts. Budget per invocation in the recommended design: about 36 (see 3.12). [Certain] (limits page: "Each subrequest in a redirect chain counts against this limit")
- **INFRA-08. Do not start on Cloudflare Browser Run (headless rendering) in v1.** If added later, it is a user-triggered "deep render" button with its own Turnstile check. It is hard-capped by Cloudflare at **10 browser-minutes/day** on Free, with a 429 "Browser time limit exceeded for today" error after that. [Certain] (Browser Run limits/pricing)

### B. Domain and DNS

- **INFRA-09. Attach `<sub>.wethejamesphoto.com` to the Pages project with one CNAME record at Squarespace Domains** (`<sub>` → `<project>.pages.dev`). Do **not** change nameservers or touch any other record. Exact steps are in 3.9. [Certain] (Pages custom-domain docs; Squarespace DNS help)
- **INFRA-10. Current DNS facts, checked 2026-11-08, that the setup relies on:**
  - Registrar: Squarespace Domains LLC (IANA 3827).
  - Nameservers: `ns-cloud-d1…d4.googledomains.com`.
  - DNSSEC: off (no DS record).
  - **No CAA records**, so Cloudflare's CAs are not blocked.
  - No wildcard record.
  - MX points to Google Workspace (`aspmx.l.google.com`).
  - `www` CNAME points to `ext-sq.squarespace.com`.
  - Domain expires **2027-08-24**.

  Re-check CAA right before setup. If anyone adds CAA records later, they must include `letsencrypt.org`, `pki.goog; cansignhttpexchanges=yes` and `ssl.com`. [Certain] (live DNS/RDAP queries; Pages debugging docs)
- **INFRA-11. The Function rejects any request whose `Host` is not the production hostname.** That covers `*.pages.dev` and preview aliases, which would otherwise expose the proxy on Cloudflare-owned hostnames outside Turnstile's hostname check. Set preview builds to "None" or production-only. [Likely]

### C. Endpoint API shape (recommended "text bundle + per-image" design)

- **INFRA-12. `POST /api/extract`**, body `{url, turnstileToken}` (JSON, ≤ 4 KB, URL ≤ 2,048 chars). One invocation does the following:
  - (a) Verify Turnstile.
  - (b) Guarded-fetch the page HTML: ≤ 5 redirect hops, ≤ 2 MB.
  - (c) One `HTMLRewriter` pass collects stylesheet hrefs, inline `<style>`, icon/apple-touch-icon/og:image/logo-candidate URLs, and metadata (`<title>`, `og:site_name`, `meta description`, `theme-color`).
  - (d) Guarded-fetch up to 10 stylesheets plus one level of `@import`: ≤ 1 MB each, ≤ 3 MB total, at most 6 concurrent.
  - (e) Return JSON `{html, css[], meta, imageTicket, imageCandidates[]}`.

  `imageTicket` is an HMAC-SHA256 (WebCrypto) signature over `{exp: now+120s, urls: [≤ 8 discovered image URLs]}`. [Likely] (design; limits from the Workers limits page)
- **INFRA-13. `POST /api/image`**, body `{imageTicket, i}`. The Function verifies the HMAC and expiry, then streams **only `urls[i]`** through the guard (≤ 1 MB raster, ≤ 512 KB SVG). The client can fetch only images the server itself discovered on the page the human asked for. No Turnstile per image. [Likely] (design)
- **INFRA-14. Image responses are not renderable.** Return them as `Content-Type: application/octet-stream`, with the validated real type in a custom header (for example `X-BK-Type`). The client builds a `Blob` with that type after checking magic bytes. SVG is always sanitized client-side (for example DOMPurify) before any inline use. [Likely]

### D. SSRF guard (applies to every outbound fetch, including each redirect hop)

- **INFRA-15. Scheme allowlist: `http:` and `https:` only.** Reject URLs carrying credentials (`user:pass@`). Strip the fragment. [Likely] (OWASP-standard practice)
- **INFRA-16. Reject IP-literal hosts.** After WHATWG `new URL()` parsing, reject any hostname that is dotted-quad IPv4 or bracketed IPv6. The parser already normalizes decimal, octal and hex forms such as `2130706433` and `0x7f.1` to dotted-quad, so a single check catches them. The platform also refuses IP fetches (error 1003, "Direct IP access not allowed"), but reject explicitly so failures are clean. [Certain] that Workers refuse IP literals (known-issues page); [Likely] on parser normalization.
- **INFRA-17. Hostname rules.** The name must contain a dot and be ≤ 253 chars. Reject:
  - `localhost` and `*.localhost`;
  - the suffixes `.local .internal .intranet .lan .home .home.arpa .corp .test .invalid .example .onion .arpa`;
  - wildcard-IP DNS services (`nip.io`, `sslip.io`, `xip.io`, `localtest.me`, `lvh.me`, `1u.ms`, `traefik.me`);
  - the tool's own hostname and `*.pages.dev` (loop protection).

  [Likely] (Portkey's published blocklist mirrors this)
- **INFRA-18. Port allowlist: the URL port must be empty (default), `80` or `443`.** On current compatibility dates, Workers honor custom ports for non-Cloudflare origins, so this check is required. [Certain] (`allow_custom_ports` flag text)
- **INFRA-19. Optional DNS pre-check of the page host.** Resolve A/AAAA through DoH (`cloudflare-dns.com/dns-query`, JSON). Reject if any answer falls in a private, reserved, loopback, link-local, CGNAT, multicast, documentation, NAT64-embedded or IPv4-mapped range. The check costs 2 subrequests and does not stop time-of-check/time-of-use rebinding, so it is defense in depth only. [Likely]
- **INFRA-20. Follow redirects manually.**
  - Use `redirect: 'manual'` and allow at most 5 hops.
  - Resolve `Location` relative to the current URL.
  - Re-run the full guard on every hop and count each hop as a subrequest.
  - Cancel the body of every 3xx response.

  [Certain] that hops count; [Likely] on the design.
- **INFRA-21. Timeouts.** Use `AbortSignal.timeout(8000)` per subrequest and an overall 20 s deadline per invocation. There is no platform duration limit, so the code must impose its own. [Certain] (limits page: no duration limit for HTTP-triggered invocations)
- **INFRA-22. Byte caps with a streaming cutoff.** First reject on `Content-Length` over the cap. Then pipe through a counting `TransformStream` that errors at the cap; a missing or false header must not bypass the limit. Caps: HTML 2 MB; CSS 1 MB each, 3 MB total; raster 1 MB; SVG 512 KB. [Likely]
- **INFRA-23. Content-type allowlist** (media type before `;`):
  - Page: `text/html`, `application/xhtml+xml`.
  - CSS: `text/css`; accept `text/plain` only when the path ends in `.css`.
  - Images: `image/png`, `image/jpeg`, `image/gif`, `image/webp`, `image/avif`, `image/svg+xml`, `image/x-icon`, `image/vnd.microsoft.icon`.
  - Reject everything else. Also check the image magic bytes on the client.

  [Likely]
- **INFRA-24. Build fresh outbound headers.**
  - Never pass the incoming `Request` object to `fetch()`; that forwards its headers.
  - Never forward `Cookie`, `Authorization`, `Referer`, `CF-*`, `X-Forwarded-*` or the visitor's user agent or language.
  - Send a fixed, honest `User-Agent` naming the tool and its about page, a type-appropriate `Accept`, and `Accept-Language: en`.
  - Set `cache: 'no-store'` so Cloudflare does not cache fetched third-party content. This keeps "stores nothing" literally true.

  [Certain] that `no-store` bypasses Cloudflare caches for non-Cloudflare origins (fetch docs); [Likely] on the rest.
- **INFRA-25. Build the response from scratch with an allowlist of headers.**
  - Strip every upstream header, `Set-Cookie` included.
  - Always add: `Cache-Control: no-store`, `X-Content-Type-Options: nosniff`, `Content-Security-Policy: default-src 'none'; sandbox; frame-ancestors 'none'`, `Cross-Origin-Resource-Policy: same-origin`, `Referrer-Policy: no-referrer`.
  - Never emit `Access-Control-Allow-*` headers.
  - Return fetched HTML and CSS only as JSON string fields, never as `text/html` or `text/css` responses.

  [Likely]

### E. Abuse and open-proxy prevention

- **INFRA-26. A fresh Turnstile token is required for every `/api/extract` call.** Check `success === true`, `hostname === <sub>.wethejamesphoto.com` and `action === 'extract'` in the `siteverify` response. Tokens are single-use and expire after 300 s. [Certain] (Turnstile server-side validation docs)
- **INFRA-27. Same-origin callers only.**
  - Methods: POST only (any other method gets 405).
  - Require `Content-Type: application/json`; this forces a CORS preflight that the endpoint never approves.
  - Require `Origin` to equal the production origin, and `Sec-Fetch-Site: same-origin` when that header is present.
  - Send no CORS headers.

  Non-browser clients can spoof these headers; Turnstile is the real gate. [Likely]
- **INFRA-28. Proxied content can never be fetched with a GET or rendered as a page on the tool's domain.** This kills the phishing and hosting-abuse vector that would put wethejamesphoto.com's reputation at risk. [Likely]
- **INFRA-29. Per-IP rate limiting.** Pages Functions **cannot** use the Rate Limiting binding (matrix: Workers ✅, Pages ❌), and WAF rate-limit rules need the zone on Cloudflare. v1 options:
  - (a) A best-effort in-isolate counter keyed by a salted hash of `CF-Connecting-IP`. It is per location, resets whenever the isolate is recycled, and is kept in memory only.
  - (b) Rely on Turnstile + per-call caps + the image-ticket allowlist.

  Upgrade path if abuse appears: a separate Worker with a SQLite Durable Object, bound to the Pages project, enforcing strict counters. Workers Free includes 100,000 DO requests/day. [Certain] (matrix, WAF table, DO pricing); [Likely] (design)

### F. Capacity and graceful degradation

- **INFRA-30. Treat these responses from `/api/*` as "live scanning unavailable today":**
  - HTTP 429;
  - a body or status reflecting Cloudflare error 1027 or 1102;
  - any response missing the tool's marker header (for example `X-BK: 1`);
  - any non-JSON body, since fail-open may serve a static 404 or 405 instead of the Function.

  [Certain] (1027 on quota exhaustion); [Likely] (fail-open fall-through shape, see Unverified)
- **INFRA-31. Capacity-mode UI.**
  - Disable the URL input.
  - Show the reset time in the visitor's local time, computed in the browser from 00:00 UTC.
  - Point the visitor to the in-browser inputs: logo upload, screenshot-to-palette, brand-guide PDF, manual entry.

  Example copy: "Live site scanning hit today's free limit. It resets at 6:00 pm your time. Uploads still work: add your logo, a screenshot, or a brand PDF. They're processed in your browser." Keep the state in memory only, with no storage write. [Likely]
- **INFRA-32. Never put a capacity number in UI copy.** The owner's rule is "never invent numbers". The quota is shared and changes, so do not print "X scans per day" anywhere. [Likely]

### G. Analytics and observability

- **INFRA-33. Default to no client-side analytics.** The companion privacy research (PRIV-09) recommends no JS beacon in v1. The Pages project dashboard's Functions metrics (requests and errors per day) already count extractions with no client code. If the owner wants page-view counts, Cloudflare Web Analytics is free and cookieless: it uses "no client-side state, such as cookies or localStorage" and "no fingerprinting". It can be enabled in one click (Pages > Metrics > Enable), but it is still a JS beacon. Decide this in the privacy spec, not here. [Certain] (Web Analytics product page and docs)
- **INFRA-34. No persisted logs.** Pages Functions have real-time logs only; Workers Logs is ❌ for Pages in Cloudflare's matrix. Never `console.log` target URLs, IPs or bodies. [Certain] (matrix)

### H. CI/CD and repository hygiene

- **INFRA-35. Use Pages Git integration (Cloudflare builds from GitHub) rather than Direct Upload**, so no Cloudflare API token is stored in the public repo's secrets. The choice cannot be switched later in either direction without creating a new project. Free plan: 500 builds/month, 1 concurrent. Builds time out after 20 min. [Certain] (Pages limits; Direct Upload docs)
- **INFRA-36. Dependabot (npm + github-actions ecosystems, weekly, grouped), secret scanning with push protection, and CodeQL default setup**, all on standard GitHub-hosted runners. GitHub states that standard runners in public repositories are free and do not reduce the included usage. Without a payment method, usage over included amounts is blocked, not billed. [Certain] (GitHub Actions billing doc); [Likely] (Dependabot on standard runners is free)
- **INFRA-37. Secrets (`TURNSTILE_SECRET`, `TICKET_HMAC_KEY`) live only in Pages encrypted secrets.** Never commit them; the repo is public. Use Turnstile's test keys for local and preview builds. [Likely]

### I. Cost-trap guardrails

- **INFRA-38. No paid Cloudflare products and no card on file.** Avoid R2, Workers Paid, Logpush, Images, Stream, Argo and Load Balancing. [Likely]
- **INFRA-39. Self-host font files as static assets, not through a Function.** Static requests are free and unlimited. A Function proxy would burn the daily quota on every font preview. Pages limits: 20,000 files per site, 25 MiB per file. [Certain] (limits); [Guessing] that a Latin-subset Google Fonts catalog fits under 20,000 files, so measure it.
- **INFRA-40. The tool depends on the photography domain.** wethejamesphoto.com expires 2027-08-24. Keep auto-renew on at Squarespace; if the domain lapses, the tool's URL dies with it. [Certain] (RDAP)

---

## 2. Recommendation: one stack

**Cloudflare Pages + Pages Functions + Turnstile on Workers Free, DNS left at Squarespace, source and CI on GitHub (public).**

| Need | Choice | $ | Key limit | Tag |
|---|---|---|---|---|
| Static frontend | Pages static assets | 0 | Free and unlimited requests; 20,000 files; 25 MiB/file; 500 builds/mo | [Certain] |
| Fetch endpoint | Pages Function `/api/*` | 0 | 100,000 invocations/day per account (shared with Workers); 10 ms CPU; 50 subrequests; 128 MB memory | [Certain] |
| Bot check | Turnstile Free | 0 | Up to 20 widgets; unlimited challenges; 10 hostnames/widget | [Certain] |
| Analytics (optional) | Pages Functions metrics; Web Analytics only if the privacy spec allows a beacon | 0 | Web Analytics: soft limit of 10 sites/account; 6 months of data | [Certain] |
| Headless fallback (later) | Browser Run binding (Pages ✅) | 0 | 10 browser-min/day; 3 concurrent; 1 new browser per 20 s | [Certain] |
| Custom domain | CNAME at Squarespace | 0 | Pages: 100 custom domains/project | [Certain] |
| CI | Pages Git integration + GitHub Actions/Dependabot | 0 | Standard runners free on public repos | [Certain] |

**Why not the alternatives** (details in 3.8):
- **Vercel Hobby** is "non-commercial personal use only". A portfolio piece with a footer link promoting the owner's services plausibly counts as "advertising the sale of a product or service". When a limit is exceeded, Hobby features pause for up to 30 days.
- **Netlify Free** has a hard cap of 300 credits/month; every production deploy costs 15 credits, and at zero credits **all projects go offline**.
- **GitHub Pages** is static only.
- **Fly.io** has no free tier and requires a card.
- **Render Free** spins down after 15 minutes idle; bandwidth overage is billed if a card is on file.
- **Deno Deploy** is viable but has a smaller quota (1M requests/month) and an uncertain future: its site says "Deno is joining Cloudflare!"

**Alternative only if the owner later moves DNS:** move wethejamesphoto.com to a Cloudflare Free zone and use Workers with static assets. That unlocks the Rate Limiting binding, the one free WAF rate-limit rule, Workers Logs and Durable Objects without a second Worker. **Not recommended for v1.** It means re-creating every record for the Squarespace site and the Google Workspace email (MX, plus SPF/DKIM/DMARC TXT records if present), and a mistake takes down the owner's business email. [Likely]

---

## 3. Detailed findings

### 3.1 Cloudflare Workers Free / Pages Functions limits (read 2026-11-08)

Source: https://developers.cloudflare.com/workers/platform/limits/ ("Last updated Oct 29, 2026") unless noted.

| Limit | Workers Free value | Tag |
|---|---|---|
| Requests | **100,000/day**, account-wide, reset at **midnight UTC** | [Certain] |
| Over the daily limit | **Error 1027**. Routes can be set to fail open (bypass the Worker) or fail closed | [Certain] |
| CPU time per HTTP request | **10 ms**. Fetch, KV and DB waits **do not count**. An isolate has "some built-in flexibility" for infrequent overruns | [Certain] |
| Over CPU | **Error 1102** "Worker exceeded resource limits" | [Certain] |
| Memory | 128 MB per isolate | [Certain] |
| Subrequests per invocation | **50**. Each redirect hop counts. Cloudflare-internal services: 1,000 | [Certain] |
| Simultaneous connections waiting for headers | **6**; a 7th is queued | [Certain] |
| Duration (wall clock), HTTP | **No limit** while the client is connected. `waitUntil` adds ≤ 30 s | [Certain] |
| Response body size | "No enforced limit" (CDN cache limit 512 MB on Free) | [Certain] |
| Incoming request body | 100 MB on a Free zone (413 above) | [Certain] |
| URL size / request headers / response headers | 16 KB / 128 KB / 128 KB | [Certain] |
| Worker size / startup | 64 MiB uncompressed / 1 s | [Certain] |
| Env vars | 64 per Worker, 5 KB each | [Certain] |
| Workers per account | 100 | [Certain] |
| Cache API calls | 50 per request, shared with the subrequest quota | [Certain] |
| Logs | Workers Logs: 200,000 events/day, 3-day retention. **Not available to Pages.** From **2026-12-01** the unified Observability Free tier is 0.5 GB/day ingestion with 7-day retention, and on Free "Cloudflare stops ingesting new data when the account reaches its daily limit" (no billing) | [Certain] (https://developers.cloudflare.com/workers/platform/pricing/, https://developers.cloudflare.com/observability/pricing/) |

Pages-specific (https://developers.cloudflare.com/pages/platform/limits/, updated Sep 5, 2026):
- 500 builds/month and 1 concurrent build on Free; builds time out after 20 min.
- 100 custom domains per project; 20,000 files per site; 25 MiB per file.
- `_headers` max 100 rules; `_redirects` 2,000 static + 100 dynamic.
- 100 projects per account.
- "Requests to Pages functions count towards your quota for Workers plans."

[Certain]

Pages Functions pricing (https://developers.cloudflare.com/pages/functions/pricing/): "On both free and paid plans, requests to static assets are free and unlimited. A request is considered static when it does not invoke Functions." Also: "you could use 50,000 Functions requests and 50,000 Workers requests to use your full 100,000 daily request usage." [Certain]

Workers Static Assets (for comparison; https://developers.cloudflare.com/workers/static-assets/billing-and-limitations/): static requests are also free. On Free, routes covered by `run_worker_first` return **429** after the quota instead of falling back to assets. [Certain]

### 3.2 What happens at a limit: errors, never bills

- Workers Free: "When a Worker exceeds this limit, Cloudflare returns Error 1027." There is no overage price for Free; overage prices exist only on the $5/mo Standard (Paid) plan. [Certain] (limits and pricing pages)
- KV, Hyperdrive, D1 and Queues on Free behave the same way: "If you exceed any one of these limits, further operations of that type will fail with an error." [Certain] (pricing page)
- Pages Functions on Free: "Fail open means that static assets will continue to be served, even if Pages Functions would ordinarily have run first. Fail closed means an error page will be returned." [Certain] (https://developers.cloudflare.com/pages/functions/routing/)
- Browser Run on Free: 429 "Browser time limit exceeded for today" until the next UTC day. [Certain] (https://developers.cloudflare.com/browser-run/limits/)
- **Card on file:**
  - Community and Q&A sources (2026) say Workers Free, Pages, Turnstile and Web Analytics activate without a card, while R2 requires one. [Likely]
  - A Cloudflare community answer (Oct 2026) says: "Nothing on Free switches you to a paid plan automatically… having a payment method on the account doesn't change that." That is a community answer, not official text, so it is [Likely].
  - Safest rule: no card, no R2, no Paid plan.
- **Commercial use:** Cloudflare's Developer Platform service-specific terms contain no non-commercial restriction for Free. They do reserve the right to "temporarily limit … the number of requests" that put "undue burden" on the network, and to suspend for suspected violations. [Certain] (https://www.cloudflare.com/service-specific-terms-developer-platform/)

### 3.3 Pages vs. Workers: the decision driver

The Cloudflare doc "Migrate from Pages to Workers" (updated Oct 12, 2026) has a compatibility matrix with these rows:
- "Custom domains outside Cloudflare zones: Workers ❌ | Pages ✅"
- "Rate Limiting [binding]: Workers ✅ | Pages ❌"
- "Workers Logs: Workers ✅ | Pages ❌"
- "Durable Objects: Workers ✅ | Pages 🟡 (separate Worker + binding)"
- "Browser Run: ✅ both"
- "Service bindings: ✅ both"

The same doc says: "Unlike Pages, Workers does not support any domain whose nameservers are not managed by Cloudflare." [Certain]

Workers Custom Domains require "An active Cloudflare zone." [Certain] (https://developers.cloudflare.com/workers/configuration/routing/custom-domains/)

Pages: "If you are deploying to a subdomain, it is not necessary for your site to be a Cloudflare zone. You will need to add a custom CNAME record to point the domain to your Cloudflare Pages site." [Certain] (https://developers.cloudflare.com/pages/configuration/custom-domains/)

**Risk:** Cloudflare's own messaging now steers new projects to Workers; secondary sources quote the Pages docs opening with "Are you sure you want to use Pages?" [Likely]. Pages is not announced as deprecated, but feature work is going to Workers. If Pages is ever sunset, the migration path is either (a) moving DNS to Cloudflare or (b) a different host. Note this in the spec as a known platform risk. [Likely]

### 3.4 Rate-limiting options on this stack

| Option | Available on Pages-with-external-DNS? | Notes | Tag |
|---|---|---|---|
| Workers Rate Limiting binding | **No** (Pages ❌). Possible via a service binding to a separate Worker | Periods only 10 s or 60 s. Counts are per Cloudflare location and "approximate". **Open bug (workers-sdk #16097, Oct 2026):** several Free-plan users report `limit()` always returning `success: true`; it was labeled "internal" on Nov 4, 2026. Do not rely on it | [Certain] (docs and issue page) |
| WAF rate-limiting rule | **No** (needs the zone on Cloudflare) | Free plan: 1 rule, 10 s counting period, 10 s mitigation timeout, counted by IP, matching on path | [Certain] (https://developers.cloudflare.com/waf/rate-limiting-rules/) |
| Durable Object counter | Yes, via a separate Worker bound to Pages | Free: SQLite-backed DOs only; 100,000 DO requests/day; 13,000 GB-s/day; 5M rows read and 100k rows written per day; 5 GB. Strict and consistent. Set `workers_dev = false` on that Worker so it has no public URL | [Certain] (pricing page); [Likely] (design) |
| In-isolate memory counter | Yes | Best-effort only: per location, lost when the isolate is recycled. Zero storage | [Likely] |
| Turnstile per extraction | Yes | Makes each extraction cost one human-verified challenge. This is the real limiter | [Certain] |

### 3.5 Turnstile (bot check)

- **Plans** (https://developers.cloudflare.com/turnstile/plans/, updated Aug 14, 2026). Free:
  - "Up to 20 widgets"
  - "Unlimited challenges (traffic or verification requests)"
  - "10 hostnames per widget"
  - "Any hostname widget": No
  - Analytics lookback: 7 days
  - Ephemeral IDs: No
  - Offlabel (remove branding): No
  - "WCAG 2.2 AAA compliance": Yes

  [Certain]
- **Server-side validation** (https://developers.cloudflare.com/turnstile/get-started/server-side-validation/):
  - Endpoint: `POST https://challenges.cloudflare.com/turnstile/v0/siteverify`.
  - Token max length 2048 chars. "Validity period: 300 seconds". "Single use: Each token can only be validated once".
  - Response fields: `success, challenge_ts, hostname, error-codes, action, cdata`.
  - Docs say: "Validate the action and hostname when specified."
  - Optional `idempotency_key` for safe retries; optional `remoteip`.

  [Certain]
- **Privacy addendum** (https://www.cloudflare.com/turnstile-privacy-policy/, last updated June 18, 2025):
  - Turnstile processes "client IP address, TLS Fingerprint, User-Agent Header and Sitekey and associated origin" "solely to detect and block bots".
  - "Cloudflare is a data controller of Signals that we process to improve Turnstile's bot detection capabilities."

  This is a cross-dependency for the privacy policy. [Certain]
- **Design notes:**
  - Render the widget only when the user starts a URL scan. This matches PRIV-08 in the privacy research.
  - Use `action: 'extract'`.
  - Use the test sitekeys for local and preview builds. [Guessing] on exact test-key values; copy them from the docs.
  - The siteverify call is one subrequest from the Function. It is not an extra invocation.

### 3.6 Web Analytics and observability

- "Cloudflare Web Analytics provides free, privacy-first analytics … without changing your DNS or using Cloudflare's proxy." "does not collect or use your visitors' personal data." [Certain] (https://developers.cloudflare.com/web-analytics/about/)
- Product page: "does not use any client-side state, such as cookies or localStorage … We also don't 'fingerprint' individuals via their IP address, User Agent string, or any other data for the purpose of displaying analytics." [Certain] (https://www.cloudflare.com/web-analytics/)
- FAQ:
  - Soft limit of **10 sites per account**.
  - **6 months** of data.
  - Unsampled for 7 days, then aggregated to about 10%.
  - **Blocked by ad blockers.**
  - No UTM or query-string logging; no custom events.
  - Manual snippet needs `script-src https://static.cloudflareinsights.com` and `connect-src cloudflareinsights.com` in the CSP.
  - Pages one-click: Workers & Pages > project > Metrics > Enable.

  [Certain] (https://developers.cloudflare.com/web-analytics/faq/)
- **Cross-research flag:** the privacy report recommends no JS beacon in v1 (EDPB Guidelines 2/2023 on ePrivacy Art. 5(3)). The no-script option is the Pages project's Functions metrics, which show daily requests and errors. Because every extraction is exactly one `/api/extract` call, those metrics are an exact count of extractions. [Likely]

### 3.7 Browser Run (formerly Browser Rendering) as an optional fallback

- Workers Free limits:
  - "Browser hours: 10 minutes per day"
  - Concurrent browsers: 3
  - New instances: "1 every 20 seconds"
  - Browser timeout: 60 s
  - Quick Actions REST: "1 every 10 seconds"
  - `/crawl`: 5 jobs/day, 100 pages each

  Over the limit: 429 until the next UTC day. [Certain] (https://developers.cloudflare.com/browser-run/limits/, updated Oct 14, 2026)
- Paid comparison: 10 h/month included, then $0.09/h. Not applicable on Free. [Certain] (https://developers.cloudflare.com/browser-run/pricing/)
- Pages supports the Browser Run binding (matrix ✅). [Certain]
- Practical capacity at about 3–6 s of browser time per render: about 100–200 renders/day. [Guessing]; measure it. Recommendation: v1.1 at the earliest, behind its own button, with `browser.close()` in `finally`. Unclosed sessions keep burning minutes until timeout. [Certain] (FAQ)

### 3.8 Alternatives (read 2026-11-08)

| Host | Free allowance relevant here | At the limit | Commercial or other restriction | Card | Verdict | Tag |
|---|---|---|---|---|---|---|
| **Vercel Hobby** | 1,000,000 function invocations/mo; 4 CPU-hrs Active CPU; 360 GB-hrs memory; 1M CDN requests; 100 GB fast data transfer | "if you exceed your usage limits on the Hobby plan, you will have to wait until 30 days have passed before you can use the feature again" | "Hobby teams are restricted to non-commercial personal use only." Commercial = "any Deployment that is used for the purpose of financial gain of anyone involved…", with examples including "Advertising the sale of a product or service." A footer promoting the owner's design/marketing services is a plausible violation. **The tool does not qualify cleanly** | Not needed for Hobby | Reject | [Certain] on quotes (https://vercel.com/docs/limits/fair-use-guidelines, https://vercel.com/docs/plans/hobby); [Likely] on interpretation |
| **Netlify Free** (credit-based) | 300 credits/mo, hard limit. Web requests 2 credits/10k; bandwidth 20 credits/GB; functions 10 credits/GB-hr; **production deploy 15 credits each** | "Once your credit balance is completely used up, all of your web projects (sites/apps) are paused and visitors … will find a Site not available page" | None noted | Not needed for Free | Reject: 20 deploys exhaust the month, and failure takes the whole site down | [Certain] (https://docs.netlify.com/manage/accounts-and-billing/billing/billing-for-credit-based-plans/credit-based-pricing-plans/, …/how-credits-work/) |
| **GitHub Pages** | Static only. 1 GB site; soft 100 GB/mo bandwidth; soft 10 builds/hr | May stop serving; "polite email" | Not for "online business, e-commerce … or SaaS" | No | Could host the static UI but not the proxy. No benefit over Pages | [Certain] (https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits) |
| **Deno Deploy Free** | 1M requests/mo; 20 GiB egress; 10 h active CPU; 150 GiB-hr memory; 5 custom domains | Not stated on the pricing page | "For personal use and smaller projects". Banner: "Deno is joining Cloudflare!" | Not stated | Viable backup; future uncertain | [Certain] (https://deno.com/deploy/pricing); [Guessing] on limit behavior |
| **Render Free** | 750 instance-hours/mo; spins down after 15 min idle | Hours exceeded: services suspended until next month. Bandwidth: "Render bills you" if a payment method is on file, otherwise suspends | — | Optional, but a card enables overage bills | Reject (cold starts; card risk) | [Certain] (https://render.com/docs/free) |
| **Fly.io** | "New organizations don't have a free tier"; $1 / 7-day trial | — | — | "All organizations … require a credit card on file" | Reject: **card required** | [Certain] (https://docs.fly.io/about/pricing) |

### 3.9 DNS: serving the app at `<sub>.wethejamesphoto.com`

**Live state on 2026-11-08** (queried via dns.google and Verisign RDAP):

| Check | Result | Tag |
|---|---|---|
| Registrar | Squarespace Domains LLC (IANA 3827). Registered 2025-08-24; **expires 2027-08-24**; transfer lock on | [Certain] |
| NS | `ns-cloud-d1/d2/d3/d4.googledomains.com`. SOA `cloud-dns-hostmaster.google.com` | [Certain] |
| Who edits DNS | Squarespace's default DNS runs on Google Cloud DNS nameservers (`ns-cloud-*.googledomains.com`), so records are edited in the Squarespace domains dashboard | [Likely] (Squarespace forum threads; dnsspy). Confirm in the dashboard that it shows Squarespace nameservers, not "custom nameservers" |
| DNSSEC | Not signed (no DS; RDAP `delegationSigned: false`) | [Certain] |
| CAA | **None**, so any CA may issue | [Certain] |
| Wildcard | None (random subdomain returns NXDOMAIN) | [Certain] |
| MX | `aspmx.l.google.com` + alt1–alt4 (Google Workspace) | [Certain] |
| `www` | CNAME to `ext-sq.squarespace.com` | [Certain] |

**Squarespace supports custom CNAMEs on subdomains.** The help article "DNS records for web hosting" (updated July 14, 2026) gives these steps:

1. Open the domains dashboard.
2. Click the domain > **DNS** > **DNS Settings**.
3. Scroll to **Custom Records** > **Add record**.
4. Re-enter your password or 2FA.
5. Type **CNAME**; **Name** = the subdomain label ("It's not possible to add a CNAME with @"); **Data** = target hostname ("can't contain special characters such as slashes (/) or colons (:)").
6. **Save**.

If the domain were connected via **DNS Connect**, records would be edited at the third-party provider instead. [Certain] (https://support.squarespace.com/hc/en-us/articles/31119879125645-DNS-records-for-web-hosting)

**How Pages verifies the domain without moving nameservers:**
- Pages issues the certificate by **HTTP validation**: "Pages uses HTTP validation and needs to hit an HTTP endpoint during validation." No TXT record is needed.
- The custom domain shows **Verifying** until validation succeeds.
- If it is still verifying after 15 minutes, check (a) that nothing redirects `/.well-known/acme-challenge/*` and (b) CAA records.

[Certain] (https://developers.cloudflare.com/pages/configuration/debugging-pages/)

**Exact steps (recommended order):**
1. Create the Cloudflare account. Add no card; enable 2FA.
2. Workers & Pages > Create > Pages > **Connect to Git** > `jjamesjgw/brand-kit-extractor`. Set the production branch to `main` and the build command and output directory. Under Settings > Builds, set preview deployments to **None** or restrict them.
3. Deploy once and confirm that `https://<project>.pages.dev` loads.
4. Pages project > **Custom domains** > **Set up a domain** > enter `<sub>.wethejamesphoto.com` > Continue. Cloudflare shows the CNAME to add.
5. Squarespace: domains dashboard > wethejamesphoto.com > DNS > DNS Settings > Custom Records > Add record > CNAME, Name `<sub>`, Data `<project>.pages.dev` > Save.
6. Wait for Cloudflare to show **Active**. Verify the record and the response headers:
   ```
   dig +short CNAME <sub>.wethejamesphoto.com
   curl -sI https://<sub>.wethejamesphoto.com
   ```
7. Turnstile: create a widget whose hostnames list contains `<sub>.wethejamesphoto.com`. Put the secret in Pages encrypted variables.
8. Pages > Settings > Runtime > **Fail open**.
9. **Do not** change nameservers, the apex A records, the `www` CNAME, MX or TXT records. Do not enable DNSSEC changes or URL forwarding on `<sub>`.

**If DNS turns out to be somewhere else** (another registrar, a Google Cloud DNS zone the owner controls, or Squarespace "DNS Connect"):
- Run `dig NS wethejamesphoto.com +short`. The provider that hosts those nameservers is where the CNAME goes.
- Check that no record already exists at `<sub>`. A CNAME cannot share a name with any other record type.
- Check for a wildcard (`*`) record that might mask mistakes.
- Check for **CAA** records. If any exist, add `0 issue "letsencrypt.org"`, `0 issue "pki.goog; cansignhttpexchanges=yes"`, `0 issue "ssl.com"` (plus `issuewild` versions).
- Check that registrar features like "URL forwarding" or "parking" are not applied to the subdomain.
- If the provider rejects a trailing dot or the FQDN in the Name field, enter just the label.
- If DNS is already on Cloudflare (full setup), Pages adds the record automatically, and Workers Custom Domains become an option. [Certain] (Pages docs on auto-added CNAME)
- Partial (CNAME) Cloudflare setup is a paid-plan feature and not usable at $0. [Likely]

**Gotcha:** "Once a custom domain is set up, if you change the DNS entry to point to something else … the custom domain will become inactive." Repointing it back causes errors until it re-validates. [Certain] (Pages custom-domain docs)

### 3.10 SSRF: what the platform does and what the code must do

**Platform behavior:**
- **IP literals:** "For Workers subrequests, requests can only be made to URLs, not to IP addresses directly." The failure appears as error 1003 "Direct IP access not allowed". Error 1024 is "Worker cannot make a subrequest to a Cloudflare-owned IP address." [Certain] (https://developers.cloudflare.com/workers/platform/known-issues/; errors table in limits/observability docs; community thread 595868)
- **Ports:** `allow_custom_ports` has been the default since 2024-09-02. "When you make a subrequest to a website that does not use Cloudflare ('Grey Clouded') - any port can be specified." For Cloudflare-proxied sites, only Cloudflare-supported ports are honored. **So arbitrary ports are reachable on non-Cloudflare hosts. The code must allowlist ports.** [Certain] (https://developers.cloudflare.com/workers/configuration/compatibility-flags/)
- **Private IPs via hostnames** (for example a DNS name resolving to 127.0.0.1 or 10.x): Cloudflare publishes no guarantee. A Nov 2, 2026 community thread asking for exactly that policy got a DNS-resolution error (1001) in a loopback probe but no official statement. Workers egress is Cloudflare's network, not a VPC with the owner's services, so there is no internal network of the owner's to reach. The residual risk is Cloudflare-side surfaces and third parties. [Likely]
- **No DNS pinning:** Workers have no `dns.lookup` or connect hook. Portkey's docs: "On Cloudflare Workers, request-time validation still applies; connection-time DNS pinning is a Node.js-only hardening layer." Vercel's AI SDK notes say the same for workerd. [Certain] that it is unavailable.
- **Metadata endpoints** (169.254.169.254 and similar): unreachable as IP literals (rejected above). Cloud metadata services do not exist on Cloudflare's edge. [Likely]
- **Same-zone fetch:** with `global_fetch_strictly_public`, fetches to your own zone go through the front door. Without it, they hit the origin directly and bypass security settings. For Pages on a CNAME subdomain this is mostly moot; enable it anyway. [Certain] (flag text)
- **The target sees the caller:** Cloudflare adds a `CF-Worker` header with "the name of the zone which owns the Worker" to every subrequest. Sites can block Worker traffic with WAF field `cf.worker.upstream_zone`. Expect some Cloudflare-protected sites to refuse or challenge the fetch, and show the uploads fallback. [Certain] on the header; [Guessing] on what the zone value is for a Pages project (probably `pages.dev` or the project hostname).

**Guard sketch** (load-bearing logic; the TypeScript is illustrative):

```ts
const PORTS = new Set(['', '80', '443']);
const BAD_SUFFIX = ['.localhost','.local','.internal','.intranet','.lan','.home','.home.arpa',
  '.corp','.test','.invalid','.example','.onion','.arpa','.pages.dev'];
const REBIND = ['nip.io','sslip.io','xip.io','localtest.me','lvh.me','1u.ms','traefik.me'];

function checkUrl(raw: string, selfHost: string): URL {
  const u = new URL(raw);                      // WHATWG parse; numeric IPv4 forms normalize to dotted-quad
  if (u.protocol !== 'https:' && u.protocol !== 'http:') throw 'scheme';
  if (u.username || u.password) throw 'credentials';
  if (!PORTS.has(u.port)) throw 'port';
  const h = u.hostname.toLowerCase().replace(/\.$/, '');
  if (h.startsWith('[') || /^\d{1,3}(\.\d{1,3}){3}$/.test(h)) throw 'ip-literal';
  if (h.length > 253 || !h.includes('.') || h === 'localhost') throw 'host';
  if (BAD_SUFFIX.some(s => h.endsWith(s)) || REBIND.some(s => h === s || h.endsWith('.' + s))) throw 'host';
  if (h === selfHost) throw 'self';
  u.hash = '';
  return u;
}

async function guardedFetch(start: URL, kind: 'html'|'css'|'img', budget: {left: number}) {
  let url = start;
  for (let hop = 0; hop <= 5; hop++) {
    if (budget.left-- <= 0) throw 'subrequest-budget';           // 50/invocation hard cap incl. hops
    const res = await fetch(url.toString(), {
      method: 'GET', redirect: 'manual', cache: 'no-store',
      headers: { 'User-Agent': UA, 'Accept': ACCEPT[kind], 'Accept-Language': 'en' },
      signal: AbortSignal.timeout(8000),
    });
    if (res.status >= 300 && res.status < 400) {
      const loc = res.headers.get('Location'); await res.body?.cancel();
      if (!loc) throw 'redirect';
      url = checkUrl(new URL(loc, url).toString(), SELF_HOST);  // re-validate every hop
      continue;
    }
    if (!res.ok) { await res.body?.cancel(); throw `upstream-${res.status}`; }
    const ct = (res.headers.get('Content-Type') ?? '').split(';')[0].trim().toLowerCase();
    if (!TYPES[kind].has(ct)) { await res.body?.cancel(); throw 'type'; }
    if (Number(res.headers.get('Content-Length') ?? 0) > MAX[kind]) { await res.body?.cancel(); throw 'size'; }
    return { url, ct, body: res.body!.pipeThrough(byteCap(MAX[kind])) }; // TransformStream that errors past cap
  }
  throw 'too-many-redirects';
}
```

Ranges to reject in the optional DoH pre-check (INFRA-19):
- IPv4: `0/8, 10/8, 100.64/10, 127/8, 169.254/16, 172.16/12, 192.0.0/24, 192.0.2/24, 192.88.99/24, 192.168/16, 198.18/15, 198.51.100/24, 203.0.113/24, 224/4, 240/4`.
- IPv6: `::/128, ::1/128, ::ffff:0:0/96` (check the embedded v4), `64:ff9b::/96` (check embedded), `100::/64, 2001:db8::/32, 2002::/16` (check embedded), `fc00::/7, fe80::/10, ff00::/8`.

[Likely] (standard IANA special-purpose registries; Portkey's list matches)

### 3.11 Keeping the endpoint from becoming an open proxy, and protecting the domain

| Abuse | Control | Tag |
|---|---|---|
| Scripted mass use, quota drain | Fresh Turnstile per `/api/extract`; tokens are single-use with a 300 s TTL | [Certain] |
| Using the image endpoint as a general fetcher | HMAC ticket lists only server-discovered URLs; 120 s TTL; image types only; ≤ 1 MB | [Likely] |
| Cross-site use from other origins | POST + JSON + Origin/Sec-Fetch-Site check + no CORS headers | [Likely] |
| Phishing or malware served from the tool's domain | Never return upstream HTML/SVG as renderable types. JSON strings and octet-stream only. CSP `sandbox` + `nosniff` + `Content-Disposition` not needed because nothing renders | [Likely] |
| Laundering requests to attack a third party | Per-extraction caps (≤ 1 page, ≤ 10 CSS, ≤ 5 imports, ≤ 8 images, ≤ 5 hops each); 8 s timeouts; 20 s total; honest UA naming the tool | [Likely] |
| Preview/pages.dev endpoints bypassing the hostname check | Host check; previews off; Turnstile hostname list contains only the production host | [Likely] |
| Storage or caching of third-party content | `cache: 'no-store'`; no KV/R2/DO writes of content; no logging of URLs | [Certain] on `no-store` semantics |

### 3.12 Capacity math

**Inputs:** quota Q = 100,000 Function invocations/day/account, all `/api/*` combined. Static loads cost 0. [Certain]

| Design | Invocations per extraction | Extractions/day at Q | Notes |
|---|---|---|---|
| A. Per-resource proxy (1 page + N CSS + M images), typical N=4, M=4 | 9 | 11,111 | Simplest, but every CSS file is a call |
| A, worst case N=10, M=6 | 17 | 5,882 | |
| **B. Recommended:** `/api/extract` (HTML+CSS) + one `/api/image` per image, typical M=4 | **5** | **20,000** | CSS is fetched in parallel inside one invocation (≤ 6 concurrent) |
| B, worst case M=8 | 9 | 11,111 | |
| C. Single bundle incl. images | 1 | ≤ 100,000 | Binary framing needed; highest CPU-overrun risk; consider only after measuring CPU |

The figures are arithmetic: Q ÷ invocations. Retries and failed attempts reduce them. [Certain] (arithmetic on documented quota)

**Subrequest budget, design B, one `/api/extract` invocation:**

| Item | Subrequests |
|---|---|
| siteverify | 1 |
| HTML, ≤ 5 hops | 5 |
| 10 CSS files × ≤ 2 hops | 20 |
| ≤ 5 `@import`s × ≤ 2 hops | 10 |
| **Total** | **36** |

Adding the optional DoH pre-check (+2) gives 38, still under 50. Each `/api/image` call uses ≤ 6 (5 hops + 1). [Certain] (limit); [Likely] (design)

**CPU:** fetch waits do not count. `HTMLRewriter` is a native streaming parser, so one pass over ≤ 2 MB of HTML is expected to stay well under 10 ms. The bounded `@import` regex over CSS is cheap. Base64 is the expensive part, so design B streams image bytes untouched. [Guessing] on exact ms; measure with real-time logs (`wrangler pages deployment tail`) or locally before launch.

**Traffic spike:** Cloudflare has "no general limit on requests per second." A spike drains the day's quota early. After that, `/api/*` returns 1027 (or falls through to static content with fail-open) until 00:00 UTC. No charge accrues. The UI and the upload features keep working because static requests are free and unlimited. [Certain]

Illustrative scenario (hypothetical, not a forecast): if a post sends 30,000 visitors in a day and one-third run a scan, design B needs about 50,000 invocations, which fits. Design A needs about 90,000 typical, near the limit, and exceeds it in the worst case.

**Graceful mode:** see INFRA-30–32. The reset is 00:00 UTC, which is 18:00 CST (UTC−6, Nov 1 2026 – Mar 14 2027) or 19:00 CDT. Let the browser render local time. [Likely] on US DST dates.

### 3.13 CI/CD and GitHub

- **Actions billing:** "The following usage remains free independently and does not reduce the included compute usage: Standard GitHub-hosted runners in public repositories." And: "If your account does not have a valid payment method on file, usage is blocked once you consume the included compute usage. Larger runners require a valid payment method." [Certain] (https://docs.github.com/en/billing/concepts/product-billing/github-actions)
- **Dependabot:** runs on GitHub Actions runners. "For Dependabot on larger runners, GitHub will bill" at the regular rate, so keep it on standard runners. [Certain] on the quote (https://docs.github.com/en/code-security/concepts/supply-chain-security/dependabot-on-actions); [Likely] that standard runners in a public repo are $0.
- **Pages deploy method:**
  - "If you choose Direct Upload, you cannot switch to Git integration later." "If you deploy using the Git integration, you cannot switch to Direct Upload later." [Certain] (https://developers.cloudflare.com/pages/get-started/direct-upload/, …/git-integration/)
  - Recommended: Git integration, so no Cloudflare token sits in GitHub.
  - Dependabot PRs trigger preview builds, which likely count toward 500/month [Likely]. Weekly grouped updates keep this small, or turn previews off.
- **Font list for the picker:** fetch the Google Fonts Developer API (`googleapis.com/webfonts/v1/webfonts`, needs a free API key) in a scheduled Action. Store the key as an Actions secret and commit the generated JSON. Never call the API from the browser. [Likely]; the key-requirement claim is from secondary sources, so verify the quota.

### 3.14 Cost traps (all avoidable)

| Trap | How it bites | Guard | Tag |
|---|---|---|---|
| Workers Paid ($5/mo minimum) | One click in the dashboard. Paid usage then "continues automatically after an allowance is exhausted" for logs/traces from 2026-12-01 | Never upgrade; no card | [Certain] |
| R2 | Requires a payment method; usage-billed | Do not use; nothing needs object storage | [Likely] |
| Browser Run beyond 10 min/day | Paid plan only | Stay on Free; 429 is the cap | [Certain] |
| Turnstile Enterprise features | Ephemeral IDs and offlabel are Enterprise only | Not needed | [Certain] |
| Vercel/Render overage | Render bills bandwidth if a card is on file | Not used | [Certain] |
| Netlify credits | 15 credits per production deploy; site paused at 0 | Not used | [Certain] |
| GitHub larger runners / Codespaces / LFS | Billable | Standard runners only; no LFS | [Certain] (runners) / [Likely] (others) |
| Google Fonts API | Free key and quota | Build-time only | [Likely] |
| Domain | Squarespace renewal for wethejamesphoto.com; **expires 2027-08-24** | Owner already pays; keep auto-renew on | [Certain] (date) |
| Second project on the same Cloudflare account | Shares the 100k/day quota | Dedicated account | [Certain] (account-wide quota) |

### 3.15 Conflicts and flags against locked decisions and companion research

1. **"Stores nothing" (competitor wording) vs. infrastructure.** No hard conflict, but the wording must be exact.
   - Turnstile makes Cloudflare a **controller** of bot signals, including IP.
   - Cloudflare edge logs exist outside the owner's control.
   - Real-time function logs are ephemeral; Pages has no persisted Workers Logs.
   - With `cache: 'no-store'` and no KV/DO content writes, the tool itself stores nothing.

   Recommended claim: "We don't store your inputs or results." Disclose Cloudflare and Turnstile in the privacy policy. [Likely]
2. **Fonts same-origin (privacy PRIV-02) vs. capacity.** Self-host as static files, not through a Function proxy; static is free and unlimited, while a Function proxy would consume the shared 100k/day. Watch the 20,000-file limit per Pages site. [Certain] (limits); [Guessing] (file count)
3. **Logo bytes through the server (privacy PRIV-03).** This matches design B. Do not hotlink images directly from the target site in the browser, even where it sends CORS headers. [Likely]
4. **Platform direction.** Cloudflare now prefers Workers for new projects. Pages is the only $0 Cloudflare option that keeps DNS at Squarespace. This is a known, accepted risk, not a blocker. [Likely]
5. **Web Analytics.** Researched as asked, but the privacy research recommends no beacon in v1. The no-script alternative is Functions metrics, which count extractions exactly. [Likely]
6. **No hard conflicts found** with the $0, TCPA, no-PII or worldwide requirements in this research area. [Likely]

---

## 4. Unverified / needs follow-up

1. **Card-free signup for Workers Free, Pages and Turnstile.** Community and Q&A sources say no card is needed, but no official Cloudflare page states "no card required." Verify during account creation. [Likely]
2. **Fail-open response shape.** What exactly a POST to `/api/extract` returns when the Functions quota is exhausted and fail-open is on (static 404, 405, or the SPA's `index.html`). The client heuristic in INFRA-30 covers all three, but test it by temporarily setting fail closed/open on a test project. [Guessing]
3. **CPU per `/api/extract`.** Real CPU ms for HTMLRewriter over 0.3–2 MB pages plus `@import` scanning. This must be measured; if p95 is above about 8 ms, cut the HTML cap or move CSS discovery to the client. [Guessing]
4. **Rate Limiting binding bug** (workers-sdk #16097, open, "internal" label since 2026-11-04). Re-check before adopting even via a service binding. [Certain] that it is open as of today.
5. **Hostname → private IP behavior on Cloudflare's edge.** No official policy; the Nov 2, 2026 community question was unanswered when read. [Likely] that it is unreachable/irrelevant, but do not rely on it.
6. **`CF-Worker` header value for Pages Functions** (`pages.dev` vs. the project hostname vs. none). Matters for which sites block the fetch. [Guessing]
7. **Squarespace DNS panel control.** Confirm in the dashboard that wethejamesphoto.com shows "Squarespace nameservers" with editable Custom Records. Some forum threads describe panels showing Google NS without zone access. [Likely]
8. **Pages preview builds from Dependabot or fork PRs.** Whether they count toward 500 builds/month and whether fork PRs build at all. Set previews to None to make this moot. [Likely]
9. **Turnstile test sitekey and secret values.** Copy them from the docs at build time. [Guessing] on exact strings.
10. **Google Fonts Developer API key requirement and quota** (secondary sources say 10,000 requests/day per key). Verify on developers.google.com. [Likely]
11. **Whether Cloudflare will deprecate Pages.** No date announced; watch the Cloudflare changelog. [Likely] no near-term deprecation.
12. **DO free-tier quota independence.** Whether the 100,000 DO requests/day is separate from the 100,000 Worker requests/day. The pricing table lists them separately, and a Pages→Worker service-binding call should not count as a second request ("do not incur additional request fees"), but "fees" is billing language. Test this before relying on the DO upgrade path. [Likely]
13. **Partial (CNAME) Cloudflare zone setup plan requirement** (believed Business/Enterprise only). Not needed for the recommended path. [Likely]

---

## 5. Sources (all accessed 2026-11-08)

**Cloudflare**
- Workers limits: https://developers.cloudflare.com/workers/platform/limits/
- Workers pricing: https://developers.cloudflare.com/workers/platform/pricing/
- Static assets billing: https://developers.cloudflare.com/workers/static-assets/billing-and-limitations/
- Pages limits: https://developers.cloudflare.com/pages/platform/limits/
- Pages Functions pricing: https://developers.cloudflare.com/pages/functions/pricing/
- Pages Functions routing (`_routes.json`, fail open/closed): https://developers.cloudflare.com/pages/functions/routing/
- Pages Functions bindings: https://developers.cloudflare.com/pages/functions/bindings/
- Pages custom domains: https://developers.cloudflare.com/pages/configuration/custom-domains/
- Debugging Pages (HTTP validation, CAA): https://developers.cloudflare.com/pages/configuration/debugging-pages/
- Direct Upload: https://developers.cloudflare.com/pages/get-started/direct-upload/
- Migrate from Pages to Workers + compatibility matrix: https://developers.cloudflare.com/workers/static-assets/migration-guides/migrate-from-pages/
- Workers Custom Domains: https://developers.cloudflare.com/workers/configuration/routing/custom-domains/
- Workers known issues (fetch to IP addresses): https://developers.cloudflare.com/workers/platform/known-issues/
- Workers fetch API (`cache` modes): https://developers.cloudflare.com/workers/runtime-apis/fetch/
- Compatibility flags (`allow_custom_ports`, `global_fetch_strictly_public`): https://developers.cloudflare.com/workers/configuration/compatibility-flags/
- Rate Limiting binding: https://developers.cloudflare.com/workers/runtime-apis/bindings/rate-limit/
- Rate Limiting binding bug: https://github.com/cloudflare/workers-sdk/issues/16097
- WAF rate limiting rules: https://developers.cloudflare.com/waf/rate-limiting-rules/
- Turnstile plans: https://developers.cloudflare.com/turnstile/plans/
- Turnstile server-side validation: https://developers.cloudflare.com/turnstile/get-started/server-side-validation/
- Turnstile privacy addendum: https://www.cloudflare.com/turnstile-privacy-policy/
- Web Analytics about: https://developers.cloudflare.com/web-analytics/about/
- Web Analytics FAQ: https://developers.cloudflare.com/web-analytics/faq/
- Web Analytics product page: https://www.cloudflare.com/web-analytics/
- Browser Run limits: https://developers.cloudflare.com/browser-run/limits/
- Browser Run pricing: https://developers.cloudflare.com/browser-run/pricing/
- Observability pricing (from 2026-12-01): https://developers.cloudflare.com/observability/pricing/
- Developer Platform service-specific terms: https://www.cloudflare.com/service-specific-terms-developer-platform/
- `CF-Worker` header: https://developers.cloudflare.com/fundamentals/reference/http-headers/
- Blocking Worker subrequests: https://developers.cloudflare.com/waf/custom-rules/use-cases/block-worker-subrequests/

**Cloudflare community / secondary**
- Fetch with IP hostname (error 1003): https://community.cloudflare.com/t/exception-when-fetching-url-with-ip-hostname-in-workers/595868
- Workers global-fetch private-address / DNS rebinding question (Nov 2, 2026): https://community.cloudflare.com/t/workers-global-fetch-connection-time-private-address-and-dns-rebinding-policy/975978
- Rate Limiting binding on Free plan: https://community.cloudflare.com/t/are-workers-rate-limiting-api-bindings-free-on-the-workers-free-plan/971323
- Free products that require a card: https://agents.stackoverflow.com/questions/14a78294-fd6c-4da1-a386-cc4a819bd63d
- Portkey (no DNS pinning on Workers; blocklists): https://docs.portkey.ai/docs/product/ai-gateway/custom-hosts

**Other hosts**
- Vercel fair use guidelines: https://vercel.com/docs/limits/fair-use-guidelines
- Vercel Hobby plan: https://vercel.com/docs/plans/hobby
- Netlify credit-based plans: https://docs.netlify.com/manage/accounts-and-billing/billing/billing-for-credit-based-plans/credit-based-pricing-plans/
- Netlify, how credits work: https://docs.netlify.com/manage/accounts-and-billing/billing/billing-for-credit-based-plans/how-credits-work/
- GitHub Pages limits: https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits
- Deno Deploy pricing: https://deno.com/deploy/pricing
- Render free tier: https://render.com/docs/free
- Fly.io pricing: https://docs.fly.io/about/pricing

**GitHub**
- GitHub Actions billing: https://docs.github.com/en/billing/concepts/product-billing/github-actions
- Dependabot on GitHub Actions: https://docs.github.com/en/code-security/concepts/supply-chain-security/dependabot-on-actions

**DNS and Squarespace**
- Squarespace DNS records: https://support.squarespace.com/hc/en-us/articles/31119879125645-DNS-records-for-web-hosting
- Squarespace nameserver forum thread: https://forum.squarespace.com/topic/338481-dns-panel-shows-google-nameservers-but-says-using-squarespace-nameservers-no-access-to-google-dns-zone/
- Live DNS queries (NS/SOA/MX/CAA/DS/www/wildcard): `https://dns.google/resolve?name=wethejamesphoto.com&type=…`
- RDAP: https://rdap.verisign.com/com/v1/domain/wethejamesphoto.com
