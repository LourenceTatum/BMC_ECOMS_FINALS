---
name: supabase-database-security
description: Work with the BMC Commerce Supabase database safely. Use when creating
  or editing migrations, tables, Row Level Security policies, grants, indexes,
  storage buckets, or database functions. Covers version-controlled migrations,
  privilege correctness, ownership checks, and testing both permitted and
  prohibited access paths.
---

# Supabase database security

Rules for schema and security work in `supabase/`.

## Before touching the database
1. Read `docs/DATABASE_SCHEMA.md` (canonical tables/shapes/state machines) and
   `docs/DATABASE_SECURITY.md` (grants + RLS per table).
2. Check `supabase/migrations/` for the latest migration before writing a new
   one. Rebase your numbers off the newest timestamp; name files
   `<timestamp>_<snake_case>.sql`.
3. Confirm the task's acceptance criteria and which data-access paths it touches.

## Migration and RLS rules
1. **All schema lives in version-controlled migrations.** Never rely on ad-hoc
   Dashboard SQL edits as the source of truth.
2. For every public table:
   - `alter table ... enable row level security;`
   - explicit `GRANT`s (RLS is not a substitute for privileges);
   - policies with a name that states the purpose and a comment noting the
     abuse case it prevents.
3. Ownership scoping: `using (id = auth.uid())`-style predicates on all
   user-owned rows. Detective checks: anonymous must not read drafts; users
   must not read/write other users' rows; the client must have **no**
   INSERT/UPDATE/DELETE grants on `orders`, `payments`, or
   `inventory_reservations`.
4. Storage: private buckets with path-prefix policies; server-side uploads where
   sensible (`docs/DATABASE_SECURITY.md` §14).
5. Use `SECURITY DEFINER` rarely and only with internal `auth.uid()` checks and
   `REVOKE ... ; GRANT EXECUTE ...` hygiene. Never authorize via client-editable
   columns.

## Money and integrity
- Monetary columns: `numeric(12,2)`; CHECK constraints (>=0, currency='PHP'
   where relevant). States carry CHECK constraints matching the documented
   vocabularies (`docs/DATABASE_SCHEMA.md` §14).
- Unique constraints the app depends on: `orders.idempotency_key`,
   `payments.provider_event_id`, `carts.user_id`, `cart_items(cart_id,
   variant_id)`, `favorites(user_id, product_id)`.

## Test both sides
- Write and run DB tests for permitted AND prohibited access paths
   (`docs/TESTING_STRATEGY.md` §4) against the local stack:
   `supabase start` then `supabase db reset`, then run the test scripts.
- Include: anon catalog reads OK but drafts blocked; cross-user order access
   blocked; client `UPDATE orders SET payment_status='paid'` fails; duplicate
   idempotency key yields one order; concurrent last-unit purchase yields one
   winner; replaying a webhook event id is a no-op.

## Finish
- `supabase db push --dry-run` (or `db diff`) reviewed before applying.
- Update `docs/DATABASE_SCHEMA.md` / `docs/DATABASE_SECURITY.md` if the design
   changed, and note it in the report.
