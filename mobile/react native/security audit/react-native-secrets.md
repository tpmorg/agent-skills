# React Native Secrets & Config Audit

## What to Scan

1. **Environment files**: `.env*`, `.env.local`, `.env.production`
2. **App config**: `app.json`, `app.config.js`, `app.config.ts`, Expo `extra`
3. **Config modules**: files exporting API keys, URLs, or constants
4. **Native config**: `Info.plist`, `AndroidManifest.xml`, Gradle properties
5. **Git hygiene**: `.gitignore`, committed sample env files, leaked secrets in history

## Key Concept: Bundled App Config Is Not Secret

Anything compiled into the JS bundle, Expo config, native manifests, or generated
constants should be treated as public. If a value ships with the app, assume an
attacker can extract it.

## Checklist

### Client-Visible Config

- [ ] **Secret keys are not bundled into the app.**
  API keys with billing or privileged access must not appear in env files consumed
  by the client bundle, Expo `extra`, hardcoded config files, or native constants.
  **Critical** if found.

- [ ] **Public keys are correctly classified as public.**
  Publishable/anon keys are fine if the backend is the security boundary, but they
  should be documented and not confused with secret keys. **Info**.

- [ ] **Expo `extra` does not contain secrets.**
  Values in `expo.extra` or `Constants.expoConfig` are readable at runtime from the app.
  **Critical** if secrets are stored there.

- [ ] **No production secrets in `app.config.*` or `react-native-config`.**
  These frequently look private during development but are still client-visible.
  **Critical** for privileged keys, **High** for test secrets.

### Source Code Hygiene

- [ ] **No hardcoded tokens, passwords, or private keys in source.**
  Search for common patterns like `sk_`, `AIza`, `ghp_`, `-----BEGIN`, long JWT-like
  strings, or credential literals. **Critical** if found.

- [ ] **No URLs with embedded credentials.**
  Patterns like `https://user:pass@host` or auth tokens in query strings are
  directly extractable and leak through logs. **Critical** if found.

- [ ] **Debug values are separated from release values.**
  Dev endpoints or test API keys should not silently ship in production config.
  **Medium** if mixed, **High** if production secrets are used in local/dev paths.

### Repository Hygiene

- [ ] **Sensitive env files are ignored by Git.**
  `.env`, platform local secrets, and signing files should not be committed.
  **High** if tracked in the repository.

- [ ] **Sample env files avoid real values.**
  `.env.example` should document keys without including live secrets. **Medium** if not.

## Common Patterns to Flag

```ts
// BAD: Secret bundled into the client
export const OPENAI_API_KEY = process.env.OPENAI_API_KEY;

// BAD: Expo config used as a secret store
export default {
  expo: {
    extra: {
      stripeSecretKey: process.env.STRIPE_SECRET_KEY,
    },
  },
};

// GOOD: Only public config in the app, privileged calls proxied through the server
export const API_BASE_URL = process.env.EXPO_PUBLIC_API_BASE_URL;
```
