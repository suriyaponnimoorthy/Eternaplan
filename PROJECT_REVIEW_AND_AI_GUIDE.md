# EternaPlaan implementation and deployment guide

Updated: 1 November 2026. This guide supersedes the first prototype review and older readiness claims in historical documentation.

## What changed

| Earlier gap | Current implementation |
|---|---|
| Missing company/object checks | Staff middleware authorizes company-owned records, nested tasks/guests and foreign references. Shared supplier/venue catalogs are read-only to other tenants. Non-staff users have a separate linked-record portal. |
| Broad role access | Admins control payments, supplier mutations, private documents, invitations and reports; event managers manage planning. A super-admin role does not bypass company isolation. |
| Staff registration/OTP/session gaps | Company-owner signup is transactional. Other roles join by hashed, expiring, single-use invitations. OTPs/refresh tokens are hashed; logout/reset/MFA changes revoke access tokens using token versions. MFA is TOTP-based with replay prevention. |
| Concurrent/double booking and code collisions | Event and supplier writes share transactional resource locks and interval checks. Adjacent intervals are allowed; overlapping ones are rejected. IDs use cryptographic UUID-derived codes. |
| Unverified manual payment recording | Manual receipts are explicit admin attestations. Stripe checkout, signed webhook confirmation, request idempotency, exact minor-unit ledger entries, partial refunds and reconciliation are implemented separately. |
| Public uploads/bank detail responses | Public `/uploads` serving is removed. New uploads use encrypted private storage, admin-only tenant-scoped downloads and download auditing. Vendor response fields are allowlisted. |
| Mock data/fake success screens | Primary workspace pages now load API data and save through API calls. RSVP only shows success after persistence. The old sample modules are retired. |
| Unreproducible build/dependency advisories | Lockfiles, Node 22 containers, current payment/email/upload/router/build dependencies, dependency overrides, production audits and a CI workflow are included. |
| No deploy/recovery procedure | Compose includes migrations, a worker, ClamAV and a production Caddy HTTPS override. Offsite-backup and isolated-restore scripts plus health/queue checks are supplied. Execution on your host remains a live check. |
| AI scalability/evaluation gaps | Discovery is bounded to 500 filtered candidates per type, uses catalog/range indexes and records explicit helpful-match feedback. AI chat adds shared database-backed per-user and per-company usage quotas. Training still needs real consented outcome data. |

## Access and workflows

Create the company owner through registration. Verification email is placed in the durable encrypted notification queue; the worker must run and SMTP must be configured. Settings lets administrators invite an event manager, customer or vendor. Customer/vendor invites must link an existing record in the same company. Invitees accept a time-limited link and set their own password.

The current customer/vendor portal is intentionally limited to linked event or booking summaries and chat. It is not a full customer self-service booking/payment portal. Staff workspace API calls from those roles return 403. The system uses one stored refresh token per user; a newer sign-in replaces the previous refresh session. MFA recovery requires an operator to verify identity and run `backend/scripts/recover-mfa.js`; there is no self-service recovery-code UI.

Event managers can manage events, tasks and guests but cannot make financial or supplier-administration changes. Admins approve new suppliers before recommendation/booking. Records created without a company are treated as shared catalog entries; introducing an explicit public-catalog approval workflow is recommended before opening a marketplace.

All event start/end dates and times are explicit planning values. Overnight/multi-day bookings must include an end date. Database infrastructure timestamps now use UTC. Audit existing business data and historical timestamp assumptions during upgrade.

An event with active supplier bookings cannot be moved or cancelled until those bookings are cancelled/rescheduled. Events with booking/financial history cannot be deleted. Cancellation/status restoration takes the same resource lock and rechecks conflicts. Date-only recommendations exclude entire days conservatively; their availability is provisional.

## Payments and refunds

The implementation uses **Stripe-hosted Checkout**. Configure `STRIPE_SECRET_KEY` and `STRIPE_WEBHOOK_SECRET` on the backend only. Your provider account must support your business, country and INR payments. No live account or payment credentials are included.

Configure the endpoint:

```text
https://YOUR_DOMAIN/api/v1/billing/webhook
```

Subscribe to `checkout.session.completed`, `checkout.session.async_payment_succeeded`, `checkout.session.expired`, `refund.created`, `refund.updated` and `refund.failed`. Configure a webhook API version compatible with the installed Stripe SDK. Confirmation uses the unmodified request body and Stripe signature verification; relevant objects are retrieved from the provider before reconciliation. A success-page redirect does not mark an invoice paid.

In Payments, select an invoice to create hosted checkout. The backend calculates the remaining amount and reserves one active checkout per invoice. Client-generated request keys and persisted local IDs prevent duplicate creation on retry. Manual payments are blocked while an online checkout is pending. Refund requests reserve the remaining refundable amount and use their own idempotency keys. Only succeeded refunds reduce invoice paid balances; pending/failed states remain visible.

