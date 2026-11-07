# EternaPlaan drawback fixes and verification

Updated 6 November 2026.

| Reported drawback | Fix | Regression coverage |
|---|---|---|
| Mixed-case paths bypassed tenant or role restrictions | Express routes match canonical casing. Protected handlers have route-level company and role checks, and affected SQL queries also enforce company scope. | A foreign vendor remains inaccessible, a shared vendor remains readable, and an event manager cannot create an expense through either path casing. |
| Encoded IDs bypassed tenant checks | The API rejects percent-encoded and ambiguous path segments. Numeric resource routes validate IDs before querying. Query-string encoding remains supported. | A percent-encoded foreign vendor ID returns HTTP 400. |
| A signed-in session could replace a configured authenticator | Authenticator setup and changes require the current password. An existing factor must be disabled with a fresh code before replacement. Pending enrollment expires after 10 minutes and is bound to the session version. Changing the factor revokes old sessions. | Missing or wrong credentials, expired setup, attempted replacement, reused codes and revoked sessions are covered. |
| A booking could be reactivated after its event was cancelled | Booking status changes lock and verify the parent event before changing the reservation. | The cancelled booking stays cancelled, and concurrent supplier reservations still allow one winner. |
| Checkout could become stuck after losing the provider response or browser request key | Administrators can retry, reconcile or expire checkout by order ID. New orders store the original Stripe account and mode, checkout parameters and a stable idempotency key. Another checkout or manual payment is blocked while the outcome remains uncertain. | Lost responses, repeated provider calls with the same payload and key, cross-company access, wrong provider account, incomplete searches, confirmed expiry and verified absence are covered. |
| Manual receipts stored a default bank transfer, the current date and a synthetic reference | The form and API store the supplied method, receipt date, reference and payment type. The internal idempotency reference is separate from the receipt. The server rejects changes to a receipt submitted with the same request key. | A dated UPI receipt retains its reference and payment type without a gateway transaction ID. Invalid dates and changed retry data are rejected. |

## Upgrade an existing installation

Back up the database and deploy the API and frontend together. Run migration `003_security_workflow_fixes.sql` with the existing migration runner. Production Compose runs the migration before the backend starts. Readiness checks require migrations 002 and 003. The migration clears pending authenticator enrollments and revokes existing sessions, so users must sign in again. Existing enrolled authenticators and historical payment records are preserved.

## Recover checkout from the Payments page

1. Find the order in **Online checkout** and select **Retry checkout**. During the first 23 hours, a missing session can be retried with the original Stripe idempotency key and saved request parameters. An existing session is checked against its current provider status.
2. Select **Reconcile** to look for the session by order ID or refresh its provider status. A confirmed payment updates the ledger once.
3. Select **Expire checkout** to close an open link. An invoice stays reserved while an attempted expiration is unconfirmed or a payment is pending. A new order with no session can be released after a complete search on the recorded Stripe account and mode once the request window and buffer have elapsed.
4. For older orders that lack both a session ID and a recorded account, reconciliation can recover a matching Stripe session. If it cannot find one, the invoice stays reserved for an operator to investigate with the original Stripe account and provider logs. The application cannot prove historical absence from an unbound order.

For a new manual receipt, the form requires the payment method and date. The reference is optional. Older API clients that omit the fields continue to work. Their method defaults to `other` and their date to the database's current date.

## Verification

- Backend unit and tooling suite. All 15 checks passed.
- HTTP and workflow integration suite against a disposable MySQL 8.0.46 database. All 19 checks passed. Stripe workflow responses were simulated, and webhook signatures were generated and validated with the Stripe SDK.
- Frontend production build and backend JavaScript syntax checks passed.

The live deployment, domain and HTTPS, real Stripe account, email/SMS delivery, malware scanner, offsite restore and optional external AI still require the target environment. See [LIVE_DEPLOYMENT_SETUP.md](LIVE_DEPLOYMENT_SETUP.md).

Provider reference pages

- [Stripe idempotent requests](https://docs.stripe.com/api/idempotent_requests)
- [Stripe Checkout Session list](https://docs.stripe.com/api/checkout/sessions/list)
- [Stripe Checkout Session expiration](https://docs.stripe.com/api/checkout/sessions/expire)
