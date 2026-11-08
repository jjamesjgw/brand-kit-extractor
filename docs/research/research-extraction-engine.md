# Research: deterministic extraction engine (URL, logo, image palette, brand-guide PDF)

**Product:** free, anonymous brand-kit tool by Justin James. No AI, no accounts, uploads never leave the browser, $0 to run.
**Research date:** 2026-11-08
**Confidence tags:** [Certain] = primary source read, or measured/tested by me. [Likely] = strong inference or consistent secondary sources. [Guessing] = gap-filling; the value needs tuning on the test corpus.

**How this was researched.** I read vendor docs, specs and source code. I also ran these lab tests in this session:

- Bundle sizes: built with esbuild 0.28.2 (`--bundle --minify --format=esm`), then measured gzip -9 size.
- Browser behavior: Google Chrome for Testing 153.0.8010.12, headless, driven by playwright-core, against local test servers.
- Palette and ramp prototypes: run in Node 22.

The test scripts are in `/tmp/claude-0/-home-user/02d649eb-8126-5ff7-985f-ad69e6e47e14/scratchpad/browsertest/` and `/tmp/claude-0/-home-user/02d649eb-8126-5ff7-985f-ad69e6e47e14/scratchpad/libtests/`. Only Chrome was available. Firefox and Safari behavior is inferred from specs, not tested.

**Cross-track inputs.** This report follows the locked privacy and IP/abuse research in the same folder:

- `research-privacy-law.md`: requirements PRIV-01 to PRIV-13.
- `research-ip-abuse-law.md`: requirements F-1 to F-10, E-1 to E-5 and T-1 to T-9.

Where those reports set a rule (caps, no hotlinking, no site font files, wording), this report designs within it.

---

## 0. Bottom line

1. **The browser can resolve the CSS cascade for us, safely.** The approach:
   - Parse the fetched HTML inertly with `DOMParser`.
   - Strip anything active.
   - Inline the fetched CSS into `<style>` elements.
   - Mount the result in an `<iframe sandbox="allow-same-origin" srcdoc=…>`. The sandbox has no `allow-scripts`, and a CSP blocks all network access.
   - Read `getComputedStyle` from the parent page.

   This resolves var() chains, the cascade, `@layer`, nesting, `color-mix()`, relative colors and `light-dark()` with the browser's own engine, and adds 0 KB of parser code. In my Chrome 153 tests:
   - Scripts did not run.
   - The parent read every computed value.
   - The inherited parent CSP blocked 100% of external loads: 0 hits reached the test server.
   - Without CSP, the same document made 5 requests to the "target site".

   [Certain for Chrome 153; Likely for Firefox and Safari, per the HTML and CSP specs]
2. **Computed styles are only deterministic if four things are neutralized.** Each of these changed results in testing [Certain]:
   - **The user's OS dark mode.** Body background changed from #FFFFFF to #111111.
   - **Windows forced-colors mode.** All author colors were replaced by system colors.
   - **Browser default (UA) styles.** Link blue `rgb(0,0,238)` and button font Arial are not brand choices.
   - **Invalid `var()` values.** These silently fall back to inherited values.

   The fixes are tested: rewrite media queries, force `color-scheme`, inject `forced-color-adjust:none`, compare against a no-author-CSS baseline, and use a sentinel probe.
3. **Neither culori 4.0.2 nor colorjs.io 0.7.1 parses `color-mix()`, `light-dark()` or relative color syntax** [Certain, tested]. Use the browser to resolve those, then culori to convert. culori costs 11.6 KB gz tree-shaken, against 21.5 KB gz for colorjs.io's functional API.
4. **The common MMCQ palette algorithm invents colors** [Certain, tested]. On a synthetic two-color logo, `quantize` (MMCQ, the algorithm used by classic color-thief):
   - returned `#e4542c` and `#1c3c8c` instead of the real `#e4572e` and `#1e3a8a`;
   - added two colors that are not in the image.

   A deterministic OKLab k-means that reports each cluster's **most frequent exact pixel** returned the exact hex values. Write the ~150-line quantizer ourselves.
