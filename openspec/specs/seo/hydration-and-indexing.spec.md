---
type: Behaviour Spec
title: "SEO — Rendering Strategy, Hydration Safety & Indexing"
description: "Rules for SSR vs CSR rendering splits, stable useFetch keys to prevent soft-404 traps, JSON-LD schema, and hreflang/canonical configuration."
domain: seo.indexing
tags: [seo, ssr, hydration, vetcard]
status: stable
last_verified_at: 2026-09-13
verification:
  type: e2e
  command: "npx cypress run --spec cypress/e2e/ssr-rendering.cy.ts"
sources:
  - id: catalog_page
    resource: repo://app/pages/index.vue
  - id: clinic_page
    resource: repo://app/pages/[slug]/index.vue
  - id: nuxt_config
    resource: repo://nuxt.config.ts
  - id: ssr_acceptance_test
    resource: repo://cypress/e2e/ssr-rendering.cy.ts
  - id: seo_locale_head
    resource: repo://app/composables/useSeoLocaleHead.ts
  - id: clinic_load_error
    resource: repo://app/utils/clinicLoadError.ts
---

# OpenSpec: SEO — Rendering Strategy, Hydration Safety & Indexing

## Domain
`seo.indexing`

## Overview
Defines how search-engine optimization and rendering strategies are structured across VetCard. The two indexable, SEO-critical pages are the **catalog** (`app/pages/index.vue`) and the **clinic page** (`app/pages/[slug]/index.vue`). Booking (`*/appointment`) and `my-appointments` are intentionally client-only and remain excluded from search indexes.

---

## 1. Rendering Strategy

- **Catalog & Clinic Pages (SSR)**: The server returns fully-rendered HTML (200 OK) with real content and meta tags, allowing web crawlers to index without executing client-side JavaScript. Both pages must remain strictly **hydration-safe**: no direct access to `localStorage`, `window`, or unseeded `new Date()` in the initial render path.
- **Booking & My-Appointments (`ssr: false`)**: Gated in `routeRules` within `nuxt.config.ts`. These pages rely on browser-specific state (`localStorage`, dynamic booking steps) and must not be indexed (`noindex`).

### Stable `useFetch` Keys (Soft-404 Trap Prevention)
The clinic page fetches clinic details using an explicit, URL-independent key:
```ts
const { data: clinic } = await useFetch(clinicApiUrl, {
  key: `clinic-${slug}-${branchId || 'main'}`,
  // ...
})
```
- **Rationale**: `ssrApiUrl` differs between server (internal `config.apiUrl`, e.g., `http://caddy`) and client (public `apiBaseUrl`). Without an explicit key, `useFetch` derives its cache key from the request URL. Because server and client URLs do not match, the SSR payload is not reused; the client triggers a redundant refetch. If this refetch encounters an intermittent failure, `clinic` becomes `null` and throws `createError(404)`.
- **Soft-404 Result**: The crawler receives a 200 SSR response, but the page turns into a 404 after client hydration. A stable, explicit cache key preserves the SSR payload and prevents the refetch. The catalog uses the same pattern (`key: 'clinics-catalog'`).

## 2. i18n: Alternate Hreflang, Canonical & HTML Lang

Locales are `uk` (default), `en`, `pl` with `strategy: 'prefix'`. All public routes are locale-prefixed. `i18n.baseUrl` (`nuxt.config.ts`) provides the absolute origin for canonical and alternate URLs (configured via `NUXT_PUBLIC_SITE_URL`).

