# Clinica — Healthcare Appointment & Follow-up Manager

Clinica is a role-aware healthcare appointment platform built with **React, TypeScript, Express, tRPC, Drizzle ORM, and MySQL/TiDB**. It provides separate, server-enforced patient, doctor, and administrator workflows; persisted scheduling; conflict prevention; transparent AI summaries; configuration-ready Calendar/email integrations; and patient-owned medication reminders.

> **Clinical safety notice.** AI-generated material is a summary of user- or clinician-supplied content only. It is prominently labeled as **not a diagnosis, medical advice, or a substitute for professional judgement**.

## Features and status

| Area | Implemented behavior | Current status |
| --- | --- | --- |
| Authentication | OAuth sessions plus development-only non-sensitive demo credentials; all protected procedures use the persisted database role. | Functional |
| Scheduling | Doctor working hours, leave, availability, booking, rescheduling, cancellation, lifecycle history, and 15-minute slot locks. | Functional |
| Conflict prevention | Unique patient/doctor slot constraints plus transaction-scoped locks block double booking and leave conflicts. | Functional |
| AI summaries | Server-only structured pre-visit and patient-friendly post-visit summaries with validated, persisted fallback states. | Functional with managed model proxy |
| Google Calendar | Server OAuth architecture, encrypted at-rest token persistence, callback-state binding, and appointment event create/update/delete paths. | Configuration-ready; no external event is attempted until credentials are configured |
| Email | Server-side Resend delivery adapter and immutable appointment/reminder delivery ledger. | Configuration-ready; unavailable provider is recorded as `unconfigured`, never as sent |
| Medication reminders | Patient-owned persisted reminder CRUD, validation, pause/resume/delete, due-time advancement, and scheduled delivery handler. | Functional management; external email delivery needs configuration and an enabled production schedule |

## Architecture

| Layer | Responsibility |
| --- | --- |
| `client/src/` | Role-aware workspaces and tRPC client. The browser never receives Google tokens, email keys, model credentials, or database connection data. |
| `server/routers/` | Validated role-gated tRPC contracts for scheduling, integrations, and medication reminders. |
| `server/integrations/calendar.ts` | Google OAuth state, token exchange/refresh, AES-256-GCM token encryption, and calendar event synchronization. |
| `server/integrations/email.ts` | Resend-backed email delivery and explicit unavailable/failed delivery tracking. |
| `server/reminders.ts` | Deterministic due-reminder processor. It has no in-process timer. |
| `server/reminderRoutes.ts` | Authenticated scheduled callback at `/api/scheduled/medication-reminders`. |
| `drizzle/schema.ts` | Durable users, clinician scheduling, appointment data, AI states, Calendar state, notification ledger, and medication-reminder records. |

## Data model

| Table | Purpose |
| --- | --- |
| `users` | Persisted OAuth/demo identity and `user` / `doctor` / `admin` role. |
| `doctorProfiles`, `doctorWorkingHours`, `doctorLeaves` | Clinician identity, weekly availability, and leave ranges. |
| `appointments`, `appointmentSlots`, `appointmentStatusEvents` | Appointment lifecycle plus database-level 15-minute conflict locks and history. |
| `calendarConnections`, `calendarOAuthStates`, `calendarEvents` | Server-only Calendar authorization state, encrypted tokens, and appointment event sync outcome. |
| `notificationDeliveries` | Immutable email attempt ledger. `sent`, `unconfigured`, and `failed` are intentionally distinct. |
| `medicationReminders` | Patient-owned medication schedule, local time zone, next due time, pause state, and delivery history reference. |

## Local development

```bash
pnpm install
pnpm dev
pnpm check
pnpm test
pnpm tsx scripts/session-control-smoke.mjs
pnpm tsx scripts/demo-account-smoke.mjs
pnpm tsx scripts/phase1-db-smoke.mjs
pnpm tsx scripts/integrations-fallback-smoke.mjs
pnpm build
```

The database migrations are under `drizzle/`. For every schema change, generate the migration, review its SQL, and apply it to the configured database through the project’s migration workflow.

## Demo accounts

The preview/development sign-in screen supports the following **non-sensitive test accounts**. They are persisted identities, not client-side role toggles. The demo credential path is disabled whenever `NODE_ENV=production`.

| Role | Displayed name | Identifier | Test password |
| --- | --- | --- | --- |
| Administrator | Avery Admin | `admin@clinica.demo` | `ClinicaDemo!Admin2026` |
| Doctor | Drew Doctor | `doctor@clinica.demo` | `ClinicaDemo!Doctor2026` |
| Patient | Parker Patient | `patient@clinica.demo` | `ClinicaDemo!Patient2026` |

After sign-in, the header displays the authenticated persisted name and role. **Log out** clears the existing server session. In development/preview only, **Switch demo account** re-authenticates through the same server-side credential validator and signed-session path.

