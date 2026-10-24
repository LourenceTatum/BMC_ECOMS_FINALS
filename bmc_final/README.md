# bmc_final — BMC Commerce Flutter App

The Flutter client for the **BMC Commerce** gothic luxury streetwear store
(Philippines · PHP · Supabase · Xendit).

> **Engineering foundation**: the repository root (`../`) contains the project's
> governing docs. **Read `../AGENTS.md` before working in this app**, then
> `../docs/` as needed. Implementation is phased — see
> `../docs/IMPLEMENTATION_ROADMAP.md`. The current code is the Flutter starter
> template; the app itself is built in later phases.

## Quick start (Windows)

```powershell
flutter pub get
flutter run --dart-define=SUPABASE_URL=https://<ref>.supabase.co `
  --dart-define=SUPABASE_ANON_KEY=<anon-key>
```

Client config (URL + anon key) is **publishable only** — see
`../docs/SECURITY.md`. No provider or service-role secrets belong here.

## Checks

```powershell
dart format .
flutter analyze
flutter test
```

## References

- Engineering rulebook: `../AGENTS.md` · Claude guidance: `../CLAUDE.md`
- Project overview: `../docs/PROJECT_OVERVIEW.md`
- Features/UI/architecture/database/payments docs: `../docs/`
- Skills: `../.claude/skills/`
