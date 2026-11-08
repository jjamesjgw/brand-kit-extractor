# Research: PDF brand guide, PNG brand board, URL-fragment share links, kit schema

**Scope:** in-browser generation and sharing only. Nothing in this area touches a server except static font files on the tool's own origin.
**Research date:** 2026-11-08.
**Confidence tags:** [Certain] = read on a primary source today, or measured by me today (method given). [Likely] = strong inference or consistent secondary sources. [Guessing] = gap-filling; verify before relying on it.
**How measurements were made:** Node 22.23.4 in a Linux sandbox. Bundle sizes come from `esbuild --bundle --minify --format=esm --platform=browser`, then `gzip -9`. PDF validity was checked with poppler `pdffonts`/`pdftoppm` and `pypdf`. Compression used Node `CompressionStream`, `zlib` and fflate 0.8.3. QR versions came from `qrcode` 1.5.4. Library versions are the npm "latest" on 2026-11-08. Scripts and samples are in `/tmp/claude-0/-home-user/02d649eb-8126-5ff7-985f-ad69e6e47e14/scratchpad/bench/`.
**Related tracks:** this report assumes the hosting decision in `research-infra-zero-cost.md` (Cloudflare Pages, static assets, `_headers`). It also assumes the privacy rules in `research-privacy-law.md` (PRIV-01 same-origin only, PRIV-12 local draft, PRIV-15–17 share view) and the abuse rules in `research-ip-abuse-law.md` (S-1…S-8, T-1…T-9). Where my findings differ from theirs, I say so in section 2.11.

---

## 1. Concrete requirements for this product

IDs (OUT-xx) are for the spec.

### A. PDF brand guide

- **OUT-01. Generate the PDF with PDFKit 0.20.2 (MIT) plus svg-to-pdfkit 0.1.8 (MIT), lazy-loaded on the first "Download PDF" intent.**
  - Measured cost: about 213 KB gzip for PDFKit's browser build plus 19.5 KB gzip for svg-to-pdfkit. Neither is in the first-load bundle. [Certain] (measured)
  - Why PDFKit:
    - It was the only engine tested that applies GPOS kerning. Its content stream has `TJ` arrays with kern offsets for "AV", "To" and "Wa". pdf-lib and jsPDF emit a flat `Tj`, so they render without kerning. [Certain] (measured)
    - It subsets fonts properly: a font face costs 3.7–6.9 KB in the PDF. [Certain] (measured)
    - svg-to-pdfkit drew a raw Adobe-Illustrator-style SVG correctly, including `<style>` classes. It also drew a simple-icons SVG and an SVGO-minified SVG, all as vectors. [Certain] (measured)
    - It supports tagged PDF / PDF-UA options. [Certain] (pdfkit.org/docs/accessibility.html)
  - A realistic test guide had 7 Letter pages, 3 subset fonts, 6 vector logos and 35 swatches. It was 57 KB and took 451 ms in Node. [Certain] (measured)
- **OUT-02. Run PDF generation in a module Web Worker.**
  - PDFKit and svg-to-pdfkit need no DOM: both ran in plain Node. [Certain] (measured)
  - The worker fetches font bytes from the same origin, builds the PDF, and posts back a `Blob`.
  - `DOMParser` is not available in workers, so canonicalize SVG on the main thread first (OUT-31). [Likely]
- **OUT-03. Feed the PDF engine only static TTF or WOFF 1.0 instances. Never WOFF2. Never variable fonts.**
  - Measured failures:
    - pdf-lib with @pdf-lib/fontkit 1.1.1 and `subset:true` on WOFF2 throws an **uncatchable async RangeError** (`Index out of range` in `TTFSubset._addGlyph`, thrown from a timer).
    - PDFKit with fontkit 2.0.4 throws `Offset is outside the bounds of the DataView` on the same files.
    - This happened for Fontsource WOFF2 and for Google-served Latin WOFF2, both static and variable.
    - pdf-lib with `subset:false` embeds raw WOFF/WOFF2 bytes as `FontFile2`. Poppler then reports "Embedded font file may be invalid" and cannot render.
  - WOFF 1.0 and TTF worked in pdf-lib and PDFKit for all 6 families tested: Inter, Playfair Display, Roboto, Lato, Montserrat, Open Sans.
  - [Certain] (measured). PDFKit's docs claim WOFF2 support; the claim did not hold for these files.
