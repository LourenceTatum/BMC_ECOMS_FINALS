# ADR 0004 — Xendit Hosted Checkout with Server-Side Payment Processing

- **Status**: Proposed (contracts to be re-validated at Phase 6 implementation).
- **Date**: 2026-10-24

## Context

The app must take real PHP payments in the Philippines. The Xendit API surface
has two candidate paths: **Payment Sessions** (`POST /sessions` with
`mode: PAYMENT_LINK`, hosted page) and the older Payment Request flow
(`v3/payment_requests`). A wrong integration risks insecure client-side keys or
charging with a mis-formatted amount.

## Decision

- Use **Xendit Payment Sessions, `session_type: PAY`, `mode: PAYMENT_LINK`**:
  Xendit hosts the checkout; the server creates the session with a
  server-computed PHP total; success/failure arrive as **webhooks**.
- **No Xendit API call or secret key ever exists in the Flutter app.** All
  provider calls happen in authenticated Edge Functions.
- **Payment completion is confirmed only by a webhook** whose authenticity is
  verified via the `x-callback-token` header (constant-time compare), then
  applied idempotently with conditional DB transitions.
- Amount conversion (`numeric(12,2)` → provider representation) is centralized
  in one server helper and **validated with a test-mode session** in Phase 6
  because the Session examples use integer minor units for PHP while the
  `v3/payment_requests` API uses decimals.
- If Sessions/PAYMENT_LINK proves unsupported for the activated account/market
  at Phase 6, switch to the officially supported equivalent (documented in
  `PAYMENT_ARCHITECTURE.md` §1) keeping the same security model.

## Alternatives considered

- Direct card capture in the app — rejected (PCI scope, secret hygiene).
- Client-side session creation with the anon role — rejected (would require
  provider keys client-side).
- Payment Requests v3 hosted action — viable fallback; held in reserve.

## Advantages

PCI burden on Xendit; minimal client surface; clear audit trail of payment →
webhook → order; strongly idempotent.

## Disadvantages

Async dependency on webhooks (training UX to not trust redirects); docs churn
(mitigated by Phase 6 verification step); test-mode ≠ live (mitigated by a
formal E2E gate).

## Security implications

The webhook endpoint is the highest-risk surface: unauthenticated at the JWT
level and **must** verify the callback token, reject mismatches, dedupe on a
unique event id, and enforce legal transitions. Amount/reference mismatches
never mutate state.

## Consequences

Checkout creation and status changes are server-only; the app polls/refreshes
server state; reconciliation + refund (Phase 7) are server-only. Revisit this
decision if Xendit changes the Session API contract or deprecates PAYMENT_LINK.
