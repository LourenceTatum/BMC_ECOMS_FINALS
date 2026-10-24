---
name: flutter-feature-development
description: Implement scoped Flutter features in the BMC Commerce app (bmc_final).
  Use when building or modifying screens, providers, repositories, or models within
  a feature folder. Covers inspecting the feature structure, tracing state/data
  flow, handling loading/empty/error states, following conventions, and running the
  relevant checks before finishing.
---

# Flutter feature development

For building or editing Flutter features in `bmc_final/` without breaking existing
functionality or the architecture.

## When to use
- Adding or changing a screen, provider, repository, or model in `bmc_final/lib/`.
- Updating auth, catalog, cart, checkout, orders, profile, addresses, favorites.

## Before coding
1. Read `AGENTS.md` (repo root) and the docs that govern the area:
   - any feature: `docs/ARCHITECTURE.md`, `docs/APPLICATION_STRUCTURE.md`
   - payments/checkout: `docs/PAYMENT_ARCHITECTURE.md`, `docs/API_CONTRACTS.md`
   - auth: `docs/AUTHENTICATION.md`
2. Inspect the existing code for the feature folder. Note:
   - the provider/repository seams that already exist;
   - riverpod usage pattern (Notifier vs FutureProvider) already in the folder;
   - which models already exist (never redefine a model that exists).
3. Confirm the acceptance criteria in `docs/REQUIREMENTS.md` for the feature.

## Implementation rules
1. Keep changes minimal and scoped to the task: one screen, one controller, one
   repository method at a time. Do not refactor unrelated folders.
2. Follow the folder conventions (`data/`, `domain/`, `presentation/` only where
   useful). Never create speculative empty folders.
3. Route new screens through the `go_router` config (`lib/app/router/`); protect
   screens via the existing auth redirect, never via one-off `if` blocks in
   `build()`.
4. State flows through Riverpod providers that call repositories. Widgets must not
   call the Supabase client directly.
5. Money: use `core/utils/money.dart` (`int` minor units at boundaries). Never
   introduce `double` for currency.
6. Every async surface handles **loading, error, data, empty** states via the
   established `AsyncView` pattern (see `docs/ERROR_HANDLING.md`).

## After coding
1. Format and analyze from `bmc_final/`:
   - `dart format .`
   - `flutter analyze` (zero new warnings/errors)
2. Run and write relevant tests: `flutter test`. Update or add widget/unit tests
   for the feature (`docs/TESTING_STRATEGY.md`).
3. If the task touched a contract (schema, status vocabulary, API shape, routes),
   update the matching doc under `docs/`.
4. Review the diff: `git diff --check`; discard unrelated changes.

## Common traps
- Re-creating a repository/model that already exists elsewhere — search first.
- Using `double` for prices or totals.
- Adding a second state-management/DI mechanism.
- Calling `functions.invoke` or PostgREST from widgets.
- Claiming a real provider payment was tested when only mocked.
