# Serein storefront

A responsive React and Tailwind storefront with an AI shopping assistant, MongoDB-backed OTP login, secure sessions, Razorpay payments, cash-on-delivery checkout, and order-email handoff.

## Run locally

```bash
npm install
npm run dev
```

`npm run dev` now starts the full local application: Vite plus the API routes, with no Vercel login required. Create `.env.local` first. Use `npm run dev:frontend` only when you intentionally want a frontend-only preview with no login, payment, or API functionality. `npm run dev:vercel` remains available when you specifically need to test the Vercel runtime.

If `.env.local` does not yet contain Resend credentials, the local server generates temporary secrets and uses a development OTP fallback. The six-digit code is shown in the sign-in form and printed in the terminal. This fallback is disabled in production; production always requires Resend and a verified sender domain. Placeholder values copied from `.env.example` are treated as unconfigured, so local login keeps working until real Resend keys are added.

If `ADMIN_EMAIL` and `ADMIN_PASSWORD` are missing or still placeholders, local admin login uses `admin@localhost.test` / `serein-local-admin`. Those credentials are shown in the admin login form and printed in the terminal. Production never uses this fallback.

## Local setup

MongoDB is expected at `mongodb://127.0.0.1:27017` by default. Copy `.env.example` to `.env.local`, generate the two random secrets, and configure your verified Resend domain and Razorpay test keys. The Vercel-compatible endpoints are in `api/`.

- `api/auth/request-otp.js` and `api/auth/verify-otp.js` issue single-use email codes and create server-side HTTP-only sessions. OTPs expire after ten minutes and are rate-limited.
- `api/products.js` seeds the starter catalog into MongoDB and requires an admin session to create a product.
- `api/orders.js` creates server-repriced cash-on-delivery orders.
- `api/payments/create-order.js` creates Razorpay orders; `api/payments/verify.js` verifies the returned signature before confirming an order.
- `api/payments/webhook.js` verifies Razorpay webhook signatures and records captured or failed payment status.
- `api/seller-applications.js` accepts authenticated seller applications and encrypts KYC, tax, address, payout, and document data before it reaches MongoDB.

## Seller onboarding

“Sell with Serein” in the footer opens the marketplace seller application. Applicants verify their email first, then provide business identity, GSTIN, KYC and registration proofs, bank/IFSC details with bank proof, a starter catalog, and fulfillment rules. Documents are limited to PDF, JPG, PNG, or WebP at 750 KB each so they fit the serverless request limit. The `SELLER_DATA_ENCRYPTION_KEY` must be a unique 32-byte base64url key before seller applications can be accepted.

## Razorpay configuration

1. Create a Razorpay account and use Test Mode keys while developing.
2. Enable UPI in your Razorpay account. The checkout offers an explicit UPI option first and uses Razorpay's supported UPI Intent or QR flow; it does not collect a customer UPI ID directly.
3. Add `payment.captured` and `payment.failed` webhook events in Razorpay, pointing to `https://your-domain/api/payments/webhook` and using the same `RAZORPAY_WEBHOOK_SECRET` configured on the server.
4. Enable automatic payment capture in Razorpay, test a payment, then replace only the server environment variables with live keys when ready. The browser receives the public key ID only; the key secret is never sent to it.

Razorpay requires a server-created order and server-side signature verification; this implementation follows that flow. See the [Razorpay Standard Checkout integration guide](https://razorpay.com/docs/payments/payment-gateway/web-integration/standard/integration-steps/).

## Deployment notes

Deploy the frontend and `api/` folder together on a serverless host such as Vercel so session cookies remain same-origin. Set every `.env.local` value in the host's encrypted environment-variable settings; do not commit `.env.local`. The generated tracking ID is an internal reference—replace it with a carrier tracking integration before fulfillment.

## Verification

With local MongoDB running, `npm test` runs the full API integration scenario: OTP request and invalid/valid verification, secure session and logout, catalog persistence, cash-on-delivery, UPI order/signature/webhook confirmation, seller application encryption, and admin review. The test stubs Resend and Razorpay, so it never sends a real email or creates a charge. Use `npm run build` to verify the production frontend bundle.
