# Authentication & Authorization (Supabase Auth)

Design for the full email/password auth lifecycle using `supabase_flutter`.
Matches `ARCHITECTURE.md` routing and `DATABASE_SECURITY.md` ownership rules.

## 1. Auth primitives

- `Supabase.initialize(url, publishableKey, ...)` at app start.
- `supabase.auth.signUp(email:, password:)` — registration.
- `supabase.auth.signInWithPassword(email:, password:)` — login.
- `supabase.auth.onAuthStateChange(...)` — session lifecycle + initial restore.
- `supabase.auth.resetPasswordForEmail(email)` + deep link → set new password
  via `supabase.auth.updateUser(password:)` after OTP/recovery validation.
- `supabase.auth.signOut()` — explicit logout (also clears persisted session).

## 2. Registration behavior

- Client validates email format + password (≥8 chars). Server re-validates.
- Duplicate email: `signUp` returns an `AuthException` (code
  `email_exists`/`user_already_exists`) → UI shows a "This account already
  exists. Sign in instead." message with a link to login.
- On successful signup with **email confirmation disabled** (local dev),
  Supabase returns a session → user lands in the app as authenticated.
- With **email confirmation enabled** (production), user is signed out and must
  verify; UI shows a "We sent a verification link to {email}." state. A
  "Resend verification" action calls `resendSignUpOtp(email:)` when the current
  user's `email_confirmed_at` is null.

## 3. Login / logout

- `signInWithPassword`; wrong credentials → friendly error; rate-limit hints kept
  generic ("Check your email and password.").
- Logout always calls `signOut()`; protected providers are invalidated
  (Riverpod providers refresh to unauthenticated states).

## 4. Session persistence and restoration

- `supabase_flutter` persists the session (SharedPreferences by default) and
  restores it on start. The app listens to
  `onAuthStateChange(initialSession: true)`:
  - `INITIAL_SESSION` → restore session → redirect guards re-evaluate.
  - `SIGNED_IN` / `SIGNED_OUT` / `TOKEN_REFRESHED` / `USER_UPDATED` → update
    `currentUserProvider`/`sessionProvider`.
- Expired/revoked sessions surface as `AuthException` (401 on refresh). The app
  shows a non-blocking "session expired, sign in again" message and redirects to
  login (preserving the intended route).

## 5. Password recovery

1. Login screen → "Forgot password?" → `resetPasswordForEmail(email)`.
2. Supabase emails a **recovery link** (configured redirect URL / deep link).
3. The deep link opens the app's **reset-password** route (Phase 1 config:
   Android App Links / deep link URI — e.g.
   `bmcapp://auth/callback` or `https://<project>.supabase.co/auth/v1/verify`
   handled by `app_links` — exact value fixed in Phase 3).
4. App calls `supabase.auth.verifyOTP(type: OtpType.recovery, tokenHash: ...)`
   or updates the user password with the session from the link; then
   `updateUser(password: newPassword)`.
5. The reset screen requires the user to remain signed in (recovery session),
   validates new password, and shows success → navigate home.

## 6. Protected routes (go_router guards)

- `routerConfig` uses a `redirect` that reads `sessionProvider`:
  - unauthenticated → any route in the protected set redirects to
    `/auth/login?redirect=<intended>`.
  - authenticated → auth screens redirect to `/`.
- Guard checks are **UI convenience**; true enforcement is RLS + server checks
  (an authenticated user who forges a route still cannot read others' data).

## 7. First-login profile initialization

- A database trigger on `auth.users` insert (or a post-signup edge function,
  whichever is adopted in Phase 3) ensures a `profiles` row exists:
  `id = new.id`, `email = new.email`. Client may then edit `full_name`, phone, etc.
- Client-side: after first sign-in, if profile is missing, show a short
  "complete your profile" screen (name + phone).

## 8. Authorization boundaries

- **Normal user**: own profile/addresses/cart/favorites/orders (RLS).
- **Future administrator**: separate `is_admin` flag **on a server-only column**
  read by Edge Functions (not by client RLS); an admin panel would use separate
  server-gated endpoints, not broader client grants. Documented as Phase 8+.
- **Cross-user access**: opening another user's order must end in an
  authorization failure. Because orders have `GRANT SELECT` + RLS to own rows,
  PostgREST returns 0 rows for foreign rows. For detail fetches, the repository
  treats `row == null` on an owned-route as **not found**; additionally, future
  RPC `get_order_for_user` (service-path) returns an explicit `403 Forbidden`
  when the caller is authenticated but not the owner. The app maps both to a
  "This order is unavailable or you are not authorized" state — never a silent
  fake-success.

## 9. Redirect URLs / deep links (dev + prod)

- Supabase Auth redirect URLs and Flutter deep links are configuration items:
  - Development: Android emulator deep link (e.g.
    `bmcapp://auth/callback`) registered in the manifest; Supabase allowed
    redirect URLs updated in Dashboard → Auth.
  - Production: HTTPS App Links (verified domains) — exact values defined in
    Phase 3 and re-verified before release (Phase 8).
- The same deep-link mechanism may later serve Xendit return URLs (payment
  result landing inside the app) — see `PAYMENT_ARCHITECTURE.md §7`.

## 10. Failure/edge behaviors

| Scenario | Behavior |
|---|---|
| Signup network failure | Retry with the form state preserved; error mapped to safe message |
| Email already verified but reused | Supabase error → "account already exists" |
| Recovery link expired | Message + "request another link" |
| Session invalid mid-flow | Redirect to login, preserve `redirect`; on return, re-attempt |
| User deletes account | Future phase (documented; auth.deleteUser + cleanup job) |
