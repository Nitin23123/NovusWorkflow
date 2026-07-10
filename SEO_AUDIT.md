# Brand, SEO & Frontend Visibility — Audit Action Plan

Gap analysis of the **"Novus Aegis AI Brand, SEO and Frontend Visibility Audit"** (July 2026)
against the actual codebase. Every item below was verified in code — this is the
implementation-ready version of that document.

**Status legend**

| | |
|---|---|
| ✅ | Already implemented, verified in code |
| 🐛 | Implemented but **broken/wrong** — fix required |
| ⚠️ | Partially implemented |
| ❌ | Missing — code change needed |
| 🏢 | Business/legal/content decision needed before code changes |
| 🔧 | Server/DevOps-side (nginx/DNS), not repo code |

---

## 1. What is ALREADY in place (stronger than the audit assumed)

The audit repeatedly says items "could not be verified." Most of them **exist**:

| Audit item | Status | Where |
|---|---|---|
| Canonical tags (self-referencing, per route) | ✅ | `src/index.html:16` + `SeoService.setCanonical()` (`src/app/services/seo.service.ts:85`) runs on every route change |
| Per-route titles & meta descriptions | ✅ | `src/seo-routes.json` (all 20 routes, unique) applied by `SeoService`, baked into static HTML by the prerender build |
| Open Graph tags (og:type/site_name/title/description/image/url/locale) | ✅ (but see 🐛 image) | `src/index.html:23-38` + per-route via `SeoService` |
| X/Twitter card (`summary_large_image`) | ✅ (but see 🐛 image) | `src/index.html:40-50` + per-route |
| robots.txt | ✅ | `src/robots.txt` (see ⚠️ notes below) |
| XML sitemap | ✅ | `src/sitemap.xml` (all routes; `lastmod` is stale — 2026-05-23) |
| Organization schema (JSON-LD) | ✅ | `src/index.html:56-77` |
| SoftwareApplication schema | ✅ | `src/index.html:79-97` |
| FAQPage schema | ✅ (but see claims 🏢) | `src/index.html:99-147` |
| WebSite schema (+SearchAction) | ✅ | `src/index.html:149-162` |
| Page-specific JSON-LD support (e.g. blog Article) | ✅ | `SeoService.setStructuredData()` |
| Form security: server-side validation, rate-limiting, CAPTCHA verify, honeypot | ✅ | Implemented July 2026 — `api/` service (see `api/README.md`) |
| Image `width`/`height` (CLS) + `loading="lazy"` | ✅ | Site-wide convention (see `README.md` performance section) |
| WebP imagery, `@defer` heavy components, reduced-motion respect | ✅ | Site-wide; `app.component.ts` Lenis honors `prefers-reduced-motion` |
| Legacy `/phising` → `/phishing` redirect | ✅ | `app-routing.module.ts:26` |
| noindex support for special pages | ✅ | `SeoData.noindex` in `SeoService` |

---

## 2. CRITICAL BUG — social preview image 🐛

**The single highest-impact quick fix in this whole audit.**

`src/index.html` and `SeoService.defaultImage` point OG/Twitter previews at:

```
https://novusaegis.ai/assets/site/images/novus-3d.png
```

That file is **77×80 pixels (6 KB)** — while the meta tags declare `1200×630`.
Every LinkedIn/WhatsApp/Slack/X share of the site renders a tiny/blank preview.

