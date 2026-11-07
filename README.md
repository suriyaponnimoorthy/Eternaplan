# EternaPlaan — Event Planning Workspace

React + Express + MySQL event management with explainable supplier discovery, an optional AI concierge, hosted payments, private documents and separate staff/customer/vendor access.

**Updated: 1 November 2026.** Code-level hardening is implemented. Live infrastructure and external-provider verification remain deployment gates. See [IMPLEMENTATION_AND_VERIFICATION.md](IMPLEMENTATION_AND_VERIFICATION.md).

For the low-cost hosting recommendation and executable live checks, see [LIVE_DEPLOYMENT_SETUP.md](LIVE_DEPLOYMENT_SETUP.md).

## Start with Docker Compose

```bash
node ops/create-env.mjs
# Configure .env: mail provider, public origin and any optional payment/AI keys.
docker compose up --build -d
```

The initial schema and hardening migration run automatically on a fresh database. Sample seed data is **not** imported by default. ClamAV may take several minutes to download signatures; private uploads fail closed until scanning is ready. Start at http://localhost, register a company and verify the code delivered by the email worker.

For production, set `DOMAIN`, `ACME_EMAIL` and provider settings, point DNS at the host, allow ports 80/443 and use:

```bash
docker compose -f docker-compose.yml -f docker-compose.production.yml up --build -d
```

Never commit `.env`. Back up `DATA_ENCRYPTION_KEY` securely outside this project; it protects private files, MFA secrets and queued notifications. Production startup rejects missing/weak secrets, an HTTP frontend URL or disabled malware scanning.

## Existing database upgrade

Back up the database, private files and encryption key first. Pause application writers. Run the new migration with an account allowed to modify the schema; then deploy the backend, frontend and worker together. Do not re-import `schema.sql` over an existing database. The migration invalidates existing sessions, widens legacy code/QR fields, preserves invoice opening paid balances and replaces trigger-driven payment accounting with a ledger.

```bash
cd backend
npm ci --ignore-scripts
# Configure backend/.env; DB_MIGRATION_USER/PASSWORD may name a separate migration account.
npm run migrate
npm start
# In a second process with the same environment:
npm run worker
```

Use Node 22.13 or later in the Node 22 release line. For local source development, configure `backend/.env` from its example, import the initial schema only into a new database, run migrations and start the worker; run `npm ci --ignore-scripts && npm run dev` in `frontend`.

## Main features

- Events, customers/leads, tasks, supplier approvals, bookings, guests, invoices, expenses, reports and settings use real API data with visible errors and empty states.
- Admins manage finances, private documents and team invitations. Event managers plan events and guests. Customer/vendor portal users see only linked event/booking summaries.
- Venue and supplier reservations use transactional resource locks, interval-overlap checks and collision-resistant identifiers.
- Stripe-hosted checkout uses signed webhooks, idempotent requests, exact minor-unit accounting, partial refunds and explicit reconciliation.
- Private PDFs/JPEGs/PNGs are size/signature checked, malware scanned in production, encrypted and downloaded only after authorization.
- MFA enrollment, hashed rotating refresh tokens, session revocation and invitation-based staff onboarding are implemented.
- AI Discover explains each content-based match. The concierge supports local guidance and optional user-enabled conversational AI. Feedback collection is implemented; no trained recommendation model is claimed.

Read [PROJECT_REVIEW_AND_AI_GUIDE.md](PROJECT_REVIEW_AND_AI_GUIDE.md) for API behavior, limitations, configuration and deployment checks.
