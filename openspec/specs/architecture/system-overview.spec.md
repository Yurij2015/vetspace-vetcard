---
type: Behaviour Spec
title: "VetCard — System Overview"
description: "Top-level architecture of the VetCard public portal: a multi-tenant Nuxt 4 SSR application that serves a clinic catalog, per-clinic branded pages, and online booking by proxying requests to each clinic's own tenant backend."
domain: architecture.overview
tags: [architecture, overview, multi-tenancy]
status: stable
last_verified_at: 2026-09-21
verification:
  type: manual
  command: "reviewed against app/composables/useClinicApi.ts and nuxt.config.ts"
sources:
  - id: clinic-api
    resource: repo://app/composables/useClinicApi.ts
  - id: nuxt-config
    resource: repo://nuxt.config.ts
---

# OpenSpec: VetCard — System Overview

## Domain
`architecture.overview`

## Overview

VetCard is the public-facing surface of the VetSpace platform — a Nuxt 4 SSR
application serving two distinct products from one deployment:

1. **The VetSpace catalog** — a searchable directory of veterinary clinics
   (`/` index), branded as VetSpace, fed by the central catalog API.
2. **VetCard pages** — per-clinic branded micro-sites at `/{slug}` plus an
   online booking flow at `/{slug}/appointment`. These are the clinic's own
   presence, themed by the clinic's brand color, with a "Powered by VetSpace"
   badge.

## Request routing

Two API surfaces are consumed:

| Surface | Endpoint pattern | Used for |
|---|---|---|
| Central catalog API | `{apiBaseUrl}/api/clinic-catalog/*` | clinic list, clinic detail, reviews, guest cancellation |
| Per-tenant API | `{tenantDomain}/api/public/appointments`, `{tenantDomain}/api/appointments/public-available-slots` | slot availability, booking submission |

The tenant domain is resolved at runtime from the clinic payload
(`clinic.tenant.domains[0]`), so a single deployment serves booking for every
tenant without per-tenant builds or config.

## Rendering strategy (SEO-critical)

- `prefix` i18n strategy: every route lives under `/{uk|en|pl}`.
- Catalog and clinic pages are **SSR** — they must stay hydration-safe
  (no `new Date()` / `localStorage` in their render path).
- `/{slug}/appointment` and `/my-appointments` are **client-only**
  (`ssr: false` via `routeRules`) because they depend on local state.
- During SSR the catalog is fetched through an internal API URL
  (`config.apiUrl`, e.g. `http://caddy` in Docker) with the original `Host`
  header re-attached; the client uses the public base URL.

## Premium vs Basic listings

`isBasic = clinic.is_premium === false || clinic.tenant_id === null`

- **Basic** (catalog-only, unclaimed or free): name, contacts, map,
  "Is this your clinic?" claim CTA.
- **Premium** (paying tenant): rich layout — opening hours, doctors,
  services, gallery, reviews, per-clinic theme color, online booking.
