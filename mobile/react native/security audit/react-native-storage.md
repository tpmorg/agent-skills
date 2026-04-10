# React Native Local Storage Audit

## What to Scan

1. **Storage libraries**: AsyncStorage, SecureStore, MMKV, Keychain, SQLite, Realm
2. **Persisted state**: Redux persist, Zustand persist, custom caches
3. **Offline data**: queued requests, cached API responses, downloaded files
4. **Sensitive artifacts**: tokens, PII, access flags, billing state, user-generated content

## Checklist

### Sensitive Data Placement

- [ ] **Secrets and tokens are not stored in AsyncStorage.**
  AsyncStorage is convenient but not appropriate for refresh tokens, session secrets,
  or encryption keys. **High** if found.

- [ ] **PII is minimized in local caches.**
  User profile data, addresses, IDs, or sensitive content should only be cached when
  needed and ideally encrypted if high risk. **Medium** if broad caches exist.

- [ ] **Persisted feature flags or entitlements are not treated as trusted.**
  Local values can be tampered with on rooted/jailbroken devices. **High** if premium
  or admin access is granted from local storage alone.

### Storage Hygiene

- [ ] **Expired or logged-out state is cleared.**
  Cached responses, downloaded files, and persisted stores should be pruned on logout.
  **Medium** if sensitive remnants remain.

- [ ] **Downloaded files avoid public/shared locations when sensitive.**
  Storing exports or reports in world-readable paths increases leakage risk.
  **Medium** if sensitive files land in shared storage by default.

- [ ] **SQLite/Realm databases do not silently store secrets in plaintext.**
  If local databases hold messages, medical/financial data, or auth artifacts, consider
  at-rest protection. **Medium** to **High** depending on data sensitivity.

## Common Patterns to Flag

```ts
// BAD: Trusting a local premium flag
const isPro = await AsyncStorage.getItem("is_pro");

// GOOD: Fetch entitlement from server and cache only as a hint
const entitlement = await api.getEntitlement();
```