- **OUT-04. Page size follows the browser locale: US Letter for `en-US`, `en-CA`, `es-MX` and `fr-CA`; A4 for everything else.** Offer a toggle. [Guessing] (product choice)
- **OUT-05. Use PDFKit's accessibility options:** `tagged: true`, `displayTitle: true`, `lang` from the UI locale, `info.Title = "<Brand> brand guide"`. The measured test produced `Tagged: yes` per `pdfinfo`. [Certain] (measured). Whether that is enough for full PDF/UA conformance is [Guessing].
- **OUT-06. Do not print CMYK or Pantone values computed from hex in v1.** A naive RGB→CMYK formula is not a print spec. Printing it would invent numbers, which the owner has banned. Show HEX, RGB and HSL only, and add the line "Screen colors (sRGB)". [Likely]
- **OUT-07. Download handling.**
  - **Default:** `<a download="<slug>-brand-guide.pdf" href="blob:…">`. The slug is ASCII, at most 60 characters, derived from the name. This is supported on iOS Safari 13+, Chrome, Firefox 20+ and Safari 10.1+. [Certain] (MDN browser-compat-data 8.1.8)
  - **Second button on iOS and Android:** a "Share / Save to Files" button that calls `navigator.share({files:[pdfFile]})`. Supported on Safari 14+, iOS 14+ and Chrome Android 76+. Not supported on Firefox, and on desktop Chrome only on Windows and ChromeOS. [Certain] (BCD)
  - **Generate the Blob before showing that button.** Web Share needs a live user activation, and a slow async build can lose it. [Likely]
  - **In-app browsers:** Instagram, Facebook, WhatsApp and LinkedIn in-app browsers silently drop `blob:` downloads, on both iOS WKWebView and Android WebView. Detect them from the UA and show "Open this page in Safari/Chrome to download." [Likely] (WebKit bug 216918; docuseal#719)
- **OUT-08. Not the primary path: HTML plus `window.print()`, Paged.js or `@react-pdf/renderer`.** Reasons are in section 2.1. A print stylesheet for the on-screen kit is a cheap optional extra.

### B. PNG brand board

- **OUT-09. Draw the board with Canvas 2D directly. Use no DOM-to-image library.** The library cost is 0 KB.
  - html-to-image, modern-screenshot and snapdom all use SVG `foreignObject`. html-to-image has open Safari bugs: blank first render, missing images, no SVG images. [Likely] (html-to-image issues #461, #361)
  - satori with resvg-wasm costs about 685 KB gzip of JS plus a 2.48 MB wasm file (949 KB gzip), under the MPL-2.0 licence. [Certain] (measured)
- **OUT-10. Render at a fixed 2× scale.** The layout is 1600×1000 CSS px, the canvas is 3200×2000 device px, and the code calls `ctx.scale(2,2)`.
  - 6.4 MP is well under iOS Safari's 16,777,216-pixel canvas-area limit. [Likely] (pqina.nl)
  - A flat-design board at this size encoded to about 130 KB of PNG with Pillow. Browser encoders compress less, so expect 150–500 KB. [Guessing]
  - Free the canvas after export by setting width and height to 0.
- **OUT-11. Create the context with `getContext('2d', {colorSpace:'srgb'})` so hex values stay exact.** Supported in Chrome 92+ and Safari 15.2+. Firefox ignores the option and is sRGB anyway. [Certain] (BCD)
- **OUT-12. Load fonts through `FontFace(alias, ArrayBuffer)` using internal alias names such as `bk-h-700`.** Await `document.fonts.load('700 96px bk-h-700', sample)`. If it resolves to an empty array, fall back to the generic stack and flag "font unavailable". Never put user-supplied family names into `ctx.font`. [Likely]
- **OUT-13. Draw only `blob:` images built from in-memory bytes:** uploads, or bytes proxied by `/api/image` as octet-stream. Never draw a cross-origin `<img>`, because it taints the canvas and `toBlob` then throws `SecurityError`. Before rasterizing an SVG logo, give it explicit `width`/`height`. Firefox does not draw viewBox-only SVGs to canvas reliably. [Likely]
- **OUT-14. Do not use `ctx.letterSpacing`.** It needs Safari 18.4+, and `fontKerning` and `fontStretch` are missing in Safari. Bake any tracking into the template by drawing glyph runs. [Certain] (BCD) on support
- **OUT-15. Export with `canvas.toBlob(cb,'image/png')`.** Download works as in OUT-07. In in-app browsers, show the PNG in an `<img>` overlay so the user can long-press and save. [Likely]

### C. Font binaries

- **OUT-16. Self-host font files as static assets on the tool's own origin.** Do not link Google Fonts, Bunny, jsDelivr or Fontsource from the browser, and do not proxy fonts through a Pages Function.
  - This matches PRIV-01 and PRIV-02 and costs $0.
  - A Function proxy would spend the shared 100,000 invocations/day on every font file. [Certain] (Cloudflare Pages limits: Functions count toward the Workers quota)
- **OUT-17. Build the font mirror in GitHub Actions, not in Pages CI.** Pages builds time out after 20 minutes. [Certain] (Pages limits page)
  - Pipeline:
    1. Get the family list from the Google Fonts Developer API, `webfonts?key=…`. A key is required, and the API returns static TTF instances by default. [Certain] (developers.google.com/fonts/docs/developer_api)
    2. Download each static TTF.
    3. Run `pyftsubset` from fonttools (MIT) to Latin + Latin-Ext, keeping `--layout-features='*'` and `--name-IDs='*'`, and output `--flavor=woff`.
    4. Write `/fonts/<slug>/<weight>[i].woff`.
    5. Write `/fonts/<slug>/LICENSE.txt` and `/fonts/<slug>/menu.woff`, a subset holding only the glyphs of the family name, for the picker.
    6. Write `/fonts/catalog.json`.
  - pyftsubset's default `--name-IDs` keeps only IDs 0–6, which drops the licence fields (13 and 14). That is why `--name-IDs='*'` is needed. [Likely]
- **OUT-18. One file per style serves both the on-screen preview (FontFace) and the PDF (fontkit).** Measured Latin+Latin-Ext WOFF size per face: 29.8–62.4 KB, average about 41 KB over 6 families. Every file embedded cleanly in pdf-lib and PDFKit. [Certain] (measured)
- **OUT-19. Catalog size and Pages fit.**
  - google-font-metadata 6.0.8 lists 1,908 families. 1,892 have Latin. They have 7,501 static variants in all. 525 families are variable. Licences: OFL 1,828, Apache 36, UFL 5. [Certain] (package data)
  - Mirroring every variant gives about 7,501 font files, 1,892 menu files and 1,892 licence files, roughly 11,300 files and about 300 MB. That is under the Pages Free limits of **20,000 files per site and 25 MiB per file**, and Pages states no total-size cap. [Certain] (limits page); [Guessing] (bytes, extrapolated from 6 families)
  - If deploy time is a problem, mirror only 400, 700 and 400-italic. That is 2,656 regular-weight files. [Certain] (count)
- **OUT-20. Families with a Reserved Font Name.** The build greps each family's OFL.txt for "Reserved Font Name". For those families, ship the unmodified Google static TTF (no subsetting), or leave the family out. See T-1/T-9 in the IP report. [Guessing] (pending legal review)
- **OUT-21. Fonts that are not on Google Fonts** (Adobe, self-hosted commercial, system) are display-only.
  - Show the name, the source label and the generic fallback.
  - The preview, PNG and PDF use a fallback face, labelled "Shown in <fallback>. <Name> is licensed separately."
  - Never download font binaries from the scanned site (T-1). [Certain] (licensing logic per the IP report)

### D. Share links: format, codec, size budget

- **OUT-22. Link format: `https://<host>/k#1.<payload>.<len>`.**
  - `1` is the codec and schema major version.
  - `<payload>` is base64url (RFC 4648 §5, unpadded) of a **zlib-format** stream: deflate plus a 2-byte header plus Adler-32. The stream uses preset dictionary **D1**.
  - `<len>` is the payload length in base36. It detects truncation, and because it ends in an alphanumeric character, auto-linkers that drop trailing `_`, `.` or `~` cannot cut the payload short. [Likely] (GFM autolink rules)
  - The base64url alphabet `A–Z a–z 0–9 - _` is unreserved in URI fragments, so it needs no percent-encoding. [Certain] (RFC 3986 §3.5)
- **OUT-23. Codec: fflate 0.8.3 (MIT) `zlibSync`/`unzlibSync` with a frozen dictionary.**
  - The editor loads 5.0 KB gzip; the viewer loads only `unzlibSync`, 2.3 KB gzip. [Certain] (measured)
  - With D1, a no-logo kit is 263 base64url characters instead of 310 without a dictionary. Logo kits shrink 6–25%. [Certain] (measured)
  - `CompressionStream('deflate-raw')` would cost 0 KB and is supported everywhere relevant (Chrome 103, Firefox 113, Safari 16.4). But it cannot use a preset dictionary. [Certain] (BCD)
  - Native `brotli` in CompressionStream is not in Chrome. [Certain] (BCD)
  - D1 is versioned with the format. **Never edit D1.** A new dictionary means format `2.`.
- **OUT-24. Size budget, using a host of about 31 characters:**

  | Kit | Full URL length (measured) |
  |---|---|
  | No logo | ~300 chars |
  | Icon-only SVG | ~410 |
  | Typical single-path mark (simple-icons median) | ~840 |
  | Outlined wordmark | 2,560–3,280 |
  | Wordmark + outlined tagline | 4,100–5,850 |

  [Certain] (measured; see 2.6)

  Rules:
  - **Canonical SVG cap: 8,192 bytes** (matches S-7).
  - **Fragment hard cap: 6,000 characters.** The generator refuses above it. The decoder rejects any fragment over 8,192 characters before decoding, and caps inflate output at 32 KB.
  - **Soft tiers:**
    - ≤ 1,900 characters: "fits everywhere", including a Discord message (2,000-character limit for non-Nitro users).
    - ≤ 2,900: fits a LinkedIn post (3,000-character post limit).
    - Above that, warn: "Long link: works in email, Slack, iMessage and X; too long for LinkedIn and Discord posts."
  - Always offer **"Copy link without logo"**, which is always under about 450 characters.
  - Offer a **QR code only for the no-logo link.** About 300–540 characters needs QR version 12–18 at ECC M. A 2,600-character link needs version 38 at ECC L, which is impractical. [Certain] (measured with qrcode 1.5.4)
- **OUT-25. Survive mail-gateway rewriting.**
  - Microsoft Defender Safe Links has been observed to rewrite `#` into `%23`. The payload then lands in the **path** (`/k%231.<payload>.<len>`), which is sent to the server. [Likely] (SkySend issue #68, July 2026)
  - The share-route bootstrap must detect `/k%23` and `/k%2523` with an anchored regex. Payload characters never include `%`. It then moves the text into `location.hash` with `history.replaceState`.
  - Cloudflare's URL limit is 16 KB, so these paths still load. [Certain] (Cloudflare connection-limits doc)
  - **Privacy copy must not claim the server "can never" see a link.** See 2.11.
- **OUT-26. Static, generic preview for the share route.**
  - `/k` is a static HTML file with no fragment-dependent metadata:
    - `<title>Shared brand kit · {Tool}</title>`
    - `og:title` "A brand kit made with {Tool}"
    - `og:site_name` "{Tool}"
    - `og:description` "Made by a visitor. Not verified by or affiliated with any brand."
    - `og:image` = `/og/share.png`, 1200×630, flat, the tool's own branding, never a kit
    - `og:url` = `https://<host>/k` (no fragment)
    - `twitter:card=summary_large_image`
    - `robots` meta `noindex, nofollow, noarchive`
  - Messages (iMessage) previews "do not follow meta redirects, nor run JavaScript". Images should be at least 900 px wide, and the main resource is capped at 1 MB. [Certain] (Apple TN3156)
  - JS must never rewrite `document.title` or OG tags with kit text (S-4, PRIV-15).

### E. Share-link security

- **OUT-27. Treat the decoded fragment as hostile.** Pipeline:
  1. Length gate.
  2. Format and version check.
  3. `<len>` check.
  4. base64url decode, then `unzlibSync` with D1 and a 32 KB output cap. Adler-32 catches corruption.
  5. UTF-8 decode with `fatal:true`.
  6. `JSON.parse`.
  7. **valibot 1.5.0** (MIT, about 1.7 KB gzip for this schema) `object` schemas. They strip unknown keys, which also drops `__proto__`.
  8. Field normalization (OUT-28).
  9. Static blocklist check against `/blocklist.json` (S-8). The hash is SHA-256 of the **canonical JSON**, not of the compressed bytes.
  10. Render.

  Any failure shows a specific, non-technical message (2.9.6). [Likely]
- **OUT-28. Text normalization for `n`, `t` and color names.**
  - NFC normalize.
  - Reject ill-formed UTF-16 (`String.prototype.isWellFormed`, Safari 16.4+, with a regex fallback).
  - Strip C0/C1 controls and these format characters: U+200B–U+200F, U+202A–U+202E, U+2060–U+2069, U+061C, U+FEFF, and U+E0000–U+E007F (tags).
  - Collapse whitespace.
  - Count graphemes with `Intl.Segmenter`, capped at 60 for the name, 120 for the tagline and 32 for color names.
  - Reject URLs, emails and phone-like digit runs (S-6).
  - Render only with `textContent` inside elements carrying `dir="auto"` and `unicode-bidi: isolate`. [Likely] (Trojan Source, CVE-2021-42574)
- **OUT-29. Colors are `^[0-9a-f]{6}$` only, lowercase, with no alpha.** Use them only through `element.style.setProperty('--c1', '#'+hex)` and canvas/PDF APIs. Never concatenate them into CSS text. [Likely]
- **OUT-30. Font names never reach CSS text.**
  - A share link names a family only by exact match against `catalog.json`. A name not in the catalog is display text, matched against `^[\p{L}\p{N} '&.-]{1,64}$`u.
  - CSS uses internal aliases (`bk-h`, `bk-b`) registered with `FontFace`.
  - `@font-face` URLs are built only from catalog slugs (`^[a-z0-9-]{1,64}$`). [Likely]
- **OUT-31. SVG logos are rebuilt from an allowlist, not blocklist-sanitized.**
  - First, refuse any input that contains `<!DOCTYPE` or `<!ENTITY` (entity bombs) or is longer than 512 KB for uploads or 8 KB for links.
  - Parse with `DOMParser('image/svg+xml')`.
  - Walk the tree and **re-serialize from scratch**, emitting only:
    - **Elements:** `svg g path rect circle ellipse line polyline polygon linearGradient radialGradient stop defs`.
    - **Attributes, each validated:**
      - `viewBox`, `width`, `height`, `x`, `y`, `cx`, `cy`, `r`, `rx`, `ry`, `x1`, `x2`, `y1`, `y2`, `points`: numeric.
      - `d`: `^[MmLlHhVvCcSsQqTtAaZz0-9eE.,\s+-]+$`.
      - `transform`: a strict regex.
      - `fill`, `stroke`, `stop-color`: hex, `none`, or `url(#id)` pointing at a local id only.
      - `fill-rule`, `clip-rule`, `opacity`, `fill-opacity`, `stroke-width`, `stroke-linecap`, `stroke-linejoin`, `offset`, `gradientUnits`, `gradientTransform`, `id` (`^[A-Za-z][\w-]{0,31}$`).
  - Never emit `script`, `foreignObject`, `image`, `use`, `a`, `animate*`, `set`, `style`, `href`/`xlink:href`, `on*` or `url()` that leaves the document.
  - Before canonicalizing, map inline `style` and `<style>` class rules to attributes; SVGO's `inlineStyles`/`convertStyleToAttrs` can do this.
  - Then render **only** as `<img src=blob:image/svg+xml>`, where scripts are disabled and external resources are not loaded. [Certain] (MDN "SVG as an image")
  - DOMPurify (3.4.17, MPL-2.0 or Apache-2.0) is optional defense in depth, not the primary control. It had more than 10 advisories in 2025–2026, the latest fixed in 3.4.16 and 3.4.17. [Certain] (GitLab advisory DB, npm)
- **OUT-32. Headers for `/k` (set via Pages `_headers`; each header is limited to 2,000 characters):**
  ```
  Content-Security-Policy: default-src 'none'; script-src 'self'; connect-src 'self'; img-src 'self' blob:; font-src 'self'; style-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'; object-src 'none'; require-trusted-types-for 'script'; trusted-types 'none'
  X-Robots-Tag: noindex, nofollow, noarchive
  Referrer-Policy: no-referrer
  X-Content-Type-Options: nosniff
  Cross-Origin-Opener-Policy: same-origin
  Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=(), usb=()
  ```
  - Trusted Types has been Baseline since February 2026: Chrome 83, Firefox 148 (released 2026-02-24), Safari 26. [Certain] (BCD; Firefox 148 release notes; MDN)
  - The app route needs the Turnstile hosts added (infra report) and `worker-src 'self'`.
  - The PDFKit, svg-to-pdfkit, fflate, valibot and SVGO bundles contain no `eval` or `new Function`, so no `'unsafe-eval'` is needed. pdfmake's bundle does contain `new Function`. [Certain] (grep of built bundles)
- **OUT-33. Anti-impersonation chrome (aligns with S-3/S-4/PRIV-17):**
  - A persistent top banner that cannot be dismissed: "Made with {Tool} by a visitor. User-generated, not verified by or affiliated with any brand." with a "Report" mailto link.
  - The tool's own favicon.
  - A generic `document.title`.
  - No clickable links in kit content.
  - PDF and PNG exported **from the share view** print "Shared kit. Not verified by the brand owner." in the footer or colophon. [Guessing] on the exact wording; owner decision.

### F. Local persistence

- **OUT-34. Use IndexedDB (database `bk`, version 1) with two stores.**
  - `draft` holds one key `current`, containing `KitDocV1`.
  - `blobs` holds `logoRaster` and `favicon` as Blobs.
  - Writes are debounced by 500 ms and wrapped in try/catch, because storage can be missing (private modes, quota).
  - localStorage holds only `bk.theme` and `bk.pageSize`.
  - localStorage is limited to 5 MiB per origin; IndexedDB stores Blobs natively. [Certain] (MDN storage quotas)
  - No IDs, no sync, no cookies, no service worker (PRIV-12, PRIV-13).
- **OUT-35. Never persist:**
  - uploaded brand-guide PDFs;
  - "image → palette" source images (keep only the resulting colors);
  - fetched HTML and CSS;
  - Turnstile tokens;
  - the source URL's query string.
- **OUT-36. Microcopy:** "Your draft is saved in this browser only. Safari may clear it after 7 days without a visit."
  - Safari's tracking prevention deletes all script-writable storage (IndexedDB, localStorage and others) after 7 days of Safari use without user interaction on the site. [Certain] (webkit.org/tracking-prevention)
- **OUT-37. One-click "Clear my data" button:**
  1. Revoke all `blob:` URLs.
  2. Close the database.
  3. `indexedDB.deleteDatabase('bk')`.
  4. Remove the `bk.*` localStorage keys.
  5. `sessionStorage.clear()`.
  6. Reset in-memory state.
  7. Navigate to `/clear`. Pages serves `/clear` with `Clear-Site-Data: "storage"` (Chrome 61+, Firefox 63+, Safari 17+) as a backstop.
  - Do **not** send `"cache"`: Chrome documents hangs of several seconds. [Certain] (BCD notes)
  - The UI must say: "Links you already shared still work, because the kit lives inside the link."

### G. Schema

- **OUT-38. Adopt the schema in section 3.** It has two layers:
  - `KitDocV1`, the local superset.
  - `ShareV1`, a short-key wire format.

  Version rules are in 3.3.
  - Share links carry **inputs only**: name, tagline, colors, two fonts, type-scale settings, logo SVG and logo rules.
  - Everything derived (ramps, contrast grid, approved pairs, type steps, fallback stacks) is recomputed. The algorithm version is pinned by field `a`.

---

## 2. Detailed findings

### 2.1 PDF engines compared

Sizes are minified ESM bundles, gzip -9, measured 2026-11-08. "Kerning" means GPOS pair kerning in the output, observed in the content stream with Inter TTF and the text "AVATAR Type To Wa fi".

| Engine (version, publish date, licence) | gzip size | Font input that worked | Subsetting (measured) | Kerning | SVG | Notes |
|---|---|---|---|---|---|---|
| **PDFKit 0.20.2** (2026-09, MIT) + **svg-to-pdfkit 0.1.8** (2022-06, MIT) | 213 KB + 19.5 KB | TTF, WOFF1. WOFF2 failed. | Yes, 3.7–6.9 KB per face, name tag `CZZZZZ+` | **Yes** (`TJ` with offsets) | Full SVG 1.1 subset via svg-to-pdfkit; drew Illustrator `<style>` SVG correctly | Tagged PDF and PDF/UA options in docs; runs without a DOM. svg-to-pdfkit has not been updated since 2022. [Certain] (measured) |
| pdf-lib 1.17.1 (**2021-11**, MIT) + @pdf-lib/fontkit 1.1.1 (2020-11) | 180 KB + 383 KB = **548 KB** | TTF, WOFF1 with `subset:true` only | Yes for TTF/WOFF1 (3.8 KB) | No (`Tj` only) | None built in (`drawSvgPath` takes one path) | WOFF2 subset throws an **uncatchable** async RangeError. `subset:false` with WOFF/WOFF2 embeds invalid font bytes. No release in about 5 years. [Certain] (npm, measured) |
| @cantoo/pdf-lib 2.11.1 (2026-09, MIT) | 251 KB (+ fontkit) | TTF, WOFF1 | Yes | Not tested; assume no | `drawSvg` exists but **silently drew nothing** for a raw Illustrator SVG and a simple-icons SVG; worked only on SVGO output | Maintained fork. [Certain] (measured) |
| jsPDF 4.2.1 (2026-03, MIT) + svg2pdf.js 2.9.0 (2026-10, MIT) | 134 KB + 26 KB = **160 KB** (html2canvas, DOMPurify and canvg split into lazy chunks) | **TTF only**. WOFF fails with "No unicode cmap" and produces a PDF with a broken non-embedded font. | Partial: 83.5 KB embedded for one short line of Inter (325 KB TTF) | No (`Tj` only) | svg2pdf.js is mature, needs a DOM, main thread only | Set `compress:true` (default off: 88 KB → 41 KB). [Certain] (measured) |
| pdfmake 0.3.14 (2026-11-08, MIT) | 299 KB (bundles PDFKit) | Same as PDFKit | Yes (PDFKit) | Yes (PDFKit) [Likely] | Built-in `svg` node (svg-to-pdfkit) | Declarative layout with good tables; bundle contains `new Function` (CSP concern). [Certain] on size and grep |
| @react-pdf/renderer 4.9.0 (2026-08, MIT) | 453 KB **+ React** | Docs v4: "only TTF and WOFF"; package README also claims WOFF2 | Yes (fontkit) | Yes [Likely] | Own `<Svg>` primitives, so a raw SVG string must be converted | Heaviest option; flexbox layout. [Certain] on size; [Likely] on fonts |
| HTML + print CSS (`window.print()`), optionally Paged.js 0.4.3 (**2023-07**, MIT, 102 KB) | 0 KB / 102 KB | Anything the browser renders, including variable fonts | Browser-dependent | Yes | Native | No one-click file. User needs the print dialog; on iOS: Share sheet → Save to Files. `@page` `size` only since Safari 18.2. Margin boxes are Chrome 131+ only. Backgrounds need `print-color-adjust: exact` (unprefixed only in Chrome 136+). Chrome adds headers/footers unless the user turns them off. Output varies by browser. [Certain] (BCD) |

**Decision: PDFKit plus svg-to-pdfkit.** It is the only option that:
- produces kerned, properly subset type,
- draws real-world SVG exports as vectors,
- runs in a worker,
- stays near 230 KB gzip,
- and avoids `eval`.

The design directives (asymmetric editorial layouts, a dominant headline, real size contrast) suit absolute positioning with a small layout helper, which is how PDFKit works. pdfmake is the fallback if a declarative layout is preferred; it costs about 70 KB more and has the `new Function` CSP issue. [Likely]

**Gotchas found while testing:**
- svg-to-pdfkit 0.1.8 is unmaintained but small and stable. Pin it, vendor it if needed, and only ever feed it canonical SVG (OUT-31). Its parsing surface is then limited to what we emit. [Likely]
- Variable fonts: a PDF embeds one instance. Embedding a variable font shows its default instance, so the kit must name static weights. Serve static instances for screen too, so preview matches PDF. [Likely]
- Gradient logos render as vectors through svg-to-pdfkit. Any SVG feature the canonicalizer drops (filters, masks, patterns) falls back to embedding a 4× PNG raster made with Canvas. [Likely]
- Raster logos: PDFKit embeds PNG and JPEG. Convert WebP or anything else to PNG with Canvas first. [Likely]

**Content per page.** This is template guidance, not a locked decision; the kit contents are locked elsewhere.
1. Cover: name, tagline, logo, date.
2. Logo: on light, on dark, one-color, clear-space diagram, minimum size, don'ts.
3. Palette: HEX, RGB, HSL and role.
4. Tint and shade ramps.
5. WCAG contrast grid and approved pairs.
6. Typography: specimens, type scale, source, licence, fallbacks.
7. Colophon: "Made with {Tool}", font licences, generation date.

### 2.2 Mobile download behavior

- `a[download]` is supported in Safari iOS 13+, Chrome Android 18+ and Firefox Android 20+. Since Chrome 65, cross-origin downloads are not honored, which does not affect same-origin `blob:` URLs. [Certain] (BCD)
- iOS Safari sometimes opens an `application/pdf` blob inline instead of downloading it. A common workaround is the `application/octet-stream` type for the download blob. [Likely] (StackOverflow 55760169, secondary)
- `navigator.share` with files: Safari and iOS 14+, Chrome Android 76+, Samsung Internet 11+. On desktop Chrome, Windows and ChromeOS only. Not in Firefox. [Certain] (BCD)
- WKWebView-based in-app browsers ignore `download` on `blob:` hrefs (WebKit bug 216918). Android WebView does not forward `blob:` downloads (docuseal#719, 2026). [Likely]
- `showSaveFilePicker` is Chromium-only. Skip it. [Certain] (BCD)

### 2.3 PNG brand board options

| Approach | Size | Fidelity and risks | Verdict |
|---|---|---|---|
| **Canvas 2D, hand-drawn layout** | 0 KB | Full control. Fonts via `FontFace` + `document.fonts.load`. No taint if all images are `blob:`. Safari lacks `letterSpacing` (<18.4), `fontKerning` and `fontStretch` [Certain] (BCD). Kerning is applied by default text rendering [Likely]. | **Use** |
| html-to-image 1.11.13 (2025-02) / modern-screenshot 4.7.0 / @zumer/snapdom 3.7.1 | 5.2 / 9.6 / 103 KB | SVG `foreignObject` serialization. Fonts must be inlined as data URLs. Cross-origin images taint the canvas or are dropped. Safari: blank first call, missing images, no SVG `<img>` (html-to-image #461, #361, #488) [Likely] | Avoid |
| satori 0.63.2 + @resvg/resvg-wasm 2.6.2 | 685 KB JS + 2.48 MB wasm (949 KB gzip) | Subset of CSS. Needs TTF/OTF/WOFF; the README now says WOFF2 is also supported, older guides say no [Likely]. Needs `wasm-unsafe-eval` in CSP. MPL-2.0. | Avoid (size) |

Board layout at 2×: canvas 3200×2000 = 6.4 MP. iOS Safari refuses canvases over 16,777,216 px. Total canvas memory limits of 224–576 MB are reported per device. [Likely] (pqina.nl; react-pdf#1020)

### 2.4 Getting font binaries

**What each source serves (observed 2026-11-08):**

| Source | What it serves | CORS | Static vs variable | Privacy |
|---|---|---|---|---|
| Google Fonts CSS2 (`fonts.googleapis.com/css2`) | WOFF2 to modern UAs. Full static TTF to a non-browser UA (curl). WOFF to an IE11 UA. | `access-control-allow-origin: *` on CSS and on `fonts.gstatic.com` files [Certain] (curl) | **Variable families get the same variable file for every weight**: Inter 400 and 700 resolved to the same URL [Certain] (curl). You cannot set the UA from browser JS, so the browser path always gets variable WOFF2. | Visitor IP goes to Google. LG München I, 20 Jan 2022, 3 O 17493/20, awarded €100 (see privacy report) [Certain] |
| Google Fonts Developer API (`googleapis.com/webfonts/v1/webfonts?key=`) | JSON with per-variant **static TTF** URLs (full script coverage). `capability=VF` returns variable files; `capability=WOFF2` returns WOFF2. | n/a (build time) | Static by default ("instantiated at standard positions") | Build time only, so no visitor data [Certain] (docs) |
| Fontsource via jsDelivr (`cdn.jsdelivr.net/fontsource/fonts/{id}@{ver}/{subset}-{weight}-{style}.{ext}`) | WOFF2, WOFF and **TTF per subset**. The API for Inter 5.3.0 lists `woff2`, `woff` and `ttf` for every subset, weight and style. npm packages ship only WOFF and WOFF2. | Not testable here (jsDelivr blocked by the sandbox) [Likely] `*` | Static per weight; variable at `{id}:vf@…` | Visitor IP goes to jsDelivr. Fontsource itself recommends self-hosting [Certain] (fontsource.org CDN doc) |
| Bunny Fonts (`fonts.bunny.net/css`) | Compatible with the Google CSS **v1** API. Static per-weight WOFF2 + WOFF per subset, using Fontsource-style names | Not testable here [Guessing] | Static | EU company, claims "zero-tracking and no-logging" [Certain] that they claim it; [Likely] that it is true |
| **Own origin (recommended)** | Static WOFF1, Latin + Latin-Ext, one file per style, built at deploy time | Same-origin, so no CORS needed | Static | No third party; matches PRIV-01/02 |

**Format compatibility for the PDF engine:** WOFF2 fails (OUT-03). TTF and WOFF1 work. The browser preview accepts all of them. One WOFF1 file therefore serves both uses. Measured Latin+Latin-Ext subset sizes: Inter 400 = 62.4 KB WOFF (TTF+brotli 53.8 KB); Playfair Display 700 = 37.6 KB; Roboto 400 = 30.7 KB; Lato 400 = 32.1 KB; Montserrat 700 = 52.7 KB; Open Sans 400 = 29.8 KB. [Certain] (measured)

**Proxying through our own endpoint:** not recommended. It spends the Function quota, adds a server data flow, and buys nothing over static self-hosting. [Certain] on quota; [Likely] on the rest.

**OFL notes relevant to the pipeline:**
- OFL-FAQ 1.10: the licence text need not accompany a font that is embedded in a document. A redistributed font file needs the licence, either as a separate file or in metadata. A link alone is legally sufficient but discouraged.
- Embedding a full font or a subset in PDFs is allowed.

[Certain] (openfontlicense.org/ofl-faq, update7 Nov 2023). RFN and web-subsetting questions are in the IP report (T-*).

### 2.5 Share-link compression and payload size

The test kit had:
- name "Rowlett Roofing & Restoration" (29 characters);
- a 57-character tagline;
- 5 colors with names;
- 2 fonts with 2 weights each;
- a type scale;
- logo rules.

The JSON uses short keys. The logo variants came from my own outlined-text wordmark: Playfair Display path outlines, an icon, a two-color fill, and an Illustrator-style export run through SVGO 4.1.0.

**SVG sizes (raw → SVGO):**
- icon: 581 B → 163 B;
- wordmark: 14,878 B → 6,293 B at `floatPrecision 2`, or 4,168 B at precision 1;
- wordmark + outlined tagline: 30,680 B → 11,607 B at precision 2, or 7,783 B at precision 1.

For simple-icons (3,470 single-path icons), file sizes are: p50 887 B, p90 2,844 B, p99 9,384 B. [Certain] (measured)

**Encoded fragment length in characters.** Lower is better.

| Case | JSON bytes | CompressionStream deflate-raw → b64url | lz-string `compressToEncodedURIComponent` | fflate zlib, no dict → b64url | **fflate zlib + D1 → b64url** | Full URL (31-char host) |
|---|---|---|---|---|---|---|
| No logo | 303 | 322* | 371* | 310 | **263** | **~300** |
| + icon SVG 163 B | 514 | 494* | 567* | 506 | **372** | ~410 |
| + simple-icons median (822 B) | 1,171 | 932* | 1,113* | 947 | **804** | ~840 |
| + wordmark SVGO p1 (4.2 KB) | 4,523 | 2,632* | 3,085* | 2,715 | **2,523** | ~2,560 |
| + wordmark SVGO p2 (6.3 KB) | 6,648 | 3,312* | 4,482* | 3,452 | **3,248** | ~3,285 |
| + wordmark+tagline p1 (7.8 KB) | 8,142 | 4,107* | 5,166* | 4,275 | **4,067** | ~4,100 |
| + wordmark+tagline p2 (11.6 KB) | 11,966 | 5,744* | 7,606* | 6,043 | **5,814** | ~5,850 (over the 8 KB SVG cap) |

\* These columns come from the first run, with a slightly longer JSON (a nested `colors` object). Compare them with each other only. [Certain] (measured)

Takeaways:
- lz-string 1.5.0 (2023-03, MIT, 1.7 KB) was 15–35% longer than deflate in every case with a logo. Its alphabet also includes `+` and `$`. Do not use it. [Certain]
- Brotli at quality 11 would save another ~10–15%, but browsers cannot produce it natively (Chrome lacks it). [Certain] (BCD)
- On tiny payloads, deflate barely beats raw JSON length. The preset dictionary is what makes the no-logo link short: 310 → 263 characters. [Certain]
- `Uint8Array.prototype.toBase64({alphabet:'base64url'})` needs Chrome 140, Firefox 133 or Safari 18.2. Feature-detect it, and fall back to a 15-line encoder. [Certain] (BCD)

**Preset dictionary D1 (illustrative; freeze it at launch):**
- the JSON skeleton with every short key;
- the ~40 most-used Google family names written as `["Name",[400,700]]`;
- SVG boilerplate: `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 `, `<path fill="#`, `d="M`.

Order it so the most likely strings come last. Keep it under 4 KB and ship it as a constant in both encoder and decoder bundles. The test D1 was 1,241 bytes. [Certain] that fflate supports dictionaries (CHANGELOG 0.8.0, typings `dictionary?: Uint8Array`)

**Why zlib and not raw deflate:** zlib adds 6 bytes (8 characters), and its Adler-32 checksum (RFC 1950) turns silent corruption into a clean error. [Certain] (RFC 1950); [Likely] on the UX value

### 2.6 Where long links break

| Surface | Limit (source) | Effect on our tiers | Confidence |
|---|---|---|---|
| Chrome | URLs up to **2 MB**; omnibox displays up to 32 KB (Chromium URL display guidelines) | No issue | [Certain] |
| Firefox | Address bar stops displaying after 65,536 chars; longer URLs still work (Mozilla support, secondary) | No issue | [Likely] |
| Safari | No official limit; about 80,000 chars reported (secondary) | No issue | [Likely] |
| Cloudflare edge (only if the fragment is rewritten into the path) | URL limit 16 KB (Cloudflare docs) | OK up to the 6,000-char cap | [Certain] |
| Slack | Message text truncated at 40,000 chars (Slack changelog 2018; chat.postMessage docs) | OK | [Certain] |
| Discord | 2,000 chars per message for standard users, 4,000 with Nitro (Discord support "Account Caps") | Logo links over ~1,900 fail for most users | [Certain] |
| LinkedIn post | 3,000 chars (LinkedIn Help a528176) | Wordmark links at 2,560–3,285 are borderline or fail | [Certain] on limit; [Likely] that URLs count in full |
| X | Every URL counts as 23 chars via t.co (X counting-characters docs) | OK for counting; t.co may impose a maximum URL length | [Certain] on the 23; maximum unverified |
| iMessage / SMS | No published limit. Green-bubble SMS fallback splits long texts. | Test before claiming | [Guessing] |
| Gmail / Outlook | No published URL cap found. Plain-text mail may wrap long lines. **Safe Links rewrites `#` → `%23`** (OUT-25). | Recovery handled by OUT-25 | [Likely] |
| QR codes | Version 40-L holds 2,953 bytes. Measured: ~340-char URL → v12 (L) / v14 (M); ~540 → v16/v18; ~1,040 → v23/v26; ~2,040 → v33 (L); ~2,940 → v40 (L); >3,300 → does not fit | QR for no-logo links only | [Certain] (qrcode 1.5.4) |

**Recommended budget (OUT-24):**
- Target ≤ 1,900 characters for the default copy.
- Refuse above 6,000.
- Always offer the no-logo link, which is ≤ ~450 characters and QR-friendly.

### 2.7 Link previews and the OG card

- The server never sees the fragment (RFC 3986 §3.5; it is also stripped from `Referer`), so every unfurler gets the same static `/k` page. [Certain]
- Messages does not run JavaScript for previews. The other unfurlers fetch HTML server-side. [Certain] for Messages (TN3156); [Likely] for Slack, LinkedIn and X.
- OG image spec: 1200×630 PNG, under 300 KB, flat neutral background, one accent, the tool name, and the line "Brand kit. Open the link to view. Made by a visitor; not verified by any brand." This follows the owner's design directives. It is at least 900 px wide for Messages and 1.91:1 for LinkedIn and X. [Likely]
- Following TN3156, put the site name in `og:site_name`, not in `og:title`. [Certain]

### 2.8 Security details

1. **Fragment payload threats:** script injection through the SVG or text, CSS injection through font names or colors, phishing copy, bidi spoofing, decompression bombs, entity bombs, prototype pollution, and oversized input that freezes the tab. OUT-27 to OUT-33 cover each one. [Likely]
2. **Why rebuild SVG from an allowlist:**
   - DOMPurify's 2025–2026 advisory history includes mXSS, prototype pollution, hook bypasses and `selectedcontent` re-cloning. Patched versions run up to 3.4.16 and 3.4.17. [Certain] (GitLab advisory DB; GHSA-h8r8-wccr-v5f2)
   - An allowlist serializer cannot emit anything it does not know.
   - `<img>` rendering is a second wall: in image mode SVG has JavaScript disabled and cannot load external resources. [Certain] (MDN)
3. **CSP and Trusted Types:**
   - Trusted Types became Baseline in February 2026. With `require-trusted-types-for 'script'; trusted-types 'none'`, any `innerHTML`-style sink throws, so the share viewer cannot regress into DOM XSS. [Certain] (MDN TT page "Since February 2026"; BCD)
   - Framework caveat: some compilers emit `innerHTML` for templates; Svelte 5 is suspected to. Write the share viewer in vanilla TS or Preact, and check the framework under TT before committing. [Guessing] on Svelte
   - Setting styles through the CSSOM (`el.style.setProperty`) is not blocked by `style-src 'self'`; inline `<style>` and `style=""` attributes are. [Likely]
   - `FontFace` built from an `ArrayBuffer` does no fetch, so `font-src` does not apply to it. [Likely]
4. **`img-src blob:` instead of `data:`:** render logos with `URL.createObjectURL(new Blob([svg],{type:'image/svg+xml'}))`, which allows dropping `data:`. That is one fewer scheme than S-5 allows. [Likely]
5. **Blocklist hashing:** different encoders or dictionaries produce different bytes for the same kit. Hash **canonical JSON** (keys sorted, normalized values), plus normalized brand names and domains. [Likely]
6. **Error states on `/k`:**

   | Condition | Message |
   |---|---|
   | No fragment | "This link has no kit in it." |
   | Unknown version | "This kit was made with a newer version of {Tool}. Reload the page." |
   | Length or Adler mismatch | "This link looks cut off. Ask the sender to copy it again." |
   | Validation failure | "This link can't be displayed." |
   | Blocked | "This kit link has been disabled." |

7. **Where the link really travels:** the fragment is not sent in HTTP requests. The full URL still lives wherever the user pastes it: chat and mail providers, browser history and history sync, and mail-gateway logs. The share copy must say "Anyone with the link can see the kit." [Likely]

### 2.9 Local persistence details

- **Quotas:** localStorage is 5 MiB per origin. IndexedDB can use up to about 60% of disk in Chromium and WebKit. Private modes may apply different quotas and clear data at session end. [Certain] (MDN)
- **Safari ITP:** 7-day deletion of IndexedDB, localStorage, sessionStorage, media keys and service-worker caches when there has been no interaction. Home-screen web apps are exempt. [Certain] (webkit.org)
- **`navigator.storage.persist()`:** do not call it. Firefox shows a permission prompt, and the draft is not critical data. [Guessing] (product choice)
- **Multiple tabs:** last write wins. Optionally, a `BroadcastChannel('bk')` tells other tabs that the draft changed. [Guessing]
- **Opening a share link** never writes storage by itself. The "Edit a copy" button imports the kit into the draft, asking first if a draft already exists. [Likely]

### 2.10 Bundle and loading plan

| Chunk | When loaded | gzip (measured) |
|---|---|---|
| App shell + extraction UI | First load | Target ≤ 60 KB [Guessing] |
| Share codec (fflate zlib+unzlib) + valibot schema | When the Share button becomes enabled; idle prefetch | 5.0 + ~1.7 KB |
| Share viewer `/k` | On visiting a link | Target ≤ 30 KB (codec decode 2.3 KB + valibot + renderer) [Guessing] |
| PDF worker (PDFKit + svg-to-pdfkit + templates) | On hover or focus of "Download PDF"; skipped if `navigator.connection.saveData` | 213 + 19.5 KB + templates |
| SVGO (`svgo/browser` 4.1.0) | Only when an SVG is uploaded or extracted; run in a worker | 166 KB |
| Font catalog JSON | When the font picker opens | ~25 KB [Guessing] |
| Font files | Per family on demand (WOFF ~30–62 KB each, menu subsets ~1–3 KB [Guessing]) | — |

Support floor implied by these choices: Safari/iOS 16.4+ (March 2023), Chrome 103+, Firefox 113+. String `isWellFormed`, `OffscreenCanvas` and CompressionStream (if used) need it. Below the floor, show a "please update your browser" note instead of shipping polyfills. [Likely]

### 2.11 Conflicts and cross-track notes

1. **Hard conflict with the "never sent to a server" promise.** Safe Links-style rewriting (`#` → `%23`) puts the share payload, including any user-uploaded SVG logo, into the request path that reaches Cloudflare.
   - OUT-25 restores the link, but cannot undo the exposure.
   - Privacy copy should say "never sent to our server when opened normally" and disclose the mail-gateway case in the privacy notice.
   - This does **not** affect uploads in the editor; they still never leave the device. [Likely]
2. **Text caps.** The IP report (S-7) caps the name at 60 and the tagline at 120. The privacy report (PRIV-17) suggests 80 and 160. I used **60/120**: stricter, and enough for real brand names.
3. **SVG cap.** S-7's "~8 KB" is a guess; my measurements support it. A realistic outlined wordmark is 4.2 KB at precision 1 and 6.3 KB at precision 2. A wordmark with an outlined tagline is 7.8 KB at precision 1, or 11.6 KB at precision 2, which is over the cap. Offer a "Simplify logo" step (precision 1) before falling back to "share without logo".
4. **Font hosting vs "same origin only".** The full mirror fits Pages limits (OUT-19). No second origin or CDN is needed, so PRIV-01 holds literally.
5. **Reserved Font Names** (OUT-20) need sign-off from the IP and legal track.

---

## 3. Proposed kit JSON schema v1

### 3.1 `KitDocV1`: local document (IndexedDB `bk/draft/current`)

| Field | Type and constraint | Share link? | Notes |
|---|---|---|---|
| `schema` | literal `"bk.kit/1"` | no (the version is in the URL prefix) | |
| `savedAt` | ISO 8601 string, minute precision | no | Display only ("Draft from …"). Not an identifier (PRIV-12). |
| `brand.name` | string, 1–60 graphemes, OUT-28 rules | **yes** `n` | Required |
| `brand.tagline` | string, 0–120 graphemes | **yes** `t` | Optional |
| `source.kind` | enum `url` \| `upload` \| `manual` | no | |
| `source.url` | string ≤ 2,048, scheme+host+path only (no query or fragment) | no | Kept out of links: no impersonation aid, no privacy leak |
| `colors[]` | array, 1–8 items, unique `hex` | **yes** `c` | |
| `colors[].hex` | `^#[0-9a-f]{6}$` | yes (without `#`) | No alpha |
| `colors[].role` | enum, fixed index order: 0 `primary`, 1 `secondary`, 2 `accent`, 3 `background`, 4 `text`, 5 `neutral`, 6 `other` | yes (index) | At most one each of primary, background and text. **Append-only enum.** |
| `colors[].name` | string, 0–32 graphemes | yes (optional) | |
| `colors[].origin` | enum `site` \| `logo` \| `image` \| `guide` \| `manual` | no | Provenance for the UI |
| `colors[].locked` | boolean | no | User override flag |
| `fonts.heading`, `fonts.body` | `FontRef` | **yes** `h`, `b` | Both required; may name the same family |
| `FontRef.family` | string 1–64. An exact catalog family name, or display-only text matching `^[\p{L}\p{N} '&.-]{1,64}$`u | yes | Exact match against `catalog.json` decides whether it is loadable |
| `FontRef.source` | enum, index order 0 `google`, 1 `bunny`, 2 `adobe`, 3 `self-hosted`, 4 `system`, 5 `unknown` | yes (index) | The load-source label, as in the competitor |
| `FontRef.weights` | int[] 1–4, each in {100,…,900} step 100, sorted, unique | yes | Snap to the nearest mirrored instance when rendering |
| `FontRef.italic` | boolean | yes (0/1) | |
| `FontRef.category` | enum `sans-serif` \| `serif` \| `monospace` \| `display` \| `handwriting` | no (catalog provides it; display-only fonts default to `sans-serif`) | Drives the fallback stack |
| `type.base` | number 12–24 (px), step 0.5 | **yes** `y[0]` | Default 16 |
| `type.ratio` | enum {1.125, 1.2, 1.25, 1.333, 1.414, 1.5, 1.618} | **yes** `y[1]` | Default 1.25 |
| `algo` | int ≥ 1 | **yes** `a` | Pins the ramp, contrast and type-step algorithms; default 1 |
| `logo.svg` | canonical SVG string (OUT-31), ≤ 65,536 bytes locally | **yes** `l.s`, only if ≤ 8,192 bytes | Raster logos never ride |
| `logo.raster` | `{blobKey:'logoRaster', mime: png\|jpeg\|webp, w ≤ 4096, h ≤ 4096, bytes ≤ 5 MB}` | **no** | IndexedDB `blobs` |
| `logo.usage.clearSpace` | number 0.1–2.0 (× logo height) | **yes** `l.c` | |
| `logo.usage.minPx` | int 16–512 | **yes** `l.p` | |
| `logo.usage.minMm` | number 5–100 | **yes** `l.m` | |
| `logo.usage.bg` | enum index 0 `auto`, 1 `light`, 2 `dark` | **yes** `l.g` | Preferred background for the primary lockup |
| `logo.donts` | array of enum (`stretch`, `recolor`, `rotate`, `effects`, `busy-bg`, `outline`, `rearrange`), ≤ 8 | no (a fixed template set is rendered) | Could join links in a later minor version |
| `favicon` | `{blobKey:'favicon', mime, size}` | no | |
| `export.pageSize` | enum `letter` \| `a4` | no | |

**Derived, never stored and never shared:**
- tint and shade ramps;
- the contrast matrix and AA/AAA flags;
- approved text-on-background pairs;
- type-scale steps;
- fallback stacks;
- light, dark and one-color logo variants.

### 3.2 `ShareV1`: wire JSON inside `#1.<payload>.<len>`

```jsonc
{
  "n": "Rowlett Roofing & Restoration",          // required
  "t": "Storm damage repair and full roof replacement across DFW", // optional
  "c": [["c8102e",0,"Signal Red"],["1b2a41",1],["f2a900",2],["fafaf7",3],["14171a",4]], // 1–8
  "h": ["Playfair Display",[700,900],0,0],       // [family, weights, italic 0|1, sourceIdx]
  "b": ["Inter",[400,600],0,0],
  "y": [16,1.25],                                 // optional
  "a": 1,                                         // optional, default 1
  "l": {"s":"<svg xmlns=…>…</svg>","c":0.5,"p":96,"m":25,"g":0} // optional; "s" optional inside
}
```

Measured: about 263 characters encoded without `l`, and about 2,500–3,250 with a typical wordmark (2.5). [Certain]

### 3.3 Versioning and forward-compatibility rules

1. The **URL prefix** (`1.`) is the *format major*. It fixes the codec (zlib), dictionary D1, the base64url alphabet and the key meanings. Decoders must keep every past major forever, because links are permanent. [Likely]
2. **Within a major, only additive changes:**
   - new **optional** keys (decoders strip unknown keys);
   - new enum values appended at the end (an unknown index maps to `other` or `unknown`);
   - no changes to existing key meanings, units or caps that would reject old links.
3. **`a`, the algorithm version,** pins derived output. A new ramp or contrast algorithm bumps `a` for new links; the renderer keeps old algorithms so old links look the same. [Likely]
4. **Encoders emit canonical JSON:** a fixed key order, no defaults (omit `a:1`, `y` when default, `t` when empty), and integers where possible. Canonical output keeps links short, makes blocklist hashes stable and makes tests deterministic. [Likely]
5. **Golden tests:** commit 10–20 fixture links per major, covering edge cases (max lengths, RTL text, gradient SVG, every enum). CI decodes every fixture on each release. [Likely]

### 3.4 Validation sketch (valibot 1.5.0)

```ts
import * as v from 'valibot';
const Hex = v.pipe(v.string(), v.regex(/^[0-9a-f]{6}$/));
const Txt = (max: number) => v.pipe(v.string(), v.transform(normalizeText), v.check(s => graphemes(s) <= max));
const Font = v.tuple([v.pipe(v.string(), v.maxLength(64)),
  v.pipe(v.array(v.picklist([100,200,300,400,500,600,700,800,900])), v.minLength(1), v.maxLength(4)),
  v.picklist([0,1]), v.pipe(v.number(), v.integer(), v.minValue(0))]);
export const ShareV1 = v.object({
  n: v.pipe(Txt(60), v.minLength(1)),
  t: v.optional(Txt(120)),
  c: v.pipe(v.array(v.union([v.tuple([Hex, v.number()]), v.tuple([Hex, v.number(), Txt(32)])])), v.minLength(1), v.maxLength(8)),
  h: Font, b: Font,
  y: v.optional(v.tuple([v.pipe(v.number(), v.minValue(12), v.maxValue(24)), v.picklist([1.125,1.2,1.25,1.333,1.414,1.5,1.618])])),
  a: v.optional(v.pipe(v.number(), v.integer(), v.minValue(1))),
  l: v.optional(v.object({ s: v.optional(v.pipe(v.string(), v.maxLength(8192), v.check(isCanonicalSvg))),
    c: v.optional(v.pipe(v.number(), v.minValue(0.1), v.maxValue(2))), p: v.optional(v.pipe(v.number(), v.integer(), v.minValue(16), v.maxValue(512))),
    m: v.optional(v.pipe(v.number(), v.minValue(5), v.maxValue(100))), g: v.optional(v.picklist([0,1,2])) })),
});
```

`isCanonicalSvg` re-runs the OUT-31 canonicalizer and requires the output to equal the input byte for byte. That rejects anything the allowlist would change. [Likely]

---

## 4. Unverified / needs follow-up

1. **iMessage/SMS, X (t.co maximum), Gmail and Outlook behavior with 2,500–6,000-character links.** Send real test links through each, including Outlook with Safe Links on and off, Gmail web and iOS Mail. Record truncation, wrapping and whether `#` survives. Proofpoint and Mimecast rewriting is unknown; the SkySend issue says some gateways drop fragments entirely. [Guessing]
2. **Whether LinkedIn counts a URL's full length against the 3,000-character limit**, or shortens it (lnkd.in). [Guessing]
3. **Firefox and Safari URL maxima** come from secondary sources only. They are irrelevant at our sizes.
4. **The WOFF2 failure in fontkit** was seen with Inter (Fontsource and Google files) and in @pdf-lib/fontkit and fontkit 2.0.4. @cantoo/fontkit 2.0.12 succeeded on Playfair Display WOFF2 but failed on Inter. I did not find the root cause; possibly the transformed `glyf` table. Not needed if OUT-03 is followed.
5. **@cantoo/pdf-lib `drawSvg` failures** were not investigated beyond the observation. Irrelevant if PDFKit is chosen.
6. **Kerning in @react-pdf/renderer and pdfmake** is inferred from their fontkit and PDFKit bases, not measured.
7. **iOS Safari PDF blob download:** whether `application/pdf` opens inline versus downloads on iOS 18/26, and whether the `application/octet-stream` workaround is still needed. Test on a device.
8. **Canvas behavior on real devices:** `toBlob` PNG sizes for the 3200×2000 board in Chrome and Safari (Pillow gave about 130 KB), and Safari's per-device total canvas memory limit.
9. **Font mirror totals:** the real byte total and file count for all 7,501 variants (the 300 MB figure is extrapolated from 6 families). Also the Wrangler Direct Upload time for about 11,300 files, and whether unchanged files are skipped on redeploy.
10. **Google Fonts Developer API quota and key restrictions** (also flagged by the infra report). Also whether a family rename (for example "Source Sans Pro" → "Source Sans 3") breaks exact-name matching in old share links. Plan: keep a rename alias table in `catalog.json`.
11. **Reserved Font Name families on Google Fonts:** how many exist, and whether build-time subsetting of them is acceptable. This needs the legal track.
12. **CSP details:** whether Turnstile needs extra `style-src` or `img-src` sources, and whether the chosen UI framework works under `require-trusted-types-for 'script'`.
13. **`font-src` and `FontFace(ArrayBuffer)`:** confirm in all three engines that no CSP violation is reported.
14. **Firefox drawing of viewBox-only SVG to canvas** (OUT-13) is from memory, not re-tested today.
15. **Bunny Fonts and jsDelivr CORS headers** could not be fetched from this sandbox. They are not needed under the recommendation.
16. **PDF/UA conformance:** PDFKit's tagged output was not validated with veraPDF or PAC.

---

## 5. Sources

**Libraries and packages** (npm registry, versions and dates read 2026-11-08):
- https://www.npmjs.com/package/pdfkit
- https://pdfkit.org/docs/text.html
- https://pdfkit.org/docs/accessibility.html
- https://github.com/alafr/SVG-to-PDFKit
- https://www.npmjs.com/package/pdf-lib
- https://github.com/Hopding/pdf-lib
- https://www.npmjs.com/package/@cantoo/pdf-lib
- https://github.com/parallax/jsPDF
- https://github.com/yWorks/svg2pdf.js
- https://www.npmjs.com/package/pdfmake
- https://react-pdf.org/fonts
- https://github.com/diegomura/react-pdf/blob/master/packages/font/README.md
- https://www.npmjs.com/package/pagedjs
- https://github.com/vercel/satori
- https://www.npmjs.com/package/@resvg/resvg-wasm
- https://github.com/bubkoo/html-to-image/issues/461
- https://github.com/bubkoo/html-to-image/issues/361
- https://github.com/101arrowz/fflate (CHANGELOG 0.8.0, dictionaries)
- https://www.npmjs.com/package/lz-string
- https://valibot.dev
- https://www.npmjs.com/package/zod
- https://github.com/svg/svgo
- https://www.npmjs.com/package/dompurify
- https://advisories.gitlab.com/pkg/npm/dompurify/
- https://github.com/cure53/DOMPurify/security/advisories/GHSA-h8r8-wccr-v5f2
- https://www.npmjs.com/package/google-font-metadata
- https://fonttools.readthedocs.io/en/latest/subset/

**Browser support:**
- MDN browser-compat-data 8.1.8 (npm `@mdn/browser-compat-data`, published 2026-11-02): https://github.com/mdn/browser-compat-data
- https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria
- https://developer.mozilla.org/en-US/docs/Web/SVG/Guides/SVG_as_an_image
- https://developer.mozilla.org/en-US/docs/Web/API/Trusted_Types_API
- https://www.firefox.com/firefox/148.0/releasenotes/
- https://webkit.org/tracking-prevention/
- https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/
- https://bugs.webkit.org/show_bug.cgi?id=216918
- https://github.com/docusealco/docuseal/issues/719
- https://pqina.nl/blog/canvas-area-exceeds-the-maximum-limit/
- https://stackoverflow.com/questions/55760169/alternative-for-download-attribute-in-safari-ios

**URLs, links and previews:**
- https://www.rfc-editor.org/rfc/rfc3986#section-3.5
- https://www.rfc-editor.org/rfc/rfc4648#section-5
- https://www.rfc-editor.org/rfc/rfc1950
- https://chromium.googlesource.com/chromium/src/+/HEAD/docs/security/url_display_guidelines/url_display_guidelines.md
- https://stackoverflow.com/questions/417142/what-is-the-maximum-length-of-a-url-in-different-browsers
- https://support.mozilla.org/en-US/questions/924935
- https://docs.slack.dev/changelog/2018-truncating-really-long-messages
- https://support.discord.com/hc/en-us/articles/33694251638295-Discord-Account-Caps-Server-Caps-and-More
- https://docs.x.com/fundamentals/counting-characters
- https://www.linkedin.com/help/linkedin/answer/a528176
- https://github.com/Skyfay/SkySend/issues/68
- https://learn.microsoft.com/en-us/defender-office-365/safe-links-about
- https://developers.cloudflare.com/fundamentals/reference/connection-limits/
- https://developer.apple.com/documentation/technotes/tn3156-create-rich-previews-for-messages
- https://github.github.com/gfm/#autolinks-extension-
- ISO/IEC 18004 QR capacities, measured with https://www.npmjs.com/package/qrcode

**Fonts:**
- https://developers.google.com/fonts/docs/developer_api
- https://fontsource.org/docs/getting-started/cdn
- https://api.fontsource.org/v1/fonts/inter
- https://fonts.bunny.net/about
- https://openfontlicense.org/ofl-faq/
- https://www.gesetze-bayern.de/Content/Document/Y-300-Z-GRURRS-B-2022-N-612 (LG München I, 3 O 17493/20)
- Live `curl` of `fonts.googleapis.com/css2` and `fonts.gstatic.com` on 2026-11-08 (headers and URLs quoted in 2.4)

**Hosting:**
- https://developers.cloudflare.com/pages/platform/limits/ (20,000 files, 25 MiB per file, 20-minute builds, `_headers` 100 rules and 2,000 chars per header)

**Unicode:**
- https://trojansource.codes/ (CVE-2021-42574)
- https://www.unicode.org/reports/tr39/