5. **pdf.js 6.5.438 (published 2026-11-01) no longer has an `isEvalSupported` option.** The eval-based font path behind CVE-2024-4367 was removed (mozilla/pdf.js#18015). OSV shows no known vulnerability for 6.5.438. Load it lazily: 136 KB gz main plus 378 KB gz worker. [Certain]
6. **No headless-browser fallback in v1.** Cloudflare Browser Run's free tier is 10 browser-minutes per day. It also conflicts with the IP track's "do not run JavaScript" rule (F-1). Show an honest "partial result" and offer screenshot or logo upload instead. [Certain on limits; Likely on the recommendation]
7. **Real fixture snapshots must not go in the public repo.** Third-party HTML and CSS are copyrightable code (E-1). Commit only the expected *facts* (hex values, font names) plus synthetic fixtures we write ourselves. [Likely]

---

## 1. Concrete requirements for this product

IDs (EXT-xx) are for the spec. Thresholds marked [Guessing] are starting values to tune on the corpus in §3.18.

### A. Architecture and module boundaries

- **EXT-01. The server fetches bytes and nothing else.** The Worker endpoint does:
  - URL validation and SSRF blocking (F-8);
  - robots.txt (F-4) and the do-not-fetch list (F-6);
  - fetching the page HTML;
  - discovering and fetching the linked stylesheets and their `@import`s;
  - later, fetching up to N image or manifest files the client picks.

  It does no color, font, logo or name logic. Link discovery uses Cloudflare's built-in streaming `HTMLRewriter` with fixed selectors (`link[rel]`, `style`, `meta`, `script[type="application/ld+json"]`), plus a regex for `@import` inside fetched CSS. Why:
  - The Workers Free plan allows **10 ms CPU per invocation**, 100,000 requests per day, 50 subrequests per request and 6 open connections. Source: https://developers.cloudflare.com/workers/platform/limits/ [Certain on limits]
  - F-9 says to parse fetched content as inert text in the client. [Likely on design]
- **EXT-02. Two calls per analysis.**
  1. `POST /api/page {url, humanCheckToken}` returns `{finalUrl, status, charset, html, stylesheets:[{url, media, order, text}], skipped:[{url, reason}], ticket}`.
  2. `POST /api/assets {ticket, urls[≤6]}` returns the image or manifest bytes.

  Rules for the second call:
  - **Ticket:** an HMAC over (page origin, expiry 120 s, nonce). It is stateless, so it needs no KV writes.
  - **Allowed URLs:** the page's registrable domain, plus a fixed list of platform CDNs (§3.2).
  - **Content types:** `image/*`, `application/manifest+json` or JSON.
  - **Size:** ≤ 1 MB each, within F-1 (≤ 5 images, ≤ 20 subrequests and ≤ 5 MB per analysis).

  The client picks which logo candidates to fetch after ranking them. That keeps all ranking logic in the browser, while the server is never an open proxy. [Likely]
- **EXT-03. Prioritize stylesheets under a hard budget.** Fetch at most 10 stylesheets and 3 MB of CSS, in this order:
  1. same-origin CSS;
  2. site-specific platform CSS, such as:
     - Squarespace `static1.squarespace.com/static/versioned-site-css/…/site.css`;
     - WordPress `/wp-content/themes/…` and `/wp-content/uploads/elementor/css/post-*.css`;
     - Shopify `cdn.shopify.com/s/files/…/assets/*.css`;
     - Webflow `cdn.prod.website-files.com/<siteId>/css/…`;
  3. everything else.

  Never fetch:
  - Google Fonts, Bunny or Fontshare CSS (family names are in the URL);
  - Adobe Fonts kit CSS (T-4);
  - reCAPTCHA or consent-manager CSS;
  - Squarespace `assets.squarespace.com/universal/…` and `definitions.sqspcdn.com` component CSS.

  Evidence: the owner's own Squarespace homepage (wethejamesphoto.com, fetched 2026-11-08) links **16 stylesheets**. Fetching all of them would use most of F-1's 20-subrequest budget, and only one of them (`site.css`) is site-specific. [Certain on the count; Likely on the priority list]
- **EXT-04. Client module boundaries.** Each module is a pure function of its inputs, except the four that need a real DOM or worker (marked ★).
  - `fetch-client`: talks to the Worker and maps HTTP and challenge outcomes to UI states.
  - `html-ingest`: parses with `DOMParser` (inert); extracts head metadata, JSON-LD, manifest and icon refs; assembles the sanitized probe document.
  - ★ `render-probe`: the sandboxed-iframe harness. Runs the UA-baseline, light and dark passes, the element sampler and the token resolver. Main thread.
  - `css-tokens`: walks the CSSOM. Outputs custom properties per scope, color declarations with their selectors, `@font-face`, and the var() reference graph, plus the platform detector.
  - `color-core`: a culori wrapper for parsing, conversion, gamut mapping, ΔE_OK, WCAG 2.x contrast and alpha compositing.
  - `color-roles`: scoring and slot assignment (pure).
  - `font-detect`: font-stack cleaning, next/font decoding, load-source labeling, catalog match and substitution (pure).
  - `logo-rank`: candidate scoring (pure), plus a ★ worker step that decodes candidates for size and dHash dedupe.
  - `identity`: brand name and tagline (pure).
  - ★ `image-palette`: Web Worker for decode, sampling and clustering.
  - ★ `svg-logo`: DOMPurify sanitizing (main thread), paint extraction via the probe, variant generation, rasterizing.
  - `raster-logo`: worker for alpha analysis, trimming and silhouettes.
  - `pdf-guide`: lazy chunk with pdf.js plus our own parser.
  - `color-system`: ramps, contrast grid and passing pairs (pure).
  - `type-system`: scale detection and generation, fallback stacks (pure).
  - `kit-model`: merges all sources with provenance; manual overrides win.

  [Likely]
- **EXT-05. Determinism contract.** The same fetched bytes or uploaded file must produce byte-identical kit JSON, regardless of the visitor's:
  - OS theme;
  - forced-colors setting;
  - viewport and device (hover, pointer, resolution);
  - run count.

  Enforce this with EXT-08 to EXT-11, a seeded PRNG for k-means, and stable sort keys (score, then hex). [Certain that each listed factor changed results in tests]

### B. Sandboxed render probe (URL path)

- **EXT-06. Probe document construction.**
  1. Parse with `new DOMParser().parseFromString(html, 'text/html')`. This is inert: no fetches, no scripts.
  2. Remove `script`, `noscript`, `iframe`, `frame`, `object`, `embed`, `base`, `meta[http-equiv]`, `link`, `portal` and `template[shadowrootmode]`.
  3. Remove all `on*` attributes, plus `src`, `srcset` and `poster` on media.
  4. Prepend `<meta http-equiv="Content-Security-Policy" content="default-src 'none'; style-src 'unsafe-inline'; img-src data: blob:; font-src data:">`.
  5. Mount the result as `<iframe sandbox="allow-same-origin" srcdoc=…>`. Never add `allow-scripts`.

  The iframe is 1280×800, positioned off-screen, and not `display:none`.

  Sources:
  - MDN: "When `allow-same-origin` is present, a same-origin parent document can still access and interact with the iframe's DOM even if `allow-scripts` is not set." https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/iframe (mdn/content#42649)
  - CSP3 §7.8: srcdoc documents inherit the embedder's policy. https://www.w3.org/TR/CSP3/#security-inherit-csp

  Tested in Chrome 153:
  - an inline `<script>` and an `onerror` handler did not run;
  - external `@import`, `<img>`, `@font-face` and `background-image` loads were blocked by CSP (parent CSP alone, meta CSP alone, and both), with 0 server hits;
  - with neither CSP, 5 requests reached the external server.

  [Certain for Chrome; Likely for other engines]
- **EXT-07. Inline all CSS; never `<link>` it.** Insert each fetched stylesheet as `<style>` in original document order, with the `media` attribute kept. Linked cross-origin sheets throw `SecurityError` on `.cssRules`, which I hit in the control test. They would also leak the visitor's IP to the target site (PRIV-03). [Certain]
- **EXT-08. Three passes.**
  1. **UA baseline:** the same DOM with no author CSS. Any computed value equal to the baseline for that element is a browser default and is never brand evidence. Examples: link `rgb(0, 0, 238)` in light and `rgb(158, 158, 255)` in dark, button font `Arial`.
  2. **Light pass.**
  3. **Dark pass**, run only if dark signals exist: `prefers-color-scheme: dark` rules, or token sets under `.dark`, `[data-theme=dark]`, `[data-bs-theme=dark]` or `[data-mode=dark]`.

  [Certain on the UA values observed; Likely on the method]
- **EXT-09. Neutralize environment-dependent media queries.** Rewrite them through CSSOM (`CSSMediaRule.media.mediaText = …`), capturing the original texts once, before any rewrite:
  - `(prefers-color-scheme: dark|light)` becomes `(min-width: 0px)` (true) or `(max-width: 0px)` (false), depending on the pass.
  - These are forced to fixed values: `prefers-contrast`, `forced-colors`, `inverted-colors`, `prefers-reduced-transparency`, `prefers-reduced-data`, `hover`, `any-hover`, `pointer`, `any-pointer`, `resolution`, `dynamic-range`.

  Tested: with the visitor's OS set to dark, the light pass still returned light values. The dark pass returned the dark token (`#ff8a65`) and the dark-only `h2` rule. I found one trap: CSSOM wrapper objects lose expando properties, so originals must be held in a list of `[rule, originalText]`. [Certain]
- **EXT-10. Injected override stylesheet,** last in the cascade:
  - `*,*::before,*::after{forced-color-adjust:none!important;animation:none!important;transition:none!important}`
  - plus `:root{color-scheme:light!important}` in the light pass, or `dark` in the dark pass.

  Tested: with forced colors active, the probe read body, button and link colors as system colors (`rgb(255,255,255)`, `rgb(255,255,255)`, `rgb(0,0,159)`). With `forced-color-adjust:none` it read the author values (`rgb(253,246,227)`, `rgb(228,87,46)`, `rgb(38,139,210)`). Forcing `color-scheme` makes `light-dark()` deterministic; tested. Disabling animation stops mid-animation values from being read. [Certain on forced colors and light-dark; Likely on animations]
- **EXT-11. Token resolver with a sentinel.** To resolve a custom property or token expression:
  1. Put a probe `<span>` inside a wrapper.
  2. Set the wrapper's `color` to sentinel A, set the probe's `color: <expr>`, and read the computed value.
  3. Repeat with sentinel B.
  4. If the result equals the sentinel both times, the expression is invalid at computed-value time.

  Expressions to try per token:
  - `var(--x)`
  - `hsl(var(--x))` (shadcn v3 triplets)
  - `rgb(var(--x))` (Tailwind or Shopify RGB triplets)
  - `hsla(var(--x),1)` (Squarespace comma-separated HSL)

  Mount the probe inside the token's scope: `:root`, `body` (Elementor's `.elementor-kit-N` is on `<body>`), or the first element matching the scope selector.

  Tested: without the sentinel, `color: var(--undefined)` silently returned the inherited text color.

  | Input | Computed value |
  |---|---|
  | shadcn triplet `222.2 47.4% 11.2%` via `hsl(var(--primary))` | `rgb(15, 23, 42)` |
  | Squarespace `41.54, 23.64%, 89.22%` via `hsla(var(--sq),1)` | `rgb(234, 230, 221)` |
  | Tailwind v3 `rgb(var(--tw-ring) / var(--tw-bg-opacity, 1))` | `rgb(37, 99, 235)` |

  [Certain]
- **EXT-12. Don't trust layout for image sizes.** The probe never loads images, so layout boxes for `<img>` are unreliable. A logo with `width=120 height=40` rendered as 96.9×20, its alt-text box. Take logo dimensions from attributes, CSS, and the decoded bytes fetched in call 2. [Certain]
- **EXT-13. Exclude third-party UI.** Skip elements and subtrees whose id or class matches `/cookie|consent|gdpr|ccpa|cmp|onetrust|cookiebot|cky-|osano|termly|intercom|drift|tawk|hubspot-messages|zendesk|livechat|crisp|grecaptcha|g-recaptcha|tidio|elfsight|trustindex|birdeye|podium/i`. Also skip third-party iframes, which are already removed. These widgets paint saturated colors that are not the brand. [Likely; list is a starting point]
- **EXT-14. Performance caps.**
  - Sample at most 4,000 visible elements, in DOM order.
  - Yield to the event loop between passes.
  - Show progress.

  Measured on a synthetic 2.19 MB CSS / 2.59 MB HTML / 16,000-element page (4 vCPU, headless Chrome):

  | Step | Time |
  |---|---|
  | DOMParser + sanitize + serialize | 94 ms |
  | iframe load | 422 ms |
  | 4,000 `getComputedStyle` reads | 8 ms |
  | CSSOM walk of 15,001 rules / 212,000 declarations | 19 ms |

  A mid-range phone is likely 3–5× slower, roughly 1.5–2.5 s of main-thread work at the F-1 caps. [Certain on the measurements; Guessing on the phone multiple]

### C. Color evidence, normalization and the "never invent" invariant

- **EXT-15. Use culori 4.0.2 (MIT) for color math.** Import from `culori/fn` and register only the needed modes: rgb, hsl, hwb, lab, lch, oklab, oklch, p3, rec2020, a98, prophoto, xyz50, xyz65 and lrgb. That bundle measured 11.6 KB gz. Output hex via `toGamut('rgb','oklch')`, the CSS Color 4 gamut mapping, never plain clipping. For example, `color(display-p3 1 .5 0)` becomes `#ff7d00` mapped versus `#ff7600` clipped. Keep the original CSS string as provenance. [Certain, tested]
- **EXT-16. Alpha.** Composite any color with alpha < 1 over the element's painted background chain before scoring. Drop paints with alpha < 0.3 as overlay noise. [Guessing on 0.3]
- **EXT-17. Duplicates and distinctness.**
  - Two colors are the same if ΔE_OK (Euclidean in OKLab) < 0.02. That is the JND CSS Color 4 uses in gamut mapping. When merging, keep the hex with the most evidence; never average.
  - Two role slots must differ by ΔE_OK ≥ 0.08.
  - A color is **neutral** if OKLCH C < 0.03.

  [Likely on 0.02; Guessing on 0.08 and 0.03]
- **EXT-18. Never invent (hard invariant, enforced by tests).** Every output color needs at least one provenance record of one of these kinds:

  | Kind | Meaning |
  |---|---|
  | painted | Computed on a visible element. |
  | declared+used | Declared in a rule that matches at least one element, or a token referenced by such a rule. |
  | meta | `theme-color` or the manifest. |
  | pixel | An uploaded or fetched logo image. |
  | svg-paint | A paint in an SVG. |
  | pdf-text or pdf-fill | Found in a brand-guide PDF. |
  | manual | Entered by the user. |

  Derived values (ramps, gamut-mapped hex values from out-of-gamut colors, CMYK conversions) carry `derived:true`. They are labeled as derived in the UI and never fill a role slot by themselves. Slots with too little evidence stay empty, with no filler. [Likely]

### D. Role inference (primary / secondary / accent / background / text)

- **EXT-19. Scoring.** Add up evidence per candidate color from the light pass:

  | Evidence | Base weight | Size factor |
  |---|---|---|
  | `background-color` (composited) | 3 | `min(8, log2(1 + visibleArea/1000))`, area clipped to the first 3 viewports (1280×2400) |
  | `color` on an element with direct text (≥ 2 chars) | 2 | `min(4, log2(1 + chars/20))` |
  | border color (width ≥ 1 px, style not none, differs from bg) | 1 | `min(3, log2(1 + perimeter/200))` |
  | inline SVG fill/stroke inside header, logo or button | 2 | 1 |
  | gradient color stops (parsed from computed `background-image`) | 1 | 1 |
  | declared+used rule not visible in the DOM sample | 0.2 | 1 (tie-breaker only) |
  | meta `theme-color` / manifest `theme_color` | 3 (once) | 1 |

  Context multiplier: take the highest that applies.
  - Button-like: `button`, `[role=button]`, `input[type=submit|button]`, `a` with class `/\b(btn|button|cta)\b/`, or `a` with its own non-transparent background: **×4**
  - `h1`: ×3; `h2`: ×2; `h3`: ×1.5
  - plain `a`: ×2
  - inside header, nav or `[role=banner]`: ×1.5
  - inside footer: ×1.2
  - class token `/brand|primary|accent|cta|highlight/`: ×2

  Token boost ×1.5 applies when a painted color equals (ΔE_OK < 0.02) a used token whose name or platform mapping gives a role hint (EXT-21). [Guessing on all weights; the structure follows dembrandt's `CONTEXT_SCORES` and Cloudflare's reference script, see §3.1]
- **EXT-20. Slot assignment,** in this order:
  1. **background:** the composited background with the largest painted area among `html`, `body`, `main` and full-width wrappers in the first viewport. If no author color applies, report `#FFFFFF` labeled "browser default (the site sets none)".
  2. **text:** the color with the most text weight among `p`, `li`, `td`, `dd`, `blockquote` and plain text containers, excluding headings, links and buttons. It must contrast ≥ 3:1 with the background; otherwise take the next.
  3. **primary:** the chromatic (C ≥ 0.03) candidate with the highest score. Button-background evidence counts double for this slot. Tie-breaks: a token hint, then larger button area, then hex. A neutral can be primary only when no chromatic candidate reaches 10% of total non-bg/text score **and** the neutral is a button background.
  4. **secondary:** the next candidate with ΔE_OK ≥ 0.08 from all filled slots and score ≥ 15% of primary. Prefer large-surface or heading usage.
  5. **accent:** the next candidate with ΔE_OK ≥ 0.08 from all filled slots and score ≥ 8% of primary. Prefer link and small-element usage.

  Show up to 8 more colors as "Other colors on the page" with usage counts. Each slot gets a confidence:
  - **high:** 2 or more independent evidence kinds agree;
  - **medium:** a single kind;
  - **low:** static-only.

  [Guessing on thresholds; Likely on structure]
- **EXT-21. Platform token dictionary.** These give role hints. A hint only applies when the token is used:

  | Platform | Tokens |
  |---|---|
  | Elementor | `--e-global-color-primary`, `-secondary`, `-text`, `-accent` |
  | Astra | `--ast-global-color-0` (Brand), `-1` (Alternate brand), `-2` (Headings), `-3` (Text), `-4…7` (backgrounds) |
  | Kadence | `--global-palette1…9` |
  | Blocksy | `--theme-palette-color-1…` |
  | WordPress | `--wp--preset--color--{primary,secondary,accent,base,contrast,…}` |
  | Squarespace | `--sqsp-palette-{lightest,light,accent,dark,darkest}-color`, `--sqsp-link-color`, `--sqsp-heading-1-color`, `--accent-hsl` |
  | Wix | `--wst-button-color-fill-primary`, `--wst-color-action`, `--wst-color-fill-accent-1…4` |
  | Shopify | `--color-button`, `--color-background`, `--color-foreground` |
  | shadcn | `--primary`, `--background`, `--foreground`, `--accent` |
  | Tailwind v4 | `--color-<custom-name>`, where the name is not one of Tailwind's default palette names |
  | Generic | names matching `/primary|brand|accent|secondary|cta|theme|main/` |

  Sources in §3.5. [Certain for the documented ones; Likely for Kadence and Blocksy]
- **EXT-22. Known defaults to discount.** These score 0 unless painted:
  - WordPress core default palette (`vivid-red #cf2e2e`, `luminous-vivid-orange #ff6900`, `luminous-vivid-amber #fcb900`, `vivid-cyan-blue`, `vivid-purple`, and the rest);
  - Tailwind default palette variables;
  - shadcn `--chart-*` and `--sidebar-*`;
  - Bootstrap's default variables.

  Note that the shadcn default `--primary` is `oklch(0.205 0 0)`, near-black. A token *named* primary is not proof of a brand color, so token hints only boost painted evidence. [Certain on the values; Likely on the rule]
- **EXT-23. Accent label check.** For primary and accent, record the computed text color painted *on* them, from button text. The kit's contrast checks use that pair. If no label color exists, skip the check; never invent a pair. This matches the competitor's stated behavior. [Likely]

### E. Fonts

- **EXT-24. Heading and body selection.**
  - **Heading:** the computed `font-family` of the first visible `h1`. Fall back to `h2`, then to the largest computed font-size (≥ 24 px) in the first two viewports.
  - **Body:** the family covering the most text characters in `p`, `li` and similar elements, excluding headings, nav and buttons.

  `getComputedStyle().fontFamily` returns the declared stack, so fonts never need to load. Tested: it returned `__Inter_d65c78, __Inter_Fallback_d65c78, system-ui, sans-serif`. [Certain]
- **EXT-25. Clean the stack.** Walk the families left to right and skip:
  - CSS generics: `serif`, `sans-serif`, `monospace`, `cursive`, `fantasy`, `math`, `emoji`, `fangsong`, `system-ui` and `ui-*`;
  - system aliases: `-apple-system`, `BlinkMacSystemFont`;
  - emoji fonts: Apple Color Emoji, Segoe UI Emoji/Symbol, Noto Color Emoji;
  - icon fonts (§3.8 list);
  - next/font fallbacks: `__X_Fallback_<hex>`, and `X Fallback` when X precedes it.

  Decode next/font hashed names with `/^__(.+?)_(?:Fallback_)?[0-9a-f]{6,8}$/`, then replace `_` with spaces. The first remaining family is the candidate. If it has no `@font-face`, no provider link and isn't a known system font, label it "declared, not loaded by this page" and evaluate the next family too. [Certain on the webpack 6-hex and Turbopack 8-hex formats from Next/Turbopack sources; Guessing on Next 15/16 formats, see Unverified]
- **EXT-26. Label the source with two fields.** **Catalog** is Google Fonts, Adobe Fonts, Fontshare, system, or unknown. **Served from** is the provider CDN, the platform CDN or the site's own server.

  Detection order:
  1. `link[href*="fonts.googleapis.com/css"]`, `@import` of the same URL, or Next.js's inlined `<style data-href="https://fonts.googleapis.com/…">`: **Google Fonts** (family names from `family=`).
  2. `fonts.bunny.net/css`: **Bunny Fonts**.
  3. `use.typekit.net`, `use.typekit.com` or `p.typekit.net`: **Adobe Fonts**. Detect only; never fetch the kit CSS or font files (T-4).
  4. `api.fontshare.com`: **Fontshare**.
  5. `fast.fonts.net` = Monotype; `cloud.typography.com` = Hoefler&Co; `*.myfonts.net` = MyFonts.
  6. Inline `WebFont.load({google:{families:[…]}, typekit:{id:…}})`, read as text, never run. Webflow uses this loader.
  7. `@font-face` served from a platform CDN or the same origin: "served by <platform/site>". If the family name matches the Google catalog, label it "Google Fonts family, served by <platform>".
  8. Known system family: "System font (varies by device)".
  9. Otherwise, no label. Never guess.

  Evidence: the owner's Squarespace site serves Google families Almarai and Ovo from `file.squarespace-cdn.com/content/v2/namespaces/fonts/…` with inline `@font-face`. "Roboto" appears 27 times in its HTML, all from platform UI. That is why fonts are chosen by element (EXT-24), not by frequency. [Certain on the observation; Likely on the method]
- **EXT-27. Rendering in the kit** follows T-1 to T-5 and PRIV-02/04:
  - Google-catalog families render from our own self-hosted copy.
  - Everything else is named, with a labeled open-licensed substitute picked by category.
  - Never load the scanned site's font files.

  Ship the catalog as a static JSON built at deploy time (family, category, variants/axes, license). [Likely]
- **EXT-28. Typography evidence for the type system.** Record computed `font-size`, `font-weight`, `line-height` and `letter-spacing` for `h1`–`h3`, `p` and buttons at 1280 px. [Likely]

### F. Logo candidates and favicon

- **EXT-29. Candidate sources:**
  - `img`, inline `svg` and `picture` elements in header, nav or `[role=banner]`, or in the first 15% of DOM order;
  - any element whose own or ancestor (≤ 3 levels) class, id, alt, src, aria-label or title contains `logo`;
  - known platform logo classes:

    | Platform | Logo class or selector |
    |---|---|
    | WordPress | `custom-logo` |
    | WordPress blocks | `wp-block-site-logo` |
    | Elementor | `elementor-widget-theme-site-logo` |
    | Squarespace | `header-title-logo` |
    | Shopify Dawn | `header__heading-logo` |
    | Webflow | `w-nav-brand` |

  - elements with "logo" classes whose computed `background-image` is a URL;
  - JSON-LD `Organization.logo` (string, `ImageObject.url` or `contentUrl`), `publisher.logo`, `Organization.image` and `WebSite.image`;
  - `link[rel~=icon]`, `apple-touch-icon(-precomposed)`, `mask-icon` and the manifest's `icons`;
  - `og:image` and `twitter:image`.

  [Likely]
- **EXT-30. Scoring.**

  | Signal | Points |
  |---|---|
  | wrapped in `a[href]` that resolves to the site root (`/`, `./`, origin, or a locale root like `/en/`) | +40 |
  | `logo` token in self or ancestor | +30 |
  | JSON-LD `Organization.logo` | +25 |
  | in header, nav or banner, or the first 15% of DOM | +20 |
  | alt or filename matches the site name or domain root | +15 |
  | `WebSite.image` / `Organization.image` | +10 |
  | SVG | +10 |
  | PNG with alpha | +5 |
  | footer only, not home-linked | −30 |
  | alt or filename names another brand, or matches `/partner|client|as seen|featured|award|badge|certified|bbb|angi|homeadvisor|yelp|google.?review|trustpilot|gaf|owens.?corning|certainteed|visa|mastercard|paypal|financ/i` | −40 |
  | social icon (`facebook|instagram|linkedin|youtube|tiktok|x-logo|twitter|pinterest`) or declared longest edge < 24 px | −20 |
  | `og:image` near 1.91:1 with no logo token (likely a social banner) | −20 |

  The badge patterns matter for the DFW home-services audience: roofers' headers often carry GAF, BBB and Angi badges. [Guessing on points; Likely on signals; home-link and alt rules mirror dembrandt's `isHomeHref`, `thirdPartyBrandFromAlt` and `isOwnBrandMark`]
