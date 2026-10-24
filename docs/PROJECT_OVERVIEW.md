# Project Overview — BMC Commerce

## What the app does

BMC Commerce (working codename; brand name TBD) is a mobile-first e-commerce
store for gothic luxury streetwear, sold in the Philippines and priced in PHP.
Customers can:

- register and log in (email/password, Supabase Auth);
- browse a catalog of products with images, descriptions, and prices;
- select purchasable variants (size/color) and quantities;
- buy an item immediately (**Buy Now**) or via a cart;
- pay with a **real Xendit hosted checkout** (local PHP payment methods);
- view order history and trust only server-confirmed payment statuses.

## Who will use it

Philippine consumers who buy premium, fashion-forward streetwear online. The
app is Android-first; iOS is a later target.

## Visual direction

Premium gothic luxury aesthetic inspired by Chrome Hearts — **aesthetic only**.
The app is an independent store and must never claim to be, or be an authorized
reseller of, Chrome Hearts. Product photography and catalog data must be legally
sourced; development catalogs use clearly labeled placeholder products until
verified data exists.

## Technology stack

| Layer | Technology | Role |
|---|---|---|
| App | Flutter 3.47.8 / Dart 3.13.7 | UI on Android (primary) |
| State/nav (planned) | Riverpod, go_router | DI, state, guarded routing |
| Auth | Supabase Auth | Email/password, sessions, password recovery |
| Database | Supabase PostgreSQL + RLS | Canonical data: users, products, orders, payments |
| Server logic | Supabase Edge Functions | Checkout session creation, webhook intake |
| Payments | Xendit Payment Sessions (`mode: PAYMENT_LINK`) | Hosted checkout, PHP currency |
| Storage | Supabase Storage | Product imagery |

## High-level system context

```
Flutter app (Android)
  ├─> Supabase Auth            (sessions, email/password)
  ├─> Supabase PostgREST       (catalog reads via RLS; own-data writes via RLS)
  ├─> Edge Function: create-checkout-session   (authenticated)
  │       └─> Xendit API       (create hosted Payment Session; PHP total)
  ├─> Edge Function: xendit-webhook            (unauthenticated at app level,
  │       authenticated via Xendit x-callback-token) → DB updates
  ├─> Supabase Storage         (product images)
  └─> Hosted checkout page     (external browser, Xendit-hosted)
```

## Architectural boundaries

- **Client** (Flutter): presentation, validation of user input, navigation,
  opening the hosted checkout, reading own/ authorized data via RLS.
- **Server** (Edge Functions): all pricing, stock reservation, order/entity
  creation, Xendit calls, webhook verification, payment transitions, money math
  against canonical DB values.
- **Database**: source of truth for catalog, inventory, orders, payments.
  RLS enforces ownership; constraints enforce invariants.

## Development environment

- Windows 11 host (PowerShell); see `ENVIRONMENT_SETUP.md`.
- Verified toolchain on 2026-10-24: Flutter 3.47.8 stable (revision
  737ba8f885), Dart 3.13.7, DevTools 2.60.0.
- Git repo root is the workspace root; the Flutter app is in `bmc_final/`.

## Scope — initial release (MVP)

Authentication (register/login/logout/verify/recover), catalog + product details
with variants and stock visibility, cart, **Buy Now**, checkout review with real
totals from the server, Xendit hosted payment, order history/detail with
server-confirmed statuses, profile and address book, favorites.

## Intentionally deferred

Subscriptions, loyalty points, chatbots/reviews, marketing automation, push
notifications, admin dashboard (a placeholder `lib/admin/` exists — its full
dashboard is deferred), iOS polish, multiple currencies/shipping providers,
refunds self-service, and discounts coupons (design allows coupons later).

## External services and their responsibilities

| Service | Responsibility | Credential model |
|---|---|---|
| Supabase | Auth, DB, storage, functions, RLS | anon/publishable in app; secrets server-only |
| Xendit | Hosted payment sessions, webhooks | secret key + callback token server-only |
| (Future) image CDN | optional | — |

## Confirmed repository facts vs assumptions

**Confirmed (inspected 2026-10-24):** repo has 2 commits on `main`; Flutter
app in `bmc_final/` is the stock counter template; only dependencies are
`cupertino_icons` and `flutter_lints`; `lib/admin/admin.dart` is empty; default
smoke test passes structure; no Supabase/Xendit code, migrations, or docs exist;
Android is the scaffolded platform; app id `com.example.bmc_final` (unfinished —
must be changed before release).

**Assumptions (to verify during implementation):** email verification will be
enabled; standard commercial direct-to-consumer sales do not exceed Xendit's
default limit without account activation; Supabase DB region near the Philippines;
final brand name and legal entity. Each is flagged in the phase where it is
relevant.
