# Requirements (MVP)

Feature specification and testable acceptance criteria. "Verified on server"
means the authoritative value must come from Supabase/Edge Functions, never a
client claim. A feature is done when its acceptance criteria are met **and** the
definition of done (`docs/DEFINITION_OF_DONE.md`) is satisfied.

## 1. Authentication (Supabase Auth, email/password)

| # | Requirement | Acceptance criteria |
|---|---|---|
| A1 | Register with email + password | Unauthenticated user submits valid email + password (≥ 8 chars). App calls Supabase Auth `signUp`. Duplicate email returns a clear "account already exists" message. With verification disabled (dev) the user is signed in; with verification enabled (prod) the app shows a "verify your inbox" state. |
| A2 | Log in / log out | `signInWithPassword` with correct credentials restores the session; wrong credentials show a friendly error; logout (`signOut`) clears the session and protected routes redirect to login. |
| A3 | Email verification | When enabled, unverified users cannot complete purchases; resend-verification action exists; the verified flag is refetched from the server. |
| A4 | Forgot / reset password | "Forgot password" triggers `resetPasswordForEmail`; the password-recovery deep link opens the reset-password screen; a new password updates the user and permits sign-in. |
| A5 | Session persistence/restore | `onAuthStateChange(initialSession)` restores a live session on cold start; expired/revoked sessions are detected and routed to login with an informative message. |
| A6 | Protected screens | Catalog is public; cart, checkout, orders, profile, addresses, favorites require a session. Unauthenticated attempts redirect to login with an intended-route return. |
| A7 | Validation & UX | Inline field validation (email format, password strength), per-field errors, loading states on buttons, no sensitive data in logs. |
| A8 | First-login profile creation | On first sign-in a `profiles` row exists with `id = auth.uid()` (trigger or function). User can later edit display name/phone/avatar. |

## 2. Product catalog

| # | Requirement | Acceptance criteria |
|---|---|---|
| C1 | Storefront | Public home screen shows featured/active products, an editorial hero, and category rails. Loading, error, and empty states handled. |
| C2 | Product listing | Grid/list of active products with image, name, PHP price, and availability indicator; pagination or load-more for large sets. |
| C3 | Categories/collections | Products grouped by active `categories`; browsing by category preserves filters. |
| C4 | Search & filter | Search box over name/description (`ilike`), category + sort (price, newest) filters. Practical only — no fuzzy engine. |
| C5 | Product detail | Image gallery, name, price, description, variant selector (sizes/colors), per-variant availability and stock ("Only 3 left"), sold-out state, quantity stepper bounded by stock, Add-to-Cart and Buy Now. |
| C6 | Variants & stock | A purchasable SKU is a `product_variants` row with its own `stock_qty`; a sized product is never one undifferentiated stock bucket. Stock always reflects server data. |
| C7 | Sold-out handling | Sold-out variants are disabled with a "Sold out" state; no purchase path exists for them. |

## 3. Shopping, checkout, and payment (Buy Now = core flow)

| # | Requirement | Acceptance criteria |
|---|---|---|
| S1 | Buy Now | From product detail (variant + qty chosen), Buy Now creates a checkout for that single line without adding to cart. Button shows loading; duplicate taps are idempotent (one order). |
| S2 | Cart | Add-to-cart persists `cart_items` per user; quantity updates are clamped by stock; remove item; cart badge reflects line count; anonymous visitors are prompted to log in (no anonymous persistence). |
| S3 | Quantity/variant validation | Client validates input bounds; **server revalidates** that the variant is active, exists, stock ≥ qty, and prices are canonical. |
| S4 | Checkout review | Shows item snapshots, subtotal, shipping fee, discount (0 in MVP), and total — computed from the server at session creation (`PAYMENT_ARCHITECTURE.md`). |
| S5 | Address for delivery | User selects a delivery address from their own address book; the order records `shipping_address_id` at creation. |
| S6 | Xendit-hosted payment | App opens the Xendit hosted checkout URL returned by `create-checkout-session` in an external browser. No raw payment credentials ever pass through the app. |
| S7 | Payment states | Order/payment screens render **pending**, **paid**, **failed**, **cancelled**, **expired** exactly as the server reports. Returning from the hosted page is NOT success; the app shows "confirming payment…" until the server confirms otherwise. |
| S8 | Order confirmation/history | After the server confirms payment, the order appears in history with items, totals, address, and status timeline (pending_payment → paid → …). |
| S9 | Status refresh | Pull-to-refresh refetches order/payment status from authorized Supabase queries; realtime may be layered on later. |

## 4. Account

| # | Requirement | Acceptance criteria |
|---|---|---|
| P1 | Profile | View/edit `full_name`, phone, avatar; read-only email. Only own profile editable. |
| P2 | Address book | CRUD own addresses (label, recipient, phone, street, city, province, postal code, country = PH default); one address marked default. |
| P3 | Order history/detail | Read own orders, order items, and payment status. Accessing another user's order raises an authorization failure — it is never merely hidden. |
| P4 | Sign out & protection | Sign out returns to the storefront; protected routes redirect when unauthenticated. |

## 5. Non-functional requirements

- **Security**: RLS enabled everywhere (`DATABASE_SECURITY.md`); no secrets ship
  in the app; webhooks verified; no client trust of totals or payment success;
  PHP money handled exactly end-to-end (never binary floats).
- **Reliability**: idempotent checkout creation and webhook processing; payment
  state converges under retries, duplicates, and out-of-order events.
- **Maintainability**: follow `ARCHITECTURE.md`; scoped changes; zero meaningful
  analyzer warnings; docs updated when a contract changes.
- **Responsive mobile layouts**: Android portrait-first, safe-area aware, works
  down to ~360dp width, respects large-font accessibility settings.
- **Accessibility**: AA contrast for text, ≥48dp touch targets for key actions,
  semantic labels for icon-only controls.
- **Performance**: cached image loading with placeholders/skeletons; list
  recycling; smooth catalog scrolling on mid-range Android.
- **Debuggability**: structured, safe logging — identifiers only, never secrets,
  tokens, passwords, or PAN data.
- **Data consistency**: DB constraints + transactions guarantee order snapshots,
  inventory, and idempotency invariants (`DATABASE_SCHEMA.md`).
- **Testability**: layered unit/widget tests + RLS/database tests + one E2E
  scenario (`TESTING_STRATEGY.md`).

## 6. Definition of done — critical features (payment emphasis)

For payment-related features: the server creates the Xendit session with a
server-computed PHP amount; the webhook intake verifies the `x-callback-token`,
dedupes by provider event/payment id, and enforces legal transitions; an order
shows `paid` only after verified server-side success; and no code path lets a
client mark an order paid or alter inventories outside authorized operations.

## 7. Out of first release

Subscriptions; loyalty/rewards; chatbots/community; marketing automation;
coupons/discount engine; multi-currency; admin dashboard build-out (a placeholder
`lib/admin/` exists; its full dashboard is deferred); self-service refunds;
iOS release engineering.
