# Client-Side Exposure Audit

## What to Scan

1. **Client components**: Files with `"use client"` directive
2. **Layout and page files**: `layout.tsx`, `page.tsx` — determine if server or client
3. **API response shapes**: What data do API routes return to the client?
4. **React context / state**: What's stored in global state accessible to all components?
5. **Client-side routing guards**: How are protected pages gated?
6. **Bundle contents**: What gets shipped to the browser?

## Checklist

### Data Leakage

- [ ] **API responses don't include unnecessary fields.**
  When a server route queries `SELECT * FROM users` and returns the result,
  the client receives every column — potentially including `stripe_customer_id`,
  `role`, `email_verified`, internal flags. Map to a DTO/view model before responding.
  **Medium** for most fields, **High** if it includes tokens, keys, or admin flags.

- [ ] **Server components don't pass sensitive props to client components.**
  A server component can access secrets, but if it passes them as props to a
  `"use client"` component, they end up in the client bundle. Check the boundary
  between server and client components. **High** if sensitive data crosses.

- [ ] **No sensitive data in `useSearchParams`, URL params, or localStorage.**
  Tokens, user IDs, or subscription details in the URL are visible in browser history,
  referrer headers, and logs. **Medium**.

- [ ] **Console.log statements don't leak sensitive data in production.**
  Debug logging that prints user objects, tokens, or API responses should be removed
  or conditionally disabled. **Low** but easy to fix.

### Authorization in the Client

- [ ] **Client-side route guards are backed by server-side checks.**
  A client-side `if (!user) redirect('/login')` is a UX convenience, not a security
  boundary. The API routes and server components must independently verify auth.
  **High** if client guard is the _only_ protection and the protected page fetches
  sensitive data in `useEffect` without server auth.

- [ ] **Premium/gated content isn't in the JS bundle.**
  If premium features are conditionally rendered client-side, the code (and potentially
  the data) is still shipped to the browser. A user can see it in DevTools.
  Server Components or API-gated data fetching is the fix. **Medium** for code,
  **High** for data.

- [ ] **Admin UI isn't accessible by URL.**
  If `/admin` exists and is protected only by a client-side role check, any user
  can navigate there and see the admin layout (even if the data calls fail).
  Use middleware or server-side redirect. **Medium**.

### Supabase Client in Browser

- [ ] **Only the anon key is used in client-side Supabase client.**
  The `createBrowserClient()` should only receive `NEXT_PUBLIC_SUPABASE_URL` and
  `NEXT_PUBLIC_SUPABASE_ANON_KEY`. If any other key appears here, it's **Critical**.

- [ ] **Client-side Supabase queries rely on RLS for access control.**
  When the client calls `supabase.from('table').select(...)`, RLS policies are
  the security boundary. This is fine if RLS is correctly configured (see RLS audit).
  Flag as **Info** to cross-reference with RLS findings.

- [ ] **Real-time subscriptions are filtered.**
  If using Supabase Realtime, check that `.on('postgres_changes', { filter: ... })`
  includes a filter. Without a filter, the client subscribes to all changes on the
  table (subject to RLS, but the intent may be narrower). **Low**.

### Third-Party Scripts

- [ ] **Analytics scripts don't receive sensitive data.**
  Check event tracking calls for user emails, payment info, or internal IDs being
  sent as event properties. **Medium** depending on the data and the analytics provider.

- [ ] **Third-party scripts are loaded with integrity checks or from trusted CDNs.**
  `<script src="...">` without `integrity` attributes or from unknown CDNs is a
  supply chain risk. **Low** for well-known providers, **Medium** for lesser-known ones.

### Server-Side Rendering Hydration

- [ ] **`getServerSideProps` / server component data doesn't over-fetch.**
  Data fetched server-side for SSR is serialized into the HTML as `__NEXT_DATA__`
  or RSC payload. View source on a rendered page to check what data is embedded.
  **Medium** if sensitive fields are included in the page payload.

- [ ] **Error boundaries don't render sensitive data.**
  Custom error boundaries or error.tsx files should show generic messages, not
  the error object which might contain query details or stack traces. **Low**.

## What to Look For in the Bundle

While we can't run a production build in a static audit, flag these patterns
that indicate potential bundle exposure:

```typescript
// BAD: Service key in client component
"use client"
import { createClient } from '@supabase/supabase-js'
const supabase = createClient(url, process.env.SUPABASE_SERVICE_ROLE_KEY!)

// BAD: Sensitive data passed as prop across the server/client boundary
// Server component
export default async function Dashboard() {
  const adminData = await getAdminConfig(); // includes API keys
  return <ClientDashboard config={adminData} /> // serialized to client!
}

// BAD: Premium content gated only by client-side check
"use client"
function PremiumArticle({ content, isPremium }) {
  // content is already in the bundle — checking isPremium is just UI
  if (!isPremium) return <Paywall />;
  return <article>{content}</article>;
}

// GOOD: Premium content fetched server-side with auth
export default async function PremiumArticle() {
  const user = await getUser();
  const sub = await getSubscription(user.id);
  if (sub.status !== 'active') redirect('/upgrade');
  const content = await getPremiumContent(); // only fetched if authorized
  return <article>{content}</article>;
}
```
