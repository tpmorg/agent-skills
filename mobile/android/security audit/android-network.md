# Android Network Security Audit

## What to Scan

1. **`network_security_config.xml`**: TLS, cleartext, certificate pinning, trust anchors
2. **OkHttp / Retrofit configuration**: Interceptors, TLS settings, timeouts
3. **Ktor client configuration**: Used by Supabase SDK — logging, TLS
4. **Endpoint URLs**: All API endpoints the app communicates with
5. **Certificate pinning**: If implemented, verify correctness
6. **Debug vs. release network behavior**: Are debug relaxations removed in release?

## Checklist

### Network Security Config

- [ ] **`network_security_config.xml` exists.**
  If absent, the app uses system defaults (which are generally secure on modern
  Android — cleartext blocked by default on API 28+). If present, audit its contents.
  **Info** if missing (defaults are fine for API 31+).

- [ ] **Cleartext traffic is not permitted.**
  Check for `<base-config cleartextTrafficPermitted="true">` or per-domain overrides.
  Since `minSdk = 31`, the default is cleartext blocked, which is correct.
  **High** if cleartext is explicitly enabled for production domains.

  Exception: `localhost` / `10.0.2.2` cleartext for development is fine if
  wrapped in a `<debug-overrides>` block. **Info**.

- [ ] **Debug overrides don't trust user CAs in release.**
  `<debug-overrides>` block can trust user-installed certificates for debugging
  (e.g., Charles Proxy). This is safe — it only applies to debug builds. But verify
  similar trust isn't in `<base-config>`. **High** if user CAs are trusted globally.

- [ ] **Certificate pinning is considered for critical endpoints.**
  For Supabase and your own API, certificate pinning prevents MITM even if the
  device's CA store is compromised. It's not required but is defense-in-depth.
  **Info** as a recommendation, not a finding.

  Note: Pinning requires careful management — pinned certs must be rotated before
  expiry or the app breaks. Include backup pins.

### OkHttp / Retrofit

- [ ] **No custom `TrustManager` that accepts all certificates.**
  A common anti-pattern during development: `X509TrustManager` that returns empty
  `checkServerTrusted`. This disables TLS entirely. **Critical** if in production code.

  ```kotlin
  // CRITICAL: Disables all certificate validation
  val trustAll = object : X509TrustManager {
      override fun checkServerTrusted(chain: Array<X509Certificate>, type: String) {}
      // ...
  }
  ```

- [ ] **`HostnameVerifier` is not overridden to accept all hostnames.**
  Similar to trust manager bypass — allows connecting to any server regardless of
  certificate hostname. **Critical** if in production.

- [ ] **HTTP logging interceptor is disabled in release builds.**
  `HttpLoggingInterceptor` with `Level.BODY` logs full request/response bodies
  including auth tokens, user data, and API responses. This shows up in logcat.
  **Medium** if enabled in release.

- [ ] **Timeouts are set.**
  Missing timeouts can lead to resource exhaustion (not directly a security issue
  but affects availability). **Low** if defaults are used.

- [ ] **No sensitive data in URL query parameters.**
  Tokens, keys, or user data in query strings appear in server logs, CDN logs,
  and HTTP `Referer` headers. Should be in headers or request body. **Medium** if found.

### Ktor Client (Supabase SDK)

- [ ] **Ktor logging plugin is configured for production.**
  Check if `Logging` plugin is installed with `LogLevel.ALL` or `LogLevel.BODY`.
  The Supabase SDK's Ktor client may log auth headers and tokens. **Medium** if
  verbose logging is enabled in release.

  ```kotlin
  // Check for this in Supabase client setup:
  install(Logging) {
      level = LogLevel.ALL  // BAD in production
  }
  ```

- [ ] **Ktor engine uses system TLS defaults.**
  The `ktor-client-android` engine uses `HttpURLConnection` under the hood, which
  respects `network_security_config.xml`. Verify no custom SSL config overrides this.
  **Info** — usually correct by default.

### Endpoints & Data in Transit

- [ ] **All endpoint URLs use HTTPS.**
  Check every hardcoded URL and BuildConfig URL:
  - `SB_URL` — should be `https://*.supabase.co`
  - `LOGGING_URL` — verify it's HTTPS
  - Any other API endpoints in source code
  **Critical** if any production endpoint uses HTTP.

- [ ] **The logging endpoint doesn't receive sensitive data.**
  The `LOGGING_URL` (`https://www.smartshotsai.com/api/logs`) receives log data
  from the app. Audit what data is sent:
  - Device info, app version — **Info**, fine
  - User ID, email — **Low**, should be documented in privacy policy
  - Screenshot content, OCR text — **High** if sent
  - Auth tokens — **Critical** if sent
  
  Also verify the endpoint requires some form of authentication or at minimum
  rate limiting to prevent abuse. **Medium** if unauthenticated.

- [ ] **API requests include appropriate auth headers.**
  Requests to Supabase should include the user's JWT in the `Authorization` header.
  Requests that accidentally omit this may work if RLS has a permissive fallback.
  Cross-reference with RLS audit. **Medium** if auth headers are inconsistently applied.

### WebView (if present)

- [ ] **No WebView JavaScript bridge exposing native methods.**
  `@JavascriptInterface` methods are callable from any JavaScript in the WebView.
  If a WebView loads external content, any exposed method is an attack surface.
  **High** if found with sensitive operations.

- [ ] **WebView doesn't load arbitrary URLs.**
  If a WebView URL comes from an intent or deep link without validation, an attacker
  could load a phishing page inside your app. **High** if URL is unvalidated.

## Common Patterns to Flag

```kotlin
// CRITICAL: Trust all certificates
val unsafeOkHttpClient = OkHttpClient.Builder()
    .sslSocketFactory(createUnsafeSSLSocketFactory(), trustAllManager)
    .hostnameVerifier { _, _ -> true }
    .build()

// BAD: Logging in production
val client = OkHttpClient.Builder()
    .addInterceptor(HttpLoggingInterceptor().apply {
        level = HttpLoggingInterceptor.Level.BODY
    })
    .build()

// GOOD: Conditional logging
val client = OkHttpClient.Builder().apply {
    if (BuildConfig.DEBUG) {
        addInterceptor(HttpLoggingInterceptor().apply {
            level = HttpLoggingInterceptor.Level.BODY
        })
    }
}.build()

// BAD: Sensitive data in query string
val url = "https://api.example.com/data?token=${userToken}&email=${email}"

// GOOD: Sensitive data in headers
val request = Request.Builder()
    .url("https://api.example.com/data")
    .addHeader("Authorization", "Bearer $userToken")
    .build()
```
