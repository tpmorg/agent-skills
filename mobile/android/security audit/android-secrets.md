# Android Secrets & BuildConfig Audit

## What to Scan

1. **`build.gradle.kts`**: `buildConfigField` declarations, `signingConfigs`
2. **`local.properties`**: Keys stored here (should be gitignored)
3. **`.gitignore`**: Verify `local.properties`, keystores, and `.env` files excluded
4. **`BuildConfig.java`** (generated): Understand what ends up in the APK
5. **ProGuard/R8 rules**: `proguard-rules.pro` — does obfuscation help or not?
6. **Source files**: Hardcoded keys, URLs, tokens anywhere in Kotlin/Java

## Key Concept: BuildConfig Strings Are Not Secret

BuildConfig fields are compiled as static final String constants. Anyone who downloads
the APK can extract them in seconds with `apktool` or `jadx`. R8/ProGuard minification
does NOT remove BuildConfig values — it only renames classes and methods.

This means: **every BuildConfig field should be treated as public information.**

## Checklist

### BuildConfig Fields

- [ ] **Audit each `buildConfigField` for sensitivity.**
  For each field, ask: "Would it be a problem if this appeared on Reddit?"

  Typical findings:
  - `SB_URL` (Supabase URL) — **Info**. Public by design, same as `NEXT_PUBLIC_SUPABASE_URL`.
  - `SB_ANON_KEY` (Supabase anon key) — **Info**. Public by design, intended for client use.
    RLS is the security boundary, not key secrecy.
  - `OPENAI_API_KEY` — **Critical**. This is a secret key with billing attached. Anyone who
    decompiles the APK gets your OpenAI key and can run up charges. This must be proxied
    through your server.
  - `LOGGING_URL` — **Low**. An endpoint URL is not secret, but verify it doesn't accept
    unauthenticated writes that could be abused for log injection.

- [ ] **Any API key with billing or elevated access must be proxied.**
  The pattern: client calls your Next.js API → your API calls OpenAI/etc. with the
  secret key server-side. The client never sees the key. **Critical** for any key that
  can incur costs or access sensitive data.

- [ ] **Debug vs. release BuildConfig values are separate.**
  Check if debug builds use test/dev keys and release builds use production keys.
  If the same key is used for both, flag as **Low** (risk of test data in production
  or production costs during development).

### Signing Configuration

- [ ] **Release keystore password is not hardcoded in `build.gradle.kts`.**
  Should come from `local.properties` or environment variables. Checked at build time,
  not committed to VCS. **High** if hardcoded.

- [ ] **Debug keystore uses a project-specific key, not the default.**
  Using a custom debug keystore (as seen in `../keys/debug.keystore`) is fine. Verify
  the `keys/` directory is gitignored. **Medium** if committed.

- [ ] **Release keystore file is not in the repository.**
  Grep for `.jks` or `.keystore` files in the repo. **High** if found.

- [ ] **`local.properties` is in `.gitignore`.**
  Contains API keys and keystore passwords. **Critical** if committed.

### APK / AAB Decompilation Exposure

- [ ] **ProGuard/R8 is enabled for release builds.**
  Check `isMinifyEnabled = true` in release buildType. This obfuscates class/method
  names but does NOT hide string constants. **Medium** if disabled.

- [ ] **`isShrinkResources = true` for release.**
  Removes unused resources, reducing information leakage from unused layouts,
  drawables, etc. **Low** if disabled.

- [ ] **No sensitive strings in resource files.**
  Check `strings.xml`, `xml/` configs for API keys, internal URLs, or debug endpoints.
  **High** if found.

- [ ] **Network security config doesn't leak internal hostnames.**
  If `network_security_config.xml` lists internal/staging domains, those are visible
  in the APK. **Low** but worth noting.

### Hardcoded Secrets in Source

- [ ] **Grep for common secret patterns in Kotlin/Java files.**
  Patterns to search:
  - `"sk-"`, `"sk_live"`, `"sk_test"` (OpenAI, Stripe)
  - `"eyJ"` (JWT tokens)
  - `"AIza"` (Google API keys)
  - `"ghp_"`, `"gho_"` (GitHub tokens)
  - `password`, `secret`, `token` as variable names with string literal assignments
  - Long base64-like strings (40+ chars) assigned to variables

  **Critical** for any production secret. **High** for test secrets.

- [ ] **No URLs with embedded credentials.**
  Pattern: `https://user:password@host` or query strings with `?key=...`.
  **Critical** if found.

## Common Patterns to Flag

```kotlin
// BAD: API key with billing in BuildConfig — extractable from APK
buildConfigField("String", "OPENAI_API_KEY",
    "\"${localProperties.getProperty("openai.api.key", "")}\"")

// GOOD: Client calls your proxy endpoint, server holds the key
// In Android:
val response = api.analyzeScreenshot(screenshotData, userToken)
// Your Next.js API route holds the OpenAI key server-side

// BAD: Hardcoded key in source
private const val API_KEY = "sk-abc123..."

// BAD: Key in strings.xml
<string name="openai_key">sk-abc123...</string>

// GOOD: Only public/anon keys in BuildConfig
buildConfigField("String", "SB_URL", "\"${localProperties.getProperty("supabase.url")}\"")
buildConfigField("String", "SB_ANON_KEY", "\"${localProperties.getProperty("supabase.anon.key")}\"")
```

## Remediation Pattern for Secret API Keys

When a secret key is found in BuildConfig:

1. Create a server-side proxy endpoint: `POST /api/proxy/openai`
2. The endpoint validates the user's Supabase auth token
3. The endpoint calls OpenAI with the secret key
4. The endpoint returns the result to the client
5. Remove the key from `build.gradle.kts` entirely

This also gives you rate limiting, cost tracking per user, and the ability to
revoke access without shipping an app update.
