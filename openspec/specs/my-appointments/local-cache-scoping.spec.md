---
type: Behaviour Spec
title: "My Appointments — Local Cache Scoping"
description: "Defines when a completed booking is written to the device-local `localStorage['vetcard_appointments']` cache (`app/composables/useAppointmentStorage.ts`), and how `app/pages/my-appointments.vue` reads it back."
domain: my-appointments.local_cache_scoping
tags: [my-appointments, local-cache-scoping]
status: stable
last_verified_at: 2026-09-09
verification:
  type: test
  command: "npx vitest run tests/unit/useAppointmentStorage.test.ts"
sources:
  - id: useappointmentstorage
    resource: repo://app/composables/useAppointmentStorage.ts
  - id: my-appointments
    resource: repo://app/pages/my-appointments.vue
  - id: storage-test
    resource: repo://tests/unit/useAppointmentStorage.test.ts
---

# OpenSpec: My Appointments — Local Cache Scoping

## Domain
`my-appointments.local-cache-scoping`

## Overview
Defines when a completed booking is written to the device-local `localStorage['vetcard_appointments']` cache (`app/composables/useAppointmentStorage.ts`), and how `app/pages/my-appointments.vue` reads it back. The cache is a single, unscoped, device-wide list — it carries no per-account identity of its own — so the write side must not add entries that belong to an authenticated account, or they leak into the guest view after logout.

## 1. Write side — `app/pages/[slug]/appointment.vue`
- On a successful booking, `saveAppointment()` is called **only when `!isAuthenticated.value`**.
- Rationale: an authenticated booking is already persisted server-side against the account and is fetched via `/api/profile/appointments` (`my-appointments.vue`'s `loadAppointments`). Writing it to the local cache as well would leave a stale copy on the device after the user logs out.
- `saveUserProfile()` (owner name/phone/email autofill) is unaffected and still runs for every booking, authenticated or not.

## 2. Read side — `app/pages/my-appointments.vue`
- **Guest (`!user.value`)**: shows the local cache only, unfiltered (`getSavedAppointments()`), since guests have no account to fetch from.
- **Authenticated**: fetches `/api/profile/appointments` (account-scoped, source of truth) and merges in local-cache entries whose `id` isn't already present in the backend result. This merge exists to surface a booking made while still a guest, in the same session, before the backend has necessarily indexed it — not to reconcile identity. It assumes every local-cache entry belongs to the current visitor, which only holds because §1 keeps authenticated bookings out of the cache in the first place.

## 3. Known limitation
Devices that already accumulated a stale authenticated-booking entry in the cache before this fix keep it until the user manually removes it via the delete action in `my-appointments.vue` — there is no migration/cleanup of existing `localStorage` data.
