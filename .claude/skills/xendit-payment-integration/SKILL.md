---
name: xendit-payment-integration
description: Implement or verify Xendit payments for BMC Commerce. Use when working
  on checkout session creation, webhook intake, payment/order status transitions,
  idempotency, reconciliation, or any code that touches Xendit. Enforces
  server-side provider calls, webhook verification via x-callback-token, exact
  PHP amounts, and never trusting client redirects as payment proof.
---

# Xendit payment integration

Governing docs: `docs/PAYMENT_ARCHITECTURE.md` (normative) and
`docs/API_CONTRACTS.md` (wire contracts). **Re-read the current official Xendit
docs before every implementation task** — the API surface changes.

## Non-negotiables
1. Xendit calls happen **only** in Edge Functions with `XENDIT_SECRET_KEY` from
   secrets — never in `bmc_final/lib/`, assets, or app config.
2. Client redirects, client booleans, and app navigation are **never** evidence
   of payment. Only a verified webhook transition to `succeeded`/`paid` counts.
3. No client can directly mutate `orders`, `payments`, or stock (SQL grants
   enforce it; do not weaken them).
4. Totals are computed server-side from DB prices. Ignore any amount the client
   sends.

## Payment Session creation (create-checkout-session)
1. Verify caller JWT (`auth.getUser`). Return 401 otherwise.
2. Re-read product/variant under row lock inside one transaction; validate
   active/variant/quantity/stock including **active reservations**.
3. Compute pricing in exact decimals; convert to the provider's PHP amount
   representation via the single server helper and **validate against a
   test-mode session** (number format has changed before; see
   `docs/DATABASE_SCHEMA.md` §16).
4. Enforce idempotency: unique `orders.idempotency_key`; on a repeat call for
   the same user return the stored `{orderId, checkoutUrl}`.
5. Persist the payment attempt before returning; return only what the app needs.

## Webhook intake (xendit-webhook)
1. Respond 2xx immediately; process asynchronously.
2. Authenticate: constant-time compare of the callback token header with
   `XENDIT_WEBHOOK_VERIFICATION_TOKEN`. Unverified → 401.
3. Associate via `provider_session_id`/`provider_reference_id`; **dedupe** on the
   unique provider event id.
4. Validate amount + currency + reference exactly against the stored `payments`.
5. Transition state with conditional SQL in one transaction; enforce legal
   transitions (`paid` never → `failed`). Duplicates and out-of-order events
   must be no-ops.
6. Reconcile late/delayed events according to Stage 9; log identifiers only.

## Test-mode gate
- Complete the E2E scenario in `docs/TESTING_STRATEGY.md` §5 with a test-mode
  provider account and record the result **before** any live-charge work.
- Never describe a mocked checkout screen as a passed payment integration.

## Diff review
- Confirm no secret strings, no client `paid` writes, and consistent status
  mapping between `DATABASE_SCHEMA.md`, `PAYMENT_ARCHITECTURE.md`, and the tests.