## Managed environment requirements

Do **not** put values in client-side `VITE_*` variables, commit a real `.env` file, commit OAuth client files, or hardcode provider credentials. This managed project uses a documented configuration contract rather than a tracked `.env.example`.

| Variable | Required for | Server-only use |
| --- | --- | --- |
| `DATABASE_URL` | Application persistence | Drizzle/MySQL connection |
| `JWT_SECRET` | Existing session signing | Existing authenticated session model |
| `BUILT_IN_FORGE_API_URL`, `BUILT_IN_FORGE_API_KEY` | AI summaries and managed services | Existing managed AI proxy |
| `GOOGLE_CALENDAR_CLIENT_ID` | Calendar OAuth | Consent URL and code exchange |
| `GOOGLE_CALENDAR_CLIENT_SECRET` | Calendar OAuth | Code exchange and refresh-token exchange |
| `GOOGLE_CALENDAR_REDIRECT_URI` | Calendar OAuth | Must exactly match the Google Cloud authorized callback, ending in `/api/integrations/google-calendar/callback` |
| `CALENDAR_TOKEN_ENCRYPTION_KEY` | Calendar token storage | Base64-encoded 32-byte key for AES-256-GCM encryption at rest |
| `RESEND_API_KEY` | Live email delivery | Resend API authorization header |
| `EMAIL_FROM` | Live email delivery | Resend-verified sender address |

### Google Calendar activation

Create a Google OAuth **web application** client, enable Calendar API access, and register the exact production callback URL. The server requests only `https://www.googleapis.com/auth/calendar.events` when a patient explicitly connects Calendar. It generates a short-lived opaque state, hashes it in the database, validates it during callback, and encrypts access/refresh tokens at rest. Appointment creation/rescheduling syncs an event; cancellation deletes it. Without the four Calendar variables, the UI states that Calendar is unconfigured and the database records `unconfigured` rather than inventing a synced event. See [Google’s web-server OAuth guide](https://developers.google.com/identity/protocols/oauth2/web-server) and [Calendar Events API](https://developers.google.com/workspace/calendar/api/v3/reference/events/insert).

### Email activation

Set a Resend API key and verified sender address in managed server configuration. Appointment confirmation, reschedule, cancellation, and due medication reminders create delivery records. When no provider is configured, the application records `unconfigured` with a user-visible explanation; it never reports a message as sent. The code is compatible with the [Resend email API](https://resend.com/docs/api-reference/emails/send-email).

### Medication reminder delivery

Medication reminders are fully persisted and manageable from the patient workspace. The scheduled processor exists at `POST /api/scheduled/medication-reminders`; it authenticates the platform job and processes due records idempotently through the delivery ledger. It deliberately does not use `setInterval` or an in-process cron timer. Before enabling automatic delivery, publish the app and configure one production scheduled callback for that path. With no email provider configured, the processor advances the reminder and creates an explicit `unconfigured` delivery record without claiming notification delivery.

## Verification

| Check | What it verifies |
| --- | --- |
| `pnpm check` | Complete TypeScript contract validation. |
| `pnpm test` | Unit coverage for scheduling, AI fallback validation, demo authentication, authenticated UI contract, integration configuration/token encryption, and reminder due-time validation. |
| `scripts/session-control-smoke.mjs` | Logout invalidation, protected-route denial, and server-issued role switching. |
| `scripts/demo-account-smoke.mjs` | Database-derived demo roles, signed sessions, protected access, and credential rejection. |
| `scripts/phase1-db-smoke.mjs` | Admin/doctor/patient scheduling flow, concurrent booking, leave conflicts, rescheduling, cancellation, outcomes, and persisted AI generated-or-fallback state. |
| `scripts/integrations-fallback-smoke.mjs` | Explicit Calendar unconfigured state, patient-owned medication reminder CRUD, due processing, and honest unconfigured email-delivery ledger. |

## Deployment checklist

1. Run `pnpm check`, `pnpm test`, all smoke tests, and `pnpm build`.
2. Confirm the production database has applied all reviewed migrations.
3. Confirm the OAuth callback URL and verified email sender are registered **only** if Calendar/email are to be enabled.
4. Add provider values through managed secrets; never commit them or expose them to the browser.
5. Publish from a saved project checkpoint.
6. After publishing, configure the medication scheduled callback if automatic reminder processing is desired.

## Known limitations in current fallback mode

No real Google Calendar connection, calendar event, external email, or email reminder has been sent because external credentials have intentionally not been supplied. The underlying OAuth, encryption, provider call, event-sync, delivery-ledger, and scheduled processing paths are implemented and tested in configuration-absent fallback mode only. To validate real third-party delivery, provide managed credentials and run an isolated staging test with a non-sensitive test Calendar and a verified test email address.
