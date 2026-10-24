# Database Security (RLS + Grants)

Defines *who can do what* on every table. **RLS is not a substitute for
correct `GRANT`s** — both are specified here. This is a design spec; actual
migrations land in Phase 2 and are verified with the tests in
`TESTING_STRATEGY.md`.

## 0. Roles and key distribution

- `anon` — unauthenticated PostgREST role (public catalog reads).
- `authenticated` — any signed-in user (own-data reads/writes).
- `service_role` — **server-only** (Edge Functions), bypasses RLS; used for
  privileged mutations. Never exposed to the app.
- Database functions used by clients are `SECURITY DEFINER` only when they must
  act on protected internal state, and then only if they enforce `auth.uid()`
  checks internally and grants are limited to the exact `execute` needed.

### Abuse cases each policy group prevents

1. One user reading/altering another user's orders, cart, addresses, profile.
2. Anonymous visitors mutating cart/order state.
3. A client marking its own order `paid` or flipping payment status.
4. A client adjusting stock, reservations, or payments.
5. A client reading draft/archived catalog rows (premature content leaks).
6. Duplicate order creation (unique `idempotency_key` + server validation).
7. Profile-field privilege escalation (see §13).

## 1. profiles

- `GRANT SELECT, UPDATE ON profiles TO authenticated;`
- RLS enabled. Policies:
  - `"users select own profile"` — `for select to authenticated
    using (id = auth.uid())`.
  - `"users update own profile"` — `for update ... using (id = auth.uid())
    with check (id = auth.uid())`.
- No client insert (profile row created by a signup trigger/function using
  `service_role`). No anon access.

## 2. categories

- `GRANT SELECT ON categories TO anon, authenticated;`
- Policy: `"public read active categories"` — `for select using (is_active = true)`.

## 3. products

- `GRANT SELECT ON products TO anon, authenticated;`
- Policy: `"public read active products"` — `for select using (status = 'active')`.

## 4. product_images

- `GRANT SELECT ... TO anon, authenticated;`
- Policy: `"public read images of visible products"` — select where
  `exists (select 1 from products where id = product_id and status = 'active')`.
- Images of draft/archived products stay invisible.

## 5. product_variants

- `GRANT SELECT ... TO anon, authenticated;`
- Policy: select only when the parent product is `active` **and**
  `product_variants.is_active`.
- `stock_qty` is intentionally visible ("Only 3 left"); if hiding exact counts
  becomes necessary, record it as a Phase 2 policy review decision.

## 6. addresses

- `GRANT SELECT, INSERT, UPDATE, DELETE ON addresses TO authenticated;`
- Policies (all constrain to `user_id = auth.uid()`): select own, insert own
  (`with check user_id = auth.uid()`), update own, delete own.
- `is_default` uniqueness enforced per user (partial unique index).

## 7. carts / cart_items

- `GRANT SELECT, INSERT, UPDATE, DELETE ON carts, cart_items TO authenticated;`
- Policies: cart ops own (`user_id = auth.uid()`); cart_items through own cart
  (`exists (select 1 from carts where id = cart_id and user_id = auth.uid())`).
- No price columns exist here, so there is no price-manipulation surface;
  quantities are validated/clamped server-side at checkout.

## 8. favorites

- `GRANT SELECT, INSERT, DELETE ON favorites TO authenticated;`
- Policies: `user_id = auth.uid()` for select/insert/delete (update not needed;
  the row is deleted and recreated).

## 9. orders

- `GRANT SELECT ON orders TO authenticated;`
- Policy: `"users read own orders"` — `for select using (user_id = auth.uid())`.
- **No client INSERT/UPDATE/DELETE granted.** Orders are created and mutated
  only by Edge Functions (service_role, or `SECURITY DEFINER` RPCs limited to
  server callers). This is the core guarantee that clients cannot self-mark paid.

## 10. order_items

- `GRANT SELECT ON order_items TO authenticated;`
- Policy: select through own order:
  `exists (select 1 from orders where id = order_id and user_id = auth.uid())`.
- No client write.

## 11. payments

- `GRANT SELECT ON payments TO authenticated;`
- Policy: `"users read payments of own orders"` — select via the owning order
  (`exists (select 1 from orders where id = order_id and user_id = auth.uid())`).
- No client write. Status/amount mutations happen only inside the webhook
  function's transaction.

## 12. inventory_reservations

- No client grants (`revoke all on inventory_reservations from anon, authenticated`).
  Server-only. Availability is exposed indirectly through the catalog; the
  reservation ledger is private.

## 13. Privilege-escalation guard

- Client-editable profile fields (`full_name`, `phone`, `avatar_url`, …) are
  **never** used in RLS policies, never grant roles, and never act as
  authorization flags. Authorization always derives from `auth.uid()` in the
  verified JWT.
- No `SECURITY DEFINER` function may consult client-provided role/flag columns
  for authorization.

## 14. Storage buckets

- `product-images`: **proposed** private bucket; product image URLs surfaced as
  cached/public URLs stored in `product_images.url`. Uploads are server-side only
  (seed/admin flows in Phase 2) — no client upload in MVP.
- `avatars`: private bucket; user uploads to their own folder `{user_id}/...`
  via signed upload URL; read policy constrains to path prefix matches
  `auth.uid()`. Final path-prefix policy is recorded in the Phase 2 migration.
- Storage RLS (when applicable) mirrors table RLS semantics; deny-by-default.

## 15. Required verification (Phase 2 + after every security change)

Per `TESTING_STRATEGY.md` database tests: anon cannot read draft products;
user B cannot read user A's orders — RLS returns 0 rows, and any direct-access
RPC must raise an authorization error (not silently hide); unauthenticated cart
writes fail; a direct client `UPDATE orders SET payment_status='paid'` fails at
privilege level; webhook dedupe prevents double `paid`.
