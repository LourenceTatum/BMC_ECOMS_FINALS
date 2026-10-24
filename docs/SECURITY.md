# Security

What may live in the app, what must stay server-side, and how secrets are
managed. Non-negotiable: **no provider or service-role secret ever ships in the
mobile app.**

## 1. Public client configuration (safe in the app)

The following are considered **public publishable** values and may be compiled
into the app (typically via `--dart-define` or safe build-time config):
- Supabase project URL (`SUPABASE_URL`).
- Supabase **anon/publishable** key (`SUPABASE_ANON_KEY`) — safe only because
  RLS + grants enforce access. A leaked anon key lets someone call the public
  API *as an anonymous/own-user* principal, nothing more.
- Deep-link scheme identifiers (e.g. `bmcapp://`).

**Never** in the app: Supabase service-role/secret keys, `SUPABASE_SECRET_KEYS`,
Xendit secret API keys, Xendit webhook verification tokens, or any database
credentials.

## 2. Server-only secrets

| Secret | Where it lives |
|---|---|
| `XENDIT_SECRET_KEY` | Edge Function secret (`supabase secrets set` / local `supabase/functions/.env`) |
| `XENDIT_WEBHOOK_VERIFICATION_TOKEN` | Edge Function secret |
| Supabase service-role key | Edge Function (available by default as `SUPABASE_SECRET_KEYS`/`SUPABASE_SERVICE_ROLE_KEY`) — used only where RLS would be bypassed deliberately (webhook writes) |

## 3. Loading environment variables

- **Flutter**: no automatic `.env` loading. Client values are injected via
  `--dart-define=SUPABASE_URL=... --dart-define=SUPABASE_ANON_KEY=...` (and an
  `app_config.dart` that validates presence, failing fast with a clear message in
  debug builds). A committed `.env.example` documents the placeholders only.
- **Edge Functions (local)**: `supabase/functions/.env` that `supabase start`
  loads; gitignored (root `.gitignore` lines include it).
- **Edge Functions (hosted)**: `supabase secrets set XENDIT_SECRET_KEY=...`.

## 4. `.gitignore` protection

- Root `.gitignore` ignores `.env`, `.env.*` (except `*.env.example`),
  `supabase/functions/.env`, `.supabase/`, and key materials. **Verify with
  `git status` after adding any env file** — never commit values.

## 5. Database defense

- RLS + explicit grants per table: `DATABASE_SECURITY.md`.
- Clients can read own orders/payments but can never INSERT/UPDATE those tables;
  privileged transitions are Edge-Function-only (conditional SQL inside
  transactions).
- Profile fields are never authorization inputs (`DATABASE_SECURITY.md` §13).

## 6. Webhook verification

- The xendit-webhook endpoint has no user-session auth (the provider cannot
  hold one), so its **only** authentication is the callback token
  (`x-callback-token`), compared constant-time against a server secret. No
  alternative unauthenticated path exists; unverified events are rejected 401.

## 7. Logs and diagnostics

- Logs carry identifiers and error codes only. No tokens, passwords, PANs,
  or full provider payloads (store masked provider payload in `raw_payload`
  fields with access restricted to server roles).
- Access to Supabase Edge Function logs is limited to project admins.

## 8. Test vs. live separation

- Development uses Xendit **test mode** (`xnd_development_*` key) and a Supabase
  dev project. Live (`xnd_production_*`) only after E2E test-mode sign-off
  (`TESTING_STRATEGY.md`, `DEPLOYMENT.md`).
- Credentials are never shared between phases except by deliberate secret
  promotion during go-live (documented step, not an env merge).

## 9. Build-time hygiene

- Before any release build: `flutter build apk --release` and inspect the APK
  for secret strings; run the checklist in `DEPLOYMENT.md` (search for
  `xnd_`, `service_role`, `SUPABASE_SERVICE_ROLE_KEY` in artifacts).
