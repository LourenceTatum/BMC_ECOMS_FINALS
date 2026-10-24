# ADR 0003 — Supabase as the Managed Backend Platform

- **Status**: Proposed (foundation work in Phases 2–3).
- **Date**: 2026-10-24

## Context

We need authentication, a relational store, storage, and server-side logic for a
PHP/Philippines e-commerce app, managed by AI agents on a tight budget.

## Decision

Use **Supabase** for all managed backend needs:

- **Supabase Auth** (email/password; session persistence via supabase_flutter).
- **Postgres** (via PostgREST) as the single source of truth with **RLS + grants**
  per `docs/DATABASE_SECURITY.md`.
- **Edge Functions** (Deno) for privileged operations (checkout creation,
  webhook intake) holding `service_role` credentials server-side.
- **Storage** for product images/avatars with documented bucket policies.

Version control: migrations in `supabase/migrations/`; secrets via
`supabase secrets set` + gitignored local `.env`.

## Alternatives considered

- Firebase Auth/Store — rejected: not a requirement; Postgres/RLS models the
  order/inventory invariants better; edge functions give vendored control.
- Custom Node/Go API — rejected: extra infra and ops burden with no new
  capability for this team.
- Local-only storage — rejected explicitly (order/payment truth must be server
  side).

## Advantages

Fast, auth+DB+function+storage cohesion; RLS is a strong ownership primitive;
Dart client is first-class.

## Disadvantages

Lock-in to Supabase; PostgREST faces fun edges (composite queries); local
replication needs Docker. Mitigated by thin data layer (repositories) and the
ADR 0001 boundaries.

## Security/maintenance implications

The single risk is secret handling and RLS mistakes; they are codified in
`SECURITY.md`, `DATABASE_SECURITY.md`, and enforced by DB tests.

## Consequences

All backend code is Supabase-native; alternative backends would require
replacing repositories/functions only (the layer under the UI).
