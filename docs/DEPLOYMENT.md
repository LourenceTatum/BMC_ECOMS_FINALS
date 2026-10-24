# Deployment

Go-live workflow for Android, Supabase, Edge Functions, secrets, and webhooks.
Production specifics are kept as a **checklist** until actually configured —
documenting a step here is not evidence it was done.

## 0. Principles

- Deploy in the same order as the roadmap: DB → functions → app → webhook config.
- Test-mode first, production after the smoke E2E passes.
- Every environment (dev/staging/prod) has its own Supabase project keys and
  Xendit credentials; never share or merge them.

## 1. Supabase database migrations

```powershell
supabase link --project-ref <prod-ref>
supabase db push                      # apply pending migrations
supabase db push --dry-run            # preview diff first (supported CLI)
```

- Migrations are additive and reviewed (`git diff`) before push; destructive
  changes (drops) require a documented rollback plan.
- RLS/grants live in migrations — never patch policies only in the Dashboard.

## 2. Edge Functions

```powershell
supabase functions deploy create-checkout-session --project-ref <ref>
supabase functions deploy xendit-webhook            --project-ref <ref>
```

- Deploy after the DB functions/tables they depend on exist.
- Secrets must be set **before** the first function call:
  `supabase secrets set --env-file supabase/functions/.env.production`.

## 3. Server-side secrets checklist

- [ ] `XENDIT_SECRET_KEY` = production key (`xnd_production_...`)
- [ ] `XENDIT_WEBHOOK_VERIFICATION_TOKEN` = production webhook token
- [ ] Service-role access used only inside functions (auto-injected)
- [ ] Function logs scanned for accidental secret output after a dry run

## 4. Xendit webhook registration

- [ ] In the Xendit Dashboard (production), register the public HTTPS URL:
      `https://<project-ref>.supabase.co/functions/v1/xendit-webhook`
- [ ] Copy the callback token into the function secret (same value that signs
      requests).
- [ ] Verify from Dashboard: trigger a test event → inspect function logs and
      DB rows (expected transitions).
- [ ] Confirm retry behavior: a 2xx ack in <timeout> with async processing.

## 5. Flutter Android builds

```powershell
cd bmc_final
flutter build apk --release           # or --split-per-abi / appbundle
```

Pre-release checklist:

- [ ] `applicationId` changed from `com.example.bmc_final` to the owned domain
  key (update `android/app/build.gradle.kts` and namespaces) — **required**.
- [ ] `android:label` set to the real app name; INTERNET permission present.
- [ ] Release signing configured (remove debug-signing config) + keystore
  secrets kept out of git.
- [ ] Client config injected via `--dart-define`; **grep the APK** for
  `xnd_`, `service_role`, `supabase_secret_keys` (must be absent).
- [ ] Deep-link scheme manifest filters for `bmcapp://` and auth redirects.
- [ ] `flutter test` + `flutter analyze` green on the tagged commit.

## 6. Production environment config

- `SUPABASE_URL` / `SUPABASE_ANON_KEY` for production project.
- Auth: email confirmation **enabled**, redirect URLs configured, recovery
  deep links verified on a real device.
- Storage buckets created with the RLS policies from `DATABASE_SECURITY.md`.

## 7. Smoke tests after deployment (production, soft-launch)

1. Register + verify email + login.
2. Browse catalog (seeded production-correct data only — placeholder products
   are clearly labeled or removed).
3. Buy Now for a low-value item with a production-provider action.
4. Confirm webhook → DB → app order status `paid`.
5. Cancel/expired path: initiate, abandon, confirm consistency (no stock leak).

## 8. Rollback & payment reconciliation

- **Database**: migrations are additive; rollback = deploy a reverse migration
  or (better) forward-fix; never drop paid-order rows.
- **Functions**: previous deployed function version can be redeployed; keep the
  last-good version tag.
- **Payments**: reconciliation job (Phase 7) catches provider/DB divergence;
  refund path for accidental double-charges is Phase 7+.
- Documented incident drill: webhook outage → Xendit Dashboard re-trigger →
  idempotent reprocessing (safe by design).

## 9. Post-go-live monitoring

- Supabase function logs + error codes; Xendit Dashboard transaction reports.
- Weekly reconciliation spot-check until the automated job is proven stable.
