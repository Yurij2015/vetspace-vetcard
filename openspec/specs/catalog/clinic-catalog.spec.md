---
type: Behaviour Spec
title: "Catalog — Clinic Directory"
description: "The SSR clinic catalog: search, country/city filtering, verified badges, premium cards, and the per-branch listing model where each branch of a premium clinic is a separate catalog row."
domain: catalog.directory
tags: [catalog, ssr, seo]
status: stable
last_verified_at: 2026-09-21
verification:
  type: manual
  command: "reviewed against app/pages/index.vue"
sources:
  - id: index-page
    resource: repo://app/pages/index.vue
  - id: sitemap-clinics
    resource: repo://server/api/sitemap-clinics.get.ts
---

# OpenSpec: Catalog — Clinic Directory

## Domain
`catalog.directory`

## Overview

The index page (`/{locale}`) is the platform's public acquisition surface:
a paginated, searchable directory of every registered clinic. It is SSR and
SEO-critical — each clinic page is individually indexable, and
`server/api/sitemap-clinics.get.ts` feeds every clinic slug into the sitemap.

## Behavior

- Data: `GET {apiBaseUrl}/api/clinic-catalog/clinics` with `page`, `per_page`,
  `search`, `city_id`, `country` params.
- Filters: search-by-name, country, and city selects. The default country is
  admin-configured (see `catalog/country-filter.spec.md`), not hardcoded per
  locale.
- Premium clinics render as rich cards (logo/photo, verified badge, themed
  border); basic listings render plainly.
- For premium (tenant) clinics, **each branch is its own catalog row** — a
  3-branch clinic occupies three cards, each deep-linking to the clinic page
  with `?branch=<id>`.
- Grid/list view toggle; the choice is presentation-only.

## SEO

SSR-rendered headings, canonical links per locale (`uk`, `en`, `pl`),
`og:`/JSON-LD metadata — see `seo/hydration-and-indexing.spec.md`.
