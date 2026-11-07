# Implementation and verification record

Date: 1 November 2026.

## Delivered

This update implements the pending code-level security and workflow work identified in the first EternaPlaan review:

- Central company/role/object checks, foreign-record checks and separate linked-record customer/vendor portals.
- Transactional owner signup, hashed/expiring verification codes, invitation onboarding, rotating hashed refresh tokens and immediate session revocation.
- TOTP authenticator enrollment/sign-in with replay prevention and an audited operator recovery command.
- Transactional venue/vendor reservation locking, overlap checks, cancellation/restoration checks, reservation synchronization and collision-resistant codes.
- API-backed customer, event, task, booking, guest/RSVP, invoice, expense, payment, report and settings workflows, replacing the reachable sample screens.
- Stripe-hosted checkout, raw-body signed webhook verification, operation/event idempotency, exact currency arithmetic, a receipt/refund ledger, partial refunds and reconciliation.
- Private encrypted document upload/list/download, type/size checks, production malware-scan enforcement, download audit records and removal of public upload serving.
- Encrypted durable email jobs with leases/retries/failure states, a real SMS adapter, and queue-health checks.
- Bounded/indexed AI discovery, explicit helpful-match feedback and shared chat quotas, preserving optional conversational AI and labeled local fallback.
- Updated dependencies/lockfiles, reproducible containers, migration checksums, a CI workflow, local/production Compose configurations, HTTPS configuration, backup/restore scripts and setup/runbooks.

## Validation results

| Validation | Result and scope |
|---|---|
| Backend fast suite | **11 passed**, including matching, request validation, quotas/routing, AI provider contract/fallback, exact currency conversion, AES-GCM tamper detection, interval validation and the TOTP reference vector |
| Real MySQL suite | **14 passed on MySQL 8.0.46**, with fresh schema import and migration re-run, two-company isolation, nested resource checks, simultaneous conflicting/adjacent reservations, supplier cancellation/restoration, manual receipt idempotency, exact invoice/refund balances, Stripe signature verification/deduplication, private upload encryption/isolation, portal roles, logout/reset revocation, shared quotas, signup/OTP/refresh rotation, invitation single-use and MFA enrollment/replay checks |
| Browser journeys | Passed with a real Chromium browser and the running MySQL/API/Vite application: sign-in, live dashboard, customer save, task save/completion, discovery, feedback, local chat, private upload, payment history and reports |
| Mobile/keyboard | Verified at 390×844: no horizontal page overflow, closed navigation is hidden, chat fits the viewport and Escape closes it. Screenshots were visually inspected; a navigation overlay issue was repaired |
| Frontend production build | Passed using Vite 8; generated assets successfully |
| Production dependency audits | `npm audit --omit=dev` reports **0 known vulnerabilities** for both resolved trees at the time of this update. This excludes development-only packages and does not guarantee absence of undisclosed vulnerabilities |
| Configuration | Local and production Compose files passed Compose 2.40.3 configuration validation; Caddy configurations parsed successfully using Caddy 2.6.2; backup/restore scripts passed shell syntax checks |
| Provider execution | Payment/refund behavior used mocked provider API objects with a real database and the actual Stripe signature implementation. No live charge/refund, actual SMTP/Twilio delivery or real conversational-AI request was made |
| Infrastructure execution | Docker image builds/services, public certificate issuance, malware scanner execution and offsite recovery were not executed on a production host |

The above are focused regression checks, not a full security certification, load test or comprehensive accessibility audit. Configuration parsing does not prove that containers will run or a domain will receive a valid certificate. Read the live verification gates below before launch.

## Still needs your infrastructure or business data

1. Configure your server/domain/secrets and verify Docker deployment, HTTPS issuance/renewal and monitoring alerts.
2. Configure SMTP/Twilio where used; verify actual delivery, retry/failure behavior and queue monitoring.
3. Configure an eligible Stripe account, webhook signing secret and supported currency. Verify test and authorized live payments/refunds, provider settlement and operational reconciliation.
4. Verify ClamAV signature updates and clean/test-malware upload behavior on the deployed host.
5. Configure offsite restic storage, preserve the encryption key and complete an isolated restore drill.
6. Configure a supported OpenAI model/key and evaluate live chat accuracy, safety, latency and cost.
7. Collect consented interaction/outcome data before training/evaluating a recommendation model. This release provides explainable content-based matching and feedback collection.

Additional product boundaries remain explicit: no automated bank settlement/dispute accounting, no S3 storage or multi-host file coordination, no full customer self-service checkout portal, no scheduled SMS campaign UI, and no self-service MFA recovery codes. These are not represented as completed features.

## Upgrade and review

Start with [README.md](README.md), then read [PROJECT_REVIEW_AND_AI_GUIDE.md](PROJECT_REVIEW_AND_AI_GUIDE.md). Back up before migrating. Migration `002_hardening.sql` preserves opening invoice balances, adds the new security/workflow tables and invalidates existing sessions. It does not sanitize existing business data or revoke old public document URLs automatically.

Production audit snapshots and concise test/build records are included in `review/`. The browser screenshots contain synthetic test data and were used only for internal layout inspection; they are not deliverables.


## Live deployment tooling follow-up

Added `LIVE_DEPLOYMENT_SETUP.md`, a deployment/preflight runner, provider-specific opt-in probes and an isolated restic recovery drill. The fast suite now has 15 passing checks; the real MySQL suite has 15 passing checks. This follow-up does not claim a Docker deployment, public certificate, delivered message, real provider payment/refund, offsite recovery or external AI call occurred. The current live report is explicitly incomplete.