- **EXT-31. Fetch, decode, dedupe.**
  1. Fetch the top 5 by score (call 2).
  2. Decode in a worker (`createImageBitmap`) to get true pixel size.
  3. Reject logos with longest edge < 32 px; they stay favicon candidates.
  4. Normalize URLs, dropping size parameters such as `?format=1500w`, `w=`, `width=`, Shopify `_NNNx`, and WordPress `-150x150`.
  5. Dedupe with a 64-bit dHash (32×32, alpha composited on white). Hamming ≤ 6 means the same image; keep vector over raster, then the larger one.

  Show the top 3 with their reasons ("linked to home page", "named 'logo'", "from structured data"). [Guessing on thresholds]
- **EXT-32. Inline SVG logos.**
  - Serialize from the fetched HTML, not the probe.
  - Resolve `<use href="#id">` against same-document `<symbol>`s. External sprite files are not fetched.
  - Sanitize (EXT-45) and cap at 100 KB.
  - Use `<picture><source media="(prefers-color-scheme: dark)">` and `-dark`/`-white` filename pairs as on-dark variant hints.

  [Likely]
- **EXT-33. Favicon pick:** SVG `rel=icon`, then the largest PNG ≥ 32 px by `sizes`, then `apple-touch-icon` (Google honors `icon`, `apple-touch-icon` and `apple-touch-icon-precomposed`), then manifest icons, then `/favicon.ico`. https://developers.google.com/search/docs/appearance/favicon-in-search [Certain on the rel values; Likely on order]

### G. Brand name and tagline

