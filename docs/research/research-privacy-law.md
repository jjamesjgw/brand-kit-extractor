# Research: privacy, child-safety, marketing-consent and accessibility law

**Product:** free, anonymous, no-account brand-kit tool. Operated by Justin James, an individual (sole proprietor) in Rowlett, Texas, USA. Hosted on a subdomain of wethejamesphoto.com. Reachable worldwide.
**Research date:** 2026-11-08
**Confidence tags:** [Certain] = primary source read or text quoted. [Likely] = strong inference or consistent secondary sources. [Guessing] = gap-filling.
**Scope note:** This is research for a product spec, not legal advice from a licensed attorney. Where the law is unsettled, the report says so.

---

## 0. Bottom line

1. Almost every privacy, child-privacy and marketing law reviewed either **does not apply** or applies only in a minimal form. That holds as long as the tool keeps to the locked decisions: no PII fields, no server storage, uploads stay in the browser, no tracking. [Likely]
2. The **one category of personal data the tool cannot avoid** is network metadata: visitor IP address, user agent and request time. The host/CDN sees it, and so does the bot check (Cloudflare Turnstile). IP addresses are "personal information" under COPPA (a persistent identifier) and "personal data" under GDPR and UK GDPR. So the obligations that remain are mostly about **(a) keeping that metadata minimal and short-lived, (b) never leaking it to extra third parties (fonts, logos, analytics, CDNs), and (c) describing it accurately in a privacy policy.** [Certain] on the definitions; [Likely] on the conclusion.
3. **The biggest real legal risk is deception, not non-compliance.** The competitor's tagline-level promises ("stores nothing", "uploads never leave your device") become false the moment a font, script, logo or analytics beacon loads from a third-party host, or the host keeps request logs. A false privacy promise is a deceptive practice under FTC Act §5 and the Texas DTPA. Every privacy claim in the UI and policy must be **literally true at the network level**. [Likely]
4. **No cookie banner, no age gate, no EU/UK representative and no ICO fee are needed.** This holds if the spec follows the requirements in Section 1. Each has a condition, listed in Sections 3 and 5. [Likely]
5. **Hard conflicts with locked decisions.** None is fatal; all are resolvable at $0.
   - **C1. GDPR/UK GDPR representative vs. $0.** If GDPR Art. 3(2) were triggered, Art. 27 would require an EU representative and a UK representative. Commercial representative services charge fees, which breaks the $0 rule. Resolution: design the tool so Art. 3(2) is not triggered: no EU/UK targeting and no behavioral monitoring. Section 3.2 has the details. [Likely]
   - **C2. Google Fonts picker vs. the privacy claims.** Loading Google Fonts from Google's CDN in the visitor's browser sends the visitor's IP to Google. A German court awarded damages for exactly this (LG München I, 20 Jan 2022, 3 O 17493/20). Resolution: serve fonts same-origin (self-host or proxy). [Certain] on the case; [Likely] on the fix.
   - **C3. "Stores nothing" vs. host logs.** Every host logs IPs for security. Cloudflare Workers Logs keeps 3 days on the free plan today and 7 days after the 1 Dec 2026 pricing change. GitHub Pages also logs visitor IPs. Resolution: say "we don't store your inputs", and disclose the provider logs with the actual retention period. [Likely]
   - **C4. A privacy contact is needed.** Art. 13 GDPR requires controller contact details, so the privacy policy needs a working contact email. The email must cost nothing: a free mailbox or the domain registrar's free forwarding, not a paid Google Workspace or Squarespace mailbox. [Likely]
   - **C5. Cookieless analytics vs. no banner.** Cloudflare Web Analytics' JS beacon sets no cookies, but under the EDPB's reading a script that makes the browser send device data is still "gaining access" under ePrivacy Art. 5(3). Resolution: no JS analytics in v1. Use only edge-side request metrics that need no client code. [Likely]

---

## 1. Concrete requirements for this product

IDs (PRIV-xx) are for the spec. Each requirement lists its legal driver.

### A. Network and data-flow architecture

- **PRIV-01. Same-origin-only browser requests.** The app shell, share view, generated-PDF preview and PNG renderer may make network requests only to (a) the tool's own origin and (b) `challenges.cloudflare.com`, for the bot check only, and only when the user starts a URL fetch. Enforce this with a Content-Security-Policy header, roughly: `default-src 'self'; script-src 'self' https://challenges.cloudflare.com; frame-src https://challenges.cloudflare.com; connect-src 'self'; img-src 'self' data: blob:; font-src 'self' data:; style-src 'self' 'unsafe-inline'; object-src 'none'; base-uri 'none'; form-action 'self'`. The `'unsafe-inline'` for styles is a build-time trade-off. Drivers: GDPR Art. 5(1)(c) minimization, ePrivacy Art. 5(3), FTC Act §5 truthfulness, LG München I 3 O 17493/20. [Likely]
- **PRIV-02. Fonts are served same-origin.** UI fonts and every font the Google Fonts picker can preview are served from the tool's own origin. They are either self-hosted (downloaded at build time; most Google Fonts are SIL OFL 1.1 or Apache 2.0, which allow redistribution) or fetched through the tool's own server function and cached. **Never** link `fonts.googleapis.com`, `fonts.gstatic.com`, `use.typekit.net`, `fonts.bunny.net` or jsDelivr/unpkg font CSS from the browser. The picker's font list must be a static JSON file built at deploy time, not a live call to the Google Fonts Developer API from the browser. Google's own FAQ says that when fonts are self-hosted "Google does not receive any kind of data related to the visits". [Certain] on the Google FAQ and the ICO font guidance; [Likely] on the build approach.
- **PRIV-03. Logos and favicons from scanned sites are never hotlinked.** The server-side fetch downloads the candidate logo, favicon and og:image bytes and returns them to the browser as data/blob payloads. The visitor's browser never contacts the scanned site. This applies to the results view, the PDF, the PNG and the share view. [Likely]
- **PRIV-04. Don't render the scanned site's own web-font files.** When a scanned site uses a font that isn't in the self-hosted or proxied catalog (for example an Adobe Fonts or commercial self-hosted font), show its name plus a fallback stack. Do not load the site's font file into the visitor's browser or embed it in the PDF. This avoids both an IP leak and a font-license problem. [Likely]
- **PRIV-05. Third-party library assets are bundled.** pdf.js (worker, CMaps, standard fonts), color-quantization libraries and SVG sanitizers are bundled and served same-origin. pdf.js defaults to fetching CMaps and standard-font data by URL, so point those at local paths. [Likely]
- **PRIV-06. Server-side URL fetch rules.**
  - (a) The browser sends the URL in a POST body, not a query string, so it never appears in access-log URLs.
  - (b) Normalize to scheme + host + path. Drop query strings and fragments, which may carry emails or reset tokens. Reject URLs with embedded credentials (`user:pass@`).
  - (c) Do not forward the visitor's IP, cookies, `Accept-Language` or user agent to the target. Use a fixed, honest user agent that names the tool and links its about page.
  - (d) Parse in memory and return only the extracted fields: colors, fonts, logo candidates, favicon, name, tagline, contrast results. Never write fetched HTML, CSS or images to KV, D1, R2, a cache or logs.
  - (e) Do not log the target URL. If the code has `console.log`, do not log request bodies.

  Drivers: GDPR Art. 5(1)(c) and 5(1)(e); FTC §5. [Likely]
- **PRIV-07. Host logging and retention, set and documented.** Either disable platform request/invocation logging (Workers Logs / observability), or keep it at the minimum retention. Record the actual retention in the privacy policy. As of this research, Workers Logs on the free plan keeps 3 days. From 1 Dec 2026, Cloudflare Observability pricing gives the free plan 0.5 GB/day ingestion with 7-day retention. [Likely; see Unverified]
- **PRIV-08. The bot check loads only when needed.** Load Turnstile only when the user clicks "Fetch" on a URL, not on page load. Users who only upload files or type colors then never contact Cloudflare's challenge endpoint. Do not send the optional `remoteip` parameter to Siteverify unless abuse forces it. Do not enable Turnstile pre-clearance, which sets the `cf_clearance` cookie. [Certain] that pre-clearance sets `cf_clearance`; [Likely] on the minimization design.
- **PRIV-09. No client-side analytics in v1.** That means no Cloudflare Web Analytics JS beacon, Google Analytics, Plausible script, Hotjar, Sentry, LogRocket or session replay. If the owner wants traffic numbers, use the host's edge/request metrics, which need no script in the page. Drivers: EDPB Guidelines 2/2023 (JS that makes the browser send information is "gaining access" under Art. 5(3)); UK PECR statistical-purposes exception requires notice plus an opt-out; COPPA (behavioral uses are outside "support for internal operations"). [Likely]
- **PRIV-10. Parent-domain cookie hygiene.** The tool lives on a subdomain of wethejamesphoto.com. If the Squarespace site sets any cookie with `Domain=wethejamesphoto.com`, browsers attach it to every request to the tool. Squarespace's documented analytics cookies `ss_cvr` and `ss_cid` identify unique visitors for 2 years. The server code must never read, log or forward the `Cookie` request header. Verify in browser DevTools which Squarespace cookies are domain-wide (see Unverified). The tool itself sets **no cookies**. [Likely]
- **PRIV-11. Referrer and outbound links.** Send the `Referrer-Policy: strict-origin-when-cross-origin` header or stricter (`no-referrer` is fine). Give outbound links, including the "Built by Justin James" footer link and links to scanned sites, `rel="noopener noreferrer"`. URL fragments are never sent in `Referer` headers, but the path could be. [Likely]

### B. Browser storage

