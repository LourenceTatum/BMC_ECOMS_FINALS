---
name: testing-and-quality-assurance
description: Define and run tests for BMC Commerce. Use when writing or updating
  unit, widget, database, or end-to-end tests, or before finishing any task that
  changes behavior. Covers test-first habits, covering positive and negative cases,
  running format/analyze/test/build gates, and reporting results honestly.
---

# Testing and quality assurance

Policy: `docs/TESTING_STRATEGY.md`. Gates and honesty rules: `docs/DEFINITION_OF_DONE.md`.

## When to use
- Before implementing anything with logic worth regressing.
- After any change to money math, state machines, RLS, or provider flows.
- Before declaring a task finished.

## Define cases first
For each feature identify at minimum:
- **positive**: happy path renders/transitions correctly;
- **negative**: validation failures, unauthorized access, sold-out, mismatch,
  duplicate events;
- **edge**: zero quantity, negative, max stock, empty result sets, unknown
  statuses, network errors.

## Which layer
| Kind | Where | Covers |
|---|---|---|
| Unit | `bmc_final/test/unit/` | money formatting, validators, cart math, status mapping, controllers with fake repos |
| Widget | `bmc_final/test/widget/` | flows and states; provider overrides + fake repositories only (no network) |
| Database | `supabase/tests/` (local stack) | RLS allowed/denied, idempotency, inventory races, webhook dedupe, invalid transitions |
| E2E | documented smoke scenario | real test-mode provider payments (manual + scripted steps, provider-dependent) |

## Gates (run from `bmc_final/`, in order)
1. `dart format --output=none --set-exit-if-changed .`
2. `flutter analyze` — zero new issues in changed code.
3. `flutter test` — all tests pass; never delete/weaken a failing assertion to
   force green. If a test is wrong, fix the test with an explicit comment.
4. DB tests against `supabase start` when migrations/RLS changed.
5. `git diff --check`.

## Honest reporting
- Separate **executed** tests (with exact commands and results) from
  **planned/manual** steps.
- E2E provider results require real evidence: test-mode payment completed,
  webhook processed, DB state verified, app displayed it.
- If a platform/account secret is missing, say so and list the manual setup step
  instead of fabricating a pass.
