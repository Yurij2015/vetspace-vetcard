---
type: Behaviour Spec
title: "My Appointments — Cancellation"
description: "The \"Remove\" action on `/my-appointments`."
domain: my-appointments.cancellation
tags: [my-appointments, cancellation]
status: stable
last_verified_at: 2026-09-09
verification:
  type: test
  command: "npx vitest run tests/unit/useAppointmentCancellation.test.ts"
sources:
  - id: composable
    resource: repo://app/composables/useAppointmentCancellation.ts
  - id: page
    resource: repo://app/pages/my-appointments.vue
  - id: useappointmentcancellation-test
    resource: repo://tests/unit/useAppointmentCancellation.test.ts
---

# OpenSpec: My Appointments — Cancellation

## Domain
`my-appointments.cancellation`

## Overview
The "Remove" action on `/my-appointments`. The page serves two kinds of visitor from one list — an account holder whose bookings come from the backend, and a guest whose bookings exist only in `localStorage` — and the action has to reach the clinic in both cases.

Previously it did not: a guest's "Remove" deleted the local record and never called the API, so the clinic kept holding the slot and had no idea the client had walked away.

---

## 1. Outcomes

`cancelAsAccountHolder` and `cancelAsGuest` both return a `CancelOutcome`:

- `'cancelled'` — the backend accepted it.
- `'gone'` — there is nothing on the backend to cancel (`404`, or a local record missing the slug/phone needed to identify it). Treated as success: the record is cleared locally, because leaving an entry the owner cannot ever remove is worse than losing it.
- `'failed'` — anything else (network, `403` on a past slot, `5xx`). The local record is kept and `myAppointments.removeError` is shown, so the list never claims a cancellation the clinic did not receive.

`cancelAppointment(id, isAccountHolder)` in `useAppointmentCancellation` dispatches on `!isNaN(Number(id))`: a numeric id is a real backend appointment, a generated id (`{timestamp}-{random}`, see `useAppointmentStorage`) is local-only, skips the network entirely and returns `'gone'`.

The page keeps only the UI half: `confirmRemove` reads `removeId`, clears it immediately, and treats anything but `'failed'` as success. The network half lives in the composable so it can be unit-tested — `tests/unit/useAppointmentCancellation.test.ts` covers both callers and every outcome.

## 2. Account Holder

`PATCH {apiBaseUrl}/api/profile/appointments/{id}/cancel` with the `auth_token` cookie as a bearer token.

## 3. Guest

`PATCH {apiBaseUrl}/api/clinic-catalog/vet-card/{clinic_slug}/appointments/{id}/cancel`, body `{ phone }`.

Both values come from the saved record (`getSavedAppointments().find(a => a.id === id)`): a guest booking stores the backend's own numeric id alongside `clinic_slug` and the `phone` it was booked with, which is exactly what the central endpoint needs to resolve the tenant and authorise the call. If either is absent — an old record written before this shape — the outcome is `'gone'`.

The route is central and slug-addressed rather than tenant-addressed because the stored booking does not carry the clinic's tenant domain; only the page that created it ever had that.

## 4. Headers

Both requests send `X-Frontend-Key`, `X-Frontend-App: vetcard`, `X-Locale` and `Accept: application/json`; the guest request adds `Content-Type: application/json` for its body. `X-Locale` matters here — the backend's cancellation messages are localised, and without it the owner reads an English error on a Ukrainian page.

## 5. What Stays Visible

A cancelled booking is **not** deleted from the clinic's CRM; it keeps its row with `status = 'cancelled'` (see `appointments.cancellation` in `digispace-core-api`). For an account holder it therefore comes back from `/api/profile/appointments` on the next load and the list renders it with the existing "cancelled" label instead of a Remove button. A guest has no such record to come back to, so their entry simply disappears.
