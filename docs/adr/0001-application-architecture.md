# ADR 0001 — Feature-First Flutter Application Architecture

- **Status**: Proposed (approved as the working model; marked Accepted after
  Phase 1/5 implementation matches it).
- **Date**: 2026-10-24

## Context

The repository is a starter Flutter template. The team (LLM + human) will build
a 15-screen e-commerce app; without an agreed structure, agents tend to create
inconsistent layouts, duplicate models, or a monolith `main.dart`.

## Decision

Adopt a **feature-first modular architecture** with three optional layers per
feature (`data/`, `domain/`, `presentation/`) and shared cross-cutting code in
`core/` (`config`, `errors`, `theme`, `utils`, `widgets`). Layering rules:
presentation → application logic (Riverpod providers) → repositories → Supabase
client/edge functions. See `docs/ARCHITECTURE.md` and
`docs/APPLICATION_STRUCTURE.md`.

## Alternatives considered

- Layered "clean architecture" with use cases per feature — rejected as
  overkill for this app size.
- All-models-in-`models/` flat layout — rejected (couples features).
- Single `lib/pages/` monolith — rejected (not maintainable).

## Advantages

Small PRs; agents can reason about a feature from one folder; shared rules are
explicit in `core/`; testing boundaries are clear.

## Disadvantages

Folder naming must be policed (agents invent folders); some small features have
near-empty `domain/` folders — mitigated by "only create layers you use".

## Security/maintenance implications

Centralizing provider repos behind feature boundaries keeps privileged DB writes
out of widgets. No architectural change weakens RLS.

## Consequences for future implementation

Every feature follows this shape; adding a feature without touching the
conventions file is a doc miss. Revisit if the app grows a real admin panel
(separate, server-gated module).