**Fix:**
1. Design a dedicated **1200×630** social card (per audit spec: shield logo, "Novus
   Aegis AI", tagline, dark background, ≥80px safe margins, no small paragraphs).
   Export as `src/assets/social/novus-aegis-og-home.jpg` (JPEG, <300 KB).
2. Point `og:image` / `twitter:image` in `src/index.html` and
   `SeoService.defaultImage` at it. Add `og:image:secure_url`, `og:image:type`,
   `og:image:alt`, `twitter:image:alt` (audit gives exact markup).
3. Re-test with LinkedIn Post Inspector, Meta Sharing Debugger, X Card Validator.

---

## 3. P0 items — status & actions

### 3.1 Canonical domain 🔧
- **Audit:** `novusaegis.ai` redirects to `home.novusaegis.ai`; make the root
  canonical, 301 the subdomain.
- **Code state:** all canonical/OG/sitemap URLs already use the root domain ✅.
- **Action (DevOps):** nginx 301 `home.novusaegis.ai/* → https://novusaegis.ai/$1`;
  root serves the site directly. No repo change needed.

### 3.2 OG / X-card image 🐛
See §2. Meta tags exist; the image asset is the problem.

### 3.3 Favicon package ⚠️
- **Have:** `src/favicon.ico` only, `theme-color` (#400B68) in index.html.
- **Missing:** `favicon.svg`, `favicon-16x16.png`, `favicon-32x32.png`,
  `apple-touch-icon.png` (180×180), `android-chrome-192/512.png`, `site.webmanifest`.
- **Action:** generate the set from the shield logo, add to `src/` + `angular.json`
  assets + the `<link>` block from the audit into `index.html`.

### 3.4 Canonical tags ✅ — done, no action.

### 3.5 Robots & sitemap ✅⚠️
- Both exist. Three flags to decide:
  1. `Disallow: /request-demo` in robots.txt **hides the main conversion page from
     Google** — almost certainly unwanted. 🏢 confirm, then remove.
  2. GPTBot/CCBot blocked — deliberate anti-LLM-training stance? 🏢 confirm intent.
  3. `lastmod` stale (2026-05-23) — regenerate when routes/content change
     (could script at build time).
- **External:** register sitemap in Google Search Console + Bing Webmaster Tools
  (nobody in-repo can verify this — 🏢 confirm it's done).

### 3.6 Product-category clarity 🏢
- Current homepage/title positions the product as "AI-Powered Honeypot Platform";
  audit recommends the master position **"Autonomous Cyber Defense and Deception
  Platform"** with hero H1 **"Turn Your Environment Into a Trap."**
- **This is a leadership/marketing decision.** Once approved, the code changes are:
  hero copy in `home.component.html`, `<title>`/descriptions in `index.html` +
  `seo-routes.json`, OG/Twitter titles, and the social-card artwork text.

### 3.7–3.12 Claims audit 🏢 (legal/marketing sign-off, then find-replace)

Exact locations of every risky claim (verified by grep):

| Claim | Locations | Audit's safer replacement |
|---|---|---|
| **"Patent-Approved LLM Honeypot System"** | `home.component.html:533,918` · `mssps.component.html:487` · `ai-soc-analyst.component.html:137` · `index.html:64,87` (schema) · `seo-routes.json:38-40` | "Patent pending" or "Protected by U.S. Patent No. ___" — whichever is legally accurate |
| **"SOC 2 Type II Certified"** | `home.component.html:930-933` · `mssps.component.html:502` · `ai-soc-analyst.component.html:152` · `security.component.html:135` · `seo-routes.json:69-70,103-104` | "SOC 2 Type II examined" / "report available under NDA" — SOC 2 is an attestation, not a certification |
| **"Zero False Positives"** | `home.component.html:534,779` (comparison matrix) · `security.component.html:299` | "High-fidelity signals from confirmed interaction with decoy assets" |
| **"Reduces MTTR by 90%+"** | `faqs.component.html:115` · `index.html:126` (FAQ schema) · stat counters `mssps.component.html:244`, `ai-soc-analyst.component.html:91` | "Can materially reduce investigation and response time in validated workflows" (or attach methodology) |
| **"1 agent = 20+ analyst capacity"** | `faqs.component.html:118` · `index.html:126` | "Designed to process investigations continuously without increasing analyst headcount" |
| **"How does Novus Aegis AI replace SOC analysts?"** (analyst-replacement framing) | `index.html:123-126` (FAQ schema) · `faqs.component.html` (same Q visible) | Reframe as capacity amplification / supervised autonomy |
| **"zero human input"** | `index.html:110` (FAQ schema) | "Supports supervised and policy-controlled autonomous workflows" |
| **No-hallucination framing** | `index.html:115-119` + visible FAQ | "Evidence-grounded outputs with linked sources, confidence indicators and reviewable reasoning" |
| **"Invest in the future…" CTA** | `home.component.html:984` · `request-demo.component.html:269` · `company.component.html:109` | Route through a compliant investor page with disclaimers, or remove 🏢⚖️ |

> **Important:** the FAQ **schema** in `index.html` must stay in sync with the
> **visible** FAQ (`faqs.component.html`) when rewording — search engines penalize
> mismatches.

Also per audit: remove the fabricated `offers` block (price "0") from the
SoftwareApplication schema (`index.html:89-95`) — unsubstantiated properties.

---

## 4. P1 items — status & actions

| Item | Status | Action |
|---|---|---|
| Brand consistency (site vs LinkedIn vs GitHub) | 🏢 | One approved messaging hierarchy applied to all profiles; add GitHub to `sameAs` in Organization schema (currently only LinkedIn + Twitter, `index.html:66-69`) |
| Search-title clarity | ⚠️🏢 | Current: "AI-Powered Honeypot Platform \| Active Deception Security". Audit: "Novus Aegis AI \| Autonomous Cyber Defense & Honeypots". Depends on §3.6 decision; then edit `index.html:5` + `seo-routes.json` home entry |
| Meta descriptions | ✅ | Unique per route already — no action |
| Organization schema | ✅⚠️ | Exists; polish: `@id` cross-linking, logo as ImageObject w/ dimensions, add GitHub `sameAs` |
| Software schema | ✅⚠️ | Exists; remove `offers`, remove "patent-approved" from description pending §3.7 |
| FAQ schema | ✅⚠️ | Exists & matches visible FAQ; must be updated **together with** claim rewording |
| Comparison matrix (Dropzone/CrowdStrike/SOC Radar) | 🏢 | Date it, cite sources, drop unsupported cells (`home.component.html` ~line 779 table, `security.component.html` ~299) |
| Evidence architecture (testimonials) | 🏢 | Consent/role verification, measurable case studies — content work |
| Trust center (`/trust/`) | ❌🏢 | New page; `security.component.html` covers some of it — could be expanded/renamed |
| Deployment clarity | 🏢 | Content: deployment modes, permissions, data flow |
| Human oversight documentation | 🏢 | Content: approval gates, dry-run, rollback, emergency stop |
| AI safety / assurance page | ❌🏢 | New page + FAQ hallucination rewording |
| Per-integration pages (`/integrations/splunk/` etc.) | ❌ | Currently one `/integrations` page. New lazy routes + seo-routes entries per integration |
| Conversion paths per audience | ⚠️🏢 | `/request-demo` + `/mssps` exist; investor/government funnels don't. Analytics per CTA not instrumented |
| Form security | ✅ | **Done** (July 2026): server-side validation, rate-limit, reCAPTCHA verify, honeypot — `api/` service. **Remaining ❌: consent text** ("By submitting you agree to the Privacy Policy") on all 5 forms + a data-retention statement |
| Form wording ("Register" on the contact modal) | ❌ **quick win** | `app.component.html:188` — change `Register` → `Send Message` |

---

## 5. P2 items — status & actions

| Item | Status | Action |
|---|---|---|
| Accessibility (alt text) | ⚠️ | Many alts present (some meaningful, some empty-decorative); needs one audit pass over `src/app/pages/**` for missing/generic alts |
| Heading structure (one H1, logical H2/H3) | ⚠️ | Needs manual outline audit per page (home especially — many competing sections) |
| Page speed (LCP ≤ 2.5s) | ✅⚠️ | WebP/lazy/defer/preconnect done; measure real LCP on prod, then tune hero preload |
| Layout stability (CLS ≤ 0.1) | ✅ | width/height convention enforced |
| JS performance (INP ≤ 200ms) | ⚠️ | Lenis respects reduced-motion; legacy jQuery/owl-carousel/wow.js still global — candidates for deferral |
| Blog architecture | ⚠️ | Blog + per-post pages exist; ensure Article schema per post (via `SeoService.setStructuredData`), authors/dates visible, ≥3 evidence-led articles 🏢 |
| External identity signals | 🏢 | Align GitHub/LinkedIn descriptions; `sameAs` completion |
| Security headers (CSP, HSTS, etc.) | 🔧 | nginx config — add to `deploy/` snippets & apply on VPS |
| Error states | ❌ | `{ path: '**', redirectTo: '' }` (`app-routing.module.ts:33`) silently redirects unknown URLs home. Build a branded 404 page + wildcard route; keep form inline errors (MSSP forms have them; contact/demo still use `alert()` ⚠️ — replace with inline messages) |

---

## 6. Recommended execution order

### Phase A — Quick wins, no business sign-off needed (≤1 day)
1. 🐛 **Social-share image**: create 1200×630 card, swap all references (§2)
2. Favicon package (§3.3)
3. Contact button `Register` → `Send Message`
4. Remove `offers` from SoftwareApplication schema; add GitHub to `sameAs`
5. Consent line + Privacy Policy link on all 5 forms
6. Branded 404 page (replace silent wildcard redirect)
7. Replace `alert()` with inline success/error on contact & demo forms
8. Refresh sitemap `lastmod`

### Phase B — Needs business/legal decisions first 🏢
1. Claims rewording (§3.7-3.12) — **one approval doc, then a single sweep**
   (visible copy + FAQ schema + seo-routes together)
2. Master positioning / hero rewrite ("Turn Your Environment Into a Trap.")
3. robots.txt decisions (`/request-demo` disallow, GPTBot)
4. Comparison matrix sourcing; investor CTA compliance

### Phase C — DevOps 🔧
1. 301 `home.novusaegis.ai` → root domain
2. Security headers (CSP, HSTS, Referrer-Policy, Permissions-Policy, nosniff)
3. Confirm Search Console + Bing registration

### Phase D — Content expansion (multi-week)
1. Trust Center, AI assurance page
2. Per-integration pages, solution pages (`/solutions/ransomware/` etc.)
3. Blog: 3 evidence-led research articles
4. Audience-specific funnels + CTA analytics
5. Accessibility & heading-structure pass; Core Web Vitals measurement
