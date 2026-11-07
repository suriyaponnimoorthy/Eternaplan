# Quick start

Follow [README.md](README.md) for local setup and [PROJECT_REVIEW_AND_AI_GUIDE.md](PROJECT_REVIEW_AND_AI_GUIDE.md) for production and migration instructions.

1. Generate secrets: `node ops/create-env.mjs`.
2. Configure email before registering; optional Stripe/OpenAI settings can be added later.
3. Run `docker compose up --build -d`.
4. Register the company owner and verify the emailed code.
5. Add customer/vendor records and invite staff or portal users from Settings.
6. Add events, tasks, bookings, guests and invoices. Use AI Discover to shortlist suppliers.
7. Test payments with provider test credentials and signed webhook deliveries before enabling live payments.

Do not use the original sample passwords or manually import sample data into production. The `frontend/demo.html` and `frontend/landing.html` files are historical visual references, not authenticated production applications.
