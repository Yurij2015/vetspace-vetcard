---
type: Behaviour Spec
title: "Booking — Turnstile Bot Protection"
description: "Defines the Cloudflare Turnstile CAPTCHA integration on the public appointment-booking form (`app/pages/[slug]/appointment.vue`), backed by `digispace-core-api`'s `turnstile` middleware — see that repo's `auth.turnstile` OpenSpec for the shared backend verification behavior (skip conditions, error codes, config keys)."
domain: booking.turnstile
tags: [booking, turnstile]
status: stable
last_verified_at: 2026-09-09
verification:
  type: e2e
  command: "npx cypress run --spec cypress/e2e/turnstile.cy.ts"
sources:
  - id: appointment_page
    resource: repo://app/pages/[slug]/appointment.vue
  - id: turnstile_test
    resource: repo://cypress/e2e/turnstile.cy.ts
---

# OpenSpec: Booking — Turnstile Bot Protection

## Domain
`booking.turnstile`

## Overview
Defines the Cloudflare Turnstile CAPTCHA integration on the public appointment-booking form (`app/pages/[slug]/appointment.vue`), backed by `digispace-core-api`'s `turnstile` middleware — see that repo's `auth.turnstile` OpenSpec for the shared backend verification behavior (skip conditions, error codes, config keys). This spec covers what's specific to this repo's frontend wiring and booking-form placement.

---

## 1. Site Key Wiring
- `nuxt.config.ts` sets an **explicit** `turnstile: { siteKey: process.env.NUXT_TURNSTILE_SITE_KEY || '' }` block — unlike `vetspace-front-app` (a sibling repo, no `NuxtTurnstile` config block there, relies on Nuxt's automatic `NUXT_PUBLIC_*` env override instead). Because this repo assigns the value explicitly from `process.env` at config-eval time, the env var here does **not** need a `PUBLIC_` prefix — `NUXT_TURNSTILE_SITE_KEY` (bare) is correct in this repo specifically. Do not "fix" this to match the other repo's naming; the two repos wire the same module two different ways, and each is correct for its own wiring.
- Each repo/site has its own Cloudflare site key (different registered origins) — do not copy one repo's key into the other's `.env`.

## 2. Widget Placement — Two Instances, One Mounted
- Two `<NuxtTurnstile data-testid="appointment-turnstile">` elements exist in the template (`turnstileRefA` in the desktop tree, `turnstileRefB` in the mobile tree) sharing the same `v-model="turnstileToken"`.
- Desktop vs. mobile are **not** a CSS-only `hidden lg:block` split — they're two entirely separate `v-if`/`v-else` component trees, gated on `isDesktop` (synchronous `window.matchMedia('(min-width: 1024px)')` read at setup; safe because this route is `ssr: false`). Only one tree — and therefore only one Turnstile instance — is ever actually mounted at a time. This is deliberate: `NuxtTurnstile` mounts side effects (script/API calls, timers) that must not run twice for one logical widget.
- The widget only renders for anonymous visitors: `v-if="!submitSuccess && !isAuthenticated"`. A logged-in owner booking an appointment does not see it, and neither layout renders it after a successful submission.

## 3. Acceptance Coverage
- `cypress/e2e/turnstile.cy.ts`: widget visible for an anonymous visitor; a full booking submission (mocked `POST /api/public/appointments`, same intercept pattern as `appointment.cy.ts`) still succeeds with the widget present; the mobile viewport (375px) renders the widget too (proving the mobile tree's own instance mounts, not just the desktop one).
- These tests don't assert a solved challenge — the site key is in Cloudflare's Testing Mode for this site (always passes, shows "For testing only..."), and the backend skips verification in `local`/`testing` `APP_ENV` regardless (see the `auth.turnstile` spec in `digispace-core-api`).
