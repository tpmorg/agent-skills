# React Native Deep Links & OAuth Audit

## What to Scan

1. **Linking config**: `Linking`, Expo linking, navigation deep link config
2. **Redirect URIs**: OAuth providers, email magic links, password reset links
3. **Universal/App Links**: associated domains, Android intent filters, iOS associated domains
4. **Callback parsing**: how the app reads and trusts inbound URLs

## Checklist

### Link Trust Boundaries

- [ ] **Sensitive auth flows prefer universal/app links over custom schemes.**
  Custom schemes are easier to hijack because other apps can register them.
  **High** if OAuth tokens or codes rely on an unverified custom scheme.

- [ ] **Incoming URLs are validated before use.**
  Host, path, and expected parameters should be checked before navigating or consuming
  tokens. **High** if the app trusts arbitrary incoming URLs.

- [ ] **Open redirect patterns are not present.**
  If a callback includes a `next`, `redirect`, or screen name that is used without
  validation, the app may navigate somewhere unintended or complete an auth flow
  unsafely. **Medium** to **High** depending on the context.

### OAuth Safety

- [ ] **OAuth callbacks validate state/nonce and expected provider.**
  Missing validation weakens replay/CSRF protections. **Medium** if missing.

- [ ] **Raw callback URLs are not logged or stored unnecessarily.**
  Auth codes, tokens, or reset tokens should not end up in logs or analytics.
  **Medium** if found.

- [ ] **Password reset and magic links are scoped and short-lived.**
  If the app handles these flows directly, verify the backend expectations are safe.
  **Info** or **Medium** depending on implementation.

## Common Patterns to Flag

```ts
// BAD: Trusting any incoming URL
Linking.addEventListener("url", ({ url }) => handleUrl(url));

// BETTER: Parse and validate host/path before acting
const parsed = new URL(url);
if (parsed.host !== "auth.example.com") return;
```