Use **Reconcile** to retrieve the session and refund state. An order without a recorded provider session ID must be investigated using its local order ID. Retry the same request key within 23 hours to recover a lost create response; after that the code fails closed rather than risking a second operation after the provider's idempotency retention window. Investigate ambiguous refund outcomes likewise. Refund reconciliation reads up to 100 refunds per order and explicitly reports when more require operator review.

The initial migration stores existing invoice `amount_paid` as an opening balance. New receipts/refunds use an append-only ledger and update the legacy payment history for reports. Verify opening balances against actual bank/provider records before launch. Settlement fees, payout-to-bank reconciliation, chargeback/dispute automation, tax compliance and accounting-system integration are not implemented; use provider reports and an accountant for these processes.

References: [Checkout](https://docs.stripe.com/api/checkout/sessions/create), [refunds](https://docs.stripe.com/api/refunds/create), [webhook signatures](https://docs.stripe.com/webhooks/signature). No live money movement was performed during this update.

## Private files

Settings supports PDF, PNG and JPEG uploads up to 5 MB. The backend validates the file signature, scans via ClamAV when scanning is required, encrypts with AES-256-GCM and stores ciphertext in `PRIVATE_UPLOAD_DIR`. Downloads require an authenticated administrator in the same company and return attachments with no-store headers. The original bank/document fields are excluded from supplier detail responses.

This implementation uses a private filesystem volume, not S3. Protect/backup that volume and the database together; use a dedicated storage service if operating across multiple hosts. New files are protected, but existing externally public URLs or files in the original `uploads` directory are not automatically migrated/revoked. Audit them, copy appropriate files through the private workflow and revoke old public access before using real KYC documents. Content-type checks do not replace malware scanning. The malware scanner and signature updates were not exercised in this environment.

`DATA_ENCRYPTION_KEY` is also used for encrypted notification payloads and MFA secrets. Keep it in a secret manager and offline recovery plan. Key rotation needs a controlled re-encryption migration; blindly replacing it makes existing protected data unreadable.

## AI features

AI Discover ranks up to 500 filtered venues and approved vendors per type, then returns up to six of each. City, category, guest capacity, budget and date constraints filter candidates. Ratings adjusted for review evidence and capacity fit contribute explainable fit points. Unknown prices/reviews are labeled. This is content-based matching, not a trained collaborative model or a guarantee of final availability/price.

The chat panel supports local guidance by default and appears in both the staff workspace and linked-record portal. Successful Discover filters personalize the current chat shortlist. To enable external conversational AI, set `OPENAI_API_KEY` and an explicitly available Responses-compatible `OPENAI_MODEL`, restart the backend, and opt in using the checkbox in chat. No secret belongs in a `VITE_*` variable.

Portal chat and recommendations use the shared catalog. Staff can also use their company catalog. The provider receives a limited catalog snapshot, the question and at most ten recent messages. The assistant has no tools for bookings, payments, messages or database mutations. Replies render as text. Provider calls have bounded output, timeouts and labeled local fallback. Requests specify `store:false`; this is not a zero-retention guarantee. Users must avoid entering personal/payment data in free text. Real-provider quality, safety and latency still need evaluation with your account.

Shared MySQL quotas limit chat to 20 requests/user/minute and, by default, 500/company/day (`AI_DAILY_REQUEST_LIMIT`). The existing local process limiter also covers assistant routes. Helpful-match feedback is stored only when the user clicks the feedback action. Training, offline ranking evaluation and conversion/satisfaction evaluation require real consented data; no trained model is included.

## Deployment

Run `node ops/create-env.mjs` once from the project root. It refuses to overwrite an existing `.env`. Configure mail credentials before registration. Production additionally needs a domain, DNS, `ACME_EMAIL`, HTTPS port access and a protected host. The production Compose override enables Caddy-managed TLS and persistent certificate volumes. The database has no published port. ClamAV needs internet access to update signatures.

Fresh startup:

```bash
docker compose up --build -d
docker compose logs -f migrate backend worker clamav
```

Production startup:

```bash
docker compose -f docker-compose.yml -f docker-compose.production.yml up --build -d
```

Local startup uses HTTP for development. Production startup rejects HTTP frontend origins, missing/weak secrets or disabled scanning. Set frontend/backend origin configuration consistently so refresh cookies work. If changing reverse-proxy topology, review `TRUST_PROXY_HOPS` rather than trusting arbitrary forwarded client addresses.

For a manual source deployment, import the original schema into a **new** database, configure `backend/.env`, run `npm run migrate`, start `npm start` and `npm run worker` in separate processes, then build/serve the frontend. Existing databases must only run migrations after backup, not re-import the schema. Migrations need DDL rights; use separate migration credentials and restrict the normal runtime user to necessary data permissions after setup. The supplied first-time Compose database user initially has schema-wide privileges.

The migration runner records file checksums and completed steps to support re-running after partial DDL execution. It tolerates duplicate-field/index/table errors for interrupted steps; operators must verify the shape of pre-existing manually created objects. Do not edit a migration once deployed. No automated downgrade is supplied: restore the matched database/files/key backup and redeploy the previous code if rollback is required.

## Mail, SMS, monitoring and recovery

The worker claims durable jobs with leases, retries with a delay and marks jobs failed after five attempts. Payloads are encrypted; delivery is at-least-once, so a crash after provider acceptance may deliver an email twice. The implemented account workflows enqueue emails. SMS delivery has a real Twilio adapter that fails when unconfigured, but no new SMS reminder campaign UI or scheduling flow is claimed.

Monitor:

```bash
HEALTHCHECK_URL=https://YOUR_DOMAIN node ops/check-health.mjs
# From backend, with the deployment environment:
node scripts/check-queue.js
```

`/health` is process liveness and `/ready` verifies database connectivity. These do not certify external providers or restore readiness. Send health failures, worker failure logs, failed/stale queue counts, disk alerts and database alerts to your monitoring system. Schedule checks on your own host and verify that alerts actually arrive.

For offsite backups, configure restic with a remote `RESTIC_REPOSITORY` and `RESTIC_PASSWORD_FILE` stored outside the project. Initialize the repository separately. Run `sh ops/backup.sh` from the project root. It briefly pauses writers, dumps the database, captures encrypted private files, resumes services, uploads encrypted snapshots and runs a repository check. No retention pruning is automatic. Ensure no other application instance writes during the snapshot. Secure temporary backup storage and database credentials. Back up the encryption key separately.

Recovery drill:

1. Restore a selected restic snapshot into a private directory.
2. Create an isolated deployment with matching secrets, database engine and application version.
3. Set `RESTORE_SQL`, `RESTORE_DOCUMENTS` and `RESTORE_CONFIRM=isolated-recovery`; run `sh ops/restore.sh` against that isolated Compose project.
4. Verify schema/version, row counts, an authorized document download, invoice balances, sign-in and `/ready`.
5. Record recovery time and recovered-data timestamp before considering offsite recovery verified.

Scripts are provided; offsite upload, restore and actual HTTPS/host deployment were not executed here.

## New/changed API routes

| Route | Purpose/access |
|---|---|
| `POST /api/v1/auth/resend-otp` | Requeue verification without exposing account existence |
| `POST /api/v1/team/invitations` | Admin invitation creation; customer/vendor links checked |
| `POST /api/v1/team/accept` | Public single-use invitation acceptance |
| `GET /api/v1/portal` | Only linked customer events or vendor bookings |
| `GET /api/v1/mfa`, `POST /mfa/setup`, `/mfa/enable`, `/mfa/disable` | Authenticated TOTP lifecycle |
| `GET/POST /api/v1/tasks` | Company-scoped task list/create |
| `GET /api/v1/finance/payments` | Company payment history |
| `POST /api/v1/finance/payments` | Manual receipt; admin, invoice and request key required |
| `POST /api/v1/billing/checkout` | Admin, invoice ID and request key |
| `POST /api/v1/billing/refunds` | Admin, local order ID, positive integer `amount_minor` and request key |
| `POST /api/v1/billing/orders/:id/reconcile` | Admin, company-scoped provider reconciliation |
| `GET /api/v1/billing/orders`, `/billing/refunds` | Admin operation history |
| `POST /api/v1/billing/webhook` | Public route secured by raw-body provider signatures |
| `GET/POST /api/v1/documents`, `GET /documents/:id` | Admin private upload/list/download |
| `GET/PUT /api/v1/settings` | Own profile name changes |
| `POST /api/v1/ai/feedback` | Explicit company-scoped helpful-match feedback |

Online checkout always calculates the invoice balance server-side. Monetary API inputs use nonnegative decimal strings with at most two decimals; refunds use integer minor units. Password creation/reset requires at least 12 characters and at most 72 UTF-8 bytes to avoid bcrypt truncation. API keys and encryption keys remain server-side.

## Remaining launch gates

- Deploy and verify Docker, Caddy certificate issuance/renewal, production secrets and access controls on your host.
- Configure/verify SMTP and any Twilio use; confirm delivery and queue-failure alerting.
- Configure a supported Stripe account, test signed events, payment/refund success/failure and bank settlement in test mode; then perform authorized live verification.
- Verify ClamAV readiness and a clean/known-test-malware upload flow.
- Perform offsite backup/restore drills with your own repository and retained encryption key.
- Evaluate live conversational AI and set provider spend limits.
- Perform broader accessibility, load, penetration and business-acceptance testing. The focused checks in this update do not replace those reviews.
