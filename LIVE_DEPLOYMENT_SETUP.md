# Live deployment setup and verification

Prepared 1 November 2026. Hosting preference: lowest practical recurring cost.

## Selected starting approach

Use one x86 Linux VPS with 8 GB RAM for this initial Docker Compose stack. My recommendation is the Hetzner CX33 in Germany or Finland **when available**: 4 vCPUs, 8 GB RAM and 80 GB storage. Official listed server pricing is €8.49/month plus €0.50/month for Primary IPv4, excluding VAT. This is €8.99/month for server plus IPv4 before tax. Availability is limited; verify the console before purchasing. Do not silently select a more expensive plan if CX33 is unavailable.

This is a low-cost starting recommendation, not a load-tested capacity guarantee. European hosting can add latency for users in India. ClamAV recommends 3 GiB+ for itself, so I recommend 8 GB for the combined database, application, worker and scanner. A single VPS is not highly available.

The server price excludes your domain, offsite backup storage, provider messaging, payment fees and AI usage. No paid server or account has been created during this work.

Sources, checked on 1 November 2026:

- [Hetzner server specification](https://www.hetzner.com/cloud/cost-optimized/)
- [Hetzner official price adjustment](https://docs.hetzner.com/general/infrastructure-and-availability/price-adjustment/)
- [Primary IP pricing](https://docs.hetzner.com/cloud/servers/primary-ips/overview/)
- [ClamAV recommended resources](https://docs.clamav.net/)

## What is implemented, and what still needs access

| Requested item | Prepared implementation | Access/evidence still needed |
|---|---|---|
| Server, domain, secrets, HTTPS | Secret generator, production Compose, Caddy, deployment script and DNS/HTTPS checks | An actual server, chosen domain, DNS pointed to its public IP, secure server access and configured `.env` |
| Actual Docker deployment | `ops/deploy-production.sh` builds/starts the stack; verification checks engine, configuration, containers and migration | Run it on a Docker host; no engine is available in the current workspace |
| Payments, refunds and signed webhooks | Sandbox payment/refund probe, unsigned-event rejection check and genuine-event/local-ledger matching | Supported Stripe account, TEST credentials, public webhook endpoint and a completed app test Checkout; live money tests need explicit transaction details |
| Email/SMS and malware scanning | SMTP TLS/auth check, explicit test message checks and clean/EICAR scanner check | SMTP/Twilio configuration, explicitly selected test recipients, receipt confirmation, a running scanner with signatures |
| Offsite backup/restore | Encrypted restic backup, checksum/file manifest, isolated recovery drill and evidence output | Remote repository credentials, a real snapshot and operator confirmation that storage is outside the app host |
| Conversational AI | Bounded real-provider smoke probe that rejects local fallback as proof of AI | Configured key/model and explicit smoke-test flag, or keep conversational AI disabled |

## Required account inputs

Keep secrets in the server's `.env`/secret store. Do not paste passwords, secret API keys or SSH private keys into chat.

Non-secret details needed to continue remotely are the server provider/IP, operating system, deployment domain and confirmation of how authorized server access is available. If you do not own a domain, choose that before configuring production HTTPS.

On the server, configure:

```dotenv
DOMAIN=events.your-domain.example
ACME_EMAIL=your-operations-address
FRONTEND_URL=https://events.your-domain.example
SMTP_HOST=
SMTP_PORT=587
SMTP_USER=
SMTP_PASS=
EMAIL_FROM=
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_FROM_NUMBER=
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
OPENAI_API_KEY=
OPENAI_MODEL=
```

Generate database/JWT/encryption secrets with `node ops/create-env.mjs`. It refuses to overwrite an existing `.env`. Preserve the encryption key securely outside the VPS. Configure SMTP before registering the first company account. Verify that your Stripe account supports your business/country/currency before relying on it; the current app uses Stripe, not Razorpay.

## Deploy on the selected host

Install supported Docker Engine/Compose, Node 22.13+ and restic as appropriate. Use SSH-key authentication and restrict SSH to authorized addresses. Permit inbound HTTP/HTTPS for certificate issuance and the app. Keep MySQL private. Use the provider's official Docker/Linux instructions for the selected OS; no host has been modified here.

Upload/unpack this project and configure DNS/`.env`. From the project root:

```bash
sh ops/deploy-production.sh
```

It validates Compose without printing expanded secrets, builds/starts services, waits for readiness and runs the live verification tool. Existing databases must be backed up before migrations. Migration 003 revokes existing sessions and clears unfinished authenticator setup. Migration 004 revokes outstanding password-reset links. See [security remediation](SECURITY_REMEDIATION.md) and [checkout recovery](DRAWBACK_FIXES_AND_VERIFICATION.md). Do not treat an unconfigured provider or local fallback as a passing verification.

The script can exit with status 2 after containers start because one or more provider/recovery checks remain blocked or pending. That is an evidence result, not proof the deployment failed to start. Read `deployment-verification.json` and container logs.

## Verify without automatically sending messages

```bash
node ops/verify-deployment.mjs
```

The runner records `passed`, `failed`, `blocked` or `pending`, and writes a report without including credential values. Missing Docker/accounts are reported as blocked. SMTP authentication is distinct from inbox delivery. Optional conversation AI remains blocked until enabled/tested; if you intentionally leave AI or SMS unused, record that product decision separately rather than claiming an external service was verified.

The runner invokes checks inside the deployed backend container so host credentials do not need to be printed or copied into command arguments. A valid domain alone is not enough: compare its DNS IP with your actual server IP.

## Opt in to the specific provider tests

Use only your own deliberately selected test inbox/phone. These flags authorize one test message/request per verification run; repeated runs can send more messages and incur usage charges. Keep these flags off after finishing.

```dotenv
VERIFY_EMAIL_TO=your-test-inbox
VERIFY_SEND_EMAIL=true
VERIFY_SMS_TO=your-own-test-number
VERIFY_SEND_SMS=true
VERIFY_AI=true
VERIFY_STRIPE_TEST=true
```

These flags can be in the root `.env` or the command's environment. `VERIFY_STRIPE_TEST` refuses live-mode keys and uses a sandbox payment method, then requests a full sandbox refund. It is a provider connectivity test, not proof the app's own Checkout flow was completed. Do not use a live key for the sandbox probe. Email acceptance by SMTP remains `pending` until you confirm inbox receipt. SMS is passed only if the provider reports delivered during the bounded polling period; also confirm the intended handset receives it.

The scanner check accepts clean bytes and expects the standard harmless EICAR antivirus test sample to be detected. Separately verify an authenticated admin upload, reject unauthorized access and verify malware rejection through the upload API. Production scanning fails closed.

## Signed webhook and application-ledger proof

1. Configure `https://YOUR_DOMAIN/api/v1/billing/webhook` and the signing secret in Stripe test mode.
2. Subscribe to the Checkout/refund events documented in the main guide.
3. Create a test customer/event/invoice in the app, open hosted Checkout and complete it with the provider's official test method.
4. Confirm the test event is delivered in the provider dashboard; copy the event ID and matching local order UUID into `VERIFY_STRIPE_EVENT_ID` and `VERIFY_ORDER_ID`.
5. Run the verifier. It checks the provider event is test-mode, its session matches the local paid order, the event is recorded and exactly one capture ledger entry exists for the expected amount.
6. Replay the genuine event through the provider and verify no duplicate receipt. Request a partial/full refund in the app, use Reconcile and verify invoice/ledger/provider totals and pending/failure behavior.

No real-money charge or refund is made by the verification tools. Authorized live financial verification must be performed deliberately using specified amounts/accounts after sandbox results are accepted.

## Offsite recovery drill

Configure restic with a remote repository and a password file outside this project; initialize the repository separately. Confirm the repository is actually off the application host. Keep a securely stored copy of the encryption key independent of both application and repository credentials.

```bash
sh ops/backup.sh
# Select the exact snapshot created by this backup; review restic snapshots.
RESTORE_SNAPSHOT=YOUR_HEXADECIMAL_SNAPSHOT_ID \
CONFIRM_OFFSITE_STORAGE=true \
sh ops/recovery-drill.sh
node ops/verify-deployment.mjs
```

The backup pauses writers, dumps SQL, captures encrypted files and creates a row-count/checksum/file-hash manifest before resuming services. Avoid simultaneous backup runs and any writers outside the stopped Compose services.

The drill uses a new `eternaplan-recovery-*` Compose project, restores only into its isolated database/private volume, starts no email worker/public proxy, compares ten table checksums/counts and every recorded encrypted file hash, and decrypts one private document when present. It records `offsite-recovery-evidence.json`; the main verifier accepts successful operator-confirmed evidence for 24 hours. A snapshot with no private documents cannot prove a real document restore; add a non-sensitive test document before the drill if you need that evidence.

Keep restored temporary files private. The drill intentionally retains its isolated volumes for review; clean up that specifically named project afterward without removing production volumes. Never put a production project name into a destructive cleanup command. `restic check` alone does not prove recovery, and a local repository is not offsite storage.

## Current verification result

In this workspace, live checks are **incomplete/blocked** because no target domain/server configuration, Docker engine or provider accounts have been supplied. No service was marked verified using dummy credentials. A sanitized execution-environment report is in `review/live-check-current.json`.

Local checks passed: 15 fast backend tests and 15 real MySQL tests, including the new manifest tamper check. These validate code behavior, not the six account-dependent live operations above.
