# Database Schema (Supabase PostgreSQL)

Canonical relational model for BMC Commerce. All schema changes are applied as
**version-controlled SQL migrations** under `supabase/migrations/` (Phase 2).
These are design specifications; no table is created until Phase 2.

## 0. Conventions

- Natural keys: UUIDs (`gen_random_uuid()`). Primary keys are `uuid` unless noted.
- All tables include `created_at timestamptz not null default now()` and, where
  mutable, `updated_at timestamptz not null default now()`.
- `auth.users` is the identity source; customer rows reference it, never duplicate
  credentials.
- **Money**: monetary columns are `numeric(12,2)` — PHP with 2 decimals. Never
  `float`/`double`. All totals are computed and validated on the server.
- `currency char(3) check (currency = 'PHP')` on money-bearing tables for MVP.
- Statuses are `text` with CHECK constraints (or enums in functions); changing a
  status value requires a migration — prefer small, additive status sets.
- Every table documented as **client-readable** also needs RLS policies and
  grants (`DATABASE_SECURITY.md`).

## 1. profiles

Purpose: user-facing profile for each authenticated user.

| Column | Type | Notes |
|---|---|---|
| `id` | `uuid pk` | = `auth.users.id` (FK `auth.users(id)` on delete cascade) |
| `email` | `text not null` | copied at signup; display-only |
| `full_name` | `text` | editable |
| `phone` | `text` | PH mobile, client+server validated |
| `avatar_url` | `text` | Supabase storage public URL |
| `created_at`, `updated_at` | `timestamptz` | |

Access: user read/update own row. Never the source of truth for authorization —
only `auth.uid()` and RLS do that.

## 2. categories

| Column | Type | Notes |
|---|---|---|
| `id` | `uuid pk` | |
| `name` | `text not null` | |
| `slug` | `text not null unique` | URL/filter key |
| `description` | `text` | |
| `sort_order` | `int not null default 0` | |
| `is_active` | `boolean not null default true` | |
| timestamps | | |

Public read of `is_active = true` rows only.

## 3. products

| Column | Type | Notes |
|---|---|---|
| `id` | `uuid pk` | |
| `category_id` | `uuid fk → categories(id)` null | `set null` on delete |
| `name` | `text not null` | |
| `slug` | `text not null unique` | |
| `description` | `text` | editorial copy |
| `price` | `numeric(12,2) not null check (price >= 0)` | base unit price (PHP) |
| `compare_at_price` | `numeric(12,2) check (>= 0)` null | "was" price display;
  **not** a discount engine yet |
| `currency` | `char(3) not null check (currency='PHP')` | |
| `status` | `text not null check (status in ('draft','active','archived'))` | |
| `is_featured` | `boolean not null default false` | storefront rail |
| timestamps | | |

Indexes: `(status)`, `(category_id) where status='active'`, `slug` (unique).
Public reads: `status='active'` only.

## 4. product_images

| Column | Type | Notes |
|---|---|---|
| `id` | `uuid pk` | |
| `product_id` | `uuid fk → products(id) on delete cascade` | |
| `storage_path` | `text not null` | path in the `product-images` bucket |
| `url` | `text` | cached public URL (regenerable) |
| `alt_text` | `text` | accessibility |
| `sort_order` | `int not null default 0` | |
| `is_primary` | `boolean not null default false` | one per product |

## 5. product_variants

**A purchasable SKU is one `product_variants` row; stock is tracked per variant.**

