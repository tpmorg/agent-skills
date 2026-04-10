# Android Auth & Session Management Audit

## What to Scan

1. **Supabase client initialization**: How `SupabaseClient` is configured, where tokens are stored
2. **Google Credential Manager / OAuth**: Token handling, nonce usage
3. **Session persistence**: DataStore, SharedPreferences, or EncryptedSharedPreferences
4. **Token refresh logic**: How expired tokens are handled
5. **Deep link / App Link handlers**: OAuth callback processing
6. **Auth state management**: How auth state flows through the app

## Checklist

### Token Storage

- [ ] **Auth tokens are stored in encrypted storage.**
  Supabase Kotlin SDK stores sessions by default. Check what `SessionManager` or
  `SettingsSessionManager` implementation is used. If tokens end up in plain
  SharedPreferences or DataStore without encryption, they're readable on rooted
  devices or via ADB backup. **Medium** (requires physical/root access, but still).

  Preferred: `EncryptedSharedPreferences` backed by Android Keystore, or the
  Supabase SDK's built-in encrypted storage if available.

- [ ] **Refresh tokens are stored as securely as access tokens.**
  A refresh token is more sensitive than an access token (longer-lived, can mint
  new access tokens). If access tokens are encrypted but refresh tokens aren't,
  the encryption is pointless. **High** if refresh tokens are in plaintext.

- [ ] **Tokens are cleared on logout.**
  Verify that signing out removes tokens from storage, not just from memory.
  Check for `supabase.auth.signOut()` and that the storage backend is actually
  cleared. **Medium** if tokens persist after logout.

- [ ] **No tokens in logs.**
  Check Timber logging configuration. In debug builds, Supabase SDK and Ktor can
  log full HTTP requests/responses including `Authorization` headers. Verify that
  production builds suppress this. **Medium** if auth headers could appear in logcat.

### Google OAuth / Credential Manager

- [ ] **ID token is validated server-side or by Supabase.**
  The Google ID token received on the client should be passed to Supabase auth
  (which validates it), not used directly for authorization decisions in the app.
  **High** if the app trusts the ID token without backend validation.

- [ ] **Nonce is used in the credential request.**
  A nonce prevents replay attacks. Check the `GetGoogleIdOption` or equivalent
  for a nonce parameter. **Medium** if missing.

- [ ] **`GET_ACCOUNTS` and `USE_CREDENTIALS` permissions are still needed.**
  These are legacy permissions. With Credential Manager API, they may not be
  necessary. Unnecessary permissions increase the app's attack surface and may
  cause Play Store review issues. **Info**.

### Deep Link / OAuth Callback Security

- [ ] **OAuth callback URLs use App Links with `autoVerify="true"`.**
  Verified App Links (HTTPS with domain verification via `.well-known/assetlinks.json`)
  are resistant to hijacking by other apps. Custom schemes (`smartshots://`) are not —
  any app can register the same scheme. Check that:
  - The `https://` intent filters have `android:autoVerify="true"` ✓
  - The `assetlinks.json` is deployed and valid on the domain
  **High** if OAuth tokens are delivered via the unverified `smartshots://` scheme.

- [ ] **The `smartshots://` custom scheme is only used for non-sensitive deep links.**
  Custom schemes are globally registerable — another app could intercept them.
  If OAuth tokens, auth codes, or sensitive data are delivered via `smartshots://`,
  it's **High**. If it's just for navigation (open the app to a screen), it's **Info**.

- [ ] **OAuth callback handler validates the state parameter.**
  If using OAuth with a `state` parameter for CSRF protection, the callback handler
  must verify it matches the original request. **High** if state is ignored.

- [ ] **Token fragments in callback URLs are handled securely.**
  Supabase delivers tokens in the URL fragment (`#access_token=...`). The app should
  extract these promptly, pass them to the Supabase SDK, and not persist the raw URL
  in history, logs, or analytics. **Medium** if the raw URL is logged.

- [ ] **Multiple intent filters for the same path don't create ambiguity.**
  The manifest has filters for both `www.smartshotsai.com` and `smartshotsai.com`
  (with and without www). Both need valid `assetlinks.json`. If only one domain
  verifies, the other falls back to unverified (user disambiguation dialog). **Medium**
  if both aren't verified.

### Session Lifecycle

- [ ] **Session refresh happens before API calls, not reactively.**
  If the app makes an API call, gets a 401, and only then refreshes — there's a
  window where stale tokens could leak information about expired state. Better:
  proactively refresh before expiry. The Supabase SDK typically handles this, but
  verify it's configured. **Low**.

- [ ] **Concurrent token refresh doesn't cause race conditions.**
  If multiple coroutines try to refresh simultaneously, they could invalidate each
  other's tokens. Check for synchronization. **Low** in practice but worth noting.

- [ ] **App handles "signed out elsewhere" gracefully.**
  If the user revokes sessions via the web app or changes password, the Android app
  should detect this and sign out locally. **Low** if it fails silently with
  confusing errors.

### Biometric / Device Lock Integration

- [ ] **Sensitive operations don't require re-authentication.**
  Operations like deleting all data, changing email, or accessing billing should
  ideally require re-authentication or biometric confirmation. **Low** if missing —
  it's a defense-in-depth measure.

## Common Patterns to Flag

```kotlin
// BAD: Token in plain SharedPreferences
val prefs = context.getSharedPreferences("auth", MODE_PRIVATE)
prefs.edit().putString("refresh_token", token).apply()

// GOOD: EncryptedSharedPreferences
val masterKey = MasterKey.Builder(context)
    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
    .build()
val prefs = EncryptedSharedPreferences.create(
    context, "auth_prefs", masterKey,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
)

// BAD: Logging auth headers in production
HttpLoggingInterceptor().apply {
    level = HttpLoggingInterceptor.Level.BODY // logs everything including tokens
}

// GOOD: Conditional logging
HttpLoggingInterceptor().apply {
    level = if (BuildConfig.DEBUG) Level.BODY else Level.NONE
}

// BAD: Custom scheme for OAuth callback (hijackable)
// Any app can register smartshots:// scheme
<data android:scheme="smartshots" />

// BETTER: Verified App Links for OAuth (not hijackable)
<intent-filter android:autoVerify="true">
    <data android:scheme="https" android:host="smartshotsai.com" />
</intent-filter>
```
