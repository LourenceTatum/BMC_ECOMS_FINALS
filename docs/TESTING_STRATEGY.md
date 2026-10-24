# Testing Strategy

Layered, deterministic testing. **Never weaken an assertion to make a suite
green.** Tests that require real accounts (Supabase hosted, Xendit test-mode
cards) are clearly separated from automated tests and gated by documentation.

## 1. The pyramid

1. **Unit tests** (fast, no I/O) — `test/unit/...`
2. **Widget tests** (fast, mocked repositories) — `test/widget/...`
3. **Database/backend tests** (local Supabase stack) — `supabase/tests/...`
   executed via scripts/SQL harness
4. **Integration/E2E** (test-mode provider) — documented manual+scripted run

## 2. Flutter unit tests

Cover at minimum:

- `core/utils/money.dart` — minor-units <-> decimal <-> ₱ formatting
  (2-decimal rounding, large values, zero).
- `core/utils/validators.dart` — email, password (≥8), PH phone.
- Cart math — quantity bounds, stock clamp, subtotal (int cents).
- Product/variant validation — variant inactive/sold-out/product archived.
- Order/payment **status mapping** — every DB status to UI state and back;
  unknown statuses map to a safe fallback.
- `checkout_controller` — idempotency key reuse, error mapping, retry-once
  logic with a stubbed repository.
- Repository behavior with mocked Supabase/HTTP (structured JSON, RLS errors).

## 3. Widget tests

Cover at minimum:

- Registration/login validation and error messages; loading state on submit.
- Product list — loading, error, empty, and populated grids.
- Product detail — variant selection, sold-out disabled, Buy Now loading +
  failure states.
- Cart — quantity steppers, totals, remove.
- Checkout review — totals render; place-order disabled until address/variant
  valid; failure shows message + Retry.
- Payment status — pending (pulse), paid, failed, expired, cancelled screens
  driven by a fake order provider.
- Order history/detail — status chip mapping, item snapshots.
- Session restoration and route guards — unauthenticated → login redirect.

Widget tests use provider overrides + a fake repository; they never hit the
network. Guard: if a test needs a real network call, it belongs in E2E.

## 4. Database/backend tests (local stack)

Local Supabase (Docker) with migrations replayed. Assert with SQL scripts:

- **RLS ownership**: anon can select active products but not drafts; user B
  cannot select user A's orders/addresses/cart/favorites; unauthenticated
  cart insert fails.
- **Unauthorized order access**: direct PostgREST access by user B to order A
  → zero rows; the detail RPC (if adopted) → authorization error, not silence.
- **Invalid input**: negative quantities rejected; quantity > stock rejected.
- **Inventory race**: two concurrent `create-checkout-session`-style RPCs for the
  last unit → exactly one succeeds (row-lock behavior).
- **Duplicate checkout**: same idempotency key twice → one order, same result.
- **Invalid transitions**: `pending_payment → delivered` direct update fails;
  `paid → failed` blocked by conditional guards.
- **Payment/amount mismatch**: webhook payload with wrong amount → no state
  change + alert row.
- **Duplicate and invalid webhooks**: replaying the same event id → no second
  transition; token mismatch → rejected; out-of-order (failure after paid) →
  no regressions.
- **Late webhooks**: capture event arriving after reservation expiry → handled
  per reconciliation rules.

These tests live as SQL here-docs/scripts under `supabase/tests/` and are
documented in `ENVIRONMENT_SETUP.md` (local stack) — executed by an agent or
CI job, not skipped.

## 5. Integration/E2E (test mode; provider-dependent)

Single canonical scenario ("smoke E2E"):

1. Create a test user (Supabase test project, email confirmation disabled or
   verified).
2. Seed one product + variant + address.
3. Flutter app runs in test mode: log in, browse, open product, select variant.
4. Tap Buy Now → pending order created → receive Xendit **test-mode** hosted
   URL (this step confirms the real server flow + channel activation).
5. Complete payment using the provider's documented test-card/wallet scenario
   (this is a **real provider interaction**, not a mocked checkout screen).
6. Provider sends the webhook to the dev function; function verifies token,
   processes, and the DB reflects `paid`.
7. App refreshes and displays the server-confirmed `paid` state.
8. Duplicate the webhook delivery manually → verify no double
   transition/stock.

**Requirement:** a mocked checkout screen is never evidence that the payment
integration passed. Manual steps (Dashboard webhook trigger, test-card values)
are listed explicitly and separated from automated tests.

## 6. What must never happen

- Deleting/weakening a failing assertion just to go green.
- Disabling RLS in a test to "simplify".
- Committing tests that claim a live payment was processed when it wasn't.
- Running E2E against production keys/accounts.

## 7. CI note (future)

When CI is introduced (Phase 8): run format+analyze+unit+widget per PR, and
database tests in a job with Docker. E2E stays gated + documented.
