# BNI Chapter Management & Visitor-to-Member System

A working full-stack implementation of the core chapter workflow: visitor
registration → meeting selection → QR + OTP attendance verification →
feedback → Expression of Interest → payment upload/verification → automatic
Member Master creation — plus chapter accounting, notifications, and an
admin/coordinator/member/visitor role system with a single shared login.

This is a real, running application (Node/Express/TypeScript API backed by a
real SQL database, React/TypeScript frontend), not a mockup. It implements
the highest-value, hardest-to-get-right slice of the full specification end
to end. See **"What's built vs. roadmap"** below for exactly what's covered.

## Quick start (local, no Docker)

**Requirements:** Node.js 20+

```bash
# 1. Backend
cd backend
npm install
npm run seed        # creates roles, a demo chapter, and one login per role
npm run dev          # http://localhost:4000

# 2. Frontend (in a second terminal)
cd frontend
npm install
npm run dev          # http://localhost:5173
```

Open http://localhost:5173 and sign in with any of the seeded demo accounts
(password for all of them is `Password@123`):

| Role        | Email                        |
|-------------|-------------------------------|
| Admin       | admin@bnichapter.test         |
| Coordinator | coordinator@bnichapter.test   |
| Member      | member@bnichapter.test        |
| Visitor     | visitor@bnichapter.test       |

Or register a brand-new visitor from the login screen's "Register as a
visitor" link.

Email is not required for local testing: if `SMTP_HOST` is left blank in
`backend/.env`, OTP and notification emails are printed to the backend's
console instead of sent, so you can copy the OTP straight from the terminal.

## Quick start (Docker)

```bash
docker compose up --build
```
Frontend: http://localhost:8080 · API: http://localhost:4000. Run the seed
script once inside the backend container to get demo logins:
```bash
docker compose exec backend npm run seed
```

## How the core lifecycle works

1. **Visitor registers** (`/register`) — creates a login + visitor profile,
   optionally naming the member who referred them.
2. **Visitor picks an upcoming meeting** from their portal. The backend
   creates a registration and emails a one-time OTP bound to that specific
   registration + meeting.
3. **At the meeting**, the coordinator projects/prints the meeting's QR code
   (Meetings → Generate QR). Scanning it opens `/attend/:qrToken`, a public
   page. The visitor enters their Registration ID and OTP.
4. The backend verifies, **in order**: the QR session is active → the OTP is
   correct, unexpired, and unused → the QR's meeting matches the
   registration's meeting. Only then is attendance recorded — and the OTP is
   immediately marked used, so it can't be replayed.
5. **Post-meeting feedback** — if the visitor says they're interested, an
   Expression of Interest is opened automatically and the coordinator +
   referring member are notified.
6. **Coordinator reviews the EOI** and moves it to "approved for payment."
7. **Visitor uploads payment proof** (screenshot/PDF, method, reference).
8. **Coordinator verifies the payment.** On approval, if the payment purpose
   contains "Membership", the system **automatically creates the Member
   Master record** (with a generated membership number), assigns the MEMBER
   role to the visitor's existing login, opens their ledger, and emails a
   welcome message.

This entire chain was tested end-to-end against the real API (see
`backend/scripts/smoke_test.sh`), including verifying that a wrong OTP is
rejected, the correct OTP is accepted, and a used OTP cannot be replayed.

## Architecture

```
backend/    Node.js + Express + TypeScript API
  src/db/          SQLite connection + schema.sql (dev) — swap-in path to
                    PostgreSQL documented below; postgres_schema.sql is the
                    equivalent production DDL, already generated.
  src/routes/       One file per resource (auth, visitors, members, meetings,
                    attendance, feedback, eoi, payments, accounting, ...)
  src/services/     Cross-cutting logic: RBAC helpers, membership creation
  src/utils/        JWT, password hashing (Argon2id), email, OTP/id
                    generation, file storage abstraction, audit logging
  scripts/seed.ts   Demo data
  scripts/smoke_test.sh   End-to-end lifecycle test against a running server

frontend/   React + TypeScript + Vite + Tailwind v4
  src/pages/        One page per screen
  src/layouts/      Role-aware sidebar app shell
  src/lib/          API client (JWT attach + silent refresh), auth context
```

**Why SQLite instead of Prisma/Postgres in this build:** the sandbox this
was built in blocks the network calls Prisma's engine installer needs, so
the ORM was swapped for a lightweight `better-sqlite3` data layer with hand
-written SQL. All queries live behind `src/db/index.ts`, so moving to
Postgres is a driver swap in that one file, not an application rewrite —
`src/db/postgres_schema.sql` is the ready-to-run equivalent schema. This
mirrors the specification's own requirement that storage/database backends
be swappable via configuration.

### Server & Database Migration (LOCAL → REMOTE/CLOUD, SQLite → Postgres)

1. Provision a PostgreSQL database and run `backend/src/db/postgres_schema.sql`
   against it (`CREATE EXTENSION IF NOT EXISTS pgcrypto;` first, for UUID
   generation).
2. In `backend/src/db/index.ts`, replace the `better-sqlite3` connection with
   a `pg` Pool, keeping the same `db.get/all/run/transaction` interface so no
   route file needs to change.
3. Set `DATABASE_URL` to the Postgres connection string and `SERVER_MODE` to
   `REMOTE` or `CLOUD`.
4. For file storage beyond local disk (S3/GCS/Azure/R2), implement the same
   two functions exported from `src/utils/storage.ts` against that
   provider's SDK — nothing above that layer changes.

## What's built vs. roadmap

**Built and working end-to-end:**
- Auth (JWT + Argon2id) and backend-enforced RBAC for Admin / Coordinator /
  Member / Visitor — the frontend "portal" you land on is a convenience;
  every API route independently re-checks your role.
- Multi-chapter data model (organizations → chapters → users/members/
  visitors/meetings all chapter-scoped).
- Visitor self-registration, admin/coordinator visitor directory.
- Member Master (single source of truth) with self-service profile editing
  and a running ledger.
- Meeting scheduling + QR session generation/regeneration/disable.
- QR + OTP + meeting-date-bound attendance verification, with attempt
  limiting, expiry, and one-time use.
- Post-meeting feedback capture → automatic EOI creation → coordinator EOI
  review workflow.
- Payment proof upload (file validation, size limit) → coordinator verify/
  reject → automatic Member Master creation and welcome email on approval.
- Central receipts, chapter expenses, and cash/bank account balances.
- Notification center and a full audit log (every write records who/what/
  when).
- Admin user management (create/disable/reset-password, assign roles +
  chapters).
- System settings key/value store (`/api/settings`) as the foundation for
  the admin configuration screens.

**Data model already scaffolded (tables + relations exist in
`src/db/schema.sql`) but not yet wired to endpoints/UI** — the natural next
phases:
- Roster/PPT/Badge/Certificate **template-based document generation**
  (`templates`, `template_fields`, `generated_documents` tables are ready;
  needs a rendering engine, e.g. a docx/pptx templating library).
- Birthday/anniversary reminder scheduling (tables ready; needs a cron
  job/worker).
- Deeper reporting/exports (CSV/PDF chapter reports beyond the dashboard
  numbers already implemented).
- SMS/WhatsApp notification channels (email is implemented; the
  `notifications` table is channel-agnostic).
- Scheduled backups.
- Fine-grained permission editing UI on top of the existing
  `roles`/`permissions`/`role_permissions` tables (role-level RBAC is
  enforced today; per-permission UI is not built).

If you tell me which of these to build next, I can pick up exactly where
this leaves off — the schema and patterns are already in place for all of
them.
