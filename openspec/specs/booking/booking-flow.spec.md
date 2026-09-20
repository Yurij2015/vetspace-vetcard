---
type: Behaviour Spec
title: "Booking — Public Appointment Flow"
description: "End-to-end public booking: branch selection, date picking, doctor-scoped slot availability fetched from the clinic's own tenant API, guest form with Turnstile protection, and local persistence of the guest's appointments."
domain: booking.flow
tags: [booking, appointments, tenant-api]
status: stable
last_verified_at: 2026-09-21
verification:
  type: manual
  command: "reviewed against app/pages/[slug]/appointment.vue and app/composables/useClinicApi.ts"
sources:
  - id: appointment-page
    resource: repo://app/pages/[slug]/appointment.vue
  - id: clinic-api
    resource: repo://app/composables/useClinicApi.ts
  - id: appointment-storage
    resource: repo://app/composables/useAppointmentStorage.ts
---

# OpenSpec: Booking — Public Appointment Flow

## Domain
`booking.flow`

## Overview

`/{slug}/appointment` is a client-only page that lets any visitor book a visit
without an account. The flow talks to the **tenant backend of that specific
clinic**, not the central API — slots and bookings live in tenant scope.

## Steps

1. **Branch selection** — multi-branch clinics list their branches; the choice
   scopes slots, doctors and rooms. `?branch=<id>` is threaded from the clinic
   page.
2. **Date + slot** — a calendar picks the day; `GET {tenant}/api/appointments/
   public-available-slots` returns the grid. Each slot carries a `count` of
   free doctors, rendered as a scarcity badge (red ≤1, amber 2–3, emerald ≥4).
3. **Owner/pet details** — name, phone (international format), pet name,
   species, breed, reason. Cloudflare Turnstile guards submission
   (see `booking/turnstile.spec.md`).
4. **Submit** — `POST {tenant}/api/public/appointments`. On success the booking
   is stored in `localStorage` (`vetcard_appointments`) so a guest can find and
   cancel it later — see `my-appointments/local-cache-scoping.spec.md`.

## Guest vs signed-in

Signed-in pet owners (Google OAuth) get their saved appointment list merged
from the backend; guests rely on the phone-number-verified cancellation
endpoint (`PATCH /api/clinic-catalog/vet-card/{slug}/appointments/{id}/cancel`,
rate-limited — the phone number is the only credential).
