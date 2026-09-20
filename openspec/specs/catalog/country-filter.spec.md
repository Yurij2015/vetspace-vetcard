---
type: Behaviour Spec
title: "Catalog — Country Filter"
description: "Defines how the public clinic catalog (`app/pages/index.vue`) consumes the admin-configured country filter settings from `digispace-core-api` (see that repo's `catalog.country_filter_settings` spec) instead of hardcoding a default country by locale, and how the filter's visibility and initial value are derived."
domain: catalog.country_filter
tags: [catalog, country-filter]
status: stable
last_verified_at: 2026-09-09
verification:
  type: test
  command: "npx vitest run tests/unit/catalogCountryFilter.test.ts"
sources:
  - id: index
    resource: repo://app/pages/index.vue
  - id: catalogcountryfilter
    resource: repo://app/utils/catalogCountryFilter.ts
  - id: catalogcountryfilter-test
    resource: repo://tests/unit/catalogCountryFilter.test.ts
---

# OpenSpec: Catalog — Country Filter

## Domain
`catalog.country_filter`

## Overview
Defines how the public clinic catalog (`app/pages/index.vue`) consumes the admin-configured country filter settings from `digispace-core-api` (see that repo's `catalog.country_filter_settings` spec) instead of hardcoding a default country by locale, and how the filter's visibility and initial value are derived.

---

## 1. Settings Consumption
- On load, fetches `GET {ssrApiUrl}/api/dictionary/catalog-settings` (`useFetch` key `catalog-settings`), defaulting to `{ country_filter_enabled: true, default_country: null }` if the response hasn't resolved yet — this default must never implicitly restrict results by country before the real settings arrive.
- Country dictionary options still come from `GET /api/dictionary/countries`, which itself only returns countries active in the admin (`is_active`, enforced backend-side).

---

## 2. Default Country Resolution
- `resolveDefaultCountryCode(settings, locale)` (`app/utils/catalogCountryFilter.ts`) is the single source of truth for the filter's initial value:
  1. If `settings.country_filter_enabled` is `false` (or `settings` is missing/unresolved) → `''` (no restriction).
  2. Else if `settings.default_country` is set → that country's `code`.
  3. Else → locale-based fallback: `uk` → `UA`, `pl` → `PL`, any other locale → `''`.
- `pages/index.vue`'s `localeCountry` computed delegates to this function; it is used both as the filter's initial value (`countryFilter` ref, only applied when `country_filter_enabled` is true — an explicit `?country=` query param is ignored while the filter is admin-disabled) and as the "is this the default, or has the user actively filtered" comparison baseline (`activeFilterCount`, `hasActiveFilters`, `clearFilters()`, and the URL-sync watcher that omits `?country=` when it equals the default).

---

## 3. Filter Visibility
- `resolveAvailableCountries(settings, countries, localizedName)` (`app/utils/catalogCountryFilter.ts`) returns `[]` whenever `settings.country_filter_enabled` is `false`, regardless of how many countries the dictionary has; otherwise it maps+sorts the dictionary countries by localized name.
- `pages/index.vue`'s `availableCountries` computed delegates to this function. No changes are needed in `CatalogFilters.vue` / `CatalogMobileFilters.vue` — both already hide the country dropdown via their existing `v-if="countries.length > 0"`, so an empty list from this function hides the control with zero template changes.

---

## 4. Acceptance Coverage
- `tests/unit/catalogCountryFilter.test.ts` covers `resolveDefaultCountryCode` (disabled → always `''` regardless of configured default/locale; enabled + configured default → that code; enabled + no default → locale fallback; missing settings → fails closed to `''`) and `resolveAvailableCountries` (disabled → `[]`; enabled → mapped+sorted list; missing/empty country list handled without throwing).
