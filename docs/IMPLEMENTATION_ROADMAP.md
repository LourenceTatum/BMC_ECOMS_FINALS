# Implementation Roadmap

Dependency-aware phases. Follow the ordering: each phase's prerequisites must be
complete before its tasks start. Every phase ends with an acceptance review
against `DEFINITION_OF_DONE.md`. Never implement the whole app in one change —
work task-by-task.

Legend: **Prereq** = must exist first · **Deliverables** = produces ·
**Exit** = what proves the phase is done.

## Phase 0 — Repository & engineering foundation ✅ (current commit)

- **Work**: establish this documentation system (AGENTS.md, CLAUDE.md, docs/,
  skills, env templates); validate consistency rules.
- **Exit**: docs index cross-links resolve; git tree clean; this roadmap is
  committed. (This phase is the task that created these files.)

## Phase 1 — App foundation

- **Prereq**: Phase 0.
- **Tasks**:
  1. Replace the counter template: `main.dart` → `app/app.dart` bootstrap +
     `MaterialApp.router`; delete the demo counter; keep `// TODO` free.
  2. Add dependencies (verify SDK compatibility with Dart 3.13 / Flutter
     3.47.8 first): `supabase_flutter ^2.x`, `go_router ^18.x`,
     `flutter_riverpod ^3.x`, `url_launcher` (transitive already), and only
     what the current task needs.
  3. `app/router/app_router.dart` with public/protected route skeletons and
     guards (auth stream placeholder).
  4. `app/providers/supabase_provider.dart` + `auth_providers.dart`;
     `core/config/app_config.dart` with `--dart-define` validation.
  5. Theme in `core/theme/` + tokens from `DESIGN_SYSTEM.md`; font decision
     recorded (OFL font, Phase 1 commit note).
  6. `core/errors/*`, `core/utils/money.dart` with unit tests; `AsyncView`
     widget.
  7. Android: INTERNET permission, app label, applicationId change (owned
     domain) — decided with the user before release; keep as a checklist for now.
- **Affected**: `bmc_final/lib`, `android/`, `pubspec.yaml`.
- **Exit**: `flutter build apk --debug` succeeds; `flutter analyze` clean;
  app opens to a placeholder storefront shell behind auth guards.

## Phase 2 — Supabase foundation

- **Prereq**: Phase 1; Supabase CLI; a dev project; Docker for local stack.
- **Tasks**:
  1. Author migrations for all tables in `DATABASE_SCHEMA.md` + RLS/grants in
     `DATABASE_SECURITY.md` + storage buckets + seed script with **clearly
     labeled placeholder products**.
  2. Apply locally (`supabase db reset`) and to the dev project (`db push`).
  3. Create the DB tests from `TESTING_STRATEGY.md` §4 and run them.
- **Affected**: `supabase/migrations/`, `supabase/seed.sql`, storage buckets.
- **Exit**: all DB tests pass; RLS verified (user B blocked from user A data);
  catalog readable by anon; seeds clearly marked.

## Phase 3 — Authentication

- **Prereq**: Phase 1 (routing/providers), Phase 2 (profiles trigger).
- **Tasks**: register/login/logout, email verification UX, forgot+reset
  password with deep links, session restore, protected routes, first-login
  profile creation, `AUTHENTICATION.md` behaviors.
- **Exit**: full auth E2E on a device; recovery deep link works; unauthenticated
  users cannot reach protected screens; unit/widget tests pass.

## Phase 4 — Catalog & product detail

- **Prereq**: Phases 2–3.
- **Tasks**: storefront, listing grid + category/search/filter, detail screen
  with gallery, variants, stock display, sold-out handling; image loading via
  Supabase Storage URLs; states per `ERROR_HANDLING.md`.
- **Exit**: browse/search/filter flows work with seeded data on a real device;
  loader/empty/error states verified.

## Phase 5 — Shopping and order foundation

- **Prereq**: Phases 3–4.
- **Tasks**: cart (persisted `cart_items`), checkout review screen, addresses,
  `create-checkout-session` **minus** the Xendit call (returns a stubbed
  session URL behind a feature flag) with idempotency + order snapshots +
  reservations, order history/detail reads, `API_CONTRACTS.md` §1 normalization.
- **Exit**: duplicate taps and timeouts create one order; warehouse logic
  (snapshot, totals, reservation) proves out in DB tests; statuses match design.

## Phase 6 — Xendit integration (real payments)

- **Prereq**: Phase 5; Xendit test-mode account + activated PH channels;
  webhook URL reachable in dev.
- **Tasks**:
  1. Verify the current Xendit Session API contracts (fields, amount
     representation for PHP, webhook event shapes) against official docs; update
     `PAYMENT_ARCHITECTURE.md`/ADR 0004 if they differ; create a test-mode
     session manually to lock the number format.
  2. Implement the Xendit session call inside `create-checkout-session`
     (server secret in env; items/amount/references per §Stage 5).
  3. Implement `xendit-webhook`: token verification, dedupe, transitions,
     idempotency, reconciliation hooks.
  4. Wire the Flutter handoff (`url_launcher` external) + payment-status
     polling/refresh.
  5. Run the **smoke E2E** (`TESTING_STRATEGY.md` §5) in test mode and record
     the result; test delayed/duplicate webhooks.
- **Exit**: real test-mode payment goes pending → webhook → `paid` in DB and
  app; duplicate/late webhooks proven harmless; no client secret exposure;
  E2E evidence recorded.

## Phase 7 — Order lifecycle & hardening

- **Prereq**: Phase 6.
- **Tasks**: order history/detail full flows, payment-status refresh &
  realtime, expiry/reconciliation job, retry-payment UX, cancellation of unpaid
  orders, low-stock and reservation reports, refund path design (execution if
  scoped).
- **Exit**: failed/expired/cancelled/delayed paths behave per Stage 9 matrix;
  reconciliation job correct; monitoring hooks in place.

## Phase 8 — Hardening & release

- **Prereq**: Phases 0–7.
- **Tasks**: full test run (format, analyze, unit, widget, DB tests, E2E),
  security review (RLS + secrets + webhook), APK secret scan, release build +
  signing, production env + webhook registration, soft-launch smoke tests,
  production data sourcing (replace placeholders — legal catalog ownership),
  ADR statuses updated to Accepted where proven.
- **Exit**: `DEPLOYMENT.md` checklist complete; go/no-go document signed off;
  known limitations recorded.

## Task sizing rule

Each task above is decomposed into small PR-sized changes (one screen, one
migration, one function, one test batch). If a phase's task list feels large, an
agent decomposes it into tasks with their own acceptance criteria before coding
— and verifies that each task's dependencies are already present.

## Ordering rationale

DB before auth because profile triggers are needed by auth flows; catalog after
auth because detail/Buy Now need users; orders foundation before payments so the
provider call lands on an already-tested order machine; E2E gated on a real
provider in test mode so "payment works" means something.