- **EXT-34. Name candidates,** each normalized:
  - JSON-LD `WebSite.name` / `Organization.name`. Google says WebSite structured data "is most important", and it also considers `og:site_name`, `<title>` and headings. https://developers.google.com/search/docs/appearance/site-names
  - `og:site_name`, `application-name`, `apple-mobile-web-app-title`, manifest `name`/`short_name`;
  - the logo's alt text, if not generic;
  - the `<title>` split on `| – — - · • : »`;
  - the domain root, as a last resort and labeled as such.

  Pick the name supported by the most sources (normalized similarity ≥ 0.8). For split titles, choose the segment that matches the domain root or another source, not the SEO keyword segment. Example: in "Roof Repair Dallas TX | Acme Roofing", pick "Acme Roofing". [Certain on Google's sources; Likely on method]
- **EXT-35. Tagline candidates,** first match wins:
  1. JSON-LD `slogan`;
  2. `Organization`/`WebSite.description` if ≤ 120 chars;
  3. the first sentence of `og:description` or meta description if ≤ 120 chars, ≤ 18 words and not starting with "Welcome";
  4. empty.

  Always show where it came from; the user can edit it. No text is ever generated. The owner's site had an empty meta description, so an empty result is a normal outcome. [Certain on that observation; Likely on rules]

### H. Partial results, SPAs, blocked sites

- **EXT-36. Detect an empty JavaScript shell.** Flag the page if:
  - visible body text is under 200 chars **and** there is an empty mount node (`#root`, `#app`, `#__next`, `#___gatsby`, `app-root`, `[data-reactroot]`) or a noscript "enable JavaScript" message, **or**
  - there is no `h1`, `button` or `a` in the probe.

  When flagged, run static CSS token analysis only. Show which slots are empty and why, in specific words: "This page builds itself with JavaScript after it loads. We read the page as the server sent it and found 2 of 5 colors and no fonts. Add a screenshot or your logo to fill the gaps." [Likely; copy is a draft]
- **EXT-37. Detect blocks and gates.** On HTTP 401, 403 or 429, a `cf-mitigated: challenge` header, a title like "Just a moment…", or a login, paywall or consent wall: stop. Never retry another way (F-5). Offer the uploads instead. [Likely]
- **EXT-38. No headless rendering in v1.** See §3.11 for numbers and the conflict with F-1. [Likely]
- **EXT-39. Run the image palette on the chosen logo automatically.** Its colors join the evidence with provenance "logo pixels". This covers sites whose CSS carries only neutrals. [Likely]

### I. Image to palette (uploads, logo bytes, screenshots, rendered PDF pages)

- **EXT-40. Run it in a Web Worker.**
  1. Decode with `createImageBitmap(file)` (Baseline widely available since 2026-06-11).
  2. Read pixels from an `OffscreenCanvas` (Baseline widely available since 2025-09-27).
  3. Check the header dimensions before decoding. Reject files over 50 MP or 20 MB.
  4. Sample with a fixed stride (≤ 250,000 samples), or decode with `resizeQuality:'pixelated'` so the sampled values are exact image pixels. Never use smoothing resamplers.

  Accept PNG, JPEG, WebP, GIF (first frame) and AVIF where the browser supports it. For HEIC, show "Convert to JPG or PNG first". [Certain on Baseline dates; Likely on the rest]
- **EXT-41. Algorithm.**
  1. Skip pixels with alpha < 250 (anti-aliased edges). Skip near-white (L_OK > 0.97, C < 0.02) and near-black (L_OK < 0.12, C < 0.02) unless together they exceed 25% of pixels.
  2. Build an exact 24-bit histogram.
  3. **If there are ≤ 64 unique colors (flat art):**
     - merge colors within ΔE_OK < 0.04, keeping the most frequent;
     - drop anti-aliasing blends: colors under 2% share that lie within ΔE_OK 0.02 of the line between two larger clusters.
  4. **Otherwise (photos, screenshots):** run k-means in OKLab with K = 8, k-means++ seeding from a fixed-seed LCG, and 20 iterations. Merge clusters within ΔE_OK < 0.05 and drop clusters under 2% share.
  5. Report each cluster's **medoid**, its most frequent exact pixel color, never the centroid.
  6. Choose 3–6 swatches: the top 2 by share, then by share × (0.5 + C) for chromatic clusters, plus the lightest and darkest neutral if each is ≥ 5%.
  7. Suggest roles: background = largest light neutral; text = darkest with ≥ 4.5:1 against it; primary = highest share × chroma. All marked "suggested from image".

  [Certain that the medoid approach returned exact hex values where MMCQ did not (§3.12); Guessing on thresholds]

### J. Logo upload (SVG, PNG, JPG)

- **EXT-42. SVG intake.** Steps:
  1. Read the file as text and cap it at 1 MB and 10,000 elements.
  2. Parse with `DOMParser('image/svg+xml')` and reject parse errors.
  3. Sanitize with DOMPurify 3.4.17 using `{USE_PROFILES:{svg:true, svgFilters:true}}`.
  4. Then run our own hook that does three things:
     - removes any `href`/`xlink:href` not starting with `#` or `data:image/`;
     - strips `@import` and non-fragment `url()` from `<style>` and `style=`;
     - removes `<a>` wrappers.

  Tested: DOMPurify removed `onload`, `<script>`, `<foreignObject>`, `javascript:` hrefs and external `<use>`. It **kept** `<style>` with an external `@import` and `<image href="http://…">`, so the extra hook is needed. Display SVGs only as `<img src="blob:…">`; in an image context SVG scripts never run and external resources are not loaded. https://developer.mozilla.org/en-US/docs/Web/SVG/Guides/SVG_as_an_image [Certain]
- **EXT-43. SVG paint extraction.** Mount the sanitized SVG in the sandboxed probe (EXT-06). Read computed `fill`, `stroke`, `stop-color`, opacity and visibility per shape. This handles classes inside `<style>`, `currentColor` and inheritance. Weight each paint by its pixel share in a 512 px rasterization, snapping each pixel to the nearest paint within ΔE_OK 0.04. [Likely]
- **EXT-44. SVG variants** are rule-based and editable before export.
  - **On-dark:** swap paints with < 3:1 contrast against the chosen dark background (the WCAG 1.4.11 non-text threshold) to `#FFFFFF` or the kit's lightest neutral.
  - **On-light:** the mirror image.
  - **One-color:** a rect filled with the chosen color, masked by a recolored copy of the logo. Ink becomes white, and "knockout" paints become black. Knockout paints are near-white (L_OK > 0.95, C < 0.02) paints drawn above other ink; the user can toggle which colors count as knockouts.

  Tested: the knockout pixel came out fully transparent `[0,0,0,0]` and ink `[17,17,17,255]`, rendered through a `blob:` `<img>` under the strict CSP. Gradients get mapped per stop. If the SVG holds raster `<image>`s or live `<text>`, show "Convert text and images to outlines for accurate variants". [Certain on the mask test; Likely on rules]
- **EXT-45. Raster logo limits.**
  - Show on-light and on-dark previews as-is, with a contrast warning.
  - One-color silhouette only if ≥ 10% of pixels are fully transparent; recolor opaque pixels and keep alpha. An optional "treat white as cut-out" toggle uses a luminance threshold.
  - No per-paint swapping.
  - Offer optional white-background removal for JPGs: flood-fill from the edges at ΔE_OK < 0.03, user-confirmed.
  - Warn below 500 px width that the logo is too small for print.

  [Guessing on thresholds]

### K. Brand-guide PDF (in-browser)

- **EXT-46. pdf.js version and loading.**
  - Pin `pdfjs-dist` ≥ 4.2.67; it is currently 6.5.438 and OSV lists no known vulnerability as of 2026-11-08.
  - Load it as a lazy chunk only after a PDF is chosen: `pdf.min.mjs` is 136 KB gz and the worker is 378 KB gz.
  - Self-host the worker, `cmaps/`, `standard_fonts/` and `wasm/` (PRIV-05).
  - CSP: `worker-src 'self'`; no `unsafe-eval`.
  - Never load `pdf.sandbox` or the QuickJS scripting wasm.
  - Pass `isEvalSupported:false` anyway; v6 ignores it.
  - Add a CI check that fails if `new Function(` or `eval(` appears in the shipped pdf.js files.

  [Certain on versions and file sizes; Certain that v6 has no `isEvalSupported` (grep of 6.5.438 source)]
- **EXT-47. `getDocument` options:**
  - `enableXfa:false`, `useSystemFonts:false`, `fontExtraProperties:true` (needed to read embedded font `name`s)
  - `maxImageSize: 16_000_000`, `stopAtErrors:false`
  - `cMapUrl`, `standardFontDataUrl` and `wasmUrl` pointing at same-origin paths

  Caps: file ≤ 25 MB, ≤ 40 pages processed, 20 s timeout, then call `loadingTask.destroy()`. [Likely]
- **EXT-48. Text extraction.** For each page, call `getTextContent()`. Rebuild lines by y (`transform[5]`, rounded), then sort by x. Normalize with NFKC. Find codes with the regexes in §3.14 (HEX with `#` or a HEX/HTML/WEB label; RGB 0–255; CMYK 0–100; Pantone/PMS). Group codes into "color records" by spatial proximity, attaching the nearest label line above, such as "Brand Orange". Cross-check HEX against RGB: an exact match gives high confidence; a mismatch shows both, flagged. [Likely]
- **EXT-49. Vector swatch evidence.** `page.getOperatorList()` gives fill colors as hex strings. The pdf.js worker converts `setFillRGBColor`, gray and CMYK fills into `['#rrggbb']`. A color found both as a filled shape and as text is "confirmed". If the PDF has no text codes (outlined or scanned), offer the top fill colors as "sampled from the PDF artwork". Pages that are only images go to the image palette after rendering at 1×. [Certain on the operator-list representation (source read); Likely on use]
- **EXT-50. Fonts in PDFs.** Find typeface names in text near keywords such as typeface, font, typography, headline, body copy, primary or secondary. Match them against the Google catalog (longest match first) plus a curated list of common commercial families. Also read embedded font names (`page.commonObjs.get(id).name` with `fontExtraProperties`):
  - strip subset prefixes `/^[A-Z]{6}\+/`;
  - split style suffixes (`-Bold`, `-Regular`, `MT`, `PS`, `Std`, `Pro`);
  - ignore ArialMT, Helvetica, TimesNewRomanPSMT, Calibri and similar unless named in the text.

  [Certain on prefix format and API; Likely on usefulness]
- **EXT-51. Pantone and CMYK.** Show Pantone references verbatim as text and never convert them to hex. Pantone color data is licensed; Adobe removed the Pantone libraries in 2022 over licensing (https://creativepro.com/pantone-color-libraries-are-leaving-the-adobe-apps/). Convert CMYK to RGB only when the guide gives no HEX or RGB, labeled "approximate (converted from CMYK, no color profile)". [Likely]

### L. Color system

- **EXT-52. Tint and shade ramps,** 11 stops (50–950) at fixed OKLCH lightness: 50 = .97, 100 = .93, 200 = .87, 300 = .79, 400 = .70, 500 = .62, 600 = .54, 700 = .46, 800 = .38, 900 = .30, 950 = .22.
  - Keep the brand hue constant.
  - Use relative chroma: chroma = (brand chroma ÷ max in-sRGB chroma at the brand's L and h) × the max chroma at each stop's L and h, capped at the brand's own chroma.
  - The brand color replaces its nearest stop, unchanged.
  - All ramp values are `derived:true`.

  Tested: hue drift for `#e4572e` stayed between 35.5° and 36.6° across all stops. A naive fixed-chroma ramp drifted to pure red (`#820100`) at the dark end. [Certain on test; Likely on design]
- **EXT-53. Contrast math.** Use WCAG 2.2 relative luminance with the 0.04045 threshold. W3C notes: "Before May 2021 the value of 0.04045 in the definition was different (0.03928)." Composite alpha first and never round ratios. Thresholds:

  | Use | AA | AAA |
  |---|---|---|
  | Normal text | 4.5 | 7 |
  | Large text | 3 | 4.5 |
  | Non-text UI | 3 | — |

  Name the output "pairs that pass WCAG AA/AAA" (E-2) and use the E-4 disclaimer. Check: `#777777` on `#ffffff` gives 4.478 in both culori and colorjs.io, so it fails AA. Sources: https://www.w3.org/TR/WCAG22/#dfn-relative-luminance and https://www.w3.org/WAI/WCAG22/Understanding/contrast-minimum.html [Certain]
- **EXT-54. No APCA in v1.** WCAG 3's contrast method is "yet to be determined"; APCA was pulled from the July 2023 draft; WCAG 3 is unlikely before 2030 (https://adrianroselli.com/2026/04/wcag3-contrast-as-of-april-2026.html). The `apca-w3` package's license is "All Rights Reserved. Patent(s) pending", limits use to W3/WCAG purposes, and restricts use of the APCA name. [Certain]

### M. Typography system

- **EXT-55. Scale.** Fit a ratio from the computed h1, h2, h3 and body sizes at 1280 px: the geometric mean of size ratios per step. Snap to the nearest named ratio within 3%: 1.125, 1.2, 1.25, 1.333, 1.414, 1.5 or 1.618. Otherwise use 1.25. Base = computed body size rounded to whole px (default 16). [Guessing on snap tolerance]
- **EXT-56. Fallback stacks by category.**
  - Sans: `system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif`
  - Serif: `Georgia, "Times New Roman", serif`
  - Mono: `ui-monospace, SFMono-Regular, Menlo, Consolas, monospace`
  - Display and handwriting: map to sans or serif per the catalog's classification; never `cursive`.

  Optional: precompute `size-adjust` and `ascent-override` fallback metrics at build time from `@capsizecss/metrics` 4.4.0 (MIT, "Font metrics library for system and Google fonts"). It is 27.7 MB unpacked, so use it at build time only. [Certain on package facts]

### N. Libraries and budgets

- **EXT-57. Runtime libraries.** Sizes measured as minified + gzip -9 of a realistic import:

  | Library | Version | Size | Notes |
  |---|---|---|---|
  | culori | 4.0.2 | 11.6 KB gz | MIT |
  | DOMPurify | 3.4.17 | 12.1 KB gz | MPL-2.0 OR Apache-2.0 |
  | pdfjs-dist | 6.5.438 | 136 + 378 KB gz | Apache-2.0; lazy-loaded |

  Everything else is our own code. No CSS parser library (css-tree full is 56.2 KB gz, postcss parse 14.7 KB gz); the browser CSSOM does that work. No quantization library (§3.12). The initial extraction bundle should stay under about 100 KB gz excluding pdf.js. [Certain on measured sizes; Guessing on our own code size]
- **EXT-58. Licenses.** Do not copy data from the Wappalyzer fork (`enthec/webappanalyzer`, GPL-3.0). Write our own host patterns; hostnames are facts. dembrandt is MIT; code ideas may be borrowed with attribution. [Certain on licenses]

### O. Testing and corpus

- **EXT-59. Test layers.**
  1. Unit tests for pure modules (Node + vitest).
  2. Golden tests on public **synthetic** fixtures, each isolating one pattern (§3.18), run with Playwright on Chromium, Firefox and WebKit in GitHub Actions. Actions minutes are free for public repos.
  3. Local-only evaluation on a 30-site **real** corpus.
  4. Security fixtures: malicious SVG, a CVE-2024-4367-style FontMatrix PDF, and a script-laden HTML page. Assert nothing executes and no non-self requests happen.
  5. Network assertions: uploading files triggers zero requests carrying file bytes.

  [Likely]
- **EXT-60. Never commit real-site snapshots to the public repo.** Commit only `corpus/expected/*.json` (facts). Snapshots live in a gitignored folder, refreshed by the owner on demand, never on a schedule. A scheduled CI crawl would break the "user-triggered fetcher" posture (F-2). [Likely]
- **EXT-61. Accuracy targets for the v1 launch,** on the 30-site corpus. These are targets, not measurements:

  | Measure | Target |
  |---|---|
  | invented colors | **0** (hard gate) |
  | background and text | ≥ 28/30 exact (ΔE_OK ≤ 0.02) |
  | primary top-1 | ≥ 24/30; top-3 ≥ 28/30 |
  | heading and body family | each ≥ 27/30 |
  | font-source label precision | ≥ 95%; a wrong label fails, "not detected" is fine |
  | logo top-1 | ≥ 25/30; top-3 ≥ 28/30 |
  | favicon | ≥ 29/30 |
  | brand name | ≥ 27/30 |
  | JavaScript-shell and blocked sites flagged | 100% |
  | repeat runs | 10 runs byte-identical |
  | cross-engine role hex values | ≥ 28/30 identical |
  | client extraction time | p95 ≤ 2.5 s at 4× CPU throttle |
  | Worker CPU | p99 ≤ 8 ms |

  [Guessing on targets]

---

## 2. Conflicts and cross-track flags

1. **Headless rendering fallback versus F-1, F-5 and $0.**
   - The brief lists an "optional Browser Rendering quota" fallback.
   - The IP track's F-1 says "do not run JavaScript", and F-5 forbids retrying past gates with a headless browser.
   - Cloudflare Browser Run free is 10 minutes per day (§3.11).
   - Running a site's JavaScript also executes its third-party trackers from our account.

   **Recommendation:** none in v1. Revisit only with a written decision. [Likely]
2. **"Approved text-on-background pairs" (brief) versus E-2.** Use "pairs that pass WCAG AA/AAA". [Certain that E-2 says so]
3. **"Never invent a color" versus generated ramps and conversions.** This is not a true conflict. Ramps, gamut-mapped hex values and CMYK conversions are clearly labeled derived and never fill role slots (EXT-18). [Likely]
4. **A 30-real-site fixture corpus versus a public repo.** Third-party HTML and CSS are copyrightable code (E-1 rationale), and scheduled fetching weakens F-2. Hence EXT-60. [Likely]
5. **The F-1 20-subrequest cap versus platform CSS sprawl.** A Squarespace homepage links 16 stylesheets. Without prioritization (EXT-03), the budget runs out before images. [Certain on the count]
6. **The Adobe Fonts kit CSS would be useful** for family names, but T-4 says detect only. Family slugs used in the site's own `font-family` declarations (for example `proxima-nova`) are enough. Don't fetch kit CSS. [Likely]
7. **Typekit kits count page views** through `p.typekit.net`. Fetching kit resources could consume the site owner's quota; another reason not to fetch. [Guessing on the counting mechanism]

---

## 3. Detailed findings

### 3.1 Prior art

| Project | What it does | How | License / cost | What to take |
|---|---|---|---|---|
| **Blendin Brand Kit Extractor** (competitor) | ≤ 5 role colors, heading and body fonts with load-source label, ranked logos + favicon, name and tagline, WCAG check, human check, "stores nothing" | Self-described: "parses the stylesheets (CSS variables and their var() chains, light and dark themes, nested rules, Tailwind v3 and v4 utilities, Elementor and WordPress builder tokens) … every color format (hex, rgb, hsl, the shadcn triplet form, oklch, lab, color-mix). Then it checks the page itself: the color actually painted on the buttons, links, headings and page background wins over a token the page never uses." The method itself is not public. | Free; lead magnet for a $59/mo product | The feature bar to match, and the "never invent" and "unlabeled rather than guessed" posture. https://blendin.ai/brand-kit-extractor [Certain on claims] |
| **dembrandt** v0.39.2 (4.0k stars) | Logo, colors with semantic roles, typography, spacing, shadows, components; DTCG export | **Playwright + Chromium**; reads computed styles; documented weights in `color-heuristics.ts` (`logo:5, brand:5, primary:4, cta:4, hero:3, button:3, card…footer:2, nav:1`), status-color filter (`error|danger|warning|success|sale…` and Tailwind `red/orange/amber/yellow-N` utilities), structural-noise rule (near-neutral > 40% usage is chrome), platform hints (`theme-color`, `mask-icon color`, `msapplication-TileColor`), logo helpers (`isHomeHref`, `thirdPartyBrandFromAlt`, `isOwnBrandMark`, `isLogoSized ≥ 24px`) | MIT | Heuristic shapes and filter lists. The headless runtime is out of scope. https://github.com/dembrandt/dembrandt [Certain; code read] |
| **Cloudflare "Extract a brand kit"** tutorial (updated 2026-10-12) | Name, description, logo, scheme, ≤ 5 colors, fonts | Puppeteer on Browser Run, `getComputedStyle` over ≤ 4,000 elements; skips saturation < 0.2 and lightness < 0.07 or > 0.95; button/link/brand classes ×3; background ×2 vs text/border ×1; `theme-color` +1000; dedupes by RGB distance < 40; logo order inline SVG, then `<img>`, then icon, then og:image (SVG 16–640 px wide) | Docs sample; needs Browser Run | A minimal baseline to beat. Its naive role naming ("primary" = top score) is what EXT-20 improves on. https://developers.cloudflare.com/browser-run/how-to/extract-brand-kit/ [Certain] |
| **Firecrawl `branding` format**, context.dev styleguide API, Apify actors | Hosted brand extraction | Hosted browser APIs | Paid credits | Not usable at $0 or without third parties. Output schemas are useful references for role names (`primary, secondary, accent, background, textPrimary…`). https://www.firecrawl.dev/glossary/web-extraction-apis/extract-website-branding [Certain] |
| **Wappalyzer open fork** (enthec/webappanalyzer) | Technology detection, including category 17 "Font scripts" | Patterns: Adobe Fonts `use\.typekit\.(?:net|com)`; Google Font API `fonts\.(?:googleapis|google|gstatic)\.com` incl. `style[data-href*='fonts.g']`; Bunny `fonts.bunny.net`; Hoefler `cloud.typography.com`; MyFonts `.myfonts.net`; Font Awesome and other icon sets | **GPL-3.0**, so don't copy | Confirms host patterns; write our own (EXT-58). https://github.com/enthec/webappanalyzer [Certain] |
| **Project Wallace css-analyzer** 9.9.6 | Static CSS stats including colors and fonts | css-tree | MIT | Reference only; we don't need a JS CSS parser. [Certain on license and version] |

### 3.2 Architecture and data flow

```
Browser (all parsing)                                Cloudflare Worker (bytes only)
──────────────────────                               ─────────────────────────────
URL + human check ──POST /api/page──────────────────► validate URL, SSRF, do-not-fetch list
                                                     robots.txt (F-4) → GET page (≤2 MB, ≤5 redirects)
                                                     HTMLRewriter: <link rel=stylesheet|icon|manifest>,
                                                       inline <style> text, ld+json text (for @import/logo refs)
                                                     fetch CSS by priority (EXT-03), @import depth ≤3
◄── {html, stylesheets[], charset, finalUrl, ticket} ┘
html-ingest (DOMParser) → render-probe (sandbox iframe ×3 passes)
css-tokens (CSSOM walk) → color-roles / font-detect / identity
logo-rank → choose ≤5 URLs ──POST /api/assets {ticket}──► allowlist + type/size checks → bytes
◄── image bytes ───────────────────────────────────────┘
worker: decode, size, dHash, logo palette → kit-model (+ manual overrides) → outputs
Uploads (logo / image / PDF) → never leave the browser; same downstream modules
```

- **Why HTMLRewriter on the server:** it is the platform's native streaming parser, so discovery costs little CPU, and the server decides what to fetch, so the client can't turn it into an open proxy. Its CPU cost on a 2 MB HTML page against the 10 ms limit is **not measured** (see Unverified). [Likely]
- **Platform CDN allowlist** for call 2:
  - Squarespace: `images.squarespace-cdn.com`, `static1.squarespace.com`
  - Shopify: `cdn.shopify.com`
  - Wix: `static.wixstatic.com`
  - Webflow: `cdn.prod.website-files.com`, `uploads-ssl.webflow.com`
  - Framer: `framerusercontent.com`
  - WordPress/Jetpack: `i0.wp.com`, `i1.wp.com`, `i2.wp.com`
  - Duda: `irp.cdn-website.com`
  - GoDaddy: `img1.wsimg.com`

  Version the list in the repo. [Likely; hosts seen in practice; verify on the corpus]
- **Worker free-plan numbers:** 100,000 requests/day; error 1027 when exceeded, resetting at midnight UTC; 10 ms CPU; 50 subrequests per request; 6 simultaneous outgoing connections. https://developers.cloudflare.com/workers/platform/limits/ [Certain]
- **Character sets:** the server returns raw bytes plus the declared charset (header or `<meta charset>`). The client decodes with `TextDecoder(charset)`. [Likely]

### 3.3 CSS parsing: browser CSSOM versus JS parsers

| Option | Size (measured) | Pros | Cons |
|---|---|---|---|
| **Browser CSSOM** inside the sandbox iframe (chosen) | 0 KB | Same engine that computes styles. Handles nesting (Baseline widely available since 2026-06-11), `@layer`, `@container`, `@scope`, new color syntax. Rules are walkable (`CSSMediaRule`, `CSSLayerBlockRule`, nested `CSSStyleRule.cssRules`). | Drops what that browser doesn't understand. Specified colors in normal properties re-serialize as `rgb()`. A shorthand using `var()` returns `""` for its longhands (`background-color`), so read the shorthand or `cssText`. Comments are gone, so detect banners such as `/*! tailwindcss v4… */` on raw text. All of this was observed in tests. |
| css-tree 3.2.1 (MIT) | 56.2 KB gz full; 23.6 KB gz for parser + walker + generator | Exact source text, positions, works in workers | Second parser to keep in sync; no cascade |
| postcss 8.5.32 (MIT) + postcss-value-parser 4.2.0 | 14.7 KB gz | Small, stable | Same; no cascade |

[Certain on sizes and observed CSSOM behavior; Likely on choice]

CSSOM observations from the Chrome 153 test, using the probe document from §3.4 [Certain]:
- Custom property values are kept **exactly as authored**: `--brand=#e4572e`, `--bg=light-dark(#ffffff, #111111)`, `--primary=222.2 47.4% 11.2%`, `--sq=41.54, 23.64%, 89.22%`.
- `color: #222` read back as `rgb(34, 34, 34)`; `color: white` stayed `white`.
- `.btn {background: color-mix(in oklab, var(--brand) 80%, black)}`: `getPropertyValue('background')` returned the full text, while `'background-color'` returned `""`.
- Nested rule `.parent { & .child {…} }` is exposed as a child `CSSStyleRule` with selector `& .child`.

### 3.4 The sandboxed iframe probe in detail

**Measured results** (Chrome 153, test site `site.html`), light pass unless stated [Certain]:

| Input CSS | `getComputedStyle` value |
|---|---|
| `h1 {color: var(--brand-2)}` with `--brand-2: var(--brand)` and `--brand: #e4572e` | `rgb(228, 87, 46)` |
| `.btn {background: color-mix(in oklab, var(--brand) 80%, black)}` | `oklab(0.509791 0.118282 0.0874509)` |
| `.btn5 {background: oklch(from var(--brand) calc(l - 0.1) c h)}` | `oklch(0.537239 0.183875 36.477)` |
| `.btn2 {background-color: rgb(var(--tw-ring) / var(--tw-bg-opacity, 1))}` | `rgb(37, 99, 235)` |
| `.btn3 {background: hsl(var(--primary))}` (shadcn triplet) | `rgb(15, 23, 42)` |
| `.btn4 {background: hsla(var(--sq), 1)}` (Squarespace triplet) | `rgb(234, 230, 221)` |
| `.btn6 {background: var(--undefined-token)}` | `rgba(0, 0, 0, 0)` |
| `@layer base { nav a {color:#0a66c2} }` | `rgb(10, 102, 194)` |
| link with no author color | `rgb(0, 0, 238)`; `rgb(158, 158, 255)` when the OS is dark |
| `<button>` font with no author font | `Arial` |
| `body {font-family: "__Inter_d65c78", …}` | `__Inter_d65c78, __Inter_Fallback_d65c78, system-ui, sans-serif` |
| visitor OS dark, no neutralization | body bg `rgb(17, 17, 17)`, h1 `rgb(255, 138, 101)` |
| visitor OS dark, light pass after EXT-09/10 | body bg `rgb(255, 255, 255)`, `--brand` `rgb(228, 87, 46)` |
| dark pass after EXT-09/10 | `--brand` `rgb(255, 138, 101)`, `--bg` `rgb(17, 17, 17)`, dark-only `h2` `rgb(18, 52, 86)` |
| `html.dark` class added | `--brand` `rgb(0, 255, 0)` |

Notes:
- Computed colors come back in **mixed notations**: `rgb()`, `oklab()`, `oklch()`, and in other engines possibly `color(srgb …)`. Every value goes through culori.
- **Browser support** for features the probe relies on, from web-features 3.45.0 (https://github.com/web-platform-dx/web-features) [Certain]:

  | Feature | Baseline status |
  |---|---|
  | `color-mix()` | widely available (2025-11-09) |
  | oklab/oklch | widely available (2025-11-09) |
  | nesting | widely available (2026-06-11) |
  | `light-dark()` | newly available only (2024-05-13) |
  | relative color | newly available only (2024-09-16) |
  | constructed stylesheets | widely available |
  | iframe `srcdoc` and `sandbox` | widely available |
  | forced-colors / `forced-color-adjust` | widely available (2022) |
  | compression streams | widely available (2025-11-09) |

  On older browsers, an unsupported value is dropped at parse time. The sentinel resolver (EXT-11) then reports "unresolved" instead of a wrong color.

**Security analysis** [Likely unless marked]:
- **Scripts:** blocked by the sandbox, because `allow-scripts` is absent. [Certain, tested]
- **Network:** the inherited parent CSP (PRIV-01) plus our meta CSP block all loads. [Certain, tested]
- **Navigation:** strip `meta http-equiv=refresh`. The sandbox blocks top navigation, popups and forms (no `allow-forms`).
- **CSS-only attacks:** CSS exfiltration needs network, which is blocked. Legacy `expression()` is gone from modern engines.
- **Residual risk: denial of service.** Pathological CSS or a huge DOM runs on the app's own thread, because same-origin frames share the event loop. Mitigate with F-1 byte caps, element caps and a watchdog: if a pass exceeds 5 s, discard the iframe and return static-only results.
- **Same origin as the app:** the probe document is same-origin but runs no script. It must never be given `allow-scripts`; MDN warns that combining both lets the frame remove its own sandbox.

### 3.5 Tokens by platform: detection and mapping

| Platform (W3Techs share of all sites, 2026-11-07) | Detect | Color tokens | Font tokens and loading |
|---|---|---|---|
| WordPress (39.4%) | `generator` meta, `/wp-content/` | Core presets `--wp--preset--color--<slug>` on `:root`; the **default palette** (vivid-red `#cf2e2e`, luminous-vivid-orange `#ff6900`, luminous-vivid-amber `#fcb900`, …) is on nearly every site, so discount it (EXT-22). Theme slugs (`base`, `contrast`, `primary`, `accent-N`). Astra `--ast-global-color-0…8`; Kadence `--global-palette1…9`; Blocksy `--theme-palette-color-N`; GeneratePress `--wp--preset--color--accent` | `--wp--preset--font-family--<slug>` |
| Elementor (on WordPress) | body class `elementor-kit-N`, `generator` meta | `--e-global-color-primary/-secondary/-text/-accent` (+ custom ids) **scoped to `.elementor-kit-N`**, not `:root`; usually in `/wp-content/uploads/elementor/css/post-N.css` or inline | `--e-global-typography-<id>-font-family` |
| Shopify (5.3%) | `cdn.shopify.com` | Color scheme classes (`.color-background-1`, `.color-scheme-N`) with `--color-background`, `--color-foreground`, `--color-button`; Dawn uses RGB triplets used as `rgb(var(--color-…))` | `--font-body-family`, `--font-heading-family`; `@font-face` from `fonts.shopifycdn.com` |
| Wix (4.2%) | `generator` "Wix.com Website Builder" | Documented `--wst-*` (e.g. `--wst-button-color-fill-primary`, `--wst-color-fill-accent-1…4`, `--wst-color-text-primary`); legacy `--color_N` triplets | `--wst-font-style-h1…`; proprietary "Madefor" families |
| Squarespace (2.4%) | `static1.squarespace.com` | `--sqsp-palette-{lightest,light,accent,dark,darkest}-color` ("Each variable corresponds to a fixed position in the site's five-color palette"); theme-aware `--sqsp-background-color`, `--sqsp-link-color`, `--sqsp-heading-N-color`; older comma-HSL triplets `--white-hsl`, `--accent-hsl`, `--safeLightAccent-hsl` used as `hsla(var(--x),1)` (seen inline on the owner's site) | Inline `@font-face` from `file.squarespace-cdn.com` (seen) |
| Webflow (0.8%) | `html[data-wf-site]`, `generator` "Webflow" | Variables named by the designer (no fixed scheme) | Google and Adobe via the WebFont Loader script, the "same approach as a published Webflow site" |
| Framer (0.2%) | `generator` "Framer" | Designer-named | `framerusercontent.com` |
| Tailwind v3 | `--tw-` properties; banner on raw text | Utilities like `rgb(37 99 235 / var(--tw-bg-opacity))` (resolved by the probe); **class names carry semantics**: an element with `bg-primary` painting #X is a role hint | Font stacks in utilities |
| Tailwind v4 | `@layer theme, base, components, utilities`; `--color-*` on `:root, :host` | Docs: "By default only used CSS variables will be generated in the final CSS output", so present ≈ used. Defaults are oklch, e.g. `oklch(54.6% .245 262.881)` → `#155dfc`; custom names (not default palette names) are strong brand signals | `--font-sans`, `--font-*` |
| shadcn/ui | `--primary` with `--primary-foreground` pairs; `.dark` overrides | Current: oklch values (default neutral `--primary: oklch(0.205 0 0)` is near-black). Legacy (Tailwind v3 era): bare HSL triplets used as `hsl(var(--primary))` | — |
| Next.js | `/_next/static/`, `__NEXT_DATA__` or `self.__next_f` | — | `next/font` self-hosts Google fonts with hashed family names: webpack `__Inter_98e1b5` (6 hex), Turbopack 8-hex digest, plus `__X_Fallback_<hash>`; CSS variable `--font-*` |

Sources:
- W3Techs: https://w3techs.com/technologies/overview/content_management
- WordPress color presets: https://developer.wordpress.org/themes/global-settings-and-styles/settings/color/ and https://github.com/WordPress/wordpress-develop/pull/1831
- Elementor global styles: https://developers.elementor.com/docs/editor-controls/global-style ; `.elementor-kit-7{--e-global-color-primary:#339900` seen in the wild via search ; https://wpdesignvault.com/2026/10/14/elementor-css-variables/
- Astra: https://wpastra.com/docs/custom-global-colors/
- Kadence, Blocksy, GeneratePress: https://jetonomy.org/docs/integrations/07-theme-compatibility/
- Shopify: https://shopify.dev/docs/api/liquid/objects/color_scheme_group and https://github.com/Shopify/dawn/blob/main/layout/theme.liquid
- Wix: https://dev.wix.com/docs/develop-websites/articles/coding-with-velo/frontend-code/custom-css/site-theme-css-variables (last updated 23 Jan 2025)
- Squarespace: https://developers.squarespace.com/custom-code/css and https://forum.squarespace.com/topic/245257-how-do-i-use-squarespace-color-variables-in-my-css/
- Webflow: https://developers.webflow.com/devlink/docs/component-export/fonts
- Tailwind: https://tailwindcss.com/docs/theme
- shadcn: https://ui.shadcn.com/docs/theming
- next/font: https://github.com/mui/material-ui/issues/38540 and https://github.com/vercel/turborepo/pull/3049

[Certain for rows backed by official docs or my observation; Likely for Kadence, Blocksy, Framer and Wix legacy `--color_N`]

### 3.6 Color parsing and conversion: culori versus colorjs.io

Parse test (Node, same inputs to both) [Certain]:

| Input | culori 4.0.2 | colorjs.io 0.7.1 |
|---|---|---|
| `#3b82f680`, `rgb(37 99 235 / .5)`, `rgba(…)`, `hsl(…)` (space or comma), `hwb()`, `lab()`, `lch()`, `oklab()`, `oklch()`, named colors, `none` | all parsed | all parsed |
| `color(display-p3 1 .5 0)` | parsed; `formatHex` **clips**: `#ff7600` (`toGamut` gives `#ff7d00`) | parsed; hex output **gamut-mapped**: `#ff7d00` |
| `color-mix(…)`, `light-dark(…)`, `oklch(from …)` | **undefined** | **error** |
| `currentcolor` | undefined | error |
| WCAG contrast `#777` / `#fff` | 4.478089453577214 | 4.478089453577214 |

Bundle sizes (esbuild minify + gzip -9) [Certain]:

| Import | Size |
|---|---|
| culori/fn with 14 modes + parse, formatHex, toGamut, interpolate, ΔE2000, wcagContrast | **11,637 B** |
| culori full | 17,295 B |
| colorjs.io/fn with the same spaces and functions | 21,458 B |
| colorjs.io full `Color` | 33,763 B |

Both are MIT. culori was published 2025-06-27 and colorjs.io 0.7.1 on 2026-07-24.

**Choose culori**: it is about half the size, its accuracy is equal on the formats we need, and gamut mapping is one explicit call. Let the browser resolve mix, light-dark and relative syntax. [Likely]

### 3.7 Role inference: rationale

- **Painted evidence beats declared evidence.** That matches the competitor's claim, dembrandt and Cloudflare. Tokens alone mislead: WordPress ships unused default presets, and shadcn's "primary" defaults to near-black. Tokens only boost painted colors (EXT-19, EXT-22). [Likely]
- **Button backgrounds are the strongest single signal for primary.** Dembrandt weights `primary/cta` 4 and `button` 3, and Cloudflare gives buttons ×3. Large section and header backgrounds point to secondary, and links and small highlights to accent. [Likely]
- **Neutral handling.** Cloudflare drops saturation < 0.2 outright. That loses brands whose identity is black, navy-gray or cream. EXT-20 lets a neutral become primary only on button evidence. [Likely]
- **Status colors.** Discount colors that appear only in alert, error, success or sale contexts. Dembrandt's `STATUS_CONTEXT_SOURCE` regex is a good start. [Likely]
- **Dark theme.** If a dark pass ran, include "Dark theme: background X, text Y" as secondary information, without extra role slots. [Guessing on presentation]

### 3.8 Font detection details

- **Icon-font filter.** A starting list; match case-insensitively by substring:
  - Font Awesome (all versions, incl. "FontAwesome"), Material Icons, Material Symbols, Material Design Icons
  - Glyphicons, bootstrap-icons, Ionicons, icomoon, dashicons (WordPress), eicons (Elementor), ETmodules (Divi), fontello, Genericons
  - themify, simple-line-icons, Linearicons, feather, remixicon, boxicons, tabler-icons, socicon, WooCommerce, `star` (WooCommerce rating font)
  - slick, swiper-icons, revicons, fl-icons, ElegantIcons, Flaticon

  Also treat a family as an icon font if its `@font-face` `unicode-range` falls only in the Private Use Area (U+E000–F8FF). [Likely; the list is Guessing and must be maintained]
- **System families,** labeled "System font (varies by device)": Arial, Helvetica, Helvetica Neue, Segoe UI, Roboto *when it appears after system aliases*, San Francisco/SF Pro, Georgia, Times New Roman, Times, Verdana, Tahoma, Trebuchet MS, Courier New, Menlo, Monaco, Consolas, Ubuntu, Cantarell, Noto Sans, Liberation Sans. [Likely]
- **The Tailwind default `font-sans`** begins `ui-sans-serif, system-ui, sans-serif, …` (v4 docs show `-apple-system, BlinkMacSystemFont, …`). Report it as "System font stack". [Certain on the docs example]
- **Google Fonts URL parsing.**
  - css2: `family=Inter:wght@400;700&family=Playfair+Display:ital,wght@0,400`; split on `&family=` and take the text before `:`, with `+` as a space.
  - v1: `family=Open+Sans:400,700|Roboto`.
  - Bunny is "a fully compatible drop-in replacement for the Google Fonts V1 API" (https://bunny.net/blog/bringing-privacy-back-into-your-own-hands-introducing-bunny-fonts/), so the same parser works with a different host.

  [Certain on Bunny compatibility; Likely on the parsing]
- **WebFont Loader.** Webflow and older sites load `ajax.googleapis.com/ajax/libs/webfont/1.6.26/webfont.js` and call `WebFont.load({...})` inline (https://developers.google.com/fonts/docs/webfont_loader). Read the families from the inline script *text* with a regex on `families:\s*\[([^\]]*)\]` and `typekit:\s*{\s*id:\s*['"]([a-z0-9]+)`. Never execute the script. [Likely]

### 3.9 Logo ranking: notes

- **Home-link signal.** Dembrandt reports, from its labeled set, that no confirmed logo has a longest edge under 24 px, while header search, menu and social glyphs are 16–20 px squares. It also treats `/en`, `/de` and `/en-us` as home links. [Certain from source comments]
- **Squarespace evidence** (owner's site), all naming the same file `WTJP+Logo.PNG`:
  - the header logo: `<div class="header-title-logo"><a href="https://www.wethejamesphoto.com/"><img src="…/WTJP+Logo.PNG?format=1500w" alt="We the James Photography">`;
  - JSON-LD `WebSite.image`;
  - `og:image` (712×500).

  URL normalization must drop `?format=` to dedupe them. Three-way agreement should produce high confidence. [Certain on observation]
- **Google's logo guidance** used to require at least 112×112 px for `Organization.logo`. The logo page now redirects into the Organization doc, and I did not re-verify the number. [Guessing; see Unverified]

### 3.10 Brand name and tagline: notes

- Google: "Our site name system will also consider content in og:site_name, `<title>`, heading elements, and other text on a home page. However, WebSite structured data is most important." https://developers.google.com/search/docs/appearance/site-names [Certain]
- Organization properties Google recognizes include `name`, `alternateName`, `description`, `logo`, `url` and `sameAs`. https://developers.google.com/search/docs/appearance/structured-data/organization [Certain]
- `slogan` is a schema.org Organization property but not on Google's list. Use it when present. [Likely]

### 3.11 JavaScript-rendered sites, blocked sites, and Browser Run

- **How often static fetch fails is not known** for our audience; measure it on the corpus.
  - The Web Almanac 2024 SEO chapter compared raw and rendered word counts on desktop: "a 32% difference … at the 10th percentile, 22% … at the 25th percentile, 18% at the median, 16% at the 75th percentile, and 16% at the 90th percentile." Rendering changed canonicals on only 1.9% of desktop pages. https://almanac.httparchive.org/en/2024/seo
  - The large hosted builders render on the server: WordPress, Shopify, Squarespace and Webflow serve full HTML; Wix and Framer render server-side. Those builders cover most of the audience (W3Techs: 32.2% of sites use no monitored CMS).

  [Certain on the Almanac numbers; Likely that builder sites mostly work statically; the share of client-rendered shells is unknown]
- **Where static fetch breaks:**
  - client-rendered SPAs: CSS is often still linked (static tokens work) but the DOM is empty;
  - CSS-in-JS injected at runtime: no CSS at all;
  - bot challenges, geo blocks and consent walls (stop, per F-5).

  [Likely]
- **Cloudflare Browser Run** (formerly Browser Rendering), per https://developers.cloudflare.com/browser-run/pricing/ (last updated 2026-04-21) [Certain]:
  - Workers Free: **10 minutes per day** of browser time and **3 concurrent browsers**.
  - Paid: 10 hours per month, then $0.09/hour.
  - Cloudflare's own tutorial uses `networkidle2` with a 30 s timeout and recommends caching results in KV. Caching would break "stores nothing".

  At roughly 5–15 s of browser time per render, 10 minutes covers about 40–120 renders a day. [Guessing on render time] Rejected for v1 (§2, item 1).

### 3.12 Image palette: experiment and libraries

**Experiment** (`/tmp/claude-0/-home-user/02d649eb-8126-5ff7-985f-ad69e6e47e14/scratchpad/libtests/palette.mjs`) [Certain]. The test image is 300×120 on a transparent background:
- an orange disc `#e4572e` with anti-aliased edge;
- navy bars `#1e3a8a`;
- a white knockout;
- a 1-px orange/navy blend seam `#81485c`.

Results:
- **MMCQ** (`quantize` 1.0.2, alpha ≥ 125, non-white): `#e4542c #1c3c8c #844c5c #50485c #1c4890`. Both brand hex values are off, and two of the five colors are not in the image at all.
- **OKLab k-means with exact-pixel medoids** (k = 5, seeded): `#1e3a8a 54.4% | #e4572e 40.2% | #ffffff 5.0% | #81485c 0.5%`. The hex values are exact, and the 0.5% blend seam drops out under the 2% floor and the blend test.

| Library | Version (published) | License | Size (gz) | Notes |
|---|---|---|---|---|
| colorthief | 3.5.0 (2026-08-02) | MIT | 8.2 KB | MMCQ; `colorSpace:'oklch'` default; `alphaThreshold 125`; `ignoreWhite`; worker-friendly. Returns box averages (not exact pixels) [Likely] |
| quantize | 1.0.2 (2016) | MIT | 2.2 KB | MMCQ core; unmaintained |
| image-q | 4.0.0 (2022) | MIT | 13.5 KB | Wu/NeuQuant + CIEDE2000; stale |
| extract-colors | 4.2.1 (2025) | MIT | 2.3 KB | Simple binning |
| node-vibrant (browser) | 4.0.4 (2026) | MIT | 5.3 KB | Semantic swatches; MMCQ-based |
| **Own OKLab k-means** (chosen) | — | ours | ~2–3 KB est. | Exact medoids, deterministic, uses culori already in the bundle [Guessing on size] |

### 3.13 SVG sanitizing and variants: test notes

DOMPurify 3.4.17 with the SVG profile was given a test SVG containing `onload`, `<script>`, `<foreignObject>` with an `<img onerror>`, `javascript:` in an `<a href>`, external `<use>`, external `<image>`, and `@import` inside `<style>`. The output kept `<style>…@import url(http://…)</style>` and `<image href="http://…">` and removed everything else listed. [Certain] So the extra hook in EXT-42 is required.

DOMPurify needs a DOM, so run it on the main thread. SVGs are small, so this is fast. [Likely]

### 3.14 Brand-guide PDF: details

- **CVE-2024-4367:**
  - What it is: "If pdf.js is used to load a malicious PDF, and PDF.js is configured with `isEvalSupported` set to `true` (which is the default value), unrestricted attacker-controlled JavaScript will be executed in the context of the hosting domain."
  - Mechanism: an injected `FontMatrix` reaching a `new Function` body.
  - Fix: PR #18015, "remove the use of eval in the font loader", shipped in 4.2.67.
  - Status: pdfjs-dist 6.5.438 has no `isEvalSupported` and no `new Function` in `build/pdf.mjs` (grep, 2026-11-08).

  Sources: https://github.com/advisories/GHSA-wgrm-67xf-hhpq ; https://codeanlabs.com/2024/05/cve-2024-4367-arbitrary-js-execution-in-pdf-js/ ; https://github.com/mozilla/pdf.js/pull/18015 [Certain]
- **OSV query, 2026-11-08:** 0 vulnerabilities for pdfjs-dist 6.5.438, 4.2.67, and DOMPurify 3.4.17. https://osv.dev [Certain]
- **Asset sizes** (6.5.438, raw): `cmaps/` 1.7 MB, `standard_fonts/` 820 KB, `wasm/` 1.4 MB (jbig2, openjpeg, qcms, quickjs), `iccs/` 24 KB. Self-host only what's needed: cmaps and standard fonts for text in CJK or non-embedded fonts; image codecs only if rendering pages. [Certain on sizes]
- **Embedded font names.** In v6 the font `name` is in `EXPORT_DATA_EXTRA_PROPERTIES`, sent only when `fontExtraProperties:true` (source read). [Certain]
- **Regexes.** Starting point; test against real guides:
  - HEX: `(?<![\w#])#([0-9a-f]{6}|[0-9a-f]{3})(?![0-9a-z])` and `\b(?:hex|html|web)\b\s*[:#]?\s*#?([0-9a-f]{6})\b`. The label is required when `#` is absent, because 6-digit numbers such as Pantone or phone fragments otherwise match.
  - RGB: `\bR(?:GB)?\s*[:=]?\s*(\d{1,3})\s*[,/ ]\s*(?:G\s*[:=]?\s*)?(\d{1,3})\s*[,/ ]\s*(?:B\s*[:=]?\s*)?(\d{1,3})\b`, values 0–255.
  - CMYK: `\bC(?:MYK)?\s*[:=]?\s*(\d{1,3})%?\s*[,/ ]\s*(?:M\s*[:=]?\s*)?(\d{1,3})%?\s*[,/ ]\s*(?:Y\s*[:=]?\s*)?(\d{1,3})%?\s*[,/ ]\s*(?:K\s*[:=]?\s*)?(\d{1,3})%?`, values 0–100.
  - Pantone: `\b(?:PANTONE|PMS)\s*®?\s*([0-9]{3,4}(?:\s?[A-Z]{1,3})?|[A-Z][A-Za-z]+(?:\s[A-Z][A-Za-z]+)*\s(?:C|U|CP|UP|TCX|TPX))\b`.

  [Guessing; fuzz and tune]

### 3.15 Color system: tested ramp outputs

Relative-chroma OKLCH ramps (`/tmp/claude-0/-home-user/02d649eb-8126-5ff7-985f-ad69e6e47e14/scratchpad/libtests/ramp2.mjs`; * = brand anchor) [Certain]:
- `#e4572e`: `#fdf2ef #fbe1da #f8c7b9 #f5a38c #f17451 #e4572e* #b64525 #92361c #702813 #501a0b #320d04`
- `#2563eb`: `#f1f5fd #dee8fb #c2d5f8 #9cbbf4 #729df0 #4c81ec #2563eb* #164bc1 #0e3895 #07266b #031544`
- `#0f766e`: light stops came out too saturated (`#96fef2`). That is the reason for the "cap at brand chroma" rule in EXT-52.

WCAG check on the naive ramp of `#e4572e`: the brand color is 3.68:1 on white (large text AA only) and 5.70:1 on black (AA). The 600 stop (`#c13703`) reaches 5.50:1 on white. This is the kind of row the "pairs that pass" list will show. [Certain]

### 3.16 Typography system

Covered by EXT-55 and EXT-56. The metrics package that can compute size-adjusted fallbacks the way next/font does: `@capsizecss/metrics` 4.4.0 (MIT, updated 2026-10-15). [Certain on package facts]

### 3.17 Known failure modes

| # | Failure | Effect | Mitigation | Tag |
|---|---|---|---|---|
| 1 | Brand colors live only in images (hero photo, raster logo) | CSS yields neutrals only | EXT-39 logo palette; prompt to upload a screenshot | Likely |
| 2 | Cookie, chat, review and booking widgets | Saturated non-brand colors | EXT-13 exclusions | Likely |
| 3 | Platform defaults (WordPress presets, Tailwind and Bootstrap defaults) | False tokens | EXT-22; painted evidence only | Certain (WP defaults exist) |
| 4 | Browser default styles | Link blue, buttonface gray, Arial | EXT-08 UA baseline | Certain |
| 5 | Visitor OS dark mode, forced colors, hover or pointer differences | Results differ per visitor | EXT-09/10 | Certain |
| 6 | Mid-animation values (fade-ins, Squarespace `preScale scaleIn`) | Transparent or odd colors | Animations off (EXT-10) | Likely |
| 7 | Invalid or undefined `var()` | Silent inherited value | Sentinel resolver | Certain |
| 8 | Gradients as brand | Missed | Parse stops at low weight; list under "Other colors" | Likely |
| 9 | SPA shell or runtime CSS-in-JS | Empty DOM, missing CSS | EXT-36 partial UI | Likely |
| 10 | Bot challenges, 403, geo, consent walls | No data | EXT-37 stop + uploads | Likely |
| 11 | Badge walls in the header (GAF, BBB, Angi) | Wrong logo | EXT-30 negative tokens | Likely |
| 12 | Logo as CSS background, JS-injected, or external SVG sprite | Missed | CSS bg detection; JSON-LD, og:image and icon fallbacks | Likely |
| 13 | Icon font first in stack; Typekit slugs; Wix proprietary fonts | Wrong or ugly family names | EXT-25 filters; show the slug, substitute per T-5 | Likely |
| 14 | Font declared but not loaded | Report would name a font nobody sees | "declared, not loaded" label (EXT-25) | Likely |
| 15 | > 10 stylesheets or > 3 MB CSS | Missing tokens | EXT-03 priority; report skipped files | Certain (Squarespace: 16) |
| 16 | Non-UTF-8 pages | Mojibake names | Decode with the declared charset | Likely |
| 17 | `meta refresh` or JS redirects | Analyzes a stub page | Detect; offer "Analyze <target> instead?" on click | Likely |
| 18 | Declarative shadow DOM or web components | Styles not visible | Out of scope v1; flag partial | Guessing |
| 19 | PDF with outlined or scanned text | No codes | Operator-list fills; page render + palette | Likely |
| 20 | CMYK-only or Pantone-only guides | No exact screen color | Label approximations; Pantone as text only | Likely |
| 21 | SVG with embedded raster or live text | Variants wrong | Warn; ask for outlined SVG | Likely |
| 22 | Older browser without `light-dark()` or relative color | Tokens unresolved | Mark unresolved; never guess | Certain (parse-time drop) |

### 3.18 Test strategy and fixture corpus

**Synthetic fixtures** (public, in repo, CI on Chromium, Firefox and WebKit). One mini-site per pattern, about 35:
- **Tailwind:** v3 opacity vars; v4 oklch theme; v4 custom `--color-brand`.
- **shadcn:** v3 HSL triplets; v4 oklch with `.dark`.
- **WordPress:** presets with defaults only; block theme `primary`.
- **Builders:** Elementor kit scoped to body; Astra global colors; Squarespace comma-HSL + `--sqsp-palette-*`; Wix `--wst-*`; Shopify color schemes; Webflow WebFont.load; next/font hashed names.
- **Font loading:** Typekit link; Bunny; Fontshare; Google css2 multi-family; Next `style[data-href]`; icon font first in stack.
- **Distractors:** cookie banner with saturated color; UA-default-only page.
- **Environment:** OS dark preference; forced colors.
- **Color syntax:** `color-mix`, relative color, `light-dark`, `@layer`, nesting, an undefined var, a gradient brand.
- **Page shapes:** SPA shell; 403/challenge page; `meta refresh`.
- **Logos:** inline SVG sprite logo; CSS background logo; badge wall in the header; JSON-LD logo only; og:image banner only; Squarespace triple-agreement logo.
- **Malicious inputs:** SVG; PDF with a FontMatrix payload; HTML with scripts and handlers.

Each fixture has an `expected.json`.

**Real corpus** (30 sites; local snapshots only, EXT-60). The mix roughly follows W3Techs shares and the audience:

| Group | Count | Notes |
|---|---|---|
| Squarespace 7.1 | 4 | Includes wethejamesphoto.com |
| Wix | 4 | Incl. 1 Wix Studio |
| WordPress | 6 | 2 Elementor, 1 Astra, 1 Kadence or Blocksy, 1 block theme, 1 Divi |
| Shopify | 3 | Dawn-family, newer theme, custom |
| Webflow | 3 | |
| Tailwind / Next.js | 4 | v3, v4, shadcn-oklch, next/font; one pure client-rendered SPA as the expected-partial case |
| Framer | 1 | |
| GoDaddy or Duda builder | 2 | |
| Small DFW home-service sites | 3 | Roofing, junk removal, plumbing; at least one with a badge wall |

**Ground truth.** The owner, a designer, labels each site from visual inspection and any public brand materials:
- allowed hex values per role (with alternates);
- `mustNotInclude` colors (widgets);
- heading and body family;
- expected source label, or "none";
- logo URL pattern;
- name;
- allowed taglines or null;
- `expectPartial`.

**Harness and measures:**
- The harness runs the real client code in Playwright against snapshot bytes, served by a local stub of the Worker API.
- Report per-metric scores against the EXT-61 targets.
- The "invented colors = 0" check is a property test: every output color must trace to a provenance record whose raw value parses to the same hex within ΔE_OK 0.02, or is flagged `derived`.
- Determinism: 10 runs plus a three-engine diff.
- Performance: Playwright CPU throttling at 4×.
- Worker CPU: run `wrangler dev` locally against snapshot hosts, not production logs, to stay consistent with PRIV-06/07.

[Likely]

### 3.19 Library summary (measured 2026-11-08)

| Package | Version | Published | License | Min+gz | Use |
|---|---|---|---|---|---|
| culori (`culori/fn` subset) | 4.0.2 | 2025-06-27 | MIT | 11.6 KB | **Yes**: all color math |
| colorjs.io (fn / full) | 0.7.1 | 2026-07-24 | MIT | 21.5 / 33.8 KB | No |
| css-tree (full / parser+walker+generator) | 3.2.1 | 2026-03-05 | MIT | 56.2 / 23.6 KB | No |
| postcss + postcss-value-parser | 8.5.32 / 4.2.0 | 2026-10-26 | MIT | 14.7 KB | No |
| DOMPurify (SVG profile) | 3.4.17 | 2026-10-12 | MPL-2.0 OR Apache-2.0 | 12.1 KB | **Yes**: SVG uploads and logos |
| pdfjs-dist (main / worker) | 6.5.438 | 2026-11-01 | Apache-2.0 | 136 / 378 KB | **Yes**: lazy, PDF only |
| colorthief | 3.5.0 | 2026-08-02 | MIT | 8.2 KB | No (box averages) |
| quantize | 1.0.2 | 2016-04-06 | MIT | 2.2 KB | No |
| image-q | 4.0.0 | 2022-01-08 | MIT | 13.5 KB | No |
| extract-colors | 4.2.1 | 2025-08-04 | MIT | 2.3 KB | No |
| node-vibrant (browser) | 4.0.4 | 2026-01-27 | MIT | 5.3 KB | No |
| apca-w3 | 0.1.9 | 2022-07-04 | "Limited W3 License", All Rights Reserved | — | **No** |
| @capsizecss/metrics | 4.4.0 | 2026-10-15 | MIT | build-time only | Optional |
| web-features (Baseline data) | 3.45.0 | — | — | dev only | Reference |

[Certain]

---

## 4. Unverified / needs follow-up

1. **Firefox and Safari behavior of the sandbox probe.** All browser tests ran on Chrome 153 only. To check in other engines:
   - `CSSMediaRule.media.mediaText` rewriting;
   - computed color serialization formats;
   - `forced-color-adjust` effect;
   - srcdoc CSP inheritance (specified in CSP3 but untested here);
   - layout and computed styles in an off-screen `visibility:hidden` iframe.

   Run the synthetic fixtures on Playwright Firefox and WebKit first.
2. **How often static fetch fails** (empty shells, CSS-in-JS, bot challenges against an honest User-Agent) for small-business sites. No primary data was found; measure it on the corpus.
3. **HTMLRewriter CPU cost** on 0.5–2 MB HTML against the 10 ms free-plan limit. Not measured. If it's too high, fall back to a client-driven discovery call with the EXT-02 allowlist.
4. **Next.js 15/16 (Turbopack) `next/font` family-name format.** Confirmed only for webpack (6-hex `__Inter_98e1b5`) and the 2022 Turbopack PR (8-hex). Check a current Next 16 site.
5. **Elementor kit CSS location.** It may be an external `post-N.css` file or inline, depending on the "CSS Print Method" setting. Also check the Divi 5 and Shopify "Horizon" theme variable names. None of these were verified.
6. **Wix markup on live sites:** legacy `--color_N` versus documented `--wst-*` prevalence, the "Wix Harmony" variables, and whether Wix pages pass the empty-shell check.
7. **Google `Organization.logo` minimum size** (112×112 px recalled). The dedicated logo doc now redirects; not re-verified.
8. **All numeric thresholds marked [Guessing]:** ΔE distinctness 0.08, neutral C < 0.03, weights, slot ratios, logo points, palette floors. Tune on the corpus and freeze before launch.
9. **pdf.js v6 font-name retrieval** (`commonObjs.get(id).name` with `fontExtraProperties`) and **CMYK-to-RGB fidelity in operator-list fills**. Confirmed by reading the source only; test on 5–10 real brand guides.
10. **`createImageBitmap` support** for `resizeQuality:'pixelated'` and for ICO and AVIF decoding in Safari. Not tested.
11. **colorthief 3.x OKLCH mode.** It may return exact colors; I only tested classic MMCQ (`quantize`). This doesn't change the recommendation, since our own code is small.
12. **Phone performance.** The 1.5–2.5 s main-thread estimate is extrapolated from a 4-vCPU container. Measure on a real mid-range Android device.
13. **Platform CDN allowlist** (§3.2): hosts are from experience and one observed site. Verify each on the corpus.
14. **Bot-protection behavior** of Cloudflare, Wix and Squarespace toward a self-identified fetcher (F-3 User-Agent). Unknown; affects the blocked-site rate.
15. **Typekit page-view counting** through kit resources (§2, item 7). Unverified; moot if kit CSS is never fetched.
16. **Competitor internals.** Everything about Blendin comes from its marketing page; its actual accuracy is unknown.
17. **Google Fonts catalog build source.** Choose the Fontsource API, the `google/fonts` GitHub metadata, or the Google Fonts Developer API (needs a key). Licensing and refresh cadence are to be confirmed with the font-sourcing track (T-1, PRIV-02).
18. **Network-access limits during this research:** `WebFetch` could not reach blendin.ai or developers.cloudflare.com from this session. Those pages were read through the Firecrawl scraper instead, so their content is still primary-source.
