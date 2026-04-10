# React Native Network Security Audit

## What to Scan

1. **HTTP clients**: `fetch`, axios, Apollo, urql, custom wrappers
2. **Base URLs**: config modules, environment selection, release endpoints
3. **Request/response logging**: interceptors, devtools, error reporters
4. **TLS behavior**: certificate pinning, native network config, insecure overrides
5. **WebViews**: external content loading, JS bridges, URL validation

## Checklist

### Transport Security

- [ ] **Production endpoints use HTTPS only.**
  Any release API, auth, or logging endpoint using HTTP is a direct transport risk.
  **Critical** if found.

- [ ] **No certificate validation bypass exists in native code or wrappers.**
  Some custom native modules or debug helpers disable TLS checks. **Critical** if
  present in production paths.

- [ ] **Certificate pinning, if used, is implemented safely.**
  Pinning should include backup pins and avoid debug-only logic leaking into release.
  **Medium** if brittle or inconsistently applied.

### Request Hygiene

- [ ] **Sensitive values are not placed in query strings.**
  Tokens, emails, or IDs in URLs leak into logs and analytics. **Medium** if found.

- [ ] **Verbose network logging is disabled in release.**
  Axios interceptors, Sentry breadcrumbs, custom logging, or Flipper integrations
  should not dump auth headers or payload bodies in production. **Medium**.

- [ ] **Error reporting scrubs sensitive request data.**
  Crash reporters should not receive tokens, passwords, or raw user content. **High**
  if unsanitized payloads are sent to third parties.

### WebView Surface

- [ ] **WebViews do not load arbitrary untrusted URLs without validation.**
  A user-controlled WebView URL can become a phishing or token exfiltration surface.
  **High** if found.

- [ ] **JavaScript bridges expose only minimal native functionality.**
  Overpowered bridges are dangerous when combined with remotely loaded content.
  **High** if sensitive native actions are bridged.

## Common Patterns to Flag

```ts
// BAD: Token in query string
axios.get(`${API_URL}/me?token=${token}`);

// GOOD: Token in Authorization header
axios.get(`${API_URL}/me`, {
  headers: { Authorization: `Bearer ${token}` },
});
```
