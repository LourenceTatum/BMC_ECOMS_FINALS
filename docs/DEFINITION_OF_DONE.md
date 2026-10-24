# Definition of Done

Every task — from a one-file fix to a roadmap phase — is accepted only when
**all** of the following hold. Agents must report each item honestly, including
anything not verified.

## 1. Functionality

- [ ] Code exists, is wired into the router/providers, and matches the
      documented contract (`docs/REQUIREMENTS.md`, `docs/API_CONTRACTS.md`).
- [ ] Acceptance criteria for the task are met and demonstrable.
- [ ] Existing functionality still passes its tests (no regressions).

## 2. Security

- [ ] RLS + grants unchanged-or-verified for any touched table
      (`docs/DATABASE_SECURITY.md`); no new privilege-escalation path.
- [ ] No secret material in source, assets, config, or docs.
- [ ] No client path can mark an order paid, mutate payments, or alter stock.
- [ ] Webhooks, if touched, still verify tokens, dedupe, and enforce legal
      transitions.
- [ ] Money uses exact arithmetic only (never `double` at boundaries).

## 3. UX states

- [ ] Every affected async screen handles loading, error, data, and empty states
      per `docs/ERROR_HANDLING.md` and `docs/DESIGN_SYSTEM.md`.

## 4. Quality gates (actually executed)

- [ ] `dart format --output=none --set-exit-if-changed .` (from `bmc_final/`)
      passes.
- [ ] `flutter analyze` passes with **zero** warnings/errors in changed code.
- [ ] Relevant tests were written, executed, and **passed** (`flutter test`
      and/or local-stack DB tests).
- [ ] No assertions were weakened, and no RLS/security checks were removed, to
      obtain green.

## 5. Documentation & diff

- [ ] `docs/` updated when a contract, schema, state machine, env var, or
      design token changed.
- [ ] `docs/README.md` consistency rules satisfied.
- [ ] `git diff --check` is clean; final `git status` matches the report.
- [ ] No unrelated refactors, added dependencies, or speculative files.

## 6. Honest reporting

- [ ] Report lists files changed, commands executed + results, failures, and
      limitations.
- [ ] Untested or unverified items are explicitly labeled as such.
- [ ] Manual provider steps (accounts, webhooks, test-mode actions) are listed
      as outstanding, never claimed as done.
