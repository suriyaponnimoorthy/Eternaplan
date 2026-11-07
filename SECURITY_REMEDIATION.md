# EternaPlaan security remediation

## Findings fixed

| Finding | Change | Verification |
|---|---|---|
| Password-reset recipient substitution | Verify that the database candidate matches the submitted email after Unicode NFC and case normalization. Send recovery and resend codes only to the stored address. Revoke pending reset tokens during migration 004. | MySQL matched a look-alike under its collation. The fixed API queued no message and created no token for that request. A normal case-insensitive request queued the message to the stored address and completed a password reset. |
| Private supplier exposure through portal AI | Customer and vendor recommendations, chat context and feedback use the shared catalog. Staff retain access to their own company catalog. | A customer could discover a shared supplier and submit feedback, but could not discover or submit feedback for a private supplier. Authorized staff could still discover it. |
| Targeted account lockout | Failed passwords no longer put the account into a global lock. The existing source-IP limits still apply. The API does comparable password verification work for unknown addresses, and writes new sessions only if the session version still matches. | Five incorrect passwords followed by a correct password succeeded for the legitimate account. Existing MFA replay and session revocation checks passed. |
| Credential-bearing URL exposure | Application and bundled Nginx access logs omit URLs and referrers. Missing routes return a generic response. Sensitive RSVP API responses use no-store and no-referrer. The production proxy and SPA use no-referrer. | HTTP tests verified the application logging and error responses. The regression also checked both bundled Nginx log formats. |

## Upgrade and verify

Back up the database and deploy the backend and frontend proxy configuration together. Run migration `004_revoke_pending_resets.sql` through the existing migration runner before serving traffic. Production Compose runs migrations during deployment. Readiness now checks for migration 004.

Users with outstanding password reset links must request a new one. If an older release was deployed, review access to historical proxy logs and any log exports. If invite or RSVP links were exposed there, invalidate or rotate them and reissue them through a trusted channel. Check any additional load balancer, CDN, or log pipeline for URL and referrer collection.

All 15 backend unit and tooling checks and all 23 MySQL 8.0.46 integration checks passed. The frontend production build passed. The security tests used synthetic accounts and inspected queued messages locally. Live SMTP, external AI, and deployed proxy behavior remain environment checks. This work did not assess production logs for a historical breach.
