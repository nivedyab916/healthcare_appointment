# Clinica — Healthcare Appointment & Follow-up Manager

Clinica is a full-stack, role-aware healthcare appointment and follow-up manager. It gives **patients** a way to find clinicians, book and manage appointments, review follow-up information, and manage medication reminders. **Doctors** maintain their availability, leave, schedule, and visit outcomes. **Administrators** provision clinician profiles and monitor persisted scheduling operations.

The project is designed as a final GitHub submission: it includes the React client, Express/tRPC server, Drizzle schema and migrations, automated tests, smoke scripts, safe configuration documentation, and non-sensitive demo accounts. It intentionally contains **no production secrets, OAuth client secrets, provider tokens, database exports, or real personal credentials**.

> **Clinical safety notice.** Any AI-generated content is a summary of supplied symptoms or clinician-authored material only. Clinica labels it as **not a diagnosis, medical advice, prescription, or substitute for professional judgement**.

## Contents

- [Feature overview](#feature-overview)
- [Technology stack](#technology-stack)
- [Architecture](#architecture)
- [Quick start](#quick-start)
- [Database setup and migrations](#database-setup-and-migrations)
- [Demo accounts](#demo-accounts)
- [Authentication and authorization](#authentication-and-authorization)
- [AI summary behavior and fallback](#ai-summary-behavior-and-fallback)
- [Medication reminders](#medication-reminders)
- [Google Calendar and email integration status](#google-calendar-and-email-integration-status)
- [Optional live-integration variables](#optional-live-integration-variables)
- [Testing and verification](#testing-and-verification)
- [Deployment preparation](#deployment-preparation)
- [Security notes](#security-notes)
- [Known limitations](#known-limitations)

## Feature overview

| Area | Included functionality | Status |
| --- | --- | --- |
| Patient workspace | Searchable clinician directory, persisted availability, symptom intake, booking, rescheduling, cancellation, appointment status, follow-up information, and medication reminders. | Functional |
| Doctor workspace | Persisted weekly working hours, leave requests, role-gated schedule, clinician-facing pre-visit summary, post-visit notes, prescription capture, and completed visit status. | Functional |
| Administrator workspace | Persisted clinician-account linking and live counts for clinician profiles, appointments, allocated slot locks, and leave records. | Functional |
| Appointment safety | Transaction-scoped locks plus unique 15-minute patient and clinician slot allocations block double booking and scheduling over doctor leave. | Functional |
| AI assistance | Server-only structured pre-visit and patient-friendly post-visit summaries, validation, disclosure, persistence, and honest fallbacks. | Functional with the managed model proxy |
| Google Calendar | Server-side OAuth design, state binding, encrypted token storage, and appointment event create/update/delete code paths. | Configuration-ready; no external event is attempted until configured |
| Email | Server-side provider adapter and immutable delivery ledger for confirmations, reschedules, cancellations, and reminders. | Configuration-ready; unconfigured delivery is recorded honestly |
| Medication reminders | Patient-owned persisted create/read/update/delete, pause/resume controls, due-time calculation, and an authenticated scheduled processor. | Functional management; automatic email delivery needs a provider and schedule |

## Technology stack

| Layer | Technologies | Purpose |
| --- | --- | --- |
| Client | React 19, TypeScript, Tailwind CSS, shadcn/ui, Wouter, TanStack Query | Responsive role-aware healthcare workspaces. |
| Server | Node.js, Express 4, tRPC 11, Zod | Typed APIs, validation, session-aware procedures, and integration endpoints. |
| Persistence | Drizzle ORM with MySQL/TiDB | Relational users, schedules, appointment locks, reminder records, and integration audit state. |
| Authentication | Manus OAuth session framework and signed development demo sessions | Server-enforced sessions and database-derived roles. |
| AI | Managed server-side LLM proxy with Zod validation | Structured, non-diagnostic pre-visit and patient-friendly summaries. |
| Integrations | Google Calendar REST/OAuth design and Resend-compatible email adapter | Optional server-only Calendar and email delivery. |
| Test tooling | Vitest and database-backed Node smoke scripts | Unit, role, scheduling, fallback, and production-build validation. |

## Architecture

| Location | Responsibility |
| --- | --- |
| `client/src/` | Patient, doctor, and administrator UI plus typed tRPC client. The browser receives no provider token, OAuth secret, model credential, or database connection string. |
| `server/routers/` | Validated tRPC procedures for scheduling, authentication, integrations, notifications, and reminders. |
| `server/integrations/calendar.ts` | Google OAuth authorization URL creation, callback verification, token exchange/refresh, AES-256-GCM encryption, and calendar event synchronization. |
| `server/integrations/email.ts` | Provider-ready delivery adapter and explicit `sent`, `unconfigured`, or `failed` notification records. |
| `server/reminders.ts` | Deterministic due-reminder processor; it does not use an in-process timer. |
| `server/reminderRoutes.ts` | Authenticated scheduled endpoint at `POST /api/scheduled/medication-reminders`. |
| `drizzle/schema.ts` | Database definition for users, doctors, appointments, locks, AI states, Calendar state, notifications, and medication reminders. |
| `scripts/` | Disposable smoke tests that create and remove test records in the configured development database. |

### Core data model

| Table | Purpose |
| --- | --- |
| `users` | Persisted identity with `user`, `doctor`, or `admin` role. |
| `doctorProfiles`, `doctorWorkingHours`, `doctorLeaves` | Clinician profile, weekly working hours, and leave ranges. |
| `appointments`, `appointmentSlots`, `appointmentStatusEvents` | Appointment lifecycle, 15-minute conflict locks, and immutable status history. |
| `calendarConnections`, `calendarOAuthStates`, `calendarEvents` | Server-only Calendar authorization state, encrypted tokens, and event synchronization result. |
| `notificationDeliveries` | Immutable notification audit ledger; provider absence is distinct from delivery success. |
| `medicationReminders` | Patient-owned medication schedule, time zone, next due time, pause state, and delivery tracking. |

## Quick start

### Prerequisites

Install a supported Node.js runtime, pnpm, and access to a MySQL 8+ or TiDB database. The managed project runtime also supplies existing OAuth and AI configuration. For a GitHub clone outside that runtime, provide only the required variables through your own secure secret manager; do not commit them.

### Install and run

```bash
git clone https://github.com/<YOUR_GITHUB_ACCOUNT>/healthcare_appointment.git
cd healthcare_appointment
pnpm install
pnpm dev
```

The development server starts through the project’s existing server entry point. Use the displayed local URL to open the application.

### Seed safe demo identities

```bash
pnpm tsx scripts/seed-demo-accounts.mjs
```

This command creates only the documented non-sensitive demo identities. It does not create real provider accounts, real Calendar connections, or real patient data.

## Database setup and migrations

Clinica stores business timestamps as UTC and displays them in the client’s local time. Clinician working hours are interpreted in the clinician’s configured IANA time zone.

1. Create or provision an empty MySQL/TiDB database through your hosting provider.
2. Configure a **server-only** `DATABASE_URL` using your deployment secret manager.
3. Review the SQL files under `drizzle/` before applying them to any shared environment.
4. Apply the reviewed migrations using the project’s Drizzle workflow.
5. Run the smoke scripts only against a disposable development or staging database; they create and clean up temporary test records.

```bash
pnpm drizzle-kit generate
pnpm db:push
```

> Do not commit a database dump, connection string, backup, or a local `.env` file. Use the platform’s secret-management interface for database configuration.

## Demo accounts

The sign-in screen includes a development/preview-only **Demo access** panel. These are intentionally non-sensitive test identities, not real people and not production credentials. The demo credential endpoint is disabled when `NODE_ENV=production`.

| Role | Displayed name | Identifier | Non-sensitive test password |
| --- | --- | --- | --- |
| Administrator | Avery Admin | `admin@clinica.demo` | `ClinicaDemo!Admin2026` |
| Doctor | Drew Doctor | `doctor@clinica.demo` | `ClinicaDemo!Doctor2026` |
| Patient | Parker Patient | `patient@clinica.demo` | `ClinicaDemo!Patient2026` |

Each demo login is validated **on the server**, then receives the same signed-session format used by OAuth. The server loads the matching persisted user and enforces its database role on every protected procedure. The header displays the authenticated user’s persisted name and role. The development-only **Switch demo account** control re-authenticates through the server; it is not a browser-only role preview.

## Authentication and authorization

| Capability | How it is enforced |
| --- | --- |
| Session issuance | OAuth callback or server-validated development demo credentials issue the signed, HTTP-only session. |
| Logout | The existing logout procedure clears the session cookie and preview state. Protected procedures reject the post-logout request. |
| Patient access | Patients may only view and change their own appointments and medication reminders. |
| Doctor access | A doctor must have a linked clinician profile to manage availability, leave, schedule, and assigned visit outcomes. |
| Administrator access | Administrator procedures provision clinician profiles and access operational aggregate counts. |
| Role source of truth | The `users.role` column is evaluated on the server; the client never grants or changes a role. |

## Appointment scheduling and conflict prevention

Every active appointment owns a record in `appointmentSlots` for each 15-minute interval in its duration. Database uniqueness prevents the same patient or doctor from obtaining an overlapping active slot. Transaction-scoped named locks serialize booking, rescheduling, cancellation, leave, and status changes so the conflict checks cannot be bypassed by simultaneous requests.

The server also validates clinician activity, configured working hours, leave ranges, appointment ownership, assigned-doctor access, and permitted lifecycle transitions. Rescheduling retains the old appointment as `rescheduled` history, releases the old slot locks, and allocates locks for the new confirmed appointment.

## AI summary behavior and fallback

### Pre-visit symptom summary

When a patient books or reschedules, the server requests a concise clinician-facing summary, neutral urgency label, and exactly three review questions. The model is called only from `server/aiSummaries.ts`; no LLM credential is present in browser code.

### Patient-friendly post-visit summary

When an assigned doctor or administrator records the post-visit outcome, the server can produce a plain-language summary of the clinician-authored notes, prescription, and follow-up material. It does not add clinical facts or alter clinician instructions.

### Validation and honest fallback

Model output is requested as strict JSON and validated with Zod. If model selection, request delivery, JSON parsing, or schema validation fails, appointment booking or outcome recording still succeeds. The application persists an explicit `fallback` state and useful original-content-based material; it never fabricates a generated summary. The UI tells users that AI content is a summary, **not a diagnosis**.

## Medication reminders

Patients can create, edit, pause, resume, and delete only their own medication reminders. Each reminder stores a medication label, instructions, local time, time zone, active state, and next due time. The patient workspace shows reminder and provider status without implying that an email was sent.

The scheduled due-reminder processor is exposed at:

```text
POST /api/scheduled/medication-reminders
```

It uses an authenticated scheduled-job request and advances due reminders idempotently through `notificationDeliveries`. It intentionally avoids `setInterval` or an in-memory cron loop, which would not be reliable in a serverless deployment. Before automatic delivery is enabled, publish the application and configure a production schedule for this endpoint.

## Google Calendar and email integration status

### Current credential-free fallback mode

No live Google Calendar event, external email, or email reminder is sent in the current submission because provider credentials were intentionally not supplied. The app records `unconfigured` instead of `synced` or `sent`, and the patient UI explains the configuration state. This is deliberate: the application does not simulate third-party success.

### Google Calendar implementation

The implementation includes server-side OAuth initiation and callback routes, short-lived opaque state bound to the user, hashed state persistence, encrypted access/refresh tokens, refresh support, and appointment event create/update/delete paths. It requests the event scope only when a patient explicitly connects Calendar. Set the optional Calendar variables below and register the exact callback before conducting a staging test. [1] [2]

### Email implementation

The implementation has a server-only Resend-compatible adapter. Appointment confirmation, reschedule, cancellation, and due medication processing write notification-delivery records. Without an email provider, records remain `unconfigured`; with a configured provider, the adapter can record `sent` or `failed` according to the real response. [3]

## Optional live-integration variables

The GitHub submission intentionally does not include a tracked `.env.example` or any secret values. Add values through a deployment secret manager only. Do **not** put server credentials in a browser-readable `VITE_*` variable.

| Variable | Required for | Server-side use only |
| --- | --- | --- |
| `DATABASE_URL` | Database persistence | Drizzle/MySQL connection. |
| `JWT_SECRET` | Session signing | Existing authenticated session model. |
| `BUILT_IN_FORGE_API_URL` | Managed AI access | Existing server-side model proxy endpoint. |
| `BUILT_IN_FORGE_API_KEY` | Managed AI access | Existing server-side model proxy authorization. |
| `GOOGLE_CALENDAR_CLIENT_ID` | Calendar OAuth | Consent URL and authorization-code exchange. |
| `GOOGLE_CALENDAR_CLIENT_SECRET` | Calendar OAuth | Authorization-code and refresh-token exchange. |
| `GOOGLE_CALENDAR_REDIRECT_URI` | Calendar OAuth | Exact registered callback URL ending in `/api/integrations/google-calendar/callback`. |
| `CALENDAR_TOKEN_ENCRYPTION_KEY` | Calendar token security | Base64-encoded 32-byte AES-256-GCM key used only by the server. |
| `RESEND_API_KEY` | External email delivery | Resend API authorization header. |
| `EMAIL_FROM` | External email delivery | Verified sender address. |

To activate live integrations, configure values in a staging environment, register the OAuth callback with Google, use a non-sensitive test Calendar and verified test recipient, run a real appointment lifecycle, and inspect the persisted `calendarEvents` and `notificationDeliveries` outcomes. Do not enable on a production data set before completing that staging validation.

## Testing and verification

Run the following from the project root:

```bash
pnpm check
pnpm test
pnpm tsx scripts/session-control-smoke.mjs
pnpm tsx scripts/demo-account-smoke.mjs
pnpm tsx scripts/phase1-db-smoke.mjs
pnpm tsx scripts/integrations-fallback-smoke.mjs
pnpm build
```

| Command or script | Verified behavior |
| --- | --- |
| `pnpm check` | TypeScript contracts across client and server. |
| `pnpm test` | Unit coverage for scheduling rules, AI generated/fallback validation, demo authentication, authenticated header binding, integration configuration/token encryption, and reminder due-time validation. |
| `session-control-smoke.mjs` | Logout invalidation, protected-route rejection after logout, and role switching via server-issued sessions. |
| `demo-account-smoke.mjs` | Database-derived demo roles, signed session issuance, protected role access, and invalid credential rejection. |
| `phase1-db-smoke.mjs` | Administrator–doctor–patient workflow, concurrent booking, leave conflict prevention, reschedule lifecycle, cancellation release, visit outcomes, AI generated-or-fallback persistence, and honest Calendar/email fallback statuses. |
| `integrations-fallback-smoke.mjs` | Explicit unconfigured Calendar state, patient-owned medication reminder CRUD, due processing, and unconfigured email ledger behavior. |
| `pnpm build` | Production client/server build. |

## Deployment preparation

1. Run the complete verification commands above.
2. Apply reviewed Drizzle migrations to the production database.
3. Configure OAuth, model, database, Calendar, and email values only through managed secrets.
4. Publish from a saved project checkpoint or your approved GitHub CI/CD pipeline.
5. Register the Calendar callback and verified sender only when those optional integrations are enabled.
6. Configure the production medication-reminder scheduled callback only after publishing.
7. Run an isolated live-provider staging test before asserting external delivery in production.

See [`GITHUB_SUBMISSION.md`](GITHUB_SUBMISSION.md) for package contents and a safe GitHub upload workflow.

## Security notes

| Control | Implementation |
| --- | --- |
| Secret isolation | Provider keys, OAuth secrets, encrypted Calendar tokens, and database credentials are server-side only. |
| Token protection | Calendar access and refresh tokens are encrypted at rest with AES-256-GCM; OAuth state is stored as a hash and expires. |
| Authorization | Every protected route resolves the session and persisted role server-side; no client role switch grants access. |
| Scheduling integrity | Database unique constraints and transaction-scoped locks prevent active appointment overlap. |
| Fallback transparency | Missing configuration or provider failure records `unconfigured`/`failed`, never a fabricated sent/synced result. |
| Submission hygiene | `.gitignore` excludes environment files, dependencies, build output, logs, platform artifacts, and local configuration. |
| Test data | Documented credentials belong only to synthetic demo accounts and are disabled in production. |

## Known limitations

The implementation is fully verified in **credential-free fallback mode** for Calendar and email. It has not sent a real external email or created a real Google Calendar event because no provider credentials were supplied for this project. Automatic medication reminder processing requires a published deployment and a configured scheduled callback; actual reminder email delivery also requires the optional email provider configuration.

## References

[1]: https://developers.google.com/identity/protocols/oauth2/web-server "Google OAuth 2.0 for Web Server Applications"
[2]: https://developers.google.com/workspace/calendar/api/v3/reference/events "Google Calendar Events API"
[3]: https://resend.com/docs/api-reference/emails/send-email "Resend Send Email API"
