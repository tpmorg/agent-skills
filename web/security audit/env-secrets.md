# Env & Secrets Hygiene Audit

## What to Scan

1. **`.env*` files**: `.env`, `.env.local`, `.env.production`, `.env.example`
2. **`.gitignore`**: Verify env files are excluded
3. **`next.config.*`**: Environment variable forwarding, `env` and `publicRuntimeConfig`
4. **Source files**: Hardcoded secrets, API keys, connection strings
5. **CI/CD config**: GitHub Actions workflows, Vercel project settings references
6. **`package.json` scripts**: Inline env vars in scripts

## Checklist

### NEXT_PUBLIC_ Exposure

- [ ] **Audit every `NEXT_PUBLIC_*` variable.**
  These are embedded in the client JS bundle and visible to anyone. Each one should
  be intentionally public. **Critical** if any of these are secret keys:
  - `NEXT_PUBLIC_SUPABASE_SERVICE_ROLE_KEY` → **Critical** (this is the #1 mistake)
  - `NEXT_PUBLIC_STRIPE_SECRET_KEY` → **Critical**
  - Any `*_SECRET*`, `*_KEY*` (non-publishable), `*_PASSWORD*` → **Critical**

- [ ] **Only these should typically be `NEXT_PUBLIC_`:**
  - `NEXT_PUBLIC_SUPABASE_URL` — the project URL (public by design)
  - `NEXT_PUBLIC_SUPABASE_ANON_KEY` — the anon/public key (public by design)
  - `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` — the pk_* key
  - App configuration: site URL, feature flags, analytics IDs
  - Any value you'd be comfortable pasting into a public GitHub issue

### .gitignore Coverage

- [ ] **`.env.local` is in `.gitignore`.**
  This is the standard location for local secrets. **High** if missing.

- [ ] **All `.env*` files except `.env.example` are gitignored.**
  Check for `.env`, `.env.local`, `.env.production.local`, `.env.development.local`.
  **High** if any secret-containing env file is committed.

- [ ] **`.env.example` contains no real values.**
  It should have placeholder values like `your-key-here` or be empty. **High** if it
  contains actual API keys (even test ones, which can reveal account structure).

### Hardcoded Secrets

- [ ] **No API keys or secrets in source code.**
  Grep for patterns:
  - `sk_live_`, `sk_test_` (Stripe secret keys)
  - `eyJ` (JWT tokens, base64-encoded)
  - `supabase` + long alphanumeric strings
  - `password`, `secret`, `token` followed by `=` or `:` with a literal value
  - `ghp_`, `gho_` (GitHub tokens)
  - `AIza` (Google API keys)
  - Connection strings with credentials

  **Critical** for any production secret. **High** for test secrets.

- [ ] **No secrets in `next.config.js` env section.**
  Values in `next.config.js`'s `env: {}` are inlined into the client bundle.
  This is equivalent to `NEXT_PUBLIC_`. **Critical** if secrets are here.

### Supabase-Specific

- [ ] **`SUPABASE_SERVICE_ROLE_KEY` is server-only.**
  This key bypasses RLS entirely. It must never be in a `NEXT_PUBLIC_` var or
  in any file that reaches the client. **Critical** if exposed.

- [ ] **Supabase connection string / database URL is server-only.**
  Direct database access (Prisma, Drizzle, raw pg) connection strings contain
  credentials. Must be server-only. **Critical** if exposed.

- [ ] **Multiple Supabase client instantiations are consistent.**
  Check that every `createClient()` or `createServerClient()` call uses the
  correct key for its context (anon for client, service role for server admin tasks,
  server client with cookies for authenticated server operations). **High** if mixed up.

### CI/CD & Deployment

- [ ] **GitHub Actions don't echo secrets.**
  Check workflow files for `echo ${{ secrets.* }}` or debug steps that might
  print environment variables. **High** if found.

- [ ] **Build logs don't contain secrets.**
  Next.js build output can include env var values in certain error conditions.
  This is hard to check statically but worth noting. **Info**.

- [ ] **Vercel environment variables are scoped correctly.**
  Production secrets should only be in the Production environment, not Preview
  or Development (which can be accessed by anyone with branch push access). **Medium**
  — this can't be checked from source, but flag as a manual verification step.

## Common Patterns to Flag

```bash
# BAD: Secret key as NEXT_PUBLIC
NEXT_PUBLIC_SUPABASE_SERVICE_ROLE_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6...

# GOOD: Service role is server-only
SUPABASE_SERVICE_ROLE_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6...

# BAD: Secret in next.config.js env (inlined into client bundle)
module.exports = {
  env: {
    STRIPE_SECRET_KEY: process.env.STRIPE_SECRET_KEY,
  },
};

# BAD: .env.example with real values
SUPABASE_URL=https://abcdefg.supabase.co
SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZ...
```

## Manual Verification Steps

These can't be checked from source but should be included in the report as action items:

1. Verify Vercel environment variable scoping (Production vs Preview vs Development)
2. Rotate any keys that were ever committed to source control (even if removed later —
   they're in git history)
3. Review Supabase dashboard → Settings → API for key exposure
4. Check Stripe Dashboard → Developers → API keys for any unexpected keys
