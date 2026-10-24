# Application Structure

Target layout for the Flutter app (`bmc_final/lib/`). The current repository
only contains the starter template (`lib/main.dart`, `lib/admin/admin.dart`) —
this is the **target** the roadmap builds toward (Phase 1+). Keep related files
together; never split simple screens into many trivial files.

## Target layout

```
bmc_final/lib/
├── main.dart                     App entry: init Supabase, run App
├── app/
│   ├── app.dart                  Root MaterialApp.router + theme wiring
│   ├── router/
│   │   ├── app_router.dart       go_router config + redirect guards
│   │   └── routes.dart           Route path/policy constants
│   └── providers/
│       ├── supabase_provider.dart   SupabaseClient provider
│       └── auth_providers.dart      currentUser/session providers
├── core/
│   ├── config/
│   │   ├── app_config.dart       URL/key from --dart-define, kDebugMode warns on empty
│   │   └── environment.dart      dev/staging/prod flags
│   ├── constants/                 app-wide constants, storage buckets
│   │   └── assets_paths.dart      (later)
│   ├── errors/
│   │   ├── app_error.dart         domain error types
│   │   └── error_mapper.dart      error → user message + log severity
│   ├── theme/
│   │   ├── app_theme.dart         ThemeData for the design system
│   │   ├── tokens.dart            colors, spacing, typography tokens
│   │   └── text_styles.dart       serif/body styles
│   ├── utils/
│   │   ├── money.dart             int minor-units ↔ format ₱ (no doubles)
│   │   ├── formatters.dart        dates, quantity
│   │   └── validators.dart        email, password, PH phone
│   └── widgets/
│       ├── async_view.dart        loading/error/empty/data frame
│       ├── primary_button.dart
│       ├── product_media.dart     cached image w/ placeholder (gothic style)
│       └── motif/                 cross ornament divider, mark, etc.
└── features/
    ├── auth/
    │   ├── domain/                AuthUser model (session wrapper)
    │   ├── data/                  auth_repository.dart
    │   └── presentation/          login, register, forgot_password,
    │                              reset_password, verify_email screens +
    │                              auth_controller providers
    ├── home/                      Storefront (editorial hero, rails)
    ├── catalog/                   List + category/search/filter, providers
    ├── product_detail/            Gallery, variant select, qty, Buy Now
    ├── cart/                      Cart screen + cart provider (server-backed)
    ├── checkout/
    │   ├── review/                Checkout review screen
    │   ├── payment/               Handoff interstitial + payment status screen
    │   └── providers/             checkout_controller, payment_status
    ├── orders/
    │   ├── history/               order list
    │   ├── detail/                order + item + payment status
    │   └── providers/             orders_provider, order_detail_provider
    ├── profile/                   Profile screen + edit
    ├── addresses/                 Address book CRUD
    ├── favorites/                 Wishlist
    └── shared/
        └── models/                Product/Variant/Order shared models if
                                  needed by multiple features (else feature-local)
```

## Feature folder convention

A feature has up to three subfolders — only as needed:

- `data/`: repository + DTO mappers for Supabase/edge function access.
- `domain/`: immutable feature models and pure business rules.
- `presentation/`: screens, widgets, and Riverpod controllers for this feature.

Examples:
- `auth` needs all three.
- `favorites` may only need `data/` + `presentation/` and a small model in
  `favorites/domain/favorite.dart` if it needs more than a product id list.
- `home` is mostly presentation, reusing catalog data providers.

## Ownership of shared concepts

- Money: `core/utils/money.dart` only.
- Order/payment status enums: defined once (feature `orders/domain/order_status.dart`)
  and shared by `checkout` and `orders` via the feature's domain file.
- Auth state: `app/providers/auth_providers.dart`; features never call
  `Supabase.instance.client.auth` directly.

## What not to do

- Do not put the whole app in `main.dart`.
- Do not create `screens/`, `pages/`, `widgets/`, `models/`, `repositories/`
  mega-folders at `lib/` root; keep files in feature folders.
- Do not add empty `data/`/`domain/`/`presentation/` folders speculatively.
- Do not introduce a second router or state-management stack.

## Relationship to documentation

- `ARCHITECTURE.md` describes this structure's rationale and data flow.
- `DESIGN_SYSTEM.md` describes what shared widgets (`core/widgets`) must look like.
- Modifying this layout requires updating both this file and `ARCHITECTURE.md`.
