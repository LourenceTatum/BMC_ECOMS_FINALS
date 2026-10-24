# CLAUDE.md — Claude Code operating guidance

This file tells Claude Code how to operate in this repository. It is intentionally
short: the project's engineering authority is **`AGENTS.md`** (same directory).

## Before you start any session

1. Read `AGENTS.md` in full. It governs architecture, conventions,
   database/payment security rules, testing, and scope control.
2. Read `docs/PROJECT_OVERVIEW.md` for the product and current state, then read
   only the topic docs relevant to the task you were given.
3. Load the matching project skill from `.claude/skills/` when one applies
   (flutter-feature-development, flutter-ui-design, supabase-database-security,
   xendit-payment-integration, testing-and-quality-assurance).
4. Check `git status` and `git log --oneline -5` so you know what is committed
   and what may be work-in-progress.

## Repository facts you need while operating

- The Flutter app is **`bmc_final/`**; the Git/workspace root is the repo root.
  Run `cd bmc_final` before `flutter`/`dart` commands.
- The app is a starter Flutter template at this stage. `docs/` is all
  **specification** — nothing in `docs/` means a feature already exists.
- The `IMPLEMENTATION_ROADMAP.md` defines phases; do not jump ahead of the
  currently assigned phase without explicit instruction.
- Never fabricate credentials, test results, or external configuration. If an
  account/secret/webhook setting is missing, document the manual setup step.

## Claude-specific behaviors

- Keep changes minimal and scoped; prefer the smallest change that satisfies the
  requirement correctly.
- Do not "fix" things unrelated to the assigned task, and never loosen RLS,
  remove webhook verification, or weaken tests to get a green check.
- When a task changes a contract covered by `docs/`, update the document.
- Prefer `/analysis`… no: use the normal workflow — `flutter analyze`, `dart
  format`, `flutter test` from `bmc_final/` — and report actual results.

This file must never contradict `AGENTS.md`. If you find a conflict, `AGENTS.md`
wins; file a doc update rather than working around it.
