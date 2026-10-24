# Error Handling

Consistent patterns for loading, errors, retries, and empty states across the
app. These rules apply to every async screen; UI implementation guidance is in
`DESIGN_SYSTEM.md`.

## 1. Error taxonomy (Dart)

`core/errors/app_error.dart` defines one sealed hierarchy:

| Type | Meaning | User-facing default |
|---|---|---|
| `ValidationError` | bad local input | inline field messages |
| `UnauthorizedError` | session missing/expired/forbidden | "Sign in to continue." → login |
| `NetworkError` | connectivity/timeout | "Check your connection and try again." |
| `ConflictError` | duplicate key / concurrent change | "Please refresh and try again." |
| `OutOfStockError` | variant unavailable at checkout | "This item is out of stock." |
| `CheckoutError` | session creation/provider failure | "We couldn't start checkout. Retry." |
| `PaymentRequiresAction` | async provider flow active | show payment-status screen |
| `PaymentRejected` | failed/cancelled/expired payment | status-specific screen |
| `UnknownError` | anything else | "Something went wrong." + support ask |

Rules:
- Repositories map Supabase/Xendit exceptions into these types at the boundary.
- No raw exception strings shown to users; the error code + technical detail go
  to logs only.

## 2. UI state contract for async data

Every async surface uses four states:

- **Loading**: skeleton (catalog), brand pulse (full-screen), disabled button
  with spinner (actions).
- **Error**: message + contextual retry. Never a blank screen. Retry refetches.
- **Data**: content rendered from provider state.
- **Empty**: motif + short copy + next action (e.g. "Your cart is empty" →
  Browse).

Widget implementation: `core/widgets/async_view.dart` (planned) wraps providers'
`AsyncValue` and enforces this contract with themed styling.

## 3. Action-level error handling

- Submit buttons (login/register/place order): disable while in flight; on
  failure, keep form values, show a contextual error above the action, log code.
- Buy Now and checkout creation: if the provider returns `CheckoutError`/network,
  one automatic retry with the **same idempotency key**, then surface.
- Webhook/status refresh: silent background refresh on pull-to-refresh shows
  stale-ok UI + status chip with a subtle timestamp; a failed refresh shows an
  inline error banner with Retry (never a destructive reset of the view).

## 4. Authentication failures

- `UnauthorizedError` during an active flow → navigate to login with
  `redirect=<intended>` and show "Your session expired — sign in again."
- Forbidden data access (own-route but no row / RLS empty) → specific
  not-authorized/not-found copy; never present as a success state.

## 5. Payment-specific error/status states

- Pending: "Confirming your payment… do not close this screen."
- Failed/Cancelled/Expired: primary + secondary actions (retry / view orders).
- After any provider return, the app always reconciles with the server first
  before switching screens.

## 6. Logging

- Structured key=value lines: `event`, `orderId`, `paymentId`, `function`,
  `errorCode`. Never: secrets, JWTs, passwords, PAN, callback tokens.
- Edge Functions use `supabase functions logs`; app uses `debugPrint`/`Logger`
  with a debug flag. Supabase client logs at INFO+ by default (tokens are at
  FINEST — keep default filtering in production builds).

## 7. Retry and backoff conventions

- User-triggered actions: one immediate auto-retry for idempotent operations.
- Polling (payment status screen): every 5s up to 2 min, then 15s, until the
  server state is terminal; stop when terminal.
- Webhook/internal retries: the provider's documented retry policy governs;
  our handler remains idempotent and responds 2xx fast.

## 8. Empty-state catalogue

Cart empty, no search results, no orders yet, no addresses, no favorites, sold
out. All defined in `DESIGN_SYSTEM.md` with the motif treatment.
