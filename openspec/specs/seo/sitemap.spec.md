---
type: Behaviour Spec
title: "SEO — Sitemap & Public Crawl Contract"
description: "Public sitemap structure (`robots.txt` → `sitemap_index.xml` → per-locale `__sitemap__` files), the clinic-slug source route (`server/api/sitemap-clinics.get.ts`), the routes that must never appear in the sitemap, and the read-only public audit (`scripts/audit-vetcard.mjs`, `npm run audit:public`) that guards this contract across environments."
domain: seo.sitemap
tags: [seo, sitemap, robots, audit, vetcard]
status: stable
last_verified_at: 2026-09-11
verification:
  type: script
  command: "npm run audit:public && npx vitest run tests/unit/vetcardAudit.test.ts"
sources:
  - id: nuxt_config
    resource: repo://nuxt.config.ts
  - id: sitemap_clinics_route
    resource: repo://server/api/sitemap-clinics.get.ts
  - id: audit_script
    resource: repo://scripts/audit-vetcard.mjs
  - id: audit_unit_test
    resource: repo://tests/unit/vetcardAudit.test.ts
---

# OpenSpec: SEO — Sitemap & Public Crawl Contract

## Domain
`seo.sitemap`

## Overview
`@nuxtjs/sitemap` (bundled by `@nuxtjs/seo`, configured in `nuxt.config.ts`) exposes the site's public URLs to crawlers. Its identity is shared via the `site` block (`site.url` ← `NUXT_PUBLIC_SITE_URL`, default `https://vet-card.digispace.pro`). Clinic pages are injected from a custom server source (`server/api/sitemap-clinics.get.ts`) so every clinic slug appears once per locale. A read-only audit script (`scripts/audit-vetcard.mjs`) treats the resulting public surface as a contract and fails on violations. Only indexable, SSR-rendered content belongs in the sitemap — the CSR/`noindex` pages from `seo.indexing` must stay out.

---

## 1. Public Entry Points

On every environment origin the following URLs form the crawl contract:

| URL | Purpose |
| --- | --- |
| `/robots.txt` | Must contain `Sitemap: {origin}/sitemap_index.xml` for the **same** origin. |
| `/sitemap_index.xml` | Sitemap index (HTTP 200). |
| `/sitemap.xml` | Fallback sitemap (HTTP 200, or a same-origin 3xx redirect to the index). |
| `/__sitemap__/uk-UA.xml` | Per-locale sitemap (HTTP 200) — locales `uk-UA`, `en-US`, `pl-PL`. |

- The per-locale path format `__sitemap__/{uk-UA,en-US,pl-PL}.xml` is the live layout the audit asserts. Do not change module options so that this path moves without updating the audit too.
- Locale code ⇒ sitemap name mapping (`uk`→`uk-UA`, `en`→`en-US`, `pl`→`pl-PL`) is hardcoded in `scripts/audit-vetcard.mjs` (`LOCALE_SITEMAPS`). Adding/removing a locale requires updating that list.
- The sitemap set is `https://{origin}/` root-relative: every `<loc>` and every `xhtml:link rel="alternate" hreflang` in the per-locale files must resolve to the same origin. Cross-origin URLs and alternates are contract violations.

## 2. Environments

`ENVIRONMENTS` in `scripts/audit-vetcard.mjs` is the source of truth for audited origins:

- **testing** → `https://vet-card.digispace.pro` (also the `NUXT_PUBLIC_SITE_URL` default in `nuxt.config.ts`)
- **production** → `https://vetcard.pro`

`robots.txt` must reference the **current** environment's sitemap index — a stale/cross-environment `Sitemap:` line is a critical audit finding. In practice this means `NUXT_PUBLIC_SITE_URL` must be set to the real domain per environment (set in `.env.production` via CI).

## 3. Clinic Slug Source Route

`server/api/sitemap-clinics.get.ts` is registered as `sitemap.sources: ['/api/sitemap-clinics']` in `nuxt.config.ts`:

- It pages through the central catalog API (`GET {apiUrl}/api/clinic-catalog/clinics`, `per_page=100`, hard cap `maxPages = 200`) using `ssrApiUrl` + `apiHeaders`-equivalent headers (`X-Frontend-Key`, `X-Frontend-App: vetcard`, and the rewritten `Host` header when an internal `NUXT_API_URL` is configured) — the same host-rewrite rules as the clinic-catalog fetches in `app/composables/useClinicApi.ts`.
- It emits one entry per locale per unique slug: `loc: '/{uk|en|pl}/{slug}'`. The `@nuxtjs/sitemap` prefix strategy resolves these to absolute same-origin URLs.
- The route returns only `loc` (no `lastmod`/alternates) for the slug entries; static app routes are discovered automatically by the module.

## 4. Routes That Must Never Appear

`siteMap.exclude` in `nuxt.config.ts` removes the CSR/personal routes: `/*/my-appointments`, `/*/my-pets`, `/*/my-clinics`, `/*/auth/callback`, `/*/pets/link/**`. Beyond that, the audit treats any sitemap entry whose pathname matches

```
/(_nuxt|api|panel|auth|reset-password|verify-email|success|my-appointments|my-pets|my-clinics)(?:\/|$)/
```

as a **critical** violation — service, private, or one-time-token pages (`/pets/link/<token>`, reset/verify emails) must not be indexable. Booking (`*/appointment`) pages are excluded from SEO-indexable crawler content by the `seo.indexing` rules and the audit flags nothing for them by design, but they must not end up with a title/description/canonical mismatch either.

## 5. Verification

- **Unit** — `tests/unit/vetcardAudit.test.ts` pins the shared contracts: `ENVIRONMENTS` origins, `LOCALE_SITEMAPS`, and `parseSitemap` behaviour (`loc`/`lastmod`/`alternates`, relative→absolute, decoded entities).
- **Read-only audit** — `npm run audit:public` fetches, per environment: `robots.txt`, `sitemap_index.xml`, `sitemap.xml`, and the three per-locale sitemaps; collects all entries; and checks: robots references the current sitemap index, all sitemap/alternate URLs are same-origin, no duplicates, no service/private paths, every alternate is itself present in the sitemap set, every sitemap URL returns HTTP 200 with a title/description/canonical (same origin), a single visible H1, the expected `html lang`, and veterinary JSON-LD. Critical findings exit non-zero. Emits `vetcard-audit.{json,md}` (dir via `VETCARD_AUDIT_OUT`, default `audit-reports/`).
- Audit tuning env vars: `VETCARD_AUDIT_CONCURRENCY` (default 12), `VETCARD_AUDIT_TIMEOUT_MS` (20000), `VETCARD_AUDIT_RETRIES` (2), `VETCARD_AUDIT_DELAY_MS` (0), `VETCARD_AUDIT_ENV` (audit a single environment).