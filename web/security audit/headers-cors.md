# Headers & CORS Audit

## What to Scan

1. **`next.config.*`**: `headers()` configuration
2. **`middleware.ts`**: Custom header injection, CORS handling
3. **API routes**: Per-route CORS or header logic
4. **`vercel.json`**: Header and redirect rules
5. **HTML `<meta>` tags**: CSP via meta tags, `X-Frame-Options` equivalent

## Checklist

### CORS (Cross-Origin Resource Sharing)

- [ ] **No wildcard `Access-Control-Allow-Origin: *` on authenticated endpoints.**
  Wildcard CORS means any website can make requests to your API. This is fine for
  truly public endpoints (public data, health checks) but **High** for any endpoint
  that uses cookies or returns user-specific data.

- [ ] **CORS origin is validated against an allowlist.**
  If CORS is dynamic (reflecting the `Origin` header), check that it validates against
  a list of allowed origins. Reflecting any origin is equivalent to wildcard. **High**.

- [ ] **`Access-Control-Allow-Credentials: true` is not paired with wildcard origin.**
  Browsers won't send cookies with `*` origin anyway, but this configuration suggests
  a misunderstanding of CORS. **Medium**.

- [ ] **Preflight responses cache appropriately.**
  `Access-Control-Max-Age` should be set to avoid excessive preflight requests.
  Not a security issue, but **Info** for performance.

- [ ] **CORS is not set on webhook endpoints.**
  Webhook endpoints receive server-to-server requests that don't go through browsers.
  CORS headers on these are unnecessary and could increase attack surface. **Info**.

### Content Security Policy (CSP)

- [ ] **CSP header exists.**
  A Content-Security-Policy header prevents XSS, clickjacking, and other injection
  attacks. Missing CSP is **Medium** for most apps.

- [ ] **`script-src` doesn't include `'unsafe-inline'` or `'unsafe-eval'`.**
  These directives largely defeat the purpose of CSP. Next.js requires `'unsafe-eval'`
  in development but it should not be present in production. **Medium** if in production.

  Note: Next.js uses inline scripts for hydration. The recommended approach is
  `script-src 'self' 'nonce-...'` with Next.js's built-in nonce support, or
  `'unsafe-inline'` as a fallback (less secure). Flag `'unsafe-inline'` as **Low**
  with a note about nonce-based CSP.

- [ ] **`default-src` is restrictive.**
  `default-src 'self'` is a good baseline. `default-src *` provides no protection.
  **Medium** if overly permissive.

- [ ] **`frame-ancestors` restricts embedding.**
  This replaces `X-Frame-Options` for modern browsers. Should be `'self'` or `'none'`
  unless your app is designed to be embedded. **Medium** if missing or permissive.

- [ ] **`connect-src` includes only necessary domains.**
  Should list your API domain, Supabase URL, Stripe, analytics, etc. Wildcard here
  allows data exfiltration via XSS to any domain. **Medium** if wildcard.

### Security Headers

- [ ] **`X-Frame-Options` is set.**
  `DENY` or `SAMEORIGIN`. Prevents clickjacking. **Low** if missing (CSP
  `frame-ancestors` is the modern replacement, but belt-and-suspenders is good).

- [ ] **`X-Content-Type-Options: nosniff` is set.**
  Prevents MIME type sniffing. **Low** if missing — it's a one-line fix.

- [ ] **`Referrer-Policy` is set.**
  `strict-origin-when-cross-origin` or `no-referrer` are recommended.
  Default browser behavior leaks the full URL in the `Referer` header.
  **Low** if missing, **Medium** if your URLs contain sensitive parameters.

- [ ] **`Strict-Transport-Security` (HSTS) is set.**
  `max-age=31536000; includeSubDomains` ensures HTTPS. Vercel sets this by default,
  but check if a custom domain overrides it. **Low** if missing (most hosts enforce
  HTTPS anyway), **Medium** if the app handles sensitive data.

- [ ] **`Permissions-Policy` restricts browser features.**
  Limits access to camera, microphone, geolocation, etc. Not critical for most
  web apps but good hygiene. **Info**.

- [ ] **`X-XSS-Protection` is NOT set to `1; mode=block`.**
  This header is deprecated and can actually introduce vulnerabilities in older
  browsers. Set to `0` or omit entirely. Modern CSP is the replacement. **Info**.

### Cookie Security

- [ ] **`Secure` flag is set on all cookies.**
  Ensures cookies are only sent over HTTPS. **Medium** if missing on auth cookies.

- [ ] **`HttpOnly` flag is set on auth/session cookies.**
  Prevents JavaScript from reading the cookie, mitigating XSS token theft.
  **High** if missing on session cookies.

- [ ] **`SameSite` attribute is set.**
  `Lax` (default in modern browsers) or `Strict`. `None` requires `Secure` and
  enables cross-site cookie sending — only appropriate if your app is embedded
  cross-origin. **Medium** if explicitly set to `None` without clear need.

- [ ] **Cookie path is scoped appropriately.**
  Session cookies should typically use `/` path. More sensitive cookies can be
  scoped to specific paths. **Info**.

### Next.js Specific

- [ ] **`next.config.js` headers are applied to the right paths.**
  Headers in `next.config.js` use a `source` pattern. Verify that security headers
  apply to all routes, not just a subset. Common mistake: headers only apply to
  `/:path*` and miss the root `/`. **Low**.

- [ ] **Vercel auto-headers are not overridden.**
  Vercel sets some security headers by default. Custom `vercel.json` header config
  can accidentally override these. **Low**.

## Example Configuration

```javascript
// next.config.js — security headers
const securityHeaders = [
  { key: 'X-Frame-Options', value: 'DENY' },
  { key: 'X-Content-Type-Options', value: 'nosniff' },
  { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
  { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
  {
    key: 'Content-Security-Policy',
    value: [
      "default-src 'self'",
      "script-src 'self' 'nonce-${nonce}'",  // requires nonce implementation
      "style-src 'self' 'unsafe-inline'",     // Tailwind needs this
      "img-src 'self' blob: data: https://*.supabase.co",
      "connect-src 'self' https://*.supabase.co https://api.stripe.com",
      "frame-ancestors 'none'",
    ].join('; '),
  },
  {
    key: 'Strict-Transport-Security',
    value: 'max-age=31536000; includeSubDomains',
  },
];

module.exports = {
  async headers() {
    return [{ source: '/(.*)', headers: securityHeaders }];
  },
};
```
