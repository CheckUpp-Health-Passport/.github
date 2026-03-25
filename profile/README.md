# CheckUpp

CheckUpp is a digital health passport and preventive care planner. It helps individuals stay on top of health screenings, store medical documents, plan pregnancy milestones, track immunizations, set reminders, and optionally share health data with clinicians through explicit, patient-controlled consent.

The product spans four applications: a patient-facing **mobile app** (React Native / Expo), a shared **backend API** (Express / Prisma / PostgreSQL), a **clinician web portal** (Next.js), and a public **marketing website** (Next.js). Each is built and deployed independently.

---

## Table of Contents

- [Why CheckUpp Exists](#why-checkupp-exists)
- [The Patient Mobile App](#the-patient-mobile-app)
  - [Onboarding](#onboarding)
  - [Home Dashboard](#home-dashboard)
  - [Cancer Screening](#cancer-screening)
  - [General Health Screening](#general-health-screening)
  - [How Screening Scheduling Works](#how-screening-scheduling-works)
  - [Screening History and PDF Export](#screening-history-and-pdf-export)
  - [Pregnancy Planner](#pregnancy-planner)
  - [Doc Wallet](#doc-wallet)
  - [Reminders](#reminders)
  - [Immunization Tracker](#immunization-tracker)
  - [Consent Center](#consent-center)
  - [Profile and Settings](#profile-and-settings)
- [The Backend API](#the-backend-api)
  - [Authentication and Roles](#authentication-and-roles)
  - [Screening Module](#screening-module)
  - [Wallet, Pregnancy, and Feedback Modules](#wallet-pregnancy-and-feedback-modules)
  - [Patient Consent Module](#patient-consent-module)
  - [Clinician Module](#clinician-module)
  - [Data Model Overview](#data-model-overview)
  - [Audit Logging](#audit-logging)
  - [Data Migration from Appwrite](#data-migration-from-appwrite)
- [The Clinician Web Portal](#the-clinician-web-portal)
  - [Clinical Operations Dashboard](#clinical-operations-dashboard)
  - [Clinician Profile](#clinician-profile)
  - [Patient Directory, Linking, and Consent Management](#patient-directory-linking-and-consent-management)
  - [Patient Timeline](#patient-timeline)
- [The Marketing Website](#the-marketing-website)
- [Security and Privacy](#security-and-privacy)
- [Future Direction](#future-direction)

---

## Why CheckUpp Exists

Preventive healthcare is hard to maintain. Screening guidelines vary by age, sex, and risk profile. Intervals range from six months (dental) to five years (cervical). People receive advice from GPs, specialists, workplace health programs, and national screening invitations that do not synchronize with each other. Life events—moving cities, changing jobs, pregnancy—break continuity.

CheckUpp addresses this by giving individuals a single place to:

1. See which screenings are due, overdue, or not yet applicable to them based on their age and sex.
2. Log results and track history so they are not starting from zero at each appointment.
3. Store documents (referrals, imaging reports, prescriptions, eScript links) so they are portable.
4. Plan pregnancy-related milestones with gestational week anchoring.
5. Record immunization history with dose series, batch numbers, and category typing.
6. Set reminders that connect to device calendars and notifications.
7. Choose whether and how to share any of this data with a clinician.

CheckUpp frames its recommendations around evidence-based national guidelines. Marketing copy references Australian preventive-health guidelines specifically, and screening definitions in the backend carry guideline version metadata so intervals can evolve.

---

## The Patient Mobile App

The mobile app is built with **Expo SDK 54**, **React Native**, and **expo-router** for file-based navigation. Authentication uses **Firebase** (email/password and Google sign-in via `@react-native-firebase/auth` and `@react-native-google-signin/google-signin`). Server-state is managed with **TanStack Query** and local/UI state with **Zustand**. Forms use **Formik** with **Yup** validation. Styling uses **NativeWind** (Tailwind for React Native).

The app is registered under the Expo slug `healthpassport` and bundle identifier `com.app.healthpassport`, currently at version **2.1.2**.

### Onboarding

New users see a multi-screen onboarding flow that introduces the product's purpose: why regular checkups matter, how often to visit based on age and health status, and what types of screenings the app covers. Users can sign up with email/password or Google, or sign in if they already have an account. Onboarding routes to `/home` once the user is authenticated.

### Home Dashboard

The home screen loads the user's cancer and health screening data from local storage, aggregates eligible screenings across both domains, computes completed counts and a completion percentage, and displays:

- A **time-of-day greeting** (Good Morning / Afternoon / Evening) with the user's first name.
- A **completion progress** summary showing how many eligible screenings have been completed.
- Up to **three upcoming screenings** sorted by due date.
- Quick-action cards that route into the cancer or health screening tabs.

The dashboard supports **pull-to-refresh** and uses scroll-driven header animations for a polished feel.

### Cancer Screening

CheckUpp tracks five cancer screening programs. Each has explicit eligibility rules defined in `cancerScreeningChecks.ts`:

| Screening | Eligible Sex | Age Range | Interval |
|-----------|-------------|-----------|----------|
| Cervical Cancer | Female | 25–75 | 5 years |
| Breast Cancer | Female | 40–70 | 2 years |
| Bowel Cancer | All | 45–70 | 2 years |
| Prostate Cancer | Male | 50+ | 2 years |
| Lung Cancer | All | 50+ | 2 years |

For each program, the scheduling engine determines:

- **Eligibility** based on the user's age and gender.
- **Next due date** calculated from the last test date (per-test specific if available, otherwise the generic last screening date). If the user has never been screened, the due date is today.
- **Overdue status** by comparing the computed next date against today.
- **Recommended flag** set when the item is both eligible and overdue.

Users interact with cancer screening through tabbed views: a **schedule** showing all five programs with status indicators, a **data entry form** (`CancerScreeningForm`) to input age, gender, last test dates, and per-screening results, and **action buttons** to save data, push dates to the calendar, and schedule notifications.

### General Health Screening

Health screening covers five additional domains, defined in `nutritionChecks.ts`:

| Screening | Min Age | Interval | Result Units |
|-----------|---------|----------|--------------|
| Cardiovascular Health | 45 | 2 years | mmHg, kg, cm, mmol/L |
| Diabetes Check | 40 | 3 years | mmol/L |
| Vision Check | 16 | 2 years | 6/6 |
| Dental Check | 16 | 6 months | Yes/No |
| Mental Health Check | 0 | 2 years | K10, AUDIT C |

These use the same scheduling logic as cancer screening (eligibility → last date → next date → overdue → recommended) but with age-only eligibility (no gender restriction). Users input data through `HealthCheckForm` and see results in the same tabbed screening view alongside cancer.

### How Screening Scheduling Works

Both cancer and health screening scheduling follow the same algorithm:

1. Look up the screening's configuration (interval in years, age/gender eligibility).
2. Check whether the user is eligible based on their current age and gender.
3. Find the best anchor date: the per-test result date if the user recorded one for this specific test, or the generic last screening date, or nothing if they have never been screened.
4. If never screened or no anchor date, the next due date is today.
5. Otherwise, add the interval in years to the anchor date. If the result is in the past, set to today (overdue).
6. Mark recommended if eligible AND overdue.

The backend mirrors this with a richer model: `ScreeningDefinition` entries store intervals in months rather than years, and `ScreeningPlan` / `ScreeningDueItem` records persist calculated due dates and completion states.

### Screening History and PDF Export

Users can review their screening history through a `HistoryViewer` component that renders past entries with parsed results, structured fields, measurements, and notes. There is also a PDF export utility (`exportPdf.ts`) that generates styled HTML summaries of screening history using `expo-print` and shares them via `expo-sharing`, so users can produce a printable summary for a doctor's visit.

### Pregnancy Planner

The pregnancy planner is visible only when the user's profile gender is female (the tab is hidden via `href: null` for other users). It is built around a **checkup milestone sequence** anchored to gestational weeks, calculated by subtracting 40 weeks from the due date to get the estimated LMP, then adding each milestone's week offset:

| Milestone | Week |
|-----------|------|
| Book First Hospital Appointment | 4 |
| Non-Invasive Prenatal Testing (NIPT) | 10 |
| Routine Blood & Urine Tests | 10 |
| First Trimester Screening (cFTS) | 12 |
| Influenza Vaccination | 14 |
| Morphology Ultrasound | 20 |
| Book Parent Education Classes / Hospital Tour | 24 |
| Gestational Diabetes Screening | 26 |
| Pertussis (Whooping Cough) Vaccination | 28 |
| Group B Streptococcus (GBS) Swab | 36 |
| Third Trimester Ultrasound | 37 |
| Birth Plan Discussion | 37 |
| Postnatal: Newborn Check (2 weeks) | 42 |
| Postnatal: Maternal Check (6 weeks) | 46 |

Users set their due date, the app calculates all milestone dates, and items can be marked as completed. The planner integrates with calendar and notifications so users get prompted before each milestone. Data is persisted server-side via the pregnancy plan API endpoint and also synced with the legacy Appwrite backend during the transition period.

### Doc Wallet

The wallet organizes health documents into two tabs:

- **Documents** — uploaded files and images (PDFs, photos of referrals, imaging reports, medical certificates). Users can upload via the device's document picker or camera.
- **Links** — eScript links and other external URLs stored with metadata.

Each document has a title, document type, file type (`FILE`, `IMAGE`, or `LINK`), and optional descriptions. The wallet includes inline viewers for images (`ImageViewer`) and documents (`DocumentViewer`), plus an upload modal (`UploadModal`) for new entries. Files are stored via the API with object storage keys and served back via URLs.

### Reminders

The reminders system allows users to create, view, edit, and delete reminders for any health event. Each reminder is stored locally with a title, date, and optional notes. The system:

- Schedules local **push notifications** using `expo-notifications`.
- Can add events to the **device calendar** using `expo-calendar`.
- Supports **cancellation** of specific notifications when reminders are deleted or modified, using stored notification identifiers.

Reminders are user-defined and complement the system-generated screening due dates — they cover things like "call GP to book," "follow up on blood test results," or "pick up referral."

### Immunization Tracker

The immunization module records vaccinations with rich typing per entry:

- **Vaccine identity**: name, type (`routine`, `travel`, `occupational`, `catch-up`, `booster`), brand, batch number.
- **Dose tracking**: dose number within a series and total doses expected.
- **Administration**: site (left-arm, right-arm, left-thigh, right-thigh, oral, nasal), provider name, clinic, and location.
- **Side effects**: none / mild / moderate / severe flags with optional description.
- **Travel context**: whether the vaccine is travel-related, destination, and departure date.
- **Next due date** for multi-dose series.

The module also includes a reference section with the **NSW 2025 Immunisation Schedule** covering infants (birth through 18 months), children (4 years), adolescents (12–13), adults (18+ through 70+), pregnancy vaccines, and travel vaccine recommendations by destination region. This serves as an educational reference alongside the user's personal records.

### Consent Center

The consent center (`/consent-requests`) is where patients manage clinician access to their data. The screen has two sections:

**Pending Requests** — consent requests from clinicians in `REQUESTED` status. For each request, the patient sees the clinician's name, specialty, organization, and the requested scope. The patient can:

- **Approve** — opens a modal where they can customize the grant: select which domains to share (`screenings`, `documents`, `pregnancy`, `feedback`, `profile`), choose access level (`READ_ONLY` or `READ_WRITE`), and toggle whether history is included. At least one domain must be selected.
- **Decline** — with confirmation dialog.

**History** — past consent decisions filtered by tabs: `ACTIVE`, `DECLINED`, `REVOKED`. Active consents can be **revoked** at any time with a confirmation that the clinician will immediately lose access.

The mobile app calls the patient-side consent API: `GET /me/consents/requests`, `POST /me/consents/:id/approve`, `POST /me/consents/:id/decline`, `POST /me/consents/:id/revoke`.

### Profile and Settings

The profile screen displays the user's avatar (or initials), name, and email. It provides navigation to:

- **Personal Details** — edit name, contact info, demographics.
- **Health Data** — shortcut back to the screenings tab.
- **Reminders** — manage health reminders.
- **Consent Requests** — manage clinician access.

A menu provides access to **Privacy Policy**, **Help & Support**, and **Settings**. Logout requires confirmation ("Are you sure you want to logout?") and routes back to the onboarding screen.

---

## The Backend API

The API is built with **Express 5**, **TypeScript**, and **Prisma ORM** against **PostgreSQL**. Request validation uses **Joi**. Token verification uses **Firebase Admin SDK** in production mode. Security baseline includes **Helmet** (security headers), **CORS** with an allowlist, **morgan** logging, and **express-rate-limit** (default: 400 requests per 15-minute window).

The API serves at a configurable port (default **3090**) under a versioned prefix (default `/api/v1`). Health checks are available at `GET /healthz` and `GET /readyz`.

### Authentication and Roles

The system defines three roles: `PATIENT`, `CLINICIAN`, and `ADMIN`.

In production (`AUTH_MODE=firebase`), every request must carry a valid Firebase ID token as a bearer token. The middleware verifies the token, looks up or creates the corresponding `User` record by `firebaseUid`, and attaches auth context (userId, email, role, name) to the request.

In development (`AUTH_MODE=dev`), requests can pass `x-user-id` or `x-user-email` headers directly, with optional `x-user-role` and `x-user-name` headers. This bypasses Firebase for local testing.

Route-level authorization uses an `authorize()` middleware that checks the user's role against allowed roles per route group. All clinician routes require `CLINICIAN` or `ADMIN`. Patient consent routes require `PATIENT` or `ADMIN`. The screening seed endpoint requires `ADMIN`.

### Screening Module

The screening module is the most complex part of the API. It manages the full lifecycle of preventive screening:

**Definitions** — a catalog of screening types seeded via `POST /internal/screenings/seed` (admin-only). Each definition has a `code`, `displayName`, `domain` (CANCER or HEALTH), `defaultIntervalMonths`, `minEligibleAge`, `maxEligibleAge`, `genderEligibility` (ALL, MALE, or FEMALE), and `guidelineVersion`. The seeded catalog contains 10 definitions:

| Code | Domain | Interval | Age Range | Gender |
|------|--------|----------|-----------|--------|
| CERVICAL_CANCER | CANCER | 60 months | 25–75 | FEMALE |
| BREAST_CANCER | CANCER | 24 months | 40–70 | FEMALE |
| BOWEL_CANCER | CANCER | 24 months | 45–70 | ALL |
| PROSTATE_CANCER | CANCER | 24 months | 50+ | MALE |
| LUNG_CANCER | CANCER | 24 months | 50+ | ALL |
| CARDIOVASCULAR_HEALTH | HEALTH | 24 months | 45+ | ALL |
| DIABETES_CHECK | HEALTH | 36 months | 40+ | ALL |
| VISION_CHECK | HEALTH | 24 months | 16+ | ALL |
| DENTAL_CHECK | HEALTH | 6 months | 16+ | ALL |
| MENTAL_HEALTH_CHECK | HEALTH | 24 months | 0+ | ALL |

**Plans** — per-user screening plans linking a user to a definition. Tracks `neverScreened`, `lastScreeningDate`, and `source` (SYSTEM, USER_OVERRIDE, or CLINICIAN_OVERRIDE).

**Due Items** — generated items with concrete `dueDate`, `eligible`, `recommended`, `overdue`, `completed`, and `completedAt` fields. Indexed for efficient queries by user + completion + date.

**Records** — individual screening events with `performedAt`, `outcomeStatus` (NORMAL, ABNORMAL, INCONCLUSIVE, NOT_DONE, PENDING), `resultSummary`, `notes`, `source` (MOBILE_FORM, MOBILE_IMPORT, CLINICIAN, MIGRATION), and optional `enteredByUserId`. Each record can carry:

- **Measurements** — typed values (NUMBER, TEXT, BOOLEAN, DATE, CODED, JSON) with units, reference ranges, abnormal flags, and interpretation text.
- **Flags** — severity-coded alerts (INFO, WARNING, CRITICAL) with codes and messages.
- **Attachments** — links to wallet documents by ID.
- **Domain-specific details**:
  - `CancerScreeningDetail` — cancer type (CERVICAL, BREAST, BOWEL, PROSTATE, LUNG, SKIN, OTHER), test method, specimen info, lab reference, result category, follow-up info.
  - `CardiovascularDetail` — systolic/diastolic BP, heart rate, ECG, full lipid panel.
  - `DiabetesDetail` — fasting/random/post-meal glucose, HbA1c, ketones, BP, weight/height/BMI.
  - `VisionDetail` — acuity per eye, color vision, peripheral vision, pressure, symptom flags (blurred, strain, dry eyes, night vision, double vision).
  - `DentalDetail` — hygiene habits, cavity/filling/missing tooth counts, gum health flags, pain flags, last cleaning/x-ray dates.
  - `MentalHealthDetail` — K10 score/level, DASS-21 sub-scores (depression, anxiety, stress) with levels, sleep metrics, social/occupational factors, mood symptom flags.

**Snapshots** — `CancerScreeningSnapshot` and `HealthScreeningSnapshot` store legacy JSON-shaped calculations per user for compatibility during the migration from Appwrite. These coexist temporarily with canonical records.

**Import** — `POST /me/screenings/history/import` allows batch importing historical screening data, tracked by `ScreeningImportBatch` with attempt/success/error counts and status.

API routes:

- `GET /me/screenings/definitions` — list catalog
- `GET /me/screenings/plans` — list user's plans
- `PUT /me/screenings/plans/:screeningCode` — upsert a plan
- `GET /me/screenings/due-items` — list due items
- `POST /me/screenings/records` — create a record
- `GET /me/screenings/records` — list records
- `GET /me/screenings/records/:recordId` — get single record
- `GET/PUT/DELETE /me/screenings/cancer-snapshot` — snapshot CRUD
- `GET/PUT/DELETE /me/screenings/health-snapshot` — snapshot CRUD
- `POST /me/screenings/history/import` — batch import

### Wallet, Pregnancy, and Feedback Modules

**Wallet** (`/me/wallet/*`) — CRUD for `WalletDocument` records. Documents have `title`, `description`, `documentType`, `fileType` (FILE, IMAGE, LINK), `objectKey` or `publicUrl` or `externalUrl`, `mimeType`, and `sizeBytes`. Indexed by user + creation date.

**Pregnancy** (`/me/pregnancy-plan`) — CRUD for a single `PregnancyPlan` per user. Stores `conceptionDate`, `expectedDueDate`, and `estimatedCheckupDates` as a JSON array of milestone objects.

**Feedback** (`/me/feedback`) — stores `FeedbackEntry` records with `feedback` text and optional `rating`.

### Patient Consent Module

Patient-facing consent routes let users manage consent grants initiated by clinicians:

- `GET /me/consents/requests` — list consent requests filtered by status, paginated.
- `POST /me/consents/:consentId/approve` — approve with customized scope (domains array, access level, include history flag, optional expiry).
- `POST /me/consents/:consentId/decline` — decline with optional reason.
- `POST /me/consents/:consentId/revoke` — revoke active consent.

All four routes are restricted to `PATIENT` or `ADMIN` roles.

Consent scope is stored as a JSON object with the structure:

```
{
  accessLevel: "READ_ONLY" | "READ_WRITE",
  domains: ["screenings", "documents", "pregnancy", "feedback", "profile"],
  includeHistory: true | false,
  note: "optional string"
}
```

The scope normalization logic (`consent.scope.ts`) validates domains against a fixed set, defaults `accessLevel` to `READ_ONLY`, defaults `includeHistory` to `true`, and strips empty domains.

### Clinician Module

All clinician routes are gated to `CLINICIAN` or `ADMIN` roles. The module provides:

**Profile management:**

- `GET /clinician/profile` — read clinician profile (auto-created for admins).
- `PUT /clinician/profile` — update organization, license number, specialty, active status.

**Patient relationships:**

- `GET /clinician/patients` — paginated patient list with search. Admins see all patients; clinicians see only their linked patients with the latest consent status per patient.
- `POST /clinician/patients/link-by-email` — link to a patient by email address.
- `POST /clinician/patients/:patientId/link` — link by patient ID with relationship type.
- `DELETE /clinician/patients/:patientId/link` — soft-unlink (sets `isActive: false`, records `unlinkedAt`).

**Consent management:**

- `POST /clinician/patients/:patientId/consent/request` — create or refresh a consent request. Validates that an active link exists, no active consent already exists (must revoke first), and can update a pending request in-place. Accepts requested scope, expiry date, and message.
- `POST /clinician/patients/:patientId/consent/revoke` — revoke active or pending consent with optional reason.

**Patient timeline:**

- `GET /clinician/patients/:patientId/timeline` — the core clinical read surface. The service layer:
  1. Verifies the clinician has an active link and active consent for the patient.
  2. Checks consent expiry at read time (auto-transitions to EXPIRED if past).
  3. Parses the consent scope to determine which domains are visible.
  4. Respects `includeHistory` — if false, limits records/documents/feedback to 1 item each.
  5. Fetches only consented domains in parallel:
     - `profile` domain → full patient profile, or redacted (no email/phone/dob/firebaseUid) if not consented.
     - `screenings` domain → due items (with definitions), records (with all details, measurements, flags, attachments), and snapshots.
     - `pregnancy` domain → pregnancy plan.
     - `documents` domain → wallet documents.
     - `feedback` domain → feedback entries.
  6. Returns relationship and consent metadata alongside data for the clinician UI to render access state.

### Data Model Overview

The Prisma schema defines the following core models:

- **User** — id, firebaseUid, email, role (PATIENT/CLINICIAN/ADMIN), name, demographics (gender, dob, phone, avatar), soft-delete flag.
- **Organization** — id, name, slug. Clinician profiles optionally belong to an organization.
- **ClinicianProfile** — userId, organizationId, licenseNumber, specialty, isActive.
- **PatientLink** — clinicianId ↔ patientId with relationship type, active flag, linked/unlinked timestamps. Unique on (clinician, patient).
- **ConsentGrant** — patientId ↔ clinicianId with status (REQUESTED/ACTIVE/DECLINED/REVOKED/EXPIRED), scope JSON, requestedScope JSON, request message, response reason, timestamps for each lifecycle stage, optional expiry.
- **ScreeningDefinition** — catalog entry (described above).
- **ScreeningPlan** — user + definition link with plan source and last date.
- **ScreeningDueItem** — computed due date with eligibility/completion flags.
- **ScreeningRecord** — individual event with outcome and source.
- **ScreeningMeasurement** — typed values with reference ranges.
- **ScreeningFlag** — severity-coded alerts on records.
- **ScreeningAttachment** — links records to wallet documents.
- **CancerScreeningDetail / CardiovascularDetail / DiabetesDetail / VisionDetail / DentalDetail / MentalHealthDetail** — domain-specific structured fields on records.
- **ScreeningImportBatch** — migration tracking.
- **CancerScreeningSnapshot / HealthScreeningSnapshot** — legacy compatibility.
- **PregnancyPlan** — conception date, due date, checkup dates JSON.
- **WalletDocument** — document metadata and storage pointers.
- **FeedbackEntry** — user feedback with ratings.
- **AuditLog** — actor, action, resource type/id, status, IP, user agent, structured meta.
- **LegacyAppwriteMap** — maps old Appwrite document IDs to new UUIDs for idempotent migration.

### Audit Logging

Sensitive operations are logged to the `AuditLog` table with the acting user, action name, resource identifiers, success/failure status, IP address, user agent, and a JSON meta field for additional context. Logs are indexed by resource and by actor + timestamp.

### Data Migration from Appwrite

The project originally used **Appwrite** as its backend. Migration scripts in `scripts/` move data to PostgreSQL:

- `migrate-users.ts` — user accounts and profiles.
- `migrate-wallet.ts` — wallet documents.
- `migrate-pregnancy.ts` — pregnancy planner data.
- `migrate-screenings.ts` — seeds definitions, then migrates cancer and health snapshots into canonical records.
- `migrate-feedback.ts` — feedback entries.
- `verify-migration.ts` — validates record counts post-migration.

Each script uses retry logic with configurable attempts and backoff for cloud database reliability (especially Neon-hosted Postgres). The `LegacyAppwriteMap` table ensures idempotent re-runs by tracking which Appwrite documents have already been migrated.

---

## The Clinician Web Portal

The clinician portal is built with **Next.js** (App Router), **TypeScript**, **TanStack Query** for server-state, **Zustand** for session and local UI state, **Zod** for API response validation, **Tailwind CSS** + **shadcn/ui** for component primitives, and **Sonner** for toast notifications. Authentication uses the **Firebase client SDK** (email/password + Google sign-in).

The app is role-gated: only users with `CLINICIAN` or `ADMIN` Firebase custom claims can access the authenticated shell. A session cookie is written at sign-in to support Next.js middleware route guards. Development mode supports a fallback role via `NEXT_PUBLIC_DEV_USER_ROLE`. In production, role claims must be issued in Firebase and enforced by the API.

Route structure:

| Route | Purpose |
|-------|---------|
| `/auth/sign-in` | Firebase sign-in (email/password + Google) |
| `/app` | Clinical operations dashboard |
| `/app/profile` | Clinician profile management |
| `/app/patients` | Patient directory, linking, and consent management |
| `/app/patients/[patientId]` | Patient detail and timeline view |
| `/app/patients/[patientId]/timeline` | Alias for patient detail page |

### Clinical Operations Dashboard

The dashboard at `/app` is the landing page after sign-in. It loads the clinician's profile and full patient list in parallel and displays four summary cards:

- **Clinician** — the clinician's name and active/inactive badge.
- **Assigned Patients** — total count of linked patients from the API's pagination metadata.
- **Data Access** — a static indicator that audit logging is enabled for compliance.
- **Pending Requests** — count of patients whose latest consent status is `REQUESTED` (awaiting patient action).

Below the summary cards, two main sections appear:

**Pending Consent Requests** — lists up to six patients with pending consent, showing name, email, request date, and a "Review" button that navigates to the patient list. This gives clinicians immediate visibility into who they are waiting on.

**Recent Patient Links** — the five most recently updated patient relationships. Each row shows the patient name and email. If the patient has active consent, a "Timeline" link navigates directly to their timeline. If consent is pending, the row shows "Consent pending" instead.

A bottom action card provides a prominent "Open Patients" button to enter the full patient management workspace.

### Clinician Profile

The profile page at `/app/profile` displays a form with:

- **Email** — read-only, pulled from the authenticated Firebase account.
- **License Number** — editable text field for the clinician's medical license.
- **Specialty** — editable text field (e.g. "Oncology", "General Practice").
- **Organization** — read-only, showing the organization name if the clinician has been assigned to one, otherwise "Not assigned."

The form submits via `PUT /clinician/profile` and shows success/error toasts. The profile is auto-created server-side for admin users on first access.

### Patient Directory, Linking, and Consent Management

The patients page at `/app/patients` is the primary workspace for managing patient relationships and consent. It is titled "Patient Relationship & Consent" and contains several interactive sections:

**Quick Link by Patient Email** — a card at the top with an email input, a relationship type selector (PRIMARY, SPECIALIST, CONSULTING, FOLLOW_UP, OTHER), and a "Link Patient" button. This creates a new patient link by email, validating the email format client-side before calling `POST /clinician/patients/link-by-email`.

**Patient Directory Table** — a paginated, searchable table showing all linked patients (20 per page). Features:

- **Search** — filters by name or email with debounced input.
- **Include inactive toggle** — a switch to show or hide unlinked (inactive) relationships.
- **Table columns**: Patient (name, email, phone), Relationship (badge showing PRIMARY/SPECIALIST/etc. or "Not linked"), Consent (status badge with contextual hint text), Last Updated (date), Actions.
- **Consent status badges** with computed states:
  - **Active** — green badge with expiry countdown ("Expires in 3 days", "No expiry").
  - **Pending Approval** — amber badge, "Awaiting patient consent action."
  - **Declined** — red badge, "Patient declined this consent request."
  - **Revoked** — red badge with revocation date.
  - **Expired** — red badge with expiry date.
  - **No consent** — muted badge, "Patient has not granted consent."
- **Action buttons per row**:
  - **Timeline** — navigates to the patient timeline (only enabled when consent is active; shows "Timeline Locked" otherwise).
  - **Link / Unlink** — link inactive patients with a relationship type dialog, or unlink active patients with a confirmation dialog warning that timeline access will stop.
  - **Request / Revoke/Cancel** — opens the consent request dialog for patients without active/pending consent, or the revoke dialog for those with active or pending consent.

**Consent Request Dialog** — a modal for composing a consent request with:

- **Access level selector** — Read-only or Read + Write.
- **Include Historical Records toggle** — whether the clinician can see past records or only the most recent.
- **Scope Domains** — five toggleable switches, each with a label and description:
  - Screenings: "View screening schedules, due items, and historical screening outcomes."
  - Documents: "Access patient wallet files and shared health documents."
  - Pregnancy Plan: "Access pregnancy milestones, checkups, and estimated due timeline."
  - Feedback: "View patient-submitted ratings and service feedback entries."
  - Patient Profile: "Access demographics and patient identity profile details."
- **Suggested Expiry** — optional calendar date picker for consent expiration.
- **Request Message** — optional textarea for context (e.g. "oncology follow-up review").
- At least one domain must be selected before the request can be sent.

**Revoke / Cancel Request Dialog** — contextually titled "Revoke Consent" for active grants or "Cancel Consent Request" for pending requests. Includes an optional reason textarea and a destructive confirmation button.

**Unlink Patient Dialog** — an alert dialog confirming that unlinking disables the active relationship and stops timeline access.

**Pagination** — Previous/Next buttons with page count display.

### Patient Timeline

The timeline at `/app/patients/[patientId]` is the main clinical read surface. The page header shows the patient's name (or "Patient Timeline" while loading) and a "Back to patients" navigation link.

**Patient Summary Card** — a two-column grid showing the patient's full name, gender, email, phone, date of birth, and "Patient Since" date. Fields that fall under the `profile` consent domain (email, phone, DOB) display "Not shared by patient consent" when the profile domain is not in the consent scope, rather than showing empty or null — making the consent boundary visible to the clinician.

**Access State Card** — displayed alongside the patient summary, showing:

- **Relationship** — badge with the relationship type (PRIMARY, SPECIALIST, etc.) or INACTIVE.
- **Consent** — status badge (Active, Expired, Revoked, Pending approval, No consent) with color coding.
- **Access Level** — READ_ONLY or READ_WRITE from the consent scope.
- **Include History** — Yes/No from the consent scope.
- **Consent Expires** — formatted date or "N/A".
- **Granted At** — timestamp of when consent became active.

**Consent Domain Coverage** — a five-column grid showing each consent domain (Profile, Screenings, Pregnancy, Documents, Feedback) with a "Shared" or "Not shared" badge. This gives the clinician an immediate visual map of what they can and cannot see.

**Screening Due Items** — a list of upcoming and overdue screening events (when the `screenings` domain is consented). Each item shows the screening definition name, a status badge (Overdue in red, Completed, or Upcoming), due date, and metadata (eligible, recommended, interval in months). If the screenings domain is not shared, a `ConsentRestrictedState` component renders an amber dashed-border panel stating "Not shared in consent scope" with an explanation.

**Screening Records** — historical screening entries (when `screenings` domain is consented). Each record shows the screening name, outcome status badge (ABNORMAL in red, others as outline), performed date, result summary, notes, and:

- **Measurements** — each measurement displays its name, value (rendered by type: number, text, boolean, date, coded, or JSON), unit, reference range, and an abnormal flag badge when present.
- **Flags** — severity-coded entries (INFO, WARNING, CRITICAL) with their message text.
- **Domain-specific details** — expandable sections for cancer details (cancer type, test method, specimen info, lab reference, result category, follow-up info), cardiovascular details (BP, HR, ECG, lipids), diabetes details (glucose types, HbA1c, BMI), vision details (acuity, pressure, symptoms), dental details (oral health indicators), and mental health details (K10/DASS scores, sleep, mood flags). Each detail type renders its fields in a labeled grid, skipping null/empty values.
- **Attachments** — listed with file name, mime type, and size.

**Pregnancy Plan** — (when `pregnancy` domain is consented) shows conception date, expected due date, and the estimated checkup dates parsed from JSON with count indicators. If the pregnancy domain is not shared, the consent-restricted panel appears.

**Wallet Documents** — (when `documents` domain is consented) a list showing each document's title, type, file type badge (FILE/IMAGE/LINK), size, and creation date. If not shared, the consent-restricted panel appears.

**Feedback** — (when `feedback` domain is consented) shows feedback text and optional rating for each entry. If not shared, the consent-restricted panel appears.

**Cancer and Health Snapshots** — legacy compatibility data rendered as JSON item counts with expandable raw views, shown at the bottom of the timeline when present in the response.

**Error handling** — if the timeline API returns a 403, the page shows "Access denied. This usually means the patient link or active consent is missing." If it returns a 404, "Patient timeline was not found." Other errors show the API message. This makes access failures diagnosable without exposing internal details.

---

## The Marketing Website

A public Next.js site with the following sections:

- **Hero** — "Stay on top of your health with CheckUpp" headline, subheadline about personalized screenings and secure storage, and App Store / Play Store download buttons.
- **Stats** — three trust signals: Evidence-Based (built on national healthcare guidelines), Personalized Care (adapts to age, gender, and health profile), Secure & Private (data never shared without consent).
- **Benefits** — three expandable sections: Guided Onboarding, Smart Reminders, and Doc Wallet, each with bullet points and mockup images.
- **Testimonials** — user quotes about screening scheduling and pregnancy planning.
- **FAQ** — four questions: what makes CheckUpp different, is data secure, who should use it, and does it work for pregnancy.
- **CTA** — repeated call-to-action for app download.
- **Footer** — contact at <support@checkupp.com>, legal links.

---

## Security and Privacy

- **Authentication** — Firebase Auth with support for email/password and Google OAuth. API verifies Firebase ID tokens server-side via Firebase Admin SDK.
- **Authorization** — three-tier role model (PATIENT, CLINICIAN, ADMIN) enforced at the route level. Clinician data access additionally requires an active patient link AND active consent grant.
- **Consent scoping** — data shared with clinicians is filtered per domain (screenings, documents, pregnancy, feedback, profile). History inclusion is controllable. Consent can expire automatically.
- **Transport security** — HTTPS enforced in production; API uses Helmet for security headers.
- **Rate limiting** — 400 requests per 15-minute window per IP by default.
- **CORS** — allowlisted origins only (defaults to localhost:3000 and localhost:8081 in development).
- **Audit logging** — sensitive reads and writes are recorded with actor, action, resource, status, IP, and user agent.
- **Secure mobile storage** — uses `expo-secure-store` for sensitive tokens and credentials on the device.
- **Crash reporting** — Firebase Crashlytics is integrated on mobile for stability monitoring.

---

## Future Direction

A detailed expansion plan exists in `docs/clinician-and-admin-expansion-plan.md` covering:

- **Clinician workflow depth** — clinical task management, structured clinical notes (SOAP format with addenda and version history), alert and triage queues for abnormal results or missed follow-ups, patient cohort tools with saved filters and bulk actions, document verification workflows, and clinical summary PDF exports.
- **Admin portal** — organization management, user and workforce management, RBAC with permission matrices and role templates, consent policy templates and expiry rules, searchable audit logs with PHI access reports, data retention and legal hold tools, operational monitoring dashboards, and time-bound impersonation with full audit trail.
- **Platform maturity** — MFA for clinician and admin roles, device session management, interoperability roadmap (FHIR/HL7), and comprehensive E2E and load testing.

The plan is phased so each stage delivers independent value while building toward a production-grade clinical operations platform.
