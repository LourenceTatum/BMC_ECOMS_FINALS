# AGENTS.md — BMC Commerce Engineering Rulebook

This is the primary, concise engineering rulebook for the **BMC Commerce** project
(a Flutter + Supabase + Xendit gothic luxury streetwear store). Every coding agent
working in this repository **must** read this file before editing anything.

- Full documentation lives in `docs/` (see index in section 3).
- Project-specific procedures live in `.claude/skills/` and load on demand.
- If this file conflicts with `CLAUDE.md` or any `docs/` file, this file wins;
  a conflict in `docs/` is a bug to fix (see `docs/README.md` consistency rules).

---

## 1. Project purpose and required stack

Mobile-first e-commerce store for the Philippine market selling **gothic luxury
streetwear** (Chrome Hearts-inspired *aesthetic only* — never claim affiliation
with, or endorsement by, Chrome Hearts). Users can register, log in, browse
products, select variants, **Buy Now**, pay through a **real Xendit hosted
checkout**, and view orders and payment statuses.

Required stack (verified against current official docs on 2026-10-24):

| Concern | Technology |
|---|---|
| Mobile app | Flutter 3.47.8 (stable) / Dart 3.13.7 |
| Authentication | Supabase Auth (email/password) |
| Database | Supabase PostgreSQL with Row Level Security |
| Backend logic | Supabase Edge Functions (Deno / TypeScript) |
| Payments | Xendit Payment Sessions — `mode: PAYMENT_LINK`, `session_type: PAY`, `currency: PHP`, `country: PH` |
| Image storage | Supabase Storage |
| Target market | Philippines; currency **PHP** |
| Primary platform | Android (iOS later) |

## 2. Repository layout

```
/                       Git repo root
├── AGENTS.md           <- this file (read first)
├── CLAUDE.md           Claude-specific operating guidance
├── README.md           Engineering hub
├── .gitignore          Repo-level ignores (env/secrets)
├── .env.example        Placeholder-only env template
├── docs/               All project documentation (index in section 3)
├── .claude/skills/     Claude Code project skills
└── bmc_final/          The Flutter application
    ├── lib/            App source (currently starter template only)
    ├── test/           Tests (starter counter smoke test)
    ├── android/        Android host project (primary platform)
    └── pubspec.yaml    App dependencies
```

## 3. Documentation index

Start at `docs/PROJECT_OVERVIEW.md`. Documents are authoritative for their topics
and are kept in agreement (`docs/README.md` defines the consistency rules).

| Area | Document |
|---|---|
| Overview & scope | `docs/PROJECT_OVERVIEW.md`, `docs/REQUIREMENTS.md` |
| Architecture | `docs/ARCHITECTURE.md`, `docs/APPLICATION_STRUCTURE.md` |
| Decisions | `docs/adr/` (0001–0005) |
| Database & security | `docs/DATABASE_SCHEMA.md`, `docs/DATABASE_SECURITY.md` |
| Authentication | `docs/AUTHENTICATION.md` |
| Payments | `docs/PAYMENT_ARCHITECTURE.md`, `docs/API_CONTRACTS.md` |
| UI/UX | `docs/DESIGN_SYSTEM.md` |
| Errors / env / security / deploy | `docs/ERROR_HANDLING.md`, `docs/ENVIRONMENT_SETUP.md`, `docs/SECURITY.md`, `docs/DEPLOYMENT.md` |
| QA | `docs/TESTING_STRATEGY.md` |
| Execution | `docs/IMPLEMENTATION_ROADMAP.md`, `docs/DEFINITION_OF_DONE.md` |

## 4. Engineering rules

### 4.1 Architecture principles

- Feature-first, modular Flutter architecture (see `docs/ARCHITECTURE.md`).
- Boundaries: **presentation → application/business logic → data access**. No
  excessive enterprise abstractions, no speculative features.
- State management: **Riverpod** (`flutter_riverpod`). Navigation: **go_router**
  with route guards. Supabase: **supabase_flutter** using the publishable/anon
  client. Do **not** add a competing state manager, router, or code-gen stack
  without a documented reason (see ADR 0002).
- Provider truth lives on the **backend**. The client is never authoritative for
  prices, stock, totals, or payment status.
- Dependencies stay minimal; reuse established project conventions.

### 4.2 Flutter/Dart conventions

- Format with `dart format` (line length 100). Run `flutter analyze` and keep
  warnings at zero in changed code; do not suppress lints to make checks pass.
- Prefer `const` constructors and immutable value models. Code generation is not
  in use; avoid adding it without a documented reason.
- Feature folders: `data/`, `domain/`, `presentation/`. Small features may use
  fewer subfolders — never create trivial single-file folders.
- Files/folders `snake_case`; classes `PascalCase`; members `camelCase`.
- **Money is never `double`.** Use `int` minor units in Dart at boundaries and
  convert from/to the database's `numeric` server-side. Binary floating point is
  forbidden for currency (see `docs/DATABASE_SCHEMA.md`).