| Column | Type | Notes |
|---|---|---|
| `id` | `uuid pk` | |
| `product_id` | `uuid fk → products(id) on delete cascade` | |
| `sku` | `text unique` | merchant SKU |
| `name` | `text not null` | e.g. "Black / Small" |
| `size` | `text` | nullable if not size-varies |
| `color` | `text` | nullable |
| `price_delta` | `numeric(12,2) not null default 0 | price = product.price + price_delta (server-side) |
| `stock_qty` | `int not null default 0 check (stock_qty >= 0)` | per-variant inventory |
| `is_active` | `boolean not null default true` | |
| timestamps | | |

Unique `(product_id, size, color)` (nullable-safe via `nulls not distinct` if a
product has no size but has colors). Index: `(product_id) where is_active`.
Public read: active variants joined to active products.

## 6. addresses

| Column | Type | Notes |
|---|---|---|
| `id` | `uuid pk` | |
| `user_id` | `uuid fk → profiles(id) on delete cascade` | |
| `label` | `text` | e.g. "Home" |
| `recipient_name` | `text not null` | |
| `phone` | `text not null` | PH format |
| `line1` | `text not null` | street/bldg |
| `line2` | `text` | unit/landmark |
| `barangay`, `city`, `province` | `text not null` | PH addressing |
| `postal_code` | `text not null` | |
| `country` | `text not null default 'PH' check (country='PH')` | MVP |
| `is_default` | `boolean not null default false` | one default per user |

## 7. carts and cart_items

| Column | Type | Notes |
|---|---|---|
| `carts.id` | `uuid pk` | |
| `carts.user_id` | `uuid fk → profiles(id) unique` | one cart per user |
| `cart_items.id` | `uuid pk` | |
| `cart_items.cart_id` | `uuid fk → carts(id) on delete cascade` | |
| `cart_items.product_id` | `uuid fk → products(id)` restrict | |
| `cart_items.variant_id` | `uuid fk → product_variants(id)` restrict | |
| `cart_items.quantity` | `int not null check (quantity between 1 and 99)` | |
| timestamps | | |

Unique `(cart_id, variant_id)`. Client may write own cart rows (RLS) but
quantity is server-clamped at checkout. The cart **never stores prices** — line
prices resolve from catalog at read/checkout time.

## 8. favorites

| Column | Type | Notes |
|---|---|---|
| `id` | `uuid pk` | |
| `user_id` | `uuid fk → profiles(id) on delete cascade` | |
| `product_id` | `uuid fk → products(id) on delete cascade` | |
| `created_at` | `timestamptz` | |

Unique `(user_id, product_id)`.

## 9. orders

| Column | Type | Notes |
|---|---|---|
| `id` | `uuid pk` | |
| `order_number` | `text not null unique` | human reference, e.g. `BMC-XXXXXX` |
| `user_id` | `uuid fk → profiles(id) restrict` | who owns the order |
| `status` | `text not null check (status in ('pending_payment','paid','processing','shipped','delivered','cancelled'))` | order lifecycle |
| `payment_status` | `text not null check (payment_status in ('pending','requires_action','paid','failed','expired','cancelled','refunded'))` | **kept separate from `status`** |
| `currency` | `char(3) not null default 'PHP' check (currency='PHP')` | |
| `subtotal` | `numeric(12,2) not null check (>= 0)` | server-computed |
| `shipping_fee` | `numeric(12,2) not null default 0 check (>= 0)` | |
| `discount` | `numeric(12,2) not null default 0 check (>= 0)` | reserves 0 for MVP |
| `total` | `numeric(12,2) not null check (total >= 0)` | subtotal + fee − discount |
| `idempotency_key` | `uuid not null unique` | **dedupe checkout** (see 11) |
| `shipping_address_id` | `uuid fk → addresses(id)` null | snapshot not linked (address history is acceptable to break) |
| `expires_at` | `timestamptz` | pending payment expiry (reservation horizon) |
| `paid_at` | `timestamptz` | |
| `cancelled_at` | `timestamptz` | |
| timestamps | | |

Client: select own rows; **no insert/update** (orders are created and mutated
only by Edge Functions) except updates the owner may need (e.g. cancel an
unpaid order) — decided in Phase 7; default is server-only.

## 10. order_items (immutable snapshots)

| Column | Type | Notes |
|---|---|---|
| `id` | `uuid pk` | |
| `order_id` | `uuid fk → orders(id) on delete cascade` | |
| `product_id` | `uuid fk → products(id)` null (`set null`) | provenance only |
| `variant_id` | `uuid fk → product_variants(id)` null | provenance only |
| `snapshot_name` | `text not null` | product/variant name **at purchase time** |
| `snapshot_sku` | `text` | |
| `unit_price` | `numeric(12,2) not null check (>=0)` | price **at purchase time** |
| `quantity` | `int not null check (quantity >= 1)` | |
| `line_total` | `numeric(12,2) not null check (>=0)` | unit_price × quantity |

Orders never re-derive totals from live products; **snapshots are the permanent
record**. Editing a product later never changes a historical order.

## 11. payments (payment attempts)

| Column | Type | Notes |
|---|---|---|
| `id` | `uuid pk` | |
| `order_id` | `uuid fk → orders(id) on delete cascade` | |
| `provider` | `text not null default 'XENDIT'` | |
| `provider_session_id` | `text unique` | Xendit session id from create |
| `provider_reference_id` | `text` | Xendit `reference_id` we sent |
| `provider_event_id` | `text unique` | dedupe key for capture events |
| `amount` | `numeric(12,2) not null check (amount >= 0)` | PHP, mirrors order.total |
| `currency` | `char(3) not null check (currency='PHP')` | |
| `status` | `text not null check (status in ('pending','requires_action','succeeded','failed','expired','canceled','refunded'))` | provider-confirmed |
| `checkout_url` | `text` | hosted payment link URL |
| `raw_payload` | `jsonb` | last provider event (diagnostics; no PAN) |
| `paid_at` | `timestamptz` | |
| timestamps | | |

Client read-only (own order's payments). Inserts/updates only via Edge Functions.

## 12. inventory_reservations

| Column | Type | Notes |
|---|---|---|
| `id` | `uuid pk` | |
| `order_id` | `uuid fk → orders(id)` | |
| `variant_id` | `uuid fk → product_variants(id)` | |
| `quantity` | `int not null check (quantity > 0)` | |
| `status` | `text not null check (status in ('active','released','fulfilled','expired'))` | |
| `reserved_until` | `timestamptz not null` | = order.expires_at |

Partial unique: `unique (order_id, variant_id) where status = 'active'`.
Server-only writes. See §14 for the stock strategy.

## 13. Indexes and constraints summary

- Partial indexes for catalog reads (`is_active`, `status='active'`).
- `orders(idempotency_key) unique`, `payments(provider_event_id) unique`,
  `payments(provider_session_id) unique`, `carts(user_id) unique`,
  `cart_items(cart_id, variant_id) unique`, `favorites(user_id, product_id) unique`,
  `product_variants(product_id, size, color) unique`.
- All FK actions chosen above (`cascade` where child rows must vanish with parent;
  `restrict` where history must survive).

## 14. Order and payment state machines

### Order `status`
```
pending_payment → paid → processing → shipped → delivered
pending_payment → cancelled          (user or expiry, before paid)
paid → cancelled                    (rare: manual, refund path; Phase 7+)
```
### Payment `payment_status` / `payments.status`
```
pending → requires_action → paid        (provider-actuated)
pending → failed                        (provider)
pending → expired                       (session lifetime elapsed)
pending → cancelled                     (user cancelled before capture)
paid → refunded                         (Phase 7+, via backend only)
```
Rules:

- **Statuses are kept separate.** `order.status = paid` is derived from
  `payment_status = paid` (a verified capture). A failed webhook process never
  flips an order to `paid`.
- Allowed transitions are enforced, not just implied. Enforce via a check
  constraint on `(old.status, new.status)` or (cleaner) a server-side
  transaction that conditionally updates (e.g. `update ... where status =
  'pending_payment'`), plus Edge Function state checks. Prefer conditional SQL
  guards over triggers for auditability.
- Only these trusted operations trigger transitions:
  - `create-checkout-session` → creates order (`pending_payment`) and payment
    (`pending`), reservation (`active`).
  - `xendit-webhook` (verified, deduped) → payment transitions →
    order `paid` on capture; `failed`/`expired`/`cancelled` paths release stock.
  - scheduled job (Phase 7) → expires stale `pending_payment` orders/reservations.
  - admin/manual backend action → `processing/shipped/delivered/cancelled/refunded`.

### Client-visible mapping (same names everywhere)

- order `pending_payment` + payment `pending|requires_action` → UI "Pending"
- order `pending_payment` + payment `failed|expired|cancelled` → UI per state
- order `paid` + payment `paid` → UI "Paid" (server-confirmed)
- order `processing|shipped|delivered` → fulfillment timeline
- order `cancelled` → UI "Cancelled"

## 15. Inventory / stock management strategy

**Accept**: reserve-on-checkout with expiry (a pragmatic middle ground between
over-selling and blocking carts forever).

1. During `create-checkout-session`, inside a single transaction:
   `select ... for update` on the `product_variants` row, verify
   `stock_qty >= requested`, then insert an `inventory_reservations` row
   `(status='active', reserved_until=order.expires_at)` where `expires_at` is
   bounded (e.g. 1h — determined in Phase 6 from Xendit session expiry).
   `available = stock_qty - sum(active reservations)`, enforced by the
   conditional update returning affected rows = 0 → abort.
2. On verified `paid` webhook: reservation → `fulfilled`; no stock decrement at
   the reservation level because `stock_qty` is decremented at fulfillment time
   — **decision**: decrement `stock_qty` atomically when the order is marked
   `paid` (same transaction as the payment transition). This guarantees no
   double-fulfillment: only one transition to `paid` can win.
3. On `failed`/`expired`/`cancelled`: reservation → `released`.
4. Late payment success after reservation expiry: accept only if (a) the order is
   still `pending_payment`, (b) `stock_qty` still covers the quantity (re-check
   with row lock in the webhook transaction). If not, keep the payment, mark
   order `cancelled` high-level and refund path (Phase 7). Documented behavior —
   see `PAYMENT_ARCHITECTURE.md` "payment success after order expires".

Increments/decrements are single-row conditional SQL, never read-modify-write in
the app layer, so two customers can never buy the same final item.

## 16. Money → Xendit amount conversion

- Canonical: `numeric(12,2)` PHP in Postgres (e.g. `1599.00`).
- Flutter displays ₱ from these values via `core/utils/money.dart`.
- The edge function converts the canonical decimal **to the representation the
  currently supported Xendit Session API expects**:
  - Confirmed on 2026-10-24: the official `POST /sessions` examples for
    `currency: PHP` use integer amounts (e.g. `amount: 10000` for ₱100.00) —
    i.e. **minor units (centavos), no decimals**, and the docs state the amount
    must respect the currency's decimal places.
  - The older/alternative `v3/payment_requests` API shows decimals
    (`request_amount: 10000.01`) — a different contract.
- **Implementation rule:** the conversion lives in exactly one server helper
  (`toXenditAmountMinorUnits(pesoDecimal)`) and is validated in Phase 6 with a
  **test-mode session** before any live charge. If the official docs change,
  update this section and ADR 0004.
- Never pass a Dart `double` anywhere near this boundary. Server does all
  conversions from `numeric`/`bigdecimal`-style values.