All indexable pages (catalog, clinic, privacy, terms) invoke `useSeoLocaleHead(...)` (`app/composables/useSeoLocaleHead.ts`, a thin wrapper over `@nuxtjs/i18n`'s `useLocaleHead`), which emits:
- `<html lang>` and `dir` attributes.
- `rel="alternate" hreflang="..."` — **exactly one per language plus `x-default`**, in the region form (`uk-UA`, `en-US`, `pl-PL`). `useLocaleHead` alone also emits a language-only catch-all (`uk`) for every locale; the wrapper drops it so the page set is identical to the sitemap's `xhtml:link` alternates (which `@nuxtjs/sitemap` derives from `locale.language` with no override), `<html lang>`, `og:locale` and the audit's `LOCALE_SITEMAPS`.
- `rel="canonical"`.
- `og:locale` and `og:locale:alternate`.
- `og:url` (do not duplicate with manual `<meta property="og:url">`).

The wrapper also performs the single `MetaAttrs` → unhead cast, so pages spread its `htmlAttrs`/`link`/`meta` into `useHead` without `as any`.

### Title & Meta Description Sources (clinic page)
The API's `seo.title` / `seo.description` are auto-generated **Ukrainian** text (backend `SeoGenerator` / `VetCardBasicCatalogResource`, e.g. "… — Ветеринарні послуги в Cherkasy") unless a premium clinic authored them in the CRM, and the payload does not distinguish the two. Therefore:
- **Premium clinic on `uk`** — `seo.title` / `seo.description` from the API (may be CRM-authored), falling back to the templates below when empty.
- **Everything else** (basic clinics on any locale; premium clinics on `en`/`pl`) — built from localized templates: title `clinicSeo.titleWithCity` / `titleNoCity`; description = as many whole sentences of the localized intro (`clinicIntro.*`: name + city + address, phone, opening hours) as fit in 160 characters.
- `meta keywords` follows the same trust rule (the API's keywords are Ukrainian-only).
- The site-wide title template (`site.name`) appends ` | VetSpace`; templates must not add a brand suffix of their own.

### Canonical Strategy
- **Clinic Page**: `useSeoLocaleHead({ seo: { canonicalQueries: ['branch'] } })`. The `?branch=N` query parameter is preserved in the canonical URL because each branch provides distinct address, staff, and service information.
- **Catalog Page**: `useSeoLocaleHead()` (defaults to `{ seo: true }`). Search, filter, and pagination parameters (`?search`, `?city`, `?type`, `?page`) are excluded from the canonical link to consolidate ranking signals on the base catalog URL.

## 3. Structured Data (JSON-LD)

Structured data is injected via `useHead({ script: [{ type: 'application/ld+json', ... }] })`. Dynamic values are serialized with `JSON.stringify(...).replace(/</g, '\\u003c')` to prevent script-tag injection.

- **Clinic Page**: Generates [`VeterinaryCare`](https://schema.org/VeterinaryCare) / `LocalBusiness` including `name`, `url`, `address` (`PostalAddress`), `telephone`, `email`, `image`, `openingHoursSpecification`, and `aggregateRating` (when reviews exist). Country codes (`UA`, `PL`) are mapped to localized country names via `Intl.DisplayNames`. Includes `BreadcrumbList`.
- **Catalog Page**: Generates [`WebSite`](https://schema.org/WebSite) with a `SearchAction` and [`ItemList`](https://schema.org/ItemList) containing the clinics displayed on the current page.

## 4. Acceptance & Verification

Verified via `cypress/e2e/ssr-rendering.cy.ts`:
1. Raw HTTP request without JavaScript execution receives 200 status with full HTML content (not an empty `<div id="__nuxt"></div>` shell).
2. Booking routes (`/uk/<slug>/appointment`) return CSR shells without server-rendered localized H1 headers.
3. Parsed JSON-LD contains `WebSite`, `Organization`, `ItemList` on the catalog and `VeterinaryCare` / `BreadcrumbList` on clinic pages.
4. Non-existent slugs return 404 with `<meta name="robots" content="noindex">`. Both clinic pages map the failed fetch through `clinicLoadError()` (`app/utils/clinicLoadError.ts`): only an API 404 or an empty envelope is a 404 — a 5xx/timeout keeps its own status so a backend outage never serves crawlers soft 404s. The CSR booking page throws it with `fatal: true` (without `fatal` Nuxt keeps rendering and the page surfaces as a 500; covered by `cypress/e2e/appointment.cy.ts` "Unknown clinic slug").
5. hreflang alternates on the clinic page are exactly `x-default` + `uk-UA`, `en-US`, `pl-PL` (no language-only duplicates).
6. On `en`/`pl` the clinic title and description come from the localized templates and never contain the backend's Ukrainian "Ветеринарні послуги".
