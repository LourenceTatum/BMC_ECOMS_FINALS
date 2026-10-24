# ADR 0005 — Product, Variant, Inventory, and Order-Snapshot Model

- **Status**: Proposed (schema in Phase 2; transitions proven in Phase 5/6).
- **Date**: 2026-10-24

## Context

Streetwear products vary by size/color, which impacts pricing and stock. Sales
records must survive product edits. We must prevent overselling and duplicate
orders without over-engineering.

## Decision

- **Purchasable SKU = one `product_variants` row** owning `stock_qty`; products
  without variants still get one variant. `available = stock_qty − Σ active
  reservations`, evaluated server-side under row locks.
- **Order preservation via immutable `order_items` snapshots** (name, SKU, unit
  price, qty, line total) — historical totals never change when a product is
  edited; orders reference live products only for provenance.
- **Money** stored as `numeric(12,2)`; totals computed in exact decimal on the
  server (never Dart/JS floats); converted to the provider representation at
  one server boundary.
- **Inventory strategy**: reserve-on-checkout with expiry (`expires_at`), fulfill
  (decrement `stock_qty`) inside the same transaction that marks an order `paid`,
  release on fail/expire/cancel, reconcile late captures under lock.
- **Idempotency**: `orders.idempotency_key unique` + `payments.provider_event_id
  unique` prevent duplicate orders/payments.
- **Order `status` and `payment_status` are separate fields** with enforced,
  documented transitions (`DATABASE_SCHEMA.md` §14).

## Alternatives considered

- No variants (single `stock_qty` per product) — rejected: sized apparel would
  mislead customers and oversell sizes.
- Inventory decrement at order creation — rejected: leaks stock on abandoned
  payments.
- Hold-until-forever reservations — rejected: carts would strand inventory.
- Store floating-point prices — rejected outright (exactness).

## Advantages

Per-size correctness; snapshot history survives edits; idempotent checkout;
decisions map 1:1 to SQL constraints and DB tests.

## Disadvantages

Reservation machinery adds a table and an expiry job; late-payment edge cases
need explicit reconciliation. Both are documented (DB §15, payment Stage 9).

## Security implications

Clients have no grants on reservations/orders/payments; all stock and status
mutations happen server-side with conditional updates, so "last unit" races
resolve safely regardless of agent code quality.

## Consequences

Catalog queries expose per-variant availability; checkout is order-first then
provider-second; any change to the status vocabulary or inventory rules requires
updating `DATABASE_SCHEMA.md`, `PAYMENT_ARCHITECTURE.md`, and their tests
together.
