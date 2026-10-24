# ADR 0002 — State Management, Dependency Injection, and Routing

- **Status**: Proposed (dispatched in Phase 1).
- **Date**: 2026-10-24

## Context

The template has no state management or routing. Choices here affect every
screen and agent task, so they should be pinned early and rarely changed.

## Decision

- **State + DI**: Riverpod (`flutter_riverpod` 3.x) — manual providers; no
  code-gen stack. Async states modeled with `AsyncNotifier`/`FutureProvider`;
  controller providers for interactive flows.
- **Routing**: go_router (18.x) — declarative path routes with a `redirect`
  guard driven by a session provider, `ShellRoute` for the bottom-nav shell.
- Supabase client exposed as a `Provider<SupabaseClient>`; repositories are
  providers too (overridable in tests).

## Alternatives considered

- Provider/Bloc — valid, but Riverpod wins on testability + the team's reduced
  boilerplate preference; adding Bloc later would be a large rework.
- `go_router` vs `auto_route` — go_router is official, no code-gen, redirect
  guards fit auth streaming; auto_route adds generation and was rejected.
- Code-gen riverpod (`@riverpod`) — deferred; manual is sufficient at this
  size and keeps the build simple.

## Advantages

Single state model + single router = low context-switch for agents; easy test
overrides; auth-aware redirects.

## Disadvantages

Riverpod boilerplate for tiny screens; API churn between major versions
(mitigated by pinning to stable and upgrading deliberately).

## Security/maintenance implications

Route guards are UX, not enforcement: RLS/server still enforce authorization
(state the dependency explicitly so agents never rely on guards alone).

## Consequences

New features use these providers/routes; introducing a second state manager or
router requires a new ADR. Revisit only on a major package migration.
