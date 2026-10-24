# Environment Setup (Windows)

Reproducible developer setup. Placeholders only — no real credentials.

## 1. Required tools

- **Flutter SDK 3.47.8 (stable)** with Dart 3.13.7. Verified installed
  (`flutter --version` on 2026-10-24) on this machine.
- **Android Studio** (or Android SDK + cmdline tools) and an emulator/device.
- **Supabase CLI** (`npm install -g supabase`) for local functions/migrations.
  Version as installed; minimum: the release that matches your hosted project.
- **Git** (present), PowerShell (default shell).
- Optional: VSCode extensions for Dart/Flutter and Supabase.

## 2. Verify the toolchain

```powershell
flutter --version      # expects Flutter 3.47.x, Dart 3.13.x
dart --version
flutter doctor -v      # Android toolchain OK
supabase --version     # after installing the CLI
```

## 3. Flutter project commands

Every Dart/Flutter command runs from **`bmc_final/`**:

```powershell
cd bmc_final
flutter pub get
flutter analyze
flutter test
dart format --output=none --set-exit-if-changed .
flutter build apk --debug     # quick Android smoke build
```

## 4. Supabase project configuration

1. Create the project in the Supabase Dashboard (region near PH recommended).
2. Settings → API: note `SUPABASE_URL` and the **anon/publishable** key.
3. For local work: `supabase init` (creates `supabase/`), then put migration
   files under `supabase/migrations/`.
4. Run migrations locally: `supabase start` (Docker required), then
   `supabase db reset` to replay migrations.
5. Apply to hosted later: `supabase link --project-ref <ref>` then
   `supabase db push`.

## 5. App configuration (client, publishable only)

Client config is injected at build/run time:

```powershell
flutter run --dart-define=SUPABASE_URL=https://<ref>.supabase.co `
  --dart-define=SUPABASE_ANON_KEY=<anon key>
```

`core/config/app_config.dart` (Phase 1) fails fast if these are missing in debug.

## 6. Edge Function development

```powershell
cd <repo root>
supabase functions new create-checkout-session   # creates supabase/functions/...
# local secrets (gitignored):
#   supabase/functions/.env
#     XENDIT_SECRET_KEY=xnd_development_...
#     XENDIT_WEBHOOK_VERIFICATION_TOKEN=...
supabase functions serve create-checkout-session --env-file supabase/functions/.env
```

Hosted secrets:

```powershell
supabase secrets set --env-file supabase/functions/.env.production
```

Supabase injects `SUPABASE_URL` and `SUPABASE_SECRET_KEYS` automatically; read
user JWTs via `createClient(..., { global: { headers: { Authorization: req.headers['authorization'] } } })`
and `auth.getUser()`.

## 7. Xendit test-mode setup

1. Register at Xendit Dashboard; create a **test** integration (or toggle test
   mode). Development API keys look like `xnd_development_...`.
2. Dashboard → Webhooks: register the webhook URL for the environment and copy
   the verification token (the `x-callback-token` payload).
3. Activate the required PH payment channels (e.g. GCash) in the Dashboard;
   note that activation is account-dependent.
4. Test cards / test wallets: use Xendit's documented test-mode values to
   simulate successful and failed captures (usually suffix amounts, e.g.
   ₱…01-style flags per the current docs — always read the current test guide).

## 8. Webhook development and testing

- Produce local endpoint reachable by the provider via one of:
  - hosted Supabase Edge Function deploy for the dev project
    (`supabase functions deploy xendit-webhook --project-ref <dev-ref>`);
  - or a tunneling tool (ngrok) pointed at `supabase functions serve`
    (never expose local secrets via a public tunnel in production).
- Verify: send a signed test event (Dashboard → Webhooks → resend/test) and
  confirm table state with SQL in Dashboard's SQL editor.

## 9. Database migrations

```powershell
supabase db reset                 # replay all migrations locally (drops dev data)
supabase db push                  # apply pending migrations to linked project
```

Migration naming: `<timestamp>_<snake_case_description>.sql`.

## 10. Build and test commands (summary)

| Action | Command (from bmc_final unless noted) |
|---|---|
| Get deps | `flutter pub get` |
| Analyze | `flutter analyze` |
| Format check | `dart format --output=none --set-exit-if-changed .` |
| Unit/widget tests | `flutter test` |
| Coverage (optional) | `flutter test --coverage` |
| Android debug build | `flutter build apk --debug` |
| Android release build | `flutter build apk --release` |
| Local functions | `supabase functions serve <name> --env-file supabase/functions/.env` |
| Deploy functions | `supabase functions deploy <name> --project-ref <ref>` |
| Replay migrations | `supabase db reset` / `supabase db push` |

## 11. Common failures and troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `flutter analyze` fails on dart-define | missing client config | run with the dart-defines above |
| Android build needs INTERNET permission | default template lacks it | add `<uses-permission android:name="android.permission.INTERNET"/>` to the manifest (Phase 1) |
| Supabase function can't read secret | local `.env` not at `supabase/functions/.env` | place it there; `supabase functions serve --env-file ...` |
| Webhook 401s | token mismatch | re-copy the callback token from Dashboard and set the env |
| Xendit `INVALID_PAYMENT_CHANNEL` | channel not activated for account | activate in Dashboard test mode |
| Xendit `INVALID_AMOUNT` | amount representation wrong | verify minor-units conversion + decimals for PHP (Phase 6) |
| CORS on web tests | web not a release target | ignore for Android-first; configure if web builds are added |

## 12. Recommended IDE aliases and settings

- Set line length 100 for Dart formatting (repository-selected).
- Windows paths use backslashes in PowerShell; project docs use forward slashes
  in path references.
