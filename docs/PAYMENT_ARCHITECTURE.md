# Payment Architecture — Xendit Buy Now (Critical)

Secure, server-authoritative hosted checkout. This document is normative for
Phase 6; `API_CONTRACTS.md` holds the exact wire contracts.

## 0. Mandatory flow

```
Flutter (authenticated) → supabase.functions.invoke('create-checkout-session')
    → Edge Function validates + computes PHP total from DB
    → Xendit Session API (PAYMENT_LINK mode) → hosted checkout URL
    → user pays on Xendit-hosted page (external browser)
    → Xendit webhook (x-callback-token verified) → Edge Function
    → trusted DB update (payment + order state)
    → Flutter reads updated status via authorized Supabase query (poll/refresh)
```

**A redirect back to the app or a client-side "success" is never payment proof.**

## 1. Integration decision (verified against official docs 2026-10-24)

- Use **Xendit Payment Sessions**, `session_type: PAY`, `mode: PAYMENT_LINK`
  (Xendit-hosted checkout), `currency: PHP`, `country: PH`.
- Confirmed: session lifecycle ACTIVE → COMPLETED / EXPIRED / CANCELED with
  webhooks `payment_session.completed` and `payment_session.expired`; payment
  outcomes arrive as capture/failure events (`payment.capture`,
  `payment.failure`). Webhooks are signed with the **`x-callback-token`**
  header — the official verification mechanism and **must** be implemented.
- Confirmed PH channels (examples): GCASH (EWALLET, PHP, PH, min ₱1 /
  max ₱100,000) plus GrabPay, Maya, over-the-counter/QR channels depending on the
  merchant's activated set. The hosted page shows activated methods; the server
  may restrict via `allowed_payment_channels` (final choice in ADR 0004 / Phase 6).
- Amount conventions: Session API PHP examples use integer minor units
  (`amount: 10000` ≈ ₱100.00). The `v3/payment_requests` API uses decimals —
  a different contract. The exact representation must be validated with a
  **test-mode session** in Phase 6 before any live charge
  (`DATABASE_SCHEMA.md` §16). Never hardcode numeric assumptions beforehand.
- If Sessions/PAYMENT_LINK is unavailable for the activated account/market at
  implementation time, adopt the currently supported equivalent, update ADR 0004,
  and keep every security principle here unchanged.

## 2. Checkout workflow (stage by stage)

### Stage 1 — User selects Buy Now (client)

- On product detail: choose an active variant + quantity (1…stock shown).
- Tap Buy Now → `CheckoutController.startBuyNow(...)`:
  - validates locally (variant selected, qty ≥ 1),
  - generates a new `idempotencyKey` (UUID v4),
  - shows loading on the button and disables repeat triggers,
  - calls the repository → edge function.
- The client is **non-authoritative**: prices/stock it displayed can change;
  the server decides.

### Stage 2 — Flutter requests checkout creation

```
POST functions/v1/create-checkout-session
Authorization: Bearer <user JWT>     // functions.invoke sends this automatically
Body: { productVariantId, quantity, shippingAddressId, idempotencyKey }
```

- Server: `auth.getUser(jwt)`; reject unauthenticated calls with 401.
- No Xendit key, no secret, no client-computed amount in this call.

### Stage 3 — Backend validates the order

- Re-reads canonical product/variant rows (with row locks) and confirms:
  `product.status = 'active'`, `variant.is_active`, `quantity >= 1`,
  quantity within stock considering **active reservations**.
- Computes totals from DB only: unit price = `product.price` +
  `variant.price_delta`; subtotal = Σ(unit × qty); shipping fee (MVP: constant
  rule); discount = 0 (MVP); **total is server-computed**. Any client-sent
  amount is ignored/rejected.
- **Idempotency strategy**: `orders.idempotency_key` is unique. If an order
  with this key already exists **and belongs to this user**, return the existing
  `{orderId, checkoutUrl}` — retries/timeouts/duplicate taps never duplicate
  orders or charges. Otherwise proceed to create.