### 4.3 Supabase data-access rules

- App DB access uses the anon/publishable client and must be authorized by RLS.
- **Never** put service-role keys, `SUPABASE_SECRET_KEYS`, or Supabase secrets in
  Dart code, assets, APK config, or anything shipped with the app.
- Privileged operations (price computation, order creation, payment transitions,
  stock mutation, webhooks) run in **Edge Functions** with server-side creds.
- Access model: anonymous read of active catalog; users read/write only their own
  profiles, addresses, cart, favorites, orders; payments read-only per owning
  order. See `docs/DATABASE_SECURITY.md`.

### 4.4 Database migrations and RLS rules

- All schema changes are version-controlled SQL migrations under
  `supabase/migrations/` (Supabase CLI). Ad-hoc Dashboard edits are not the source
  of truth.
- Every public table: `enable row level security` + explicit `GRANT`s (RLS is not
  a substitute for privileges) + purpose-stated policies with the abuse case they
  prevent + constraints/indexes. Canonical tables and shapes:
  `docs/DATABASE_SCHEMA.md`.

### 4.5 Xendit payment rules (non-negotiable)

- **Only the server talks to Xendit** — inside authenticated Edge Functions. No
  Xendit API key ever ships in the Flutter app.
- Payment success is determined **only** by a verified Xendit webhook
  (`x-callback-token` header) processed server-side. A redirect back from the
  hosted checkout, a client boolean, or "success" navigation in the app is **not**
  proof of payment.
- The client can never mark an order paid or mutate payments/privileged order
  state. The server never trusts a client-supplied total.
- Idempotency everywhere: unique `idempotency_key` per checkout request, provider
  event dedup, enforced order/payment state transitions.
- Amounts: canonical DB storage `numeric(12,2)` PHP; server converts to the
  Xendit-expected representation (Session API PHP examples use integer minor
  units — confirm with a test-mode session during Phase 6). See
  `docs/PAYMENT_ARCHITECTURE.md` and ADR 0004.
- Webhook receiver returns 2xx immediately, then processes idempotently.

### 4.6 Error handling / loading / empty states

- Every async screen handles **loading, error, data, and empty** states.
- Errors are mapped to user-safe messages; logs contain identifiers but never
  keys, tokens, passwords, or PAN data. See `docs/ERROR_HANDLING.md`.

### 4.7 Testing requirements

- Unit tests for calculations/validation/state transitions; widget tests for
  flows; database/RPC tests for RLS and payment logic; one integration test for
  end-to-end checkout (see `docs/TESTING_STRATEGY.md`).
## 5. Feature implementation workflow

Every coding agent must:

1. **Read the rules**: `AGENTS.md` + the docs for the affected area(s). If a
   relevant skill exists in `.claude/skills/`, load it.
2. **Inspect the existing implementation** before designing anything.
3. **Understand** requirements and acceptance criteria (`docs/REQUIREMENTS.md`).
4. **Identify affected screens, providers, repositories, tables, policies, and
   backend functions.**
5. **Propose a concise implementation plan** for non-trivial work.
6. **Implement ONLY the requested task** and its necessary dependencies. No
   unrelated refactoring, new dependencies, duplicate abstractions, or
   speculative features.
7. **Reuse established conventions** and existing components where appropriate.
8. **Update documentation** when the change alters an established contract
   (state machine, API contract, schema, env var, design token).
9. **Run relevant tests, static analysis, and build checks** whenever possible.
10. **Inspect the final diff** for accidental or unrelated changes.
11. **Report honestly**: completed work, changed files, tests executed and results,
    failures, and remaining limitations. Never claim untested things passed.

### Scope-control (operating principle)

- "Implement **only** the requested task. Preserve existing functionality. Do not
  silently expand scope."
- Do not automatically rewrite working code because a different style seems nicer.
- Do not perform broad refactors while implementing a focused feature.
- Do not change database architecture, auth behavior, the payment lifecycle, or
  design conventions without documenting the reason and updating the specs/ADRs.
- Simple, isolated tasks: no new planning documents or abstractions.

## 6. Definition of done

From `docs/DEFINITION_OF_DONE.md`: a task is complete when **all** of these hold:

- Code exists, is wired into the router/providers, and matches the documented
  contract and acceptance criteria.
- Security rules in section 4 (RLS, server-only secrets, webhook verification,
  idempotency, PHP money handling) hold for the change.
- Loading, error, and empty states are handled for every affected async screen.
- Tests cover the change (unit/widget/database as applicable) and actually pass:
  `dart format .`, `flutter analyze`, `flutter test` run without meaningful
  warnings/failures from `bmc_final/`.
- Documentation was updated if any contract changed; the final
  `git diff --check` and `git status` are clean of accidental changes.
- The agent's report distinguishes what was implemented, what was verified, and
  what remains planned or blocked.

