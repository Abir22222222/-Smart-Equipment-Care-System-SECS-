# Smart Equipment Care System (SECS)

A web-based platform for university laboratories to **report, track, repair and audit equipment faults**. Every fault becomes a ticket with an owner, a status and a public record, and every repair budget goes through a Technician → Higher Authority → Accounts approval chain.

> Course project: **CSE-416 Software Project Design and Development**, Autumn 2026

**Live demo:** https://smart-equipment-care-system.vercel.app

---

## Why SECS?

In most labs, equipment faults are reported by phone or on paper. Tickets get lost, nobody owns the problem, repair money waits on untracked approvals, and unresolved faults go unnoticed. SECS fixes four things:

| Problem | What SECS does |
|---|---|
| No ownership | A ticket is automatically assigned to the technician of that lab room |
| No visibility | Public live tracker with a status for every ticket (reporter e-mail masked) |
| No time limits | SLA priorities (Critical, High, Medium, Low) and a background reminder job |
| No money trail | Budget request → approval → payment chain and a public audit ledger |

## Features

- **Fault reporting** by form or by **scanning an equipment QR sticker** (lab and equipment are pre-filled)
- **QR generator** for equipment tags, including batch sequences (e.g. `PC-01` to `PC-40`)
- **Automatic technician assignment** based on the lab room mapping
- **Ticket workflow state machine** enforced by the REST API (see below)
- **Budget chain:** Technician requests, Higher Authority approves or rejects, Accounts confirms payment
- **Public Transparency Tracker** and **Public Audit Ledger** (payment disputes are published)
- **Role-based portals** for Admin, Higher Authority, Accounts and Technician; reporting for Student, Teacher and Researcher
- **JWT authentication**, registration, forgot/reset password with a 6-digit code
- **E-mail notifications** (Brevo API with SMTP fallback): password reset, staff accounts, ticket resolved, 7-day progress reminder
- **SLA reminder job:** unresolved tickets older than 7 days trigger a status e-mail to the reporter (scanned every 2 hours)
- Admin tools: lab directory, technician reassignment, staff accounts, departments, equipment types, university name

## Ticket workflow

```
Submit ──► Assigned ──► Assess ──┬─► (no budget needed) ──► In Progress ──► Resolved
(student/  (by lab)   (technician)│
 teacher)                         └─► Request budget ──► Higher Authority approves
                                                        ──► Accounts approves payment
                                                        ──► Funds Approved ──► Resolved
```

Exception paths: budget rejected (returned to technician), payment disputed (published to the audit ledger), report cancelled (Rejected by Higher Authority).

Status actions used by `PATCH /api/v1/issues/:id/status`: `confirm_no_budget`, `request_budget`, `approve_budget`, `approve_payment`, `complete_repair`, `dispute_payment`, `cancel_report`.

## Tech stack