### Stage 4 — Create a pending order (server, one transaction)

1. Insert `orders`: `status='pending_payment'`, `payment_status='pending'`,
   totals from Stage 3, `expires_at = now() + expiryWindow` (default 1h,
   reviewed in Phase 6 against Xendit session expiry), `idempotency_key`.
2. Insert `order_items` with **immutable snapshots** (name, SKU, unit price,
   qty, line total).
3. Insert `inventory_reservations` `(status='active', reserved_until =
   expires_at)` guarded by a conditional per-variant update
   (affected rows = 0 → rollback). No one can buy the same final item.
4. Commit order state, then create the Xendit Session (Stage 5). If Xendit
   fails, the order remains `pending_payment` with `checkout_url = null`; a retry
   with the same key reuses the order and only retries session creation.

### Stage 5 — Create the Xendit Session (server only)

- `POST https://api.xendit.co/sessions` with HTTP Basic auth; credential from
  `Deno.env.get('XENDIT_SECRET_KEY')` — never from the client.
- Body (per official create-session contract):
  - `reference_id`: our `order.id` (or `order_number`) — the reconciler key.
  - `session_type: "PAY"`, `mode: "PAYMENT_LINK"`, `currency: "PHP"`,
    `country: "PH"`.
  - `amount`: server-converted minor units of `order.total`
    (`DATABASE_SCHEMA.md` §16). Amount equality with the stored order total is
    validated by tests.
  - `customer`: `{ reference_id: <user id>, type: INDIVIDUAL, email: <user
    email> }`.
  - `items`: one per order line (`name`, `net_unit_amount`, `quantity`, `type:
    PHYSICAL_PRODUCT`, `category: CLOTHES`, `currency: PHP`, `reference_id:
    <item id>`).
  - `success_return_url` / `cancel_return_url`: public HTTPS URLs that only
    land the user back — **never trusted as success**.
- Persist on `payments`: `provider_session_id`, `provider_reference_id`,
  `checkout_url`, `amount`. (The exact response field names are confirmed during
  Phase 6 test-mode calls.) Return **only** `{ orderId, checkoutUrl, expiresAt }`.

### Stage 6 — Open checkout (client)

- The app shows a "handoff" screen, then opens `checkoutUrl` in the external
  browser (`url_launcher` with `launchMode: externalApplication`) — payment
  credentials never enter Flutter-controlled web views.
