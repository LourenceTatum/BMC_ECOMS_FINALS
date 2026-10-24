# Design System — Gothic Luxury Streetwear

Premium editorial aesthetic: **black, charcoal, off-white, restrained silver**;
strong serif display type; dramatic product photography; original ornamental
cross-inspired details used sparingly. Avoid generic marketplace UI, excessive
gradients, random rounded cards, and gratuitous animation.

> Aesthetic guidance is not permission to misrepresent product authenticity or
> commercial affiliation. All typefaces must be properly licensed (Google Fonts
> under OFL are the preferred default). Motifs are original design work — not
> copied brand glyphs.

## 1. Brand principles

1. Black + negative space first; silver/off-white as restrained accents.
2. Typography does the drama; decoration is minimal and ornamental.
3. Product photography is hero-level: dark, dramatic, consistent crops.
4. Every screen prioritizes clarity: one primary action per screen.
5. The brand never mimics specific Chrome Hearts products or logos.

## 2. Color tokens

| Token | Value | Use |
|---|---|---|
| `surfaceBlack` | `#0A0A0B` | App background (primary) |
| `elevatedCharcoal` | `#141416` | Cards, sheets, raised surfaces |
| `surfaceMuted` | `#1E1E21` | Inputs, chips, inactive states |
| `offWhite` | `#F1EDE6` | Primary text on dark |
| `textSecondary` | `#B8B4AB` | Body/secondary text |
| `silver` | `#C9C9CE` | Accents, icons, motif strokes |
| `silverDim` | `#838389` | Disabled text, borders |
| `hairline` | `#2A2A2E` | Dividers, outlines |
| `danger` | `#D24A43` | Errors, destructive actions |
| `success` | `#6FA387` | Paid/confirmed states |
| `warning` | `#C9A35C` | Pending/warning states |

Contrast: all body text on `surfaceBlack` must meet WCAG AA (≥4.5:1);
`silverDim` is only for disabled/supplementary content. Accessibility token:
`textOnDark` uses `offWhite` at full opacity; never reduce below AA.

## 3. Typography

- **Display (serif, editorial):** e.g. *Cormorant Garamond* or *Marcellus*
  (OFL licensed). Used for product names, section headers, numbers. All-caps
  with wide tracking for editorial impact.
- **Body (sans):** e.g. *Inter* or the platform default. For descriptions,
  forms, price labels, badges.
- Roles:
  - `displayHero` — serif, 34–40 sp, home hero / product name.
  - `headline` — serif, 24–28 sp, screen titles.
  - `title` — serif, 18–20 sp, cards/sections.
  - `body` — sans, 15–16 sp, readable at 1.4–1.6 line height.
  - `caption` — sans, 12–13 sp, uppercase + letterSpacing 1.2 for labels.
- Fallbacks: if the licensed serif fails, fall back to `Georgia, serif`; sans
  falls back to the platform font.
- Never set prices in italics; prices use the silver token on black.

## 4. Spacing scale

Base 4: 4 / 8 / 12 / 16 / 24 / 32 / 48 / 64 / 96. Standard screen inset: 16;
max content width on tablets: 720. Generous whitespace around hero and product
imagery; tight spacing inside cards.

## 5. Buttons and inputs

- **Primary button:** min 48dp tall, off-white text on `surfaceMuted` with a thin
  `silver` 1px border; pressed state inverts (off-white bg, black text). Uppercase
  label, letterSpacing 2.
- **Ghost/secondary:** transparent, `silver` 1px border, off-white text.
- **Danger:** reserved for destructive confirmation.
- **Text fields:** dark `surfaceMuted` fill, `hairline` border, off-white label
  above the field, small caption errors in `danger`. Focus border `silver`.
- Disabled = `silverDim` text/border, no elevation.

## 6. Cards & product imagery

- Product cards: black background, **4:5 image ratio**, name in serif below,
  PHP price in sans caption style. Corner radius ≤4px (editorial edge).
- Image treatment: dark exposure, consistent crop, subtle 1px hairline frame.
- Loading: neutral charcoal skeleton, then fade-in; `ProductMedia` fallback with
  an ornamental placeholder.

## 7. Motifs and icons

- One original cross-inspired mark (thin-stroke, geometric) used as: footer
  divider, empty-state centerpiece, full-screen loading pulse.
- Icons: Material Outlined style, `silver`; sizes 20/24.
- Restraint: one motif per screen; no gradients on motifs.

## 8. Loading, empty, and error states

- Loading: skeleton blocks matching final layout; brand pulse for full-screen
  loads.
- Empty: centered motif + serif title + supporting body + one action.
- Error: `danger` accent icon, message, "Try again" action; stays on-brand.
- Message rules: `docs/ERROR_HANDLING.md`.

## 9. Navigation patterns

- Primary: bottom navigation (storefront, catalog, favorites, profile) via
  go_router `ShellRoute`; cart/checkout/order flows are stacked (AppBar + back).
- Checkout flow: cart → review → payment → confirmation with clear titles.
- Payment status screens are full-bleed states: pending (pulse), paid (success),
  failed/expired (danger) with explicit next actions.

## 10. Motion

- 200–300 ms ease-in-out fades/slides for page and image transitions.
- No springy rebound or perspective transforms. Micro-interactions only on
  primary actions (button press, favorite toggle).

## 11. Screen-by-screen

- **Auth:** single-column, centered; brand mark above; form rows; quiet motif
  footer. Loading on submit; inline field errors.
- **Home/storefront:** full-bleed editorial hero, category rails, featured grid,
  "New arrivals" section, footer motif divider.
- **Catalog:** sticky filter bar (category + sort), product grid; skeleton grid
  while loading; empty state for no-results.
- **Product details:** 4:5 gallery (swipe), serif name, price, variant pills
  (size, then color), stock caption ("Only 3 left" / "Sold out"), quantity
  stepper, sticky bottom bar: **Buy Now** (primary) + **Add to Cart** (ghost).
- **Cart:** line items with thumbnails, quantity steppers, totals block,
  continue-to-checkout.
- **Checkout review:** address selector → item summary → totals (subtotal,
  shipping, total) → Place order. Totals are server-derived.
- **Payment handoff:** "You'll continue in your browser…" state;
  **payment-pending**: "Confirming your payment… do not close".
- **Payment result:** pending / paid / failed / cancelled / expired per server.
- **Order history/detail:** timeline rows (pending_payment → paid → processing →
  shipped → delivered) + payment status chip + item snapshots + totals.
- **Profile/addresses:** list-based, quiet, full-width controls.

## 12. Accessibility & touch

- Key targets ≥48dp; text ≥12sp; AA contrast; semantics for icon-only controls;
  layout respects text-scale factor ≥1.3.

## 13. Reuse guidance

Reusable components are justified only where reuse exists: `core/widgets`
(`AsyncView`, `PrimaryButton`, `ProductMedia`, motif). Build a shared component
when the second use appears; do not pre-build a generic design framework.