| Layer | Technology |
|---|---|
| Front end | React 19, TypeScript 5.7, Vite 8, Tailwind CSS 4 |
| QR | `qrcode` (generation), `html5-qrcode` (camera scanning) |
| Back end | Node.js 22, REST API under `/api/v1`, JWT (HMAC-SHA256) |
| Database | PostgreSQL ([Neon](https://neon.com)) via `pg` |
| E-mail | Brevo API, Nodemailer SMTP fallback |
| Hosting | Vercel (static build + serverless function `api/index.js`) |
| Tests | Node.js scripts (no test framework needed) |

Architecture: a three-tier design (React SPA, Node.js REST API, PostgreSQL). The database has 11 tables: `users`, `labs`, `issues`, `budgets`, `audit_incidents`, `notifications`, `announcements`, `password_resets`, `departments`, `system_settings`, `equipment_types`.

## Project structure

```
├── api/index.js              Vercel serverless entry point
├── server/
│   ├── app.js, server.js     API app and local server (port 5000)
│   ├── routes/               auth, users, labs, issues, budgets, other, neon
│   ├── middleware/           auth (JWT), error handling
│   ├── services/             e-mail service
│   ├── db/                   schema.sql, seeds.sql, init.js, pool.js
│   └── tests/                api, end-to-end audit and QR pipeline tests
├── src/
│   ├── App.tsx               public landing page, tracker, ledger
│   ├── LoginPage.tsx, RegisterPage.tsx
│   ├── DashboardPage.tsx     role-based portals
│   ├── components/           QR generator and scanner
│   ├── services/secsApi.ts   API client
│   └── utils/                QR payload and generator helpers
├── docs/                     REST API spec, architecture, requirements
├── vercel.json               build and routing configuration
└── VERCEL_DEPLOYMENT_GUIDE.md
```

## Getting started

### Prerequisites

- Node.js 22 or later
- npm
- A PostgreSQL database (optional for a quick try-out; see the note below)

### 1. Clone and install

```bash
git clone https://github.com/<your-username>/<your-repo-name>.git
cd <your-repo-name>
npm install
```

### 2. Configure environment variables

Create a file named `.env.local` in the project root:

```env
# Database (Neon, Supabase or any PostgreSQL)
DATABASE_URL=postgres://user:password@host/dbname?sslmode=require

# Authentication
JWT_SECRET=change-this-to-a-long-random-string

# E-mail (optional)
BREVO_API_KEY=
SMTP_HOST=
SMTP_PORT=
SMTP_USER=
SMTP_PASS=
SMTP_FROM_EMAIL=
```

If `DATABASE_URL` is missing or unreachable, the API falls back to an **in-memory store**. This is enough to try the app and run the tests, but data is lost when the server stops.

### 3. Create the database tables

```bash
node server/db/init.js
```

This runs `server/db/schema.sql` against `DATABASE_URL`. Optional sample data is in `server/db/seeds.sql`.

### 4. Run the app

Open two terminals:

```bash
# Terminal 1: API on http://localhost:5000
npm run server

# Terminal 2: front end on http://localhost:8443 (proxies /api to port 5000)
npm run dev
```

### 5. Create the first admin

On first run the landing page shows a first-time setup banner. Use it to create the primary System Admin account (`POST /api/v1/auth/setup-admin`). The Admin then creates the Technician, Higher Authority and Accounts accounts from the dashboard.

## Scripts

| Command | What it does |
|---|---|
| `npm run dev` | Start the Vite dev server |
| `npm run build` | Production build into `dist/` |
| `npm run preview` | Preview the production build |
| `npm run server` | Start the REST API on port 5000 |
| `npm test` | Run all three test scripts |
| `npm run test:qr` | Run only the QR pipeline tests |
| `npm run test:neon` | Check the Neon services |

## Testing

`npm test` runs three scripts and needs no database:

| Script | Checks | Covers |
|---|---|---|
| `server/tests/api.test.js` | 45 | Authentication, labs, full ticket state machine, audit ledger, staff protection, stats, filters, privacy masking, metadata |
| `server/tests/e2e-feature-audit.js` | 14 | CORS, Vercel handler, JWT per role, full lifecycle from report to audit ledger, live stats |
| `server/tests/qr-pipeline.test.js` | 7 | QR deep-link and JSON payload parsing, batch sequence, lab filter, QR-to-ticket creation |

Total: **66 checks, all passing**. TypeScript strict mode compiles cleanly (`npx tsc --noEmit`).

## API overview

Base path: `/api/v1`. The full specification is in [`docs/01-REST-API-Specification.md`](docs/01-REST-API-Specification.md).

| Area | Main endpoints |
|---|---|
| Auth | `POST /auth/login`, `/auth/register`, `/auth/setup-admin`, `/auth/forgot-password`, `/auth/reset-password`, `GET /auth/me` |
| Issues | `GET/POST /issues`, `GET /issues/public`, `PATCH /issues/:id/status`, `/issues/check-sla` |
| Budgets | `GET /budgets`, `PATCH /budgets/:id/approve`, `/budgets/:id/confirm-payment` |
| Audit | `GET/POST /audit/incidents` |
| Labs | `GET/POST /labs`, `PATCH /labs/:id`, `/labs/:id/technician` |
| Users | `GET/POST /users`, `DELETE /users/:id`, `POST /users/:id/reset-password` |
| Other | `/dashboard/stats`, `/notifications`, `/announcements`, `/departments`, `/equipment-types`, `/settings/university` |

## Deployment (Vercel + Neon)

1. Push the repository to GitHub.
2. Create a PostgreSQL database on Neon and run `node server/db/init.js` with its connection string.
3. Import the repository in Vercel. `vercel.json` already sets the build command (`vite build`), the output folder (`dist`) and routes `/api/*` to `api/index.js`.
4. Add the environment variables (`DATABASE_URL`, `JWT_SECRET` and the e-mail settings) in the Vercel project settings.
5. Deploy. Each push to `main` redeploys automatically.

More detail: [`VERCEL_DEPLOYMENT_GUIDE.md`](VERCEL_DEPLOYMENT_GUIDE.md).

## Known limitations

These are known gaps in the current release:

- **Role guards are not applied to routes.** The `requireRole` middleware exists, but most routes (for example the user list and user delete) do not use it yet. Only `/auth/me` checks the token. Do not expose a real deployment until this is fixed.
- **No 2-hour escalation for Critical tickets.** The only SLA automation is the 7-day reminder e-mail to the reporter.
- **No e-mail on ticket assignment.** E-mails are sent for password reset, staff accounts, resolved tickets and 7-day reminders.
- The default `JWT_SECRET` in the code is a development fallback. Always set your own.
- The SLA targets shown on the landing page (Critical 24 h, High 3 days, Medium 7 days, Low 14 days) are display values; only the 7-day reminder is automated.
- Out of scope in this release: real bank or mobile-banking payment, university ERP integration, native mobile apps, automatic sensor-based fault detection.

## Roadmap

- Apply role-based guards to every protected route
- Critical-ticket escalation to Admin and e-mail on assignment
- GitHub Actions CI (type check, build, tests on every pull request)
- File/photo attachments on reports
- SMS and push notifications
- Analytics: technician performance, mean time to repair, expenditure per lab
- University SSO (SAML / OAuth2), PWA for technicians, CSV/PDF export, Bangla UI

## Team

| Name | ID | Role |
|---|---|---|
| Shafiq Mollah | 0432320005101087 | Back end and database: REST API, PostgreSQL schema, JWT authentication, SLA reminder job, automated tests |
| Abir Talukdar | 0432320005101078 | Front end and UI: landing page, login and registration, role-based dashboards, QR components; requirements, system design, QA, report |

## Acknowledgements

The UI prototype and project scaffold were generated with Figma Make and then adapted to the SECS workflow and API. Built with [React](https://react.dev), [Vite](https://vite.dev), [Tailwind CSS](https://tailwindcss.com), [Node.js](https://nodejs.org), [PostgreSQL](https://www.postgresql.org), [Neon](https://neon.com) and [Vercel](https://vercel.com).

## License

No license has been chosen yet. Add a `LICENSE` file (for example MIT) if you want others to reuse the code.
