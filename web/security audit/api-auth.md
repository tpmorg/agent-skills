# API Auth & Middleware Audit

## What to Scan

1. **API route files**: `app/api/**/route.ts` (App Router) or `pages/api/**/*.ts` (Pages Router)
2. **Middleware**: `middleware.ts` at project root or in `src/`
3. **Server actions**: Files with `"use server"` directive
4. **Auth utilities**: Shared auth helpers, session validation functions
5. **Supabase server client creation**: How the server-side Supabase client is instantiated

## Checklist

### Route Protection

- [ ] **Every API route that accesses user data validates the session.**
  Look for routes that call `supabase.from(...)` or access any data store without
  first checking `supabase.auth.getUser()` or equivalent. **Critical** if a data-mutating
  route (POST/PUT/DELETE) lacks auth. **High** if a data-reading route (GET) lacks auth
  on private data.

- [ ] **Auth check uses `getUser()`, not `getSession()` alone.**
  `getSession()` reads from the JWT without validating it against Supabase's auth server.
  A tampered JWT would pass `getSession()` but fail `getUser()`. If routes rely solely
  on `getSession()` for authorization decisions, flag as **High**.

- [ ] **Admin/elevated routes have role checks beyond basic auth.**
  Routes like `/api/admin/*` should verify the user's role, not just that they're logged in.
  Check for role validation in the handler or middleware. **Critical** if admin routes
  only check for a valid session.

- [ ] **Webhook endpoints skip auth but verify signatures.**
  Routes that receive webhooks (Stripe, etc.) legitimately skip user auth, but must
  verify the webhook signature. If a webhook route has no signature verification,
  flag as **Critical**.

- [ ] **Cron endpoints verify the cron secret.**
  Routes triggered by cron jobs (Vercel Cron, etc.) should check a `CRON_SECRET`
  header or similar mechanism. Without this, anyone can trigger the cron job. **High**.

### Middleware

- [ ] **Middleware refreshes the auth session.**
  Supabase auth in Next.js requires middleware to refresh the session cookie.
  Check that `supabase.auth.getUser()` is called in middleware on protected routes.
  Missing this means sessions can expire while the user appears logged in. **Medium**.

- [ ] **Middleware matcher covers all protected routes.**
  Check the `config.matcher` export. If it's too narrow, some protected routes
  may not have session refresh. Compare matcher patterns against actual API routes. **Medium**.

- [ ] **Middleware doesn't accidentally protect public routes.**
  Static assets, public pages, and webhook endpoints should be excluded from
  auth middleware. Not a security issue per se, but **Info** for correctness.

### Input Validation

- [ ] **Request body is validated before use.**
  Look for `request.json()` or `request.body` being destructured and used directly
  without schema validation (zod, yup, etc.). **Medium** for most routes,
  **High** for routes that write to the database.

- [ ] **URL parameters and query strings are validated.**
  Dynamic route params (`[id]`) and `searchParams` should be validated as the expected
  type (UUID, number, etc.) before being used in queries. **Medium**.

- [ ] **File uploads validate type, size, and content.**
  If any route accepts file uploads, check for MIME type validation, size limits,
  and that the file is actually the type it claims. **Medium**.

### Response Security

- [ ] **API routes don't leak sensitive data in error messages.**
  Look for `catch` blocks that return the full error object or stack trace.
  Production routes should return generic error messages. **Low** for most,
  **Medium** if errors could reveal database schema or internal paths.

- [ ] **API routes don't return more data than the client needs.**
  A common pattern: selecting `*` from a table and returning it, which may include
  columns the client doesn't need (like internal IDs, timestamps, or soft-delete flags).
  **Low** but worth flagging.

- [ ] **Appropriate HTTP status codes are used.**
  401 for unauthenticated, 403 for unauthorized (authenticated but not allowed),
  404 for not found. Returning 200 with an error body masks issues. **Info**.

### Server Actions

- [ ] **Server actions validate auth at the top.**
  `"use server"` functions are callable from the client. Each one that accesses
  protected data should start with an auth check. **High** if missing.

- [ ] **Server actions validate and sanitize all arguments.**
  Arguments to server actions come from the client and can be tampered with.
  Treat them as untrusted input. **Medium** if not validated.

- [ ] **Server actions don't accept an ID and act on it without ownership check.**
  Pattern: `async function deleteItem(id: string)` — this must verify the current user
  owns the item before deleting. **Critical** if ownership isn't checked.

### Rate Limiting

- [ ] **Auth endpoints have rate limiting.**
  Login, signup, password reset, and magic link endpoints should be rate limited
  to prevent brute force attacks. **Medium** if missing.

- [ ] **Expensive operations are rate limited.**
  AI processing, bulk exports, or any resource-intensive endpoint should have
  some form of rate limiting. **Low** if missing.

### CSRF Protection

- [ ] **State-changing operations validate origin.**
  Next.js App Router API routes using cookies for auth should verify the `Origin`
  or `Referer` header matches the expected domain. **Medium**.

- [ ] **SameSite cookie attribute is set.**
  Check Supabase cookie configuration. `SameSite=Lax` is the minimum for CSRF
  protection. **Medium** if set to `None` without good reason.

## Common Patterns to Flag

```typescript
// BAD: No auth check
export async function POST(request: Request) {
  const { data } = await request.json();
  const supabase = createClient();
  await supabase.from('items').insert(data);
  return NextResponse.json({ success: true });
}

// GOOD: Auth check before data access
export async function POST(request: Request) {
  const supabase = createClient();
  const { data: { user }, error } = await supabase.auth.getUser();
  if (!user) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });

  const { data } = await request.json();
  // validate data with zod schema...
  await supabase.from('items').insert({ ...validatedData, user_id: user.id });
  return NextResponse.json({ success: true });
}

// BAD: Using getSession() for auth decisions
const { data: { session } } = await supabase.auth.getSession();
if (session) { /* proceed */ }

// GOOD: Using getUser() which validates the JWT
const { data: { user } } = await supabase.auth.getUser();
if (user) { /* proceed */ }

// BAD: Server action without auth
"use server"
export async function deleteScreenshot(id: string) {
  const supabase = createClient();
  await supabase.from('screenshots').delete().eq('id', id);
}

// GOOD: Server action with auth + ownership
"use server"
export async function deleteScreenshot(id: string) {
  const supabase = createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) throw new Error('Unauthorized');
  // RLS will also enforce this, but belt-and-suspenders
  await supabase.from('screenshots').delete().eq('id', id).eq('user_id', user.id);
}
```
