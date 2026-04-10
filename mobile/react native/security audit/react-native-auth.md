# React Native Auth & Session Audit

## What to Scan

1. **Auth SDK setup**: Supabase, Firebase Auth, Auth0, Clerk, Cognito, custom auth
2. **Session persistence**: where access and refresh tokens are stored
3. **Login/logout flows**: token lifecycle, sign-out behavior, refresh logic
4. **Protected screens and API calls**: how auth state gates access
5. **OAuth handlers**: redirects, PKCE, nonce/state validation

## Checklist

### Token Handling

- [ ] **Long-lived tokens are stored in secure storage.**
  Refresh tokens or session tokens should not live in plain AsyncStorage.
  Prefer Keychain, SecureStore, or platform keystore-backed storage. **High** if
  refresh tokens are in plaintext storage.

- [ ] **Auth tokens are cleared on logout.**
  Sign-out should clear persisted storage, in-memory state, and any cached user data.
  **Medium** if tokens or profile data persist after logout.

- [ ] **Tokens are never logged.**
  Debug logging around auth responses, request headers, or session objects can expose
  bearer tokens. **Medium** in debug-only paths, **High** if possible in release.

### Authorization Boundaries

- [ ] **Protected behavior is enforced server-side, not only in navigation.**
  Hiding screens in the app is not authorization. Sensitive data and actions must be
  protected by server checks as well. **High** if premium/admin behavior is gated
  client-side only.

- [ ] **Role or ownership checks do not rely on mutable client claims alone.**
  The client should not decide it is an admin just because a decoded token or local
  flag says so. **High** if trusted directly.

### OAuth / SSO

- [ ] **PKCE or equivalent secure OAuth flow is used.**
  Mobile auth should avoid implicit flows where possible. **High** if legacy insecure
  redirect patterns are used for sensitive accounts.

- [ ] **OAuth state or nonce is validated.**
  Missing state/nonce validation weakens CSRF/replay protections. **Medium** if missing.

- [ ] **Callback URLs are handled by trusted redirect paths only.**
  If the app accepts arbitrary redirect targets or loosely parsed callback URLs, an
  attacker may inject tokens or hijack flows. **High** if unvalidated.

## Common Patterns to Flag

```ts
// BAD: Refresh token stored in AsyncStorage
await AsyncStorage.setItem("refresh_token", token);

// GOOD: Long-lived secrets stored in secure storage
await SecureStore.setItemAsync("refresh_token", token);

// BAD: UI-only gating
if (user?.plan === "pro") {
  return <PremiumScreen />;
}
```
