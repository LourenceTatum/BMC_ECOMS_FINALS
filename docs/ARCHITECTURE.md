# Architecture

Feature-first, modular architecture for a small-to-medium Flutter e-commerce app.
Deliberately pragmatic — no layers for their own sake. Recommended packages are
flagged **planned** (to be added in Phase 1 after SDK-compatibility checks).

## 1. Layers and boundaries

```
Presentation (UI)          widgets, screens, page-scoped state
        │ watches providers / calls controllers
Application / Domain       Riverpod providers, business rules, validation
        │ uses repositories / services
Data access                repositories → Supabase (anon client),
                           Edge Function invocations
        │
External                    Supabase DB/Auth/Storage, Edge Functions,
                            Xendit (server-side only)
```

Rules:
- **UI never touches raw Supabase** except through repositories/providers.
- **Repositories** abstract where data comes from (DB query vs edge function).
- **Business rules live in application logic** (price display formatting,
  validation helpers, state mapping), but **authoritative money and stock rules
  live on the server** — the client mirrors, never overrides.
- Cross-feature models live in `core/`; feature models live with their feature.

## 2. State management and application logic

- **Riverpod (`flutter_riverpod`)** — recommended, planned. Use:
  - `Provider` for singletons/repositories and the Supabase client.
  - `AsyncNotifier`/`FutureProvider` for async feature state (catalog, orders).
  - `Notifier` for mutable UI state (cart interactions, forms).
- Controller-style providers call repositories, map errors (see
  `ERROR_HANDLING.md`), and expose loading/error/data states to widgets.
- Cart data is *displayed* from a responsive provider but **persisted** in
  `cart_items` (authoritative) so it syncs across devices and reinstalls.
  Payments never have client-only state.
- Source-of-truth hierarchy: **server (DB/edge functions) → repository →
  provider → widget**. UI state (selected variant, cart quantity) is ephemeral
  and revalidated server-side at checkout.

## 3. Navigation

- **go_router** — recommended, planned. Declarative URL-style routes.
- Route protection via `redirect` based on the Supabase auth stream
  (`onAuthStateChange` + current-user provider). Guards:
  - Public: `/` (storefront), `/catalog`, `/product/:slug`, auth screens.
  - Protected: `/cart`, `/checkout/(review|payment)`, `/orders`, `/orders/:id`,
    `/profile`, `/addresses`, `/favorites`.
- Auth screens honor a `redirect` query param and return there on success.
- Payment handoff: after returning from the hosted checkout, the app re-resolves
  `/checkout/payment` against the server — never trusts the return was success.
- Route transitions use the motion rules in `DESIGN_SYSTEM.md`.

## 4. Data access

- **Supabase client**: a single instance from `Supabase.initialize` with
  `SUPABASE_URL` + publishable anon key, exposed via a provider. Used for auth,
  RLS-limited PostgREST, storage, and `functions.invoke`.
- **Repositories per feature**:
  - `catalog_repository` — public reads: products, categories, variants, images.
  - `cart_repository` — own cart/cart_items writes.
  - `order_repository` — own orders/order_items/payments reads; checkout
    creation via the `create-checkout-session` edge function.
  - `profile_repository` — own profile and addresses.
  - Payment status is read-only for clients (own order's payment row).
- **Idempotency**: checkout requests carry an `idempotencyKey` (UUID); the
  `orders.idempotency_key` column is unique (see `DATABASE_SCHEMA.md`).

## 5. Supabase client responsibilities (in-app)

Auth state, RLS-limited queries, storage URLs, invoking Edge Functions. Never:
service-role operations, writes to privileged tables, authoritative money math.

## 6. Edge Function responsibilities

`create-checkout-session`: authenticate caller (JWT); validate input; re-read
canonical product/variant/price/stock in a transaction; compute PHP totals;
create pending order + payment attempt + inventory reservation; call Xendit;
store provider identifiers; return `{ orderId, checkoutUrl }`.

`xendit-webhook`: receive provider events; verify `x-callback-token`; locate the
payment by reference; dedupe by event id; enforce legal state transitions; update
payment and order state; return 2xx. See `API_CONTRACTS.md` and
`PAYMENT_ARCHITECTURE.md`.

## 7. External payment provider responsibilities

Xendit owns: payment-method selection, credential collection, authentication
(OTP/redirect), capture, and webhook dispatch. It does **not** own server
validation, idempotency bookkeeping, or the post-return UI — the app reconciles
through server state after the customer returns.

## 8. Models

- **Domain models** (`domain/`): `Product`, `ProductVariant`, `CartItem`,
  `Order`, `OrderItem`, `Payment`, `Profile`, `Address` — immutable.
- **DTOs** (`data/`): DB/API mappers. Edge-function contract types mirror
  `API_CONTRACTS.md` (e.g. `CreateCheckoutRequest`/`CheckoutSessionResult`).
- Money travels as `int` minor units in DTOs; formatting (₱ + 2 decimals) is a
  `core/utils` helper driven by the database `numeric` values.

## 9. Dependency injection

Manual DI via Riverpod providers (top-level providers are overridable in tests).
No mirror- or code-gen-heavy injection framework. The Supabase client is exposed
as a `Provider<SupabaseClient>`.

## 10. Error propagation

- Repositories throw domain-typed errors: `NetworkError`, `UnauthorizedError`,
  `ValidationError`, `ConflictError`, `OutOfStockError`, `CheckoutError`,
  `PaymentRequiresAction`, `UnknownError`.
- Providers catch, map, and expose `AsyncError` states; widgets render per
  `ERROR_HANDLING.md`. Raw Supabase/Xendit exceptions never reach the user.

## 11. Testing boundaries

- **Unit**: formatting, cart math, validation, order/payment status mapping,
  repositories with mocked clients.
- **Widget**: per-flow widget tests with provider overrides and fake repositories.
- **Backend/database**: RLS and RPC tests against a local Supabase stack.
- **E2E**: one full checkout scenario using Xendit test mode
  (`TESTING_STRATEGY.md`); Supabase/Xendit are only mocked for app-layer tests.

## 12. How a feature moves through the architecture (Buy Now example)

1. Product detail widget → user taps Buy Now.
2. `CheckoutController` validates the session and local input (variant/qty).
3. Calls `OrderRepository.createCheckout({ variantId, qty, addressId,
   idempotencyKey })` → `create-checkout-session` edge function.
4. Edge function verifies auth, re-reads catalog + stock, computes totals,
   creates pending order/payment/reservation, creates the Xendit Session, and
   returns `orderId` + hosted URL.
5. Controller shows a "handing off" state then opens the hosted URL via
   `url_launcher` (external browser).
6. Customer pays. Verified webhook → edge function → DB `paid`. App refreshes
   order status on return/poll; UI reflects server-confirmed state.
7. No client redirect or client boolean is accepted as payment evidence.