- **PRIV-12. The draft lives on the device and is clearable.** Keeping the in-progress kit in `localStorage` or `IndexedDB` is "storage" under ePrivacy Art. 5(3) and UK PECR reg. 6. That is true even though nothing leaves the device. It qualifies for the **strictly-necessary exemption** only because it delivers a service the user explicitly requested. Requirements:
  - (a) Keep the draft only for that purpose.
  - (b) Show one line of microcopy near the editor: "Your draft is saved in this browser only."
  - (c) Provide a visible **"Clear saved draft"** control.
  - (d) Never add an identifier, timestamp-based ID or anything readable by a server.
  - (e) Never sync the stored data off-device.

  [Likely; EDPB 2/2023 and the ICO's "remembering goods in a basket" example]
- **PRIV-13. No cookies, no service worker in v1.** No first-party cookies of any kind. Detect the theme from `prefers-color-scheme`. A stored theme toggle is acceptable as a user-requested preference. Skip a service worker or Cache Storage "offline mode" in v1, to keep the storage analysis simple. [Likely]
- **PRIV-14. No cookie banner.** Do not show a cookie or consent banner as long as PRIV-01 to PRIV-13 hold. A banner on a site with nothing to consent to adds friction and implies tracking that isn't there. Disclose the storage in the privacy policy instead. [Likely]

### C. Share link (kit in the `#fragment`)

- **PRIV-15. Fragment-only.** The kit payload must live only in the URL fragment. Browsers never send fragments to servers in requests or `Referer` headers. No code path may copy the fragment into a query string, a server call, an error report or the page `<title>`. Link-preview unfurlers (Slack, iMessage and similar) therefore see only the generic page. [Certain] on fragment handling (RFC 3986 §3.5; Fetch spec); [Likely] on the rest.
- **PRIV-16. Share-link disclosure.** Next to the share button: "Anyone with this link can see this kit. The kit is stored in the link itself; we never receive or store it." Also tell users not to put private information in kit text fields. [Likely]
- **PRIV-17. Abuse hardening of the share view.** The share view renders user-authored content on the owner's domain, so it can be abused for phishing or defamation pages. Requirements:
  - (a) Render name and tagline as plain text, never HTML or clickable links. Cap lengths (for example 80 and 160 characters).
  - (b) Allow SVG logos only after sanitization: strip `<script>`, event handlers, `<foreignObject>`, external `href`/`xlink:href`, `@import` and `url()` pointing off-document. Render through `<img src="data:...">`, not inline DOM.
  - (c) Raster logos never ride in the link (already locked).
  - (d) Serve the share view with `noindex`.
  - (e) Show a fixed banner: "Made by a visitor with [Tool]. Not reviewed or endorsed by anyone."
  - (f) Include a footer "Report a problem" mailto.

  These also reduce UK Online Safety Act exposure (Section 3.13). [Guessing] on OSA; [Likely] on the security need.

### D. Children

- **PRIV-18. No age gate.** COPPA does not require general-audience sites to screen ages. An age gate would itself collect age data and, if non-neutral, encourage lying. The FTC says a drop-down that only allows 13+ years, or an "I am over 12" checkbox, is not a neutral screen. [Certain] on the FTC FAQ; [Likely] on the recommendation.
- **PRIV-19. Stay general-audience by design.** Don't add features or marketing that make the tool child-directed under the 16 CFR 312.2 factors: no animated characters or mascots, games, child-oriented incentives, "make a logo for your class project" campaigns, or ads targeting kids. The owner's design directives already push this way. [Likely]
- **PRIV-20. Children's section in the privacy policy.** For example: "[Tool] is built for business branding and isn't directed to children under 13. It doesn't ask for or collect personal information from anyone, including children. Parents with questions can email [contact]." [Likely]
- **PRIV-21. Inbound email from a child.** If someone emails the privacy contact and reveals they are under 13, the operator now has actual knowledge. Answer at most once, then delete the message and address. This mirrors COPPA's one-time-response exception, 16 CFR 312.5(c)(3). Never add any inbound address to a list. This also enforces the owner's "never capture emails from kids" requirement for the one channel that exists. [Likely]

### E. Marketing-consent guardrails (TCPA, Texas SB 140, CAN-SPAM, CASL)

- **PRIV-22. Never collect a phone number or email.** No phone, SMS or email input fields anywhere: no "email me my kit", no newsletter, no "get notified", no lead form, no chat widget, no "text me the link".
- **PRIV-23. No operator-initiated messages.** The operator never sends calls, texts or emails to users. Sharing happens only through mechanisms where the **user** is the sender: copy link, the Web Share API, or a `mailto:` that opens the user's own mail client with the link prefilled. Do not build server-sent email or SMS. Do not add an `sms:` share button; it adds nothing over Web Share.
- **PRIV-24. Contact is `mailto:` only.** The only contact route is a `mailto:` link to the privacy/contact address. Replies to inbound questions are not marketing. Do not follow up with promotional content.

  Together these make TCPA (47 U.S.C. 227), Texas Bus. & Com. Code ch. 302/304 as amended by SB 140, CAN-SPAM (15 U.S.C. 7701 et seq.) and CASL inapplicable by construction. [Certain] that these laws regulate calls, texts and commercial email the operator sends; [Likely] on the guardrail.

### F. Notices and documents

- **PRIV-25. Required public pages.** Publish a privacy notice (`/privacy`), terms (`/terms`) and an accessibility statement (`/accessibility`), each linked in the footer of every page, including the share view. Section 4 lists their exact contents.
- **PRIV-26. In-context microcopy.** One line each near the URL input, the uploader, the draft, the share button and the bot check (Section 4.4).
- **PRIV-27. Compliance notes kept in the repo** under `/docs/compliance/`. These are internal and need not be linked.
  - (a) Data-flow inventory and the third-party-request allow-list (PRIV-01).
  - (b) GDPR/UK GDPR Art. 3(2) non-targeting note (Section 3.2).
  - (c) UK Children's Code "likely to be accessed" assessment. The ICO says to "document and support your reasons".
  - (d) UK OSA scope note.
  - (e) Log settings and retention record.

  These cost nothing and are the evidence if anyone asks. [Likely]
- **PRIV-28. Only write privacy claims you can test.** Ship an automated check, for example a Playwright test that records all network requests on the home page, during a URL scan, during an upload, on PDF export and in the share view, and fails on any non-allow-listed host. This turns "uploads never leave your device" from marketing into a tested invariant. [Likely]

### G. Accessibility

- **PRIV-29. Target WCAG 2.2 Level AA** for the tool UI and the share view. Specific criteria that matter for this UI:
  - 1.4.1 Use of Color: swatches always show a hex label.
  - 1.4.3 and 1.4.11 Contrast: the tool's own UI, including the single accent color.
  - 2.5.7 Dragging Movements: every drag-and-drop upload needs a button file picker, and every color picker needs typed hex entry.
  - 2.5.8 Target Size (Minimum): 24×24 CSS px.
  - 2.4.11 Focus Not Obscured.
  - 2.4.7 Focus Visible.
  - 3.3.1 and 3.3.3 Error identification and suggestions (bad URL, fetch blocked).
  - 4.1.3 Status Messages: announce "Scanning…" and "Done" via `aria-live`.
  - 1.1.1 Non-text content: alt text for logo candidates.
  - Turnstile must be operable without a puzzle; managed mode is normally invisible.

  [Certain] that these are WCAG 2.2 criteria; [Likely] on relevance.
- **PRIV-30. Generated PDF, best effort.** Real text rather than rasterized pages, a document title, a language tag, logical reading order, and color hex values as text. Full PDF/UA tagging is out of reach for common client-side PDF libraries, so say so in the accessibility statement and offer the HTML share view as the accessible equivalent. [Likely]

---

## 2. Applicability matrix

| Law / regime | Applies? | Why (short) | Product consequence | Conf. |
|---|---|---|---|---|
| **COPPA** (US) | **No**, conditionally | General-audience, not child-directed, no actual knowledge. IPs are COPPA "personal information" but the Rule reaches only child-directed operators or those with actual knowledge. | PRIV-18 to PRIV-21. Optional one-sentence internal-operations disclosure. | Likely |
| **2025 COPPA amendments** | No (same reason) | Published 22 Apr 2025, effective 23 Jun 2025, compliance by 22 Apr 2026 | None beyond the above | Certain (dates) |
| **GDPR** (EU) | **Probably not**: Art. 3(2) not triggered if no targeting and no monitoring. Designed to GDPR standard anyway. | No EU establishment; free service but no intent to target EU; no tracking | Art. 13-style notice; legitimate interests; no Art. 27 rep needed if Art. 3(2) is not triggered | Likely |
| **UK GDPR + DPA 2018** | Same as GDPR | Same test | Same; no ICO fee if not in scope | Likely |
| **ePrivacy Art. 5(3) / UK PECR reg. 6** | **Yes** (they apply to any site served to EU/UK users) | Draft in localStorage = "storage"; Turnstile reads device signals; any JS analytics = "access" | Strictly-necessary-only design; no banner; no JS analytics | Likely |
| **UK Children's Code** | **No** if "not likely to be accessed by children" and UK GDPR not triggered | Business-branding tool; ICO test is "more probable than not" | Document the assessment; defaults already satisfy the Code | Likely |
| **CCPA/CPRA** | **No** | Needs >$26,625,000 revenue, or buying/selling/sharing PI of 100,000+ consumers, or 50%+ revenue from selling/sharing PI. None is met. | None; still disclose "we don't sell/share" | Certain |
| **CalOPPA** | **Probably not** | Needs a "commercial" site collecting PII (name, email, phone, etc.) from "consumers" acquiring goods/services "by purchase or lease". IP addresses aren't in the PII list. | Publish a policy anyway; it is the industry norm and needed for GDPR-style transparency | Likely |
| **Delaware DOPPA** (6 Del. C. §1205C) | **Probably not** (same structure as CalOPPA) | Same reasoning | Include DNT and third-party-tracking statements to match the safe-harbor text | Likely |
| **Texas TDPSA** | **No** | Exempt as an SBA-defined small business. The only carve-out (§541.107): no selling sensitive data without consent. The tool doesn't sell anything. | Never sell data | Certain |
| **Texas SCOPE Act (HB 18)** | **No** | Requires all of: social interaction between users, profiles, and user posts viewable by other users | Don't add profiles, galleries or comments | Certain |
| **Other state comprehensive laws** (24 states as of mid-2026) | **No** | Lowest consumer thresholds are 25,000 (Montana; Alabama from 1 May 2027) or 35,000 (Delaware, New Hampshire, Rhode Island, Maryland; Vermont from 1 Jan 2028). Texas and Nebraska exempt small businesses. | Revisit only if traffic is huge (Section 5) | Likely |
| **Brazil LGPD** | Probably not / minimal | Personal-purpose exemption doesn't fit a public tool, but there is no Brazil targeting and only IP logs | Same notice covers it | Guessing |
| **Brazil ECA Digital** (Law 15.211/2025) | **No** if not "likely to be accessed" by minors | In force 17 Mar 2026; applies to services directed at or likely accessed by minors in Brazil | Keep the tool unattractive to minors (PRIV-19) | Likely |
| **Canada PIPEDA** | **No** | Needs "commercial activity"; a free, non-monetized portfolio tool is not primarily commercial | None | Likely |
| **Australia Privacy Act** | **No** | Small-business exemption (annual turnover ≤ AUD 3m) still in force as of 2026; no Australian link | None | Likely |
| **India DPDP Act** | Low/uncertain; main duties from ~13 May 2027 | Applies to foreign processing tied to offering services in India; IP logs are the only personal data | Notice covers it; keep logs minimal | Guessing |
| **TCPA / Texas SB 140** | **No** | No calls or texts | PRIV-22 to PRIV-24 | Certain |
| **CAN-SPAM** | **No** | No commercial email sent | PRIV-22 to PRIV-24 | Certain |
| **CASL** | **No** | No commercial electronic messages sent | PRIV-22 to PRIV-24 | Certain |
| **Third-party CDN fonts / hotlinked assets** | Risk exists (EU case law) | IP disclosed to Google or other hosts without legal basis = GDPR violation per LG München I | PRIV-02 to PRIV-05 | Certain (case); Likely (fix) |
| **Cloudflare Turnstile** | Usable; must be disclosed | Cloudflare is processor for bot-checking and **independent controller** for improving Turnstile | Disclose; load only on fetch | Certain |
| **Cloudflare DPA** | Yes, automatic | Incorporated into the Self-Serve Subscription Agreement, which covers free services | Cite it in the policy; nothing to sign | Certain |
| **ADA Title III** | Low exposure | Unsettled whether web-only services are "places of public accommodation"; 5th Circuit leans physical-place | WCAG 2.2 AA + statement | Likely |
| **European Accessibility Act** | **No** | Not a listed service; and microenterprises providing services are exempt (Art. 4(5)) | None; WCAG 2.2 AA anyway | Certain (exemption); Likely (scope) |
| **UK Online Safety Act** | Probably not | Share links might make it a "user-to-user service", but the "links with the UK" test is likely unmet | PRIV-17 hardening; scope note | Guessing |
| **EU DSA** | Probably not | No hosting (nothing stored); no "substantial connection" to the EU | None | Guessing |

---

## 3. Detailed findings

### 3.1 COPPA (US Children's Online Privacy Protection Act and Rule, 16 CFR Part 312)

**Who is covered.** §312.3 makes it unlawful for "any operator of a website or online service directed to children, or any operator that has actual knowledge that it is collecting or maintaining personal information from a child" to collect PI in violation of the Rule. A general-audience operator without actual knowledge is outside the Rule. [Certain] Source: https://www.ecfr.gov/current/title-16/chapter-I/subchapter-C/part-312/section-312.3

**IP addresses are COPPA personal information.** §312.2 "Personal information" item (7): "A persistent identifier that can be used to recognize a user over time and across different websites or online services. Such persistent identifier includes, but is not limited to, a customer number held in a cookie, an Internet Protocol (IP) address, a processor or device serial number, or unique device identifier." So the host's request logs contain COPPA PI. That matters only if the tool were child-directed or had actual knowledge. [Certain] Source: https://www.ecfr.gov/current/title-16/chapter-I/subchapter-C/part-312/section-312.2

**"Directed to children" factors** (§312.2): subject matter, visual content, animated characters, child-oriented activities and incentives, music, age of models, child celebrities, language, advertising directed to children, plus empirical audience evidence and "marketing or promotional materials or plans". A brand-kit generator for small businesses, freelancers and marketers is not child-directed on these factors. [Likely] Same source.

**Support-for-internal-operations exception.** Even if the tool were child-directed, §312.5(c)(7) allows collecting "a persistent identifier and no other personal information" used "for the sole purpose of providing support for the internal operations" without parental consent. That covers maintaining or analyzing functioning, performing network communications, and protecting security or integrity (§312.2(1)(i), (ii), (v)). The identifier must not be used "to contact a specific individual, including through behavioral advertising, to amass a profile ... or for any other purpose". Under the amended Rule, an operator relying on (c)(7) must state in its online notice "the specific internal operations for which the operator has collected a persistent identifier ... and the means the operator uses to ensure that such identifier is not used or disclosed" for other purposes (§312.4(d)(3)). The FTC FAQ confirms: "Persistent identifiers collected for the sole purpose of providing support for the internal operations ... do not require parental notice or consent". [Certain] Sources: §312.5 https://www.ecfr.gov/current/title-16/chapter-I/subchapter-C/part-312/section-312.5 ; §312.4 https://www.ecfr.gov/current/title-16/chapter-I/subchapter-C/part-312/section-312.4 ; FTC FAQ (updated 10 Jul 2026) https://www.ftc.gov/business-guidance/resources/complying-coppa-frequently-asked-questions
- **Spec use:** although not required, add one sentence to the privacy policy. It makes the policy robust if the audience assessment is ever challenged: "Technical data such as IP addresses is used only to deliver the site and protect it from abuse (network communications and security). It is never used to contact anyone, build profiles, or target ads." [Likely]

**2025 amendments and dates.** Published in the Federal Register 22 Apr 2025 (90 FR 16918). Effective 23 Jun 2025. Most entities had until **22 Apr 2026** to comply; some safe-harbor provisions have different dates. Changes include expanded PI (biometric identifiers), separate verifiable parental consent for third-party disclosures, a written data-retention policy (§312.10), and a written security program. None reaches a general-audience no-knowledge operator. [Certain] on dates. Source: https://www.federalregister.gov/documents/2025/04/22/2025-05904/childrens-online-privacy-protection-rule

**Age-verification enforcement policy (Feb 2026).** On 25 Feb 2026 the FTC issued an enforcement policy statement. It will not bring COPPA actions against general- or mixed-audience operators that collect PI solely to determine age, if conditions are met. It encourages age assurance but does not require it. [Likely] on the date (law-firm reports); [Certain] on content. Sources: https://www.ftc.gov/system/files/ftc_gov/pdf/coppa-age-verification-policy-statement.pdf ; https://cooley.com/news/insight/2026/2026-03-02-ftc-issues-coppa-enforcement-discretion-policy-to-incentivize-use-of-age-verification-technologies

**Should the tool have an age gate? No.**
- The FTC FAQ: "The Rule does not require operators of general audience sites to investigate the ages of visitors". [Certain]
- If a site does screen, it must be neutral. A drop-down allowing only 13+ birth years, or an "I am over 12" checkbox, "would not be considered a neutral age-screening mechanism". [Certain]
- A gate would add data processing (age plus IP) to a tool that otherwise processes almost nothing. It would also create actual-knowledge events to handle, and add friction to a portfolio piece. [Likely]
- **"Commercial purposes".** COPPA's "operator" definition and "directed to children" definition refer to commercial websites. A portfolio piece that promotes the owner's freelance business is probably "commercial" for COPPA purposes, even though it is free. So don't rely on a non-commercial argument; rely on general-audience status. [Likely]

### 3.2 GDPR and UK GDPR

**Territorial scope (Art. 3(2)).** The operator has no EU or UK establishment, so GDPR applies only if processing relates to:
- (a) "offering of goods or services, irrespective of whether a payment ... is required" to people in the Union, or
- (b) "monitoring of their behaviour".

Free services count, so the tool being free does not take it out of (a). [Certain] Source: EDPB Guidelines 3/2018 on territorial scope, https://www.edpb.europa.eu/sites/default/files/files/file1/edpb_guidelines_3_2018_territorial_scope_after_public_consultation_en_1.pdf

- **Targeting test.** Recital 23, quoted by the EDPB: "the mere accessibility of the controller's ... website in the Union, of an email address or of other contact details, or the use of a language generally used in the third country where the controller is established, is insufficient". Factors that indicate targeting include "the use of a language or a currency generally used in one or more Member States ... or the mentioning of customers or users who are in the Union". An English-only free tool from Texas, with no EU marketing, currencies or EU-specific content, is **likely not** "offering services" to EU data subjects in the Art. 3(2)(a) sense. [Likely]
- **Monitoring test.** The EDPB: "the use of the word 'monitoring' implies that the controller has a specific purpose in mind for the collection and subsequent reuse of the relevant data", and it "does not consider that any online collection or analysis of personal data of individuals in the EU would automatically count as 'monitoring'". Transient security logs and an anti-bot check are not behavioral monitoring. Analytics that profile visitors could be. [Likely]
- **Conclusion.** GDPR and UK GDPR **likely do not apply**. Because the owner wants "legally sound everywhere", the notice is written to the Art. 13 standard anyway; it costs nothing. The real point is to avoid the triggers (Section 5). [Likely]

**Controller/processor.** If GDPR applied, Justin James would be the **controller**: he decides why and how the site processes visitor IPs. Cloudflare, or whichever host, is a **processor** under its DPA. The Cloudflare Self-Serve Subscription Agreement (last updated 12 Sep 2025) covers services that "require a Cloudflare account", including free ones. It states that for EU/UK personal data "Cloudflare is a data processor or sub-processor ... and Cloudflare will handle such Personal Data in compliance with Cloudflare's Data Processing Addendum ... which is hereby incorporated by reference". The DPA (Version 6.4, effective 3 Apr 2026) relies on the EU SCCs, the UK Addendum, and the EU-US Data Privacy Framework (with the UK and Swiss extensions). **Nothing to sign.** [Certain] Sources: https://www.cloudflare.com/terms/ ; https://www.cloudflare.com/cloudflare-customer-dpa/
- Turnstile is different. Cloudflare "is a data processor of Signals that we process to provide the Turnstile service", but "is a data controller of Signals that we process to improve Turnstile's bot detection capabilities". The privacy notice must say this. (Section 3.9.) [Certain]

**IP addresses are personal data.** CJEU C-582/14 *Breyer* (19 Oct 2016): a dynamic IP address can be personal data for a website operator. Recital 30 names online identifiers. [Certain]

**Lawful basis.**
- Art. 6(1)(f) **legitimate interests** for delivering and securing the site: request logs and bot check. [Likely]
- For the server-side URL fetch, the visitor asked for it, so Art. 6(1)(f) again. Art. 6(1)(b) contract is a weaker fit for a free, termless tool. [Likely]
- No consent-based processing exists, so there is nothing to collect consent for, and **Art. 8 (child consent ages) is not engaged**. [Likely]

**Art. 8 ages, for reference only.** Member states choose 13 to 16. For example: 13 in Belgium, Denmark, Estonia, Finland, Latvia, Portugal and Sweden; 14 in Austria, Bulgaria, Italy, Lithuania and Spain; 15 in France and Czechia; 16 in Germany, Ireland, the Netherlands and others. UK: 13 (DPA 2018 s.9). These matter only if the tool ever relies on consent for an information-society service offered directly to a child. [Likely; secondary sources] Source: https://childreninthedigitalworld.twobirds.com/home/eu

**Does fetching a URL the user typed process personal data? Sometimes, yes.**
- The URL can identify a person, for example a sole proprietor's name-domain.
- The fetched page can contain names, emails and photos.
- Fetching and parsing is "collection, retrieval, consultation" under Art. 4(2), even if transient. [Likely]

Mitigations, which are PRIV-06:
- Extract only brand fields.
- Keep everything in memory.
- Drop query strings.
- Never log or persist the URL or page.

For people named on fetched pages, a separate Art. 14 notice would be "disproportionate effort" (Art. 14(5)(b)) for transient processing. Disclose the behavior in the policy. [Likely]

**Uploads processed only in the browser are not processed by the operator.** EDPB Guidelines 2/2023 (v2.0, adopted 7 Oct 2024) say browsers processing "information provided by the users themselves" inside the device is not "gaining of access ... as long as the information does not leave the device". That only holds while no code ships derived data back: no telemetry, no error reports containing content. [Certain] on the EDPB text; [Likely] on the application. Source: https://edpb.europa.eu/system/files/2024-10/edpb_guidelines_202302_technical_scope_art_53_eprivacydirective_v2_en_0.pdf

**Art. 13 notice contents** (to write into the policy even though GDPR likely doesn't apply):
- Controller identity and contact.
- Purposes and legal basis, with the legitimate interests named.
- Recipients (Cloudflare or the host).
- Where data is processed and the transfer safeguards.
- Retention period.
- Rights: access, rectification, erasure, restriction, objection, portability.
- The right to complain to a supervisory authority.
- Whether providing data is required: no.
- Automated decision-making: none.

Sources: GDPR Art. 13, https://eur-lex.europa.eu/eli/reg/2016/679/oj [Certain]. A street address is **not** required by Art. 13; name plus email is sufficient for an individual operator. [Likely]

**Art. 27 representative.**
- Art. 27 requires a non-EU controller caught by Art. 3(2) to designate an EU representative; UK GDPR Art. 27 mirrors this for the UK.
- Exemption Art. 27(2)(a): processing that "is occasional, does not include, on a large scale, processing of special categories ... and is unlikely to result in a risk".
- The EDPB reads "occasional" narrowly: "only ... if it is not carried out regularly, and occurs outside the regular course of business or activity". A website that logs every visit arguably processes regularly, so **do not rely on the exemption**. Rely on staying outside Art. 3(2). [Certain] on the text; [Guessing] whether the exemption would hold.
- A paid representative breaks the $0 rule. This is **conflict C1**. [Likely]

**UK specifics.**
- The ICO data protection fee rose on 17 Feb 2025: tier 1 £52, tier 2 £78, tier 3 £3,763. It is relevant only if UK GDPR applies. Whether a non-UK controller caught only by Art. 3(2) must pay is not clearly settled in current ICO guidance; see Unverified. [Likely] Source: https://www.lewissilkin.com/insights/2025/02/06/fee-payable-to-the-ico-to-increase-in-february-102jzb9
- The Data (Use and Access) Act 2025 received Royal Assent 19 Jun 2025. Its PECR changes took effect 5 Feb 2026. [Likely] Source: https://bratby.law/data-use-and-access-act-2025-commencement/

### 3.3 ePrivacy Directive Art. 5(3) and UK PECR reg. 6 (storage and access on the device)

**Scope is wider than cookies and wider than personal data.** Art. 5(3) covers "the storing of information, or the gaining of access to information already stored, in the terminal equipment". It covers "information", not only personal data. Storage "does not depend on the type of medium" and has no minimum duration. [Certain] Source: EDPB Guidelines 2/2023 (link above).

**How each storage mechanism in the spec is treated:**

| Mechanism | Art. 5(3)/PECR status | Basis | Needed |
|---|---|---|---|
| localStorage/IndexedDB draft of the user's kit | "Storage" | Strictly necessary for a service "explicitly requested by the subscriber or user" | Disclosure + clear button (PRIV-12). No consent. [Likely] |
| Reading `location.hash` for the share link | Not "access" while data stays on device | EDPB 2/2023: local use that doesn't leave the device | Nothing. [Likely] |
| Turnstile widget script (reads device and TLS signals, sends them to Cloudflare) | "Access" | Security / strictly necessary. Cloudflare: Signals "are strictly necessary for the purpose of detecting and blocking bots" | Disclosure. Load only on fetch (PRIV-08). [Likely] |
| Turnstile pre-clearance `cf_clearance` cookie | Storage | Possibly strictly necessary, but avoidable | Don't enable. [Certain] that it sets the cookie |
| Cloudflare Web Analytics JS beacon (no cookies) | "Access" per EDPB (JS instructs the browser to send data) | **Not** strictly necessary. UK: statistical exception with notice + opt-out. EU: consent, unless a national DPA's audience-measurement exemption fits (e.g., CNIL) | **Do not ship in v1** (PRIV-09). [Likely] |
| Fonts from an external library | IP disclosure + access | UK ICO lists external font libraries under the "appearance" exception, which requires notice + "a simple and free way to object". The ICO suggests self-hosting. | **Self-host** (PRIV-02). [Certain] on ICO text |

UK exception wording, quoted from ICO guidance (page dated 29 Apr 2026):
- **Strictly necessary** "applies when the purpose of the storage or access is essential to provide the service the subscriber or user requests". "Remembering the goods a user wishes to buy" is the canonical example. "If you use it for any other purpose as well, the exception does not apply."
- **Statistical purposes** "you must provide the user ... with clear and comprehensive information about the purpose, and a 'simple and free' means to object".
- On external fonts: "Be aware that external font libraries may collect information about your users, such as their IP address. Where this occurs, you must explain this to your users and give them a simple and free way to object. You could consider self-hosting fonts".

[Certain] Source: https://ico.org.uk/for-organisations/direct-marketing-and-privacy-and-electronic-communications/guidance-on-the-use-of-storage-and-access-technologies/what-are-the-exceptions/

**Is a cookie banner needed? No**, if the only storage and access is (a) the user-requested draft and (b) the security check. Banners exist to collect consent, and none is needed here. Disclosure in the privacy policy is still good practice. Under the strictly-necessary exemption, neither consent nor notice is legally required. [Likely]

**Pending EU change.** The Commission's Digital Omnibus proposal (Nov 2025) would move cookie rules into the GDPR (new Art. 88a) and add consent-free purposes: first-party aggregated audience measurement and security. As of a 23 Oct 2026 commentary it was still draft. The ePrivacy Directive as transposed nationally is the current law. [Likely] Source: https://www.heuking.de/en/news-events/newsletter-articles/detail/cookie-consent-under-the-digital-omnibus-planned-changes-and-technical-implementation.html

### 3.4 UK Age Appropriate Design Code (Children's Code)

**Applicability.**
- The Code covers information society services "likely to be accessed by" under-18s, not only services aimed at children.
- The ICO: "for a service to be 'likely' to be accessed, the possibility of this happening needs to be more probable than not".
- It applies to non-UK services only where the DPA 2018 applies: "if you offer your service to users in the UK, or monitor the behaviour of users in the UK".

So it needs UK GDPR to apply (the Art. 3(2) analysis) **and** likely child access. [Certain] on the text. Source: https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/childrens-information/childrens-code-guidance-and-resources/age-appropriate-design-a-code-of-practice-for-online-services/services-covered-by-this-code/

**Assessment.** A brand-kit generator for businesses has no particular appeal to under-18s. Teens could use it, but children are not a "substantive and identifiable user group". **Likely not in scope.** [Likely]

**Minimal steps that satisfy it anyway, all $0:**
- (1) Write a short "likely to be accessed" assessment in `/docs/compliance/`. The ICO: "you should document and support your reasons for your decision".
- (2) The design already meets the Code's substantive standards: high privacy by default; no profiling; no geolocation; no nudge techniques; data minimization; no sharing; no connected toys; no parental monitoring.
- (3) Write the privacy policy in plain language a 13-year-old can follow, which is the "transparency" standard.
- (4) Revisit if a significant number of children use the tool, for example classroom adoption.

[Likely]

### 3.5 US state law

**CCPA/CPRA (California).** "Business" requires annual gross revenue over **$26,625,000** (CPI-adjusted from $25m effective 1 Jan 2025), or buying, selling or sharing PI of 100,000+ consumers or households, or 50%+ of revenue from selling or sharing PI. None is met: revenue is $0, and there is no selling and no "sharing" (cross-context behavioral advertising). **Not covered.** [Certain] Sources: https://cppa.ca.gov/announcements/2024/20241217.html ; https://www.cppa.ca.gov/regulations/cpi_adjustment.html

**CalOPPA (Bus. & Prof. Code §§22575-22579).**
- It requires an "operator of a commercial Web site ... that collects personally identifiable information ... about individual consumers residing in California" to post a policy.
- PII is defined (§22577(a)) as "individually identifiable information about an individual consumer collected online by the operator from that individual and maintained by the operator in an accessible form". The list covers name, physical address, email, phone, SSN, "any other identifier that permits the physical or online contacting of a specific individual", and associated profile data.
- "Consumer" means an individual who "seeks or acquires, by purchase or lease, any goods, services ... for personal, family, or household purposes" (§22577(d)).
- IP addresses are not listed. A free business tool's users are not acquiring by purchase or lease. And the operator does not "maintain" PII "in an accessible form".
- **Likely not triggered.** Violation arises only if the policy is not posted within 30 days of notice. Post a policy anyway, and include CalOPPA's content items: categories collected, third parties, the change-notification process, effective date, DNT response, and whether third parties track across sites. That is cheap insurance. [Certain] on the text; [Likely] on the conclusion. Source: https://leginfo.legislature.ca.gov/faces/codes_displaySection.xhtml?lawCode=BPC&sectionNum=22577

**Delaware Online Privacy and Protection Act (6 Del. C. §1205C).**
- Same structure as CalOPPA, covering "individual users residing in Delaware".
- The Delaware DOJ's safe-harbor language requires disclosing how the site responds to "do not track" signals and whether third parties may collect PII "over time and across different sites".
- **Likely not triggered**, but put both statements in the policy. [Likely] Sources: https://delcode.delaware.gov/title6/c012C ; https://regulations.delaware.gov/api/register/October2024/383c8a85-43ba-4a3c-8fc7-095987fb4f62

**Texas Data Privacy and Security Act (Bus. & Com. Code ch. 541, effective 1 Jul 2024).**
- §541.002(a) applies only to a person that "(3) is not a small business as defined by the United States Small Business Administration, except to the extent that Section 541.107 applies".
- §541.107 says a small business must get consent before **selling** sensitive data. Sensitive data includes a known child's data and precise geolocation.
- A sole proprietor with $0 revenue is an SBA small business. **Exempt.** The carve-out is irrelevant because nothing is sold. [Certain] Sources: https://statutes.capitol.texas.gov/GetStatute.aspx?Code=BC&Value=541.002 ; https://www.texasattorneygeneral.gov/consumer-protection/file-consumer-complaint/enforcing-privacy-and-data-security-laws-protect-texans/texas-data-privacy-and-security-act

**Texas SCOPE Act (HB 18; Bus. & Com. Code ch. 509, effective 1 Sep 2024).**
- §509.002(a) applies "only to a digital service provider who provides a digital service that: (1) connects users in a manner that allows users to socially interact with other users ...; (2) allows a user to create a public or semi-public profile ...; and (3) allows a user to create or post content that can be viewed by other users".
- The tool has no profiles, no social interaction and no posting surface. A share link is a self-contained URL, not a feed or board. **Not covered.** [Certain]
- Litigation status: parts are enjoined; the Fifth Circuit ruled 24 Jul 2026 and en banc rehearing was sought. This is irrelevant here because the applicability test is not met. [Likely]
- Sources: https://statutes.capitol.texas.gov/Docs/BC/htm/BC.509.HTM ; https://www.texasattorneygeneral.gov/consumer-protection/file-consumer-complaint/enforcing-privacy-and-data-security-laws-protect-texans/scope
- Related: the Texas App Store Accountability Act (SB 2420) targets app stores and apps, not websites. Not relevant. [Likely]

**Other state comprehensive privacy laws (brief).** As of mid-2026, 24 states have comprehensive laws. Alabama, Louisiana, Oklahoma and Vermont were enacted in 2026. [Likely] Source: https://www.bytebacklaw.com/2026/06/u-s-state-privacy-law-landscape-expands-to-24-states-what-the-latest-legislative-wave-means-for-businesses/

Thresholds:
- **Most** need processing of 100,000+ state consumers (e.g., Virginia, Colorado, Oregon, New Jersey, Minnesota, Indiana, Kentucky; Oklahoma from 1 Jan 2027).
- **Lower thresholds:** 35,000 (Delaware, New Hampshire, Rhode Island, Maryland; Vermont from 1 Jan 2028); 25,000 (Montana as amended; Alabama from 1 May 2027).
- **Texas and Nebraska** use the SBA small-business exemption.
- **Revenue gates:** Tennessee and Florida.

[Likely; thresholds from secondary sources] Sources: https://www.hunton.com/privacy-and-cybersecurity-law-blog/vermont-becomes-23rd-state-with-comprehensive-consumer-privacy-law ; https://www.morganlewis.com/pubs/2026/07/new-us-state-consumer-privacy-laws-what-businesses-should-know

A portfolio tool will not process personal data of 25,000+ residents of any single small state in a year, unless IP addresses held only by the CDN are counted and traffic is very large (Section 5). **Not covered.** [Likely]

**Other US notes.**
- No health, biometric or precise-geolocation data is processed. Image-to-palette quantizes pixels only, with no face detection. So Washington's MHMDA, Illinois BIPA and Texas CUBI are not engaged. [Likely]
- The federal Take It Down Act covers platforms hosting user-generated content; nothing is hosted. [Likely]
- **FTC Act §5 and Texas DTPA apply to everyone.** Privacy statements must be truthful. This is the main live exposure (PRIV-28). [Certain] that §5 covers deceptive privacy claims.

### 3.6 Elsewhere (brief)

- **Brazil, LGPD.**
  - Art. 4(I) excludes processing "by a natural person exclusively for private and non-economic purposes". A public tool doesn't fit cleanly, because it promotes a business. [Likely]
  - Art. 3 reaches processing tied to offering services to people in Brazil. There is no Brazil targeting here, and the only personal data is IP logs. Low exposure. The same privacy notice covers LGPD transparency. [Guessing]
  - Law 15.352/2026 (25 Feb 2026) restructured the ANPD; no business-obligation change was reported. [Likely]
  - Sources: https://iapp.org/resources/article/brazilian-data-protection-law-lgpd-english-translation ; https://www.planalto.gov.br/ccivil_03/_ato2023-2026/2026/lei/L15352.htm
- **Brazil, ECA Digital (Law 15.211/2025).**
  - In force 17 Mar 2026. Applies to IT products/services "directed at children and adolescents in Brazil or likely to be accessed by them". Likelihood turns on attractiveness to minors and ease of use.
  - Foreign providers in scope must appoint a Brazilian legal representative.
  - A business-branding tool is **not** likely to be accessed by minors. Keep PRIV-19.
  - [Likely] Sources: https://www.mayerbrown.com/en/insights/publications/2026/04/enforcement-of-brazils-eca-digital-introduces-new-obligations-for-companies ; https://www.ashurstperkinscoie.com/en/insights/more-online-safety-legislation-preparing-for-brazils-digital-eca/
- **Canada, PIPEDA.**
  - Applies to personal information collected "in the course of commercial activities". The OPC test is "the primary characterization of the activity". Free services *can* be commercial.
  - A non-monetized, no-lead-gen portfolio tool is not primarily commercial. **Not covered.** [Likely]
  - Source: https://priv.gc.ca/en/privacy-topics/privacy-laws-in-canada/the-personal-information-protection-and-electronic-documents-act-pipeda/pipeda-compliance-help/pipeda-interpretation-bulletins/interpretations_03_ca
- **Australia, Privacy Act 1988.**
  - The small-business exemption (annual turnover ≤ AUD 3 million) is still in force; removal is planned in a "tranche 2" bill. There is also no "Australian link", since the operator doesn't carry on business in Australia. **Not covered.** [Likely]
  - Note: a statutory tort for serious invasions of privacy has applied since 10 Jun 2025 to anyone. It is irrelevant to this design. [Likely]
  - Source: https://sprintlaw.com.au/articles/privacy-law-reforms-small-business/
- **India, DPDP Act 2023 and DPDP Rules 2025.**
  - Rules notified 13 Nov 2025. Core fiduciary obligations (notice, consent, security) apply about 18 months later, around **13 May 2027**.
  - The Act reaches processing outside India "in connection with any activity related to offering of goods or services" to people in India.
  - There is no "legitimate interests" basis, only consent or listed "legitimate uses" (s.7). Whether IP logs for site delivery fit s.7(a) "voluntarily provided" is untested.
  - Exposure is low: no India targeting, no stored data. The same notice covers it. Revisit before May 2027. [Guessing]
  - Source: https://ca.hoganlovells.com/en/publications/indias-digital-personal-data-protection-act-2023-brought-into-force-
- **Others, one line each.**
  - Switzerland nFADP: a representative is needed only for large-scale, regular, high-risk processing. Not met.
  - Japan APPI: IP addresses alone are generally not "personal information".
  - China PIPL: applies where the purpose is providing services to individuals in China. No targeting.

  All [Guessing].

### 3.7 TCPA, CAN-SPAM, CASL (marketing consent)

**Non-applicability.**
- **TCPA (47 U.S.C. §227)** restricts calls and texts (texts are "calls") using autodialers or prerecorded voice, and telemarketing to DNC-listed numbers.
- **Texas SB 140** (effective 1 Sep 2025) extended the Texas telephone-solicitation law (Bus. & Com. Code ch. 302 registration; ch. 304/305 with a DTPA private right of action) to text and graphic messages. [Likely] Source: https://www.morganlewis.com/pubs/2025/09/texas-telephone-solicitation-law-now-covers-text-messages
- **CAN-SPAM (15 U.S.C. §7701 et seq.)** regulates "commercial electronic mail messages" whose primary purpose is commercial advertisement or promotion.
- **CASL** (Canada) regulates commercial electronic messages sent to Canadians. Messages "sent in response to a request, inquiry or complaint" are exempt from s.6 (Electronic Commerce Protection Regulations, SOR/2013-221, s.3(b)). [Likely]

The tool collects no phone or email and sends no messages, so none of these can be violated by the tool. [Certain]

**Guardrail text for the spec** (PRIV-22 to PRIV-24): "The product never collects phone numbers or email addresses and never sends calls, texts or emails. No phone, SMS or email input fields; no server-side email or SMS sending; no third-party chat or lead widgets. All sharing is user-initiated (copy link, Web Share API, `mailto:`). The only contact channel is a `mailto:` link. Any future feature that collects a phone number or email address requires a new legal review before build." [Likely]

**Context only: current TCPA consent landscape.**
- **One-to-one consent rule vacated.** *Insurance Marketing Coalition v. FCC*, No. 24-10277 (11th Cir. 24 Jan 2025) vacated the FCC's "one-to-one" consent rule and the "logically and topically related" requirement, days before their 27 Jan 2025 effective date. [Certain] Source: https://www.mofo.com/resources/insights/250130-eleventh-circuit-vacates-fcc-s-tcpa-one-to-one-consent-rule
- **FCC orders no longer bind district courts.** *McLaughlin Chiropractic Associates v. McKesson Corp.* (U.S. 20 Jun 2025, 6-3) held district courts are not bound by FCC interpretations of the TCPA in enforcement suits. [Certain] Source: https://www.faegredrinker.com/en/insights/publications/2025/6/supreme-court-decides-mclaughlin-chiropractic-associates-v-mckesson-corp
- **Revocation rules.** The 2024 revocation-of-consent rules took effect 11 Apr 2025: honor revocation by any reasonable means within 10 business days. The "revoke-all" provision (§64.1200(a)(10)) was delayed again, to **31 Jan 2027** (FCC CGB order DA 26-12, 6 Jan 2026). [Likely] Source: https://www.consumerfinancialserviceslawmonitor.com/2026/01/fcc-further-extends-effective-date-for-tcpa-revoke-all-rule/
- **Prior express written consent** is still required for marketing robocalls and robotexts to wireless numbers. Irrelevant here. [Certain]

**Owner-level note, out of scope for the tool.** The footer links to the owner's portfolio on Squarespace. If that site has contact forms collecting phone numbers and the owner texts leads, TCPA and Texas SB 140 apply **there**. The tool's compliance doesn't cover the portfolio site. [Likely]

### 3.8 Third-party requests from the user's browser (fonts, logos, CDNs)

- **LG München I, 20 Jan 2022, Az. 3 O 17493/20.**
  - The court held that dynamically embedding Google Fonts disclosed the visitor's dynamic IP to Google without a legal basis, violating GDPR and the right to informational self-determination.
  - It awarded **€100** non-material damages (Art. 82 GDPR) and an injunction.
  - A wave of mass warning letters followed in 2022. A later LG München I decision (30 Mar 2023, 4 O 13063/22) reportedly treated such letters as abusive.
  - [Certain] on the 2022 holding (multiple consistent sources); [Likely] on the 2023 follow-up. Sources: https://www.ihk.de/bergische/recht-und-steuern/wettbewerbsrecht/google-fonts-5646176 ; https://www.ra-plutte.de/lg-muenchen-dynamische-einbindung-google-web-fonts-ist-dsgvo/
- **Mitigations, ranked.**
  - (1) **Self-host** font files at build time. Google: "Google does not receive any kind of data related to the visits". [Certain] https://developers.google.com/fonts/faq
  - (2) **Same-origin proxy**: a server function fetches the font from Google using the server's IP and caches it. The visitor's IP never reaches Google. [Likely]
  - (3) **Bunny Fonts.** The visitor's IP still reaches Bunny's CDN, which claims "zero-logging" and EU-based systems. Acceptable only with disclosure, and it still breaks a strict "no third parties" promise. [Likely] https://bunny.net/blog/bringing-privacy-back-into-your-own-hands-introducing-bunny-fonts/
  - **Use (1) for UI fonts and (1) or (2) for the picker.**
- **Logos and favicons hotlinked from scanned sites** leak the visitor IP, and the tool page as `Referer`, to arbitrary third parties. Fix with server-side fetch and same-origin delivery (PRIV-03). [Likely]
- **Effect on outputs.**
  - **PDF:** fonts and logo bytes must be embedded at generation time from same-origin sources. A finished PDF makes no network calls when opened. [Likely]
  - **PNG brand board:** pure raster; no network on open. [Certain]
  - **Share view:** renders entirely from the fragment plus same-origin fonts. It must not fetch anything from the scanned site or font CDNs. That is why raster logos are excluded and SVGs are sanitized (PRIV-17). [Likely]
  - **SVG specifically:** SVG loaded via `<img>` does not fetch external sub-resources, but inline SVG in the DOM does. Use `<img>` plus sanitization. [Likely]

### 3.9 Bot gate and analytics (Cloudflare)

- **Turnstile data.** The Turnstile Privacy Addendum (last updated 18 Jun 2025) says Turnstile processes "client IP address, TLS Fingerprint, User-Agent Header and Sitekey and associated origin". These Signals are "strictly necessary for the purpose of detecting and blocking bots". Cloudflare is processor for the service and **controller** for improving bot detection. [Certain] Source: https://www.cloudflare.com/turnstile-privacy-policy/
- **Turnstile cookies.** Cloudflare staff and the 2022 launch post say default Turnstile does not use cookies. Pre-clearance mode issues `cf_clearance`. Mobile/WebView docs mention reliance on cookies and local storage. Treat default web Turnstile as cookieless but **verify in DevTools before writing the policy**. [Likely] Sources: https://blog.cloudflare.com/turnstile-private-captcha-alternative/ ; https://developers.cloudflare.com/turnstile/get-started/pre-clearance/
- **Disclosure needed.** Name Cloudflare Turnstile, list the signals, state that Cloudflare also uses them as its own controller to improve bot detection, and link the addendum. No consent is needed in the EU or UK under the strictly-necessary/security reasoning. [Likely]
- **Cloudflare Web Analytics.** Cloudflare: "does not use any client-side state, such as cookies or localStorage, to collect usage metrics. We also don't 'fingerprint' individuals via their IP address, User Agent string, or any other data". [Certain, vendor claim] https://www.cloudflare.com/web-analytics/
  - Even so, the JS beacon is "access" under EDPB 2/2023. That means EU consent risk and, in the UK, the statistical exception with notice and an opt-out. **Recommendation: no beacon in v1** (PRIV-09).
  - If added later: disclose it, rely on the UK statistical exception with an on-page opt-out toggle, and accept a small EU residual risk, or add consent for EU visitors. [Likely]
- **Cloudflare DPA for free plans.** Automatic, via the Self-Serve Subscription Agreement (Section 3.2). [Certain]
- **Other Cloudflare cookies.** If any Cloudflare security feature is enabled on the zone (e.g., Bot Fight Mode or challenge pages), Cloudflare may set `__cf_bm` or `cf_clearance`. Cloudflare classifies its cookies as strictly necessary. List any that actually appear. [Likely] Source: https://developers.cloudflare.com/fundamentals/reference/policies-compliances/cloudflare-cookies/
- **Host logs.**
  - Workers Logs free plan: 200,000 events/day, 3-day retention. From 1 Dec 2026 under Observability pricing: 0.5 GB/day, 7-day retention. [Likely] Sources: https://developers.cloudflare.com/workers/observability/logs/workers-logs/ ; https://developers.cloudflare.com/observability/pricing/
  - If hosted on GitHub Pages, GitHub logs visitor IPs "for security purposes". [Likely] https://docs.github.com/en/pages/getting-started-with-github-pages/what-is-github-pages

### 3.10 Accessibility law

- **ADA Title III (US).**
  - Applies to "places of public accommodation". Circuits split on web-only services. The 11th Circuit (*Gil v. Winn-Dixie*, 2021, later vacated as moot) and the 3rd, 6th and 9th Circuits require a nexus to a physical place. The Fifth Circuit, covering Texas, has not squarely ruled on websites; *Magee v. Coca-Cola Refreshments* (5th Cir. 2016) read "sales establishment" as a physical place. [Likely]
  - A free web tool by an individual with no physical location has **low** Title III exposure in Texas. Serial-plaintiff suits target revenue-generating e-commerce. [Likely]
  - DOJ's Title II rule (state and local governments, WCAG 2.1 AA) was extended by an interim final rule published 20 Apr 2026: 26 Apr 2027 for entities of 50,000+, 26 Apr 2028 for smaller ones. It does not apply to this tool but sets the national benchmark. [Likely] Source: https://www.consumerfinancialserviceslawmonitor.com/2026/04/doj-extends-title-ii-ada-web-accessibility-rule-compliance-deadlines-for-state-and-local-governments/
- **European Accessibility Act (Directive 2019/882; applies from 28 Jun 2025).**
  - Art. 4(5): "Microenterprises providing services shall be exempt from complying with the accessibility requirements". A microenterprise has <10 persons and turnover or balance sheet ≤ €2m. [Certain] Source: https://www.legislation.gov.uk/eudr/2019/882/article/4
  - The tool is also not a listed service: not an "e-commerce service" (no consumer contract), and not banking, e-books, transport or telecoms. **Not covered** twice over. [Likely]
- **UK Equality Act 2010, Canada AODA/ACA.** Service-provider duties are tied to providing services in Great Britain or Ontario, or to federal regulation. Low or no exposure. [Guessing]
- **Recommended target: WCAG 2.2 Level AA.**
  - W3C Recommendation 5 Oct 2023. [Certain] https://www.w3.org/TR/WCAG22/
  - Adopted as ISO/IEC 40500:2025. [Likely]
  - It is a superset of the WCAG 2.1 AA used by the DOJ Title II rule and EN 301 549. [Certain]
  - It is also a credibility signal for a portfolio piece. Specific criteria are in PRIV-29 and PRIV-30.

### 3.11 Hosting on a Squarespace-site subdomain (privacy side effects)

- The privacy policy, terms and contact must be the **tool's own**. Do not rely on the wedding-photography site's Squarespace policy. [Likely]
- **Parent-domain cookies.** Squarespace documents analytics cookies `ss_cvr` and `ss_cid` (2 years, "Identifies unique visitors") and `ss_cvt`/`ss_cvisit` (30 minutes). If these are set on the registrable domain rather than host-only, browsers send them to the tool. Mitigation: PRIV-10, plus verification (Unverified). [Certain] that the cookies exist: https://support.squarespace.com/hc/en-us/articles/360001264507-The-cookies-Squarespace-uses ; [Guessing] on their Domain attribute.
- **Don't publish a home street address.** No law reviewed requires an individual operator of this tool to post a street address. GDPR Art. 13 needs identity and contact, and email suffices. The German Impressum duty (DDG §5) applies to providers established in Germany. Use "Justin James, Rowlett, Texas, USA" plus an email. COPPA notices would require address and phone, but COPPA doesn't apply. [Likely]

### 3.12 Server-side URL fetch: who else sees what

- The scanned site sees the host's egress IP and the tool's user agent, never the visitor's IP. Cloudflare Workers subrequests carry a `CF-Worker` header naming the zone, which reveals the tool's domain to scanned sites. That is a transparency fact, not a user-privacy problem. [Likely]
- The tool should honor obvious opt-outs such as HTTP 401/403 and robots `noai`/`noindex`, at the owner's discretion. That is not a privacy-law requirement but supports "legally sound". [Guessing]

### 3.13 UK Online Safety Act and EU DSA (share links)

- **UK OSA s.3(1)** defines a "user-to-user service" as "an internet service by means of which content that is generated directly on the service by a user of the service, or uploaded to or shared on the service by a user of the service, may be encountered by another user". s.3(2)(a): "it does not matter if content is actually shared ... as long as a service has a functionality that allows such sharing".
- A kit built on the tool and opened by someone else through a share link **arguably** fits, even though nothing is stored. [Certain] on the text; [Guessing] on the fit. Source: https://www.legislation.gov.uk/ukpga/2023/50/section/3
- **Links with the UK (s.4(5)-(6))** need either "a significant number of United Kingdom users", the UK as a target market, or "a material risk of significant harm to individuals in the UK" from user content. A niche branding tool whose shareable content is limited to colors, fonts, a short name and tagline, and a sanitized SVG likely meets none of these. [Guessing] Source: https://www.legislation.gov.uk/ukpga/2023/50/section/4
- Ofcom does enforce against small US services. 4chan was fined £20,000 in Oct 2025 and £520,000 in total by 21 Apr 2026. [Likely] Source: https://www.rpclegal.com/snapshots/consumer/summer-2026/ofcom-fines-4chan-for-breaches-of-online-safety-act/
- **Mitigation ($0):** PRIV-17, plus a one-page scope note and a written illegal-content risk note kept in `/docs/compliance/`. These are the documents Ofcom expects first from any in-scope service. [Guessing]
- **EU DSA.** It applies to intermediary services (mere conduit, caching, hosting) with a "substantial connection" to the EU: establishment, significant user numbers, or targeting. Nothing is hosted, and there is no EU connection. **Not covered.** Note that DSA Art. 13 would require a paid EU legal representative if it ever applied, another $0 conflict to avoid. [Guessing]

---

## 4. Required site artifacts

### 4.1 Privacy notice (`/privacy`). Required sections, in this order

1. **Who runs this.** "[Tool] is made and run by Justin James, an individual in Rowlett, Texas, USA. Contact: [free email address]." (GDPR Art. 13(1)(a); CalOPPA/DOPPA practice.)
2. **Short version**, up to 5 bullets, all literally true: no accounts; no forms; no tracking cookies or ads; uploaded files never leave your browser; we don't store what you make.
3. **What happens to each input:**
   - Uploaded logo, image, PDF: processed in your browser, never sent to us.
   - Typed colors, fonts, name, tagline: stay in your browser.
   - Website URL: our server fetches that public page once, extracts colors, fonts, logo, favicon, name and tagline in memory, sends them back, and discards everything. The URL isn't logged by our code.
   - Saved draft: kept in your browser's local storage on this device until you clear it.
   - Share links: the kit is packed into the part of the link after `#`, which browsers don't send to servers. Anyone with the link can see it. We can't see or delete it.
4. **Technical data our providers process:**
   - IP address, browser user agent, time, requested path and TLS details.
   - Processed by [host, e.g., Cloudflare] to deliver the site and protect it, and by Cloudflare Turnstile when you fetch a URL.
   - Cloudflare also uses Turnstile signals as its own controller to improve bot detection; link the Turnstile addendum.
   - Retention: [actual setting, e.g., "provider logs are kept up to 7 days"].
   - Include the COPPA-style internal-operations sentence (Section 3.1).
5. **Third parties your browser contacts:** only [tool origin] and `challenges.cloudflare.com` (bot check, only when you fetch a URL). Fonts are served from our own domain. Logos from sites you scan are fetched by our server, so your browser never contacts those sites.
6. **Cookies and similar technologies:** we set no cookies. Local storage holds only your draft (strictly necessary, at your request). Name any Cloudflare security cookie that actually appears.
7. **Analytics:** "None", or describe edge request counts if used.
8. **Legal bases (EEA/UK):** legitimate interests in delivering and securing the tool and in performing the fetch you request (Art. 6(1)(f)).
9. **Where data is processed:** USA and Cloudflare's global network. Transfer safeguards per Cloudflare's DPA (EU SCCs, UK Addendum, EU-US DPF). Link the DPA.
10. **Your rights:** access, deletion, objection and the others; the right to complain to your data protection authority. Explain that because nothing is stored against you, there's usually nothing to return beyond short-lived provider logs.
11. **Sale and sharing:** "We don't sell or share personal information and don't use targeted advertising." (CCPA/state-law vocabulary.)
12. **Do Not Track and Global Privacy Control:** "We don't track you across sites, so DNT/GPC signals don't change anything here." (CalOPPA §22575(b)(5); DOPPA safe harbor.)
13. **Third-party tracking:** "We don't allow third parties to collect personal information about your activity over time and across sites through this tool." (CalOPPA §22575(b)(6); DOPPA §1205C(b)(6).)
14. **Children:** text from PRIV-20.
15. **Security:** HTTPS only; no database of user data.
16. **Changes:** effective date at top; material changes noted on the page and in the public GitHub repo history (CalOPPA §22575(b)(3)-(4)).
17. **Open source:** link to the repo so anyone can verify the claims.

### 4.2 Terms (`/terms`). Required clauses

- **Free, as-is, no warranty.** Extracted colors, fonts and contrast results can be wrong. Check before production use.
- **Your materials and third-party rights.** You must have the right to use logos and materials you upload or scan. The tool grants no trademark or font-license rights. Fonts detected on a site may be commercially licensed, so check the license. No affiliation with or endorsement by any scanned brand.
- **Acceptable use.** No bypassing the bot check or automated mass scanning. No using share links for impersonation, phishing, harassment or unlawful content. No scanning pages you aren't allowed to access.
- **Share links.** You create and control them. Their content lives in the link. The operator does not host or review them.
- **No contact or marketing.** "We never call, text or email you, and we don't collect contact details."
- **Limitation of liability** to the extent permitted by law, with a **mandatory-consumer-law carve-out**: "Nothing in these terms limits rights you have under laws of your country that can't be waived". This is needed for "legally sound everywhere", because EU, UK and Australian consumer law limit exclusions. [Likely]
- **Governing law:** Texas, USA, subject to the carve-out above.
- **Changes and contact.**
- **No minimum-age clause needed.** Optional neutral line: "Designed for business branding; not directed to children under 13."

### 4.3 Accessibility statement (`/accessibility`)

Include:
- Target: WCAG 2.2 AA.
- Date last tested and the method (automated tools plus keyboard and screen-reader pass).
- Known limitation: generated PDFs aren't fully tagged; the share view is the accessible version.
- Contact for barriers.

### 4.4 In-tool microcopy (one line each; direct, no hedging)

- URL field: "Our server fetches this public page, pulls the brand details, and discards the page."
- Uploader: "Files stay on your device. Nothing is uploaded."
- Draft: "Saved in this browser only." with a **Clear** button.
- Bot check: "Quick bot check by Cloudflare." with a link to the privacy section.
- Share: "Anyone with this link can see this kit. The kit lives in the link; we never receive it."
- Share view banner: "Made by a visitor with [Tool]. Not reviewed or endorsed."

### 4.5 Footer, on every page including the share view

`Privacy · Terms · Accessibility · Source code · Built by Justin James` (portfolio link with `rel="noopener noreferrer"`) and `Contact` (`mailto:`). Optional factual line: "No tracking. Uploads stay on your device." Use it only while PRIV-28 passes.

### 4.6 Internal documents (`/docs/compliance/`, in the repo)

1. Data-flow inventory and allowed-hosts list.
2. GDPR/UK GDPR Art. 3(2) note: no targeting, no monitoring, and why.
3. UK Children's Code "likely to be accessed" assessment.
4. UK OSA scope and illegal-content risk note.
5. Log and retention settings record, with screenshots of host settings.
6. A "change triggers" checklist (Section 5) to run before any new feature.

---

## 5. What would change the analysis

| Change | What it triggers |
|---|---|
| **Any email capture** (newsletter, "email me my kit", waitlist) | CAN-SPAM; CASL; GDPR lawful basis and likely Art. 3(2) issues; CalOPPA/DOPPA clearly triggered (email is listed PII); COPPA exposure (email = online contact info, and actual knowledge from kids becomes likely); the owner's "never capture emails from kids" rule needs an age screen; Texas breach-notification law (Bus. & Com. Code §521.053) |
| **Phone/SMS fields or texting** | TCPA consent rules; Texas SB 140 registration and private right of action; carrier (CTIA) rules |
| **Accounts/login** | Actual-knowledge age issues; data subject requests; security duties; with profiles + social features, Texas SCOPE Act |
| **Server-side storage** (short links, saved kits, public gallery) | Hosting provider status (DSA notice-and-action if EU-connected; UK OSA much more likely; DMCA agent registration, a $6 fee, a $0 conflict); retention policy; COPPA "make PI publicly available" risk; breach exposure |
| **Uploads processed on a server** (e.g., server-side PDF parsing) | Operator becomes controller of upload contents (photos, faces, names in brand guides); the "never sent to a server" promise breaks |
| **Analytics with cookies, pixels or ads** (GA4, Meta Pixel, AdSense) | ePrivacy consent banner in EU/UK; CCPA/state "sharing" and opt-outs if thresholds approach; COPPA (behavioral ads fall outside internal operations); GDPR Art. 3(2)(b) "monitoring", which brings in **Art. 27 EU and UK representatives (paid)** |
| **Cookieless JS analytics beacon** | EU Art. 5(3) consent risk; UK statistical exception (notice + opt-out) |
| **Fonts, scripts or images from third-party CDNs in the browser** | IP disclosure: notice and/or consent in EU/UK; LG München-type claims; breaks the privacy claims |
| **Localizing into EU languages, EU-targeted marketing, prices in EUR/GBP** | Art. 3(2)(a) targeting; GDPR and UK GDPR apply; Art. 27 representatives (paid); possibly ICO fee |
| **Child-appealing design or marketing** (schools, teen founders, mascots, games) | COPPA "directed to children"; UK Children's Code; Brazil ECA Digital (plus a Brazil representative); Australian online-safety codes |
| **Monetization** (ads, affiliate links, paid tier, lead gen) | PIPEDA "commercial activity"; LGPD economic purpose; possible EAA "e-commerce service" (microenterprise exemption still likely); larger ADA Title III target; possibly a Texas assumed-name (DBA) filing; owner's "no upsell" rule |
| **Forming an LLC** | Mostly neutral (SBA small-business status keeps the TDPSA exemption); update the operator identity in notices |
| **Adding AI/LLM features** | Data sent to model providers (new processors and transfers); EU AI Act Art. 50 transparency; breaks the "deterministic" promise |
| **Large traffic** (e.g., 25,000-35,000+ unique residents of one small state per year) | Re-check Montana, Delaware, New Hampshire, Rhode Island, Maryland, Alabama (from May 2027) and Vermont (from 2028) thresholds. Open question: whether IPs held only in short-lived CDN logs count as "processing personal data of" those consumers. [Guessing] |
| **Keeping inbound emails in a CRM or list** | Becomes collected PI: CalOPPA/DOPPA, GDPR, CASL/CAN-SPAM for any follow-up, COPPA if from a child |
| **Swapping Turnstile for reCAPTCHA or hCaptcha** | Google as independent controller with cookies; EU consent questions; new disclosures |
| **Social features** (comments, public gallery, likes) | UK OSA clearly in scope; Texas SCOPE Act; Take It Down Act; DSA hosting duties |

---

## 6. Unverified / needs follow-up

1. **Squarespace cookie scope.** Whether `ss_cvr`, `ss_cid` or other Squarespace cookies are set with `Domain=wethejamesphoto.com`, which would send them to the tool subdomain. Test: DevTools > Application > Cookies on www.wethejamesphoto.com and check the Domain column. The sandbox couldn't reach the site. [Guessing]
2. **Cloudflare logging defaults and retention for the chosen setup** (Pages vs Workers; whether Workers Logs/observability is on by default for new projects; whether invocation logs include client IP). Also confirm the 1 Dec 2026 Observability pricing switch and the free plan's 7-day retention in the live dashboard. [Likely, from docs search]
3. **Turnstile's current client-side storage.** Confirm in DevTools that default web Turnstile sets no cookies or localStorage on the tool origin or `challenges.cloudflare.com`. Sources conflict between staff statements and mobile docs.
4. **Cloudflare Web Analytics cookies.** A 2020 independent review said the beacon set a cookie; Cloudflare's page says no client-side state. Moot if PRIV-09 is followed.
5. **ICO fee for non-UK controllers** caught only by UK GDPR Art. 3(2). Not confirmed from a current ICO page. Moot if UK GDPR doesn't apply.
6. **Art. 27(2)(a) "occasional".** Whether any DPA has accepted the exemption for a continuously available website. Not found.
7. **UK OSA and fragment-only share links.** No Ofcom guidance found on whether content encoded entirely in a URL fragment and rendered client-side counts as user-generated content "on the service".
8. **Fifth Circuit position on web-only ADA Title III.** No controlling Fifth Circuit website decision found; *Magee* (2016) is the closest. Some secondary sources place the Fifth Circuit in the "nexus" camp without citation.
9. **SCOPE Act litigation outcome** after the 24 Jul 2026 Fifth Circuit decision and the en banc petition. Sources conflict. Irrelevant to applicability.
10. **Exact thresholds** for Louisiana (75,000 consumers or a CCPA-like revenue prong; sources conflict), Alabama (25,000; possible employee-count exemption unconfirmed), Oklahoma and Vermont. Taken from law-firm summaries, not enrolled text.
11. **EU Digital Omnibus (Art. 88a)** progress after Oct 2026. Could create a consent-free EU exemption for first-party aggregated audience measurement, which would make a later analytics beacon easier.
12. **India DPDP.** Whether s.7(a) "voluntarily provided" covers IP processing for site delivery, and whether a foreign free tool is "offering services" to India. Re-check before 13 May 2027.
13. **Australia.** Status of the reported 31 Aug 2026 exposure draft removing the small-business exemption (single source).
14. **Brazil ECA Digital.** ANPD guidance on "likely to be accessed" for general-audience business tools; enforcement timing (sanctions reportedly from Nov 2026 / Jan 2027).
15. **Texas assumed-name (DBA) filing** (Bus. & Com. Code ch. 71) if the tool's standalone name is treated as a business name. A county filing fee would conflict with $0. Mitigation: present the tool as a product "made and run by Justin James", so the operator acts under his legal name. Have a Texas attorney confirm. [Guessing]
16. **Free contact email.** Confirm a $0 option: a free webmail address, or free email forwarding on the domain's registrar/DNS (Squarespace Domains forwarding availability and limits are unverified).
17. **Delaware DOPPA PII definition.** Confirm it mirrors CalOPPA and excludes bare IP addresses. Inferred from structure; the definition section wasn't read.
18. **CNIL audience-measurement exemption conditions.** Not re-read for this report; relevant only if a beacon is added later.
19. **LG München I 30 Mar 2023 (4 O 13063/22)** on abusive Google Fonts letters. Single secondary source.
20. **WCAG 2.2 as ISO/IEC 40500:2025, and EN 301 549 updating to WCAG 2.2.** Not re-verified.
21. **FTC age-verification policy statement date** (25 Feb 2026). From law-firm alerts; the PDF itself shows no date in the extracted text.

---

## 7. Source list (primary first)

- 16 CFR 312.2 / 312.3 / 312.4 / 312.5 (eCFR, current as of 5 Nov 2026): https://www.ecfr.gov/current/title-16/chapter-I/subchapter-C/part-312
- FTC, COPPA Rule amendments, 90 FR 16918 (22 Apr 2025): https://www.federalregister.gov/documents/2025/04/22/2025-05904/childrens-online-privacy-protection-rule
- FTC, Complying with COPPA FAQ (modified 10 Jul 2026): https://www.ftc.gov/business-guidance/resources/complying-coppa-frequently-asked-questions
- FTC, COPPA age-verification enforcement policy statement: https://www.ftc.gov/system/files/ftc_gov/pdf/coppa-age-verification-policy-statement.pdf
- EDPB Guidelines 3/2018 on territorial scope: https://www.edpb.europa.eu/sites/default/files/files/file1/edpb_guidelines_3_2018_territorial_scope_after_public_consultation_en_1.pdf
- EDPB Guidelines 2/2023 on Art. 5(3) ePD, v2.0 (7 Oct 2024): https://edpb.europa.eu/system/files/2024-10/edpb_guidelines_202302_technical_scope_art_53_eprivacydirective_v2_en_0.pdf
- GDPR text: https://eur-lex.europa.eu/eli/reg/2016/679/oj
- ICO, storage and access technologies, exceptions (29 Apr 2026): https://ico.org.uk/for-organisations/direct-marketing-and-privacy-and-electronic-communications/guidance-on-the-use-of-storage-and-access-technologies/what-are-the-exceptions/
- ICO, Children's Code, services covered: https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/childrens-information/childrens-code-guidance-and-resources/age-appropriate-design-a-code-of-practice-for-online-services/services-covered-by-this-code/
- UK Online Safety Act 2023 ss.3-4: https://www.legislation.gov.uk/ukpga/2023/50/section/3 ; https://www.legislation.gov.uk/ukpga/2023/50/section/4
- CPPA CCPA threshold adjustment: https://cppa.ca.gov/announcements/2024/20241217.html
- CalOPPA §§22575, 22577: https://leginfo.legislature.ca.gov/faces/codes_displaySection.xhtml?lawCode=BPC&sectionNum=22575 ; https://leginfo.legislature.ca.gov/faces/codes_displaySection.xhtml?lawCode=BPC&sectionNum=22577
- Delaware 6 Del. C. ch. 12C: https://delcode.delaware.gov/title6/c012C
- Texas TDPSA §541.002: https://statutes.capitol.texas.gov/GetStatute.aspx?Code=BC&Value=541.002
- Texas SCOPE Act ch. 509: https://statutes.capitol.texas.gov/Docs/BC/htm/BC.509.HTM
- Cloudflare Self-Serve Subscription Agreement: https://www.cloudflare.com/terms/
- Cloudflare DPA v6.4: https://www.cloudflare.com/cloudflare-customer-dpa/
- Cloudflare Turnstile Privacy Addendum: https://www.cloudflare.com/turnstile-privacy-policy/
- Cloudflare Web Analytics: https://www.cloudflare.com/web-analytics/
- Cloudflare Workers Logs / Observability pricing: https://developers.cloudflare.com/workers/observability/logs/workers-logs/ ; https://developers.cloudflare.com/observability/pricing/
- Google Fonts FAQ: https://developers.google.com/fonts/faq
- Squarespace cookies: https://support.squarespace.com/hc/en-us/articles/360001264507-The-cookies-Squarespace-uses
- EAA Art. 4: https://www.legislation.gov.uk/eudr/2019/882/article/4
- WCAG 2.2: https://www.w3.org/TR/WCAG22/
- Secondary (law-firm and other) sources are cited inline in each section.
