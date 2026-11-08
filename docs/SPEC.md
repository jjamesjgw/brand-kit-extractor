# Kitsheet v1: build spec

| | |
|---|---|
| Status | Draft for build, revision 2: revised after adversarial review (legal, cost, security, feasibility, product). Every finding and its disposition is in Appendix A. |
| Date | 2026-11-08 |
| Owner / operator | Justin James (individual, Rowlett, Texas, USA) |
| Repo | github.com/jjamesjgw/brand-kit-extractor (public) |
| Working name | **Kitsheet** (recommendation in §12.10; owner must confirm). Every place the name or host appears is a build constant: `TOOL_NAME`, `PROD_HOST`, `SHARE_ORIGIN`, `ROBOTS_TOKEN`, `CONTACT_EMAIL`. |
| Working host | `kitsheet.wethejamesphoto.com` (written `<host>` below) |
| Inputs | Five research reports dated 2026-11-08 (privacy law; IP and abuse law; $0 infrastructure; extraction engine; outputs and sharing). Research IDs such as PRIV-08 or EXT-19 are cited for traceability only. This spec restates every requirement it adopts, so it stands on its own. |

**Evidence tags.** Claims about law, platform limits and browser compatibility carry a tag:
- **[Certain]**: a primary source was read on or about 2026-11-08, or the behavior was measured.
- **[Likely]**: strong inference or consistent secondary sources.
- **[Guessing]**: a gap filled with judgment. Every [Guessing] number is a starting value to tune or verify before launch.

Sources are reference-style links collected at the end of the document.

**Not legal advice.** The legal analysis here comes from research, not from a licensed attorney. §9.9 lists the specific items worth a one-time attorney review. That review is optional; nothing in this spec assumes it happens.

**Words this spec and the product never use:** empower, seamless, unlock, elevate, leverage. The product never invents numbers or statistics.

---

## 0. Decisions log

### 0.1 Locked interview decisions

These are fixed. The spec builds within them.

| # | Decision |
|---|---|
| D1 | Product: a free web tool that builds a brand kit. The user pastes a website URL and/or uploads brand materials; the tool produces a polished brand kit. |
| D2 | Reference competitor: Blendin Brand Kit Extractor ([blendin]). Match its extraction bar: parsed stylesheets and tokens, painted-color weighting, up to 5 role-labeled colors, never invent a color, heading and body fonts with load-source label, ranked logos and favicon, name and tagline from metadata, WCAG AA/AAA checks, a human check, store nothing. |
| D3 | Operator: Justin James, an individual sole proprietor in Rowlett, Texas, USA. Not an LLC. |
| D4 | Purpose: portfolio and credibility piece. No lead generation, no upsell, no monetization. |
| D5 | Free for users. |
| D6 | $0 cost to the owner, forever. Free tiers only. No surprise billing. |
| D7 | TCPA compliant. |
| D8 | Never capture email addresses from children. |
| D9 | "Legally sound everywhere": worldwide audience. |
| D10 | Data capture: none. No accounts, no email, no phone, no contact forms, no PII fields. Fully anonymous. |
| D11 | Social media metrics: dropped from v1. |
| D12 | AI/LLMs: none. Deterministic, rules-based extraction and generation only. |
| D13 | Input (a): website URL, fetched and extracted. |
| D14 | Input (b): logo upload (SVG, PNG, JPG). Becomes the kit logo; colors are derived from it; SVG gets light, dark and one-color variants. |
| D15 | Input (c): image to palette. Upload any brand image or screenshot; pixels are quantized. |
| D16 | Input (d): existing brand-guide PDF. Parsed for hex codes and font names. |
| D17 | Input (e): manual entry and overrides (hex codes, Google Fonts picker, name, tagline). |
| D18 | All uploads are processed in the user's browser and never sent to a server. |
| D19 | Output 1: multi-page PDF brand guide. |
| D20 | Output 2: PNG brand board. |
| D21 | Output 3: shareable link. The kit is compressed into the URL `#fragment`; it is never sent to the server and never stored. |
| D22 | Raster uploaded logos cannot ride in the link. Small SVG logos can. |
| D23 | Design tokens / code export: not selected. May be noted as a cheap optional extra. |
| D24 | Kit basics: logo and favicon, colors with hex, heading and body fonts, brand name, tagline. |
| D25 | Color system: tint and shade ramps per brand color, WCAG contrast grid, approved text-on-background pairs. |
| D26 | Typography system: type scale, heading and body specimens, font source, fallbacks. |
| D27 | Logo usage rules: clear space, minimum size, on-light and on-dark versions, don'ts. |
| D28 | Social-ready assets: not selected. |
| D29 | Audience: everyone who needs a brand kit: small business owners (for example DFW home services such as roofers and junk removal), freelancers and agencies, in-house marketers. |
| D30 | Hosting: a subdomain of wethejamesphoto.com (the owner's Squarespace wedding-photography site; DNS likely on Squarespace Domains). |
| D31 | The tool gets its own standalone name. Footer credit "Built by Justin James" links to the owner's portfolio. |
| D32 | Repo: github.com/jjamesjgw/brand-kit-extractor, public (so GitHub Actions minutes are free). |
| D33 | Design directives: flat; deliberate negative space; one strong layout idea (asymmetry, dominant headline, an intentional off-grid moment); clean utilitarian type with real size and weight contrast; restrained neutral palette with one accent used deliberately; editorial "film look" if imagery is used. |
| D34 | Design bans: glowing neon, heavy gradients, glassmorphism or blur, floating 3D isometric icons, uniform rounded corners plus drop shadows on everything, centered symmetric layouts, emoji as icons, three evenly spaced feature cards under a hero, Corporate Memphis illustrations, glossy stock imagery, vague hedging copy. |
| D35 | Copy is direct and specific. Banned words: empower, seamless, unlock, elevate, leverage. Never invent numbers or stats. |
| D36 | Claims are tagged [Certain], [Likely] or [Guessing]. |

### 0.2 Conflicts between the research reports, and how this spec resolves them

| # | Conflict | Resolution | Why |
|---|---|---|---|
| C1 | **Fetch API shape.** Infra: `/api/extract` + `/api/image` with a signed list of server-discovered image URLs. Extraction: `/api/page` + `/api/assets`, where the client may request any URL on the page's registrable domain or a platform-CDN list. | Infra principle, with both reports' host limits: the server discovers up to 32 candidate image URLs on the page's own hosts, its CSS hosts or a curated platform-CDN list (§5.2.4), and signs their hashes into a ticket bound to the caller's network prefix. The client ranks them with full DOM context and fetches only URLs in the ticket, in one batched `/api/images` call (C19). | The top infra risk is the endpoint becoming a general proxy on the owner's business domain. A hash list closes that without moving ranking logic to the server. CSS-background logos are covered by a cheap server-side scan (§5.2.4). |
| C2 | **Per-analysis caps.** IP: ≤ 20 subrequests, ≤ 5 MB, 10 s. Infra: about 36 subrequests and a 20 s deadline. Extraction: the owner's own Squarespace homepage links 16 stylesheets. | Hard cap of 45 subrequests per `/api/extract` call (platform limit 50), counted by one `budget.take(kind)` wrapper that also counts the Turnstile siteverify call and every DoH lookup, filled in a fixed priority order; anything over budget is skipped and reported. Byte caps: HTML 2 MB, CSS 3 MB total, ≤ 5 images. Deadline 15 s. | robots.txt per origin plus platform CSS sprawl make 20 unworkable. The IP report marked its caps [Guessing] pending reconciliation. The burden argument still holds: the fetcher requests a strict subset of what a browser requests to render the same page (no scripts, no fonts, few images), plus robots.txt. |
| C3 | **Name and tagline caps.** IP: 60 / 120. Privacy: 80 / 160. | 60 / 120 graphemes. | Stricter satisfies both; real brand names fit. |
| C4 | **SVG sanitizing.** Extraction: DOMPurify plus a custom hook as the main control. Outputs: rebuild every SVG from an allowlist. | Allowlist rebuild is the only path into the kit. Paints and classes are resolved by the sandboxed render probe, then re-serialized from scratch. DOMPurify is not shipped. | An allowlist cannot emit what it does not know. In testing, DOMPurify kept `<style>@import` and external `<image href>`. DOMPurify had more than 10 advisories in 2025-2026. Dropping it removes 12.1 KB and an advisory stream. |
| C5 | **CSP.** Privacy: one policy with `style-src 'unsafe-inline'` and `data:` images. IP: share route with `data:`. Outputs: strict share route with Trusted Types and `blob:` only. Extraction: the probe needs inline `<style>`, and a `srcdoc` iframe inherits the parent's CSP. | Two policies (§10.6), each sent as a header and repeated as a first-in-head meta tag. The app route `/` allows `style-src 'unsafe-inline'` because the probe requires it, and uses named Trusted Types policies. The share route `/k/` is strict: no inline styles, `blob:` images only, one Trusted Types policy for the PDF worker URL. | CSP3 §7.8: `srcdoc` documents inherit the embedder's policy ([csp3-inherit]). A nonce is impossible on static hosting. The share route never runs the probe and never calls `DOMParser`: `/k/` validates SVG with the pure grammar module (§6.2.2), never the probe. |
| C6 | **Deploy method.** Infra: Pages Git integration so no Cloudflare token sits in GitHub. Outputs: the font mirror must be built in GitHub Actions (Pages builds time out at 20 minutes). | Direct Upload from GitHub Actions with `wrangler pages deploy`. The Cloudflare token is scoped to Pages:Edit and stored only in a protected GitHub `production` environment that requires the owner's approval. | The mirror cannot be built in Pages CI, and committing about 300 MB of generated binaries to git is poor practice. Fork PRs never receive secrets, because no workflow uses `workflow_run` or `pull_request_target` (§10.13). Build and deploy are separate jobs; only the deploy job sees the token, and it runs a pinned local wrangler (§10.12). The account has no card, so a stolen token cannot create charges. The choice is permanent per project ([cf-direct-upload]). |
| C7 | **Report channel.** IP R-1 (`mailto:`) vs R-2 (a one-tap endpoint that forwards reports). | R-1, `mailto:` only. | R-2 needs a free forwarding channel; Cloudflare Email Routing needs the zone on Cloudflare, which conflicts with leaving DNS at Squarespace. It also adds a server surface. Privacy research already requires `mailto:` as the only contact route. |
| C8 | **Share path.** Outputs: `/k#…`. IP: a path ending in a slash. | `{SHARE_ORIGIN}/k/#1.<payload>.<len>`, plus a `_redirects` rewrite `/k/:payload  /k/  200` so `/k/%231.…` (mail-gateway damage) still loads (§7.3.4). | Safe Browsing can list a path prefix such as `<host>/k/` without listing `<host>/` ([safe-browsing]). |
| C9 | **Image scheme in the share view.** `data:` (IP, privacy) vs `blob:` only (outputs). | `blob:` only. | One fewer scheme. Logos render from `URL.createObjectURL`. |
| C10 | **"Stores nothing."** The competitor's phrase vs host logs, Turnstile, and Safe Links rewriting `#` into the path. | Copy says "Kitsheet doesn't store your uploads, inputs or kits." Host logs, Turnstile and its challenge cookies are disclosed. Share copy says the kit "isn't sent to Kitsheet's server when the link is opened normally", and the privacy policy explains the mail-gateway case. Cookie claims are verified across all domains, not only `<host>` (§15.6). | FTC Act §5 and the Texas DTPA make an untrue privacy claim the largest real legal risk. |
| C11 | **Fonts required in the schema.** Outputs made heading and body fonts required. The product must never invent. | Both fonts are optional. A logo-only kit has no fonts, and the UI says "Not set". | Forcing a default font would invent one. |
| C12 | **"Approved text-on-background pairs"** (locked kit content) vs IP E-2 (avoid "approved", which implies the brand owner approved). | The feature ships. Its label is "Pairs that pass WCAG AA" and "Pairs that pass WCAG AAA". | Same content; the label avoids a false-endorsement reading. Not a change to the locked decision. |
| C13 | **Minimum age clause.** Privacy: no clause needed. IP: state 13+ in the Terms. | Terms state 13+ (and parental permission under 18). No age gate anywhere. | Costs nothing, supports the general-audience posture, and addresses contract capacity. An age gate would itself collect data. |
| C14 | **Analytics.** Infra: Cloudflare Web Analytics is one click. Privacy: no JavaScript beacon in v1. | No client-side analytics. Pages Functions metrics count requests, not reads (§14); Turnstile challenges solved approximate reads. | EDPB Guidelines 2/2023 treat a beacon as "access" under ePrivacy Art. 5(3) ([edpb-2-2023]). |
| C15 | **Headless-browser fallback.** The brief allowed an optional Browser Rendering quota. Extraction and IP say no. | None in v1. JavaScript-built sites get an honest partial result plus upload prompts. | Free Browser Run is 10 browser-minutes per day ([cf-browser-run]); running a site's JavaScript conflicts with the "don't run JavaScript" fetch policy and runs that site's trackers from our account. |
| C16 | **robots.txt.** Google's user-triggered fetchers generally ignore it ([google-user-fetchers]); IP recommends honoring it. | Honored for every origin fetched, including the `*` group. | Cheap good-faith evidence (Texas Penal Code ch. 33, EU TDM opt-outs, cease-and-desist case law in N.D. Tex.). The upload paths absorb the lost fetches. |
| C17 | **pdf.js WebAssembly decoders** need `'wasm-unsafe-eval'` in CSP. | Do not ship pdf.js `wasm/`. JPX and JBIG2 images in guides are skipped. | Keeps the app CSP free of any eval-class keyword. Brand guides rarely depend on those codecs for color codes. [Guessing] on how rare; measure on 10 real guides. |
| C18 | **PDF accessibility.** Privacy: full PDF/UA is out of reach client-side. Outputs: PDFKit can tag PDFs. | Ship tagged PDFs with an explicit structure tree: every draw goes through `withStruct()` and everything else is marked as an artifact (§7.1.2). Do not claim PDF/UA. The accessibility statement names the share view and the review screen as the accessible equivalents. | `tagged: true` alone only sets the MarkInfo flag, and `pdfinfo` reports "Tagged: yes" even with an empty structure tree. veraPDF runs in CI as a report, not a gate. |
| C19 | **Image fetching: one call per image** (infra) vs batched (extraction). | One batched `POST /api/images` call per read for up to 5 images, returned as a KITX frame of raw bytes (§5.3). The ticket is single-use per nonce (best effort per isolate). | Each read costs 2 Function invocations instead of up to 6. KITX framing (C20) already avoids base64 and multipart work on the server, so the CPU profile matches per-image calls; M3 measures it, and per-image calls return only if p99 CPU exceeds 8 ms. |
| C20 | **Response format for fetched HTML, CSS and images.** Infra: JSON string fields. | A small binary frame (`KITX`, §5.2.8) sent as `application/octet-stream`: a JSON manifest plus raw bytes. Used by both endpoints. | No server-side charset decoding or JSON escaping of up to 5 MB, both of which cost CPU against the 10 ms limit. Raw bytes let the browser decode any charset correctly. Still never renderable. |
| C21 | **Logo "don'ts" and the source domain in share links.** | Neither rides in links. Don'ts are a fixed template set; the source domain stays local. | Keeps links short; a source domain in a link is an impersonation aid. |

### 0.3 Hard conflicts with locked decisions (flagged, not relitigated)

1. **Share links on the wedding business's domain.** Google Safe Browsing checks host and parent-host expressions, so `wethejamesphoto.com/` is checked for every page on the tool's subdomain ([safe-browsing]) [Certain on matching rules]. If someone reports a phishing kit, the worst case is a listing that also flags the photography site [Guessing on how Google scopes listings]. A second problem: links are meant to work for as long as Kitsheet runs, so links on `<host>` tie the tool to the wedding domain's renewal for good. If wethejamesphoto.com ever lapses and is re-registered, the new owner's JavaScript receives every old link's `#fragment`, which breaks the "never sent to a server" claim and gives them trusted-looking links to dress as phishing [Certain that fragments are readable by the page's own script]. This spec keeps `/k/` on `<host>` (as locked) with the mitigations in §10.8. The share-view origin is one build constant, `SHARE_ORIGIN`, so the owner can move only `/k/` to the project's free `*.pages.dev` hostname without code changes. That is the $0-forever option: it needs no domain, never expires with the photography business, and removes the Safe Browsing exposure in R1. `SHARE_ORIGIN` cannot change after launch without breaking every issued link. **Owner decision before M8** (§17.2 Q1).
2. **"Uploads never leave the browser" vs share links through Microsoft Safe Links.** Safe Links has been observed rewriting `#` as `%23` ([safelinks-issue]) [Likely]. The kit payload, including any SVG logo in it, then lands in the request path that reaches Cloudflare. Editor uploads are unaffected. Copy and the privacy policy are worded for this (C10).
3. **"No data capture" vs a working contact address.** A privacy policy needs a contact, and any inbound email gives the owner the sender's address, possibly a child's. Handled by a dedicated free Gmail mailbox with auto-contacts off, no forwarding and no connected apps, a no-reply rule for children and same-day deletion (§8.1, §9.0, §9.1).
4. **"Legally sound everywhere" vs $0.** If GDPR Art. 3(2) or the EU DSA ever applied, both would require a paid EU representative (GDPR Art. 27; DSA Art. 13). The design avoids both triggers (no EU targeting, no monitoring, no hosting). Any change that would trigger them requires re-review (§17.3).
5. **$0 vs a DMCA designated agent ($6 per 3 years) and a Texas assumed-name filing.** Neither is needed by this design (nothing is stored; the tool runs under the owner's own name). Both become needed only if the design changes (§17.3).
6. **Hosting dependency.** wethejamesphoto.com expires 2027-08-24 per RDAP on 2026-11-08 [Certain]. The tool's URL dies with the domain, and so do share links if `SHARE_ORIGIN` stays on `<host>` (item 1). Keep auto-renew on. This is an existing owner cost, not a new one.
7. **Website reading can be switched off for a day by junk traffic.** Every request that reaches a Function counts against the 100,000/day account quota before any of Kitsheet's code runs, rejected requests included (§10.3, §10.5). The only control that stops requests before they count, WAF rate limiting, needs the domain on a Cloudflare zone, which §11.4 #12 rules out while DNS stays at Squarespace. Accepted: the result is E-CAPACITY until 00:00 UTC, never a charge (R6).

---

## 1. Product summary, goals, explicit non-goals

### 1.1 Summary

Kitsheet turns a website, a logo file, a screenshot or an old brand-guide PDF into a brand kit: logo and favicon, up to five role-labeled colors with tints, shades and contrast checks, heading and body fonts with a type scale, starter logo-usage rules, the brand name and tagline. The user reviews and edits everything on one screen, then downloads a multi-page PDF guide and a PNG brand board, or copies a link that carries the whole kit inside the URL fragment.

Everything except reading a public web page happens in the browser. Reading a web page uses one small server function that fetches bytes and does no analysis. Nothing the user types, uploads or makes is stored on a server.

### 1.2 Goals (each one is testable)

| ID | Goal | How it is checked |
|---|---|---|
| G1 | Matches the competitor's extraction bar on real small-business sites. | Corpus targets in §15.3 (for example primary color top-1 ≥ 24/30, heading and body family ≥ 27/30 each), plus a one-time side-by-side on the same 30 URLs: Kitsheet ≥ competitor on primary top-1, heading family, body family and logo top-1, with 0 invented colors (§15.3, M9). |
| G2 | Never invents a color, font, name, tagline or number. | Property test: every output value traces to a provenance record or carries `derived: true` (§5.11). Zero invented colors is a hard release gate. |
| G3 | Costs the owner $0 per month, forever. | Monthly checklist (§11.5). No payment method on any account. |
| G4 | Every privacy claim is literally true at the network level. | Playwright network-allowlist, cookie and storage tests on every flow (§15.6); every sentence in §9.7 and the §9.1 short version maps to a named test or spec section, signed off by the owner at M10. |
| G5 | Meets WCAG 2.2 AA for the tool UI and the share view. | axe in CI plus manual screen-reader passes (§13). |
| G6 | Fast on a mid-range phone. | Budgets in §4.5 enforced in CI. |
| G7 | Looks like the owner's work: one clear layout idea, utilitarian type, one accent. | `docs/ops/design-review.md`: one yes/no line per D33 directive and per D34 ban, checked on fixed screenshots (§16 M1); the owner is the sole approver, and M1, M6 and M7 cannot close until every line is yes. |

### 1.3 Competitor parity (internal reference only; the product never names the competitor, M-3)

| Competitor capability | Kitsheet v1 |
|---|---|
| Live fetch, stylesheet parsing, var() chains, light/dark, Tailwind v3/v4, Elementor/WordPress tokens, all color syntaxes | Yes. The browser's own CSS engine resolves the cascade in a sandboxed probe (§5.4). |
| Painted colors on buttons, links, headings and background win | Yes (§5.5). |
| Up to 5 role colors; never invents | Yes; plus up to 3 extra colors the user chooses to keep. |
| Heading and body fonts with load-source label; filters icon and fallback fonts | Yes (§5.7). |
| Ranked logo candidates and favicon | Yes (§5.8). |
| Brand name and tagline from metadata | Yes (§5.9). |
| WCAG contrast with AA/AAA | Yes, plus a full grid and passing-pairs list. |
| Human check | Cloudflare Turnstile, loaded only when a URL is read. |
| Stores nothing | Yes for inputs and kits; host logs disclosed. |
| Not in competitor: uploads, PDF guide, PNG board, share link, ramps, type scale, logo rules | Yes. |

### 1.4 Explicit non-goals for v1

- **No accounts**, sign-in, profiles, saved kits on a server, or galleries.
- **No email or phone collection** of any kind: no newsletter, no "email me my kit", no "text me the link", no contact form, no chat widget, no waitlist, no `sms:` or `tel:` links.
- **No AI, LLM, or machine-learning model**, local or remote. Every result comes from published rules in this spec.
- **No server storage** of inputs, fetched pages, kits, logs of URLs, or share links. No database, KV, R2, Durable Object content, or cache of third-party content.
- **No social media metrics**, follower counts or social-ready assets.
- **No design-token or code export** in v1 (§7.7 notes it as a cheap later option).
- **No headless browser**, crawling beyond one page, or running a site's JavaScript.
- **No client-side analytics, cookies set by Kitsheet, service worker, ads, or third-party scripts** other than Turnstile at the moment a URL is read (Cloudflare's challenge frame may set its own short-lived cookies on `challenges.cloudflare.com`; disclosed, §8.2).
- **No monetization**: no paid tier, affiliate links, sponsorships or lead capture.
- **No localization** into EU languages and no EU/UK-targeted marketing (keeps GDPR Art. 3(2) untriggered, §8).
- **No CMYK or Pantone values computed from screen colors.** Pantone and CMYK found in an uploaded guide are shown verbatim as text.
- **No QR codes** in v1 (deferred; §7.3.6). Logos that contain a QR code or other scannable code are refused in links and hidden in the share view (§6.2.6).

### 1.5 Cut list, in order, if behind schedule

Cut from the top. Each item names what replaces it. D13-D27 (all five inputs, all three outputs, and the color, type and logo-rules systems) are **not** on this list, and neither is any security or privacy control.

| # | Cut | Replaced by |
|---|---|---|
| 1 | SVGO **Simplify logo for the link** (§7.3.3) | **Copy link without logo** only |
| 2 | JPEG white-background removal (§6.2.4) | The existing "One-color version needs a logo with a transparent background" note |
| 3 | Dark pass (§5.4.3) | Light pass only |
| 4 | `%2523` repair (§7.3.4) | `%23` repair only |
| 5 | `BroadcastChannel` tab sync (§7.6.1) | Last write wins |
| 6 | PNG long-press overlay for in-app browsers (§7.1.6) | The E-INAPP message only |
| 7 | dHash dedupe (§5.8.3) | Dedupe by normalized URL |
| 8 | Full font mirror (§7.1.3) | 400, 700 and 400 italic per family at launch |
| 9 | Byte-identical kit JSON across three engines (§15.2) | Byte-identical on Chromium plus identical role hex across engines |

---

## 2. User flows and screens

### 2.1 Routes

| Route | What it is | JS | Notes |
|---|---|---|---|
| `/` | The app: start, working, review/edit, export | Yes | Only route that may load Turnstile. |
| `/k/` | Read-only share view | Yes (small) | Kit comes from `location.hash`. `noindex`. Strict CSP. |
| `/privacy`, `/terms`, `/acceptable-use`, `/credits`, `/accessibility`, `/bot`, `/report` | Static pages | No | Prerendered HTML emitted as flat files (`dist/privacy.html` …) and served at the extensionless path (§10.6). |
| `/clear` | Clear-my-data page | Tiny | A **Clear now** button runs the same in-page clear as the editor (§7.6.3). No `Clear-Site-Data` header, so a cross-site link to `/clear` deletes nothing. |
| any other path | `404.html` | No | Static "No page here." page with status 404 (Pages 404 mode, §10.6). The app HTML is served only at `/`. |
| `/api/extract`, `/api/images` | Pages Functions | n/a | POST only; same-origin only. These two exact paths are the only ones that invoke Functions (`_routes.json`, §10.6). |

### 2.2 Screen anatomy (applies to every app state)

The app is one asymmetric spread (the anchor idea in §12.1):
- **Margin column** (left, 4 of 12 columns on desktop): inputs, step progress, editor controls, export buttons. Small text, quiet. The tool's single accent lives only here.
- **Sheet** (right, 8 of 12 columns): the kit as it will print, in sections that match the PDF pages (Cover, Logo, Logo use, Color, Tints and shades, Contrast, Type, Type scale). The brand name is the dominant headline, set in the brand's heading font at the top of the Cover section, hanging into the sheet's left padding (§12.4).
- **Footer** on every route: Privacy · Terms · Acceptable use · Accessibility · Credits · How the reader works · Source code · Contact · "Built by Justin James".

Below 768 px wide, the order is the one in §12.4: the sheet's Cover block, then the margin controls, then the rest of the sheet.

### 2.3 Start screen (empty state)

Margin column, top to bottom:
1. Product line, one sentence: "Paste your website. Get your logo, colors and fonts as a PDF guide, a PNG board and a link you can share."
2. **Website** field: `<input type="url" inputmode="url" autocomplete="url">`, placeholder `yourbusiness.com`, button **Read site**. Microcopy: "Kitsheet's server fetches this public page, its stylesheets and a few images, hands them to your browser to read, and keeps nothing." Always visible under the button from page load, not only after the click: "Read site first runs Cloudflare's bot check, which sends your IP address and browser details to Cloudflare and may set a short-lived cookie. Uploads don't use it. [What's sent]" (links to `/privacy#bot-check`). This notice is for transparency and Art. 13 timing; it is never described as consent (§8.2). Then, at body size (16 px) in `--ink` with an underlined link: "By pressing Read site, you agree to the [Terms] and [Acceptable Use Policy]. Build kits for brands you own or represent, or keep them for private reference." (Notice tied to the action, no checkbox; L-3, §9.2.)
3. **Or start from files** (three buttons, each a real file input with a visible label): **Logo** (SVG, PNG, JPG) · **Image to palette** (PNG, JPG, WebP, GIF, AVIF) · **Brand guide PDF** (enabled only when the §6.4 feature gate passes; otherwise E-PDFOLD under it). Microcopy: "Files stay on this device. Nothing is uploaded." Then, at body size: "By adding a file, you agree to the [Terms]."
4. **Or start blank** (quiet button): opens the editor with empty fields.
5. If a kept draft exists (the user ticked **Keep this draft on this device**, §7.6.1): "Draft from {date} · **Open draft** · **Clear saved draft**". Microcopy: "This draft is saved in this browser only because you chose to keep it. Safari may clear it after 7 days without a visit."

Sheet: an empty proof. Crop marks at the corners. The headline slot shows the page's `<h1>`, "Brand kit from your website or files.", in Archivo 800, `--sheet-ink`, at display size, with the same hang as a brand name (§12.4). It is not `aria-hidden`, and it is replaced by the brand name once a name exists. Section labels (P.01 Cover … P.10 Colophon) in mono, each with "Empty". No sample third-party brand (L-5).

### 2.4 URL flow

1. User types a URL and presses **Read site** (or Enter).
2. Client normalizes: trims; adds `https://` when no scheme; rejects anything that does not parse as `http:`/`https:` with a dotted host (error state E-INVALID). Query string and fragment are dropped before sending (PRIV-06). The field shows the normalized URL.
3. Client loads Turnstile (first time only) and renders it in the reserved slot under the field in `interaction-only` appearance with `action: 'extract'`. Microcopy beside the slot: "Quick bot check by Cloudflare. [What's sent]" (links to `/privacy#bot-check`). The fuller notice in §2.3 item 2 has been on screen since page load, so the visitor sees it before their IP reaches Cloudflare. The click is not described as consent anywhere; the legal basis stays "strictly necessary" (§8.2).
4. On token: `POST /api/extract` (§5.2). The **step list** replaces the start options in the margin:

| Step | Label | Done when |
|---|---|---|
| 1 | Bot check | Turnstile returns a token |
| 2 | Reading the page | Manifest received from `/api/extract` |
| 3 | Reading stylesheets ({n} found, {k} read) | Same response; counts come from the manifest |
| 4 | Finding colors | Probe passes and role scoring finish |
| 5 | Finding fonts | Font detection finishes |
| 6 | Ranking logos | One `/api/images` call (up to 5 images) finishes or times out |
| 7 | Checking contrast | Color system computed |

Each step shows one of: waiting, working (a 1 px indeterminate line; static under `prefers-reduced-motion`), done, skipped (with reason), failed (with reason). Each change is announced through one `aria-live="polite"` region. A **Cancel** button aborts all requests with one `AbortController` and returns to the start screen with the URL kept.

5. On success: the review screen (2.8) opens with focus moved to the sheet's "Results" heading, and the live region announces, for example, "Done. 4 colors, 2 fonts and 3 logo options found." The counts are real counts from this run.
6. A new read of another URL replaces site-sourced values but never values the user locked or typed (§6.5.4).

### 2.5 Logo upload flow

1. User picks a file (button or drop). Accepted: `.svg` (≤ 1 MB, ≤ 10,000 elements), `.png`/`.jpg`/`.jpeg` (≤ 5 MB, ≤ 40 MP). Type is decided by magic bytes, not extension.
2. Steps: "Reading file" → "Cleaning SVG" (SVG only) → "Finding colors" → "Making variants" (SVG only) → "Checking contrast".
3. Review opens with the logo in the Logo section, its colors offered as suggested roles (labeled "suggested from logo"), and variants (on light, on dark, one color) for SVG. Raster logos show on-light and on-dark previews and, when the image has transparency, a one-color silhouette (§6.2.3).
4. If an SVG uses features the canonical form drops (live text, embedded images, filters, masks, patterns), or its canonical form fails the grammar check (§6.2.2), show: "This SVG uses {features}. Variants and the share link need plain shapes. Convert text and images to outlines in your design app, or keep it as is: the PDF and PNG will use a high-resolution image of it." Kitsheet then draws the original through the raster-only sanitizer (§6.2.2), shows the resulting image, and uses it only after the user confirms it matches their logo. If the browser refuses the draw (a tainted canvas), the message asks the user to convert text and images to outlines. The logo then behaves like a raster logo.
5. If the logo contains a QR code or other scannable code (§6.2.6), it stays usable in the PDF and PNG, and the export panel offers only the no-logo link (§2.9).

### 2.6 Image to palette flow

1. User picks an image (PNG, JPEG, WebP, GIF first frame, AVIF where the browser decodes it; ≤ 20 MB, ≤ 40 MP). HEIC: "Convert to JPG or PNG first. iPhone: Settings > Camera > Formats > Most Compatible."
2. Steps: "Reading image" → "Grouping colors" → "Checking contrast".
3. Review shows 3 to 6 swatches with their share of the image ("38% of pixels"), suggested roles labeled "suggested from image", and a **Keep** toggle per swatch. The image itself is not stored (§7.6.2).

### 2.7 Brand-guide PDF flow

1. User picks a PDF (≤ 25 MB). The button is enabled only when the §6.4 feature gate passes; otherwise E-PDFOLD shows under the disabled button. The pdf.js legacy-build chunk loads on demand (§4.5).
2. Steps: "Opening PDF" → "Reading text (page {i} of {n})" → "Reading artwork colors" → "Matching fonts".
3. Review shows **Found in your guide**: color records (label such as "Brand Orange", HEX, RGB, CMYK and Pantone as printed, page number, confidence "HEX and RGB agree" or "HEX and RGB disagree: check"), and font names with page numbers. Roles are suggested only from labels in the guide ("Primary", "Secondary", "Accent", "Background", "Text"); unlabeled colors wait in the list for the user to assign.
4. If the PDF has no text codes (outlined or scanned): "No color codes in this PDF's text. These colors were sampled from its artwork." and the fill colors are offered.
5. Password-protected or broken PDFs: E-PDF (§2.10). A pdf.js load or start-up failure with `SyntaxError`, `TypeError` or `ReferenceError` (a browser too old for the library): E-PDFOLD.

### 2.8 Review and edit screen

Sheet sections, each mirroring a PDF page and labeled in mono ("P.05 Color"):

| Section | Shows | Edit controls (margin column, next to the section) |
|---|---|---|
| Cover | Name (dominant headline), tagline, logo, date | Name field (required to export), tagline field |
| Logo | Primary logo on its preferred background; on light, on dark, one color; favicon at 16, 32, 180 px | Pick among logo candidates (with reasons, for example "linked to home page", "named 'logo'", "from structured data"); upload a different logo; preferred background (auto, light, dark); for SVG, "Treat as cut-out" toggles per near-white paint |
| Logo use | Clear-space diagram, minimum size, don'ts | Clear space (0.1-2.0 × logo height), minimum size (px and mm). Label: "Starter rules computed by Kitsheet from this logo's proportions. Not the brand owner's rules." |
| Color | Role swatches with HEX, RGB, HSL, source; "Other colors on the page" with usage counts | Per color: hex text field, native color picker, role select, name, lock, remove, reorder (buttons, not drag-only); "Add color" |
| Tints and shades | 11-stop ramp per brand color, each stop with hex and a white or black "Aa" that passes AA | None (derived). Label: "Derived from the brand color. Not part of the original brand." |
| Contrast | Grid of every kit color as text over every kit color as background, plus reference white and black; passing-pairs lists | None (derived) |
| Type | Heading and body specimens; family; source label; license note; substitute note; fallback stack | Font picker for heading and for body (Google Fonts catalog; §6.5.3); weights (up to 4 each); italic |
| Type scale | Steps from caption to display with px and pt | Base size (12-24 px, step 0.5) and ratio (1.125 to 1.618) |
| Colophon | Sources, dates, licenses, disclaimers | None |

Every value shows its source in small mono text: `site`, `logo`, `image`, `guide`, `manual`, or `derived`. Edited values show "edited". A **Sources** list in the margin shows each input used (domain and read date, file names, "typed by you") with **Remove** to drop that source's values (values the user locked stay).

The margin also holds two storage controls (§7.6.1): **Keep this draft on this device** (a checkbox, unchecked by default) and **Don't remember my settings** (a quiet button that stops storing the theme and page size and deletes any stored values).

Name, tagline, color names and custom font names are checked with `containsContact()` as the user types (§6.5.5); a failing value shows E-CONTACT under its field and is not saved.

### 2.9 Export and share

Export panel at the bottom of the margin column (sticky on desktop). At its top, at body size in `--ink` with an underlined link: "By downloading or copying a link, you agree to the [Terms]." Export is blocked with E-NOCOLOR while the kit has no color, and needs a name (§3.4).

1. **Download PDF guide**. Page size control: Letter / A4, defaulting to Letter for `en-US`, `en-CA`, `es-MX`, `fr-CA` and A4 otherwise [Guessing: product choice]. Progress text: "Making PDF (page {i} of {n})". On iOS and Android a second button, **Share or save PDF**, appears once the file is built (§7.1.6).
2. **Download PNG board**. Note: "Image file. The PDF and the link contain the same details as text."
3. **Copy share link**. Before the first link in a session, a required, unchecked checkbox: "I own this brand, work for its owner, or have the owner's permission. I won't use this link to pass the kit off as the owner's official material, and I agree to the Terms and Acceptable Use Policy." Nothing is stored or sent when it is checked. The encoder runs `containsContact()` on every text field and the scannable-code check result (§6.2.6) first. Then:
   - The link is copied and shown in a read-only field. If the clipboard write fails: E-COPY.
   - Microcopy: "Anyone with this link can see this kit. The kit lives inside the link; it isn't sent to Kitsheet's server when the link is opened normally. Don't put private information in the name or tagline."
   - Length line, from the real link length: "Fits anywhere" (≤ 1,900 characters), "Too long for Discord; fine for LinkedIn, email, Slack, iMessage and X" (≤ 2,900), or "Long link: works in email, Slack, iMessage and X; too long for LinkedIn and Discord posts" (> 2,900).
   - **Copy link without logo** (always short), **Simplify logo for the link** (SVG over 8 KB only), and on devices with Web Share, **Share…** (`navigator.share({url})`).
   - Raster logos: "Your logo is an image file, so it can't ride in the link. The link carries everything else."
   - Logos flagged by the scannable-code check: only the no-logo link is offered, with "Logos that contain a QR code or other scannable code can't go in links. The link carries everything else."

If making the PDF or the PNG fails (worker error, memory, a font that will not load), the panel shows E-EXPORT inline with the reason and keeps every other export available.

### 2.10 States and their exact copy

Copy is final unless the owner edits it. `{…}` are real values from the current run.

| ID | State | Where | Copy | Actions |
|---|---|---|---|---|
| E-EMPTY | Empty | Start | §2.3 | — |
| E-WORK | Working | Margin | Step list (§2.4) | Cancel |
| E-PARTIAL | Partial result | Banner at top of sheet | "Partial result. Found {k} of 5 color roles, {fonts} and {logos}." Second line names each empty slot and why, for example "No heading font: the page's CSS sets none Kitsheet could read." | Add a screenshot · Upload logo · Enter by hand |
| E-JS | JavaScript-built page | Banner | "This page builds itself with JavaScript after it loads. Kitsheet reads the page as the server sent it, so some details are missing. Add a screenshot or your logo to fill the gaps." | Same as E-PARTIAL |
| E-ROBOTS | robots.txt disallows | Margin error | "This site asks automated tools not to read this page, so Kitsheet didn't. Upload your logo, a screenshot or a brand PDF instead." | Upload buttons |
| E-OPTOUT | On the do-not-fetch list | Margin error | "The owner of this site asked Kitsheet not to read it. Upload your files instead." | Upload buttons |
| E-BLOCKED | 401, 403, 429, challenge page, login or consent wall | Margin error | "This site blocked Kitsheet's request. It may use a bot wall or need a login. Kitsheet doesn't try to get around that. Upload your files instead." | Upload buttons |
| E-404 | 404/410 | Margin error | "That page returned 'not found'. Check the address." | Edit URL |
| E-UPSTREAM | Other non-2xx from the site (`upstream_error`) | Margin error | "That site returned an error ({status}). Try again later, or upload your files." | Retry · Upload |
| E-TLS | `tls_error` | Margin error | "That site's security certificate didn't check out, so Kitsheet didn't read it." | Upload |
| E-REDIRECTS | `too_many_redirects` | Margin error | "That address redirects too many times. Paste the final address from your browser's address bar." | Edit URL |
| E-DNS | Name does not resolve | Margin error | "No website answered at that address. Check the spelling." | Edit URL |
| E-TIMEOUT | Upstream timeout | Margin error | "The site took too long to answer. Try again in a minute, or upload your files." | Retry · Upload |
| E-TOOBIG | HTML over 2 MB | Margin error | "That page is bigger than Kitsheet reads (2 MB of HTML). Try a simpler page on the same site, such as /about, or upload your files." | Edit URL |
| E-NOTHTML | Not an HTML page | Margin error | "That address isn't a web page. It returned a {type} file." | Edit URL |
| E-PRIVATE | SSRF guard rejected the URL | Margin error | "Kitsheet reads only public websites. That address points to a private or local network, or uses a port Kitsheet doesn't read." | Edit URL |
| E-INVALID | Unparseable URL | Inline field error | "That doesn't look like a web address. Example: yourbusiness.com" | — |
| E-CONTACT | Name, tagline, color name or custom font name fails `containsContact()` (§6.5.5) | Inline field error | "Names and taglines can't include web addresses, email addresses or phone numbers. If the name ends in .com or similar, drop that ending. If it isn't a web address, add a space after the period." | — |
| E-HUMAN | Turnstile failed | Margin error | "The bot check didn't pass. Try again. Uploads don't need the check." | Retry · Upload |
| E-BUSY | 429 from Kitsheet's limiter (carries `X-Kit: 1`) | Margin error | "Too many reads from your connection in a short time. Wait a few minutes, then try again." | Upload |
| E-HEAVY | Cloudflare error 1101 or 1102, or any other 5xx without `X-Kit` (§11.3) | Margin error | "This page needs more work than Kitsheet's free server allows per read. Try a simpler page on the same site, such as /about, or upload your files." No automatic retry, no day-long lockout. | Edit URL · Upload |
| E-CAPACITY | Two consecutive capacity-shaped responses (§11.3) | URL field disabled; margin notice | "Live site reading is paused until {local time}. Uploads and manual entry still work, and they run in your browser." | Upload buttons |
| E-OFFLINE | `navigator.onLine === false` or network error | Margin notice | "You're offline. Uploads and manual entry still work." | — |
| E-SERVER | 500 with `X-Kit: 1`, or an unparseable response that carries `X-Kit: 1` | Margin error | "Kitsheet's server hit an error. Try again, or upload your files." | Retry |
| E-FILE | Wrong type, too large, corrupt | Inline under the file button | "{file} isn't a {kind} Kitsheet can read. Use {accepted list}, up to {limit}." | — |
| E-PDF | Encrypted or broken PDF (pdf.js `InvalidPDFException` or `PasswordException`) | Inline | "This PDF is password-protected or damaged. Export a copy without a password and try again." | — |
| E-PDFOLD | Fails the PDF-button gate (§6.4), or pdf.js load or start-up throws `SyntaxError`, `TypeError` or `ReferenceError` | Under the disabled button, or inline | "This browser can't read PDFs here. Update to Safari 18, Chrome or Edge 125, or a current Firefox. Or take a screenshot of the guide's color page and use Image to palette." | Image to palette |
| E-EXPORT | PDF or PNG generation failed | Inline in the export panel | PDF: "The PDF couldn't be made on this device ({reason}). Try again, or download the PNG board or copy the link." PNG: "The PNG board couldn't be made on this device ({reason}). Try again, or download the PDF or copy the link." | Retry |
| E-COPY | Clipboard write failed | Under the link field | "Copy didn't work here. Select the link above and copy it." | — |
| E-NOCOLOR | Export with no colors | Export panel | "Add at least one color to export." | Add color |
| E-STORAGE | IndexedDB unavailable after the user ticks **Keep this draft** | Margin notice | "This browser isn't saving drafts (private mode or storage is off). Download or copy your link before you close the tab." | — |
| E-INAPP | In-app browser (Instagram, Facebook, LinkedIn, WhatsApp) | Export panel | "This app's built-in browser can't save files. Open this page in Safari or Chrome to download." | Copy page link |
| E-OLD | Fails the §4.6 feature gate | Full page | "This browser is too old for Kitsheet. Update to a current Safari, Chrome, Firefox or Edge." | — |
| S-NOFRAG | Share: no fragment | `/k/` | "This link has no kit in it." | Go to Kitsheet |
| S-NEWER | Share: unknown format version | `/k/` | "This kit was made with a newer version of Kitsheet. Reload the page." | Reload |
| S-CUT | Share: length or checksum mismatch | `/k/` | "This link looks cut off. Ask the sender to copy it again." | — |
| S-INVALID | Share: fails validation | `/k/` | "This link can't be displayed." | — |
| S-BLOCKED | Share: on the blocklist | `/k/` | "This kit link has been disabled." | — |
| S-WATCH | Share: name matches the impersonation watchlist, mixes scripts, or is one edit from a watchlist name (§7.4) | `/k/` interstitial | "This kit uses the name of a large organization. It was made by an anonymous visitor, not by {name}. This page never asks you to log in, pay, share personal details, call a number or scan a code." | Show the kit · Leave |
| S-LOGO | Share: logo hidden (§7.4 steps 8 and 8a); the rest of the kit renders | Cover and Logo slots on `/k/` | Scannable code found: "Logo hidden: it contains a scannable code." Grammar check failed: "This link's logo could not be checked, so it isn't shown." | — |

The capacity reset time is computed in the browser from 00:00 UTC and printed in the visitor's local time with `Intl.DateTimeFormat` [Certain that the quota resets at 00:00 UTC; [cf-limits]]. No capacity number appears anywhere in the UI (INFRA-32).

### 2.11 Combining sources

A user may combine any inputs in any order (for example read a URL, then upload the real logo and a PDF guide). Rules in §6.5.4. The margin's **Sources** list is the single place to see and remove what fed the kit.


---

## 3. Brand kit contents

### 3.1 What a kit holds

| Part | Contents | Required to export? |
|---|---|---|
| Identity | Brand name (1-60 graphemes), tagline (0-120) | Name: yes. The user can type it; it is never generated. |
| Logo | One primary logo (canonical SVG or raster), derived variants, favicon | No |
| Colors | 1-8 colors. Roles: at most one each of primary, secondary, accent, background, text; any number of neutral and other. Each has hex, optional name, source. | At least 1 |
| Fonts | Heading and body (each optional): family, weights, italic, source label | No |
| Type scale | Base size and ratio (defaults 16 px and 1.25, labeled "default" when not detected) | Defaults apply |
| Logo rules | Clear space, minimum size (px and mm), preferred background, don'ts | Only when a logo exists |
| Derived | Ramps, contrast grid, passing pairs, type steps, fallback stacks, logo variants | Computed, never stored or shared |

### 3.2 PDF brand guide: page by page

**Format.** Landscape. US Letter (792 × 612 pt) or A4 (842 × 595 pt). Margins 36 pt. 12-column grid, 12 pt gutters. On every content page, columns 1-3 hold the section label and notes in mono; columns 4-12 hold content (the asymmetric split from §12.1). White page background for honest print color. Footer on every page at 24 pt from the bottom, mono 7 pt, `#5C594F`:

> Made with Kitsheet · {YYYY-MM-DD} · Source: {domain, "your files" or "shared link"} · Not verified by the brand owner · {page}/{total}

**Pages.** A page is omitted when it would be empty (never padded with filler). The contents page and page numbers reflect the pages actually present.

| # | Page | Content and layout | When data is missing |
|---|---|---|---|
| 1 | Cover | Brand name as the dominant headline in the heading font, fitted to columns 1-10 with `fitHeadline` (§12.2): 120 pt down to 60 pt in at most 2 lines, then 3 lines down to 36 pt, then a break inside the longest word; never truncated. Left edge starts 18 pt outside the left margin, half of the 36 pt margin (the off-grid moment, §12.1); for a right-to-left name the hang moves to the right edge (§7.1.7). If any letters use a fallback face, the mono line "Some letters shown in Noto Sans {script}; {family} has no {script} letters." sits under the name. Tagline below in the body font, 16 pt. Logo in columns 9-12, bottom, on its preferred background. "Brand kit · {Month YYYY}" in mono. | No tagline: omitted. No logo: omitted; the name gets the space. No heading font: name set in Archivo 800 with the mono note "Shown in Archivo; no heading font chosen." |
| 2 | Contents and sources | List of present pages. Sources with dates: "Read from example.com on 8 Nov 2026", "From your files: logo.svg, guide.pdf", "Typed by you". One paragraph: "Colors, fonts, logo, name and tagline come from the sources above. Tints, shades, contrast checks, the type scale and the logo rules were computed by Kitsheet." If some letters have no covering font (§7.1.7): "The PDF can't show these letters: {chars}. The PNG and the link show them." | Always present. |
| 3 | Logo | Primary logo large (columns 4-9) on its preferred background with clear space shown faintly. Three tiles in columns 10-12: on light, on dark, one color. Favicon at 16, 32 and 180 px shown at those pixel sizes (converted at 0.75 pt/px). | No logo: page omitted. Raster without transparency: one-color tile reads "One-color version needs a logo with a transparent background." No favicon: favicon row omitted. |
| 4 | Logo use (starter rules) | Clear-space diagram: logo with a dashed boundary at X = clearSpace × logo height and the label "X = {value} × logo height". Minimum size: logo drawn at the minimum, labeled "{px} px wide on screen · {mm} mm wide in print". Don'ts: up to 6 tiles with the actual logo stretched to 150% width, recolored in a non-brand color, rotated 15°, with an outer glow, on a busy photo-like noise pattern, outlined. Each tile carries a thin ink X drawn as two vector lines and a caption ("Don't stretch"). Header note: "Starter rules computed by Kitsheet from this logo's proportions. Not the brand owner's rules." | No logo: page omitted. "Recolor" tile needs a one-color version; omitted otherwise. |
| 5 | Color | Role swatches in an asymmetric stack: primary largest (columns 4-8, full height of the content area), then secondary, accent, background, text, then others. Each swatch: name if any, role, HEX, RGB, HSL, source (site, logo, image, guide, typed). Label at top: "Screen colors (sRGB)". If the guide supplied CMYK or Pantone, they are printed verbatim with "from your guide". | Missing roles are not drawn; a single line lists them: "No accent color in this kit." |
| 6 | Tints and shades | One row per chromatic color (primary, secondary, accent, then others; background and text only if they are not near-white or near-black), up to 6 rows. 11 stops (50-950) per row, each with hex and an "Aa" in white or black, whichever passes AA, plus its ratio. The brand anchor stop is marked "brand". Note: "Derived from the brand color in OKLCH. Not part of the original brand." | Fewer colors: fewer rows. No chromatic colors: page omitted. |
| 7 | Contrast | Grid: rows are each kit color as text; columns are each kit color as background plus reference white and black (labeled "reference, not a brand color"). Each cell: ratio truncated to 2 decimals and a text badge (AAA, AA, AA Large, or "Fails"). Below: "Pairs that pass WCAG AA (normal text, 4.5:1 or more)" and "Pairs that pass for large text and UI parts only (3:1 to 4.49:1)", each sorted by ratio. Disclaimer (E-4): "Ratios use the WCAG 2.2 formula (4.5:1 AA and 7:1 AAA for body text; 3:1 AA and 4.5:1 AAA for large text). Passing these checks is not a full accessibility review." | One color: grid has one row plus the reference columns. |
| 8 | Type | Heading specimen: family name at 48 pt in the heading font, the brand name at 32 pt, a pangram line. Body specimen: a 3-line paragraph of neutral sample text at 11 pt. For each font: family, weights, source label (for example "Google Fonts family, served by Squarespace"), license ("SIL Open Font License 1.1"), fallback stack in mono. Substitutes (§5.7.5) are labeled in place: "Brand font: Gotham (from a commercial service; not included). Shown in Montserrat (SIL OFL 1.1), a free substitute. License Gotham from its foundry to use it." Note: "Font sources are detected from how the site loads them. They are not a license check." Fallback-face label as on the cover when it applies (§7.1.7). | No fonts: page shows "No brand fonts chosen" with the system fallback stacks. One font: the other block reads "Not set". |
| 9 | Type scale | Steps: display, h1, h2, h3, h4, body, caption, each with px, pt, line height and a sample line (brand name for headings, pangram for body). Ratio and base shown as "1.25 (detected)" or "1.25 (default; not detected)". | Always present (defaults are labeled). |
| 10 | Colophon | "Made with Kitsheet ({host}) on {date}. Sources: {list}. Files supplied by the person who made this kit. Brand names, logos and fonts belong to their owners. Kitsheet and its maker are not affiliated with or endorsed by {brand name}, and did not check who made this kit. Fonts marked 'substitute' are free stand-ins; only open-licensed fonts are embedded." Then: embedded fonts with license names; "This PDF was made in your browser. Nothing in it was sent to a server." Exported from a share view, add: "Shared kit. Not verified by the brand owner." Last line, plain text, never a link: "Kitsheet is built by Justin James · {PORTFOLIO_URL}". | Always present. |

**Sample text.** Body specimen text is the fixed pangram "The quick brown fox jumps over the lazy dog. 0123456789" plus a neutral line "Body text set at {size} on {line height}." It never states facts about the brand.

### 3.3 PNG brand board

Canvas 1600 × 1000 CSS px drawn at 2× (3200 × 2000 device px). White background. Margins 96 px; 12 columns of 88 px with 32 px gutters (column n starts at x = 96 + (n − 1) × 120).

| Region | Columns / box | Content |
|---|---|---|
| Name | x from 48 (48 px outside the margin, half the margin; the off-grid moment, §12.1) to the end of column 8 (x = 1024); top y = 96. Right-to-left names hang past the right edge of column 8 instead | Brand name in the heading font, fitted with `fitHeadline` (§12.2): 160 px down to 72 px in at most 2 lines, then 3 lines down to 48 px, then a break inside the longest word; never truncated |
| Tagline | Columns 1-6 under the name | Body font, 28 px, max 2 lines |
| Logo tile | Columns 1-4, y 560-872 | Logo on its preferred background with clear space |
| Type tile | Columns 5-8, y 560-872 | "Aa" in the heading font at 120 px, heading and body family names in mono, one body line |
| Color stack | Columns 9-12, y 96-872 | Up to 8 full-width bands, equal heights, order: primary, secondary, accent, others, text, background. Each band: role and hex in mono, in white or black, whichever passes AA on that band. |
| Footer | Columns 1-12, y 920 | Provenance line (same as the PDF footer), mono 14 px, `#5C594F` |

Missing data follows §3.4: an absent region leaves deliberate white space; nothing is generated to fill it.

### 3.4 Missing data (never invent)

| Missing | Review screen | PDF | PNG | Share view |
|---|---|---|---|---|
| Name | Export disabled; field says "Type the brand name to export." | — | — | Links require a name |
| Tagline | Field empty | Omitted | Omitted | Omitted |
| Logo | "No logo yet" with upload button | Pages 3-4 omitted | Logo tile left empty | Logo section omitted |
| Raster logo in a link | Note in export panel | n/a | n/a | "This kit's logo is an image file and isn't included in links." |
| Logo with a scannable code, or a link logo that fails the grammar check | Export panel offers only the no-logo link | Logo used (local exports) | Logo used (local exports) | Logo hidden with the S-LOGO line; exports from `/k/` carry no logo |
| Favicon | Row omitted | Row omitted | n/a | Omitted |
| A color role | Slot shows "Empty" with "Add" | Listed as missing on page 5 | Band not drawn | Listed as missing |
| Heading or body font | "Not set" with picker | §3.2 page 8 | Family name shows "Not set"; specimen in Archivo with label | "Not set" |
| Font not in the open catalog | Name, source label, "Shown in {substitute}" | Substitute rule | Substitute rule | Substitute rule |
| Type scale not detected | Shows defaults labeled "default" | Labeled | n/a | Labeled |
| Detected value with low confidence | Shown with "low confidence" and its evidence | Not marked (the user reviewed it) | — | — |

### 3.5 Wording rules inside kits

- Kits are titled "Brand kit". Never "Official brand guidelines", "approved", or "official" (E-2).
- Logo rules are "Starter rules computed by Kitsheet…" (E-3).
- Contrast disclaimer as above (E-4). Font-source disclaimer (E-5).
- Source domains are plain text, never links (E-1, S-6).

---

## 4. Architecture

### 4.1 Diagram

```
 Visitor's browser (all analysis, all uploads, all outputs)                Cloudflare (Workers Free, no card)
 ─────────────────────────────────────────────────────────                ──────────────────────────────────
 Static app from Pages (HTML, JS, CSS, fonts, catalog.json) ◄──────────── Pages static assets (free, unlimited)

 URL flow:
  Turnstile widget (challenges.cloudflare.com) ─ token ─┐
  POST /api/extract {url, token} ───────────────────────┴──────────────► Pages Function /api/extract
                                                                         · Turnstile siteverify
                                                                         · SSRF guard + DoH check, every hop
                                                                         · do-not-fetch list, robots.txt
                                                                         · GET page HTML (≤ 2 MB, ≤ 5 hops)
                                                                         · HTMLRewriter: stylesheet, icon,
                                                                           manifest, img, ld+json refs
                                                                         · GET CSS (priority, ≤ 10 files, ≤ 3 MB)
                                                                           + @import, + manifest
                                                                         · sign ticket over ≤ 32 image URL hashes,
                                                                           bound to the caller's /64 or /32
  ◄── KITX frame: manifest JSON + raw HTML/CSS bytes ◄──────────────────┘ (nothing stored, nothing logged)
  html-ingest (DOMParser, inert) → render probe (sandboxed srcdoc iframe, no scripts, CSP blocks network)
  → css-tokens → color-roles / font-detect / identity / logo-rank
  POST /api/images {ticket, urls[≤ 5]} × 1 ────────────────────────────► Pages Function /api/images
  ◄── KITX frame: manifest + raw image bytes ◄──────────────────────────┘ (each hash must be in ticket)
  magic bytes + header dimensions → raster worker (decode, dHash, logo palette);
  SVG responses → text → §6.2 canonical pipeline (never an image/svg+xml Blob of raw bytes)
  → kit-model (+ manual overrides) → color system, type system

 Upload flows (never leave the browser):
  logo.svg → allowlist parse → probe paints → CanonNode tree → serializeCanonical → validateCanonical
           → scannable-code check → variants (on the tree)
  logo.png/jpg → raster worker (alpha, silhouette, palette)
  image → palette worker (OKLab k-means, exact-pixel medoids)
  guide.pdf → pdf.js worker (lazy) → text and fill-color parser

 Outputs (never leave the browser):
  PDF worker (PDFKit + svg-to-pdfkit, lazy) · PNG via Canvas 2D · share link (fflate zlib + dictionary D1 → base64url → #fragment)
  IndexedDB draft (this device only) · Clear my data

 Share view /k/: reads location.hash → validate (pure grammar module, no DOM parser, no probe) → scannable-code
 check → render read-only. Fetches only same-origin fonts, blocklist.json, watchlist.json, tlds.json, confusables.json.
 Any other path on <host>: static 404.html (status 404). Only /api/extract and /api/images invoke Functions.
```

### 4.2 Host, stack and why

| Layer | Choice | Why | Tag |
|---|---|---|---|
| Static hosting | Cloudflare Pages, custom domain by one CNAME at Squarespace | Static requests are free and unlimited ([cf-pages-fn-pricing]). Pages is the only Cloudflare compute product that supports a custom domain whose DNS is not on Cloudflare ([cf-migrate]). DNS for wethejamesphoto.com stays where it is; its Google Workspace MX records are untouched. | [Certain] |
| Server function | Pages Functions at exactly `/api/extract` and `/api/images` (same origin) | Shares the Workers Free quota of 100,000 requests/day per account, counted for every request that reaches a Function, rejected ones included; 10 ms CPU per request; 50 subrequests ([cf-limits], [cf-pages-fn-pricing]). | [Certain] |
| Bot gate | Cloudflare Turnstile, free | Unlimited challenges, up to 20 widgets ([ts-plans]); tokens single-use, valid 300 s ([ts-verify]). | [Certain] |
| Deploy | GitHub Actions → `./node_modules/.bin/wrangler pages deploy` (Direct Upload), pinned wrangler, separate build and deploy jobs (§4.8, §10.12) | C6. | [Likely] |
| CI | GitHub Actions on standard runners in a public repo | Free; usage beyond included amounts is blocked, not billed, without a payment method ([gh-actions-billing]). | [Certain] |
| UI framework | Preact 11.1.1 + @preact/signals 2.11.4 (MIT) | About 8.1 KB gz together (measured 2026-11-08 on 11.2.0, esbuild minify + gzip -9; re-measure on the pin). Its only `innerHTML` write is `dangerouslySetInnerHTML`, which we never use, so it works under Trusted Types. | [Certain] |
| Build | Vite 8.3.5 (MIT), multi-page | Static multi-page output: `/`, `/k/`, flat static pages, `404.html`. `build.target` is the §4.6 floor. | [Certain] on version |
| Language | TypeScript 7.0.2 (Apache-2.0), type-check only | If a plugin or editor integration breaks on 7.0, pin 6.0.3. | [Certain] on versions |
| Tests | Vitest 5.0.5, Playwright 1.64.1, @axe-core/playwright 4.14.0 | §15. | [Certain] on versions |

**Version pins.** Every pin in this spec is the newest release at least 7 days old, read from registry.npmjs.org on 2026-11-08 [Certain] (newer releases then: preact 11.2.0, vite 8.3.6, vitest 5.1.0, Playwright 1.65.0, pdfjs-dist 6.5.438, wrangler 4.158.0). They are re-pinned the same way at build start, and the `.npmrc` cooldown enforces it (§10.12).

**Runtime libraries.** Sizes are minified + gzip -9, measured by the research tracks or by this spec on 2026-11-08.

| Library | Version | License | Size (gz) | Loaded |
|---|---|---|---|---|
| preact + @preact/signals | 11.1.1 / 2.11.4 | MIT | ~8.1 KB | Initial (app and share view) |
| culori (`culori/fn`, 14 modes) | 4.0.2 | MIT | 11.6 KB | Engine chunk; share view |
| fflate (`zlibSync` / `unzlibSync`) | 0.8.3 | MIT | 5.0 KB encode+decode; 2.3 KB decode only | Share chunk; share view (decode). Does not verify Adler-32 and truncates silently into a fixed `out` buffer; `codec.ts` adds both checks (§7.4) [Certain, source read] |
| valibot | 1.5.0 | MIT | ~1.7 KB for the schema | Share chunk; share view |
| pdfkit (browser build) | 0.20.2 | MIT | ~213 KB | PDF worker, lazy |
| svg-to-pdfkit | 0.1.8 | MIT | 19.5 KB | PDF worker, lazy. Unmaintained since 2022: pin, and feed it only canonical SVG. |
| pdfjs-dist, **legacy build** (`legacy/build/pdf.min.mjs`, `legacy/build/pdf.worker.min.mjs`) | 6.4.299 | Apache-2.0 | 155.5 KB main + 395.1 KB worker (gzip -9, measured 2026-11-08 on 6.5.438; re-measure on the pin) | Only after a PDF is chosen, and only when the §6.4 gate passes |
| svgo (`svgo/browser`) | 4.1.0 | MIT | ~166 KB | Only for "Simplify logo for the link" |
| bidi-js | 1.1.0 | MIT | Record at build (§10.12) | PDF worker, lazy, only when a drawn string has right-to-left characters (§7.1.7) |

Not used: DOMPurify (C4), any CSS parser library (the browser's CSSOM does it), any quantization library (MMCQ returned colors not present in the image, EXT-4), pdf-lib (no kerning, WOFF2 crash), jsPDF, html-to-image family, satori, lz-string, colorjs.io, apca-w3 (restrictive license).

**Own code, no library:** the scannable-code detector (`src/engine/svg/code-detect.ts`, rules-based, about 1.3-4 KB gz; §6.2.6), the canonical SVG grammar (§6.2.2), the CSS URL tokenizer (§5.4.2), `containsContact()` (§6.5.5) and the Adler-32 check (§7.4).

**Build-time only:** wrangler 4.156.0 (MIT OR Apache-2.0) as an exact devDependency, invoked only as `./node_modules/.bin/wrangler` (§10.12); fonttools `pyftsubset` and `varLib.instancer` (MIT) in the font-mirror workflow, installed with `--require-hashes`; zizmor (pinned) for workflow linting; `@capsizecss/metrics` 4.4.0 (MIT) optional for fallback metrics; a license collector for `/credits`. Dev-only, never shipped: an oracle QR decoder for test fixtures (for example jsQR, Apache-2.0, §15.7) and veraPDF (report only, §7.1.2). Build-time tools are pinned like runtime libraries.

### 4.3 Repo structure

```
brand-kit-extractor/
├─ README.md                    what it is, how to run, link to SPEC
├─ LICENSE                      MIT (code only)
├─ NOTICE.md                    reserves the Kitsheet name, logo and any photography (O-1)
├─ CONTRIBUTING.md              inbound = outbound licensing
├─ SECURITY.md                  report vulnerabilities to CONTACT_EMAIL
├─ package.json, package-lock.json   exact versions only; "packageManager"/"engines" require npm ≥ 11.10
├─ .npmrc                       min-release-age=7 (§10.12)
├─ tsconfig.json, vite.config.ts, vitest.config.ts, playwright.config.ts, eslint.config.js
├─ wrangler.toml                production Pages project (§4.8)
├─ .github/
│  ├─ dependabot.yml            npm, pip (/scripts/font-mirror) and github-actions; weekly, grouped; cooldown 7 days
│  ├─ CODEOWNERS
│  └─ workflows/                every workflow: top-level permissions {}; no workflow_run, pull_request_target or issue_comment
│     ├─ ci.yml                 on pull_request, push and workflow_call: lint, typecheck, unit, build, budgets, licenses,
│     │                         no-eval, release-age, fn-bundle, copy lint, zizmor, workflow grep, e2e ×3 engines, security
│     ├─ deploy.yml             on push to main and workflow_dispatch: jobs ci → build → deploy (environment "production")
│     ├─ font-mirror.yml        manual + quarterly; job build (read-only) → job publish (contents: write); Release asset
│     └─ monthly-check.yml      opens an issue on the 1st with the $0 checklist and workflow states (§11.5)
├─ docs/
│  ├─ SPEC.md                   this file
│  ├─ compliance/               data-flow.md (incl. the Turnstile cookie/storage allowlist), art3-note.md,
│  │                            childrens-code.md, uk-osa-note.md, logging.md, change-triggers.md, claims-map.md
│  └─ ops/                      runbook.md, takedown.md, monthly-check.md, dns-setup.md, design-review.md
├─ public/                      copied verbatim to the deploy root
│  ├─ _routes.json  _redirects  robots.txt  sitemap.xml      (_headers is generated, below)
│  ├─ blocklist.json  watchlist.json  tlds.json  confusables.json
│  ├─ og/home.png  og/share.png
│  └─ favicon.svg  favicon.ico  apple-touch-icon.png
├─ functions/api/extract.ts  functions/api/images.ts    one line each: create…Handler(PROD_CONFIG, REAL_DEPS)
├─ server/                      handler.ts (factories) config.ts (frozen PROD_CONFIG) deps.ts budget.ts guard.ts doh.ts
│                               fetcher.ts robots.ts discover.ts css-scan.ts ticket.ts turnstile.ts limiter.ts frame.ts
│                               headers.ts hosts.ts (CDN list, registrable domain) do-not-fetch.json
├─ dev/                         config.ts (DEV_CONFIG, marker __KITSHEET_DEV__) worker.ts wrangler.toml; never deployed (§4.9)
├─ src/
│  ├─ app/                      gate.ts (first module, §4.6), entry.tsx, state machine, screens (Start, Working, Review), export panel
│  ├─ share/                    gate use, entry.tsx, bootstrap.ts (hash repair), viewer
│  ├─ pages/                    privacy.html terms.html acceptable-use.html credits.html accessibility.html bot.html
│  │                            report.html clear.html 404.html (flat files, §10.6)
│  ├─ ui/                       tokens.css, components, fit-headline.ts (§12.2)
│  ├─ engine/                   fetch-client html-ingest render-probe css-url-rewrite css-prepass css-tokens color-core
│  │                            color-roles font-detect logo-rank identity platform constants
│  │  ├─ kit-model/             merge, contact.ts (containsContact, §6.5.5), match-key.ts (skeletons, §7.4)
│  │  ├─ svg/                   allowlist.ts canonicalize.ts (app route only) canonical-grammar.ts (pure; also /k/)
│  │  │                         variants.ts simplify.ts code-detect.ts rasterize.ts raster-fallback.ts
│  │  ├─ raster/                worker.ts (decode, alpha, silhouette, dHash)
│  │  ├─ palette/               worker.ts kmeans.ts
│  │  ├─ pdf-guide/             worker.ts parse.ts patterns.ts gate.ts
│  │  └─ systems/               ramps.ts contrast.ts type-scale.ts fallbacks.ts
│  ├─ outputs/  pdf/ (worker.ts, layout.ts, struct.ts, coverage.ts, bidi.ts, pages/*.ts)  png/board.ts
│  │            share/ (codec.ts adler32.ts dict-d1.ts schema.ts normalize.ts blocklist.ts)
│  ├─ store/                    idb.ts draft.ts prefs.ts
│  └─ security/                 trusted-types.ts (one lazy function per policy), csp.ts (the single CSP table)
├─ scripts/                     font-mirror/build_mirror.py (+ requirements.txt with hashes), check-font-tar.py,
│                               check-budgets.mjs, check-licenses.mjs, gen-notices.mjs, check-no-eval.mjs,
│                               check-release-age.mjs, check-fn-bundle.mjs, gen-headers.mjs, check-copy.mjs
└─ tests/
   ├─ unit/                     vitest (incl. handler-level tests against PROD_CONFIG, §4.9)
   ├─ fixtures/synthetic/       ~35 mini-sites we wrote, each with expected.json
   ├─ fixtures/share-links/     golden links per format major, made in Chromium, Firefox and WebKit
   ├─ fixtures/malicious/       svg, pdf, html, images, css
   ├─ fixtures/codes/           scannable-code positives (each verified by the dev-only oracle decoder) and negatives
   ├─ fixtures/scripts/         names in Vietnamese, Greek, Cyrillic, Japanese, Arabic, Hebrew (§15.5)
   ├─ e2e/  security/
   └─ corpus/                   manifest.json, expected/*.json, competitor.json (facts only), run-corpus.ts;
                                snapshots/ is gitignored
```

### 4.4 What runs where

| Work | Where | Why |
|---|---|---|
| URL validation, SSRF guard, DoH check, robots.txt, do-not-fetch, fetching HTML/CSS/images, link discovery, ticket signing, Turnstile verification | Pages Function (`/api/extract`, `/api/images`) | Browsers cannot fetch arbitrary cross-origin pages, and the visitor's IP must never reach the scanned site (PRIV-03). |
| Everything else: parsing, cascade resolution, scoring, logo ranking, palette, SVG, PDF parsing, color and type systems, PDF/PNG generation, share encoding and decoding, storage | Browser (main thread for DOM work; Web Workers for decode, palette, pdf.js, PDF generation) | Keeps uploads on the device (D18), keeps server CPU under 10 ms, and keeps the server free of any analysis that could be logged. |

### 4.5 Performance budgets (enforced in CI by `check-budgets.mjs` reading the Vite manifest)

| Asset | Budget (gz) | Loaded when |
|---|---|---|
| `/` HTML | ≤ 15 KB | First load |
| `/` initial JS | ≤ 60 KB | First load |
| `/` CSS | ≤ 15 KB | First load |
| UI fonts | 3 WOFF files (Archivo 400, Archivo 800, JetBrains Mono 400), ≤ 160 KB total [Guessing: Latin + Latin-Ext WOFF faces measured 30-62 KB each for other families]; 2 preloaded. The mirror now carries each family's full non-CJK subsets (§7.1.3), so Archivo includes Vietnamese; if that breaks this budget, preload a separate Latin-only UI subset for the app chrome and keep the full mirror files for kits | First load |
| Engine chunk | ≤ 60 KB | On **Read site** focus, or on file choice |
| Share codec chunk | ≤ 8 KB | When the share button first becomes enabled (idle prefetch) |
| `/k/` total JS | ≤ 40 KB, including the scannable-code detector (about 1.3-4 KB) and the canonical grammar module. If the detector pushes `/k/` over budget, cut viewer code, never the detector | Share view |
| `tlds.json`, `confusables.json` | ≤ 8 KB gz each [Guessing; measure] | With `blocklist.json`, on `/k/` and when the share codec loads on `/` |
| Font catalog JSON | ≤ 30 KB | When a font picker opens |
| PDF worker | ≤ 260 KB; bidi-js loads separately and only for right-to-left text; Noto fallback faces load only when a drawn string needs them (§7.1.7) | On hover/focus of **Download PDF**; not prefetched when `navigator.connection.saveData` is true |
| pdf.js (legacy build) | ≤ 160 + 400 KB | Only when a PDF is chosen |
| SVGO | ≤ 170 KB | Only on **Simplify logo for the link** |

Runtime targets on a mid-range phone profile (Playwright, 4× CPU throttle, Fast 4G): LCP ≤ 2.5 s on `/` and `/k/`; INP ≤ 200 ms for edits on the review screen; CLS ≤ 0.1; client extraction time p95 ≤ 2.5 s; PDF generation ≤ 3 s for a full kit [Guessing; measure]; PNG ≤ 1.5 s [Guessing; measure]. Server: `/api/extract` and `/api/images` CPU p99 ≤ 8 ms (limit 10 ms) [Guessing]. `npm run dev:snapshots` runs the real handler in workerd against snapshot bytes (§4.9); profile per-request CPU with DevTools (`wrangler dev`, press D) on all 30 corpus snapshots and record p99. Local CPU is indicative only, because Cloudflare notes local CPU differs from the edge. If local p99 exceeds 6 ms, apply the §5.2.3 step-down table. Production Functions metrics (CPU p50/p99, Exceeded Resources) become the gate at M10 and in the monthly check.

### 4.6 Browser support

- Supported: current and previous major versions of Chrome, Edge, Firefox, Safari (macOS, iOS, iPadOS) and Samsung Internet.
- Hard floor, feature-tested and never user-agent-sniffed: `typeof String.prototype.isWellFormed === 'function'` and `new OffscreenCanvas(1,1).getContext('2d') !== null`. Together these imply module workers, `inert`, DOMParser and structuredClone. The resulting floor is Chrome/Edge 111, Firefox 119, Safari/iOS 16.4, Samsung Internet 22 [Certain: MDN BCD 8.1.8]. Firefox 115 ESR (Windows 7/8.1, macOS 10.12-10.14) is below the floor and gets E-OLD.
- Implementation: `src/app/gate.ts` is the first module on `/` and `/k/`. It runs the tests, then `import()`s the app entry, or renders E-OLD. Vite `build.target` is `['chrome111','edge111','firefox119','safari16.4']`, so every browser that passes the gate can parse the bundle. `index.html` carries the E-OLD paragraph with `hidden`; a 3-line classic `<script nomodule src="/assets/old.js">` removes `hidden`, which covers pre-module browsers. No polyfills.
- The PDF-guide input has its own, higher gate (§6.4): officially supported for PDF reading are Chrome/Edge 125+, Safari 18+ and Firefox ESR+, the pdf.js legacy-build targets [Certain: pdf.js FAQ].
- Trusted Types enforcement is a no-op in browsers without it (Baseline since February 2026: Chrome 83, Firefox 148, Safari 26 ([mdn-tt])) [Certain]; the code is written so it works either way.
- `Intl.Segmenter` missing (Firefox before 125): count code points instead of graphemes (stricter, never looser), and in §7.1.7 treat each code point as its own cluster.
- `light-dark()` and relative color syntax are "newly available" Baseline only ([web-features]) [Certain]. Unsupported values drop at parse time and the sentinel resolver reports "unresolved" rather than a wrong color.

### 4.7 Build constants (`src/config.ts`, `server/config.ts`)

| Constant | Value |
|---|---|
| `TOOL_NAME` | `Kitsheet` |
| `PROD_HOST` | `kitsheet.wethejamesphoto.com` |
| `SHARE_ORIGIN` | `https://kitsheet.wethejamesphoto.com` (owner may switch to the `*.pages.dev` host, §0.3; decide before M8, because it cannot change after launch) |
| `ROBOTS_TOKEN` | `Kitsheet` (letters only, per RFC 9309 product-token rules ([rfc9309]) [Certain]) |
| `USER_AGENT` | `Mozilla/5.0 (compatible; Kitsheet/1.0; +https://kitsheet.wethejamesphoto.com/bot)` |
| `CONTACT_EMAIL` | A free Gmail address used only for Kitsheet, for example `kitsheet.<something>@gmail.com` (owner picks; §9.0, §17.2 Q4). Also the fetcher's `From:` header (§10.2) and the SECURITY.md contact |
| `PORTFOLIO_URL` | Owner to supply (§17.2) |
| `FONT_MIRROR_VERSION` | e.g. `2026-11`; fonts are served from `/fonts/{version}/…`. Empty until the first mirror release (M2); the deploy skips the mirror step while it is empty |
| `FONT_MIRROR_SHA256` | SHA-256 of that release's tarball, committed by the owner from the publish job's summary (§7.1.3) |
| `LAUNCHED` | `false` until M10 (§16). While false, production serves only a static holding page and no Function |
| Fetch caps | §5.2.3 |
| Share caps | §7.3 |

### 4.8 Project configuration and domain setup

**`wrangler.toml`** (read by `wrangler pages deploy` for Pages projects [Likely]):

```toml
name = "kitsheet"
pages_build_output_dir = "dist"
compatibility_date = "<first deploy date, YYYY-MM-DD>"
compatibility_flags = ["global_fetch_strictly_public"]
```

**`deploy.yml`** (triggers: `push` to `main` and `workflow_dispatch` only; top-level `permissions: {}`):
1. Job `ci`: `uses: ./.github/workflows/ci.yml` (ci.yml also runs `on: workflow_call`).
2. Job `build`: `needs: ci`; `permissions: contents: read`; no environment and no secrets. Checkout with `persist-credentials: false`; `npm ci --ignore-scripts`; `npm run build`. If `FONT_MIRROR_VERSION` is empty, skip the mirror steps (M0-M1). Otherwise: `gh release download font-mirror-{version}`; `echo "$FONT_MIRROR_SHA256  font-mirror.tar.gz" | sha256sum -c -` before reading the file at all; `python3 -I scripts/check-font-tar.py` validates and extracts into `dist/fonts` (§7.1.3 step 8); then `find dist -type l` must be empty, `test ! -e dist/_worker.js` must pass (advanced mode would replace `/functions` and take every request [Certain: Cloudflare docs]), and the total file count must stay under 19,000. While `LAUNCHED` is false the build writes a staging directory with only the holding page, `_headers` and a `robots.txt` with `Disallow: /`, and no `functions/` folder (§16). Upload `dist` (or the staging directory) as an artifact.
3. Job `deploy`: `needs: build`; `environment: production` (deployment branches limited to `main`, required reviewer = owner); `permissions: contents: read`. Checkout with `persist-credentials: false` for `functions/`, `wrangler.toml` and the lockfile; `npm ci --ignore-scripts`; download the artifact; run `./node_modules/.bin/wrangler pages deploy dist --project-name kitsheet --branch main` from the repo root so `functions/` is compiled and uploaded with the static files [Likely that wrangler deploy works after `npm ci --ignore-scripts`; confirm in M0]. `CLOUDFLARE_API_TOKEN` is set in that step's `env:` only. The job fails if wrangler's output reports any ignored or invalid `_redirects` or `_headers` rule.

Only `main` deploys; no preview branches are ever deployed. Direct Upload cannot later be switched to Git integration ([cf-direct-upload]) [Certain]. The `*.pages.dev` hostname always invokes Functions and cannot be turned off; the Host check (§5.2.1) answers it with 404.

**DNS facts on 2026-11-08** (live DNS and Verisign RDAP queries by the infra research) [Certain]: registrar Squarespace Domains LLC; nameservers `ns-cloud-d1…d4.googledomains.com` (Squarespace's default DNS); DNSSEC off; **no CAA records**; no wildcard; MX to Google Workspace (`aspmx.l.google.com`); `www` CNAME to `ext-sq.squarespace.com`; domain expires 2027-08-24.

**Steps** (one record; nothing else changes):
1. Create a dedicated Cloudflare account for this tool only. Turn on 2FA. Add no payment method.
2. Create the Pages project by running the deploy workflow once. Confirm `https://<project>.pages.dev` loads.
3. Pages project › Settings › Runtime › **Fail open**.
4. Pages project › Custom domains › Set up a domain › `kitsheet.wethejamesphoto.com`. Cloudflare shows the CNAME target. Pages does not need the parent domain to be a Cloudflare zone for a subdomain ([cf-pages-domains]) [Certain].
5. Squarespace › Domains › wethejamesphoto.com › DNS › DNS Settings › Custom Records › Add record: type **CNAME**, name `kitsheet`, data `<project>.pages.dev` › Save. (Name cannot be `@`; data cannot contain `/` or `:`) ([sq-dns]) [Certain].
6. Wait for **Active**. Pages issues the certificate by HTTP validation, so no TXT record is needed; if it is still verifying after 15 minutes, check that nothing redirects `/.well-known/acme-challenge/*` and that no CAA record exists ([cf-pages-debug]) [Certain].
7. Verify: `dig +short CNAME kitsheet.wethejamesphoto.com` and `curl -sI https://kitsheet.wethejamesphoto.com`.
8. Turnstile › Add widget › hostname `kitsheet.wethejamesphoto.com` only › managed mode. Put the secret in Pages › Settings › Variables and Secrets (encrypted, production). The production project holds only `TURNSTILE_SECRET` and `TICKET_HMAC_KEY` (checked at M10).
9. Do **not** change nameservers, apex records, `www`, MX or TXT records, DNSSEC, or add URL forwarding on the subdomain. Repointing the CNAME later deactivates the custom domain until it re-validates ([cf-pages-domains]) [Certain].

**GitHub repo settings** (M0, before the first workflow runs):
- Actions › General › Workflow permissions: read-only default token.
- Actions › General › Fork pull request workflows: "Require approval for all external contributors".
- Environments › `production`: deployment branches limited to `main`; required reviewer = owner.
- Releases: turn on immutable releases before the first `font-mirror` release. Releases created earlier stay mutable [Certain: GitHub changelog 2025-10-28].

If the Squarespace panel shows "custom nameservers" or DNS Connect instead of editable custom records, add the CNAME at whichever provider `dig NS wethejamesphoto.com` names [Likely that Squarespace's panel is the right place; confirm in the dashboard]. If anyone adds CAA records later, they must allow `letsencrypt.org`, `pki.goog; cansignhttpexchanges=yes` and `ssl.com`.

### 4.9 Run modes

The production handler must be testable without ever loosening its checks. No environment except production runs the real Function on the real host, so the code is split into one handler and two entries.

1. **Handler factories.** `server/handler.ts` exports `createExtractHandler(cfg, deps)` and `createImageHandler(cfg, deps)`.
   - `deps` is `{ transport(req: Request): Promise<Response>, resolve(host): Promise<string[]>, siteverify(body: FormData): Promise<unknown>, now(): number }`.
   - `guardedFetch` (§10.2) makes outbound requests only through `deps.transport` and `deps.resolve`, each wrapped by `budget.take(kind)` (§5.2.3). It never calls the global `fetch` directly. The real deps are global `fetch` and the DoH resolver in §10.2 step 6.
   - The §10.2 rules (`checkUrl`, the address ranges, the redirect rules) are identical in every mode. No configuration can loosen them.
2. **Production entry.** `functions/api/extract.ts` and `functions/api/images.ts` contain only `onRequestPost = create…Handler(PROD_CONFIG, REAL_DEPS)`.
   - `PROD_CONFIG` is a frozen constant in `server/config.ts`: Host `PROD_HOST`, Origin `https://PROD_HOST`, Turnstile `hostname === PROD_HOST` and `action === 'extract'`, both required.
   - The only env values read are `TURNSTILE_SECRET` and `TICKET_HMAC_KEY`. Nothing under `functions/` or `server/` reads `NODE_ENV`, `ENV`, `MODE` or any other mode variable; an ESLint rule enforces this.
   - Unit test: the `PROD_CONFIG` handler, given a stub siteverify that returns Cloudflare's documented test-secret response `{success:true, hostname:'example.com', 'error-codes':[]}`, must return 403 `human_check_failed`. A test secret set by mistake in production therefore fails closed (and §5.2.1 also refuses known test-secret prefixes outright).
3. **Dev entry (never deployed).** `dev/config.ts` exports `DEV_CONFIG` and the marker string `__KITSHEET_DEV__`. `dev/worker.ts` is a Workers script that serves `../dist` as static assets and routes `/api/extract` and `/api/images` to `create…Handler(DEV_CONFIG, deps)`.
   - `DEV_CONFIG`: Host `localhost:8788`, Origin `http://localhost:8788`.
   - Turnstile in dev: accept only `success === true`, `hostname === 'example.com'` and no `action` field, and only when the secret equals the documented always-pass test secret `1x0000000000000000000000000000000AA`. With any other secret, refuse to start.
   - `dev/wrangler.toml`: the same `compatibility_date` and `global_fetch_strictly_public` flag as production, plus `workers_dev = false`, no routes, and `[dev] port = 8788`.
   - `npm run dev:live`: real transport, real DoH, test keys (sitekey `1x00000000000000000000AA`).
   - `npm run dev:snapshots`: a transport that serves `tests/corpus/snapshots/` and `tests/fixtures/synthetic/` bytes by URL, plus a stub resolver.
   - `deploy.yml` only ever runs `wrangler pages deploy`, and the CI token is Pages-scoped, so the dev Worker cannot be deployed by CI.
4. **CI check `scripts/check-fn-bundle.mjs`.** Build the production Functions bundle (`wrangler pages functions build` into a temp directory). Fail if it contains `__KITSHEET_DEV__`, `1x0000000000000000000000000000000AA`, `XXXX.DUMMY.TOKEN.XXXX` or `localhost:8788`.
5. **No production bypass.** There is no production Turnstile bypass of any kind, including for tests: no test key, skip header, allowlisted IP or debug flag (§10.4).


---

## 5. Extraction engine

### 5.1 Pipeline

1. **Server** fetches bytes only: the page, its stylesheets, its manifest, then, in one batched call, up to 5 images the client asks for. It does no color, font, logo or name logic.
2. **Client** parses the HTML inertly, rebuilds the page with its CSS inside a sandboxed iframe that cannot run scripts or load anything, and reads computed styles. The browser's own CSS engine resolves the cascade, `var()` chains, `@layer`, nesting, `color-mix()`, relative colors and `light-dark()` at zero library cost (EXT-1) [Certain for Chrome 153 by test; Likely for Firefox and Safari per spec].
3. **Client** scores evidence, fills role slots, detects fonts, ranks logos, picks name and tagline, then fetches the chosen images in one `/api/images` call.
4. **kit-model** merges this source with any others and applies the user's locks and edits.

### 5.2 Server: `POST /api/extract`

#### 5.2.1 Preconditions (checked in this order; failures return JSON `{error}` with header `X-Kit: 1`)

The checks run inside `createExtractHandler` with `PROD_CONFIG` (§4.9). Every request that reaches this handler has already counted against the daily quota, whatever the outcome (§10.3); the order below makes a rejected request cost one invocation and no subrequest.

| Check | Failure |
|---|---|
| `TURNSTILE_SECRET` does not start with a documented test-secret prefix (`1x000`, `2x000`, `3x000`) | 500 `misconfigured` |
| `Host` equals `PROD_HOST` (rejects `*.pages.dev` and preview aliases) | 404 `not_found` |
| Method is POST | 405 |
| `Origin` equals `https://PROD_HOST`; `Sec-Fetch-Site` is `same-origin` when present | 403 `forbidden_origin` |
| `Content-Type: application/json`; body ≤ 4 KB | 415 / 413 |
| Body is `{"v":1,"url":string ≤ 2048,"token":string ≤ 2048}` | 400 `bad_request` |
| In-memory limiter (§10.5); runs before siteverify and before any subrequest | 429 `busy` |
| Turnstile siteverify (one subrequest, counted by `budget.take`): `success === true`, `hostname === PROD_HOST`, `action === 'extract'`, `challenge_ts` no more than 300 s old. No `remoteip` and no `idempotency_key` are sent. | 403 `human_check_failed` |
| URL passes the SSRF guard (§10.2) | 422 `blocked_url` |
| Host is not on `do-not-fetch.json` (suffix match) | 200 `{ok:false, reason:'opted_out'}` |

The handler never reads, logs or forwards the `Cookie` header (Squarespace may set cookies on the parent domain; PRIV-10) and never calls `console.log` with a URL, IP or body.

#### 5.2.2 Steps

1. robots.txt for the page origin (§5.2.5). Disallowed → `{ok:false, reason:'robots_disallowed'}`.
2. GET the page with `guardedFetch` (§10.2): manual redirects, ≤ 5 hops, each hop re-guarded, DoH-checked, robots-checked for a new origin, do-not-fetch-checked.
3. Classify the response. 401, 403, 429, a `cf-mitigated: challenge` header, or a 503 with a challenge marker → `site_blocked`. 404/410 → `not_found`. Other non-2xx → `upstream_error`. Content type not `text/html` or `application/xhtml+xml` → `not_html` with the type. Over 2 MB → `too_large`.
4. Read the body into memory through a counting stream that errors at the cap. Determine `charset` from the `Content-Type` header if present; otherwise leave it unset for the client to sniff.
5. Run one `HTMLRewriter` pass (§5.2.4), fed while the body downloads rather than after buffering.
6. GET the manifest if linked (≤ 100 KB; `application/manifest+json` or `application/json`); parse `icons[]`, `name`, `short_name`, `theme_color`.
7. GET stylesheets in priority order (§5.2.6) with ≤ 6 in flight, each ≤ 1 MB, ≤ 3 MB total, ≤ 10 files. Scan each for `@import` and logo `url()`s (§5.2.4) directly on the `Uint8Array` with ASCII case-folding (`b | 0x20`): no `TextDecoder`, no `toLowerCase` copies. Follow imports to depth 2, ≤ 4 total.
8. Sign the image ticket (§5.2.7).
9. Return the KITX frame (§5.2.8).

Every outbound request (siteverify, each DoH lookup, robots.txt and its hops, the page and its hops, CSS, the manifest) goes through one `budget.take(kind)` counter; a unit test fails if any `fetch(` call bypasses it. Every step checks the remaining budget and the deadline before starting. When either runs out, remaining optional fetches are skipped and listed in `skipped[]` with a reason (`budget`, `deadline`, `robots`, `type`, `size`, `status`, `error`).

#### 5.2.3 Caps (constants in `server/config.ts`)

| Cap | Value | Tag |
|---|---|---|
| Subrequests per `/api/extract` (platform limit 50; every redirect hop, DoH lookup and the siteverify call count) | 45 hard cap through `budget.take(kind)`; 3 reserved up front for siteverify and the page host's two DoH lookups. A unit test asserts the 46th request is never issued | [Certain] on the platform limit ([cf-limits]) |
| Subrequests per `/api/images` | ≤ 45: about 5 × (≤ 3 hops + 1) + 2 DoH per distinct host + robots for origins not in `ro` | [Guessing; count in M3] |
| Redirect hops | Page 5; robots 5 for the page origin, 2 for other origins (over 2 → treat that origin as disallowed and skip it); CSS, manifest, images 3 | RFC 9309 asks crawlers to follow at least 5 for robots ([rfc9309]); the conservative skip keeps the budget [Likely] |
| HTML | 2 MB | |
| Each stylesheet / all CSS | 1 MB / 3 MB | |
| Stylesheets / imports / import depth | 10 / 4 / 2 | |
| Distinct origins contacted for CSS | 4 (page origin + 3) | [Guessing] |
| Manifest | 100 KB | |
| Images per analysis | 5 (up to 4 logo candidates + 1 favicon), in one `/api/images` call | |
| Image size | Raster 1 MB; SVG 512 KB; ICO 256 KB | |
| Timeouts | HTML 8 s; each CSS 5 s; robots 4 s; DoH 2 s; manifest 3 s; image 8 s; `/api/extract` overall deadline 15 s; `/api/images` overall deadline 15 s | Platform has no wall-clock limit while the client is connected, so the code imposes one ([cf-limits]) [Certain] |
| Concurrency | ≤ 6 outbound connections waiting for headers (a 7th queues) | [Certain] ([cf-limits]) |

Typical analysis: about 1 siteverify + 2 DoH + 1 robots + 1 HTML + 1 manifest + 3-8 CSS (+ 2 DoH and 1 robots per extra CSS host), then one `/api/images` call for 3-5 images. Worked example for `/api/extract`, an apex→www redirect plus CSS from 3 other origins: siteverify 1; apex DoH 2, robots 1, page hop 1; www DoH 2, robots 1, page 1; manifest 1; 4 same-origin CSS files 4; 3 other origins × (DoH 2 + robots 1 + CSS 1) 12; total 26. Worst case is bounded by the caps above. The fetcher requests a strict subset of what a browser requests to render the same page, plus robots.txt (C2).

**CPU step-down table.** If measured p99 CPU (§4.5) exceeds 8 ms, apply in order and re-measure after each: HTML cap 2 → 1 MB; CSS total 3 → 2 MB; imports 4 → 2. Moving scans to the client is not an option (the ticket must be built server-side).

#### 5.2.4 Discovery (server, one `HTMLRewriter` pass; no analysis)

Collect, resolving relative URLs against the first `<base href>` if present, else the final page URL:

| Selector / source | Collected |
|---|---|
| `link[rel]` | `stylesheet` (not `alternate stylesheet`; not `media="print"`), `preload` with `as="style"`, `icon`, `shortcut icon`, `apple-touch-icon`, `apple-touch-icon-precomposed`, `mask-icon`, `manifest`. Record `href`, `media`, `sizes`, `type`, and document order. |
| `style` (text) | Scan the prelude for `@import` (below) and logo `url()`s. The text itself goes to the client inside the HTML. |
| `img` (first 40 in document order) | `src`, the largest `srcset` candidate, and lazy-load attributes `data-src`, `data-lazy-src`, `data-srcset`. |
| `picture source` (inside those) | Largest `srcset` candidate. |
| `meta[property="og:image"]`, `meta[name="twitter:image"]` | URL. |
| `script[type="application/ld+json"]` (text, ≤ 64 KB each, ≤ 4 blocks) | `JSON.parse` in try/catch; collect `logo` (string, `ImageObject.url`, `contentUrl`), `publisher.logo`, `image` on `Organization` and `WebSite` nodes, including inside `@graph`. |
| Always | `/favicon.ico` on the page origin. |

**CSS scans** (cheap string scans, no CSS parser):
- `@import`: scan only the prelude, from the start of the sheet to the first `{`, because `@import` must precede all other rules except `@charset` and `@layer` statements. Capture `url(…)` or the quoted string, plus any `layer(…)`, `supports(…)` and media query, so the client can wrap the imported text in matching `@layer` / `@supports` / `@media` blocks.
- Logo backgrounds: find each case-insensitive `logo` occurrence with `indexOf`; for each, look ahead up to 500 characters before the next `}` for `url(`; collect at most 4 such URLs per analysis.

Candidate image URLs go into the ticket only when their host is one of: the page's registrable domain (from a Public Suffix List snapshot in `server/hosts.ts`, refreshed only by a manual commit); a host in the page's redirect chain; a host that served one of its fetched stylesheets; or a curated list of platform image CDNs (`static1.squarespace.com`, `images.squarespace-cdn.com`, `cdn.shopify.com`, `*.wp.com`, `static.wixstatic.com`, `cdn.prod.website-files.com`, `framerusercontent.com`, …). Coverage of that list is [Guessing]; the corpus measures how many chosen logos it drops (M4). They are deduplicated (after dropping size parameters: `?format=`, `w=`, `width=`, Shopify `_NNNx`, WordPress `-NNNxNNN`) and capped at 32, in this source priority: JSON-LD logo, `img` with a `logo` token in `class`/`id`/`alt`/`src`, other `img` in the first 40, CSS logo URLs, icons and apple-touch-icons, manifest icons, `og:image`/`twitter:image`, `/favicon.ico`.

#### 5.2.5 robots.txt

- Fetch `/robots.txt` once per origin per invocation (in-memory only; nothing cached across requests).
- Parse per RFC 9309 ([rfc9309]) [Certain on rules]: user-agent groups matched case-insensitively against `ROBOTS_TOKEN`, else `*`; longest-match `Allow`/`Disallow` with `*` and `$`; parse at least 500 KiB.
- 4xx → no restrictions. 5xx, network error or timeout → treat the origin as fully disallowed.
- Page URL disallowed → stop (E-ROBOTS). A stylesheet or image URL disallowed → skip it and list it as skipped.
- Also honor `tdm-reservation: 1` on the page response as an opt-out, the same as a disallow (F-10) [Certain on the header name; Guessing on value].

#### 5.2.6 Stylesheet priority (EXT-03)

Fetch in this order until caps are reached:
1. Same-origin stylesheets, in document order.
2. Site-specific platform CSS: Squarespace `static1.squarespace.com/static/versioned-site-css/…`; WordPress `/wp-content/themes/…` and `/wp-content/uploads/elementor/css/post-*.css`; Shopify `cdn.shopify.com/s/files/…/assets/*.css`; Webflow `cdn.prod.website-files.com/<siteId>/css/…`.
3. Everything else, in document order.

Never fetch: Google Fonts, Bunny or Fontshare CSS (family names are in the URL); Adobe Fonts kit CSS (`use.typekit.net`, `p.typekit.net`; detect only, T-4); reCAPTCHA and consent-manager CSS; Squarespace `assets.squarespace.com/universal/…` and `definitions.sqspcdn.com`. The owner's own homepage links 16 stylesheets of which one (`site.css`) is site-specific [Certain, observed 2026-11-08].

#### 5.2.7 Image ticket

`ticket = base64url(JSON) + "." + base64url(HMAC-SHA256(TICKET_HMAC_KEY, JSON))` where JSON is:

```json
{"v":1,"exp":<unix seconds, now + 60>,"n":"<16 random bytes, base64url>",
 "c":"<base64url(HMAC-SHA256(TICKET_HMAC_KEY, client network prefix))>","o":"<page origin>",
 "ro":["<origins already robots-allowed>"],"h":["<first 12 bytes of SHA-256(url), base64url>", "… ≤ 32"]}
```

The client network prefix is the full `CF-Connecting-IP` for IPv4 and its /64 for IPv6. `/api/images` rejects a ticket presented from any other prefix, so a solved ticket cannot be replayed from elsewhere. Stateless; no storage. WebCrypto only.

#### 5.2.8 Response: the KITX frame

`200` (both endpoints), headers: `Content-Type: application/octet-stream`, `Content-Disposition: attachment`, `X-Content-Type-Options: nosniff`, `Content-Security-Policy: default-src 'none'; sandbox; frame-ancestors 'none'`, `Cross-Origin-Resource-Policy: same-origin`, `Referrer-Policy: no-referrer`, `Cache-Control: no-store`, `X-Kit: 1`. No `Access-Control-Allow-*`. No upstream header is copied (`Set-Cookie` included).

Body:
```
bytes 0-3   "KITX"
bytes 4-7   uint32 big-endian: manifest length M
bytes 8..   manifest JSON (UTF-8, M bytes)
then        raw part bytes, concatenated in manifest order
```

Manifest:
```ts
interface KitxManifest {
  v: 1; ok: true;
  finalUrl: string;               // scheme+host+path only
  status: number;
  html: Part;                     // raw bytes of the page
  css: (Part & { url: string; media: string; order: number;
                 from: 'link' | 'import'; importedBy?: number;
                 layer?: string; supports?: string })[];
  skipped: { url: string; reason: 'budget'|'deadline'|'robots'|'type'|'size'|'status'|'error' }[];
  manifest?: { name?: string; shortName?: string; themeColor?: string; icons: { src: string; sizes?: string; type?: string; purpose?: string }[] };
  candidates: { url: string; source: 'jsonld'|'img'|'css'|'icon'|'apple-touch'|'mask-icon'|'manifest'|'og'|'twitter'|'favicon';
                rel?: string; sizes?: string; type?: string }[];   // ≤ 32, same order as ticket hashes
  ticket: string;
  platformHints: string[];        // e.g. ['squarespace'] from hostnames only
}
interface Part { offset: number; length: number; charset?: string }
```

Non-success outcomes are `200 application/json` `{ v:1, ok:false, reason, detail? }` with `reason` in `robots_disallowed | opted_out | site_blocked | not_found | upstream_error | timeout | too_large | not_html | dns_error | tls_error | too_many_redirects`, and `detail` limited to a status code or content type. `X-Kit: 1` is always set; its absence tells the client the Function did not run (§11.3). The client maps `upstream_error` to E-UPSTREAM, `tls_error` to E-TLS and `too_many_redirects` to E-REDIRECTS (§2.10).

### 5.3 Server: `POST /api/images`

One call per read fetches every chosen image (C19). It runs in `createImageHandler(PROD_CONFIG, deps)` (§4.9).

| Check | Failure |
|---|---|
| Same secret, host, method, origin and content-type checks as §5.2.1; body ≤ 12 KB: `{"v":1,"ticket":string,"urls":[string ≤ 2048, 1-5 items, unique]}` | 4xx as §5.2.1 |
| HMAC valid; `exp` not passed | 403 `bad_ticket` |
| `c` equals HMAC of the caller's network prefix (§5.2.7) | 403 `bad_ticket` |
| Nonce `n` not already used in this isolate (single use, best effort: memory only, per isolate) | 429 `busy` |
| For each URL: first 12 bytes of SHA-256(`url`) is in `h` | that entry `{ok:false, reason:'not_in_ticket'}` |
| For each URL: SSRF guard, DoH, robots (skipped for origins in `ro`), do-not-fetch, ≤ 3 hops; per-target limiter on its hostname (§10.5) | that entry `{ok:false, reason}` |
| For each URL: content type in `image/png, image/jpeg, image/gif, image/webp, image/avif, image/svg+xml, image/x-icon, image/vnd.microsoft.icon`; size cap by type | that entry `{ok:false, reason:'type'\|'too_large'}` |

Images are fetched with ≤ 5 in flight under the same `budget.take` counter and the 15 s deadline.

Success: one KITX frame (§5.2.8 headers and layout) whose manifest is `{v:1, ok:true, images:[{url, ok, reason?, type?, offset?, length?}]}`, one entry per requested URL in request order, followed by the raw bytes of the successful images. `type` is the validated media type. Buffering at most 5 × 1 MB is a memory copy, not parsing; M3 measures CPU and falls back to one call per image only if p99 exceeds 8 ms. No Turnstile per call: the ticket proves a verified `/api/extract` from the same network prefix within 60 s.

Client handling of each image (before any decode or display):
- Re-check magic bytes and read header dimensions (§6.1). Refuse rasters over 40 MP; cap fetched raster logos at 16 MP.
- Raw bytes only ever exist as `application/octet-stream` Blobs.
- A response typed `image/svg+xml` is decoded as UTF-8 text and goes through §6.2.1-§6.2.2 before any use. It is never turned into an `image/svg+xml` Blob; only the canonical string that passes `validateCanonical` is (§6.2.2 item 5).

### 5.4 Client: ingest and render probe

#### 5.4.1 Decode and parse

1. Split the KITX frame. Decode each part with `TextDecoder(charset)`. When the manifest gives no charset, sniff a BOM, then prescan the first 1,024 bytes for `<meta charset>` or `http-equiv` content-type (HTML spec prescan); default UTF-8.
2. `DOMParser.parseFromString(html, 'text/html')` through the Trusted Types policy `bk-inert` (§10.7). The result is inert: no scripts, no fetches.
3. From the parsed document read: `<title>`, `meta` (description, `og:*`, `twitter:*`, `application-name`, `apple-mobile-web-app-title`, `theme-color` with media), JSON-LD blocks, `generator`, `html[data-wf-site]`, body classes (for example `elementor-kit-N`), inline `WebFont.load({...})` text (read with regex; never executed), Next.js `style[data-href]`, and the `<a>`/`<img>` context needed for logo ranking.
4. **URL resolution base.** The parsed document belongs to Kitsheet's origin, so its `.href`, `.src` and `.srcset` properties resolve against the wrong base. Define `docBase` = `new URL(firstBaseHref, finalUrl)` when a `<base href>` exists, else `finalUrl`. `html-ingest` and `logo-rank` read attributes only (`getAttribute('href')` and so on) and resolve them with `new URL(attr, docBase)`; an ESLint rule bans the `.href`, `.src` and `.srcset` properties in those modules.

#### 5.4.2 Probe document

1. Clone the parsed document. Remove `script`, `noscript`, `iframe`, `frame`, `object`, `embed`, `base`, `meta[http-equiv]`, `portal`, `template[shadowrootmode]`, and every `link`.
2. Remove all `on*` attributes and `src`, `srcset`, `poster`, `data-src` attributes on media.
3. **CSS pre-pass** (`engine/css-prepass.ts`, before any CSS enters the probe; hostile-input caps, [Guessing] on values): cap total rules at 20,000; drop rules whose selector contains `:has(` or has more than 32 compound parts; cap each custom-property value at 8 KB and total custom-property bytes at 1 MB; cap the probe DOM at 20,000 elements. Dropped rules are counted and the result is marked partial when any were dropped.
4. **Absolute URLs** (`engine/css-url-rewrite.ts`, a minimal CSS tokenizer of about 100 lines handling comments, quoted strings, `url` tokens and escapes; no library): rewrite every `url(…)` token and `@import` string to an absolute URL, resolved against the part's `url` from the KITX manifest; for inline `<style>` and `style=""` in the HTML, against `docBase` (§5.4.1). Absolute third-party URLs are then blocked by both the inherited and the meta CSP, and logo URLs found in CSS match the ticket's entries.
5. Insert each fetched stylesheet as a `<style>` element **at the position of the `<link>` it came from**, with its `media` attribute, so the cascade order matches the page. Imported sheets are inserted immediately before the sheet that imported them, wrapped in `@layer`/`@supports`/`@media` blocks that match the import's conditions. Skipped sheets are dropped and reported.
6. Prepend as the first child of `<head>`: `<meta http-equiv="Content-Security-Policy" content="default-src 'none'; style-src 'unsafe-inline'">`.
7. Serialize and assign to `iframe.srcdoc` through the Trusted Types policy `bk-probe`. The iframe is `sandbox="allow-same-origin"` (never `allow-scripts`; combining both lets a frame remove its own sandbox, per MDN ([mdn-iframe])), 1280 × 800, positioned off-screen (`position: fixed; left: -10000px; top: 0`), never `display: none`, with `aria-hidden="true"`, `tabindex="-1"` and `inert` so assistive technology and keyboard focus never reach it.

Why this is safe [Certain for Chrome 153 by test; Likely elsewhere]: with no `allow-scripts`, inline scripts and handlers do not run; the inherited parent CSP (§10.6) plus the probe's own meta CSP blocked 100% of external loads in testing (0 server hits; 5 hits without CSP); the sandbox blocks navigation, popups and forms. The parent reads the probe's DOM because both are same-origin.

#### 5.4.3 Passes (EXT-08)

1. **UA baseline**: the same DOM with every `<style>` and `style=""` removed. Any computed value equal to the baseline for that element is a browser default and is never brand evidence (link `rgb(0, 0, 238)`, button font `Arial`) [Certain, observed].
2. **Light pass**: colors, backgrounds, borders and declared `font-family` values.
3. **Metric pass**: the light-pass DOM plus `*{font-family:bkm!important}`, where `bkm` is Archivo 400 added to the probe document with `contentDocument.fonts.add(new contentWindow.FontFace('bkm', arrayBuffer))` (constructed in the probe's realm; the bytes come from the same-origin mirror). Painted areas, text lengths and every size factor in §5.5.2 come from this pass, so they do not depend on the visitor's installed fonts. A binary `FontFace` is not a network fetch, so the probe CSP should not block it [Likely; verify in M4].
4. **Dark pass**, only when dark signals exist: `prefers-color-scheme: dark` rules, or token sets under `.dark`, `[data-theme=dark]`, `[data-bs-theme=dark]`, `[data-mode=dark]`. For class/attribute themes, add that class or attribute to `<html>` and `<body>` for this pass.

#### 5.4.4 Neutralize the visitor's environment (EXT-09, EXT-10)

Each of these changed results in testing [Certain]:
- Before any pass, walk the CSSOM once and store `[rule, originalMediaText]` pairs (CSSOM wrapper objects lose expando properties, so originals cannot be stored on the rule).
- Rewrite `(prefers-color-scheme: dark|light)` to `(min-width: 0px)` (true) or `(max-width: 0px)` (false) for the pass. Force fixed answers for `prefers-contrast`, `forced-colors`, `inverted-colors`, `prefers-reduced-transparency`, `prefers-reduced-data`, `hover`, `any-hover`, `pointer`, `any-pointer`, `resolution`, `dynamic-range`.
- Append last: `*,*::before,*::after{forced-color-adjust:none!important;animation:none!important;transition:none!important}`, `html{overflow:hidden!important}` (a scrollbar-free 1280 px layout on every OS) and `:root{color-scheme:light!important}` (or `dark` in the dark pass).

#### 5.4.5 Token resolver with sentinels (EXT-11)

To resolve a custom property or expression: inside the token's scope (`:root`, `body` for Elementor's `.elementor-kit-N`, or the first element matching the scope selector), set a wrapper's `color` to sentinel A, set a probe span's `color: <expr>`, read the computed value; repeat with sentinel B. If the result equals the sentinel both times, the expression is invalid at computed-value time and is reported "unresolved". Try, in order: `var(--x)`, `hsl(var(--x))`, `rgb(var(--x))`, `hsla(var(--x),1)` [Certain: these resolved the shadcn, Tailwind and Squarespace triplet forms in testing].

#### 5.4.6 Element sampler

- `TreeWalker` over `body` in document order; at most 4,000 visible elements.
- Visible: non-zero box, not `display:none`, not `visibility:hidden`, opacity > 0.05.
- Skip subtrees whose `id` or `class` matches `/cookie|consent|gdpr|ccpa|cmp|onetrust|cookiebot|cky-|osano|termly|intercom|drift|tawk|hubspot-messages|zendesk|livechat|crisp|grecaptcha|g-recaptcha|tidio|elfsight|trustindex|birdeye|podium/i` (EXT-13) [Likely; maintained list].
- Record per element: tag, role, ancestors' header/nav/footer/banner context, class tokens, direct text length, bounding box (clipped to y < 2400, the first 3 viewports), computed `color`, `background-color`, `background-image` (gradient stops and `url()`), border colors and widths, `fill`/`stroke` for inline SVG, `font-family`, `font-size`, `font-weight`, `line-height`, `letter-spacing`.
- Never trust layout for `<img>` sizes: the probe loads no images (EXT-12) [Certain].

#### 5.4.7 Watchdog

A single synchronous `getComputedStyle` call on pathological CSS cannot be interrupted, so the CSS pre-pass (§5.4.2 step 3) is the main defense. On top of it: read styles in slices of 500 elements, yielding with `scheduler.yield()` where available or `setTimeout(0)` between slices, and check the 5 s deadline at every slice and between passes. Past the deadline, discard the iframe and continue with static CSS token analysis only, marking the result partial. Measured on a synthetic 2.6 MB HTML / 2.2 MB CSS / 16,000-element page on 4 vCPU: parse 94 ms, iframe load 422 ms, 4,000 style reads 8 ms, CSSOM walk 19 ms [Certain]; a mid-range phone is likely 3-5× slower [Guessing].

### 5.5 Colors

#### 5.5.1 Normalization (`color-core`)

- culori 4.0.2 (`culori/fn`) parses every computed value (the browser returns mixed notations: `rgb()`, `oklab()`, `oklch()`, possibly `color(srgb …)`).
- Convert to sRGB hex with `toGamut('rgb', 'oklch')` (CSS Color 4 gamut mapping), never clipping. Keep the original string as provenance [Certain, tested: `color(display-p3 1 .5 0)` → `#ff7d00` mapped vs `#ff7600` clipped].
- Alpha < 1: composite over the element's painted background chain. Alpha < 0.3 after compositing: drop as overlay noise [Guessing on 0.3].
- Same color: ΔE_OK (Euclidean in OKLab) < 0.02. Keep the hex with the most evidence; never average [Likely on 0.02].
- Neutral: OKLCH chroma < 0.03 [Guessing].

#### 5.5.2 Evidence and weights (light pass; all weights [Guessing], to be tuned on the corpus, §5.13)

| Evidence | Base weight | Size factor |
|---|---|---|
| `background-color` (composited) | 3 | `min(8, log2(1 + visibleArea/1000))` |
| `color` on an element with ≥ 2 characters of direct text | 2 | `min(4, log2(1 + chars/20))` |
| Border color (width ≥ 1 px, style not none, differs from background) | 1 | `min(3, log2(1 + perimeter/200))` |
| Inline SVG fill/stroke inside header, logo or button | 2 | 1 |
| Gradient stops (from computed `background-image`) | 1 | 1 |
| Declared in a used rule but not seen painted | 0.2 | 1 (tie-breaker only) |
| `theme-color` meta or manifest `theme_color` | 3, once | 1 |

Size factors are computed from the metric pass (§5.4.3) and quantized to 0.25 steps before scoring, so sub-pixel differences between devices cannot flip ranks.

Context multiplier (take the highest that applies): button-like (`button`, `[role=button]`, `input[type=submit|button]`, `a` with class matching `/\b(btn|button|cta)\b/`, or `a` with its own non-transparent background) ×4; `h1` ×3; `h2` ×2; `h3` ×1.5; plain `a` ×2; inside header/nav/`[role=banner]` ×1.5; inside footer ×1.2; class token matching `/brand|primary|accent|cta|highlight/` ×2. Token boost ×1.5 when a painted color matches (ΔE_OK < 0.02) a used token whose name or platform mapping hints at a role (§5.6). Status colors: evidence found only inside elements whose class or role matches `/error|danger|warning|success|alert|sale|badge|notice|toast/i` is multiplied by 0.1.

#### 5.5.3 Slot assignment (EXT-20)

1. **background**: the composited background with the largest painted area among `html`, `body`, `main` and full-width wrappers in the first viewport. If no author color applies, report `#ffffff` labeled "browser default (the site sets none)". It is evidence of absence, not an invented brand color, and is labeled so.
2. **text**: most text weight among `p`, `li`, `td`, `dd`, `blockquote` and plain text containers (not headings, links or buttons). Must contrast ≥ 3:1 with background; otherwise take the next.
3. **primary**: the chromatic (C ≥ 0.03) candidate with the highest score; button-background evidence counts double here. Tie-breaks: token hint, then larger button area, then hex string. A neutral may be primary only when no chromatic candidate reaches 10% of the total non-background, non-text score **and** the neutral is a button background.
4. **secondary**: next candidate with ΔE_OK ≥ 0.08 from all filled slots and score ≥ 15% of primary; prefer large-surface or heading use.
5. **accent**: next candidate with ΔE_OK ≥ 0.08 from all filled slots and score ≥ 8% of primary; prefer link and small-element use.
6. Up to 8 further candidates appear as "Other colors on the page" with usage counts ("buttons: 3, links: 12"). They join the kit only when the user keeps them.

Thresholds 0.08, 15%, 8% and 10% are [Guessing]. A slot with too little evidence stays empty. Confidence per slot: **high** (two or more independent evidence kinds agree), **medium** (one kind), **low** (declared only).

For primary and accent, record the computed text color painted on them (button labels). The contrast check for those pairs uses that real pair; if none exists, no pair is invented (EXT-23).

If the dark pass ran, show "This site also has a dark theme: background {hex}, text {hex}" with a button to add those colors as "other".

The chosen logo's pixels (§6.3, run automatically on the top logo after `/api/images`) join the evidence with provenance `logo` and base weight 2 (EXT-39). This covers sites whose CSS carries only neutrals.

### 5.6 Platform token dictionary and defaults to discount

Role hints apply only when the token is used by a rule that matches an element.

| Platform | Detect | Tokens with role hints |
|---|---|---|
| Elementor | body class `elementor-kit-N`; generator meta | `--e-global-color-primary`, `-secondary`, `-text`, `-accent` (scoped to `.elementor-kit-N` on `<body>`) |
| Astra | class/CSS | `--ast-global-color-0` brand, `-1` alternate brand, `-2` headings, `-3` text, `-4…7` backgrounds |
| Kadence / Blocksy | CSS | `--global-palette1…9` / `--theme-palette-color-N` [Likely] |
| WordPress core | `/wp-content/`, generator | `--wp--preset--color--{primary, secondary, accent, base, contrast, …}` |
| Squarespace | `static1.squarespace.com` | `--sqsp-palette-{lightest,light,accent,dark,darkest}-color`, `--sqsp-link-color`, `--sqsp-heading-N-color`, `--accent-hsl` (comma HSL triplet) |
| Wix | generator | `--wst-button-color-fill-primary`, `--wst-color-action`, `--wst-color-fill-accent-1…4` |
| Shopify | `cdn.shopify.com` | `--color-button`, `--color-background`, `--color-foreground` (RGB triplets) |
| shadcn/ui | `--primary` with `--primary-foreground` | `--primary`, `--background`, `--foreground`, `--accent` |
| Tailwind v4 | `@layer theme, base, components, utilities` | `--color-<name>` where `<name>` is not a default palette name |
| Generic | — | names matching `/primary\|brand\|accent\|secondary\|cta\|theme\|main/` |

Discounted to 0 unless painted (EXT-22): the WordPress core default palette (for example vivid-red `#cf2e2e`, luminous-vivid-orange `#ff6900`, luminous-vivid-amber `#fcb900`), Tailwind default palette variables, shadcn `--chart-*` and `--sidebar-*`, Bootstrap default variables. The shadcn default `--primary` is near-black `oklch(0.205 0 0)`; a token named "primary" is not proof of a brand color [Certain on values].

Hostname patterns are written by us; nothing is copied from the GPL-3.0 Wappalyzer fork (EXT-58).

### 5.7 Fonts

#### 5.7.1 Selection (EXT-24)

- **Heading**: computed `font-family` of the first visible `h1`; else `h2`; else the largest computed font size ≥ 24 px in the first two viewports.
- **Body**: the family covering the most text characters in `p`, `li` and similar, excluding headings, nav and buttons.
- `getComputedStyle().fontFamily` returns the declared stack; fonts never need to load [Certain].

#### 5.7.2 Cleaning the stack (EXT-25)

Walk left to right and skip: CSS generics (`serif`, `sans-serif`, `monospace`, `cursive`, `fantasy`, `math`, `emoji`, `fangsong`, `system-ui`, `ui-*`); `-apple-system`, `BlinkMacSystemFont`; emoji fonts; icon fonts (below); next/font fallbacks (`__X_Fallback_<hex>`, and `X Fallback` after X). Decode next/font hashed names with `/^__(.+?)_(?:Fallback_)?[0-9a-f]{6,8}$/` and replace `_` with spaces [Certain for webpack 6-hex and Turbopack 8-hex; Guessing for Next 15/16, verify on a current site]. The first remaining family is the candidate. If it has no `@font-face`, no provider link and is not a known system font, label it "declared, not loaded by this page" and evaluate the next family too.

**Icon fonts** (case-insensitive substring): Font Awesome/FontAwesome, Material Icons, Material Symbols, Material Design Icons, Glyphicons, bootstrap-icons, Ionicons, icomoon, dashicons, eicons, ETmodules, fontello, Genericons, themify, simple-line-icons, Linearicons, feather, remixicon, boxicons, tabler-icons, socicon, WooCommerce, star, slick, swiper-icons, revicons, fl-icons, ElegantIcons, Flaticon. Also any family whose `@font-face` `unicode-range` lies only in U+E000-F8FF [Likely; list maintained in repo].

**System families** ("System font (varies by device)"): Arial, Helvetica, Helvetica Neue, Segoe UI, Roboto when after system aliases, San Francisco/SF Pro, Georgia, Times New Roman, Times, Verdana, Tahoma, Trebuchet MS, Courier New, Menlo, Monaco, Consolas, Ubuntu, Cantarell, Noto Sans, Liberation Sans. A stack starting `ui-sans-serif, system-ui` is reported as "System font stack".

#### 5.7.3 Source label (two fields: catalog, served from) (EXT-26)

Detection order; first match wins:
1. `link[href*="fonts.googleapis.com/css"]`, `@import` of it, or Next.js `style[data-href^="https://fonts.googleapis.com/"]` → **Google Fonts** (families from `family=`; css2 `Inter:wght@400;700&family=…`; v1 `Open+Sans:400,700|Roboto`).
2. `fonts.bunny.net/css` → **Bunny Fonts** (same v1 parser).
3. `use.typekit.net`, `use.typekit.com`, `p.typekit.net` → **Adobe Fonts**. Detect only; never fetch the kit CSS or font files (T-4). Family slugs from the site's own `font-family` values (for example `proxima-nova`) are enough.
4. `api.fontshare.com` → **Fontshare**.
5. `fast.fonts.net` → Monotype; `cloud.typography.com` → Hoefler&Co.; `*.myfonts.net` → MyFonts.
6. Inline `WebFont.load({google:{families:[…]}, typekit:{id:…}})` read with regex `families:\s*\[([^\]]*)\]` and `typekit:\s*{\s*id:\s*['"]([a-z0-9]+)`; never executed.
7. `@font-face` served from a platform CDN or the same origin → "served by {platform or the site}". If the family is in the Google catalog: "Google Fonts family, served by {platform}" (seen on the owner's Squarespace site: Almarai and Ovo from `file.squarespace-cdn.com`) [Certain on observation].
8. Known system family → "System font".
9. Otherwise no label. Never guess.

Schema mapping for `FontRef.source`: `google`, `bunny`, `adobe`, `self-hosted` (cases 4, 5, 7 when not in the Google catalog; the UI shows the specific label), `system`, `unknown`.

#### 5.7.4 Typography evidence

Record computed `font-size`, `font-weight`, `line-height`, `letter-spacing` for `h1`-`h3`, `p` and buttons at 1280 px. The type-scale fit uses them (§6.5.6).

#### 5.7.5 Rendering fonts in the kit (T-1 to T-9, PRIV-02/04)

- **Google catalog family** (exact name match, including a rename alias table, against `catalog.json`): render from our own mirror (§7.1.3). Embedding in PDFs is allowed by OFL-FAQ 1.12 ([ofl-faq]) [Certain].
- **Anything else** (Adobe, commercial self-hosted, Wix proprietary, system): show the name and source; render specimens in a labeled open-licensed substitute; never fetch or embed the site's font files.
- Substitutes: metric-compatible first for common system fonts: Arial → Arimo, Times New Roman → Tinos, Courier New → Cousine, Calibri → Carlito, Cambria → Caladea, Georgia → Gelasio [Guessing on pairings; designer review]. Otherwise by category from a curated, versioned map (geometric sans → Montserrat; humanist sans → Source Sans 3; grotesque → Inter; transitional serif → Libre Baskerville; slab → Roboto Slab; mono → JetBrains Mono; script → no substitute, shown in the fallback stack with a note) [Guessing; owner review]. Never say a substitute is "identical" or "the same".
- Before embedding any font, check OS/2 `fsType` ([os2-fstype]) [Certain]: 0 or 8 embed (subset OK); 4 embed; 2 do not embed (use substitute); bit 0x0100 embed the whole font; bit 0x0200 do not embed outlines.

### 5.8 Logos and favicon

#### 5.8.1 Candidates (EXT-29)

- `img`, inline `svg` and `picture` in header, nav or `[role=banner]`, or in the first 15% of DOM order.
- Any element whose own or ancestor (≤ 3 levels) `class`, `id`, `alt`, `src`, `aria-label` or `title` contains `logo`.
- Platform logo classes: `custom-logo` (WordPress), `wp-block-site-logo`, `elementor-widget-theme-site-logo`, `header-title-logo` (Squarespace), `header__heading-logo` (Shopify Dawn), `w-nav-brand` (Webflow).
- Elements with a logo token whose computed `background-image` is a `url()` (must be in the ticket to fetch).
- JSON-LD `Organization.logo`, `publisher.logo`, `Organization.image`, `WebSite.image`.
- `link[rel~=icon]`, `apple-touch-icon(-precomposed)`, `mask-icon`, manifest icons.
- `og:image`, `twitter:image`.

#### 5.8.2 Scoring (EXT-30; points [Guessing], signals [Likely])

| Signal | Points |
|---|---|
| Wrapped in `a[href]` resolving to the site root (`/`, `./`, origin, or a locale root like `/en/`) | +40 |
| `logo` token in self or ancestor | +30 |
| JSON-LD `Organization.logo` | +25 |
| In header/nav/banner or first 15% of DOM | +20 |
| `alt` or filename matches the site name or domain root | +15 |
| `WebSite.image` / `Organization.image` | +10 |
| SVG | +10 |
| PNG with alpha | +5 |
| Footer only, not home-linked | −30 |
| `alt` or filename names another brand, or matches `/partner\|client\|as seen\|featured\|award\|badge\|certified\|bbb\|angi\|homeadvisor\|yelp\|google.?review\|trustpilot\|gaf\|owens.?corning\|certainteed\|visa\|mastercard\|paypal\|financ/i` | −40 |
| Social icon (`facebook\|instagram\|linkedin\|youtube\|tiktok\|x-logo\|twitter\|pinterest`) or declared longest edge < 24 px | −20 |
| `og:image` near 1.91:1 with no logo token (likely a social banner) | −20 |

The badge patterns matter for DFW home-services sites, whose headers often carry manufacturer and review badges.

#### 5.8.3 Fetch, decode, dedupe (EXT-31)

1. Inline SVG candidates need no fetch: serialize them from the fetched HTML (not the probe), resolve `<use href="#id">` against same-document symbols, and send them through the SVG pipeline (§6.2.1). External sprite files are not fetched.
2. Fetch the top 4 non-inline candidates by score, plus the favicon (§5.8.4), in one `/api/images` call (only URLs in the ticket).
3. Check magic bytes and header dimensions first (§5.3 client handling). Rasters decode in the raster worker with `createImageBitmap` for true pixel size. SVG responses go through §6.2.1-§6.2.2 and are drawn with `rasterizeSvg` on the main thread (§6.2.2); `createImageBitmap` is never called on an SVG Blob. Longest edge < 32 px: not a logo (may still be a favicon).
4. Dedupe with a 64-bit dHash (32 × 32, alpha composited on white); Hamming ≤ 6 = same image; keep vector over raster, then the larger.
5. Show the top 3 with their reasons. Three-way agreement (header image, JSON-LD and `og:image` naming the same file) is high confidence (observed on the owner's site, which needs `?format=` stripped to see it) [Certain].
6. Dark-variant hints: `<picture><source media="(prefers-color-scheme: dark)">` and `-dark`/`-white` filename pairs.

#### 5.8.4 Favicon (EXT-33)

SVG `rel=icon` first, then the largest PNG ≥ 32 px by `sizes`, then `apple-touch-icon`, then manifest icons, then `/favicon.ico` ([google-favicon]) [Certain on rel values; Likely on order]. An SVG favicon is canonicalized like any SVG (§6.2) and kept only if it passes `validateCanonical`; otherwise the next candidate is used. ICO decoding is attempted with `createImageBitmap`; where the browser cannot decode ICO, the favicon is shown as "found (ICO; preview unavailable in this browser)" and not drawn in outputs [Guessing on Safari ICO support; test].

### 5.9 Brand name and tagline (EXT-34, EXT-35)

A name or tagline candidate that fails `containsContact()` (§6.5.5) is skipped silently and the next candidate is tried; Kitsheet never shows an error for text read from a site. Before scoring a name candidate, a trailing `.<tld>` is stripped (og:site_name "Booking.com" becomes "Booking").

**Name candidates**, each NFC-normalized and whitespace-collapsed: JSON-LD `WebSite.name` / `Organization.name` (Google: "WebSite structured data is most important" ([google-site-names]) [Certain]); `og:site_name`; `application-name`; `apple-mobile-web-app-title`; manifest `name`/`short_name`; the chosen logo's `alt` if not generic (not `logo`, `home`, `image`); `<title>` split on `| – — - · • : »`; the domain root as a last resort, labeled "from the web address". Pick the name with the most agreeing sources (normalized similarity ≥ 0.8). For split titles, choose the segment matching the domain root or another source, not the keyword segment ("Roof Repair Dallas TX | Acme Roofing" → "Acme Roofing").

**Tagline**, first match wins: JSON-LD `slogan`; `Organization`/`WebSite` `description` if ≤ 120 characters; first sentence of `og:description` or meta description if ≤ 120 characters, ≤ 18 words and not starting with "Welcome"; otherwise empty. Always show where it came from. An empty tagline is a normal result (the owner's own site has an empty meta description) [Certain].

### 5.10 Partial, JavaScript-built and blocked pages

- **JavaScript shell** (EXT-36): visible body text under 200 characters **and** an empty mount node (`#root`, `#app`, `#__next`, `#___gatsby`, `app-root`, `[data-reactroot]`) or a noscript "enable JavaScript" message; **or** no `h1`, `button` or `a` in the probe. Then run static CSS-token analysis only and show E-JS.
- **Blocked** (EXT-37): handled server-side (§5.2.2 step 3) plus client checks for challenge titles ("Just a moment…", "Attention Required") and login/consent walls (a `form` with a password input as the main content; a consent overlay covering the first viewport). Never retry another way (F-5).
- **Meta refresh / JS redirect stub**: detect `meta[http-equiv=refresh]` or a tiny body with a redirect script; show "This page redirects to {target}. Read that page instead?" which starts a new read only on click (with a new bot check).
- **Partial** (E-PARTIAL): any role slot empty, no fonts, or skipped stylesheets.

How often static fetching fails for this audience is unknown; the corpus measures it [Likely that builder sites mostly work, since WordPress, Shopify, Squarespace, Webflow, Wix and Framer serve rendered HTML; Certain that rendered vs raw word counts differ 18% at the median per the Web Almanac 2024 ([almanac-seo])].

### 5.11 Determinism and the never-invent invariant

- **Determinism** (EXT-05): the same fetched bytes or uploaded file produce byte-identical kit JSON regardless of the visitor's OS theme, forced-colors setting, installed fonts, scrollbar width, viewport, pointer type or run count. Enforced by §5.4.4 (including `overflow:hidden`), the `bkm` metric pass (§5.4.3), size factors quantized to 0.25, a seeded PRNG for k-means, and stable sort keys (score, then hex string). CI runs the synthetic fixture job on `ubuntu-latest`, `windows-latest` and `macos-latest` (standard runners, free for public repos) and asserts byte-identical kit JSON across the three OS jobs.
- **Never invent** (EXT-18): every color in the kit has at least one provenance record of kind `painted`, `declared+used`, `meta`, `pixel`, `svg-paint`, `pdf-text`, `pdf-fill` or `manual`. Derived values (ramps, gamut-mapped hex from out-of-gamut colors, CMYK conversions from a guide) carry `derived: true`, are labeled in the UI, and never fill a role slot by themselves. The same rule applies to fonts (provenance: `computed`, `link`, `font-face`, `pdf-text`, `pdf-embedded`, `manual`), names and taglines.
- CI property test: every output color traces to a provenance record whose raw value parses to the same hex within ΔE_OK 0.02, or is flagged `derived`.

### 5.12 Known failure modes

| Failure | Effect | Mitigation |
|---|---|---|
| Brand colors only in images (hero photo, raster logo) | CSS gives neutrals only | Logo palette (§5.5.3); prompt to add a screenshot |
| Cookie, chat, review, booking widgets | Saturated non-brand colors | Exclusion list (§5.4.6) |
| Platform defaults (WordPress presets, Tailwind/Bootstrap defaults) | False tokens | Discount list (§5.6); painted evidence only |
| Browser default styles | Link blue, Arial | UA baseline pass |
| Visitor dark mode, forced colors, hover/pointer | Per-visitor differences | §5.4.4 |
| Mid-animation values | Transparent or odd colors | Animations off |
| Invalid `var()` | Silent inherited value | Sentinel resolver |
| Gradient brands | Missed | Gradient stops at low weight; "Other colors" |
| SPA shell, runtime CSS-in-JS | Empty DOM or no CSS | E-JS |
| Bot challenge, 403, geo or consent wall | No data | Stop; uploads |
| Badge walls in the header | Wrong logo | Negative logo patterns |
| Logo as CSS background, JS-injected, external sprite | Missed | CSS logo scan; JSON-LD, og:image, icon fallbacks |
| Icon font first in stack; Typekit slugs; Wix proprietary fonts | Odd names | Filters; show slug; substitute |
| Font declared but not loaded | Names an unseen font | "declared, not loaded" label |
| > 10 stylesheets or > 3 MB CSS | Missing tokens | Priority order; skipped list |
| Non-UTF-8 pages | Garbled names | Raw bytes + client-side charset decode |
| Declarative shadow DOM / web components | Styles not visible | Out of scope for v1; flagged partial [Guessing] |
| Older browser without `light-dark()` or relative color | Unresolved tokens | "unresolved", never guessed |

### 5.13 Thresholds to tune, then freeze

All values tagged [Guessing] in §5.4 to §5.9 (ΔE 0.08, neutral C 0.03, alpha 0.3, weights, slot ratios, logo points, palette floors) are starting values. Tune them on the corpus in M9 (§16), freeze them, and bump the kit `algo` version if a later change would alter results for the same input.

---

## 6. Upload processing

### 6.1 Rules for every file

- Files are read with `File.arrayBuffer()` / `text()` and processed in the page or in a Web Worker. No request carries file bytes; a Playwright test asserts this for every upload type (§15.6).
- Type is decided by magic bytes: PNG `89 50 4E 47`, JPEG `FF D8 FF`, GIF `47 49 46 38`, WebP `RIFF….WEBP`, AVIF `ftypavif`/`ftypavis`, PDF `%PDF-`, SVG by parsing as XML with an `<svg>` root. Extension and `File.type` are hints only.
- Image dimensions are read from the header before decoding (PNG IHDR, JPEG SOF, WebP VP8/VP8L/VP8X, GIF logical screen). Over 40 MP: refuse with E-FILE. The same magic-byte and header check runs on every image returned by `/api/images` (§5.3).
- Raw file bytes only ever exist as `application/octet-stream` Blobs; typed Blobs are built only from checked bytes (and, for SVG, only from canonical strings, §6.2.2).
- Every file input is a real `<input type="file">` with a visible label; drag-and-drop is an extra (WCAG 2.5.7).
- Source files are never stored, except the chosen raster logo and favicon (as Blobs in IndexedDB, §7.6).

### 6.2 Logo

#### 6.2.1 SVG intake (pre-probe)

1. Read as text. Refuse if > 1 MB, if it contains `<!DOCTYPE` or `<!ENTITY` (entity bombs), or if parsing yields > 10,000 elements.
2. `DOMParser.parseFromString(text, 'image/svg+xml')` via `bk-inert`. Reject on `<parsererror>`.
3. **Structural allowlist walk** (removes, never rewrites): keep only `svg g path rect circle ellipse line polyline polygon linearGradient radialGradient stop defs clipPath symbol use style title desc`. Remove everything else, including `script`, `foreignObject`, `image`, `a` (children kept), `animate*`, `set`, `text`, `tspan`, `filter`, `mask`, `pattern`, `iframe`. Record which unsupported features were present (text, image, filter, mask, pattern) for the §2.5 notice. Remove all `on*` attributes and any `href`/`xlink:href` not starting with `#`. Remove any `<style>` element or `style=""` attribute whose text contains a backslash, `@import`, or `url(` not immediately followed by `#`: CSS escapes such as `u\72l(` still tokenize as a `url` token, so string deletion is not enough. (The probe's meta CSP blocks loads anyway; the canonical output never contains CSS.)
4. Expand local `<use href="#id">` (depth ≤ 3, ≤ 500 expansions) by cloning the referenced element or symbol content into a `<g>` with `translate(x,y)` plus the symbol's `viewBox` mapping when `width`/`height` are given. Unresolvable uses are removed and noted.

#### 6.2.2 Paints via the probe, then canonical serialization (C4, OUT-31)

**Code split.**
- `src/engine/svg/canonical-grammar.ts`: pure TypeScript with no DOM, no imports from the rest of `engine/` and no Trusted Types sinks, so `src/share/` and workers can import it without pulling in the engine chunk. It exports the `CanonNode` types and:
  - `serializeCanonical(tree): string`, the only SVG serializer in the codebase (plus `serializeVariant`, below, which extends it);
  - `parseCanonical(svg, maxBytes): CanonNode | null`, linear time, accepting only the grammar in step 3;
  - `validateCanonical(svg, maxBytes = 8192): string | null`, which returns `svg` only if `parseCanonical` succeeds and `serializeCanonical` of the result equals `svg` byte for byte.
- `src/engine/svg/canonicalize.ts`: app route only. It runs DOMParser through `bk-inert`, the allowlist (§6.2.1) and the probe, builds a `CanonNode` tree, and returns `validateCanonical(serializeCanonical(tree), 65536)`. If that returns `null`, the logo is treated as unsupported and takes the raster path below.

**Steps.**
1. Mount the cleaned SVG in a fresh sandboxed probe (§5.4.2 rules; same meta CSP). The browser applies `<style>` classes, `currentColor` and inheritance.
2. For each shape, read computed `fill`, `stroke`, `stop-color`, `opacity`, `fill-opacity`, `stroke-opacity`, `stroke-width`, `visibility`, `display` and `clip-path`. **Transform source:** the computed CSS `transform` if it is not `none`; otherwise `el.transform.baseVal.consolidate()?.matrix` from the SVG DOM. Firefox does not map the `transform` attribute into computed `transform`: Chrome returns `matrix(...)` and Firefox `none` for the same element [Likely; Stack Overflow 63932958]. If a CSS `transform` comes with a non-default `transform-origin` or `transform-box`, treat the logo as unsupported (raster).
3. Build a `CanonNode` tree and serialize it with `serializeCanonical`. The grammar it emits and `parseCanonical` accepts is closed:
   - No XML declaration, comments, processing instructions, CDATA, DOCTYPE, entity or character references, text nodes, or whitespace between tags. Printable ASCII only.
   - Exactly one root `<svg xmlns="http://www.w3.org/2000/svg" …>`. No nested `svg`. No other namespace or prefixed attribute.
   - Elements: `svg g path rect circle ellipse line polyline polygon linearGradient radialGradient stop defs clipPath`. Elements with no children self-close (`<path …/>`).
   - Nesting: `defs` only as a direct child of the root; `linearGradient`, `radialGradient` and `clipPath` only inside `defs`; `stop` only inside a gradient; no `clip-path` attribute anywhere inside a `clipPath` subtree.
   - Attributes come only from each element's allowlist, with no duplicates, double-quoted, and no value may contain `& < > "` (so no escaping exists). They are emitted in this fixed order: `id`, `viewBox`, `width`, `height`, `x`, `y`, `cx`, `cy`, `r`, `rx`, `ry`, `x1`, `y1`, `x2`, `y2`, `points`, `d`, `transform`, `gradientUnits`, `gradientTransform`, `clipPathUnits`, `offset`, `stop-color`, `fill`, `fill-rule`, `fill-opacity`, `stroke`, `stroke-width`, `stroke-linecap`, `stroke-linejoin`, `stroke-opacity`, `opacity`, `clip-rule`, `clip-path`. The root emits `xmlns` first, then `viewBox` (synthesized from width/height if missing), `width`, `height` (explicit, because Firefox draws viewBox-only SVGs to canvas unreliably, OUT-13 [Likely]).
   - Numbers match `-?(0|[1-9]\d*)(\.\d{0,2}[1-9])?`: at most 3 decimals, no trailing zeros, no `+`, no exponent, no `-0`. One shared number printer serves every attribute.
   - `points`: numbers separated by single spaces, an even count.
   - `d` is re-printed, never copied: command letters as given, every number through the shared number printer, single spaces between all tokens, arc flags as `0` or `1`.
   - `transform` and `gradientTransform`: `matrix(a b c d e f)`.
   - Paints (`fill`, `stroke`, `stop-color`): `#rrggbb` lowercase, `none`, or `url(#id)`. `fill-rule` and `clip-rule`: `nonzero` or `evenodd`. `stroke-linecap`, `stroke-linejoin`, `gradientUnits` and `clipPathUnits`: their keyword sets.
   - `id` matches `^[A-Za-z][\w-]{0,31}$`, and ids are unique. Every `url(#id)` resolves to an element of the right type in the same document: `fill` and `stroke` to a gradient, `clip-path` to a `clipPath`.
   - Depth ≤ 32; elements ≤ 2,000 [Guessing; tune on the corpus]; size ≤ `maxBytes` (8,192 for links, 65,536 locally).
4. The canonical string is the only SVG the kit ever stores, renders, embeds or shares.
5. Rendering: only as `<img src="blob:…">` from `new Blob([svg], {type:'image/svg+xml'})`, and only for a string returned by `validateCanonical` or by `serializeVariant` (§6.2.3). In image context SVG runs no scripts and loads nothing external ([mdn-svg-img]) [Certain]. A test wraps the `Blob` constructor and fails if an `image/svg+xml` Blob is ever built from any other string; the single exemption is the raster-only fallback draw below, whose blob URL is never attached to the document.
6. `clipPath` in the canonical set is an addition to the outputs research's list; svg-to-pdfkit support for it is verified by the M1 spike (§16; fallback: rasterize, below) [Likely].

**Rasterizing SVG** (`engine/svg/rasterize.ts`, main thread, on `/` and `/k/`): `rasterizeSvg(svg, {maxEdge, maxPixels: 16_777_216})` creates an `Image` whose `src` is the canonical blob URL, awaits `img.decode()`, draws into an `OffscreenCanvas` sized min(requested size, 4,096 px long edge, `maxPixels`), then calls `transferToImageBitmap()` and transfers the bitmap to the worker that needs it. Every SVG (fetched, inline or uploaded) goes through it; `createImageBitmap` is never called on an SVG Blob, which Chromium has historically rejected [Likely]. Every canvas in §3.3, §6.2 and §7.1.1 is clamped to the same limits (iOS Safari's canvas cap, [pqina-canvas]).

**Paint weights.** `rasterizeSvg` the canonical SVG to 512 px; snap each opaque pixel to the nearest paint within ΔE_OK 0.04; each paint's weight is its pixel share. Colors enter the kit evidence as `svg-paint`. Then run the scannable-code check (§6.2.6).

**Unsupported features present** (text, image, filter, mask, pattern), or `validateCanonical` returned `null`: the logo is treated as raster for outputs. `engine/svg/raster-fallback.ts` applies a separate raster-only sanitizer to the **original upload**: it removes `script`, `foreignObject`, every `on*` attribute and every `href` that does not start with `#` or `data:image/`, and refuses the file if any `<style>` or `style=""` contains a backslash, `@import` or `url(`. It keeps text, filter, mask and pattern. Its output is drawn exactly once, at 4× clamped by the limits above, through an `Image` whose blob URL is never attached to the document and is revoked right after decode, onto a canvas; that PNG becomes the logo raster. The user sees the result and confirms it matches their logo (§2.5). If `toBlob` throws `SecurityError` (a tainted canvas), fall back to the "convert to outlines" message. Variants follow §6.2.4; the logo cannot ride in links.

#### 6.2.3 SVG variants (EXT-44; rule-based, editable before export)

`variants.ts` takes a `CanonNode` tree from `parseCanonical`, never a DOM, on both `/` and `/k/`. Paints are read from the tree's `fill`, `stroke` and `stop-color`, recolored in the tree, and printed with `serializeVariant(tree)`, which is `serializeCanonical` extended with the variant-only `mask` element and `mask` attribute. No HTML sink is involved, so `/k/` derives on-light, on-dark and one-color under its strict CSP. Variant strings are derived: never stored, never shared. The scannable-code check (§6.2.6) runs on each variant.

- **On dark**: background = the kit's text color if dark (L_OK < 0.35), else `#111111` labeled "reference dark". Every paint with < 3:1 contrast against it (WCAG 1.4.11 non-text threshold) becomes `#ffffff` or the kit's lightest neutral.
- **On light**: the mirror image (background = kit background if light, else `#ffffff`).
- **One color**: every ink paint becomes the chosen color (default: primary; user can pick any kit color, black or white). "Knockout" paints (near-white, L_OK > 0.95 and C < 0.02, drawn above other ink) become transparent. Implementation: a rect filled with the chosen color, masked by a recolored copy of the logo where ink is white and knockouts are black; tested to give fully transparent knockouts under the strict CSP [Certain].
- Gradients are mapped per stop. The user can toggle which paints count as knockouts. A knockout set that differs from the default rule rides in links as ShareV1 `l.k` (§7.5.2). The share view's one-color variant uses the primary color; the editor's chosen one-color color is not carried.

#### 6.2.4 Raster logos (PNG, JPEG)

- Keep the original if ≤ 4096 px on the long edge and ≤ 5 MB; otherwise downscale (high-quality resampling) to 4096 px. Every canvas used here is clamped to 4,096 px on the long edge and 16,777,216 pixels (§6.2.2).
- On-light and on-dark previews show the image as is, with a contrast warning if its dominant ink has < 3:1 against that background.
- One-color silhouette only if ≥ 10% of pixels are fully transparent: recolor opaque pixels, keep alpha. Optional "Treat white as cut-out" uses a luminance threshold (L_OK > 0.95).
- JPEG with a plain white background: optional "Remove white background" (flood fill from the edges at ΔE_OK < 0.03), shown as a before/after the user must confirm.
- Width < 500 px: note "This logo is small for print. Use a larger file or an SVG if you have one."
- Colors: run the palette algorithm (§6.3) on the image; results enter as `pixel` evidence.
- Raster logos never ride in links (D22).

#### 6.2.5 Favicon upload

Optional "Add favicon" accepts PNG or SVG (same pipelines). There is no generated favicon; a downscaled logo is not the brand's favicon.

#### 6.2.6 Scannable-code check

`src/engine/svg/code-detect.ts`: our own code, no runtime dependency, rules-based so D12 holds. A prototype was 130 lines and 1.3 KB gz [Certain, measured by the legal review]. Purpose: a share link must not carry a scannable QR code (or text drawn as outlines) as its "logo", which would put a phishing call to action on the owner's domain while passing every text filter.

- **Input**: an RGBA raster composited onto the background it is shown on. Binarize with a local block-mean threshold and test both polarities.
- **QR test**: scan rows for runs in the ratio 1:1:3:1:1, each module within ±50% of total/7. Check the column through each candidate's center, then cluster the centers. Flag when three clusters (a) have similar module sizes (max/min ≤ 1.6), (b) form a near-right angle (|cos| < 0.26) with legs equal within 25%, and (c) enclose a module grid that looks like data: 25-75% dark and at least 30% neighbor transitions. The timing pattern is **not** required: decoders don't need it, so an attacker can break it.
- **Generic 2D-code test** (Data Matrix, Aztec, Micro QR): flag any region of at least 15 × 15 modules whose row and column run lengths are near-integer multiples of one module size and that passes the same data-texture test [Guessing on thresholds; tune on the corpus].
- `BarcodeDetector` is not used: it is not Baseline, with partial support in Chrome and Edge only (MDN) [Certain].

Where it runs:
- **Editor**: after paint weights, on the canonical SVG and on each derived variant (on light, on dark, one color), each drawn exactly as shown with the long edge at 2,048 px [Guessing; must be at least the largest device-pixel size the share view draws a logo]. The result is an in-memory flag only.
- **Encode** (§2.9): if the flag is set, only the no-logo link is offered.
- **Decode** (§7.4 step 8a): draw the logo and each variant off-screen as the view would and run the check. If any image hits, or the check errors or runs longer than 1 s (fail closed), delete `l.s` from the decoded kit before anything renders, exports or is imported through **Edit a copy**, and show S-LOGO. PDFs and PNGs exported from `/k/` then cannot carry the logo, so they need no separate check.

Residual risk: a URL or phone number drawn as outlines inside a logo cannot be detected without OCR; the banner, blocklist and 2-business-day takedown cover it (R1).

### 6.3 Image to palette (EXT-40, EXT-41)

Runs in a Web Worker.

1. Decode with `createImageBitmap(file)`. Pass no `imageOrientation` option: Safari added `from-image` to ImageBitmapOptions only in 17.2, so passing it throws a `TypeError` on Safari 16.4-17.1, and orientation does not change the histogram. If the header dimensions (§6.1) exceed 2 MP, decode with `createImageBitmap(blob, {resizeWidth, resizeHeight, resizeQuality: 'pixelated'})` to ≤ 2 MP (nearest neighbor), so every sampled value is a real image pixel. Never use smoothing resamplers. A fixture asserts every output pixel exists in the source; if WebKit fails it, cap raster input at 16.7 MP on WebKit.
2. Skip pixels with alpha < 250. Skip near-white (L_OK > 0.97, C < 0.02) and near-black (L_OK < 0.12, C < 0.02) unless together they exceed 25% of pixels.
3. Build an exact 24-bit histogram.
4. **≤ 64 unique colors (flat art):** merge colors within ΔE_OK < 0.04, keeping the most frequent; drop anti-aliasing blends (under 2% share and within ΔE_OK 0.02 of the line between two larger clusters).
5. **Otherwise (photos, screenshots):** k-means in OKLab, K = 8, k-means++ seeding from a fixed-seed LCG, 20 iterations. Merge clusters within ΔE_OK < 0.05; drop clusters under 2%.
6. Report each cluster's **medoid**: its most frequent exact pixel color, never the centroid. In testing this returned the exact brand hex values where MMCQ (classic color-thief) returned shifted values and two colors not in the image [Certain].
7. Choose 3-6 swatches: top 2 by share; then by share × (0.5 + C) among chromatic clusters; plus the lightest and darkest neutral if each is ≥ 5%.
8. Suggested roles, labeled "suggested from image": background = largest light neutral; text = darkest with ≥ 4.5:1 against it; primary = highest share × chroma.

All thresholds here are [Guessing] except the medoid rule. The source image is discarded after the run.

### 6.4 Brand-guide PDF (EXT-46 to EXT-51)

**Loading.** `pdfjs-dist` (Apache-2.0) at the §4.2 pin (6.4.299 on 2026-11-08), lazy, **legacy build only**: `pdfjs-dist/legacy/build/pdf.min.mjs` and `legacy/build/pdf.worker.min.mjs`, never `build/`. Reason: the default build ships with no transpiling and no polyfills, and its worker calls `Map.prototype.getOrInsertComputed` (Chrome 145, Firefox 144, Safari 26.2, no Samsung Internet) and `Math.sumPrecise` (Chrome 147, Firefox 137, Safari 26.2) in font and CFF parsing. The legacy build polyfills these with core-js and targets Chrome ≥ 125, Firefox ESR and Safari ≥ 18 (pdf.js FAQ; gulpfile `ENV_TARGETS`) [Certain]. Version 6 no longer has the eval-based font path behind CVE-2024-4367 ([cve-2024-4367], fixed by [pdfjs-18015]); OSV showed no known vulnerability for 6.5.438 on 2026-11-08 [Certain]; re-check the pinned version at build start [Likely clean]. Self-host the worker, `cmaps/` and `standard_fonts/`; do **not** ship `wasm/` (C17). Never load `pdf.sandbox` or the QuickJS scripting wasm. Create the worker ourselves through the `bk-worker` Trusted Types policy and pass it as `workerPort`. CI fails if `new Function(` or `eval(` appears in shipped pdf.js files. If `check-no-eval.mjs` also flags bare `Function(`, it allowlists exactly `Function("return this")()` in the two legacy pdf.js files only: core-js's global fallback, which never runs because `globalThis` is checked first, and any execution would trip the e2e CSP-violation rule. The §15.7 malicious-PDF suite runs against the legacy files.

**Feature gate** (`engine/pdf-guide/gate.ts`). Enable **Brand guide PDF** only when all of these natives exist, because the legacy build calls them without a polyfill: `typeof Promise.withResolvers === 'function'`, `typeof AbortSignal.any === 'function'`, `typeof ArrayBuffer.prototype.transferToFixedLength === 'function'`. Otherwise show E-PDFOLD. If the dynamic import of the pdf.js chunk, worker start-up or `getDocument` rejects with `SyntaxError`, `TypeError` or `ReferenceError`, show E-PDFOLD, not E-PDF; E-PDF stays for pdf.js's own `InvalidPDFException` and `PasswordException`.

**Options:** `enableXfa:false`, `useSystemFonts:false`, `fontExtraProperties:true` (needed to read embedded font names), `isEvalSupported:false` (ignored by v6; harmless), `maxImageSize: 16_000_000`, `stopAtErrors:false`, `disableFontFace:true`, same-origin `cMapUrl` and `standardFontDataUrl`. Caps: file ≤ 25 MB, ≤ 40 pages processed, 20 s total. On the timeout or on cancel, call `loadingTask.destroy()` **and** `worker.terminate()` on our own Worker object (pdf.js terminates only workers it created itself), and create a fresh worker for the next PDF. Password prompt: none; encrypted files → E-PDF.

**Text.** Per page, `getTextContent()`; rebuild lines by rounded y (`transform[5]`), sort by x; NFKC-normalize. Patterns (starting points, tune on 5-10 real guides) [Guessing]:
- HEX: `(?<![\w#])#([0-9a-f]{6}|[0-9a-f]{3})(?![0-9a-z])` and `\b(?:hex|html|web)\b\s*[:#]?\s*#?([0-9a-f]{6})\b` (a label is required when `#` is absent, so Pantone numbers and phone fragments do not match).
- RGB: `\bR(?:GB)?\s*[:=]?\s*(\d{1,3})\s*[,/ ]\s*(?:G\s*[:=]?\s*)?(\d{1,3})\s*[,/ ]\s*(?:B\s*[:=]?\s*)?(\d{1,3})\b`, values 0-255.
- CMYK: `\bC(?:MYK)?\s*[:=]?\s*(\d{1,3})%?\s*[,/ ]\s*(?:M\s*[:=]?\s*)?(\d{1,3})%?\s*[,/ ]\s*(?:Y\s*[:=]?\s*)?(\d{1,3})%?\s*[,/ ]\s*(?:K\s*[:=]?\s*)?(\d{1,3})%?`, values 0-100.
- Pantone: `\b(?:PANTONE|PMS)\s*®?\s*([0-9]{3,4}(?:\s?[A-Z]{1,3})?|[A-Z][A-Za-z]+(?:\s[A-Z][A-Za-z]+)*\s(?:C|U|CP|UP|TCX|TPX))\b`.

Group codes into color records by spatial proximity; attach the nearest label line above ("Brand Orange"). HEX vs RGB: exact match = high confidence; mismatch = show both, flagged.

**Vector fills.** `page.getOperatorList()` gives fill colors as hex (the worker converts RGB, gray and CMYK fills) [Certain, source read]. A color found as both a fill and a text code is "confirmed". With no text codes, offer the top fills as "sampled from the PDF artwork". Image-only pages: render at 1× to a canvas and run §6.3.

**Fonts.** Find typeface names near keywords (typeface, font, typography, headline, body copy, primary, secondary) and match against the Google catalog (longest match first) plus a curated list of common commercial families. Also read embedded font names via `page.commonObjs.get(id).name`: strip subset prefixes `/^[A-Z]{6}\+/`, split style suffixes (`-Bold`, `-Regular`, `MT`, `PS`, `Std`, `Pro`), and ignore ArialMT, Helvetica, TimesNewRomanPSMT, Calibri and similar unless named in the text [Certain on prefix format; Likely on usefulness].

**Pantone and CMYK.** Pantone references are shown verbatim and never converted (Pantone data is licensed) [Likely]. CMYK converts to RGB only when the guide gives no HEX or RGB for that color, labeled "approximate (converted from CMYK, no color profile)" and `derived: true`.

The PDF file is never stored.

### 6.5 Manual entry and overrides

#### 6.5.1 Colors

- Each color row: a **hex text field** (primary control; accepts `#abc`, `abc`, `#aabbcc`, `aabbcc`, `rgb(…)`, `hsl(…)`, `oklch(…)`; parsed with culori; stored as lowercase 6-digit hex), a **native color picker** (`<input type="color">`, secondary), role select, optional name (≤ 32 graphemes), lock toggle, move up/down buttons, remove.
- Invalid input keeps the last valid value and shows "Use a hex code like #1e3a8a, or rgb(30, 58, 138)."
- Live contrast hint next to text and background roles.

#### 6.5.2 Roles

At most one each of primary, secondary, accent, background, text; neutral and other unlimited; total ≤ 8. Assigning a role already taken moves the previous holder to "other" and announces it.

#### 6.5.3 Google Fonts picker

- Data: `/fonts/{version}/catalog.json` (§7.1.3), loaded when a picker opens.
- UI: ARIA combobox with a listbox (APG pattern), type-ahead filter, category filter (sans, serif, mono, display, handwriting), virtualized list. Each visible row loads that family's `menu.woff` (only the glyphs of its name) through `FontFace` and renders the name in itself; rows announce the plain family name. Families with no menu file (Reserved Font Name, UFL and byte-for-byte families, §7.1.3) render their row in Archivo 400 with the mono tag "preview on select"; the family's 400 file loads when the row is selected.
- On selection: load the chosen weights' WOFF files; pick up to 4 weights and italic.
- The picker never calls the Google Fonts API or any third-party host (PRIV-02).

#### 6.5.4 Merge precedence (kit-model)

1. A value the user **typed or locked** is never overwritten by any source.
2. Otherwise the latest **explicit upload** for that field wins over site values (a logo upload replaces a site logo; a guide's labeled roles replace site roles).
3. A new source's colors fill **empty** role slots; the rest go to "Other colors" for the user to keep or ignore.
4. Removing a source removes every value the user did not lock that came only from it.
5. Every value keeps its provenance list; the UI shows the winning source.

#### 6.5.5 Name and tagline

Plain text inputs. Normalization on input and on decode (OUT-28): NFC; reject ill-formed UTF-16; strip C0/C1 controls and U+200B-U+200F, U+202A-U+202E, U+2060-U+2069, U+061C, U+FEFF, U+E0000-U+E007F; collapse whitespace; cap graphemes at 60 / 120.

**Contact-string filter.** Reject any value for which `containsContact()` (`src/engine/kit-model/contact.ts`, pure, shared with `/k/`) returns non-null.
- **Where it runs:** on input, on extraction (§5.9: skip the candidate silently), at share encode and at share decode (§7.4 step 8).
- **Fields:** every free-text field that renders or rides in a link: `brand.name` (`n`), `brand.tagline` (`t`), `colors[].name` (`c[i][2]`), and any `FontRef.family` that is not an exact catalog match (`h[0]`, `b[0]`).
- **Working copy.** Detection runs on a copy; the stored value stays NFC. To build the copy: NFKC; map `\p{Pd}` to `-` and U+3002 to `.`; lowercase.
- (a) **Email:** `/[^\s@]+(?:@|\[at\]|\(at\))[^\s@]+\.[^\s@]+/u`.
- (b) **URL:** any of a scheme `/[a-z][a-z0-9+.-]*:\/\//u`; `www.` or `www2.` at a token start; or a domain-shaped token matching `/[\p{L}\p{N}-]+(?:\.|\[\.\]|\(\.\)|\[dot\]|\(dot\))(\p{L}{2,63})(?![\p{L}\p{N}])/gu` whose captured label is in the IANA root-zone list (https://data.iana.org/TLD/tlds-alpha-by-domain.txt, frozen into `public/tlds.json` at build, `xn--` entries also stored in Unicode form, refreshed only by a manual commit). If `tlds.json` cannot be loaded, every domain-shaped token counts as a URL (stricter, never looser).
- (c) **Phone:** check each maximal span matching `/\+?[ .()\-]?\p{Nd}+(?:[ .()\-]{1,2}\p{Nd}+)*/gu`. If every digit group is a year (19xx or 20xx), it is not a phone number. Otherwise it is one if any digit group has 7 or more digits; or the span starts with `+` and has 8 or more digits; or it has 9 or more digits in 2 or more groups; or it is exactly 3 digits plus 4 digits.
- **Error copy** (typed input only; E-CONTACT): "Names and taglines can't include web addresses, email addresses or phone numbers. If the name ends in .com or similar, drop that ending. If it isn't a web address, add a space after the period."
- **What this does not catch:** numbers spelled out in words, and text drawn as outlines inside an SVG logo. Those are covered by the §10.8 banner, the blocklist and the report link. Accepted trade-offs: names like "Booking.com" are refused (the user drops the ending); a missing space after a period before a ccTLD-shaped word ("last.No") is refused.

#### 6.5.6 Type scale

Fit from the computed h1, h2, h3 and body sizes at 1280 px: the geometric mean of the per-step ratios, snapped to the nearest of 1.125, 1.2, 1.25, 1.333, 1.414, 1.5, 1.618 within 3%; otherwise 1.25 labeled "default". Base = body size rounded to whole px (default 16, labeled). Steps: caption = base / r, body = base, h4 = base × r, h3 = r², h2 = r³, h1 = r⁴, display = r⁵. Line heights: body 1.5, headings 1.2, display 1.1, each rounded to a 4 px grid. pt = px × 0.75.

**Fallback stacks** by category (EXT-56): sans `system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif`; serif `Georgia, "Times New Roman", serif`; mono `ui-monospace, SFMono-Regular, Menlo, Consolas, monospace`; display and handwriting map to sans or serif by the catalog's classification, never `cursive`.

#### 6.5.7 Color system computation

- **Ramps** (EXT-52): 11 stops at fixed OKLCH lightness 50 = .97, 100 = .93, 200 = .87, 300 = .79, 400 = .70, 500 = .62, 600 = .54, 700 = .46, 800 = .38, 900 = .30, 950 = .22. Hue constant. Chroma = (brand chroma ÷ max in-sRGB chroma at the brand's L and h) × max in-sRGB chroma at each stop's L and h, capped at the brand's own chroma. The brand color replaces its nearest stop unchanged. All stops `derived: true`. Tested: `#e4572e` hue stayed within 35.5°-36.6° across stops; the cap prevents over-saturated light stops seen for `#0f766e` [Certain].
- **Contrast** (EXT-53): WCAG 2.2 relative luminance with the 0.04045 threshold ([wcag-lum]); composite alpha first; compare unrounded ratios to thresholds; display ratios truncated (not rounded) to 2 decimals so 4.499 shows "4.49" and fails. Thresholds: normal text AA 4.5, AAA 7; large text AA 3, AAA 4.5; non-text UI 3 [Certain]. Check: `#777777` on `#ffffff` = 4.478, fails AA [Certain].
- **No APCA in v1** (EXT-54): WCAG 3 contrast is undecided and `apca-w3`'s license is restrictive [Certain].


---

## 7. Outputs

### 7.1 PDF brand guide

#### 7.1.1 Engine

PDFKit 0.20.2 + svg-to-pdfkit 0.1.8 in a module Web Worker, lazy-loaded (OUT-01, OUT-02). Chosen because it was the only engine tested that applies GPOS kerning (`TJ` with kern offsets), subsets fonts properly (3.7-6.9 KB per face), draws real-world SVG as vectors including `<style>` classes, runs without a DOM, supports tagged PDF, stays near 233 KB gz, and contains no `eval` [Certain, measured by the outputs track]. A test guide of 7 pages, 3 fonts, 6 vector logos and 35 swatches was 57 KB and took 451 ms in Node [Certain].

- The worker receives the kit JSON, canonical SVG strings, raster logo bytes (PNG/JPEG; other formats converted to PNG on a canvas first) and font bytes. It returns a `Blob`.
- svg-to-pdfkit only ever receives canonical SVG (§6.2.2), which limits its parsing surface to what we emit. As defense in depth, the worker runs `parseCanonical` plus the byte-equality check (`validateCanonical`, from the pure grammar module) on every SVG string before passing it on, and embeds a PNG instead if it fails. Anything canonicalization dropped is embedded as a 4× PNG, clamped to 4,096 px and 16,777,216 pixels (§6.2.2).
- Layout uses absolute positioning on the §3.2 grid through a small helper (`layout.ts`: column math, text fitting with the shared `fitHeadline` logic measured by `doc.widthOfString`, line breaking). Every draw call goes through `withStruct()` (§7.1.2).

#### 7.1.2 Document properties

`PDFDocument({ size, layout:'landscape', margins, autoFirstPage:false, tagged:true, displayTitle:true, lang:<UI locale>, info:{ Title:'<Brand> brand kit', Author:'Made with Kitsheet', Creator:'Kitsheet', Producer:'PDFKit' }, pdfVersion:'1.7' })`. Text is real text (selectable, searchable), hex values are text, reading order follows the logical layout.

**Structure tree.** `tagged:true` only sets the MarkInfo flag; PDFKit needs explicit structure elements, and content outside them must be marked as artifacts. Structure map (`outputs/pdf/struct.ts`):
- Each page is a `Sect` titled with its page name. The brand name is `H1`; section titles are `H2`; running text is `P`.
- The contrast grid is `Table` → `TR` → `TH` (with scope) and `TD`.
- Each swatch is a `Figure` with alt such as "Primary, #c8102e". The logo is a `Figure` with alt "{name} logo". Each don'ts tile is a `Figure` with alt such as "Don't stretch".
- Crop marks, hairlines, the footer, and anything svg-to-pdfkit draws inside a `Figure` are marked `Artifact` or wrapped.

In `layout.ts`, every draw call goes through `withStruct(type, opts, fn)`; an ESLint rule bans direct `doc.text` and `doc.image` calls elsewhere. Verification (M7): `pdfinfo -struct-text` shows a non-empty tree whose text matches the kit, and veraPDF (free CLI) runs in CI as a report, not a gate. PDF/UA conformance is not claimed (C18).

#### 7.1.3 Fonts for PDF, PNG and preview: the self-hosted mirror

- **Never** link `fonts.googleapis.com`, `fonts.gstatic.com`, Bunny, jsDelivr, unpkg or Fontsource from the browser, and never proxy fonts through a Function (PRIV-02; a Function would spend the daily quota on every file) [Certain on the LG München I ruling ([lg-muenchen]) and on Functions counting toward the quota].
- **Format:** static TTF instances, subset to WOFF 1.0 where the license allows modification, otherwise the upstream TTF byte for byte (catalog `x:'ttf'`). Never WOFF2 and never variable fonts for the PDF engine: WOFF2 crashed PDFKit's and pdf-lib's fontkit in testing; TTF and WOFF1 worked for all 6 families tested [Certain]. One file per style serves `FontFace` previews, canvas and PDF.
- **Pipeline** (`.github/workflows/font-mirror.yml`, manual and quarterly; two jobs, step 7):
  1. Get the family list and static TTF URLs from the Google Fonts Developer API ([gf-api]) with a free API key stored as an Actions secret. A key is required; the API returns static instances by default [Certain]. **Fallback with no key and no Google Cloud account:** check out `google/fonts` at a pinned commit; read each family's `METADATA.pb` (family, category, license, subsets); for variable-font families run `fontTools.varLib.instancer` at the standard `wght` and `ital` positions; then continue at step 3. Instancing is a modification, so on this path every Reserved-Font-Name family that is variable-only upstream follows the substitute rule. The fallback is run once in M2 so it is known to work at $0 (R10).
  2. Download each static TTF for **all** families, and read each family's declared subsets from the API or `METADATA.pb`.
  3. Branch on the license (read from the family's own license file):
     - **OFL with a Reserved Font Name** (catalog `rfn:true`): copy the API's static TTF byte for byte to `/fonts/{v}/{slug}/{weight}{i?}.ttf`; never pass it through pyftsubset or a fontTools save; catalog `x:'ttf'`; no menu file (OFL-FAQ 2.6: subsetting a webfont is modification and drops RFN use) [Certain on the FAQ; owner/attorney sign-off listed in §9.9]. If the family is variable-only upstream (for example Raleway, which ships only `Raleway[wght].ttf`), the static TTF is an instance Google generated; until the §9.9 attorney item signs off, those families follow the substitute rule (catalog `sub:true`).
     - **UFL-1.0** (the Ubuntu families): never subset or convert; ship the upstream TTF byte for byte (`x:'ttf'`, no menu file). Under UFL 1.0 a modified version that is not "Substantially Changed" must be renamed with "derivative X" appended [Likely]. If byte-for-byte shipping fails a check, leave the UFL families out of `catalog.json`.
     - **OFL without a Reserved Font Name, and Apache-2.0**: `pyftsubset` to the union of the family's declared non-CJK subsets (latin, latin-ext, vietnamese, cyrillic, cyrillic-ext, greek, greek-ext, and any non-CJK script subset of its own such as arabic, hebrew, devanagari or thai) with `--layout-features='*' --name-IDs='*' --flavor=woff` (the default keeps only name IDs 0-6, dropping license fields 13-14) [Likely]. For Apache-2.0 families also write `/fonts/{v}/{slug}/CHANGES.txt` ("Modified by the Kitsheet font-mirror workflow on {date}: subset to {subsets} with fontTools pyftsubset {version}; converted to WOFF 1.0") and copy any upstream `NOTICE` file (Apache-2.0 §4(b) and (d)); a LICENSE.txt beside the file is not a change notice.
     - **Families that declare japanese, korean or chinese-\***: mirror 400 and 700 only, unsubset, byte for byte (`x:'ttf'`), as long as each file is under 25 MiB; otherwise the substitute rule.
  4. Check OS/2 `fsType` (§5.7.5) and record it.
  5. Write `/fonts/{version}/{slug}/{weight}{i?}.woff` (or `.ttf` when `x:'ttf'`); `/fonts/{version}/{slug}/menu.woff` (only the glyphs of the family name) **only when** the family was subset in step 3 (never for `rfn`, UFL or byte-for-byte families); `/fonts/{version}/{slug}/LICENSE.txt` (full license text and copyright; OFL-FAQ 1.8 requires the full package when hosting fonts for others ([ofl-faq]) [Certain]); and `CHANGES.txt` / `NOTICE` where step 3 requires them.
  6. Write `/fonts/{version}/catalog.json`: `[{f: family, s: slug, c: category, w: [weights], i: [italic weights], l: 'OFL-1.1'|'Apache-2.0'|'UFL-1.0', rfn: bool, fs: fsType, x?: 'ttf', sub?: true, sc: [subsets], a: [former names]}]`.
  7. **Publish with integrity.** Two jobs:
     - `build`: `permissions: contents: read`. It holds the Google Fonts key, installs fonttools with `pip install --require-hashes --only-binary=:all: -r scripts/font-mirror/requirements.txt`, builds and tars the tree, and uploads the tar as a workflow artifact.
     - `publish`: `permissions: contents: write`. No pip, npm or checkout of code it runs. It downloads the artifact, runs `sha256sum`, creates a draft release `font-mirror-{version}`, uploads the asset, publishes the release, and writes the SHA-256 to the job summary.
     Immutable releases are turned on before the first release (§4.8). The owner commits `FONT_MIRROR_SHA256` beside `FONT_MIRROR_VERSION` (§4.7). The deploy downloads the pinned version (§4.8).
  8. **Deploy-side checks.** The deploy runs `sha256sum -c` against `FONT_MIRROR_SHA256` before reading the tar at all, then `python3 -I scripts/check-font-tar.py` validates every member: only regular files and directories (no symlinks, hardlinks, devices or FIFOs); no absolute paths and no `..`; every file path matches `^{version}/(catalog\.json|[a-z0-9-]+/((\d{3}i?|menu)\.(woff|ttf)|LICENSE\.txt|CHANGES\.txt|NOTICE(\.txt)?))$`; then `tarfile.extractall('dist/fonts', filter='data')`. Post-checks are in §4.8.
  9. **Mirror-build check** (fails the workflow): for any family with `rfn:true`, `l:'UFL-1.0'` or `x:'ttf'`, a shipped file's sha256 differs from the downloaded source, a `menu.*` file exists, or a `.woff` file exists; any subset Apache-2.0 family lacks `CHANGES.txt`.
- **Size and limits:** the first estimate covered Latin families only: about 1,892 families, 7,501 static variants, roughly 11,300 files and about 300 MB [Guessing on bytes, extrapolated from 6 families]. Wider subsets do not change the file count; adding non-Latin and CJK families does, and CJK files are several MB each. Re-measure the file count and total bytes at the first mirror build. Pages Free allows 20,000 files per site and 25 MiB per file [Certain ([cf-pages-limits])]. If the count or deploy time is a problem, mirror only 400, 700 and 400 italic per family (§1.5 item 8). App files must stay under the remaining file budget; CI fails above 19,000 total files.
- **Fallback faces for the PDF** (§7.1.7), mirrored at 400 and 700: Noto Sans (Latin, Greek, Cyrillic, Vietnamese; it also covers families such as Lato that lack Vietnamese); Noto Sans Arabic, Hebrew, Devanagari and Thai (OFL, no RFN; subset to their own script plus Latin); Noto Sans JP, KR and SC (their OFL reserves "Source", so they ship byte for byte under the RFN rule, including its variable-only caveat).
- **UI fonts** come from the same mirror: Archivo 400 and 800, JetBrains Mono 400 (both OFL 1.1 with no Reserved Font Name; verified in their OFL.txt on 2026-11-08 [Certain]). If the wider subsets push them past the §4.5 budget, a separate Latin-only UI subset is built for the app chrome.

#### 7.1.4 Page size

Letter or A4, chosen per §2.9; stored in `localStorage` as `bk.pageSize`.

#### 7.1.5 File name

`{slug}-brand-kit.pdf`; slug = ASCII-folded name, lowercase, `[a-z0-9-]`, ≤ 60 characters; fallback `brand-kit`.

#### 7.1.6 Download behavior

- Default: `<a download href="blob:…">`, supported in Safari iOS 13+, Chrome, Firefox and Safari 10.1+ ([bcd]) [Certain]. If testing shows iOS opening the PDF inline instead of saving, use `application/octet-stream` for the download blob [Likely workaround; verify on device].
- iOS and Android: after the Blob exists, show **Share or save PDF** calling `navigator.share({files:[file]})` (Safari/iOS 14+, Chrome Android 76+; not Firefox) [Certain]. Build the Blob first because Web Share needs a live user activation [Likely].
- In-app browsers (Instagram, Facebook, WhatsApp, LinkedIn) silently drop `blob:` downloads ([webkit-216918]) [Likely]: detect by user agent and show E-INAPP; for the PNG, also show it in an overlay so it can be long-pressed and saved.
- Revoke `blob:` URLs 60 s after use.

#### 7.1.7 Glyph coverage and direction in the PDF

The name rules allow any script, but mirrored brand fonts may lack a script, and PDFKit has no glyph fallback and no bidi.
- **Coverage check** (`outputs/pdf/coverage.ts`). For every string drawn (name, tagline, color names, source domain), split it into grapheme clusters with `Intl.Segmenter` (code points where it is missing, §4.6). Assign each cluster to the first font in this chain that has every code point in it (`hasGlyphForCodePoint`; default-ignorable characters are skipped): the chosen brand face → the script's Noto face → Noto Sans (§7.1.3).
- **Loading.** The PDF worker fetches a Noto face only when a string needs it. While one loads, the export panel shows "Loading a font for {script} letters (about {n} MB)".
- **Label.** "Some letters shown in Noto Sans {script}; {family} has no {script} letters." in mono under the cover name and on page 8. The review preview shows the same note.
- **No covering font.** Never draw `.notdef`. Show a notice in the export panel ("The PDF can't show these letters: {chars}. The PNG and the link show them.") and print the same line on page 2.
- **Direction** (`outputs/pdf/bidi.ts`):
  - Break lines in logical order first.
  - Then resolve bidi levels per line with bidi-js (MIT; size recorded per §10.12; lazy-loaded in the PDF worker), with the paragraph direction taken from the first strong character.
  - Draw each level run, split further by font run, as its own `doc.text` call, at x positions laid out in visual order (UAX #9 rule L2), with the `features` option set so PDFKit lays out the run in one fontkit call.
  - Keep the text inside each run in logical order. fontkit already reverses right-to-left runs after shaping; reordering the characters too would reverse every word twice.
  - Right-align right-to-left paragraphs within their columns. The cover's off-grid hang moves to the right edge.
  - Measure fitted widths as the sum of the run widths.
- Devanagari and Thai are claimed only after raster goldens pass (§15.5), because fontkit's complex-script shaping is untested here [Guessing].

### 7.2 PNG brand board

- Canvas 2D drawn by our own code (0 KB of libraries; OUT-09). DOM-to-image libraries rely on SVG `foreignObject` and have open Safari bugs; satori + resvg costs about 1.6 MB [Certain on sizes].
- Canvas 3200 × 2000 device px with `ctx.scale(2,2)`; 6.4 MP is under iOS Safari's 16,777,216-pixel canvas limit ([pqina-canvas]) [Likely]. Images drawn onto it come from `rasterizeSvg` or checked raster bytes, within the §6.2.2 clamps.
- `getContext('2d', {colorSpace:'srgb'})` so hex values stay exact (Chrome 92+, Safari 15.2+; Firefox ignores it and is sRGB) [Certain].
- Fonts: `new FontFace('bk-h-700', arrayBuffer)` with internal alias names; `await document.fonts.load('700 96px bk-h-700', sample)`. If it resolves empty, use the fallback stack and label "font unavailable". User-supplied family names never enter `ctx.font`.
- Scripts and direction: canvas text already uses CSS font fallback and the bidi algorithm, so no coverage engine is needed. Set `ctx.font` to the internal alias followed by the mirrored Noto aliases and a generic family (`"bk-h-700", "bk-fb-arab", sans-serif`), set `ctx.direction` from the first strong character, and print the same fallback-face label as the PDF (§7.1.7). The name is fitted with `fitHeadline` (§12.2).
- Draw only `blob:` images from in-memory bytes (no cross-origin images, so the canvas is never tainted). Give SVGs explicit width and height before drawing.
- Do not use `ctx.letterSpacing` (Safari 18.4+ only) [Certain]. The board uses no tracking.
- Export with `canvas.toBlob(cb, 'image/png')`; file `{slug}-brand-board.png`; then set canvas width and height to 0 to free memory.

### 7.3 Share links

#### 7.3.1 Format

`{SHARE_ORIGIN}/k/#1.{payload}.{len}`
- `1` = format major: fixes the codec, dictionary D1, alphabet and key meanings.
- `payload` = base64url (RFC 4648 §5, unpadded) of a **zlib** stream (deflate + 2-byte header + Adler-32, [rfc1950]) compressed with preset dictionary **D1**. The base64url alphabet is unreserved in fragments, so no percent-encoding is needed ([rfc3986-frag]) [Certain].
- `len` = payload length in base36. It detects truncation, and because it ends in an alphanumeric character, auto-linkers that trim trailing punctuation cannot cut the payload.
- Encoding: `Uint8Array.prototype.toBase64({alphabet:'base64url', omitPadding:true})` when present (Chrome 140, Firefox 133, Safari 18.2) [Certain], else a 15-line fallback.

#### 7.3.2 Codec

fflate 0.8.3 `zlibSync(json, {level: 9, dictionary: D1})` / `unzlibSync(bytes, {dictionary: D1})`. With D1, a kit without a logo encodes to about 263 characters vs 310 without a dictionary; logo kits shrink 6-25% [Certain, measured]. Native `CompressionStream` cannot use a preset dictionary; lz-string was 15-35% longer and its alphabet includes `+` and `$` [Certain].

**D1** (`src/outputs/share/dict-d1.ts`): under 4 KB; contents: the canonical JSON skeleton with every short key; the ~40 most-used Google family names written as `["Name",[400,700],0,0]`; SVG boilerplate (`<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 `, `<path fill="#`, ` d="M`). Most likely strings last. **Frozen at launch. Never edited.** A new dictionary means format `2.`.

#### 7.3.3 Caps and length tiers

| Rule | Value | Tag |
|---|---|---|
| Canonical SVG in a link | ≤ 8,192 bytes | [Likely]: a realistic outlined wordmark is 4.2-6.3 KB [Certain, measured] |
| Generator refuses a fragment over | 6,000 characters | |
| Decoder rejects before decoding | > 8,192 characters | |
| Inflate output cap | 32 KB | |
| "Fits anywhere" | ≤ 1,900 characters (Discord 2,000 for most users) | [Certain] on the Discord limit |
| "Too long for Discord" | ≤ 2,900 (LinkedIn post 3,000) | [Certain] on the limit; [Likely] that URLs count in full |
| Long link warning | > 2,900 | |
| Link without logo | always ≤ ~450 characters | [Certain, measured] |

Measured full-URL lengths with a 31-character host: no logo ~300; icon-only SVG ~410; typical single-path mark ~840; outlined wordmark 2,560-3,280; wordmark plus outlined tagline 4,100-5,850 [Certain].

**Simplify logo for the link** (SVG > 8 KB): lazy-load `svgo/browser` 4.1.0 in a worker with `floatPrecision: 1`, re-canonicalize, require `validateCanonical(s, 8192)` to pass, re-run the scannable-code check (§6.2.6), show a side-by-side preview, and require the user to accept. If still over, offer the no-logo link only.

#### 7.3.4 Share-route bootstrap (mail-gateway repair, OUT-25)

The first script on `/k/` checks `location.pathname` with an anchored regex for `^/k/(%23|%2523)1\.[A-Za-z0-9_-]+\.[0-9a-z]+$`. On match it moves the text into the hash with `history.replaceState(null, '', '/k/#' + decoded)`. `_redirects` has `/k/:payload  /k/  200` so those paths load ([cf-pages-redirects]). `:payload` matches exactly one path segment, which is all the regex accepts. The obvious `/k/*  /k/index.html  200` must not be used: Pages drops any rule whose source ends in `/*` and whose destination ends in `/index` or `/index.html`, logging "Infinite loop detected in this rule and has been ignored" [Certain: workers-sdk `parseRedirects.ts`]. Header rules are matched against the original request path, so `/k/%231.…` still gets the `/k/*` block (§10.6). Cloudflare's 16 KB URL limit leaves room for the 6,000-character cap [Certain].

#### 7.3.5 Static preview metadata

`/k/index.html` is static and never changed by script (S-4, PRIV-15): `<title>Shared brand kit · Kitsheet</title>`; `og:title` "A brand kit made with Kitsheet"; `og:site_name` "Kitsheet"; `og:description` "Made by a visitor. Not verified by or affiliated with any brand."; `og:image` `/og/share.png` (1200 × 630, flat, the tool's own branding, never a kit); `og:url` `https://<host>/k/`; `twitter:card` `summary_large_image`; `<meta name="robots" content="noindex, nofollow, noarchive">`. Messages previews do not run JavaScript ([apple-tn3156]) [Certain]. The favicon is always Kitsheet's.

#### 7.3.6 Deferred

QR codes (only practical for the no-logo link) and link expiry (the payload carries a month stamp `d` so expiry can be added later without a format change; S-10). Both are owner decisions for after launch.

### 7.4 Decoding a share link (hostile input)

Order, each step failing to a specific S-state (§2.10):
1. Length gate (> 8,192 → S-INVALID).
2. Prefix and version check (`1.` else S-NEWER if it looks like `\d+\.`, else S-INVALID).
3. `len` check (mismatch → S-CUT).
4. base64url decode; inflate with D1 into a fixed buffer. fflate 0.8.3's `unzlibSync` neither verifies the Adler-32 trailer nor throws on overflow: given a fixed `out` buffer it truncates silently [Certain, source read]. So `codec.ts` passes `out: new Uint8Array(32769)` and treats a returned length of 32,769 as overflow (S-INVALID), then computes Adler-32 over the inflated bytes (`adler32.ts`, about 10 lines) and compares it to the stream's last 4 bytes; a mismatch → S-CUT.
5. `TextDecoder('utf-8', {fatal:true})`.
6. `JSON.parse`.
7. valibot `ShareV1` schema (§7.5.2): strips unknown keys (also drops `__proto__`).
8. Field normalization (§6.5.5) and `containsContact()` returns null for `n`, `t`, every `c[i][2]` and any non-catalog `h[0]`/`b[0]`, else S-INVALID; colors `^[0-9a-f]{6}$`; font names: exact catalog match, else display-only text matching `^[\p{L}\p{N} '&.-]{1,64}$`u; `l.k` keeps only 6-hex entries that appear among the paints in `l.s`. SVG: `validateCanonical(s, 8192)` must return `s` unchanged; otherwise delete `l.s`, render the rest of the kit, and show S-LOGO ("This link's logo could not be checked, so it isn't shown."). The share route never calls DOMParser, never mounts the probe, and never creates any Trusted Types policy except `bk-worker`.
   - **8a.** Scannable-code check (§6.2.6) on the logo and each variant, drawn off-screen as the view would draw them. Any hit, error, or a run over 1 s (fail closed) → delete `l.s` before anything renders, exports or is imported through **Edit a copy**, and show S-LOGO ("Logo hidden: it contains a scannable code.").
9. Blocklist (`/blocklist.json`, static, `Cache-Control: max-age=300`): SHA-256 of the canonical JSON as decoded (before any logo is hidden), and the name's match key. Match → S-BLOCKED. If the blocklist cannot be fetched, render anyway (it is a takedown lever, not a security control).
10. Watchlist (`/watchlist.json`; large-organization names such as banks, payment processors, large email and cloud providers, government agencies and carriers) → S-WATCH interstitial (S-9) when any of these hold: the name's match key equals a watchlist key; any word in the name mixes scripts (fails UTS #39 "highly restrictive", for example Latin plus Cyrillic); or the match key is within edit distance 1 of a watchlist key of 5 or more characters [Guessing on the threshold].
11. Render with `textContent` only, inside elements with `dir="auto"` and `unicode-bidi: isolate` (Trojan Source defense, [trojan-source]). Colors only through `element.style.setProperty('--c1', '#' + hex)` and canvas/PDF APIs; never concatenated into CSS text. Fonts only through `FontFace` with internal aliases; `@font-face` URLs only from catalog slugs `^[a-z0-9-]{1,64}$`.

**Match key** (`engine/kit-model/match-key.ts`): the UTS #39 skeleton of NFKC_Casefold(name), with whitespace and punctuation removed. Watchlist and blocklist keys are computed at build time. Only the `confusables.txt` mappings whose targets are `[a-z0-9]` ship (`public/confusables.json`, §4.5), which keeps the table small. If `confusables.json` cannot be loaded, the key falls back to NFKC_Casefold with whitespace and punctuation removed, and the mixed-script check still runs. This catches homoglyph names such as "Раураl" (Cyrillic Р, а, у), "PayPaI" (capital I) and "Chаse" (Cyrillic а), which NFC alone does not.

### 7.5 Kit JSON schema v1

#### 7.5.1 `KitDocV1` (local document, IndexedDB `bk` → `draft` → `current`)

```ts
interface KitDocV1 {
  schema: 'bk.kit/1';
  savedAt: string;                 // ISO 8601, minute precision; display only, never an identifier
  algo: number;                    // ≥ 1; pins ramp/contrast/type-step algorithms
  brand: { name: string; tagline: string; nameSrc: Prov[]; taglineSrc: Prov[] };
  sources: Source[];               // ≤ 10
  colors: KitColor[];              // 1..8 to export; 0..8 while editing
  fonts: { heading: FontRef | null; body: FontRef | null };
  type: { base: number; ratio: Ratio; detected: boolean };
  logo: {
    svg: string | null;            // canonical SVG, ≤ 65,536 bytes
    raster: { blobKey: 'logoRaster'; mime: 'image/png'|'image/jpeg'|'image/webp'; w: number; h: number; bytes: number } | null;
    usage: { clearSpace: number; minPx: number; minMm: number; bg: 'auto'|'light'|'dark' };
    knockouts: string[];           // hex paints treated as cut-out
    donts: Dont[];                 // ≤ 8
    candidates: { url: string; reasons: string[]; score: number }[];  // ≤ 3, local only
  } | null;
  favicon: { blobKey: 'favicon'; mime: 'image/png' | 'image/x-icon' | 'image/svg+xml'; size: number } | null;
                                   // invariant: if mime is SVG, the stored bytes equal a validateCanonical result
  export: { pageSize: 'letter' | 'a4' };
}
type Ratio = 1.125 | 1.2 | 1.25 | 1.333 | 1.414 | 1.5 | 1.618;
type Role = 'primary'|'secondary'|'accent'|'background'|'text'|'neutral'|'other';
interface KitColor { hex: string; role: Role; name: string; locked: boolean; prov: Prov[]; guide?: { cmyk?: string; pantone?: string } }
interface FontRef { family: string; source: 'google'|'bunny'|'adobe'|'self-hosted'|'system'|'unknown';
                    sourceLabel: string; weights: number[]; italic: boolean;
                    category: 'sans-serif'|'serif'|'monospace'|'display'|'handwriting'; inCatalog: boolean; locked: boolean; prov: Prov[] }
interface Source { id: string /* 's1', 's2', … in order added; never sent anywhere */; kind: 'url'|'logo'|'image'|'guide'|'manual'|'share';
                   label: string;  /* domain or file name */ at: string /* YYYY-MM-DD */ }
interface Prov { source: string /* Source.id */; kind: 'painted'|'declared+used'|'meta'|'pixel'|'svg-paint'|'pdf-text'|'pdf-fill'|'computed'|'link'|'font-face'|'pdf-embedded'|'manual'; detail?: string; derived?: boolean }
type Dont = 'stretch'|'recolor'|'rotate'|'effects'|'busy-bg'|'outline'|'rearrange';
```

| Field | Constraint |
|---|---|
| `brand.name` | 1-60 graphemes after §6.5.5 normalization (required to export or share); `containsContact()` returns null |
| `brand.tagline` | 0-120 graphemes; `containsContact()` returns null |
| `colors[].hex` | `^#[0-9a-f]{6}$`, unique, no alpha |
| roles | at most one each of primary, secondary, accent, background, text |
| `colors[].name` | 0-32 graphemes; `containsContact()` returns null |
| `FontRef.family` | 1-64 characters; exact catalog name, or display text matching `^[\p{L}\p{N} '&.-]{1,64}$`u and `containsContact()` returns null |
| `FontRef.weights` | 1-4 values in {100…900}, step 100, sorted, unique |
| `type.base` | 12-24, step 0.5 (default 16) |
| `logo.usage.clearSpace` | 0.1-2.0 × logo height. Default 0.25 for wide logos (aspect ≥ 2:1), 0.5 otherwise [Guessing; starter rule] |
| `logo.usage.minPx` | 16-512 px wide. Default 24 for compact marks, 80 for wide logos [Guessing; starter rule] |
| `logo.usage.minMm` | 5-100 mm wide. Default 6 for compact marks, 20 for wide logos [Guessing; starter rule] |
| `sources[].label` for URLs | scheme + host + path only; never a query string |

**Derived, never stored, never shared:** ramps, contrast matrix and AA/AAA flags, passing pairs, type steps, fallback stacks, logo variants.

#### 7.5.2 `ShareV1` (wire JSON inside `#1.…`)

```jsonc
{
  "n": "Rowlett Roofing & Restoration",                 // required, 1-60 graphemes
  "t": "Storm repair and roof replacement",             // optional, ≤ 120
  "c": [["c8102e",0,"Signal Red"],["1b2a41",1],["fafaf7",3],["14171a",4]], // 1-8; [hex, roleIndex, name?]
  "h": ["Playfair Display",[700,900],0,0],              // optional; [family, weights, italic 0|1, sourceIndex]
  "b": ["Inter",[400,600],0,0],                         // optional
  "y": [16,1.25],                                       // optional; omitted when default
  "a": 1,                                               // optional; omitted when 1
  "l": {"s":"<svg …>","c":0.5,"p":96,"m":25,"g":0,"k":["ffffff"]}, // optional; "s" optional; clearSpace, minPx, minMm, bg index; "k" optional knockouts
  "d": 202611                                           // YYYYMM the link was made; optional in the decoder, always written
}
```

Key order inside `l`: `s,c,p,m,g,k`. `k` (additive, allowed under §7.5.3 rule 2; D1 is not frozen until launch) lists knockout paints as 6-hex strings and is written only when the user's knockout set differs from the §6.2.3 default rule; the decoder keeps only entries that appear among the paints in `s`. The share view's one-color variant uses the primary color.

Role index order (append-only): 0 primary, 1 secondary, 2 accent, 3 background, 4 text, 5 neutral, 6 other; unknown index → other. Source index order (append-only): 0 google, 1 bunny, 2 adobe, 3 self-hosted, 4 system, 5 unknown. The sample name and tagline above are fictional.

**Not in links, by design:** source URL or domain (impersonation aid, privacy), raster logos and favicons (D22), provenance, locks, don'ts, candidates, page size.

#### 7.5.3 Versioning rules

1. The format major (`1.`) is permanent. Decoders keep every past major for as long as Kitsheet runs, so links work while Kitsheet is online at `SHARE_ORIGIN` (Terms §9.2 clause 9; §0.3 #1).
2. Within a major only additive changes: new optional keys; enum values appended; no change to meanings, units or caps that would reject an old link.
3. `a` pins derived output. A new ramp, contrast or type-step algorithm bumps `a` for new links; the renderer keeps the old algorithm for old links.
4. Encoders write canonical JSON: fixed key order `n,t,c,h,b,y,a,l,d`; defaults omitted; integers where possible. Canonical output keeps links short, blocklist hashes stable and tests deterministic.
5. Golden tests: 10-20 fixture links per major (max lengths, RTL text, gradient SVG, every enum) decode in CI on every release. Fixture links with transforms, gradients and clip paths are generated in Chromium, Firefox and WebKit, and each must decode with its logo shown in all three engines and in Node.
6. Logo safety checks (the scannable-code check and the canonical grammar caps) may be tightened within a major version. They can only hide a logo (S-LOGO), never reject a link. Golden tests include real logos that must stay visible.

### 7.6 Local persistence and clearing data

#### 7.6.1 What is stored

Nothing is written to persistent storage unless the user asks for it.

| Store | Key | Content | Written when |
|---|---|---|---|
| sessionStorage | `bk.session` | `KitDocV1` without Blobs | Always while editing, so a reload in the same tab keeps work; gone when the tab closes |
| IndexedDB `bk` v1, store `draft` | `current` | `KitDocV1` | Only after the user ticks **Keep this draft on this device** (unchecked by default). Unticking it runs Clear saved draft |
| IndexedDB `bk`, store `blobs` | `logoRaster`, `favicon` | Blobs | Same as the draft |
| localStorage | `bk.theme`, `bk.pageSize` | `light`/`dark`/`system`; `letter`/`a4` | Only when the user changes them, and never after **Don't remember my settings** (which deletes both; preferences then stay in memory only) |

Legal basis for each (§8.2): the kept draft is a record of information the user put into the site and asked to keep (UK PECR Sch. A1 para 4(2)(e)(ii); EU Art. 5(3) strictly necessary [Likely]); the two preferences fall under UK PECR Sch. A1 para 6 (clear information plus a free, simple way to object, which **Don't remember my settings** provides) [Certain on the text ([pecr-a1])].

Writes are debounced 500 ms and wrapped in try/catch; storage may be missing (private modes, quota) → E-STORAGE when the user has asked to keep a draft. No identifiers, no cookies, no sync, no service worker, no `navigator.storage.persist()` (Firefox would prompt) (PRIV-12, PRIV-13). Multiple tabs with a kept draft: last write wins; a `BroadcastChannel('bk')` tells other tabs to reload the draft.

#### 7.6.2 Never stored

Uploaded PDFs; image-to-palette source images; fetched HTML, CSS and KITX frames; Turnstile tokens; image tickets; URL query strings; the scannable-code flag; share links the user opened (opening a link never writes storage; **Edit a copy** imports it after asking, if a draft exists).

#### 7.6.3 Clear my data

Button in the margin column, and a link on every static page to `/clear`. Steps (the only code path that deletes data): revoke all `blob:` URLs; close the database; `indexedDB.deleteDatabase('bk')`; remove `bk.*` from localStorage; `sessionStorage.clear()`; reset in-memory state. Confirmation copy: "Cleared. Links you already shared still work, because the kit lives inside the link."

`/clear` is a static page with a **Clear now** button that runs the same steps in a small same-origin script. It is served without `Clear-Site-Data`: static hosting cannot check `Sec-Fetch-Site`, so a header on a plain GET would let any other site wipe a visitor's draft by linking to it. The page also says, for visitors without JavaScript: "Or clear site data for {host} in your browser settings."

Microcopy next to the Keep checkbox: "Kept drafts are saved in this browser only. Safari may clear them after 7 days without a visit." (WebKit deletes script-writable storage after 7 days of Safari use without interaction on the site ([webkit-itp]) [Certain].)

### 7.7 Optional later: design tokens export (not in v1)

A "Copy tokens" button could emit CSS custom properties or a DTCG JSON file from the same kit model in about 2 KB of code, with no new dependency, server call or data flow. It was not selected for v1 (D23). Adding it later needs no legal re-review.

---

## 8. Compliance-by-design matrix

### 8.1 The two answers the owner asked for

> **TCPA: compliant by construction.** Kitsheet never asks for, accepts or stores a phone number, and never places calls or sends texts. There is no phone field, no "text me the link", no `sms:` or `tel:` link, and no SMS or voice provider. Names, taglines, color names and custom font names reject phone numbers, email addresses and web addresses in common formats (§6.5.5), so share pages don't display them as text. Someone could still draw a number into an SVG logo; the share-view banner, blocklist and report link (§10.8) cover that case. None of this is a TCPA question: the TCPA (47 U.S.C. §227 ([usc-227])) and Texas's telephone-solicitation law (Bus. & Com. Code ch. 302 ([tx-bc-302])) as extended to texts by SB 140 (effective 1 Sep 2025) regulate calls and texts the operator makes, and Kitsheet makes none [Likely: statute text for §227; secondary source for the SB 140 change ([tx-sb140])]. **Any feature that would collect a phone number requires a new legal review before it is built** (§17.3).

> **Children's email addresses: never captured.** Kitsheet has no email field anywhere, so it cannot collect an email from anyone, child or adult. The only email the operator can ever receive is one a person chooses to send from their own mail app to the public contact address (`mailto:`), a free Gmail mailbox used only for Kitsheet, with Gmail's "Create contacts for auto-complete" set to "I'll add contacts myself", no forwarding, no POP or IMAP, no filters that forward mail and no connected third-party apps (§9.0). If a message shows the sender is under 13, the operator does not reply. He acts on any report it contains. The same day he deletes the message, deletes any reply from Sent, empties Trash, and removes the address from Contacts and from Contacts › Other contacts. This is stricter than COPPA's one-time-response exception (16 CFR 312.5(c)(3) ([ecfr-coppa])), which permits one reply only if the address is deleted promptly afterwards [Likely]. The report link adds: "Under 13? Ask a parent or guardian to send this report." The tool is general-audience, not child-directed, and has no age gate (an age gate would itself collect age data) [Certain that COPPA does not require general-audience sites to screen ages ([ftc-coppa-faq])].

### 8.2 Matrix

| Law / regime | Applies? | Why | How the design satisfies it | Residual risk |
|---|---|---|---|---|
| **TCPA** (47 U.S.C. §227 ([usc-227])) and **Texas SB 140** (Bus. & Com. Code ch. 302 ([tx-bc-302])/304) | No | No calls, no texts, no phone numbers [Certain on the design; Likely on the legal conclusion]. Context only: the FCC one-to-one consent rule was vacated (11th Cir., 24 Jan 2025 ([tcpa-imc])); district courts are not bound by FCC TCPA interpretations (*McLaughlin*, U.S. 20 Jun 2025 ([tcpa-mclaughlin])); the "revoke-all" rule is delayed to 31 Jan 2027 ([tcpa-revoke]) [Likely]. | §8.1. No phone fields; no operator-sent messages; sharing only via user-initiated copy, Web Share or the user's own mail app. | Low. Out of scope but flagged: if the owner's Squarespace portfolio has forms collecting phone numbers and he texts leads, TCPA and SB 140 apply **there**. |
| **CAN-SPAM** (15 U.S.C. 7701 et seq.) | No | No commercial email sent [Certain]. | No email collection; no server-sent email; replies to inbound mail are not marketing and never followed by promotion. | None while no email is sent. |
| **CASL** (Canada) | No | No commercial electronic messages sent; replies to inquiries are exempt from s.6 (SOR/2013-221 s.3(b)) [Likely]. | Same as CAN-SPAM. | None. |
| **COPPA** (16 CFR 312) and 2025 amendments | No (conditional) | Covers child-directed operators or those with actual knowledge (§312.3) ([ecfr-coppa]). IP addresses are COPPA personal information (§312.2) [Certain], but Kitsheet is general-audience. Amendments effective 23 Jun 2025, compliance 22 Apr 2026 ([coppa-2025]) [Certain]; they do not reach general-audience, no-knowledge operators [Likely]. | No PII fields; no age gate; nothing child-directed (no mascots, games, school campaigns); children's section in the policy; no reply and same-day deletion for inbound mail from children, in a dedicated mailbox with auto-contacts off (§8.1); internal-operations sentence about IPs (§9.1). | Low. Re-review if marketing ever targets students or classrooms. |
| **UK Children's Code** (AADC) | Likely not | Applies only if UK GDPR applies **and** the service is "likely to be accessed" by under-18s, meaning more probable than not ([ico-childrens-code]) [Certain on the test]. A business-branding tool is not [Likely]. | Write and keep the ICO-style assessment in `docs/compliance/childrens-code.md`. Defaults already meet the Code: no profiling, no geolocation, no nudges, no sharing, plain-language policy. | Low. Revisit on classroom adoption. |
| **GDPR / UK GDPR** | Probably not | No EU/UK establishment. Art. 3(2) needs targeting (language, currency, EU marketing) or behavioral monitoring; mere accessibility is not enough ([edpb-3-2018]) [Certain on the test; Likely on the conclusion]. | English only; no EU currencies or marketing; no analytics or profiling; data minimized to transient IPs held by Cloudflare; policy written to the Art. 13 standard anyway ([gdpr]); Cloudflare's DPA applies automatically to free plans ([cf-terms], [cf-dpa]) [Certain]. No Art. 27 representative needed while Art. 3(2) is not triggered. | Low. A paid EU/UK representative would break $0; §17.3 lists the triggers. |
| **ePrivacy Art. 5(3) / UK PECR reg. 6 with Schedule A1** | Yes | Applies to storage/access on any device in the EU/UK, personal data or not ([edpb-2-2023]) [Certain]. UK PECR Schedule A1, inserted by the Data (Use and Access) Act 2025 and in force 5 Feb 2026, lists the exceptions ([pecr-a1]) [Certain on text]. | Kitsheet sets no cookies; no beacon, no pre-clearance, fonts self-hosted (ICO names external font libraries as a disclosure-and-objection case ([ico-exceptions])) [Certain]. Storage and access items: (1) the draft, written only when the user ticks **Keep this draft** (UK Sch. A1 para 4(2)(e)(ii), a record of information put into the site; EU strictly necessary [Likely]); (2) theme and page size, written only when changed, with **Don't remember my settings** as the simple objection (UK Sch. A1 para 6) [Certain on text]; (3) Turnstile's access to device signals when a URL is read; (4) any challenge cookies Cloudflare's challenge platform sets on `challenges.cloudflare.com`: Cloudflare lists `cf_chl_rc_i`, `cf_chl_rc_ni` and `cf_chl_rc_m` "for internal use which allows Cloudflare to identify production issues on clients" ([cf-cookies]) [Likely that the widget sets them; §15.6 verifies]. Items 3 and 4 rely on the strictly-necessary exemption for the read the user asked for: EU Art. 5(3) second sentence; UK PECR Sch. A1 para 4(1), with 4(2)(c) (fraud) and 4(2)(d) (technical faults) as examples [Likely]. Weak point: Cloudflare also reuses the signals as its own controller to improve Turnstile ([ts-privacy] §4(ii)), and WP29 Opinion 04/2012 §2.4 exempts a multi-purpose access only if every purpose is exempt [Likely low risk]. Shown before the click (§2.3) and in the policy (§9.1 items 6 and 13); **no cookie banner**. | Low-medium. Added to §9.9. |
| **CCPA/CPRA** | No | Business thresholds: revenue over $26,625,000, or 100,000+ consumers' PI bought/sold/shared, or 50% revenue from selling/sharing ([cppa-threshold]) [Certain]. None met. | Policy states "we don't sell or share personal information". | None. |
| **CalOPPA** (and **Delaware DOPPA**) | Probably not | Needs a commercial site collecting PII (name, email, phone…) from consumers who acquire goods by purchase or lease ([caloppa-22577]); IPs are not in the list [Certain on text; Likely on conclusion]. | Policy includes CalOPPA/DOPPA items anyway: categories, third parties, change process, effective date, Do Not Track response, third-party tracking statement ([caloppa-22575], [de-doppa]). | None. |
| **Texas TDPSA** (Bus. & Com. Code ch. 541) | No | Exempts SBA-defined small businesses except for selling sensitive data without consent (§541.107) ([tx-tdpsa]) [Certain]. | Nothing is sold. | None. |
| **Texas SCOPE Act** (HB 18, ch. 509) | No | Applies only to services with social interaction **and** profiles **and** user posts viewable by others ([tx-scope]) [Certain]. | No accounts, profiles, galleries or comments; share links are self-contained URLs. | None unless social features are added. |
| **Other US state privacy laws** (24 states as of mid-2026 ([state-laws-2026])) | No | Lowest thresholds are 25,000 consumers (Montana; Alabama from 1 May 2027) or 35,000 (Delaware, New Hampshire, Rhode Island, Maryland; Vermont from 1 Jan 2028); Texas and Nebraska use the SBA exemption [Likely; secondary sources]. | No stored personal data. | Open question whether IPs held only in short-lived CDN logs count at very high traffic [Guessing]. |
| **Brazil LGPD / ECA Digital**, **Canada PIPEDA**, **Australia Privacy Act**, **India DPDP**, others | No or minimal | LGPD: no Brazil targeting, only IP logs [Guessing] ([lgpd]). ECA Digital (in force 17 Mar 2026) covers services directed at or likely accessed by minors in Brazil [Likely] ([eca-digital]). PIPEDA needs commercial activity; a non-monetized tool is not primarily commercial [Likely] ([pipeda]). Australia: small-business exemption (≤ AUD 3m turnover) still in force [Likely] ([au-privacy]). India DPDP: core duties from about 13 May 2027 [Guessing] ([india-dpdp]). Switzerland, Japan, China: not triggered [Guessing]. | One honest notice covers transparency everywhere; minimal data; no child appeal. | Low. Re-check India before May 2027. |
| **ADA Title III** (US) | Low exposure | Unsettled whether web-only services are public accommodations; the Fifth Circuit has no controlling website ruling (*Magee*, 2016, read "sales establishment" as physical) [Likely]. DOJ's Title II rule (not applicable) sets WCAG 2.1 AA as the benchmark ([doj-title2]). | WCAG 2.2 AA target (§13), accessibility statement, contact for barriers. | Low. |
| **European Accessibility Act** | No | Microenterprises providing services are exempt (Art. 4(5)) ([eaa-art4]) [Certain]; not a listed service [Likely]. | WCAG 2.2 AA anyway. | None. |
| **Copyright** | Applies to inputs; low risk | Colors, names, slogans and "typeface as typeface" are not copyrightable (37 CFR 202.1(a), (e)) ([cfr-202-1]) [Certain]; facts are free (*Feist*). CSS code and complex logos can be protected. | Extract facts only; never republish CSS; never store fetched content; logos appear only where the user put them; source domains as plain text; real-site snapshots never committed (§15.3). | Low. Users' own use of third-party logos is theirs (Terms). |
| **Trademark / false endorsement** | Applies | Nominative fair use (*New Kids*; Fifth Circuit *Pebble Beach*: disclaimers help but do not cure confusing presentation) [Certain on doctrine]; contributory liability requires knowledge (*Inwood*, *Tiffany v. eBay*) [Certain]. EU: *Louboutin v Amazon* on operator "use" [Likely]. | No "official"/"approved" wording; starter-rules label; provenance footer and colophon; tool branding visually separate from kit content; anti-impersonation share chrome (§10.8); share checkbox; blocklist acting on specific knowledge; demos use only the owner's or fictional brands; marketing copy is about the user's own brand (*Sony* vs *Grokster* line). | Medium-low: share links are the main exposure. |
| **Font licensing** | Applies | OFL allows PDF embedding in full or subset (OFL-FAQ 1.12) and hosting with the full package (1.8); subsetting a webfont is modification and drops Reserved Font Name use (2.6) ([ofl-faq]) [Certain]. Adobe Fonts are licensed only to subscribers and forbid self-hosting or enabling others' use ([adobe-fonts-terms]) [Certain]. | Google states its library fonts may be used commercially "on any surface" ([gf-faq]) [Certain]. Embed only open-licensed fonts from our own mirror; never fetch or embed a scanned site's font files; detect commercial fonts by host only; substitute and label; honor `fsType`; the mirror branches on license (§7.1.3): RFN and UFL families shipped byte for byte with no menu file, Apache-2.0 subsets carry `CHANGES.txt` and any upstream `NOTICE`; license files beside fonts; `/credits`. | Low [Likely]. RFN handling and UFL naming worth a one-time check (§9.9). |
| **DMCA §512 / Section 230** | Not needed / marginal | §512(c) protects storage at a user's direction; nothing is stored, so there is nothing to protect and no reason to pay the $6 designated-agent fee ([dmca-512], [dmca-fee]) [Certain on statute; Likely on fee]. §230 covers non-IP claims about user content but not IP claims (§230(e)(2)) and not outside the US ([s230]) [Certain]. | Fragment-only sharing; voluntary copyright/trademark contact; blocklist takedown within 2 business days. | Low. Register an agent only if storage is ever added. |
| **EU DSA / UK Online Safety Act** | Probably not | DSA covers intermediary services (conduit, caching, hosting) with a substantial EU connection; nothing is hosted ([dsa]) [Guessing]. UK OSA "user-to-user" (s.3) arguably fits share links, but "links with the UK" (s.4) are likely unmet ([uk-osa-3], [uk-osa-4]) [Guessing]. | Fragment-only sharing; text and payload caps; no raster in links; plain-text rendering; blocklist; report route; a written scope note and illegal-content risk note in `docs/compliance/uk-osa-note.md`. | Low; paperwork only if ever in scope. |
| **FTC Act §5 / Texas DTPA** (truthful claims) | Yes | Deceptive privacy or capability claims are actionable [Certain]. This is the largest live exposure. | Every claim is tested (§15.6 network-allowlist, cookie and storage tests); every sentence in §9.7 and the §9.1 short version maps to a named test or spec section in `docs/compliance/claims-map.md`, signed off by the owner at M10; wording avoids "stores nothing", "WCAG compliant", "legally safe", "no tracking"; contrast and font-source disclaimers. | Low while the tests run on every deploy. |
| **CFAA / Texas Penal Code ch. 33 and CPRC ch. 143** (fetching) | Applies to the fetcher | Public pages likely have no gate to lift (*Van Buren* ([vanburen]); its footnote 8 expressly left open whether limits in contracts or policies count as gates) [Likely]. *hiQ Labs v. LinkedIn*, 31 F.4th 1180 (9th Cir. 2022) ([hiq]) is persuasive only: a Ninth Circuit ruling does not bind Texas courts (Fifth Circuit). Texas Penal Code §33.02 turns on "effective consent", and consent "induced by deception" is not effective ([tx-pen-33], [tx-cprc-143]) [Certain on text]; actual knowledge via cease-and-desist made browsewrap terms bind in N.D. Tex. (*Southwest v. Kiwi* ([sw-kiwi])) [Certain]. The real mitigations are the ones in the next column. | Honest UA and `/bot` page; robots.txt honored; stop at every gate; per-analysis caps; do-not-fetch list honored within 2 business days of any request; user-triggered only. | Low. |
| **Texas assumed-name law** (Bus. & Com. Code §71.051) | Probably not | A filing may be needed if someone regularly conducts business under an assumed name ([tx-71]) [Certain on statute]. | The tool is presented as "a free tool made and run by Justin James", and the Terms use his legal name. | Low; on the attorney list. |

### 8.3 Internal compliance notes (in the repo, `docs/compliance/`, not linked from the site)

1. `data-flow.md`: every data flow and the browser's allowed hosts (PRIV-01), kept in sync with the network test.
2. `art3-note.md`: why GDPR/UK GDPR Art. 3(2) is not triggered.
3. `childrens-code.md`: the "likely to be accessed" assessment with reasons.
4. `uk-osa-note.md`: scope note and illegal-content risk note.
5. `logging.md`: Cloudflare log settings and retention, with dashboard screenshots.
6. `change-triggers.md`: the §17.3 table, run before any new feature.
7. `claims-map.md`: every sentence in §9.7 and the §9.1 short version, each mapped to the named §15.6 test or spec section that proves it; the owner signs it off at M10.
8. `data-flow.md` also holds the Turnstile allowlist: every cookie name, domain and maximum lifetime, and every storage key, observed in the challenge frame after a URL read (§15.6), matched to privacy item 6.

---

## 9. Legal pages to publish

**This is not legal advice.** The outlines below come from research. §9.9 lists items worth a one-time review by a Texas-licensed attorney. That review is optional.

### 9.0 Operator identity and contact

- Operator line on every legal page: "Kitsheet is a free tool made and run by Justin James, an individual in Rowlett, Texas, USA." Use "I" and "me", not "we" or a company name (Texas assumed-name question, C13 context) [Likely].
- **Contact: `CONTACT_EMAIL`, a free Gmail address used only for Kitsheet** (for example `kitsheet.<something>@gmail.com`; §17.2 Q4). $0. Not an alias on the owner's Google Workspace: an alias has no settings of its own, delivers into the wedding business's inbox, and replying from it would save the sender in that mailbox's Google Contacts › "Other contacts" by default, where any CRM or inbox tool reading the business mailbox could import them [Likely]. Settings on the Kitsheet mailbox, set before launch and checked monthly (§11.5):
  - Gmail › Settings › General › Create contacts for auto-complete: "I'll add contacts myself".
  - No forwarding and no POP or IMAP.
  - No third-party apps with Gmail access (Google Account › Security › third-party connections).
  - No filters that forward mail. Labels by subject ("Report:", "Privacy:", "Do not fetch:", "Accessibility:") are fine.
- **No street address, no phone number.** No US law found requires a street address for a free, non-transactional site; California Civ. Code §1789.3 applies only to an "electronic commercial service" ([ca-civ-1789]) [Certain]; GDPR Art. 13 needs identity and contact, which name plus email satisfies [Likely]. German §5 DDG and UK E-Commerce reg. 6 reach to a US individual's free English-only tool is unclear ([ddg-5], [uk-ecomm-reg6]) [Likely not required; on the attorney list].
- Inbound mail policy (published in the Privacy Policy): adult mail is used only to answer or act on the message; never added to any list (auto-contacts is off); deleted, and Trash emptied, within 30 days after the matter is closed. Children's messages get no reply and are deleted the same day (§8.1).

### 9.1 Privacy Policy (`/privacy`)

Effective date at top. Plain language a 13-year-old can follow. Sections, in order:
1. **Who runs this**: operator line and contact.
2. **Short version** (each sentence maps to a test in `docs/compliance/claims-map.md`): no accounts; no forms; Kitsheet sets no cookies; no ads or analytics; files you add never leave your browser; I don't store your uploads, inputs or kits.
3. **What happens to each input**: uploads (processed in your browser, never sent); typed values (stay in your browser tab, and in this browser only if you tick **Keep this draft**); website address (my server fetches that one public page, its robots.txt, its stylesheets and a few images, and hands them to your browser, which finds the brand details; the server keeps nothing and does not log the address); kept draft (only if you tick **Keep this draft on this device**; this device only until you untick it or clear it; Safari may clear it after 7 days); share links (the kit is packed into the part after `#`, which browsers don't send to servers when a link is opened normally; anyone with the link can see it; I can't see or delete it; some email security tools rewrite links so the kit part reaches the server as part of the address; I don't log it).
4. **Technical data providers process**: IP address, user agent, time, path, TLS details, processed by Cloudflare to deliver and protect the site, and by Cloudflare Turnstile when you read a website (client IP, TLS fingerprint, user agent, sitekey and origin). Cloudflare acts as my processor for delivery and the bot check, and as its own controller when it uses Turnstile signals to improve bot detection ([ts-privacy]) [Certain]. Retention: the actual Cloudflare setting, recorded in `docs/compliance/logging.md` (as of research: Pages Functions keep real-time logs only; from 1 Dec 2026 the Observability free tier retains 7 days ([cf-observability]) [Certain on the docs; verify in the dashboard]). Internal-operations sentence: "Technical data such as IP addresses is used only to deliver the site and protect it from abuse. It is never used to contact anyone, build profiles or target ads."
5. **Third parties your browser contacts**: only this site, and `challenges.cloudflare.com` for the bot check only when you read a website. Fonts come from this site. Logos from sites you read are fetched by my server, so your browser never contacts those sites.
6. **Cookies and similar technology**: Kitsheet sets no cookies. When you press Read site, Cloudflare's bot check runs in a frame from challenges.cloudflare.com and may set short-lived cookies there ({names and lifetimes from verification, §15.6; for example `cf_chl_rc_i`, `cf_chl_rc_ni`, `cf_chl_rc_m`}). Cloudflare says they help it find faults in the check. Browser storage holds your draft only if you tick **Keep this draft**, which you asked for; and your theme and page size only if you change them, so the site looks the way you chose. **Don't remember my settings** turns that off and deletes them.
7. **Analytics**: none in your browser. I see daily counts of requests to the website-reading service and errors in Cloudflare's dashboard.
8. **Legal bases (EEA/UK)**: legitimate interests in delivering and securing the tool and performing the fetch you request (GDPR Art. 6(1)(f)).
9. **Where data is processed**: USA and Cloudflare's global network, and Google (email you send me); Cloudflare's DPA with EU SCCs, UK Addendum and Data Privacy Framework ([cf-dpa]).
10. **Your rights**: access, deletion, objection and the rest; complaint to your data protection authority. Because nothing is stored against you, there is usually nothing to return beyond short-lived provider logs.
11. **Sale and sharing**: "I don't sell or share personal information and don't use targeted advertising."
12. **Do Not Track and Global Privacy Control**: "I don't track you across sites, so these signals don't change anything here."
13. **Third-party tracking**: "When you press Read site, Cloudflare, which runs the bot check, receives your IP address, TLS fingerprint, browser user agent and which site's check ran. It uses them to run the check for me, and as its own controller to improve Turnstile's bot detection (Turnstile Privacy Addendum, [ts-privacy]). Cloudflare says it does not use them to identify, profile or target people. No ad network, analytics company or other third party collects data through Kitsheet." (Item 12 stays, because "I don't track you" is about the operator.)
14. **Children**: "Kitsheet is for business branding and isn't directed to children under 13. It never asks anyone for personal information. Like any website, it receives your IP address to deliver pages and block abuse; Kitsheet itself keeps none of it, and Cloudflare's short-lived logs are described above. If a child under 13 emails me, I won't reply. I'll act on any report and delete the message the same day. Parents with questions can email {contact}."
15. **Email you send me**: the inbound mail policy (§9.0), plus: "Email you send to {contact} is delivered and stored by Google's Gmail service in the USA and wherever Google keeps data, under Google's own privacy policy, until I delete it." (If the owner keeps a Workspace alias instead, §17.2 Q4: "…by Google Workspace, which acts as my processor under Google's data processing terms.")
16. **Security**: HTTPS only; no database of user data; open source.
17. **Changes**: effective date; material changes noted on the page; full history in the public repo.
18. **Clear my data** button and **source code** link so anyone can verify these claims.

### 9.2 Terms of Use (`/terms`)

Assent points, each a notice tied to a specific action, at body size (16 px) in `--ink` with an underlined link (*Berman*: the notice must state the legal significance of a specific action, and small low-contrast links failed ([berman])) [Certain on Berman]:
- under **Read site**: "By pressing Read site, you agree to the Terms and Acceptable Use Policy." (§2.3);
- under the file buttons: "By adding a file, you agree to the Terms." (§2.3);
- in the export panel: "By downloading or copying a link, you agree to the Terms." (§2.9);
- the share checkbox label ends "…and I agree to the Terms and Acceptable Use Policy." (§2.9).
Still no age gate and nothing stored. "Last updated" date; prior versions in git history. Clauses:
1. **Who runs this; acceptance**: operator line. Not a company or LLC. Pressing Read site, adding a file, downloading a PDF or PNG, copying a share link, or ticking the share checkbox counts as accepting these Terms.
2. **What Kitsheet does and doesn't do**: automatic extraction can be wrong; contrast checks use the WCAG formula and are not an accessibility audit; font-source labels are not license advice; substitutes are marked.
3. **AS IS; NO WARRANTY** (conspicuous: bold heading, separate block): provided "as is" and "as available", no warranties of accuracy, merchantability, fitness or non-infringement, to the fullest extent the law allows (Texas fair-notice and conspicuousness rules, *Dresser* ([dresser])) [Certain on Dresser].
4. **LIMITATION OF LIABILITY** (conspicuous): no indirect, incidental, special, consequential or punitive damages; total liability capped at US$50; carve-out: "Nothing in these terms limits liability that can't be limited by law, such as for fraud, gross negligence, intentional misconduct, or death or personal injury caused by negligence." [Likely]
5. **Your inputs and representations**: you keep your rights; uploads stay in your browser; you confirm you may use what you submit for your purpose.
6. **Acceptable use**: incorporates `/acceptable-use`.
7. **Third-party brands and fonts**: trademarks belong to their owners; making a kit gives no rights; only open-licensed fonts are embedded; no font license is granted.
8. **Output ownership**: I claim no ownership of your kits; you may use your PDFs and PNGs for any lawful purpose, including commercial; the templates are MIT-licensed code and you may use them as they appear in your exports.
9. **Share links**: they hold your kit in the link; not stored on my servers; anyone with the link can see it; links on the blocklist stop rendering; a logo with a scannable code is hidden; links work while Kitsheet is online at {SHARE_ORIGIN host}.
10. **Indemnity (narrow)**: you cover claims arising from your content or your breach; you never indemnify my own negligence.
11. **Changes and ending**: I may change, pause or shut down Kitsheet, or block access, at any time; changes apply from the posted date and are not retroactive.
12. **Governing law and venue**: Texas law without conflict-of-laws rules; state courts in the owner's county (Dallas or Rockwall, owner to confirm) or the U.S. District Court for the Northern District of Texas, Dallas Division. Consumer carve-out: "If you're a consumer in the EU, UK or another country whose law gives you rights or a home court that contracts can't remove, those rights still apply." (Brussels Ia Art. 17-18 ([brussels-1a]); Rome I Art. 6(2) ([rome-1])) [Certain on the regulations; Likely on the effect]
13. **No arbitration, no class waiver.**
14. **Age**: "You must be 13 or older to use Kitsheet. If you're under 18, or under the age of majority where you live, use it only with a parent's or guardian's permission. Kitsheet isn't directed to children under 13." No age gate (C13).
15. **No contact or marketing**: "I never call, text or email you, and Kitsheet doesn't collect contact details."
16. **Open source**: "The source code is MIT-licensed. The hosted tool at {host} is governed by these Terms. Copies hosted elsewhere aren't run by me." (O-2)
17. **Contact**; severability; no waiver; entire agreement; assignment.

### 9.3 Acceptable Use Policy (`/acceptable-use`)

Short, plain list. Don't:
- present a kit for a brand you don't own or represent as that brand's official material (kits for brands you don't represent are allowed for private reference, such as studying a competitor, and must not be published as theirs [Guessing: owner to confirm this allowance]);
- use Kitsheet or its links for phishing, fraud, scams, impersonation, harassment, or unlawful content;
- put logins, payment requests, contact details, phone numbers, web addresses or QR and other scannable codes in a kit (including drawn into a logo);
- read sites that ask not to be read, get around a site's access controls, target private networks, or read sites in bulk or by script;
- get around the bot check, rate limits or the blocklist;
- use output in a way that infringes trademarks or copyrights.

Enforcement: blocklist, refusal of service by hostname or IP range, and reports to the relevant platform. Reporting: link to `/report`.

### 9.4 Credits and licenses (`/credits`)

Generated at build from the license collector (`gen-notices.mjs` writing `THIRD_PARTY_NOTICES.txt`): every npm dependency shipped in a bundle with its license text (MIT, BSD and Apache require notices to travel with copies; minifiers strip header comments) (O-3). UI fonts (Archivo, JetBrains Mono) with copyright and OFL 1.1. A paragraph on brand fonts: "Fonts shown in kits come from the Google Fonts open-source library (SIL Open Font License 1.1, Apache 2.0, Ubuntu Font Licence). Each family's license is in `/fonts/{version}/{slug}/LICENSE.txt`." The repo license (MIT) and NOTICE (name, logo and photography reserved).

### 9.5 Accessibility statement (`/accessibility`)

Target WCAG 2.2 Level AA ([wcag22]); date last tested and the method, listing exactly which browser, assistive-technology and device combinations were tested (from the §15.8 record; nothing is listed that was not run); known limitations (generated PDFs are tagged but not validated as PDF/UA; the PNG board is an image, and the PDF, review screen and share link carry the same information as text; the Cloudflare bot check is a third-party component); contact for barriers with a commitment to reply.

### 9.6 Fetcher page (`/bot`) and report page (`/report`)

`/bot` (F-7): what the reader fetches (one page, its stylesheets, its manifest and up to 5 images); when (only when a person pastes that address and passes the bot check); caps (§5.2.3); that nothing fetched is stored or logged; how to block it:

```
User-agent: Kitsheet
Disallow: /
```

how to get on the do-not-fetch list (email, honored within 2 business days, no argument needed); that requests come from Cloudflare's network and carry a `CF-Worker` header naming the owning zone ([cf-headers]) [Certain].

`/report`: what to report (impersonation, phishing, copyright or trademark); how (the `mailto:` link with subject "Report: Kitsheet share link" and the full link in the body); what happens (reviewed and, when warranted, added to the blocklist within 2 business days; only the payload hash is committed, never the payload); "Under 13? Ask a parent or guardian to send this report."; the inbound mail policy.

### 9.7 In-tool notices (one line each)

| Place | Text |
|---|---|
| URL field | "Kitsheet's server fetches this public page, its stylesheets and a few images, hands them to your browser to read, and keeps nothing." + "By pressing Read site, you agree to the Terms and Acceptable Use Policy." |
| Uploaders | "Files stay on this device. Nothing is uploaded." + "By adding a file, you agree to the Terms." |
| Draft | **Keep this draft on this device** (unchecked) + "Kept drafts are saved in this browser only. Safari may clear them after 7 days without a visit." + **Clear saved draft** + **Don't remember my settings** |
| Bot check | Always visible under **Read site** from page load: "Read site first runs Cloudflare's bot check, which sends your IP address and browser details to Cloudflare and may set a short-lived cookie. Uploads don't use it. [What's sent]". Beside the widget slot after the click: "Quick bot check by Cloudflare. [What's sent]" |
| Export | "By downloading or copying a link, you agree to the Terms." |
| Share | §2.9 |
| Share banner | §10.8 |
| Footer (optional) | "No ads or analytics. Uploads stay on this device." Shown only while the §15.6 network and cookie tests pass. |

Launch gate: every sentence in this table and in the §9.1 short version maps to a named §15.6 test or spec section in `docs/compliance/claims-map.md`, and the owner signs off on that mapping at M10.

### 9.8 Repo license

MIT `LICENSE` for code. `NOTICE.md`: "The MIT license covers the code. It does not grant rights to the Kitsheet name or logo, or to photographs in `/assets/brand/`, which are © Justin James, all rights reserved." Any photography of identifiable people in the UI needs a signed model release covering promotional use (O-6); v1 uses none.

### 9.9 Items worth a one-time attorney review (optional)

A one-time review by a Texas-licensed attorney (with an EU/UK consult if convenient) would be most useful on:
1. Final Terms text: conspicuousness under *Dresser*, the $50 cap, the narrow indemnity, consumer carve-outs, the age clause, and the venue county.
2. Operating under a product name without a Texas assumed-name certificate (§71.051); whether an LLC is worth its cost given personal liability (outside the locked decisions; discussion only).
3. Share links: Safe Browsing exposure of wethejamesphoto.com and whether to move `/k/` to a separate hostname; UK OSA user-to-user status.
4. Fetcher policy: robots.txt approach, do-not-fetch handling, Texas ch. 33/143 exposure.
5. Contact disclosures: whether German §5 DDG / §18 MStV or UK reg. 6 require a street address for this tool.
6. Inbound-email handling from minors under the "never capture kids' emails" rule (no reply, same-day deletion, dedicated Gmail mailbox).
7. Font substitution wording that names commercial font trademarks; Reserved Font Name handling in the mirror, including Google-generated static instances of variable-only RFN families; UFL 1.0 naming for modified copies.
8. Repo license choice and the NOTICE carve-out.
9. Whether a free, globally available web tool needs any sanctions/export geo-restriction (expected: covered by general licenses for free internet services; not researched).
10. Whether Cloudflare's controller reuse of Turnstile signals and its `cf_chl_*` cookies fit the EU/UK strictly-necessary exemption (§8.2).


---

## 10. Security

### 10.1 Threat model

| Threat | Asset at risk | Primary controls |
|---|---|---|
| SSRF: making the Function fetch private or internal addresses | Cloudflare edge, third parties | §10.2 |
| Open proxy: using the Function to fetch or launder arbitrary content | The owner's domain reputation, third parties | §10.3, §10.4, §10.5 |
| Quota exhaustion (website reading off until 00:00 UTC) | Availability of URL reading; never cost | No $0 control while DNS is at Squarespace; narrow `_routes.json`; Fail open; E-CAPACITY (§10.3, R6) |
| Phishing or impersonation pages on the owner's domain via share links (including scannable codes and contact strings) | wethejamesphoto.com reputation, victims | §10.8, §6.2.6, §6.5.5 |
| XSS through fetched HTML/CSS, uploaded SVG/PDF, or share links | Visitors | §10.6, §10.7, §10.9-§10.11 |
| Decompression, entity and image bombs | Visitors' tabs | Caps in §6, §7.4 |
| Supply chain (a malicious dependency, Action, build tool or font-mirror asset) | Everyone; attacker content on the owner's domain | §10.12, §10.13, §7.1.3 |
| Hammering a third-party site through the tool | Third parties, legal exposure | Caps (§5.2.3), robots, do-not-fetch, limiter |

### 10.2 SSRF guard (every outbound fetch, every redirect hop, robots.txt included)

1. **Parse** with WHATWG `new URL()`. Numeric IPv4 forms (`2130706433`, `0x7f.1`) normalize to dotted-quad, so one check catches them [Likely].
2. **Scheme** `http:` or `https:` only. **No credentials** (`user:pass@`). **Strip** the fragment; the page URL also drops its query string.
3. **Port** empty, `80` or `443`. Required: on current compatibility dates, Workers honor custom ports for non-Cloudflare origins (`allow_custom_ports`) ([cf-compat-flags]) [Certain].
4. **No IP literals** (dotted-quad or bracketed IPv6). The platform also refuses them (error 1003) ([cf-known-issues]) [Certain], but rejecting explicitly gives clean errors.
5. **Hostname rules**: contains a dot; ≤ 253 characters; not `localhost` or `*.localhost`; not ending in `.local .internal .intranet .lan .home .home.arpa .corp .test .invalid .example .onion .arpa .pages.dev`; not a wildcard-IP DNS service (`nip.io`, `sslip.io`, `xip.io`, `localtest.me`, `lvh.me`, `1u.ms`, `traefik.me`) or its subdomains; not `PROD_HOST`.
6. **DoH pre-check** per distinct hostname (cached per invocation): `GET https://cloudflare-dns.com/dns-query?name={host}&type=A` and `type=AAAA` with `accept: application/dns-json`, 2 s timeout. Reject if any answer is in `0/8, 10/8, 100.64/10, 127/8, 169.254/16, 172.16/12, 192.0.0/24, 192.0.2/24, 192.88.99/24, 192.168/16, 198.18/15, 198.51.100/24, 203.0.113/24, 224/4, 240/4`, or `::/128, ::1/128, ::ffff:0:0/96` (check the embedded IPv4), `64:ff9b::/96` (embedded), `100::/64, 2001:db8::/32, 2002::/16` (embedded), `fc00::/7, fe80::/10, ff00::/8`. NXDOMAIN → `dns_error`. This does not stop DNS rebinding between the check and the fetch: Workers have no DNS-pinning or connect-time hook ([cf-known-issues]) [Certain that it is unavailable]. It is defense in depth; Workers egress is Cloudflare's network, which holds none of the owner's systems [Likely].
7. **Redirects**: `redirect: 'manual'`; resolve `Location` against the current URL; re-run steps 1-6 plus robots and do-not-fetch on the new URL; cancel each 3xx body; count each hop against the subrequest budget (hops count toward the 50-subrequest limit ([cf-limits]) [Certain]).
8. **Outbound headers built fresh**: never pass the incoming `Request`; send only `User-Agent: USER_AGENT`, `From: CONTACT_EMAIL` (the dedicated Kitsheet Gmail address, §9.0), a type-appropriate `Accept`, `Accept-Language: en`. Never forward `Cookie`, `Authorization`, `Referer`, `Origin`, `CF-*`, `X-Forwarded-*` or the visitor's user agent or language. `cache: 'no-store'` so Cloudflare does not cache third-party content ([cf-fetch]) [Certain].
9. **Byte caps**: reject on `Content-Length` over the cap, then pipe through a counting `TransformStream` that errors at the cap (a missing or false header cannot bypass it).
10. **Content-type allowlist** per kind (§5.2.2, §5.3); CSS also accepts `text/plain` only when the path ends in `.css`.
11. **Timeouts**: `AbortSignal.timeout()` per subrequest (§5.2.3) plus the overall deadline.
    - **11a. Transport and budget**: every outbound request, DoH and siteverify included, goes through `deps.transport` / `deps.resolve` (§4.9) and `budget.take(kind)` (§5.2.3).
12. **Compatibility**: compatibility date = first deploy date (≥ 2024-09-02) with the `global_fetch_strictly_public` flag ([cf-compat-flags]) [Certain].

Load-bearing sketch (TypeScript; final code must keep these semantics):

```ts
const PORTS = new Set(['', '80', '443']);
const BAD_SUFFIX = ['.localhost','.local','.internal','.intranet','.lan','.home','.home.arpa','.corp',
  '.test','.invalid','.example','.onion','.arpa','.pages.dev'];
const REBIND = ['nip.io','sslip.io','xip.io','localtest.me','lvh.me','1u.ms','traefik.me'];

export function checkUrl(raw: string, selfHost: string): URL {
  const u = new URL(raw);
  if (u.protocol !== 'https:' && u.protocol !== 'http:') throw new GuardError('scheme');
  if (u.username || u.password) throw new GuardError('credentials');
  if (!PORTS.has(u.port)) throw new GuardError('port');
  const h = u.hostname.toLowerCase().replace(/\.$/, '');
  if (h.startsWith('[') || /^\d{1,3}(\.\d{1,3}){3}$/.test(h)) throw new GuardError('ip-literal');
  if (h.length > 253 || !h.includes('.') || h === 'localhost') throw new GuardError('host');
  if (BAD_SUFFIX.some(s => h.endsWith(s)) || REBIND.some(s => h === s || h.endsWith('.' + s))) throw new GuardError('host');
  if (h === selfHost) throw new GuardError('self');
  u.hash = '';
  return u;
}
```

### 10.3 Open-proxy prevention

| Abuse | Control |
|---|---|
| Scripted mass use (work and third-party load) | Fresh Turnstile token per `/api/extract`; single-use, 300 s ([ts-verify]) [Certain]. A request without a valid token causes no fetch to any third party. This stops the fetcher from working for bots; it does not protect the quota (next row). |
| Quota drain (junk requests) | No $0 control. Every request that reaches a Function counts against the 100,000/day account quota before the handler runs ([cf-pages-fn-pricing], [cf-limits]) [Certain], including requests Kitsheet rejects: bad or missing token, wrong Host, wrong method, CORS preflight, `*.pages.dev` (which also invokes Functions and cannot be disabled), limiter 429s. Turnstile and the limiter cannot stop it. About 1.2 requests per second held for a day (100,000 / 86,400 s) uses the whole quota. The exact-path `_routes.json` (§10.6) keeps scanner noise on other `/api/…` paths from counting. The result is error 1027, then fail open, then E-CAPACITY until 00:00 UTC; uploads, manual entry, PDF, PNG and share links keep working, and nothing is billed. Accepted risk (R6). The only real control, WAF rate limiting, needs the zone on Cloudflare, which §11.4 #12 rules out. |
| Using `/api/images` as a general fetcher | Only URLs whose hash the server signed in the ticket, on the page's own hosts, its CSS hosts or the curated CDN list (§5.2.4); ticket bound to the caller's /64 or /32; 60 s expiry; one call per nonce per isolate (best effort); image types only; size caps |
| Cross-site use | POST only; `Content-Type: application/json` (forces a preflight that is never approved); `Origin` and `Sec-Fetch-Site` checks; no CORS headers. Non-browser clients can spoof these; Turnstile is the real gate for work. Preflights and no-cors POSTs from other sites still invoke the Function and count toward the quota. |
| Rendering attacker content on the owner's domain | Function responses are never `text/html`, `text/css` or `image/svg+xml`: KITX frames are `application/octet-stream` with `nosniff`, `Content-Disposition: attachment`, `CSP: default-src 'none'; sandbox`, `CORP: same-origin`. No GET handler exists. Unknown paths return the static 404 page (§10.6). |
| Preview or `*.pages.dev` endpoints bypassing checks | `Host` must equal `PROD_HOST`; preview deployments off; the Turnstile widget lists only `PROD_HOST`. These requests are refused but still count toward the quota. |
| Laundering requests to attack a site | Caps per analysis (§5.2.3), honest User-Agent, per-target limiter on every outbound hostname in both endpoints (§10.5), image hosts restricted (§5.2.4), do-not-fetch list |
| Storing or caching third-party content | `cache: 'no-store'`; no KV, R2, D1, DO or Cache API writes; no logging of URLs or bodies |

### 10.4 Bot gate (Turnstile)

- One widget, managed mode, hostnames: `PROD_HOST` only. The dev entry (§4.9) uses test sitekey `1x00000000000000000000AA` and test secret `1x0000000000000000000000000000000AA` for local and CI runs only; production uses only the production keys; no preview deployments exist. Never commit production keys.
- Loaded with `render=explicit` only when the user first presses **Read site** (PRIV-08). Users who only upload or type never contact Cloudflare's challenge endpoint.
- `appearance: 'interaction-only'`, `action: 'extract'`, `retry: 'auto'`, language auto. Reset the widget after each token is used.
- Server: `siteverify` with `secret` and `response` only; no `remoteip` and no `idempotency_key`. Cloudflare's idempotency key makes a retry with the same key and token return the original result instead of `timeout-or-duplicate`, so a key derived from request content (a hash of the token, say) would let one solved token replay for its full 300 s [Likely]; the Function never retries siteverify, so the parameter adds risk and no value. If it is ever added back for retries, it must be `crypto.randomUUID()` generated once per invocation, never derived from the token, the URL or the client. Accept only `success === true`, `hostname === PROD_HOST`, `action === 'extract'` and `challenge_ts` no more than 300 s old ([ts-verify]) [Certain on fields]. A secret starting with a documented test-secret prefix returns 500 `misconfigured` (§5.2.1). Unit test: the same token sent twice gives 403 `human_check_failed` the second time.
- Never enable pre-clearance (it sets `cf_clearance`) ([ts-preclearance]) [Certain].
- The challenge frame may set `cf_chl_*` cookies on `challenges.cloudflare.com`. They are disclosed (§9.1 item 6), not prevented, and verified across all domains (§15.6).
- No production Turnstile bypass of any kind, including for tests: no test key, skip header, allowlisted IP or debug flag (§4.9). The URL-read part of the production privacy check is run by hand (§15.6).
- Accessibility: Turnstile Free lists "WCAG 2.2 AAA compliance: Yes" ([ts-plans]) [Certain on the vendor claim]. If the check fails, every non-URL input still works (E-HUMAN).

### 10.5 Rate limits

Pages Functions cannot use the Workers Rate Limiting binding, and WAF rate-limit rules need the zone on Cloudflare ([cf-migrate], [cf-waf-rl]) [Certain]. The binding also has an open Free-plan bug reported as always returning success (workers-sdk #16097, labeled internal on 2026-11-04 ([cf-ratelimit-bug])) [Certain that it is open].

v1 limiter (`server/limiter.ts`), best effort, memory only:
- Key: HMAC-SHA256 of the client network prefix (the full address for IPv4, the /64 for IPv6, taken from `CF-Connecting-IP`) with a random salt generated at isolate start. One IPv6 /64 is one key, so an attacker cannot rotate through 2^64 addresses. The raw IP is never stored or logged.
- Per key: 10 `/api/extract` calls per 10 minutes, 40 per isolate lifetime per day [Guessing].
- Per target hostname: 30 fetches per hour across all users in the isolate, keyed on every outbound hostname in both endpoints (page, CSS and image hosts), not only the page host [Guessing] (F-8).
- Per ticket nonce: 1 `/api/images` call.
- LRU map capped at 10,000 entries.
- Limits are per Cloudflare location and reset when an isolate is recycled. None of these controls, Turnstile included, protects the daily quota. Every request that reaches `/api/extract` or `/api/images` is a Function invocation and counts toward the 100,000/day Workers Free limit before Kitsheet's code runs, rejected requests included (§10.3). Turnstile limits work, not quota: it stops unverified requests from causing any third-party fetch. The limiter runs before siteverify and before any subrequest (§5.2.1 order), so a rejected request costs one invocation and nothing more. While DNS stays at Squarespace there is no $0 control in front of the Function, because WAF and rate-limit rules need a Cloudflare zone. About 1.2 requests per second sustained (100,000 / 86,400 s) exhausts the quota; the designed outcome is E-CAPACITY until 00:00 UTC, at $0 [Certain on counting, per the Pages routing and pricing docs].

A Durable Object counter cannot protect the quota, because the Function invocation that calls it has already counted, and each Durable Object call is an extra request. It could only enforce a global per-target limit (protecting third-party sites) or per-client fairness. It is not in v1, and adding it needs re-review against §11.4 #3 and #5.

### 10.6 CSP and security headers (`_headers`, generated)

Pages `_headers`: at most 100 rules, 2,000 characters per header line ([cf-pages-limits]) [Certain]. CSP is set per route, never on `/*`, because headers from several matching rules combine and two CSPs would both be enforced [Likely]. Function responses set their own headers in code (§5.2.8).

**One table, two copies.** `src/security/csp.ts` holds each route's policy. `scripts/gen-headers.mjs` writes `dist/_headers` from it and injects each policy as the first child of `<head>` in that route's HTML: `<meta http-equiv="Content-Security-Policy" content="…">` with the same directives minus `frame-ancestors`, which a meta tag cannot carry (`X-Frame-Options: DENY` on `/*` covers framing). The two copies cannot drift, and duplicating an identical policy is harmless. Reason: Pages matches `_headers` and `_redirects` against the raw request path but looks up assets after percent-decoding, so a path such as `/%6B/` can serve `k/index.html` without the `/k/*` block [Certain in source; Guessing whether the edge normalizes first]. The meta copy keeps CSP and Trusted Types on the document whatever the path.

**404 mode.** `src/pages/404.html` is a Vite entry emitted at `dist/404.html`: static, no JS, styled like the legal pages, every asset URL absolute (`/assets/…`) because it is served at any depth, `<meta name="robots" content="noindex">`, the meta policy `default-src 'none'; style-src 'self'; img-src 'self'; font-src 'self'; base-uri 'none'; form-action 'none'`, and the copy "No page here." with links to `/` and `/privacy`. With a top-level 404.html, Pages runs in 404 mode, not SPA mode: unmatched paths return 404 with this page and `Cache-Control: no-store`, and the app HTML is served only at `/` ([cf-serving-pages]) [Certain]. Without it, Pages would serve `/index.html` for every unknown path, and that response would carry only the `/*` block: the full app with no CSP and no Trusted Types.

**Flat files.** Static pages are emitted as `dist/privacy.html`, `terms.html`, `acceptable-use.html`, `credits.html`, `accessibility.html`, `bot.html`, `report.html` and `clear.html`. Pages serves `/privacy` with a 200 and sends `/privacy.html` and `/privacy/` to `/privacy` with a 308 [Certain: pages-shared `handler.ts`], so the exact rules below match the final URL. Only `/` (`index.html`) and `/k/` (`k/index.html`) stay index files. Every footer link, in-page link and Clear my data link uses the exact form (`/clear`, never `/clear/`).

**If a `/*` baseline CSP is ever added**, it must be the first rule in the file, every specific route must begin with `! Content-Security-Policy`, and `/assets/*` must also detach it. Dedicated workers take their CSP from their own script response [Likely], so a `default-src 'none'` baseline would block worker chunk imports and `/fonts/` fetches.

```
/*
  Strict-Transport-Security: max-age=31536000
  X-Content-Type-Options: nosniff
  Referrer-Policy: strict-origin-when-cross-origin
  Cross-Origin-Opener-Policy: same-origin
  X-Frame-Options: DENY
  Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=(), usb=(), browsing-topics=()

/
  Content-Security-Policy: default-src 'none'; script-src 'self' https://challenges.cloudflare.com; frame-src https://challenges.cloudflare.com; connect-src 'self'; img-src 'self' blob:; font-src 'self'; style-src 'self' 'unsafe-inline'; worker-src 'self'; manifest-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'; object-src 'none'; require-trusted-types-for 'script'; trusted-types bk-inert bk-probe bk-worker

/k/*
  Content-Security-Policy: default-src 'none'; script-src 'self'; connect-src 'self'; img-src 'self' blob:; font-src 'self'; style-src 'self'; worker-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'; object-src 'none'; require-trusted-types-for 'script'; trusted-types bk-worker
  X-Robots-Tag: noindex, nofollow, noarchive
  ! Referrer-Policy
  Referrer-Policy: no-referrer

/privacy
  Content-Security-Policy: default-src 'none'; style-src 'self'; img-src 'self'; font-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'
/terms
  Content-Security-Policy: default-src 'none'; style-src 'self'; img-src 'self'; font-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'
/acceptable-use
  Content-Security-Policy: default-src 'none'; style-src 'self'; img-src 'self'; font-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'
/credits
  Content-Security-Policy: default-src 'none'; style-src 'self'; img-src 'self'; font-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'
/accessibility
  Content-Security-Policy: default-src 'none'; style-src 'self'; img-src 'self'; font-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'
/bot
  Content-Security-Policy: default-src 'none'; style-src 'self'; img-src 'self'; font-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'
/report
  Content-Security-Policy: default-src 'none'; style-src 'self'; img-src 'self'; font-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'
/404.html
  Content-Security-Policy: default-src 'none'; style-src 'self'; img-src 'self'; font-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'

/clear
  Content-Security-Policy: default-src 'none'; script-src 'self'; style-src 'self'; font-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'
  Cache-Control: no-store

/assets/*
  Cache-Control: public, max-age=31536000, immutable
/fonts/*
  Cache-Control: public, max-age=31536000, immutable
/blocklist.json
  Cache-Control: public, max-age=300
/watchlist.json
  Cache-Control: public, max-age=300
```

The `! Referrer-Policy` line detaches the global value so `/k/` sends only `no-referrer` [Likely: Pages `_headers` supports detaching with `!`; the header test below confirms every route's final headers, including that `/k/` itself matches `/k/*`]. Turnstile needs only `script-src` and `frame-src` for `https://challenges.cloudflare.com` ([ts-csp]) [Certain]. The static legal pages run no JavaScript; their "Clear my data" control is a plain link to `/clear`, whose **Clear now** button runs the in-page clear (§7.6.3). `/clear` deliberately sends no `Clear-Site-Data` header.

Why the app route allows `'unsafe-inline'` styles: the render probe's `srcdoc` document inherits the parent's CSP ([csp3-inherit]) [Certain], and it must apply the fetched site's CSS as inline `<style>`. A per-response nonce is impossible on static hosting. The risk is limited because no untrusted markup is ever inserted into the app document itself (it goes only into the script-less sandboxed probe), and Trusted Types blocks HTML sinks.

`public/_routes.json`: `{"version":1,"include":["/api/extract","/api/images"],"exclude":[]}`. Only these two handlers invoke Functions. Any other `/api/...` path (scanner noise such as `/api/.env`) is served as static, gets the 404 page, and does not count against the quota ([cf-pages-routing]: include "Defines routes that will be invoked by Functions") [Certain]. Exact-path matching is confirmed by the header test [Likely]. `public/_redirects`: `/k/:payload  /k/  200` (§7.3.4). `public/robots.txt`: `User-agent: *` / `Disallow: /api/` (the share route stays crawlable so crawlers see its `noindex`); while `LAUNCHED` is false, `Disallow: /` (§16).

**Header test** (M1; runs against `wrangler pages dev dist` in CI and against production after every deploy). Request every `*.html` in `dist/` at its canonical URL, plus `/does-not-exist`, `/k/x/y`, `/index.html`, `/privacy/`, `/privacy.html`, `/clear/`, `/k`, `/k/%231.AAAA.4`, `/%6B/`, `/%70rivacy`, and GET and POST to `/api/anything-else`. Follow redirects. On the final response, assert:
- unknown paths return 404 with the 404 page and no `/assets/*.js` script;
- every 200 HTML response at a canonical path has exactly one `Content-Security-Policy` header equal to the table value, and a first-in-head meta CSP equal to the table value (minus `frame-ancestors`);
- `/clear` answers 200 with no 3xx, `Cache-Control: no-store` and no `Clear-Site-Data`;
- `/k/%231.AAAA.4` serves the share viewer with the `/k/*` headers;
- `/api/anything-else` does not invoke the Function (static response, no `X-Kit: 1`).
The deploy job fails if wrangler's output reports any ignored or invalid `_redirects` or `_headers` rule.

### 10.7 Trusted Types policies

| Policy | Route | Accepts | Used by |
|---|---|---|---|
| `bk-inert` | `/` | Any string; returns it unchanged | Only `engine/html-ingest.ts` and `engine/svg/allowlist.ts`, and only for `DOMParser.parseFromString`, which produces an inert document ([mdn-tt] lists it as a TrustedHTML sink) [Certain] |
| `bk-probe` | `/` | Only strings registered by the probe builder in a module-private `Set` | `engine/render-probe.ts` for `iframe.srcdoc` (a TrustedHTML sink) [Certain] |
| `bk-worker` | `/`, `/k/` | Same-origin URLs whose path matches `^/assets/[\w.-]+\.js$` | Every `new Worker()` (the URL argument is a TrustedScriptURL sink) [Certain]; Vite's automatic worker syntax is not used |

No `default` policy. ESLint bans `innerHTML`, `outerHTML`, `insertAdjacentHTML`, `document.write`, `srcdoc`, `createContextualFragment`, `setHTMLUnsafe` and `dangerouslySetInnerHTML` outside those modules.
- `security/trusted-types.ts` exports one function per policy, and each creates its policy lazily. `/k/` only ever calls the `bk-worker` one, because `createPolicy` throws for names the route's CSP does not list.
- An ESLint `no-restricted-imports` zone, plus a build check on the `/k/` chunk graph, forbids `src/share/**` and `src/outputs/share/**` from reaching `engine/html-ingest.ts`, `engine/svg/allowlist.ts`, `engine/svg/canonicalize.ts`, `engine/render-probe.ts` or any `DOMParser` or `srcdoc` use. The share route validates SVG only with `engine/svg/canonical-grammar.ts` (§6.2.2).

**Turnstile fallback:** in M3, test Turnstile under the `/` policy in Chrome, Firefox and Safari. If its script needs an HTML sink, remove `require-trusted-types-for 'script'` from `/` only (keep the `trusted-types` list and the ESLint bans); `/k/` keeps enforcement because it loads no third-party script. Record the outcome in `docs/compliance/data-flow.md`.

### 10.8 Share links: untrusted input and anti-impersonation

Decoding is §7.4. Rendering rules:
- **Banner** (persistent, cannot be dismissed, kept in print CSS and in any PDF or PNG exported from the share view; S-3): "**Unverified.** Someone made this kit with Kitsheet and shared it as a link. Kitsheet doesn't check who made it. It is not from, approved by, or connected to "{name}". This page never asks you to log in, pay, share personal details, call a number or scan a code. [Report this link]"
- **Chrome never borrows the brand** (S-4): `document.title` stays "Shared brand kit · Kitsheet"; favicon is Kitsheet's; OG tags static.
- **No interactive surface** (S-5): no `<form>`, `<input>`, `<textarea>` or `contenteditable` on `/k/`; `form-action 'none'`; `frame-ancestors 'none'`.
- **No links or scannable codes from kit data** (S-6): domains and text render as plain text. Name, tagline, color names and custom font names must pass `containsContact()` (§6.5.5) at encode and decode, which rejects URLs, email addresses and phone numbers in common formats. Logos containing a QR code or other 2D code are refused at encode and hidden at decode (§6.2.6). This stops casual callback-phishing and link bait. It is not a guarantee: spelled-out numbers and text drawn as outlines inside a logo are not detected (R1).
- **Tight caps** (S-7): name 60, tagline 120, colors 8, fonts 2, canonical SVG 8 KB, no raster.
- **Blocklist** (S-8) and **watchlist** (S-9) as in §7.4, matched on the homoglyph-resistant match key. Optional: `watchlist.json` also lists generic phishing phrases ("account security", "support center", "help desk", "verify your", "refund department") that trigger S-WATCH.
- **Search**: `noindex` header and meta; fragments are never indexed.
- **Monitoring** (S-11): verify a URL-prefix property for `https://<host>/` in Google Search Console by HTML-file upload (no DNS change) to see Security Issues notices and request reviews.
- **Isolation option** (S-1, S-2): `/k/` is its own path prefix; `SHARE_ORIGIN` can move the viewer to the `*.pages.dev` hostname (§0.3).

### 10.9 SVG

§6.2: refuse DOCTYPE/ENTITY, size and element caps; inert parse; structural allowlist; probe-computed paints; canonical re-serialization from an allowlist; render only as `blob:` `<img>` from validated strings; the canonical form is the only SVG stored, embedded or shared; decoders require that `validateCanonical` returns a shared SVG byte for byte, using the pure grammar module with no DOM parser (§6.2.2, §7.4). Fetched SVG (logos and favicons) goes through the same pipeline and is never turned into an `image/svg+xml` Blob as raw bytes (§5.3). Logos with scannable codes are refused in links (§6.2.6).

### 10.10 PDF uploads

§6.4: pdf.js legacy build at the §4.2 pin in a worker; no eval path; no wasm; no scripting sandbox; XFA off; no system fonts; `disableFontFace`; size, page and time caps; `destroy()` plus `worker.terminate()` on timeout or cancel. Text and fills are read as data; nothing from the PDF is ever inserted as HTML.

### 10.11 Fetched HTML and CSS

§5.4.2: parsed inertly; stripped of active elements, attributes and links; CSS pre-pass caps (`:has()` rules, selector size, custom-property bytes, DOM size); every CSS URL rewritten to absolute against its real base; rendered only inside a sandboxed `srcdoc` iframe without `allow-scripts`, under the inherited app CSP plus its own `default-src 'none'` meta CSP; sliced style reads with a 5 s watchdog; byte caps upstream. Text read from the page (name, tagline, alt) is normalized (§6.5.5) and rendered with `textContent`.

### 10.12 Dependency and supply-chain policy

- Exact versions and `package-lock.json`; `npm ci --ignore-scripts` in CI (Playwright browsers installed by an explicit step).
- **Cooldown.** Every pin is the newest release at least 7 days old at build start (§4.2). `.npmrc` sets `min-release-age=7`, and `packageManager`/`engines` require npm 11.10 or newer [Certain: npm config docs]; a pin that is too new makes `npm install` fail, which forces a choice. `scripts/check-release-age.mjs` runs in ci.yml: for each package whose version changed in `package-lock.json` against the base branch, it reads `time[version]` from registry.npmjs.org and fails if the release is under 7 days old. A security fix with an advisory can be exempted with `min-release-age-exclude`, in a commit that names the GHSA. In `dependabot.yml`, `cooldown: default-days: 7` for npm, pip (`/scripts/font-mirror`) and github-actions; GitHub's own default is 3 days, and security updates are exempt [Certain: GitHub changelog 2026-07-14].
- **Build tools are pinned like runtime libraries.** wrangler is an exact devDependency and is invoked only as `./node_modules/.bin/wrangler`. Never use `npx`, `npm exec`, `npm install` or `cloudflare/wrangler-action` without a local wrangler, and never download a tool unpinned (no `curl … | sh`). fonttools and zizmor are installed from hashed requirements (`pip install --require-hashes --only-binary=:all:`). Reason: the deploy job holds the Pages:Edit token, and an unpinned tool fetched at deploy time is the pattern the 2025 npm worms used to harvest CI tokens.
- A new runtime dependency requires a spec change that records its license, size and why our own code will not do.
- CI license allowlist (O-4): MIT, ISC, BSD-2-Clause, BSD-3-Clause, Apache-2.0, MPL-2.0 (file-level), 0BSD, CC0-1.0, Unlicense, OFL-1.1 (fonts only). Fail on GPL, AGPL, LGPL, CC-BY-NC, missing licenses. Do not copy data from the GPL-3.0 Wappalyzer fork.
- `npm audit --audit-level=high` and an OSV scan ([osv]) in CI; Dependabot weekly, grouped (npm, pip and github-actions); CodeQL default setup; secret scanning with push protection; all on standard runners (free) ([gh-dependabot]) [Certain/Likely].
- **Workflow hygiene.** GitHub Actions pinned by commit SHA. Every workflow has top-level `permissions: {}` and grants `contents: read` per job (the font-mirror `publish` job alone gets `contents: write`). `workflow_run`, `pull_request_target` and `issue_comment` are banned in every workflow. ci.yml runs zizmor (`zizmorcore/zizmor-action` pinned by SHA, or `pip install --require-hashes zizmor==<pinned>`) and fails on `dangerous-triggers`, `excessive-permissions`, `artipacked`, `unpinned-uses` and `cache-poisoning`, plus a grep step that fails if any workflow contains `npx `, `npm install`, `npm exec`, `workflow_run` or `pull_request_target`. Checkouts use `persist-credentials: false`.
- `check-no-eval.mjs`: fails the build if `eval(` or `new Function(` appears in any shipped bundle (PDFKit, svg-to-pdfkit, fflate, valibot and SVGO bundles contain none; pdfmake does, which is one reason it is not used) [Certain]. The only allowlisted match is core-js's `Function("return this")()` in the two legacy pdf.js files (§6.4).
- `check-fn-bundle.mjs` (§4.9) and `check-copy.mjs` (§12.9) run in ci.yml.
- No CDN-hosted scripts besides Turnstile (which cannot use SRI because Cloudflare updates it).

### 10.13 Secrets

| Secret | Where | Rotation |
|---|---|---|
| `TURNSTILE_SECRET` | Pages encrypted environment variable (production) | On suspicion; Turnstile dashboard |
| `TICKET_HMAC_KEY` (32 random bytes) | Pages encrypted environment variable | Quarterly or on suspicion; old tickets simply expire |
| `CLOUDFLARE_API_TOKEN` (scope: Account › Cloudflare Pages › Edit, this account only) | GitHub environment `production`, required reviewer = owner | On suspicion; revoke in Cloudflare |
| `CLOUDFLARE_ACCOUNT_ID` | GitHub environment variable (not secret) | — |
| `GOOGLE_FONTS_API_KEY` (restricted to the Web Fonts Developer API) | GitHub Actions secret used only by the `build` job of `font-mirror.yml` | Yearly |

The repo is public: never commit secrets. The dev entry (§4.9) uses test sitekey `1x00000000000000000000AA` and test secret `1x0000000000000000000000000000000AA` for local and CI runs only; production uses only the production keys; no preview deployments exist. Fork PRs never receive secrets because no workflow uses `workflow_run` or `pull_request_target` (enforced by zizmor); that GitHub default applies only to `pull_request`. The production Pages project holds no variables besides `TURNSTILE_SECRET` and `TICKET_HMAC_KEY` (checked at M10).

### 10.14 Abuse and takedown runbook (`docs/ops/takedown.md`)

1. Report arrives at `CONTACT_EMAIL` (subject "Report:").
2. Open the link in a private window; confirm the issue.
3. Add the SHA-256 of the canonical JSON (and, for brand-owner requests, the name's match key, §7.4) to `public/blocklist.json` in a commit whose message contains only the hash, never the payload.
4. Deploy (normal CI). Target: within 2 business days (R-3).
5. If Search Console shows a Safe Browsing issue, request a review after the block deploys.
6. Delete the report email, and empty Trash, within 30 days after closing it. If a message shows the sender is under 13, do not reply. Act on any report it contains. The same day: delete the message, delete any reply from Sent, empty Trash, and remove the address from Contacts and Contacts › Other contacts. This is stricter than COPPA's one-time-response exception (16 CFR 312.5(c)(3)), which permits one reply only if the address is deleted promptly afterwards (§8.1).
7. Do-not-fetch requests: add the hostname to `server/do-not-fetch.json` and deploy within 2 business days (F-6).

---

## 11. $0 cost model

### 11.1 Services, free limits and what happens at each limit

| Service | Used for | Free limit (read 2026-11-08) | At the limit | Payment method |
|---|---|---|---|---|
| Cloudflare Pages static assets | All pages, JS, CSS, fonts | Requests free and unlimited; 20,000 files per site; 25 MiB per file; 100 custom domains per project ([cf-pages-limits], [cf-pages-fn-pricing]) [Certain] | Deploy fails if files exceed the limit (CI pre-checks at 19,000); serving never stops for volume | None |
| Pages Functions (Workers Free plan) | `/api/extract`, `/api/images` (every request to them counts, rejected and automated ones included) | 100,000 requests/day per account, reset 00:00 UTC; 10 ms CPU per request; 50 subrequests; 128 MB memory ([cf-limits]) [Certain] | Error 1027 on Function routes until reset; with **Fail open**, static assets keep serving ([cf-pages-routing]) [Certain]. CPU overrun: error 1102 for that request. No overage billing exists on Free ([cf-pricing]) [Certain] | None |
| Cloudflare Turnstile Free | Bot check | Unlimited challenges; 20 widgets; 10 hostnames per widget ([ts-plans]) [Certain] | n/a | None |
| Cloudflare logs | Real-time Function logs only | Workers Logs is not available to Pages ([cf-migrate]) [Certain]. From 2026-12-01 the Observability free tier is 0.5 GB/day with 7-day retention and stops ingesting at the limit on Free ([cf-observability]) [Certain on docs] | Ingestion stops; nothing billed | None |
| GitHub (public repo) | Source, CI, Dependabot, CodeQL, secret scanning, Releases (font mirror) | Standard runners in public repos are free and do not consume included minutes ([gh-actions-billing]) [Certain]. Release assets in a public repo carry no charge, up to 2 GiB per file [Likely; verify on first publish]. Scheduled workflows in a public repo are disabled after 60 days without repository activity [Likely]; §11.5 does not depend on them | Without a payment method, usage over included amounts is blocked, not billed [Certain] | None |
| Google Fonts Developer API | Build-time family list | Free key; quota reported as 10,000 requests/day [Likely; verify] | Mirror build fails; production keeps the previous mirror; the no-key `google/fonts` fallback (§7.1.3 step 1) is proven in M2 | Google Cloud project with **no billing account** |
| Squarespace Domains DNS | One CNAME | Existing domain | n/a | Existing owner cost (domain renewal), not new |
| Gmail (free) | Contact mailbox used only for Kitsheet (§9.0) | n/a | n/a | None |
| Google Search Console | Safe Browsing notices | Free | n/a | None |
| npm registry, PyPI | Packages at build | Free | n/a | None |

**Card on file.** Community sources say Workers Free, Pages, Turnstile and Web Analytics activate without a card, while R2 requires one [Likely]; no official page states "no card required". Verify during account creation (M0). Do not add one even if prompted for an optional feature.

**Commercial use.** Cloudflare's developer-platform terms contain no non-commercial restriction on Free; they reserve the right to limit requests that unduly burden the network ([cf-dev-terms]) [Certain]. By contrast Vercel Hobby is "non-commercial personal use only" and a footer promoting the owner's services plausibly breaks it ([vercel-hobby]); Netlify Free pauses every site when credits run out and each production deploy costs credits ([netlify-credits]); Fly.io requires a card ([fly-pricing]) [Certain on quoted terms]. Those hosts are rejected.

### 11.2 Capacity math (arithmetic on the documented quota, not a forecast; no-abuse case)

- Quota Q = 100,000 Function invocations per day, counting every request to `/api/extract` and `/api/images`, including rejected, automated and cross-site requests, and anything else on this Cloudflare account (which is why the account is dedicated).
- One website read = 1 `/api/extract` + 1 `/api/images` = 2 invocations → at most 50,000 reads/day. (A read whose page yields no image candidates uses 1.)
- The read count above is a ceiling, not a guarantee. Anyone can use up Q with junk requests at about 1.2 requests per second (§10.3).
- Uploads, manual entry, PDF, PNG, share links and the share view use 0 invocations.
- Static traffic has no cap.
- A traffic spike, or a flood of junk requests (which count even when rejected, §10.3), drains the day's quota early; after that only website reading stops until 00:00 UTC (6:00 pm Central Standard Time or 7:00 pm Central Daylight Time). No number is ever printed in the UI (INFRA-32).

### 11.3 Behavior at limits

- **Billing fails closed everywhere**: no account has a payment method or a paid plan, so no limit can turn into a charge.
- **Functionality degrades, never breaks**: Pages is set to **Fail open** (Settings › Runtime) [Certain that the setting exists], so the app and every upload feature keep working when Functions are exhausted.
- **Client detection** (INFRA-30), a three-way classifier for every `/api/*` response:
  1. `X-Kit: 1` present: an application outcome. Kitsheet's own limiter's 429 maps to E-BUSY; other codes map per §2.10.
  2. **Capacity shape**: a response without `X-Kit` that is not JSON (the static 404 page, a 405, or any HTML fall-through under fail open), or any body containing "Error 1027". Retry once after 2 s with the same Turnstile token (the Function did not run, so the token is unused). If the retry is also capacity-shaped, show E-CAPACITY until the next 00:00 UTC, kept in memory only. One response alone never locks the day.
  3. **Heavy shape**: a Cloudflare error body for 1101 or 1102, or any other 5xx without `X-Kit`: E-HEAVY. No automatic retry (a 1102 page would fail again and spend another invocation) and no day-long lockout.
  Network errors (no response) are retried once, then E-OFFLINE or E-SERVER.
  The exact fail-open response shape for a POST is unverified; M3 records it on a scratch project by toggling fail open and closed, and the §15.5 capacity tests cover each shape [Guessing].

### 11.4 Never do (each would add cost or a path to cost)

1. Add a payment method to Cloudflare, GitHub or Google Cloud.
2. Upgrade to Workers Paid ($5/month minimum) or any paid Cloudflare plan.
3. Enable R2, KV, D1, Queues, Durable Objects, Logpush, Images, Stream, Argo, Load Balancing, Browser Run, paid Observability, or Turnstile Enterprise features.
4. Route anything but `/api/extract` and `/api/images` through Functions, or proxy fonts or images for display through a Function.
5. Put a second project on the same Cloudflare account (the quota is account-wide).
6. Add any third-party script, analytics, error tracker, chat widget, font CDN or embed.
7. Move hosting to Vercel, Netlify, Render, Fly.io or any host needing a card or with paid overage.
8. Buy a domain, a certificate, a mailbox, a DMCA agent registration, a DBA filing or an LLC as part of this project.
9. Send email or SMS from the server (also a TCPA/CAN-SPAM trigger).
10. Use GitHub larger runners, Codespaces or Git LFS.
11. Call any API that requires a billing account, including from CI.
12. Move the wethejamesphoto.com nameservers to Cloudflare. That is not a cost, but it risks the owner's business email; it is out of scope for v1. (It is also the only way to get WAF rate limiting in front of the Functions; R6 records that trade-off as an owner decision.)
13. Buy devices or a paid device-cloud subscription for testing. §15.8 uses devices the owner already owns plus free emulators.

### 11.5 Monthly $0 check (`docs/ops/monthly-check.md`)

The primary trigger is a recurring 1st-of-month event on the owner's own calendar (free) linking to `docs/ops/monthly-check.md`. The scheduled `monthly-check.yml` Action, which opens a reminder issue, is a convenience: GitHub disables scheduled workflows in public repos after 60 days without repository activity [Likely], and a finished portfolio repo will go quiet. The Action lists workflow states with `gh api repos/{owner}/{repo}/actions/workflows` and records any workflow in `disabled_inactivity` state in the issue. A disabled font-mirror schedule is harmless, because deploys pin the last Release.

0. If this month's issue did not appear, open Actions and re-enable disabled workflows.
1. Cloudflare › Billing: plan Free, no payment method, $0 invoices.
2. Cloudflare › Workers & Pages › Plans: "Free". Usage last 30 days: peak daily requests vs 100,000; count of days that hit 1027. For each day that hit 1027, compare Function requests with Turnstile challenges solved that day (§14.1). A legitimate read uses at most 2 invocations per solved challenge, so requests well above 2 × solved mean junk traffic. Record it in the issue. No action is required (accepted risk, R6); if such days recur, the owner decides whether to revisit R6.
3. Cloudflare › Pages › project › Settings › Runtime: **Fail open**. Custom domain **Active**. Functions metrics: CPU p50/p99 and Exceeded Resources (§4.5).
4. GitHub › Settings › Billing: no payment method; $0; Actions on standard runners only.
5. Google Cloud console: the fonts-API project shows "no billing account".
6. Squarespace: wethejamesphoto.com auto-renew on (expires 2027-08-24).
7. Production privacy check, in two parts:
   - Automated: run the §15.6 network-allowlist, cookie and storage tests against `https://<host>/` for page load, every upload type, PDF and PNG export, share creation and the share view. These make zero Function calls and need no Turnstile.
   - Manual: the owner does one URL read in a clean Chrome profile in a normal browser, saves the DevTools network log (HAR), and records every cookie (any domain, partitioned included) and every storage key in the Turnstile frame, with names and lifetimes, against the `data-flow.md` allowlist. Attach the HAR and results to the issue.
8. Kitsheet mailbox: auto-contacts set to "I'll add contacts myself"; Contacts and Other contacts hold no correspondents; no forwarding, POP/IMAP or connected third-party apps; Trash holds no mail from matters closed more than 30 days ago.
9. Record the date and result in the issue; close it.

---

## 12. Design direction

### 12.1 The anchor decision: the proof sheet

**One idea: the app is a printer's proof on a desk.** A narrow margin column on the left holds the tool's controls as small, quiet margin notes. A wide white sheet on the right holds the user's kit exactly as it will print. The brand name is the dominant headline. It is set huge in the brand's own heading font at the top of the sheet's Cover section, and it is the only element that starts left of the content edge: it hangs into the page's white margin by half that margin. Everything else aligns to the content edge. The same hang appears on the PDF cover (18 pt into a 36 pt margin) and the PNG board (48 px into a 96 px margin). The name never crosses from the sheet onto the desk, and it renders once, as the Cover headline.

Why it fits this audience:
- Small business owners, freelancers and marketers already know "check the proof before it prints". The screen is the proof, so what they approve is what they download.
- The tool's chrome stays quiet so the user's colors and fonts are the loudest thing on screen. That matters because the tool's single accent must never compete with, or be mistaken for, the user's palette (and it keeps the tool's branding separate from kit content, L-4).
- The contact-sheet and proof vocabulary comes from the owner's photography background and delivers the "film look" directive without any stock imagery.

### 12.2 Type system

| Token | Family / weight | Size / line height | Use |
|---|---|---|---|
| `display` | **Brand heading font** (fallback Archivo 800) | Fitted by `fitHeadline` between 56 and 160 px / 0.95; `clamp(56px, 10vw, 160px)` is only the maximum | Cover headline only (the brand name, or the empty-state `<h1>` in §2.3); fitted to at most 2 lines between 56 and 160 px; 3 lines allowed down to 32 px |
| `h1` | Archivo 800 | 40 / 44 | Section titles in the margin |
| `h2` | Archivo 800 | 28 / 32 | Panel titles |
| `h3` | Archivo 800 | 20 / 28 | Field group titles |
| `body` | Archivo 400 | 16 / 24 | All running text |
| `small` | Archivo 400 | 14 / 20 | Microcopy |
| `label` | JetBrains Mono 400, uppercase, letter-spacing 0.08em | 12 / 16 | Section labels ("P.05 COLOR"), sources, hex values |

Two weights only (400 and 800) for real contrast. Hex codes, RGB values and measurements are always mono with tabular figures. Both families are OFL 1.1 without a Reserved Font Name and are served from the mirror (§7.1.3).

**Headline fitting** (`src/ui/fit-headline.ts`, one function shared by the screen, the PDF and the PNG): `fitHeadline(text, font, width, {max, floor, maxLines, floor2, maxLines2})`. Step down from `max` to `floor` and accept the first size with at most 2 lines, breaking only at spaces and hyphens. If none fits, allow 3 lines down to a second floor (PDF 36 pt, PNG 48 px, screen 32 px). As a last resort, break inside the longest word. Never truncate or ellipsize the name. On screen it runs on resize and on font load. Fixtures: a 60-grapheme name, a 20-letter single word, a CJK name and a wide-script heading font, checked at 320 px, at 400% zoom, on the PDF cover and on the PNG.

### 12.3 Color system: neutral plus one accent

The sheet is always white (`#FFFFFF`) in both themes, because it is the printed proof; kit colors are shown on it truthfully. Only the app chrome changes theme. Contrast values below were computed with the WCAG 2.2 formula on 2026-11-08 [Certain].

| Token | Light | Dark | Use and measured contrast |
|---|---|---|---|
| `--paper` | `#F4F1EA` | `#141310` | App background (the desk) |
| `--sheet` | `#FFFFFF` | `#FFFFFF` | The proof sheet |
| `--sheet-ink` | `#16150F` | `#16150F` | Not themed. All text and focus rings on the sheet: 18.29:1 on `#FFFFFF`. (Dark-theme `--ink` would be 1.22:1 on white.) |
| `--ink` | `#16150F` | `#ECE8DF` | Text. Light 16.21:1 on paper, 18.29:1 on sheet. Dark 15.19:1 on paper |
| `--ink-2` | `#5C594F` | `#A8A396` | Secondary text. Light 6.21:1 on paper, 7.01:1 on sheet. Dark 7.38:1 |
| `--line` | `#8E897C` | `#6E6A5F` | Control borders (WCAG 1.4.11). Light 3.09:1 on paper, 3.49:1 on sheet. Dark 3.44:1 |
| `--rule` | `#C8C3B6` | `#4A473F` | Decorative hairlines only, never a control's only boundary |
| `--accent` | `#C2261A` | `#FF6B57` | The one accent ("china-marker red"). Light 5.19:1 on paper; white text on it 5.86:1. Dark 6.63:1 on paper; `#141310` text on it 6.63:1 |

Accent rules: used only in the margin column, and only for the primary action button, the current step marker, the "edited" mark, and focus rings in the margin. Never on the sheet, never in generated PDFs or PNGs, never decorative. On the sheet, focus rings are `--sheet-ink` 2 px with a 2 px white inset (the dark accent is only 2.80:1 on white [Certain], so it is not used there). Dark theme has no raised surfaces (a raised `#2A2823` surface would drop `--line` to 2.73:1 [Certain]). Theme follows `prefers-color-scheme` with a stored override (`bk.theme`).

### 12.4 Layout grid

- 12 columns, max content width 1440 px, outer margin 24 px (16 px under 768 px), gutters 24 px.
- ≥ 1200 px: margin column = columns 1-4 (sticky); sheet = columns 5-12. Sheet inner padding is 64 px. The Cover headline's left edge sits 32 px from the sheet's left edge; all other sheet content starts at 64 px.
- 768-1199 px: margin column = columns 1-5; sheet = 6-12. Padding 48 px, hang 24 px.
- < 768 px: single column, in DOM and visual order: (1) the sheet's Cover block (headline and tagline, on white, with the top crop marks); (2) the margin controls; (3) the remaining sheet sections (with the bottom crop marks). Padding 24 px, hang 12 px. Export panel becomes a bottom bar.
- All widths: the Cover block and the rest of the sheet are adjacent siblings with no gap, so they read as one sheet. DOM order is Cover block, margin column, rest of sheet (WCAG 1.3.2 and 2.4.3). Right-to-left names hang into the right padding instead.
- Spacing scale: 4, 8, 12, 16, 24, 32, 48, 64, 96 px. Large, deliberate empty space between the Cover block and the next sheet section, and around each sheet section.
- Crop marks (four flat L-shaped 1 px lines in `--ink-2`) at the sheet corners are the only ornament.

### 12.5 Components

Square corners everywhere (radius 0). No shadows. No gradients. No blur. 1 px borders in `--line`.

| Component | Notes |
|---|---|
| Button | Primary: `--accent` fill, white (light) or `--paper` (dark) text, 44 px tall. Secondary: 1 px `--line` border, `--ink` text. Quiet: text only, underlined on hover and focus. |
| Text field | 44 px; 1 px `--line` border; label above; error text below with an icon and the word "Error"; never color alone. |
| File button | A real `<input type="file">` behind a visible button label; the whole margin column is also a drop target, outlined in 2 px dashed `--line` while dragging. |
| Step list | Ordered list; each item has a text status ("Done", "Skipped: …", "Failed: …") plus a 16 px line icon. |
| Swatch | Color block with hex always printed as text in mono, in whichever of `#000000`/`#FFFFFF` passes AA on it. `forced-color-adjust: none` on the block only. |
| Color row | Hex field, native picker, role select, name, lock, move up/down, remove (all keyboard reachable). |
| Font picker | ARIA combobox + listbox (§13.3). |
| Notice | Inline block with a 2 px left rule in `--ink` and an icon; error notices start with "Error:". |
| Dialog | Native `<dialog>`; used only to confirm replacing a draft. |
| Icons | A small set of 24 px line icons (1.5 px stroke) drawn as inline SVG components. No emoji, no icon font, no isometric or 3D art. |

### 12.6 Motion

Minimal. 120 ms opacity transitions for notices and panel reveals; the step list's 1 px indeterminate line. No parallax, no animated illustrations, no page transitions. Under `prefers-reduced-motion: reduce`, all transitions are 0 ms and the progress line is static.

### 12.7 Imagery

None in v1. The OG images (`/og/home.png`, `/og/share.png`, 1200 × 630, under 300 KB) are flat and typographic: `--paper` background, the Kitsheet wordmark in Archivo 800, one line in Archivo 400 ("Brand kit from your website or files." / "Brand kit. Open the link to view. Made by a visitor; not verified by any brand."), one accent rule. If photography is ever added, it is the owner's own work with an editorial film look (natural grain, muted color, real scenes), never stock, and any identifiable person needs a model release (O-6).

### 12.8 Generated PDF and PNG templates

The same proof-sheet language: white page, the asymmetric 3 + 9 column split, the brand name hanging into the page margin on the cover and the board (the same gesture as on screen, §12.1), labels in mono, hairline rules, square swatches, no shadows, no gradients except a brand's own gradient logo. The tool's accent red is absent from every export; the Kitsheet name appears only in the footer provenance line and the colophon. The kit's own colors and fonts carry all the visual weight. Details in §3.2 and §3.3.

### 12.9 Copy rules

- Direct and specific; say what happens and what to do next. Example: "Live site reading is paused until 6:00 pm your time." Not: "Something might have gone wrong; please try again later."
- In-tool copy uses "Kitsheet" as the subject, never "we", "our" or "us" (the legal pages use "I" and "me", §9.0). Examples: "Kitsheet doesn't store your uploads, inputs or kits." / "…so Kitsheet didn't read it." / "Kitsheet's server hit an error."
- Never: empower, seamless, unlock, elevate, leverage; "AI"; invented numbers. Never use "official", "approved" or "compliant" as a claim about a kit or about Kitsheet; they may appear only as negations in disclaimers (share banner, share checkbox, Acceptable Use). Counts shown to the user are real counts from the current run.
- `scripts/check-copy.mjs` (CI) greps the built HTML and the copy strings for the banned words and, case-sensitively, for `we`, `We`, `our`, `Our`, `us`, `Us` as whole words in in-tool copy (`US` and `USA` are not matched); the legal pages are exempt from the we/our/us rule only.
- Home page has no feature-card row. Its only copy is the `<h1>` line, the one-sentence product line (§2.3) and the in-tool notices.
- Marketing copy (README, OG text, portfolio case study) talks about the user's own brand: "Build a brand kit from your website or files", never "grab any company's brand" (M-1).
- Demos and screenshots use only the owner's brands or fictional brands (L-5).

### 12.10 Name: five candidates and a recommendation

A web search on 2026-11-08 checked each candidate for software products with the same name. That is not a trademark clearance; before launch the owner should run a free USPTO search at tmsearch.uspto.gov (L-6) [Guessing on clearance for all candidates].

| Candidate | Fit | Collisions found in the web search |
|---|---|---|
| **Kitsheet** | Says what it makes (a brand kit, as a sheet); plain for a roofer and for a marketer; matches the proof-sheet design idea; descriptive enough to help search | None as a software product. "Brand kit sheet" is a common phrase on Etsy and Pinterest, so the mark is weak but the meaning is instantly clear. |
| Stepwedge | A photographic step wedge is a stepped tonal scale, like a tint ramp; ties to the owner's photography | None in software. Obscure to most of the audience. |
| Liftsheet | "Lift" the brand off a site onto a sheet | None in software; a common medical-supply term. |
| House Style | The editorial term for a style guide | Generic phrase used widely; very weak as a name. |
| Cropmark | A print-production mark; fits the crop-mark ornament | Two screenshot apps already use it (Windows and macOS). |

Rejected during the search because of direct collisions: Proofsheet (a wedding-photographer website template of the same name, the owner's other industry, plus a social-media planner product), Brandsheet (a brand-asset product), Swatchbook (a materials and color platform), PressCheck and ProofRun (print-proofing software), KitProof (a manufacturing product), Keyplate (a design tool), Hardproof and Offprint (existing companies), Grease Pencil (a Blender feature).

**Recommendation: Kitsheet.** It is the clearest name for a mixed audience, it is free of same-space collisions in the search, it matches the anchor design idea, and its robots.txt token is valid as is. Host: `kitsheet.wethejamesphoto.com`.

### 12.11 Home metadata

- `<title>Kitsheet: a brand kit from your website or files</title>`.
- `<meta name="description">` in the product line's own words: "Paste your website. Get your logo, colors and fonts as a PDF guide, a PNG board and a link you can share."
- `<link rel="canonical" href="https://<host>/">`.
- The `<h1>` is the empty-state headline in §2.3 ("Brand kit from your website or files."); the product line stays as item 1 in the margin.
- `WebApplication` JSON-LD: `name`, `url`, `applicationCategory` "DesignApplication", `isAccessibleForFree` true, and `creator` a `Person` "Justin James" with `url` `PORTFOLIO_URL`. No ratings, offers or user counts (never-invent rule). The JSON-LD block is a static `<script type="application/ld+json">`, which is data, not script, and needs no CSP change.
- `sitemap.xml` lists `/` and the static pages (never `/k/`).
- OG tags use `/og/home.png` (§12.7).


---

## 13. Accessibility plan (WCAG 2.2 AA)

Target: WCAG 2.2 Level AA ([wcag22]) for `/`, `/k/` and every static page [Certain that these are WCAG 2.2 criteria].

### 13.1 Criteria that shape this UI

| Criterion | Implementation | Test |
|---|---|---|
| 1.1.1 Non-text content | Logo candidates: alt "{name} logo option {n}: {reason}"; decorative crop marks `aria-hidden`; swatches have text labels; a logo hidden by S-LOGO is announced by its text line | axe; screen-reader pass |
| 1.3.1 Info and relationships | Real headings per sheet section; contrast grid is a `<table>` with `<th scope>`; step list is an `<ol>`; form fields have `<label>` | axe; manual |
| 1.4.1 Use of color | Every swatch prints its hex; pass/fail badges are words (AAA, AA, AA Large, Fails); errors start with "Error:" | Manual; grayscale screenshot review |
| 1.4.3 / 1.4.11 Contrast | Tokens in §12.3 meet 4.5:1 for text and 3:1 for control borders and focus rings in both themes | Unit test over the token table |
| 1.4.4 / 1.4.10 Resize and reflow | Layout reflows to one column at 320 CSS px; the headline is fitted by `fitHeadline` (§12.2), so a long single word breaks instead of overflowing; no horizontal scroll except the contrast grid, which scrolls inside a labeled region | Playwright at 320 px and at 400% zoom with the §12.2 fixtures |
| 1.4.12 Text spacing | No fixed-height text containers | Bookmarklet test |
| 2.1.1 Keyboard | Everything operable by keyboard, including reorder (Move up/Move down buttons), file pickers, font picker, dialogs | Keyboard-only e2e |
| 2.4.3 / 2.4.7 Focus order and visible focus | Logical DOM order (Cover block, margin column, rest of sheet, §12.4); 2 px focus ring (accent in the margin, `--sheet-ink` with white inset on the sheet) | Manual |
| 2.4.11 Focus not obscured (minimum) | Sticky export panel and bottom bar reserve `scroll-padding` so focused items are never hidden | Manual at mobile sizes |
| 2.5.7 Dragging movements | No drag-only action: file buttons beside drop zones; reorder buttons; typed hex beside the color picker | Manual |
| 2.5.8 Target size (minimum) | All targets ≥ 24 × 24 CSS px; primary actions 44 px | Automated size check in e2e |
| 3.3.1 / 3.3.3 Errors and suggestions | Every error says what happened and what to do (§2.10) and is tied to its field with `aria-describedby` | e2e asserts copy and wiring |
| 4.1.3 Status messages | One `aria-live="polite"` region for step changes, "Done …" summaries, "Link copied", "Cleared"; errors use `role="alert"` once | Screen-reader pass |
| 3.1.1 Language | `<html lang="en">`; kit text in other scripts rendered with `dir="auto"` | axe |

### 13.2 Color picker accessibility

- The **hex text field is the primary control**; the native `<input type="color">` is a secondary convenience (its keyboard and screen-reader behavior varies by browser).
- Each color row has a programmatic name: "Color 2, secondary, hex 1b2a41".
- Changing a color announces the result once: "Secondary set to #1b2a41. 12.10:1 on background, passes AAA."
- Role changes that move another color announce it: "Primary moved to other."
- Swatches keep `forced-color-adjust: none` so they show real colors in Windows High Contrast; their labels follow system colors.

### 13.3 Font picker accessibility

ARIA combobox with a listbox per the APG pattern: `role="combobox"`, `aria-expanded`, `aria-controls`, `aria-activedescendant`; arrow keys move; Enter selects; Escape closes; type-ahead filters; the category filter is a labeled `<select>`. Virtualization keeps `aria-setsize` and `aria-posinset` correct. Each option's accessible name is the plain family name plus category ("Playfair Display, serif"), never dependent on the preview font loading.

### 13.4 File upload accessibility

Visible buttons wrap real `<input type="file">` elements with explicit `accept` lists and labels ("Upload logo: SVG up to 1 MB, or PNG or JPG up to 5 MB"). Drop zones are an extra and announce "Drop to add your logo" only to pointer users. On selection: live-region announcement of the file name, then the step list; on rejection: inline error with the accepted types and limits, focus returned to the button.

### 13.5 Other

- Skip link to the main content. Landmarks: `header`, `nav` (footer links), `main`, `aside` for the margin column, `footer`.
- After a read finishes, focus moves to the sheet's "Results" heading and the summary is announced.
- Turnstile: interaction-only appearance; if a challenge appears, focus moves to it and the margin says "Complete Cloudflare's check to continue." If it fails, uploads remain (E-HUMAN).
- Share view: same semantics as the review screen, read-only; the banner is the first content after the skip link.
- Generated PDF: tagged, titled, language set, real text, logical order (§7.1.2). Not claimed PDF/UA (C18). PNG: the download control says the PDF and link carry the same content as text.
- Testing cadence: axe (`@axe-core/playwright` 4.14.0) on every route and every §2.10 state in CI; manual passes before launch and after major UI changes, on devices the owner already owns plus free emulators (§15.8): keyboard only; NVDA + Firefox if a Windows PC is available; VoiceOver + Safari (macOS and iOS) if owned; TalkBack + Chrome (Android Emulator); 200% and 400% zoom; Windows forced colors (emulated in Playwright with `forcedColors: 'active'`, plus a real Windows check if available); `prefers-reduced-motion`. `/accessibility` lists exactly the combinations that were run.

---

## 14. Analytics

**Nothing runs in the visitor's browser for measurement.** No JavaScript beacon (including Cloudflare Web Analytics), no cookies, no pixels, no session replay, no error-reporting service (C14, PRIV-09) [Likely that a beacon counts as "access" under EDPB 2/2023 ([edpb-2-2023])].

### 14.1 What is measured (all outside the visitor's browser, all aggregate)

| Source | Shows | Where |
|---|---|---|
| Cloudflare Pages Functions metrics | Daily Function requests to `/api/extract` and `/api/images` (website reads plus any rejected, automated or junk requests; not equal to website reads) and errors; CPU p50/p99 and Exceeded Resources | Cloudflare dashboard |
| Turnstile analytics | Challenges issued and solved, 7-day lookback ([ts-plans]). Challenges solved approximate real website reads | Cloudflare dashboard |
| Google Search Console | Search impressions and clicks for `<host>`; Security Issues | Google |
| GitHub | Repo traffic (views, clones, referrers; 14 days), stars, issues | GitHub |
| Owner's portfolio analytics (optional) | Clicks from the footer link, if the link carries `?ref=kitsheet` | The portfolio site's own analytics; nothing is added to Kitsheet |

The footer's portfolio link uses `rel="noopener noreferrer"`, so the query parameter is the only signal. It identifies the source site, not the person.

### 14.2 Never measured

URLs or domains people read; kit contents (names, taglines, colors, fonts, logos); uploads; share links created or opened; which buttons people press, funnels, time on page, scroll depth; IP addresses, device fingerprints, user agents or locations (beyond what Cloudflare itself processes to deliver the site, §9.1); client-side errors with content. Bug reports come only from people who choose to email or open a GitHub issue.

### 14.3 Changing this

Adding any client-side measurement, even cookieless, is a re-review trigger (§17.3). If the EU Digital Omnibus proposal's consent-free first-party audience measurement becomes law, revisit then [Likely that it was still a proposal as of October 2026].

---

## 15. Testing and QA

### 15.1 Unit tests (Vitest 5.0.5, Node)

Pure modules: `color-core` (parse, gamut map, ΔE, WCAG contrast including `#777777` on `#ffffff` = 4.478, fails), `color-roles` (scoring and slot rules on recorded evidence), `systems/ramps` (hue stability, anchor replacement, chroma cap), `systems/type-scale`, `font-detect` (stack cleaning, next/font decoding, source labels, Google URL parsing), `logo-rank`, `identity`, `kit-model` merge precedence, `palette/kmeans` (exact medoids on synthetic images; the MMCQ counter-example), `pdf-guide/patterns` (HEX/RGB/CMYK/Pantone regexes with true and false cases), `ui/fit-headline` (§12.2 fixtures), `css-url-rewrite` (comments, strings, escapes, `@import`), `css-prepass` (caps).

- **`svg/canonical-grammar`** (pure, Node): `validateCanonical(x)` is either `x` or `null`; it rejects each grammar violation, every malicious SVG fixture, and broken or wrongly typed `url(#id)` references. Mutation and fuzz tests over canonical strings, each of which must give `null` (and therefore S-LOGO on `/k/`): single-byte flips, reordered attributes, inserted whitespace or comments, `&#…;` and `&name;` references, `<style>`, `style=""`, `href`, an unknown attribute, a dangling `url(#x)`, a duplicate id, an over-cap depth or element count. Property test (Playwright, because `canonicalize` needs the probe): for every SVG in `tests/fixtures` and the logo corpus, `parseCanonical(canonicalize(x))` succeeds and round-trips byte for byte, which proves the checker accepts everything the app outputs; `canonicalize(canonicalize(x)) = canonicalize(x)`.
- **`svg/code-detect`**: the §15.7 scannable-code positives and negatives, plus a seeded fuzz job.
- **`kit-model/contact`**: the §15.7 must-reject and must-accept tables.
- **`kit-model/match-key`**: Cyrillic and Greek lookalikes, I/l/1 and O/0 swaps, fullwidth letters, mixed-script words, edit distance 1.
- **`outputs/share`**: encode/decode round trip, canonical JSON, D1, caps, every S-state; one flipped payload byte with a valid `len` gives S-CUT; a 6,000-character decompression bomb gives S-INVALID, not a truncated parse; `l.k` filtering.
- **Server, handler level** (§4.9): Vitest calls `createExtractHandler(PROD_CONFIG, stubDeps)` and `createImageHandler(…)` with Host, Origin and siteverify stubs that pass, so every §15.7 SSRF and open-proxy case reaches the check it targets. Each case asserts its specific outcome (`422 blocked_url`, `{ok:false, reason:'dns_error'}`, `too_many_redirects`, `403 human_check_failed`, and so on), never just "not 200". Separate cases cover each precondition failing on its own (wrong Host gives 404, wrong Origin gives 403, wrong action, reused token, `challenge_ts` older than 300 s, a test-secret prefix gives 500, the documented test-secret siteverify response gives 403). Fixture hosts are subdomains of the RFC 2606 names `example.com`, `example.net` and `example.org` (for example `acme-roofing.example.com`). The stub resolver answers with a fixed public address outside every §10.2 range (not TEST-NET, which the guard blocks); DoH cases get per-case private answers; redirect cases chain stub 3xx responses through the same transport. `budget`: the 46th request is never issued, and a test fails if any `fetch(` bypasses `budget.take`. Also `server/robots` (RFC 9309 examples), `server/ticket` (prefix binding, 60 s expiry, single use), `server/frame`.

### 15.2 Synthetic fixtures (public, in repo, run in Playwright on Chromium, Firefox and WebKit)

About 35 mini-sites we write ourselves, each isolating one pattern, each with `expected.json`:
- Tailwind v3 opacity variables; Tailwind v4 oklch theme; v4 custom `--color-brand`.
- shadcn v3 HSL triplets; shadcn v4 oklch with `.dark`.
- WordPress presets with defaults only; block theme `primary`; Elementor kit scoped to body; Astra; Squarespace comma-HSL and `--sqsp-palette-*`; Wix `--wst-*`; Shopify color schemes; Webflow `WebFont.load`; next/font hashed names.
- Font loading: Typekit link; Bunny; Fontshare; Google css2 multi-family; Next `style[data-href]`; icon font first in stack.
- Distractors: cookie banner in a saturated color; UA-defaults-only page.
- Environment: visitor OS dark mode; forced colors.
- Syntax: `color-mix`, relative color, `light-dark`, `@layer`, nesting, undefined var, gradient brand, `@import` with `layer()` and media.
- Page shapes: SPA shell; 403 and challenge page; `meta refresh`; non-UTF-8 (windows-1252, Shift_JIS) page.
- Logos: inline SVG sprite; CSS background logo; header badge wall; JSON-LD-only logo; og:image banner only; triple-agreement logo with `?format=` variants.

A local stub of the Function API serves fixture bytes as KITX frames. Pass criteria: role hex values exact, families exact, labels exact, logo pick exact, and byte-identical kit JSON across the three engines (except values marked engine-dependent in `expected.json`, which must still never be invented) and across `ubuntu-latest`, `windows-latest` and `macos-latest` runners (§5.11). Fixture additions: a sheet at `/assets/css/site.css` with `.logo{background:url(../img/logo.svg)}` and `@font-face{src:url(../fonts/x.woff2)}` on a CDN host; assert the candidate matches the ticket entry, the label is "served by {CDN}", and the probe makes zero requests.

One Playwright job runs the real client against `npm run dev:snapshots` (§4.9) on synthetic fixtures. It covers the happy path plus `robots_disallowed`, `site_blocked` and `not_html`, so the client, the real handler and KITX framing are tested end to end. The other states stay on the API stub.

### 15.3 Real-site corpus (local only; accuracy targets)

- 30 public sites: Squarespace 7.1 × 4 (including wethejamesphoto.com), Wix × 4 (one Wix Studio), WordPress × 6 (2 Elementor, 1 Astra, 1 Kadence or Blocksy, 1 block theme, 1 Divi), Shopify × 3, Webflow × 3, Tailwind/Next.js × 4 (v3, v4, shadcn-oklch, next/font; one pure client-rendered SPA as the expected-partial case), Framer × 1, GoDaddy or Duda × 2, small DFW home-service sites × 3 (roofing, junk removal, plumbing; at least one with a badge wall).
- **Snapshots are never committed** (third-party HTML and CSS are copyrightable code; EXT-60). The owner captures them manually into the gitignored `tests/corpus/snapshots/`, refreshed on demand, never on a schedule (a scheduled crawl would break the user-triggered posture, F-2). Only `tests/corpus/manifest.json` (URLs) and `expected/*.json` (facts: hex values, family names, labels, logo URL patterns, names) are committed. A site owner who asks to be removed is removed.
- Ground truth: labeled by the owner (a designer) from the live site and any public brand materials: allowed hex per role (with alternates), `mustNotInclude` colors, heading and body family, expected source label or "none", logo URL pattern, name, allowed taglines or null, `expectPartial`.
- Harness: Playwright runs the real client code against snapshot bytes through the local API stub. `npm run dev:snapshots` runs the real handler in workerd against snapshot bytes (§4.9); per-request CPU is profiled with DevTools (`wrangler dev`, press D) and p99 recorded (indicative only; production metrics gate at M10). `npm run corpus` prints the scores table.
- **Competitor comparison** (G1, M9, once): the owner runs the same 30 URLs through the competitor by hand as an ordinary user, after checking its terms allow it (if they forbid benchmarking, skip this and record G1 against the absolute targets only), and records its primary, background and text hex, heading and body family and chosen logo in `tests/corpus/competitor.json`: facts only, never snapshots, and the file does not name the competitor. Launch gate: Kitsheet ≥ competitor on primary top-1, heading family, body family and logo top-1, with 0 invented colors. Every measure where Kitsheet trails is listed in the committed corpus report.

**Launch targets (EXT-61; targets, not measurements) [Guessing on values]:**

| Measure | Target |
|---|---|
| Invented colors | **0** (hard gate) |
| Background and text | ≥ 28/30 exact (ΔE_OK ≤ 0.02) |
| Primary | top-1 ≥ 24/30; top-3 ≥ 28/30 |
| Heading family; body family | ≥ 27/30 each |
| Font-source label precision | ≥ 95% (a wrong label fails; "not detected" is fine) |
| Logo | top-1 ≥ 25/30; top-3 ≥ 28/30 |
| Favicon | ≥ 29/30 |
| Brand name | ≥ 27/30 |
| JavaScript-shell and blocked sites flagged | 100% |
| Repeat runs | 10 runs byte-identical |
| Cross-engine role hex | ≥ 28/30 identical |
| Client extraction time | p95 ≤ 2.5 s at 4× CPU throttle |
| Function CPU | p99 ≤ 8 ms |

Also record (no target): share of corpus sites that came back partial or blocked; this is the first real number for how often static reading fails for this audience. It is first recorded at the M4 exit gate (§16), not M9. Also record how many chosen logos the §5.2.4 image-host restriction dropped.

### 15.4 Brand-guide PDF and upload corpus (local only)

10 real brand-guide PDFs the owner has rights to use or that are publicly published by their owners (kept out of the repo), plus synthetic PDFs we generate: text HEX/RGB/CMYK/Pantone; outlined text; scanned (image-only); CMYK-only; encrypted; 40+ pages. Targets [Guessing]: ≥ 90% of printed HEX codes found; 0 HEX codes invented; every HEX/RGB mismatch flagged. Logos: 20 SVGs (Illustrator, Figma, Inkscape, SVGO-minified, gradient, clipPath, live text, embedded image) and 10 PNG/JPG logos.

### 15.5 End-to-end (Playwright 1.64.1; Chromium, Firefox, WebKit; desktop and mobile viewports)

Every flow in §2 and every state in §2.10 (server states via the API stub), including: URL read happy path; cancel; partial; JS shell; each blocked/robots/opt-out/error state, including E-UPSTREAM, E-TLS and E-REDIRECTS; capacity for each fail-open response shape (static 404 page, 405, HTML fall-through, "Error 1027" body), asserting one retry and that a single capacity response never locks the day; E-HEAVY for 1101 and 1102 bodies and other 5xx without `X-Kit`, asserting no retry and no lockout; E-OLD by deleting `String.prototype.isWellFormed`; E-PDFOLD (not E-PDF) by deleting `Promise.withResolvers`; E-EXPORT for PDF and PNG; E-COPY; E-NOCOLOR; E-CONTACT; S-LOGO for both causes; the Terms notices present at body size under Read site, the file buttons and the export panel; logo SVG and raster uploads; image palette; PDF guide; manual-only kit; combining sources and locks; PDF download (assert file name, page count, fonts embedded and subset with `pdffonts` from poppler-utils installed in CI, hex values present in extracted text, title and language set, a non-empty structure tree in `pdfinfo -struct-text` whose text matches the kit); PDF script fixtures: Vietnamese "Phở Hòa", Greek, Cyrillic and Japanese names (assert `pdftotext` contains the exact NFC input and `pdffonts` lists the expected fallback face), multi-word Arabic and Hebrew names that include a short number (compare a `pdftoppm` raster against a reviewed golden PNG and assert each logical word appears in `pdftotext` output; exact string equality is fragile for right-to-left extraction [Guessing]), Devanagari and Thai raster goldens before claiming support, and a Lato kit (mirror files byte-identical to the source, no `menu` file); PNG download (3200 × 2000, sRGB); share link copy, open, edit a copy, every S-state, `%23` path repair; on `/k/` in Chromium, Firefox and WebKit, golden links with no logo, a single-path mark, gradient plus `clipPath`, and an exactly 8,192-byte SVG: a `securitypolicyviolation` listener registered before load sees zero events, the logo `<img>` has `naturalWidth > 0`, and PDF and PNG export from `/k/` with a logo succeed with zero violations; Clear my data (database deleted, `bk.*` keys gone; `/clear`'s **Clear now** does the same and the page sends no `Clear-Site-Data`); no `bk` IndexedDB after any flow unless **Keep this draft** was ticked; kept-draft restore after reload; session restore after reload without Keep; **Don't remember my settings** deletes `bk.theme` and `bk.pageSize`; E-STORAGE under a storage-disabled context; keyboard-only run of the main flows; axe on each state.

### 15.6 Privacy-claim tests (run on every PR; against production monthly and at M10)

- **Network allowlist** (PRIV-28): record every request from page load through a URL read, each upload type, PDF and PNG export, share creation and the share view. Fail on any host other than `<host>` and, during a URL read only, `challenges.cloudflare.com`. Fail if any request body contains bytes of an uploaded file (compare hashes of request bodies against the file).
- **No cookies**: after every flow, read `context.cookies()` with no URL filter, in Chromium with third-party cookies allowed (partitioned cookies included).
  - Fail on any cookie for `<host>`.
  - Fail on any cookie for any other domain, except: after a URL read only, cookies on `challenges.cloudflare.com` whose names appear on the allowlist in `docs/compliance/data-flow.md`, with expiry no longer than the lifetime recorded there.
  - After a URL read, for each frame whose host is `challenges.cloudflare.com`, run `frame.evaluate` to list `localStorage` and `sessionStorage` keys and `indexedDB.databases()`, each wrapped in try/catch. Record them, and fail on any key not on the allowlist.
  - PR runs use Turnstile test keys, so the third-party part is authoritative only in the production runs. Production URL reads are done by hand (no production bypass, §10.4): the owner checks in a clean Chrome profile and records the date, browser and results (§11.5 #7).
- **Storage keys**: no `bk` IndexedDB unless **Keep this draft** was ticked; `bk.theme`/`bk.pageSize` only after the user changed them and never after **Don't remember my settings**; `bk.session` in sessionStorage only.
- **Fragment never leaves the page**: on `/k/`, no request URL, body or `Referer` contains any part of the payload.
- **Title and metadata**: `/k/` `document.title` and OG tags never contain kit text.
- **Claims map**: `docs/compliance/claims-map.md` lists each published claim with the test above that proves it; CI fails if a claim lists no test.

### 15.7 Security tests

| Area | Cases (each must be rejected or neutralized) |
|---|---|
| SSRF (`/api/extract` and redirects) | `http://localhost`, `127.0.0.1`, `0x7f.1`, `2130706433`, `[::1]`, `[::ffff:127.0.0.1]`, `169.254.169.254`, `metadata.google.internal`, `10.0.0.1.nip.io`, `*.sslip.io`, `*.local`, `*.internal`, port 8080, `file:`, `data:`, `javascript:`, `ftp:`, `user:pass@host`, the tool's own host, `*.pages.dev`, a public host redirecting to each of the above, a host whose DNS A record is private (DoH check), > 5 redirects, 2 MB+1 body without `Content-Length`, wrong content types |
| Open proxy | GET; missing or wrong `Origin`; `text/plain` body; missing, reused, stale (`challenge_ts` > 300 s) or wrong-action Turnstile token; expired ticket; tampered ticket; a ticket presented from another /64 or /32; a second `/api/images` call with the same nonce; URL not in ticket; candidate on a host outside the page's hosts, CSS hosts and CDN list; preview hostname; GET and POST to `/api/other` return a static response without `X-Kit: 1`; response never `text/html`/`image/svg+xml` and always has `nosniff` and `attachment` |
| Share-link XSS and abuse | `<script>` and `<img onerror>` in name and tagline; bidi controls and tag characters; URLs, emails and phone numbers in text (the contact table below); CSS injection in hex and font names (`red;}body{…`, `x"></style>`); `__proto__` and `constructor` keys; SVG with `<script>`, `onload`, `<foreignObject>`, external `<use>`/`<image>`, `<style>@import`, `javascript:` and `data:text/html` hrefs, `<animate>` setting `href`; an SVG that fails `validateCanonical` (logo hidden, rest renders); entity bomb; decompression bomb (inflates past 32 KB → S-INVALID); one flipped payload byte with a valid `len` (→ S-CUT via our Adler-32 check); 8,193-character fragment; truncated payload; wrong `len`; unknown version; blocklisted hash and name; watchlist name; homoglyph names ("Раураl", "PayPaI", "Chаse", fullwidth letters, O/0 swaps) reaching S-WATCH. Assert no script runs (CSP violation listener and a global canary) and the DOM contains only text nodes for kit text |
| Contact strings (`kit-model/contact`) | Must reject, in name, tagline, color name or custom font name: "469-555-0123", "(800) 555 0199", "+1 888.555.0199", "469–555–0123" (en dashes), "８８８-５５５-０１９９" (full-width), "(214) 555-0199", "214.555.0199", "+1 214 555 0199", "２１４５５５０１９９", "020 7946 0958", "+44 20 7946 0958", "01 23 45 67 89", "Call 555-0199", "acme.com", "ACME.COM", "acme[.]com", "acme(dot)com", "www.acme", "bob＠acme．com". Must accept: "Est. 1998", "Est. 1998 - 2024", "Est. 1998-2024", "24/7 service", "Since 1985", "Voted best 2023 2024 2025", "Over 1,000,000 roofs", "Serving 214, 469 and 972", "Rowlett, TX 75088", "St. Louis Roofing", "Node.js Studio", "Polka Dot Design Co", "No. 1 in Dallas", "A+ rated, 5-star reviews". A share link carrying "Call 1-888-555-0199" in `c[0][2]` and in `h[0]` decodes to S-INVALID. Extracted candidates with contact strings are skipped silently, never shown as an error |
| Scannable codes in logos (§6.2.6) | Positives: QR versions 1, 2 and 5 drawn as one merged path; rotated 17° and 45°; sheared; inverted; dot-style data with square or round finder eyes; a code at 1/6 the width of a wordmark; a code visible only in the on-dark variant (paints `#000000` and `#777777`); a code with a broken timing pattern; Data Matrix; Aztec; Micro QR. Every positive must first decode with a dev-only oracle decoder (for example jsQR, Apache-2.0, or a zxing-cpp wasm build), never shipped. A seeded fuzz job (random content, version, transform, polarity, style): every fixture the oracle decodes must be flagged. Negatives: no logo in the §15.3 corpus and none of the simple-icons SVGs (3,470 in 16.37.0) may be flagged; triage any hit before launch (the prototype flagged 1 of 3,470). On `/k/`, a flagged logo is hidden before render, export and Edit a copy |
| Malicious SVG uploads | > 1 MB; > 10,000 elements; DOCTYPE/ENTITY; deeply nested `<use>` recursion; huge path data; script and handler variants as above; `fill:u\72l(https://x)` and other CSS escapes in `<style>` and `style=""`; a `Blob` constructor wrapper asserting no `image/svg+xml` Blob is ever built from a non-canonical string (except the raster-only fallback draw) |
| Fetched images and SVG | A fetched SVG favicon and logo carrying `<script>` (must go through §6.2 and never become a raw SVG Blob); a 1 MB PNG declaring 30,000 × 30,000 fetched through the API stub (refused before decode) |
| Malicious PDFs | CVE-2024-4367 FontMatrix payload; JavaScript actions; XFA form; embedded files; 500 pages; zip-bomb streams; malformed xref; a parse-loop xref; encrypted. Run against the legacy pdf.js files. Assert no script, no network, timeouts honored, and our Worker terminated (not only `destroy()`) |
| Malicious images | Header claiming 50,000 × 50,000; truncated PNG; polyglot (PNG header with HTML body); GIF with many frames |
| Fetched HTML and CSS in the probe | Inline scripts and handlers; `meta refresh`; `<base href>`; external `<link>`, `@import`, `@font-face`, `background-image`; relative `url()` values (resolved against the sheet, never Kitsheet's origin); CSS designed to exfiltrate attribute values via `url()`; a CSS `:has()` bomb; a `var()` doubling chain. Assert zero requests leave the probe, no navigation occurs, and the tab stays responsive (the watchdog or the pre-pass ends the work) |
| Headers and routing | The §10.6 header test: 404 mode, flat-file canonical URLs, header and meta CSP on every HTML response, `/clear` with no redirect and no `Clear-Site-Data`, `/k/%231.…` with the `/k/*` block, `/%6B/` and `/%70rivacy` still carrying a meta CSP, `/api/anything-else` not invoking the Function; CSP violations in e2e runs fail the build |
| Bundles and pipeline | `check-no-eval.mjs`; license allowlist; budgets; `check-release-age.mjs`; `check-fn-bundle.mjs`; zizmor and the workflow grep; `check-font-tar.py` against a tarball with a symlink, a `..` path, an unexpected file name and a `_worker.js` (each must fail) |

### 15.8 Cross-browser and devices

- CI: Playwright Chromium, Firefox and WebKit on every PR.
- No device purchases and no paid device-cloud subscriptions (§11.4 #13). Before launch and each release with UI changes, run on devices the owner already owns plus free emulators: the Android Emulator (Chrome, TalkBack, and Samsung Internet if it installs there); the Xcode Simulator, including an iOS 16.4 runtime and an iOS 18 runtime, if a Mac is owned; Playwright `forcedColors` emulation. A free open-source device-cloud plan may be used only if the owner approves it [Guessing on availability]. Anything not run is recorded as "not tested" in the §15.8 record and on `/accessibility`; nothing untested is claimed. Until `LAUNCHED` flips (M10), device runs use a local build reached over the LAN (`vite preview --host`) for upload and export flows; the URL flow on a real device is checked at M10 after the flip and before any announcement.
- Device checklist (each where a device or emulator exists): iPhone with current iOS Safari; iOS 16.4 (the floor): run every upload flow, and the PDF button shows E-PDFOLD; iOS 18: read a real InDesign-exported (CFF-font) brand guide end to end; iPad Safari; a mid-range Android phone with Chrome; Samsung Internet, including the real-device PDF check; macOS Safari; Windows Chrome and Edge (with forced colors on); Firefox on desktop. Check: PDF download and the Share/Save path on iOS and Android; `blob:` PDF opening inline vs saving on iOS; PNG canvas memory on iPhone; a 40 MP JPEG and a 1,200 px SVG at 4× on a real iPhone; font rendering of the probe results vs Chromium; ICO and AVIF decoding; Instagram and LinkedIn in-app browsers show E-INAPP.
- Engine-dependent items from the research to confirm in Firefox and WebKit: CSSOM `mediaText` rewriting; computed color serialization; computed `transform` vs `transform.baseVal` (§6.2.2); `forced-color-adjust`; `srcdoc` CSP inheritance; computed styles in an off-screen iframe; `createImageBitmap` options including `resizeQuality: 'pixelated'`; `FontFace(ArrayBuffer)` under `font-src 'self'` and inside the probe with no CSP report.
- Long-link delivery (manual, recorded in `docs/ops/`): send 450, 1,900, 2,900 and 5,800-character links through Gmail web and iOS Mail, Outlook with Safe Links on and off, iMessage, Slack, LinkedIn and X; record truncation and whether `#` survives. Adjust the §7.3.3 tier copy to what is observed.

### 15.9 Performance tests

Lighthouse CI on `/` and `/k/` with the mobile preset; Playwright traces at 4× CPU throttle for a corpus read, a 25 MB PDF, a 50 MP image (refused; the cap is 40 MP) and a 20 MP image (accepted), PDF generation and PNG generation. Budgets in §4.5 fail the build.

---

## 16. Milestones

**Launch gate.** Until M10, `LAUNCHED=false` (build constant). When it is false:
- the build emits only `/index.html` (product line and footer, as a static holding page), `_headers` and `robots.txt` (`Disallow: /`), with no app bundle, no `/k/` and no legal drafts;
- `deploy.yml` runs `wrangler pages deploy` from a staging directory that has no `functions/` folder, so no Function is uploaded [Likely]. As a fallback, `_routes.json` excludes `/api/*`; Cloudflare says exclude rules take priority over include rules;
- a post-deploy smoke check fails if `POST https://PROD_HOST/api/extract` returns `X-Kit: 1` or if `/k/` returns 200.

All M1-M9 verification runs locally or in CI (§4.9). Every milestone has a command-based or numeric done check.

| # | Scope | Done looks like | How to verify |
|---|---|---|---|
| **M0 Accounts, repo, DNS** | Dedicated Cloudflare account (2FA, no card); Pages project created by first Direct Upload; preview deployments off; Fail open; custom domain via one CNAME at Squarespace (steps in §4.8, copied to `docs/ops/dns-setup.md`); Turnstile widget for `PROD_HOST`; Kitsheet Gmail account with the §9.0 settings; Google Cloud project with no billing and a Fonts API key; GitHub repo settings (§4.8: read-only token, fork-PR approval, `production` environment, immutable releases); wrangler pinned as a devDependency; `.npmrc` cooldown; LICENSE, NOTICE, README, CONTRIBUTING, SECURITY; Dependabot (with cooldown), CodeQL, secret scanning, zizmor; owner's monthly calendar event (§11.5); owner confirms the name (§12.10) after a USPTO search | `https://<host>/` serves the holding page over HTTPS; billing pages show no payment method; `deploy.yml` succeeds with `FONT_MIRROR_VERSION` empty | `dig +short CNAME <host>`; `curl -sI https://<host>`; screenshots of billing pages in `docs/compliance/logging.md`; Gmail settings screenshot; before adding the CNAME, re-check that wethejamesphoto.com has no CAA records (none on 2026-11-08 [Certain]); confirm `wrangler pages deploy` works after `npm ci --ignore-scripts` |
| **M1 Shell, design system, headers** | Vite multi-page with the §4.6 gate and `build.target`; Preact; tokens (incl. `--sheet-ink`); components; start screen with the `<h1>` headline and §12.11 metadata; footer; flat static page scaffolds and `404.html`; `csp.ts` → generated `_headers` and meta CSP; `_routes.json`, `_redirects`, `robots.txt`; Trusted Types policies; network-allowlist harness; `check-copy.mjs`. **Spike** (throwaway): PDFKit plus svg-to-pdfkit in a module worker, embedding one WOFF1 and drawing a gradient-plus-`clipPath` canonical SVG | Start screen matches §12 in both themes; the §10.6 header test passes against `wrangler pages dev dist`; the spike PDF opens and shows the gradient and clip | `npm run test:headers` green; axe clean; `npm run budgets` green; `docs/ops/design-review.md` all yes on the M1 screenshots (start, at 320, 768, 1280 and 1440 px, light and dark), owner approves |
| **M2 Font mirror, picker, manual kit, draft** | `font-mirror.yml` (build and publish jobs) → Release asset with SHA-256; deploy verifies and extracts with `check-font-tar.py`; `catalog.json` with license branches; Noto fallback faces; UI fonts from the mirror; font picker; manual colors, name, tagline with `containsContact()`; session restore, opt-in kept draft, **Don't remember my settings**; Clear my data and `/clear` | A complete manual kit survives a reload in the same tab, and across sessions only when Keep is ticked; picker previews load only from `<host>` | Network-allowlist test; file count < 19,000; mirror-build check green (RFN, UFL and CJK files byte-identical, no `menu` or `.woff` for them, `CHANGES.txt` on Apache subsets); the no-key `google/fonts` fallback runs once and produces a valid tree; `/credits` lists fonts |
| **M3 Fetch endpoints** | `server/handler.ts` factories, production and dev entries (§4.9), `/api/extract`, `/api/images`, SSRF guard, DoH, robots, do-not-fetch, Turnstile, prefix-bound ticket, limiter, `budget.take`, KITX frame, all failure codes; `/bot` page | Handler tests pass for every §15.7 SSRF and open-proxy case against `PROD_CONFIG`, with the asserted codes; `dev:live` reads 5 public corpus sites and returns well-formed KITX frames; `dev:snapshots` runs the corpus and records a CPU profile; nothing under `/api` is live in production | `npm test -- server`; `check-fn-bundle.mjs` passes; local CPU p99 recorded (≤ 6 ms or apply the step-down table); Turnstile under Trusted Types decision recorded; fail-open response shape recorded on a scratch project; post-deploy smoke check green |
| **M4 Extraction engine** | html-ingest with `docBase`, CSS pre-pass and URL rewrite, render probe (light, metric and dark passes, neutralization, sentinel resolver), colors, platform dictionary, fonts, logos, favicon, name, tagline, partial/JS/blocked detection, review screen for site sources | Synthetic fixtures pass on Chromium, Firefox and WebKit and on three OS runners; **exit gate**: the 30-site corpus (§15.3) runs on this engine | `npm run fixtures`; determinism (10 runs); never-invent property test; `npm run corpus` prints partial/blocked rate and top-1 for primary color, heading family and logo. If more than 8 of 30 sites come back partial or blocked, or primary top-1 is below 18 of 30, do the upload-first UX changes (§2.3, §2.10) before M7 |
| **M5 Uploads** | SVG pipeline (allowlist, probe paints, canonical tree and grammar, raster fallback, variants on the tree), scannable-code check, raster logo, image palette, PDF guide parser (legacy build, gate, terminate), favicon upload, source merging and locks | Each upload type produces a correct kit with zero requests carrying file bytes | Unit + e2e; grammar round-trip property test; scannable-code positives and negatives; malicious-file tests; PDF and logo corpora (§15.4) |
| **M6 Color and type systems; review complete** | Ramps, contrast grid, passing pairs, type scale, fallbacks, logo rules, `fitHeadline`, all §2.10 states, accessibility work | Review screen complete; every state reachable in tests | axe on every state; keyboard-only e2e; manual screen-reader pass on available devices; design-review checklist all yes on the M6 screenshots (working, review, E-PARTIAL) |
| **M7 Outputs** | PDF worker and templates (§3.2) with `withStruct` structure tree and glyph coverage/bidi (§7.1.7), PNG board (§3.3), downloads, Web Share, in-app detection, E-EXPORT | PDFs open in Acrobat, Preview, Chrome and pdf.js; PNG exact size; both match the review screen | `pdffonts`, text extraction, `pdfinfo -struct-text` non-empty and matching; veraPDF report attached; script fixtures (§15.5); visual diff of a fixed fixture kit against the design-review-approved references (PDF page 1 and the PNG); device checks |
| **M8 Share links** | Owner answers Q1 (`SHARE_ORIGIN`) first; codec with frozen D1 and our Adler-32 and overflow checks; ShareV1 schema with `l.k`; length tiers; Simplify via SVGO; `/k/` viewer with banner, pure-grammar validation, scannable-code check, match-key blocklist and watchlist; report flow, `/report`, Edit a copy, `%23` repair | Links round-trip; every S-state works; golden links made in three engines decode in all three and in Node | Golden links; share security tests; zero CSP violations on `/k/` (§15.5); privacy-claim tests on `/k/`; long-link delivery checks |
| **M9 Corpus tuning and hardening** | Real-site corpus; tune and freeze [Guessing] thresholds; competitor comparison (§15.3); cross-browser device pass; performance pass | §15.3 targets met and the competitor gate met | Corpus report committed (scores only, no snapshots); `competitor.json` committed; device record completed |
| **M10 Launch** | Final legal pages and in-tool notices; `docs/compliance/*` including `claims-map.md`; flip `LAUNCHED` to true; DevTools verification of cookies on every domain (Turnstile, Squarespace parent-domain cookies) and log retention; optional attorney review (§9.9); Search Console property; OG images; portfolio case study using only the owner's or fictional brands | Public at `https://<host>/` | Launch checklist: all CI green; §11.5 check passes; privacy tests pass against production, with the manual URL-read check recorded; the owner does one real read in a browser on `PROD_HOST` with the production keys; `https://<project>.pages.dev/api/extract` returns 404; the production project has no variables besides `TURNSTILE_SECRET` and `TICKET_HMAC_KEY`; production Functions CPU p99 ≤ 8 ms over the first reads (else the step-down table); real-device URL-flow checks before any announcement; owner signs off on copy and on the claims map |

---

## 17. Risks, open questions, and things that require re-review before adding

### 17.1 Risk register

| ID | Risk | Likelihood / impact | Mitigation |
|---|---|---|---|
| R1 | A phishing share link gets the owner's domain flagged by Safe Browsing | Low / high | §10.8 controls; contact-string filter (§6.5.5); scannable-code check (§6.2.6); Search Console monitoring; 2-business-day takedown; `SHARE_ORIGIN` switch to `*.pages.dev`. Residual: a URL or phone number drawn as outlines inside a logo can't be detected without OCR; the banner, the blocklist and the takedown cover it |
| R2 | A privacy claim becomes untrue after a change (a new script, font or hotlinked asset) | Medium / high | Network-allowlist and no-cookie tests on every PR and monthly in production; claims worded narrowly |
| R3 | Static reading fails on more sites than expected (JS shells, bot walls against an honest user agent) | Unknown / medium | Honest partial results; upload paths; measured in M9 before launch copy is final |
| R4 | `/api/extract` or `/api/images` exceeds 10 ms CPU on large pages | Unknown / medium | Local DevTools profiles of the real handler on all 30 corpus snapshots in M3 (indicative); production Functions metrics CPU p50/p99 and Exceeded Resources at M10 and monthly (the gate). Levers: byte-level CSS scans with ASCII case-folding, HTMLRewriter fed while the body downloads, the §5.2.3 step-down table (HTML 2 → 1 MB, CSS 3 → 2 MB, imports 4 → 2); per-image calls if `/api/images` is the problem. A single overrun shows E-HEAVY, never E-CAPACITY |
| R5 | Cloudflare deprecates Pages or changes free limits | Low / high | Pages is not announced as deprecated but feature work goes to Workers [Likely]; fallback is moving DNS to a Cloudflare zone and Workers static assets (risky for the owner's email) or another $0 host; watch the changelog |
| R6 | Quota exhausted by abuse or junk requests | Medium / medium | Accepted; no $0 control prevents it while DNS is at Squarespace (§10.3). Exact-path `_routes.json`; 404 mode; Fail open; graceful E-CAPACITY; upload paths unaffected; no cost. Impact is medium because the attack is cheap, can be repeated every day, and takes out the URL feature. If 1027 days recur (§11.5 #2), the owner decides whether to put a domain on a Cloudflare zone for WAF rate limiting (conflicts with §11.4 #8/#12; owner decision) |
| R7 | Font mirror too large or slow to deploy | Medium / low | Weight-limited mirror (400/700/400i) fallback; CI file-count gate |
| R8 | A dependency advisory (pdf.js, svg-to-pdfkit, fflate) | Medium / medium | Dependabot, OSV, canonical-only inputs to svg-to-pdfkit, pdf.js without eval or wasm |
| R9 | Turnstile does not work under Trusted Types on `/` | Medium / low | §10.7 fallback |
| R10 | Google requires billing for the Fonts API key | Low / low | Switch to the no-key path in §7.1.3 step 1 (`google/fonts` at a pinned commit, `METADATA.pb`, `varLib.instancer`), proven once in M2 |
| R11 | Domain lapses (expires 2027-08-24) | Low / high | Auto-renew on; monthly check |
| R12 | A user relies on contrast or font-license labels as legal advice | Medium / low | Disclaimers (E-4, E-5); Terms §2 |
| R13 | A site owner sends a cease-and-desist | Low / medium | Do-not-fetch within 2 business days; `/bot` page; honest UA |
| R14 | UK OSA or EU DSA turns out to apply | Low / medium | Paperwork already written (§8.3); fragment-only sharing stays permanent |
| R15 | Scheduled GitHub workflows silently disabled after 60 days of repo inactivity | High / low | §11.5 runs from the owner's calendar; step 0 re-enables workflows; the monthly Action reports workflow states; deploys pin the last mirror Release |
| R16 | The wedding domain lapses while share links live on it | Low / high | §0.3 #1; Q1 decided before M8; `SHARE_ORIGIN` on `*.pages.dev` removes the dependency; auto-renew and the monthly check |
| R17 | Cloudflare's controller reuse of Turnstile signals and its `cf_chl_*` cookies are found not to be strictly necessary in the EU/UK | Low / medium | Disclosed before the click and in the policy; §15.6 records every cookie; §9.9 item 10; swapping the bot check is a §17.3 trigger |
| R18 | A supply-chain compromise publishes attacker content on the owner's domain | Low / high | §10.12 cooldown, pinned wrangler, split build and deploy jobs, zizmor, immutable and hash-checked font releases (§7.1.3) |

### 17.2 Open questions for the owner

| # | Question | Default if unanswered |
|---|---|---|
| Q1 | **Needed before M8.** Keep the share viewer on `kitsheet.wethejamesphoto.com/k/`, or move only `/k/` to the project's `*.pages.dev` hostname? The `*.pages.dev` option is the $0-forever choice: it isolates the wedding domain from Safe Browsing risk, and links survive even if wethejamesphoto.com is ever dropped (§0.3 #1). `SHARE_ORIGIN` cannot change after launch. | Keep on the subdomain (locked hosting decision); Terms and §7.5.3 then say links work while Kitsheet is online at that host |
| Q2 | Confirm the name "Kitsheet" after a USPTO search, or pick another from §12.10. | Kitsheet |
| Q3 | Portfolio URL for "Built by Justin James"? Add `?ref=kitsheet`? | Needed before M10 |
| Q4 | Contact mailbox: which free Gmail address? | A free Gmail address used only for Kitsheet (for example `kitsheet.<something>@gmail.com`), with the §9.0 settings. Keep a Workspace alias instead only if the owner (a) accepts turning off auto-contacts on his business mailbox and (b) confirms that no CRM or inbox tool reads that mailbox; then policy item 15 uses the Workspace wording |
| Q5 | County for venue in the Terms: Dallas or Rockwall? | Dallas County |
| Q6 | Allow kits of brands the user doesn't represent for private reference (Acceptable Use), or ban them outright? | Allow for private reference |
| Q7 | Show a QR code for the no-logo link, and add link expiry, after launch? | No |
| Q8 | Which 10 brand-guide PDFs can be used for local testing? | Owner supplies |
| Q9 | Approve the substitute font map (§5.7.5). | Use as written |
| Q10 | Do the competitor's terms allow a one-time manual comparison on the 30 corpus URLs (§15.3)? | Check before M9; if not, skip the comparison |

**Verification items before launch (not owner decisions):** Squarespace parent-domain cookie scope (`ss_cvr`, `ss_cid` ([sq-cookies])) in DevTools on any domain; every cookie (any domain, partitioned included) and every storage key in the Turnstile frame after a URL read, with names and lifetimes recorded in `data-flow.md` and matched to privacy item 6; Cloudflare log retention actually configured; no-card signup; the fail-open response shape; HTMLRewriter CPU; Firefox and WebKit probe behavior; iOS PDF download behavior; Pages Direct Upload re-deploy skipping unchanged font files [Likely].

### 17.3 Changes that require a new review before they are built

Run `docs/compliance/change-triggers.md` for any of these. None is in v1.

| Change | What it triggers |
|---|---|
| **Any email capture** (newsletter, "email me my kit", waitlist, contact form) | CAN-SPAM and CASL; GDPR lawful basis and likely Art. 3(2) questions; CalOPPA and DOPPA clearly triggered; COPPA exposure and the owner's no-kids'-emails rule would need an age screen; Texas breach-notification duties (Bus. & Com. Code §521.053); a $0 email-sending service would be needed |
| **Phone or SMS fields, texting, calling** | TCPA consent rules; Texas SB 140 registration and private right of action; carrier (CTIA) rules |
| **Accounts or login** | Data-subject requests; security duties; actual-knowledge age issues; with profiles and social features, the Texas SCOPE Act |
| **Server storage** (short links, saved kits, galleries) | Hosting-provider status: EU DSA notice-and-action and a paid EU legal representative if EU-connected; UK OSA much more likely in scope; DMCA designated agent ($6 per 3 years); retention policy; breach exposure; the "never stored" promise ends |
| **Processing uploads on a server** | The operator becomes controller of upload contents (faces, names in guides); the "never leaves your browser" promise ends |
| **AI or LLM APIs** | New processors and international transfers; EU AI Act Art. 50 transparency; breaks the deterministic, "never invents" promise; likely paid APIs |
| **Paid tier, ads, affiliate links, lead gen** | PIPEDA "commercial activity"; LGPD economic purpose; possible EAA "e-commerce service"; larger ADA Title III target; possible Texas DBA filing; payment processing; breaks D4 |
| **Any analytics in the browser** (cookieless beacons included) | ePrivacy Art. 5(3) consent risk in the EU; UK statistical exception needs notice and an opt-out; if profiling, GDPR Art. 3(2)(b) monitoring and paid Art. 27 representatives |
| **Third-party fonts, scripts or images loaded in the browser** | IP disclosure to third parties (LG München I-type claims); breaks the privacy claims |
| **EU languages, EU/UK-targeted marketing, prices in EUR or GBP** | GDPR/UK GDPR Art. 3(2)(a) targeting; paid representatives; possibly an ICO fee |
| **Child-appealing design or school marketing** | COPPA "directed to children"; UK Children's Code; Brazil ECA Digital (plus a Brazilian representative) |
| **Swapping Turnstile for reCAPTCHA or hCaptcha** | A different controller with broader cookie and ad-linked use; EU consent questions; new disclosures |
| **Headless rendering (Browser Run)** | Running third-party JavaScript and trackers; F-1 policy change; 10 browser-minutes/day free cap |
| **Social features** (comments, public gallery, likes) | UK OSA clearly in scope; Texas SCOPE Act; Take It Down Act; DSA hosting duties |
| **Forming an LLC** | Mostly neutral; update the operator identity everywhere |
| **Letting wethejamesphoto.com lapse, or changing `SHARE_ORIGIN`** | If `/k/` is on that domain, issued links break and the fragments of old links reach the next registrant's JavaScript (§0.3 #1); move or retire share links first and update the Terms |
| **Very high traffic** (tens of thousands of residents of one small US state per year) | Re-check Montana, Delaware, New Hampshire, Rhode Island, Maryland, Alabama (from May 2027) and Vermont (from 2028) thresholds |

---

## Sources

Primary sources first within each group. Research reports also cite secondary sources inline; those carried into this spec are listed here.

**Cloudflare**
- `cf-limits`: <https://developers.cloudflare.com/workers/platform/limits/>
- `cf-pricing`: <https://developers.cloudflare.com/workers/platform/pricing/>
- `cf-pages-limits`: <https://developers.cloudflare.com/pages/platform/limits/>
- `cf-pages-fn-pricing`: <https://developers.cloudflare.com/pages/functions/pricing/>
- `cf-pages-routing`: <https://developers.cloudflare.com/pages/functions/routing/>
- `cf-pages-redirects`: <https://developers.cloudflare.com/pages/configuration/redirects/>
- `cf-pages-domains`: <https://developers.cloudflare.com/pages/configuration/custom-domains/>
- `cf-pages-debug`: <https://developers.cloudflare.com/pages/configuration/debugging-pages/>
- `cf-direct-upload`: <https://developers.cloudflare.com/pages/get-started/direct-upload/>
- `cf-migrate`: <https://developers.cloudflare.com/workers/static-assets/migration-guides/migrate-from-pages/>
- `cf-known-issues`: <https://developers.cloudflare.com/workers/platform/known-issues/>
- `cf-compat-flags`: <https://developers.cloudflare.com/workers/configuration/compatibility-flags/>
- `cf-fetch`: <https://developers.cloudflare.com/workers/runtime-apis/fetch/>
- `cf-headers`: <https://developers.cloudflare.com/fundamentals/reference/http-headers/>
- `cf-ratelimit-bug`: <https://github.com/cloudflare/workers-sdk/issues/16097>
- `cf-waf-rl`: <https://developers.cloudflare.com/waf/rate-limiting-rules/>
- `cf-observability`: <https://developers.cloudflare.com/observability/pricing/>
- `cf-browser-run`: <https://developers.cloudflare.com/browser-run/limits/>
- `cf-dev-terms`: <https://www.cloudflare.com/service-specific-terms-developer-platform/>
- `cf-terms`: <https://www.cloudflare.com/terms/>
- `cf-dpa`: <https://www.cloudflare.com/cloudflare-customer-dpa/>
- `ts-plans`: <https://developers.cloudflare.com/turnstile/plans/>
- `ts-verify`: <https://developers.cloudflare.com/turnstile/get-started/server-side-validation/>
- `ts-csp`: <https://developers.cloudflare.com/turnstile/reference/content-security-policy/>
- `ts-privacy`: <https://www.cloudflare.com/turnstile-privacy-policy/>
- `ts-preclearance`: <https://developers.cloudflare.com/turnstile/get-started/pre-clearance/>
- `cf-cookies`: <https://developers.cloudflare.com/fundamentals/reference/policies-compliances/cloudflare-cookies/>
- `cf-serving-pages`: <https://developers.cloudflare.com/pages/configuration/serving-pages/>
- `cf-pages-headers`: <https://developers.cloudflare.com/pages/configuration/headers/>
- `workers-sdk`: <https://github.com/cloudflare/workers-sdk> (Pages `_redirects` parser `parseRedirects.ts`; pages-shared asset `handler.ts`; source read for the review)

**Other hosts, accounts and DNS**
- `gh-actions-billing`: <https://docs.github.com/en/billing/concepts/product-billing/github-actions>
- `gh-dependabot`: <https://docs.github.com/en/code-security/concepts/supply-chain-security/dependabot-on-actions>
- `vercel-hobby`: <https://vercel.com/docs/plans/hobby>
- `netlify-credits`: <https://docs.netlify.com/manage/accounts-and-billing/billing/billing-for-credit-based-plans/how-credits-work/>
- `fly-pricing`: <https://docs.fly.io/about/pricing>
- `sq-dns`: <https://support.squarespace.com/hc/en-us/articles/31119879125645-DNS-records-for-web-hosting>
- `sq-cookies`: <https://support.squarespace.com/hc/en-us/articles/360001264507-The-cookies-Squarespace-uses>
- `gw-alias`: <https://support.google.com/a/thread/218324434/difference-between-alias-and-user-email-address?hl=en> (historical; the contact is now a dedicated Gmail account, §9.0)
- `gh-changelog`: <https://github.blog/changelog/> (immutable releases GA, 2025-10-28; Dependabot cooldown default, 2026-07-14)
- `gh-schedule`: <https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows#schedule>
- `npm-config`: <https://docs.npmjs.com/cli/v11/using-npm/config> (`min-release-age`)
- `zizmor`: <https://docs.zizmor.sh/>

**Privacy, children, marketing**
- `ecfr-coppa`: <https://www.ecfr.gov/current/title-16/chapter-I/subchapter-C/part-312>
- `ftc-coppa-faq`: <https://www.ftc.gov/business-guidance/resources/complying-coppa-frequently-asked-questions>
- `coppa-2025`: <https://www.federalregister.gov/documents/2025/04/22/2025-05904/childrens-online-privacy-protection-rule>
- `gdpr`: <https://eur-lex.europa.eu/eli/reg/2016/679/oj>
- `edpb-3-2018`: <https://www.edpb.europa.eu/sites/default/files/files/file1/edpb_guidelines_3_2018_territorial_scope_after_public_consultation_en_1.pdf>
- `edpb-2-2023`: <https://edpb.europa.eu/system/files/2024-10/edpb_guidelines_202302_technical_scope_art_53_eprivacydirective_v2_en_0.pdf>
- `ico-exceptions`: <https://ico.org.uk/for-organisations/direct-marketing-and-privacy-and-electronic-communications/guidance-on-the-use-of-storage-and-access-technologies/what-are-the-exceptions/> (pre-2026 guidance; the statutory list is now PECR Schedule A1)
- `pecr-a1`: <https://www.legislation.gov.uk/uksi/2003/2426/schedule/A1> (read 2026-11-08; inserted by the Data (Use and Access) Act 2025, in force 5 Feb 2026)
- `wp29-04-2012`: <https://ec.europa.eu/justice/article-29/documentation/opinion-recommendation/files/2012/wp194_en.pdf> (Opinion 04/2012 on cookie consent exemption)
- `ico-childrens-code`: <https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/childrens-information/childrens-code-guidance-and-resources/age-appropriate-design-a-code-of-practice-for-online-services/services-covered-by-this-code/>
- `cppa-threshold`: <https://cppa.ca.gov/announcements/2024/20241217.html>
- `caloppa-22575`: <https://leginfo.legislature.ca.gov/faces/codes_displaySection.xhtml?lawCode=BPC&sectionNum=22575>
- `caloppa-22577`: <https://leginfo.legislature.ca.gov/faces/codes_displaySection.xhtml?lawCode=BPC&sectionNum=22577>
- `de-doppa`: <https://delcode.delaware.gov/title6/c012C>
- `tx-tdpsa`: <https://statutes.capitol.texas.gov/GetStatute.aspx?Code=BC&Value=541.002>
- `tx-scope`: <https://statutes.capitol.texas.gov/Docs/BC/htm/BC.509.HTM>
- `state-laws-2026`: <https://www.bytebacklaw.com/2026/06/u-s-state-privacy-law-landscape-expands-to-24-states-what-the-latest-legislative-wave-means-for-businesses/>
- `tx-sb140`: <https://www.morganlewis.com/pubs/2025/09/texas-telephone-solicitation-law-now-covers-text-messages>
- `tcpa-imc`: <https://www.mofo.com/resources/insights/250130-eleventh-circuit-vacates-fcc-s-tcpa-one-to-one-consent-rule>
- `tcpa-mclaughlin`: <https://www.faegredrinker.com/en/insights/publications/2025/6/supreme-court-decides-mclaughlin-chiropractic-associates-v-mckesson-corp>
- `tcpa-revoke`: <https://www.consumerfinancialserviceslawmonitor.com/2026/01/fcc-further-extends-effective-date-for-tcpa-revoke-all-rule/>
- `lgpd`: <https://iapp.org/resources/article/brazilian-data-protection-law-lgpd-english-translation>
- `eca-digital`: <https://www.mayerbrown.com/en/insights/publications/2026/04/enforcement-of-brazils-eca-digital-introduces-new-obligations-for-companies>
- `pipeda`: <https://priv.gc.ca/en/privacy-topics/privacy-laws-in-canada/the-personal-information-protection-and-electronic-documents-act-pipeda/pipeda-compliance-help/pipeda-interpretation-bulletins/interpretations_03_ca>
- `au-privacy`: <https://sprintlaw.com.au/articles/privacy-law-reforms-small-business/>
- `india-dpdp`: <https://ca.hoganlovells.com/en/publications/indias-digital-personal-data-protection-act-2023-brought-into-force->
- `uk-osa-3`: <https://www.legislation.gov.uk/ukpga/2023/50/section/3>
- `uk-osa-4`: <https://www.legislation.gov.uk/ukpga/2023/50/section/4>
- `dsa`: <https://eur-lex.europa.eu/eli/reg/2022/2065/oj>
- `lg-muenchen`: <https://www.gesetze-bayern.de/Content/Document/Y-300-Z-GRURRS-B-2022-N-612>

**Accessibility**
- `wcag22`: <https://www.w3.org/TR/WCAG22/>
- `wcag-lum`: <https://www.w3.org/TR/WCAG22/#dfn-relative-luminance>
- `eaa-art4`: <https://www.legislation.gov.uk/eudr/2019/882/article/4>
- `doj-title2`: <https://www.consumerfinancialserviceslawmonitor.com/2026/04/doj-extends-title-ii-ada-web-accessibility-rule-compliance-deadlines-for-state-and-local-governments/>

**IP, fetching, terms**
- `cfr-202-1`: <https://www.ecfr.gov/current/title-37/chapter-II/subchapter-A/part-202/section-202.1>
- `ofl-faq`: <https://openfontlicense.org/ofl-faq/>
- `gf-faq`: <https://developers.google.com/fonts/faq>
- `gf-api`: <https://developers.google.com/fonts/docs/developer_api>
- `adobe-fonts-terms`: <https://wwwimages2.adobe.com/content/dam/cc/en/legal/servicetou/Adobe-Fonts-Product-Specific-Terms-en_US-20240618.pdf>
- `os2-fstype`: <https://learn.microsoft.com/en-us/typography/opentype/spec/os2>
- `dmca-512`: <https://www.law.cornell.edu/uscode/text/17/512>
- `dmca-fee`: <https://www.copyright.gov/rulemaking/onlinesp/NPR/faq.html>
- `s230`: <https://www.law.cornell.edu/uscode/text/47/230>
- `rfc9309`: <https://www.rfc-editor.org/rfc/rfc9309.html>
- `vanburen`: <https://www.supremecourt.gov/opinions/20pdf/19-783_k53l.pdf>
- `hiq`: <https://cdn.ca9.uscourts.gov/datastore/opinions/2022/04/18/17-16783.pdf> (*hiQ Labs v. LinkedIn*, 31 F.4th 1180 (9th Cir. 2022); persuasive only in Texas)
- `usc-227`: <https://www.law.cornell.edu/uscode/text/47/227>
- `tx-bc-302`: <https://statutes.capitol.texas.gov/Docs/BC/htm/BC.302.htm>
- `ufl`: <https://ubuntu.com/legal/font-licence>
- `apache-2`: <https://www.apache.org/licenses/LICENSE-2.0>
- `sw-kiwi`: <https://blog.ericgoldman.org/archives/2021/10/tos-supports-injunction-against-web-scraping-southwest-airlines-v-kiwi.htm>
- `tx-pen-33`: <https://statutes.capitol.texas.gov/Docs/PE/htm/PE.33.htm>
- `tx-cprc-143`: <https://tcss.legis.texas.gov/resources/CP/htm/CP.143.htm>
- `tx-71`: <https://statutes.capitol.texas.gov/Docs/BC/pdf/BC.71.pdf>
- `berman`: <https://law.justia.com/cases/federal/appellate-courts/ca9/20-16900/20-16900-2022-04-05.html>
- `dresser`: <https://www.courtlistener.com/opinion/2450513/dresser-industries-inc-v-page-petroleum-inc/>
- `brussels-1a`: <https://eur-lex.europa.eu/eli/reg/2012/1215/oj>
- `rome-1`: <https://eur-lex.europa.eu/eli/reg/2008/593/oj>
- `ca-civ-1789`: <https://leginfo.legislature.ca.gov/faces/codes_displayText.xhtml?article=1.&chapter=&division=3.&lawCode=CIV&part=4.&title=1.6D.>
- `ddg-5`: <https://www.gesetze-im-internet.de/ddg/__5.html>
- `uk-ecomm-reg6`: <https://www.legislation.gov.uk/uksi/2002/2013/regulation/6>
- `safe-browsing`: <https://developers.google.com/safe-browsing/v4/urls-hashing>
- `google-user-fetchers`: <https://developers.google.com/crawling/docs/crawlers-fetchers/google-user-triggered-fetchers>
- `google-site-names`: <https://developers.google.com/search/docs/appearance/site-names>
- `google-favicon`: <https://developers.google.com/search/docs/appearance/favicon-in-search>

**Web platform and libraries**
- `csp3-inherit`: <https://www.w3.org/TR/CSP3/#security-inherit-csp>
- `mdn-iframe`: <https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/iframe>
- `mdn-svg-img`: <https://developer.mozilla.org/en-US/docs/Web/SVG/Guides/SVG_as_an_image>
- `mdn-tt`: <https://developer.mozilla.org/en-US/docs/Web/API/Trusted_Types_API>
- `webkit-itp`: <https://webkit.org/tracking-prevention/>
- `webkit-216918`: <https://bugs.webkit.org/show_bug.cgi?id=216918>
- `apple-tn3156`: <https://developer.apple.com/documentation/technotes/tn3156-create-rich-previews-for-messages>
- `safelinks-issue`: <https://github.com/Skyfay/SkySend/issues/68>
- `rfc3986-frag`: <https://www.rfc-editor.org/rfc/rfc3986#section-3.5>
- `rfc1950`: <https://www.rfc-editor.org/rfc/rfc1950>
- `bcd`: <https://github.com/mdn/browser-compat-data>
- `web-features`: <https://github.com/web-platform-dx/web-features>
- `pqina-canvas`: <https://pqina.nl/blog/canvas-area-exceeds-the-maximum-limit/>
- `trojan-source`: <https://trojansource.codes/>
- `pdfjs-faq`: <https://github.com/mozilla/pdf.js/wiki/Frequently-Asked-Questions> (legacy build targets)
- `mdn-barcode`: <https://developer.mozilla.org/en-US/docs/Web/API/Barcode_Detection_API>
- `iana-tlds`: <https://data.iana.org/TLD/tlds-alpha-by-domain.txt>
- `uts39`: <https://www.unicode.org/reports/tr39/>
- `uax9`: <https://www.unicode.org/reports/tr9/>
- `rfc2606`: <https://www.rfc-editor.org/rfc/rfc2606>
- `so-63932958`: <https://stackoverflow.com/questions/63932958> (Firefox computed `transform` for SVG attributes)
- `bidi-js`: <https://www.npmjs.com/package/bidi-js>
- `cve-2024-4367`: <https://github.com/advisories/GHSA-wgrm-67xf-hhpq>
- `pdfjs-18015`: <https://github.com/mozilla/pdf.js/pull/18015>
- `osv`: <https://osv.dev>
- `almanac-seo`: <https://almanac.httparchive.org/en/2024/seo>
- `blendin`: <https://blendin.ai/brand-kit-extractor>

Library pages (versions read from the npm registry on 2026-11-08): https://www.npmjs.com/package/preact · https://www.npmjs.com/package/vite · https://www.npmjs.com/package/typescript · https://www.npmjs.com/package/culori · https://github.com/101arrowz/fflate · https://valibot.dev · https://www.npmjs.com/package/pdfkit · https://pdfkit.org/docs/accessibility.html · https://github.com/alafr/SVG-to-PDFKit · https://www.npmjs.com/package/pdfjs-dist · https://github.com/svg/svgo · https://github.com/dembrandt/dembrandt (heuristic shapes, MIT) · https://developers.cloudflare.com/browser-run/how-to/extract-brand-kit/ (baseline approach)

<!-- Reference-style link definitions (not rendered) -->
[cf-limits]: https://developers.cloudflare.com/workers/platform/limits/
[cf-pricing]: https://developers.cloudflare.com/workers/platform/pricing/
[cf-pages-limits]: https://developers.cloudflare.com/pages/platform/limits/
[cf-pages-fn-pricing]: https://developers.cloudflare.com/pages/functions/pricing/
[cf-pages-routing]: https://developers.cloudflare.com/pages/functions/routing/
[cf-pages-redirects]: https://developers.cloudflare.com/pages/configuration/redirects/
[cf-pages-domains]: https://developers.cloudflare.com/pages/configuration/custom-domains/
[cf-pages-debug]: https://developers.cloudflare.com/pages/configuration/debugging-pages/
[cf-direct-upload]: https://developers.cloudflare.com/pages/get-started/direct-upload/
[cf-migrate]: https://developers.cloudflare.com/workers/static-assets/migration-guides/migrate-from-pages/
[cf-known-issues]: https://developers.cloudflare.com/workers/platform/known-issues/
[cf-compat-flags]: https://developers.cloudflare.com/workers/configuration/compatibility-flags/
[cf-fetch]: https://developers.cloudflare.com/workers/runtime-apis/fetch/
[cf-headers]: https://developers.cloudflare.com/fundamentals/reference/http-headers/
[cf-ratelimit-bug]: https://github.com/cloudflare/workers-sdk/issues/16097
[cf-waf-rl]: https://developers.cloudflare.com/waf/rate-limiting-rules/
[cf-observability]: https://developers.cloudflare.com/observability/pricing/
[cf-browser-run]: https://developers.cloudflare.com/browser-run/limits/
[cf-dev-terms]: https://www.cloudflare.com/service-specific-terms-developer-platform/
[cf-terms]: https://www.cloudflare.com/terms/
[cf-dpa]: https://www.cloudflare.com/cloudflare-customer-dpa/
[ts-plans]: https://developers.cloudflare.com/turnstile/plans/
[ts-verify]: https://developers.cloudflare.com/turnstile/get-started/server-side-validation/
[ts-csp]: https://developers.cloudflare.com/turnstile/reference/content-security-policy/
[ts-privacy]: https://www.cloudflare.com/turnstile-privacy-policy/
[ts-preclearance]: https://developers.cloudflare.com/turnstile/get-started/pre-clearance/
[cf-cookies]: https://developers.cloudflare.com/fundamentals/reference/policies-compliances/cloudflare-cookies/
[cf-serving-pages]: https://developers.cloudflare.com/pages/configuration/serving-pages/
[cf-pages-headers]: https://developers.cloudflare.com/pages/configuration/headers/
[workers-sdk]: https://github.com/cloudflare/workers-sdk
[gh-actions-billing]: https://docs.github.com/en/billing/concepts/product-billing/github-actions
[gh-dependabot]: https://docs.github.com/en/code-security/concepts/supply-chain-security/dependabot-on-actions
[vercel-hobby]: https://vercel.com/docs/plans/hobby
[netlify-credits]: https://docs.netlify.com/manage/accounts-and-billing/billing/billing-for-credit-based-plans/how-credits-work/
[fly-pricing]: https://docs.fly.io/about/pricing
[sq-dns]: https://support.squarespace.com/hc/en-us/articles/31119879125645-DNS-records-for-web-hosting
[sq-cookies]: https://support.squarespace.com/hc/en-us/articles/360001264507-The-cookies-Squarespace-uses
[gw-alias]: https://support.google.com/a/thread/218324434/difference-between-alias-and-user-email-address?hl=en
[gh-changelog]: https://github.blog/changelog/
[gh-schedule]: https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows#schedule
[npm-config]: https://docs.npmjs.com/cli/v11/using-npm/config
[zizmor]: https://docs.zizmor.sh/
[ecfr-coppa]: https://www.ecfr.gov/current/title-16/chapter-I/subchapter-C/part-312
[ftc-coppa-faq]: https://www.ftc.gov/business-guidance/resources/complying-coppa-frequently-asked-questions
[coppa-2025]: https://www.federalregister.gov/documents/2025/04/22/2025-05904/childrens-online-privacy-protection-rule
[gdpr]: https://eur-lex.europa.eu/eli/reg/2016/679/oj
[edpb-3-2018]: https://www.edpb.europa.eu/sites/default/files/files/file1/edpb_guidelines_3_2018_territorial_scope_after_public_consultation_en_1.pdf
[edpb-2-2023]: https://edpb.europa.eu/system/files/2024-10/edpb_guidelines_202302_technical_scope_art_53_eprivacydirective_v2_en_0.pdf
[ico-exceptions]: https://ico.org.uk/for-organisations/direct-marketing-and-privacy-and-electronic-communications/guidance-on-the-use-of-storage-and-access-technologies/what-are-the-exceptions/
[pecr-a1]: https://www.legislation.gov.uk/uksi/2003/2426/schedule/A1
[wp29-04-2012]: https://ec.europa.eu/justice/article-29/documentation/opinion-recommendation/files/2012/wp194_en.pdf
[ico-childrens-code]: https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/childrens-information/childrens-code-guidance-and-resources/age-appropriate-design-a-code-of-practice-for-online-services/services-covered-by-this-code/
[cppa-threshold]: https://cppa.ca.gov/announcements/2024/20241217.html
[caloppa-22575]: https://leginfo.legislature.ca.gov/faces/codes_displaySection.xhtml?lawCode=BPC&sectionNum=22575
[caloppa-22577]: https://leginfo.legislature.ca.gov/faces/codes_displaySection.xhtml?lawCode=BPC&sectionNum=22577
[de-doppa]: https://delcode.delaware.gov/title6/c012C
[tx-tdpsa]: https://statutes.capitol.texas.gov/GetStatute.aspx?Code=BC&Value=541.002
[tx-scope]: https://statutes.capitol.texas.gov/Docs/BC/htm/BC.509.HTM
[state-laws-2026]: https://www.bytebacklaw.com/2026/06/u-s-state-privacy-law-landscape-expands-to-24-states-what-the-latest-legislative-wave-means-for-businesses/
[tx-sb140]: https://www.morganlewis.com/pubs/2025/09/texas-telephone-solicitation-law-now-covers-text-messages
[tcpa-imc]: https://www.mofo.com/resources/insights/250130-eleventh-circuit-vacates-fcc-s-tcpa-one-to-one-consent-rule
[tcpa-mclaughlin]: https://www.faegredrinker.com/en/insights/publications/2025/6/supreme-court-decides-mclaughlin-chiropractic-associates-v-mckesson-corp
[tcpa-revoke]: https://www.consumerfinancialserviceslawmonitor.com/2026/01/fcc-further-extends-effective-date-for-tcpa-revoke-all-rule/
[lgpd]: https://iapp.org/resources/article/brazilian-data-protection-law-lgpd-english-translation
[eca-digital]: https://www.mayerbrown.com/en/insights/publications/2026/04/enforcement-of-brazils-eca-digital-introduces-new-obligations-for-companies
[pipeda]: https://priv.gc.ca/en/privacy-topics/privacy-laws-in-canada/the-personal-information-protection-and-electronic-documents-act-pipeda/pipeda-compliance-help/pipeda-interpretation-bulletins/interpretations_03_ca
[au-privacy]: https://sprintlaw.com.au/articles/privacy-law-reforms-small-business/
[india-dpdp]: https://ca.hoganlovells.com/en/publications/indias-digital-personal-data-protection-act-2023-brought-into-force-
[uk-osa-3]: https://www.legislation.gov.uk/ukpga/2023/50/section/3
[uk-osa-4]: https://www.legislation.gov.uk/ukpga/2023/50/section/4
[dsa]: https://eur-lex.europa.eu/eli/reg/2022/2065/oj
[lg-muenchen]: https://www.gesetze-bayern.de/Content/Document/Y-300-Z-GRURRS-B-2022-N-612
[wcag22]: https://www.w3.org/TR/WCAG22/
[wcag-lum]: https://www.w3.org/TR/WCAG22/#dfn-relative-luminance
[eaa-art4]: https://www.legislation.gov.uk/eudr/2019/882/article/4
[doj-title2]: https://www.consumerfinancialserviceslawmonitor.com/2026/04/doj-extends-title-ii-ada-web-accessibility-rule-compliance-deadlines-for-state-and-local-governments/
[cfr-202-1]: https://www.ecfr.gov/current/title-37/chapter-II/subchapter-A/part-202/section-202.1
[ofl-faq]: https://openfontlicense.org/ofl-faq/
[gf-faq]: https://developers.google.com/fonts/faq
[gf-api]: https://developers.google.com/fonts/docs/developer_api
[adobe-fonts-terms]: https://wwwimages2.adobe.com/content/dam/cc/en/legal/servicetou/Adobe-Fonts-Product-Specific-Terms-en_US-20240618.pdf
[os2-fstype]: https://learn.microsoft.com/en-us/typography/opentype/spec/os2
[dmca-512]: https://www.law.cornell.edu/uscode/text/17/512
[dmca-fee]: https://www.copyright.gov/rulemaking/onlinesp/NPR/faq.html
[s230]: https://www.law.cornell.edu/uscode/text/47/230
[rfc9309]: https://www.rfc-editor.org/rfc/rfc9309.html
[vanburen]: https://www.supremecourt.gov/opinions/20pdf/19-783_k53l.pdf
[hiq]: https://cdn.ca9.uscourts.gov/datastore/opinions/2022/04/18/17-16783.pdf
[usc-227]: https://www.law.cornell.edu/uscode/text/47/227
[tx-bc-302]: https://statutes.capitol.texas.gov/Docs/BC/htm/BC.302.htm
[ufl]: https://ubuntu.com/legal/font-licence
[apache-2]: https://www.apache.org/licenses/LICENSE-2.0
[sw-kiwi]: https://blog.ericgoldman.org/archives/2021/10/tos-supports-injunction-against-web-scraping-southwest-airlines-v-kiwi.htm
[tx-pen-33]: https://statutes.capitol.texas.gov/Docs/PE/htm/PE.33.htm
[tx-cprc-143]: https://tcss.legis.texas.gov/resources/CP/htm/CP.143.htm
[tx-71]: https://statutes.capitol.texas.gov/Docs/BC/pdf/BC.71.pdf
[berman]: https://law.justia.com/cases/federal/appellate-courts/ca9/20-16900/20-16900-2022-04-05.html
[dresser]: https://www.courtlistener.com/opinion/2450513/dresser-industries-inc-v-page-petroleum-inc/
[brussels-1a]: https://eur-lex.europa.eu/eli/reg/2012/1215/oj
[rome-1]: https://eur-lex.europa.eu/eli/reg/2008/593/oj
[ca-civ-1789]: https://leginfo.legislature.ca.gov/faces/codes_displayText.xhtml?article=1.&chapter=&division=3.&lawCode=CIV&part=4.&title=1.6D.
[ddg-5]: https://www.gesetze-im-internet.de/ddg/__5.html
[uk-ecomm-reg6]: https://www.legislation.gov.uk/uksi/2002/2013/regulation/6
[safe-browsing]: https://developers.google.com/safe-browsing/v4/urls-hashing
[google-user-fetchers]: https://developers.google.com/crawling/docs/crawlers-fetchers/google-user-triggered-fetchers
[google-site-names]: https://developers.google.com/search/docs/appearance/site-names
[google-favicon]: https://developers.google.com/search/docs/appearance/favicon-in-search
[csp3-inherit]: https://www.w3.org/TR/CSP3/#security-inherit-csp
[mdn-iframe]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/iframe
[mdn-svg-img]: https://developer.mozilla.org/en-US/docs/Web/SVG/Guides/SVG_as_an_image
[mdn-tt]: https://developer.mozilla.org/en-US/docs/Web/API/Trusted_Types_API
[webkit-itp]: https://webkit.org/tracking-prevention/
[webkit-216918]: https://bugs.webkit.org/show_bug.cgi?id=216918
[apple-tn3156]: https://developer.apple.com/documentation/technotes/tn3156-create-rich-previews-for-messages
[safelinks-issue]: https://github.com/Skyfay/SkySend/issues/68
[rfc3986-frag]: https://www.rfc-editor.org/rfc/rfc3986#section-3.5
[rfc1950]: https://www.rfc-editor.org/rfc/rfc1950
[bcd]: https://github.com/mdn/browser-compat-data
[web-features]: https://github.com/web-platform-dx/web-features
[pqina-canvas]: https://pqina.nl/blog/canvas-area-exceeds-the-maximum-limit/
[trojan-source]: https://trojansource.codes/
[pdfjs-faq]: https://github.com/mozilla/pdf.js/wiki/Frequently-Asked-Questions
[mdn-barcode]: https://developer.mozilla.org/en-US/docs/Web/API/Barcode_Detection_API
[iana-tlds]: https://data.iana.org/TLD/tlds-alpha-by-domain.txt
[uts39]: https://www.unicode.org/reports/tr39/
[uax9]: https://www.unicode.org/reports/tr9/
[rfc2606]: https://www.rfc-editor.org/rfc/rfc2606
[so-63932958]: https://stackoverflow.com/questions/63932958
[bidi-js]: https://www.npmjs.com/package/bidi-js
[cve-2024-4367]: https://github.com/advisories/GHSA-wgrm-67xf-hhpq
[pdfjs-18015]: https://github.com/mozilla/pdf.js/pull/18015
[osv]: https://osv.dev
[almanac-seo]: https://almanac.httparchive.org/en/2024/seo
[blendin]: https://blendin.ai/brand-kit-extractor

---

## Appendix A: Review log

Adversarial review of revision 1 by five reviewers (legal, cost, security, feasibility, product), 2026-11-08. "Applied" names the sections changed. Where two findings overlapped, they were merged and the merge is noted. Facts re-checked during revision: npm publish dates for every pin (registry.npmjs.org, 2026-11-08) [Certain]; fflate 0.8.3 `unzlibSync` source (drops the Adler-32 trailer unchecked; no resize into a caller's `out` buffer) [Certain]; UK PECR Schedule A1 text and commencement (legislation.gov.uk) [Certain].

| # | Reviewer | Severity | Issue (short) | Disposition |
|---|---|---|---|---|
| 1 | legal | critical | Children's email survives "deletion" (Workspace auto-contacts, Sent, Trash, retention); alias lands in the wedding inbox; Google not named as processor | Applied: dedicated free Gmail mailbox with auto-contacts off, no forwarding, POP/IMAP or connected apps (§9.0, §4.7, Q4, M0, §11.1); no reply and same-day deletion for under-13 mail (§8.1, §10.14); inbound policy and privacy items 9, 14, 15; monthly mailbox check (§11.5 #8) |
| 2 | legal | high | Turnstile `cf_chl_*` cookies and Cloudflare's controller reuse make "no cookies" and "no tracking" untrue; tests only checked `<host>`; notice appeared after Turnstile loaded | Applied: all-domain cookie and challenge-frame storage test with allowlist (§15.6, `data-flow.md`); §8.2 ePrivacy/PECR row rewritten against Schedule A1 (text verified); privacy items 6 and 13; footer now "No ads or analytics"; notice visible from page load and never called consent (§2.3, §2.4, §9.7); §9.9 item 10; §10.4; §17.2; §17.3; R17; `cf-cookies` source |
| 3 | legal | high | A QR code or outlined text as a link "logo" passes every text filter | Applied: new §6.2.6 scannable-code check (rules-based, own code); encode offers only the no-logo link (§2.9); decode step 8a hides the logo (§7.4, S-LOGO); §7.5.3 rule 6; §15.7 positives verified by a dev-only oracle decoder, fuzz job and simple-icons negatives; §4.5 budget; banner and S-WATCH copy; R1 residual risk (outlined URLs need OCR) |
| 4 | cost | high | Quota can be drained for free; rejected requests, preflights and `/api/*` scanner noise all count | Applied: exact-path `_routes.json` and 404 mode (§10.6); §10.5 rewritten (no control protects the quota; Durable Object counter cannot help); §10.3 and §10.1 rows; §11.2; §11.4 #3 and #4; R6; C14; §14.1; privacy item 7; E-CAPACITY copy. Merged with #8 |
| 5 | security | high | Unknown paths serve the app with no CSP (SPA mode); trailing-slash redirects defeat exact header rules; `/k/*` → `/k/index.html` rule is silently dropped | Applied: `404.html` (404 mode), flat static files, `/k/:payload  /k/  200`, header plus first-in-head meta CSP generated from one table, baseline-CSP rule, header test (§2.1, §4.3, §7.3.4, §10.6, §15.7, M1). The `/clear` assertion now expects no `Clear-Site-Data` (finding #33) |
| 6 | security | high | Deploy pipeline supply-chain gaps: unpinned wrangler, brand-new pins, mutable font-mirror asset, unnamed trigger | Applied: wrangler exact devDependency invoked locally; `.npmrc` `min-release-age=7`, `check-release-age.mjs`, Dependabot cooldown; pins re-checked on the registry (preact 11.1.1, vite 8.3.5, vitest 5.0.5, Playwright 1.64.1, pdfjs-dist 6.4.299, wrangler 4.156.0); hashed pip installs; immutable releases; split font-mirror jobs; `FONT_MIRROR_SHA256` and `check-font-tar.py`; `deploy.yml` ci → build → deploy; banned triggers, zizmor, workflow grep; repo settings (§4.2, §4.3, §4.8, §7.1.3, §10.12, §10.13, M0) |
| 7 | security | high | `/k/` cannot run the DOM canonicalizer under its own CSP and Trusted Types | Applied, merged with #9: pure `canonical-grammar.ts` (`parseCanonical`, `serializeCanonical`, `validateCanonical`); `/k/` never calls DOMParser or the probe; variants built on the tree; PDF-worker and Edit-a-copy checks; lazy per-policy Trusted Types functions and an import zone (§6.2.2, §6.2.3, §7.1.1, §7.4, §10.7, §10.9, C5); tests in §15.1 and §15.5. Merge decision: a grammar failure hides the logo (S-LOGO) and renders the rest, rather than S-INVALID, so tightened checks can never reject a whole link (§7.5.3 rule 6); mutation tests assert the logo is not rendered |
| 8 | security | high | Quota drain without Turnstile; limiter keyed on full IPv6 address | Applied with #4: limiter key on the /64; R6 Medium/medium and accepted; §11.5 #2 compares Function requests with solved challenges (2 invocations per read after #22) |
| 9 | feasibility | high | Share view cannot validate SVG as written; computed-style numbers differ across browsers | Applied with #7: closed grammar (fixed attribute order, number regex, re-printed `d`, typed `url(#id)`, depth and element caps); transform read from `transform.baseVal` when computed is `none` (Firefox); optional `l.k` knockouts key; share view one-color uses primary; three-engine golden links (§6.2.2, §6.2.3, §7.5.2, §7.5.3) |
| 10 | feasibility | high | Browser floor wrong; pdf.js default build needs very new browsers | Applied: feature-tested floor (Chrome/Edge 111, Firefox 119, Safari/iOS 16.4, Samsung 22), `gate.ts`, `build.target`, `nomodule` fallback (§4.6); pdf.js legacy build with sizes, PDF feature gate and E-PDFOLD (§6.4, §2.7, §2.10); `imageOrientation` removed (§6.3); tests and device checks (§15.5, §15.8) |
| 11 | feasibility | high | No sanctioned way to run the real Function outside production | Applied: new §4.9 run modes (handler factories, frozen `PROD_CONFIG`, dev entry with test keys only, `check-fn-bundle.mjs`); `LAUNCHED` gate (§16); handler-level tests (§15.1); `dev:snapshots` harness (§15.2, §15.3, §4.5); M3 and M10 rewritten; §10.4 and §10.13 wording |
| 12 | feasibility | high | Non-Latin names render as missing glyphs or reversed; RFN families get a subset menu file | Applied: full declared non-CJK subsets, CJK families unsubset, RFN byte-for-byte TTF with no menu file and the variable-only substitute rule, mirror-build check (§7.1.3, §6.5.3); new §7.1.7 coverage chain with Noto fallbacks and bidi-js; PNG fallback aliases and direction (§7.2); script fixtures (§15.5); UI-font budget note (§4.5) |
| 13 | product | high | Contact filter misses formatted phone numbers and never defines a URL | Applied: `containsContact()` on name, tagline, color names and custom font names at input, extraction, encode and decode (§6.5.5, §5.9, §7.4, §7.5.1); §8.1 and §10.8 S-6 reworded to what it actually catches; must-reject and must-accept tables (§15.7) |
| 14 | product | high | Anchor layout described three incompatible ways; name could render twice; no headline on first screen | Applied: one gesture (name hangs into the page margin by half the margin) in §12.1, §12.4, §2.2, §3.2, §3.3, §12.2, §12.8, §13.1; `--sheet-ink` token; empty-state `<h1>` "Brand kit from your website or files." (§2.3) |
| 15 | legal | medium | Children's statement and URL notice untrue as written | Applied: §9.1 item 14 (worded "Kitsheet itself keeps none of it" so it stays true about Cloudflare's logs); URL microcopy (§2.3, §9.7); `claims-map.md` launch gate (§8.3, §9.7, M10) |
| 16 | legal | medium | Terms assent not tied to an action; not shown to upload-only users; "small text" | Applied: action-tied notices at body size under Read site, the file buttons and the export panel; share checkbox; Terms clause 1 lists acceptance actions (§2.3, §2.9, §9.2, §9.7) |
| 17 | legal | medium | Watchlist and blocklist evaded by homoglyphs | Applied: UTS #39 skeleton match key, mixed-script and edit-distance-1 triggers, `confusables.json` subset as static JSON with a fallback (§7.4, §10.14, §4.5); tests (§15.1, §15.7) |
| 18 | legal | medium | Mirror ignores UFL and Apache-2.0 modification conditions | Applied: license branch (UFL byte for byte, Apache `CHANGES.txt` and `NOTICE`), mirror-build check, §8.2 tag to [Likely], §9.9 item 7 |
| 19 | legal | medium | Legal conclusions over-tagged; PECR analysis predates Schedule A1 | Applied: CFAA retagged [Likely] with *Van Buren* fn. 8, Tex. Penal Code §33.02 and the *hiQ* opinion as persuasive only; TCPA cites 47 U.S.C. §227 and Bus. & Com. Code ch. 302 at [Likely]; PECR row against Sch. A1 paras 4(2)(e)(ii) and 6 (text verified); **Don't remember my settings** control (§7.6.1, §8.2, §9.1 item 6). The suggested "Saving a draft · Turn off" line is superseded by #41's opt-in Keep checkbox |
| 20 | legal | low | Phone guard misses common formats; extracted text has no rule | Applied through #13's `containsContact()`, which covers every listed case (NFKC, full-width digits, grouped numbers) and skips extracted candidates silently; the finding's single regex was not used |
| 21 | cost | medium | Every Cloudflare 5xx treated as "daily limit reached" | Applied: three-way classifier and E-HEAVY (§11.3, §2.10, §15.5); merged with #40 (one retry; two consecutive capacity signals before the day lockout) |
| 22 | cost | medium | One `/api/image` call per image triples quota use | Applied: one batched `POST /api/images` with KITX framing; 2 invocations per read (50,000 reads/day ceiling); nonce single use, best effort; per-image fallback only if p99 CPU > 8 ms (C19, §5.3, §5.2.3, §11.2, §15.7) |
| 23 | cost | medium | GitHub disables scheduled workflows after 60 inactive days | Applied: owner's calendar is the primary trigger, step 0, workflow-state listing (§11.5, §11.1, R15) |
| 24 | cost | medium | "Permanent" links tie the tool to the wedding domain; a lapsed domain leaks old fragments | Applied as a flagged hard conflict: §0.3 #1, Q1 due before M8, §7.5.3 rule 1 and Terms clause 9 ("while Kitsheet is online at…"), §17.3 trigger, R16, `*.pages.dev` named as the $0-forever option. Declined in part: the Q1 default stays on the subdomain because hosting there is a locked decision; the owner chooses |
| 25 | cost | low | Font-mirror fallback would not produce static files | Applied: no-key `google/fonts` path with `varLib.instancer`, run once in M2 (§7.1.3 step 1, R10, §11.1) |
| 26 | cost | low | Device and assistive-technology matrix assumes hardware the owner may not own | Applied: §11.4 #13; §15.8 uses owned devices and free emulators; `/accessibility` lists only what was run (§9.5, §13.5) |
| 27 | cost | low | Monthly production test cannot pass managed Turnstile headlessly | Applied: automated part makes no URL read; manual URL read with HAR; no production bypass of any kind (§11.5 #7, §15.6, §10.4, §4.9) |
| 28 | security | medium | Fetched SVG skips canonicalization; raster fallback draws the wrong file; CSS escapes beat `url(` deletion | Applied: SVG responses go through §6.2 and never become raw SVG Blobs (§5.3, §5.8); `favicon.mime` union and invariant (§7.5.1); raster-only sanitizer on the original with user confirmation (§6.2.2, §2.5); `<style>` with a backslash removed (§6.2.1); Blob-constructor test (§15.7) |
| 29 | security | medium | Reusable bearer ticket turns `/api/image` into an amplifier; subrequest tally incomplete | Applied: ticket bound to the caller's /64 or /32, 60 s expiry, image hosts restricted to page, redirect-chain, CSS and curated CDN hosts, per-target limiter on every outbound hostname, `budget.take` covering DoH and siteverify (§5.2.3, §5.2.4, §5.2.7, §5.3, §10.3, §10.5) |
| 30 | security | medium | `idempotency_key` could enable token replay; no guard against test secrets | Applied: key removed; `challenge_ts` ≤ 300 s; test-secret prefixes return 500; duplicate-token test (§5.2.1, §10.4, §15.1) |
| 31 | security | medium | Probe watchdog, pdf.js timeout and fetched-image size limits are cooperative or missing | Applied: CSS pre-pass caps and sliced reads (§5.4.2, §5.4.7); `worker.terminate()` (§6.4); header-dimension check and 16 MP cap on fetched images (§5.3, §6.1); tests (§15.7) |
| 32 | security | medium | fflate does not verify Adler-32 and truncates silently | Applied after reading the 0.8.3 source: own Adler-32 and 32,769-byte overflow detection (§7.4, §4.2); golden tests (§15.1) |
| 33 | security | low | `/clear` with `Clear-Site-Data` lets any site wipe a draft | Applied: `/clear` is a **Clear now** page without the header; in-page clear is the only deleting path (§2.1, §7.6.3, §10.6) |
| 34 | feasibility | medium | Relative URLs resolve against Kitsheet's origin | Applied: `docBase`, CSS URL tokenizer rewrite, attribute-only reads with an ESLint ban, CDN fixture (§5.4.1, §5.4.2, §10.11, §15.2) |
| 35 | feasibility | medium | CPU plan names a measurement that cannot work and a fallback that does not exist | Applied: DevTools profiles via `dev:snapshots`, byte-level scans, streaming HTMLRewriter, step-down table, `budget.take`, worked tally of 26 (§4.5, §5.2.2, §5.2.3, R4). Changed: production CPU metrics gate at M10, not M3, because no Function runs in production before M10 (#11) |
| 36 | feasibility | medium | SVG decoded with `createImageBitmap` in workers; canvases uncapped | Applied: main-thread `rasterizeSvg`, 4,096 px and 16.7 MP clamps, resize-on-decode with `pixelated`, WebKit fallback, real-iPhone checks (§6.2.2, §6.2.4, §6.3, §7.1.1, §15.8) |
| 37 | feasibility | medium | Milestones put the biggest risk last and include steps out of order | Applied: M4 corpus exit gate with thresholds; deploy skips the mirror while `FONT_MIRROR_VERSION` is empty; M1 PDFKit + svg-to-pdfkit spike replaces the M5 check; command-based done checks (§16, §4.8) |
| 38 | feasibility | medium | `tagged:true` does not produce a structure tree | Applied: structure map, `withStruct()` with a lint rule, `pdfinfo -struct-text`, veraPDF report (§7.1.2, C18, M7, §15.5) |
| 39 | feasibility | medium | Determinism ignores installed fonts and scrollbars | Applied: `overflow:hidden`, `bkm` metric pass, 0.25 quantization, three-OS CI (§5.4.3, §5.4.4, §5.5.2, §5.11, §15.2) |
| 40 | product | medium | Missing states and a false daily-limit claim | Applied: E-HEAVY, E-UPSTREAM, E-TLS, E-REDIRECTS, E-EXPORT, E-COPY, E-NOCOLOR; two-signal capacity lockout (§2.10, §2.9, §11.3, §15.5) |
| 41 | product | medium | Auto-saved draft exposes kits on shared computers | Applied: tab-scoped sessionStorage by default; opt-in **Keep this draft on this device**; preferences written only when changed (§7.6.1, §2.3, §2.8, §8.2, §9.1, §15.6) |
| 42 | product | medium | In-tool copy says "we" and "our"; "official" and "approved" used | Applied: "Kitsheet" as subject throughout (§2.3, §2.9, §2.10, §9.7, C10); §12.9 negation-only rule; `check-copy.mjs` |
| 43 | product | medium | No cut list | Applied: §1.5 |
| 44 | product | medium | G1 and G7 have no pass/fail | Applied: one-time competitor comparison in `competitor.json`, facts only, after a terms check (G1, §15.3, M9, Q10); `design-review.md` checklist with the owner as sole approver gating M1, M6 and M7 (G7, §16) |
| 45 | product | medium | Headline fitting impossible for long names | Applied: shared `fitHeadline` with 3-line floors and an in-word break, never truncating (§12.2, §3.2, §3.3, §7.2, §13.1) |
| 46 | product | medium | Home page metadata and builder credit missing | Applied: §12.11 (title, description, canonical, `WebApplication` JSON-LD, sitemap) and the colophon credit (§3.2). The `<h1>` follows #14 (the empty-state line), not the product line |
| 47 | product | medium | Pre-launch gate undefined; proposed `/app/` path with a live API before launch | Declined: a reachable app and live Function in production before M10 conflict with #11 (no app bundle and no Function in production until launch). The mechanism gap is closed by `LAUNCHED` (§16); device checks run against a LAN preview for uploads and exports, and on production after the flip and before any announcement for the URL flow (§15.8); legal drafts are not published early |
| 48 | cost | refuted | Public-repo Actions artifacts could exhaust a 500 MB pool and block deploys | Refuted; not applied. GitHub's billing docs meter artifact storage for private repositories only; the spec stores the font tree as a Release asset, not an artifact |
