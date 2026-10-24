# BMC Commerce — Gothic Luxury Streetwear Store (Engineering Hub)

Flutter e-commerce application for the Philippine market (currency **PHP**),
built on **Supabase** (Auth, PostgreSQL + RLS, Edge Functions, Storage) and
**Xendit** hosted checkout. Aesthetic: gothic luxury streetwear inspired by
Chrome Hearts — clearly as a design direction, never as an affiliation.

> **Status: engineering foundation.** The repository currently contains a starter
> Flutter template in `bmc_final/`. The documents in `docs/` and the skills in
> `.claude/skills/` are the working agreement for implementing the product phase
> by phase (see `docs/IMPLEMENTATION_ROADMAP.md`).

## Repo map

| Path | What it is |
|---|---|
| `AGENTS.md` | **Engineering rulebook. Read first.** |
| `CLAUDE.md` | Claude Code operating guidance |
| `docs/` | All specifications (architecture, database, payments, UI/UX, testing, roadmap) |
| `docs/adr/` | Architecture decision records |
| `.claude/skills/` | Claude Code project skills (5 skills) |
| `bmc_final/` | The Flutter application |
| `.env.example` | Placeholder-only environment template |

## Quick start (development on Windows)

Prereqs are documented in `docs/ENVIRONMENT_SETUP.md`. Minimum commands:

```powershell
cd bmc_final
flutter pub get
flutter analyze
flutter test
```

## Key documents

- Project overview: `docs/PROJECT_OVERVIEW.md`
- Requirements (acceptance criteria): `docs/REQUIREMENTS.md`
- Architecture & app structure: `docs/ARCHITECTURE.md`, `docs/APPLICATION_STRUCTURE.md`
- Database: `docs/DATABASE_SCHEMA.md`, `docs/DATABASE_SECURITY.md`
- Payments (critical): `docs/PAYMENT_ARCHITECTURE.md`, `docs/API_CONTRACTS.md`
- Design system: `docs/DESIGN_SYSTEM.md`
- Security & env: `docs/SECURITY.md`, `docs/ENVIRONMENT_SETUP.md`
- Rollout plan: `docs/IMPLEMENTATION_ROADMAP.md`, `docs/DEFINITION_OF_DONE.md`

## Contribution / agent workflow

Read `AGENTS.md` first. Follow the section "Feature implementation workflow" and
respect the scope-control rules. Validate with `git diff --check`, `dart format`,
`flutter analyze`, and relevant tests before finishing.
