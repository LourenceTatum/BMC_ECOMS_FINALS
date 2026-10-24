---
name: flutter-ui-design
description: Implement UI for the BMC Commerce gothic-luxury design system. Use when
  building or styling screens, widgets, themes, or image layouts in bmc_final/lib.
  Covers using docs/DESIGN_SYSTEM.md tokens and typography, reusing shared widgets,
  avoiding generic marketplace templates, and keeping UI state separate from
  authoritative backend state.
---

# Flutter UI design

Apply the gothic luxury design system consistently. Read
`docs/DESIGN_SYSTEM.md` fully before styling anything.

## Core rules
1. **Design tokens only** — pull colors and spacing from `core/theme/tokens.dart`
   (mapped to `docs/DESIGN_SYSTEM.md`). Never invent ad-hoc hex colors in a
   screen.
2. **Typography** — use the defined roles (displayHero/headline/title/body/
   caption). Serif for product names and headers; sans for descriptions and
   prices. Prices: silver token, never italic.
3. **Restraint** — black/charcoal surfaces, off-white text, silver accents,
   ≤4px radius, hairline borders. No gradients, no extreme rounded cards, no
   gratuitous animation. One motif (cross ornament) per screen at most.
4. **Product imagery** — 4:5 ratio, dark editorial exposure, placeholder
   skeletons on load, alt text. Use the shared `core/widgets/product_media.dart`
   instead of ad-hoc Image widgets.
5. **Reuse shared widgets** before creating new ones: `AsyncView`,
   `PrimaryButton`, product media, motif. Create a new shared widget only when a
   second consumer exists.

## State discipline
- UI reflects ephemeral selection state (selected variant, quantities) locally in
  the widget/controller. **Never** present client-derived totals or payment
  success as truth. Authoritative values arrive from repositories (server).
- Payment and checkout screens render exactly the server statuses
  (`docs/PAYMENT_ARCHITECTURE.md` Stage 9 matrix).

## Accessibility
- Key touch targets ≥48dp; text ≥12sp; AA contrast on all foreground text;
  semantics labels for icon-only controls; respect text scale factor.

## Each screen check
Auth (single column, centered) · Storefront (editorial hero, category rails) ·
Catalog (sticky filters, grid) · Detail (gallery, variant pills, sticky Buy
Now) · Cart/Review (line items, totals) · Payment (handoff, pending results) ·
Orders (timeline + status chip) · Profile (list). Confirm the result matches
the screen-by-screen guidance in `docs/DESIGN_SYSTEM.md` §11.

## Verify
- `flutter analyze` and `flutter test` from `bmc_final/` when a screen is added.
- Screenshot smoke-check on a device for the touched flows (reported, not
  claimed as automated).