- Returning from the browser resumes the app; the app does **not** interpret the
  return URL as a status. It shows the payment-status screen ("Confirming your
  payment…") and polls/refreshes the authorized order query until the server
  state is final (`paid`, `failed`, `expired`, `cancelled`, or still `pending`).
- App closure mid-checkout: on next launch, a deep link
  (`bmcapp://payment/<orderId>`) or the last-known order screen resumes the same
  status screen.

### Stage 7 — Verify actual payment completion (server webhook)

`xendit-webhook` edge function:

1. **Ack fast:** respond 2xx immediately (Xendit retries up to 6× on non-2xx);
   process asynchronously so the response is never delayed.
2. **Authenticate:** constant-time compare of the `x-callback-token` header with
   `XENDIT_WEBHOOK_VERIFICATION_TOKEN` (server secret). Reject mismatches 401.
   This endpoint is unreachable at the user-session level on purpose — the
   token is its authentication.
3. **Identify:** parse the event; locate the `payments` row via
   `provider_session_id` / `provider_reference_id`.
4. **Dedupe:** events already applied (`provider_event_id` unique) return 200
   with no state change.
5. **Validate content:** amount matches `payments.amount` exactly; currency PHP;
   reference matches the order. Mismatch → log + alert, no state change, 200.
6. **Transitions** (single transaction, conditional updates):
   - capture / session completed + amount match →
     `payments.status='succeeded'`, `orders.payment_status='paid'`,
     `orders.status='paid'` (only while `status='pending_payment'`), `paid_at`,
     reservation → `fulfilled`, `stock_qty` decremented by the reserved qty
     (conditional; 0 rows updated → another winner already processed → safe no-op).
   - failure event → `payments.status='failed'`, order stays `pending_payment`
     with `payment_status='failed'`, reservation → `released`.
   - expiry / cancel events → analogous; release reservations.
7. **Enforce legal transitions:** `paid` never flips to `failed` and vice versa;
   conditional `where status='pending_payment'` guards make out-of-order events
   converge.

### Stage 8 — Update the application

- After webhook processing, order/payment rows are the truth. The app polls
  (while the status screen is open, e.g. every 5s) or pull-to-refreshes
  authorized Supabase queries; realtime on own orders may be added in Phase 7.
- The confirmation screen is shown **only** when the server state is `paid`.
  There is exactly one UI rule: navigation ≠ confirmation.

### Stage 9 — Unsuccessful and unusual outcomes

| Scenario | UI | Database | Inventory | Retry/recovery |
|---|---|---|---|---|
| Payment failed | status: failed; offer retry | payments `failed`, order stays `pending_payment` | reservation released | Retry creates a **new** session with a new key or re-uses the same order for a new attempt |
| Session expired | status: expired; explain window | payments `expired`; order `pending_payment` → `cancelled` via job | released | New order required (stock recheck) |
| User cancelled on hosted page | status: cancelled | payments `cancelled` | released | user may re-initiate |
| Network timeout on create | error state; auto-retry once, same key | deduped via unique key | none created yet | retry with same key is safe |
| Duplicate Buy Now taps | button disabled during call | one order only | one reservation | dedupe returns same order |
| Duplicate webhook deliveries | unchanged | no-op (`provider_event_id` dedupe) | unchanged | idempotent |
| Delayed webhook | pending until processed, then reconciles | late capture processed if order still pending and stock allows; else refund path (Phase 7) | released/fulfilled per outcome | reconciliation loop + alert |
| Payment success after order expired | paid if reconciled, else cancelled message | per DB logic §15 | re-checked under lock | documented alert |
| Amount/currency mismatch | error surfaced, no state change | unchanged + alert | unchanged | investigate + manual reconciliation |
| DB update failed after webhook success | retry-safe | may lag; idempotency makes retry safe | safe | background retry + alert |
| App closed before return | status screen on next launch | unchanged `pending` until webhook | reservation active until expiry | expiry job releases |

## 3. Security rules (normative)

1. **No Xendit secret key, callback token, or Supabase service-role key ever**
   ships in Flutter source, assets, APK config, or committed env files.
2. **No client payment-success boolean is ever trusted.** Only a verified,
   authenticated webhook process can set `paid`.
3. **No client can update order/payment status or stock** (RLS: no client
   grants on `orders`/`payments`/`inventory_reservations`).
4. **No authoritative total is computed from client pricing.**
5. Edge Function secrets live in `supabase secrets set` / `supabase/functions/.env`
   (see `ENVIRONMENT_SETUP.md`); commit only `.env.example`.
6. Public anon keys are acceptable *only* because RLS + grants enforce the data
   access model.
7. All user-facing function calls authenticate the caller (JWT). The webhook
   endpoint authenticates via the Xendit callback token instead.
8. Logs carry identifiers (order id, payment id, event id) — never keys,
   tokens, passwords, PANs, or full card data.
9. Test mode is a prerequisite to live charges: complete the E2E test-mode
   scenario (`TESTING_STRATEGY.md`) and record it before enabling production.

## 4. Reconciliation

- A scheduled job (Phase 7) compares `payments` rows that are
  `pending`/`requires_action` past `expires_at` against the Xendit session
  status (documented GET session endpoint) and reconciles to `expired`/`failed`
  as the provider reports.
- Provider-vs-DB disagreement (capture exists, DB still pending) is resolved by
  processing the capture event idempotently — the normally-arriving webhook is
  the cause; escalate only if stock is insufficient at reconciliation time.
- Refunds (Phase 7+): server-only RPC calling the Xendit refund endpoint with
  amount checks; transition `paid → refunded`.
