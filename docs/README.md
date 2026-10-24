# Documentation Index

All project documentation lives here. `AGENTS.md` (repo root) is the primary
rulebook; these documents are the detailed specifications. The entire set is
kept in agreement — see [Consistency rules](#consistency-rules).

## Documents

| Document | Purpose |
|---|---|
| `PROJECT_OVERVIEW.md` | Product, goals, users, stack, scope, confirmed-facts vs assumptions |
| `REQUIREMENTS.md` | Functional requirements + acceptance criteria for the MVP |
| `ARCHITECTURE.md` | Feature-first Flutter architecture and layer responsibilities |
| `APPLICATION_STRUCTURE.md` | Target `lib/` layout and file organization |
| `DESIGN_SYSTEM.md` | Gothic luxury storefront visual system (colors, type, spacing) |
| `DATABASE_SCHEMA.md` | Relational model, money handling, order/payment state machines |
| `DATABASE_SECURITY.md` | Table-by-table RLS policies, grants, abuse cases |
| `AUTHENTICATION.md` | Supabase Auth lifecycle and route protection |
| `PAYMENT_ARCHITECTURE.md` | End-to-end Xendit Buy Now flow and security guarantees |
| `API_CONTRACTS.md` | Edge Function + data API contracts |
| `ERROR_HANDLING.md` | Consistent error, loading, and recovery patterns |
| `ENVIRONMENT_SETUP.md` | Developer setup on Windows + toolchain verification |
| `SECURITY.md` | What ships in the app vs stays server-side; secret hygiene |
| `TESTING_STRATEGY.md` | Unit, widget, database, and end-to-end test policy |
| `DEPLOYMENT.md` | Android, Supabase, Edge Functions, webhook, go-live checklist |
| `IMPLEMENTATION_ROADMAP.md` | Phase-by-phase implementation plan (Phase 0 → 8) |
| `DEFINITION_OF_DONE.md` | Completion checklist used by every task |
| `adr/` | Architecture Decision Records (0001–0005) |

## Skills

Project skills live at `.claude/skills/` (Claude Code Agent Skills format):

- `flutter-feature-development` — scoped Flutter feature work
- `flutter-ui-design` — implementing the gothic design system
- `supabase-database-security` — schema migrations, RLS, grants
- `xendit-payment-integration` — server-side payments, webhooks, idempotency
- `testing-and-quality-assurance` — test-first QA workflow

## Environment shortcuts

- Flutter app lives in `bmc_final/` — run `flutter`/`dart` commands from there.
- `.env.example` at repo root documents placeholder-only environment names.

## Consistency rules

Before finishing any task that touches specifications, verify:

1. `AGENTS.md` points to the actual doc paths (and vice-versa).
2. `CLAUDE.md` does not contradict `AGENTS.md`.
3. `ARCHITECTURE.md` matches `APPLICATION_STRUCTURE.md`.
4. `DATABASE_SCHEMA.md` and `DATABASE_SECURITY.md` use identical table/column/relationship names.
5. `AUTHENTICATION.md` agrees with routing and the Supabase Auth approach in `ARCHITECTURE.md`.
6. `PAYMENT_ARCHITECTURE.md` agrees with `API_CONTRACTS.md` and `DATABASE_SCHEMA.md`
   (order/payment states, amount fields, idempotency keys).
7. Order and payment states are consistent across database design, UI states,
   error handling, tests, and roadmap.
8. The inventory strategy does not conflict with the payment flow.
9. `DESIGN_SYSTEM.md` matches `REQUIREMENTS.md` and the intended aesthetic.
10. The roadmap lists prerequisites for each critical task.
11. Every listed skill exists, has valid frontmatter, and gives actionable steps.
12. Documentation never claims a feature is implemented because its design exists.

If a conflict appears, resolve it and record important decisions in `docs/adr/`.
