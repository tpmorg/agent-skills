# Stripe Security Audit

## What to Scan

1. **Webhook handler**: Usually `app/api/webhooks/stripe/route.ts` or similar
2. **Checkout creation**: Server-side code that creates Checkout Sessions
3. **Client-side Stripe usage**: Stripe.js, Elements, or Pricing Table integration
4. **Environment variables**: `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, etc.
5. **Subscription management**: Code that reads/writes subscription status
6. **Package.json**: `stripe` version for known vulnerabilities

## Checklist

### Webhook Security

- [ ] **Webhook handler verifies the signature.**
  Must call `stripe.webhooks.constructEvent(body, sig, webhookSecret)` with the
  raw body (not parsed JSON). If the handler parses JSON first or skips verification,
  anyone can forge webhook events. **Critical**.

- [ ] **Raw body is used for signature verification.**
  In Next.js App Router, this means `await request.text()`, not `await request.json()`.
  In Pages Router, body parsing must be disabled. If the body is parsed before
  signature verification, the signature check will always fail silently or the handler
  might skip it. **Critical**.

  ```typescript
  // App Router — correct
  const body = await request.text();
  const sig = request.headers.get('stripe-signature');
  const event = stripe.webhooks.constructEvent(body, sig!, webhookSecret);

  // Pages Router — must disable body parsing
  export const config = { api: { bodyParser: false } };
  ```

- [ ] **Webhook handler returns 200 quickly.**
  Long-running webhook handlers cause Stripe to retry, potentially processing
  events multiple times. Heavy work should be queued. **Low**.

- [ ] **Webhook handler is idempotent.**
  Stripe may deliver the same event multiple times. The handler should check if
  the event has already been processed (e.g., by storing `event.id`). **Medium** if
  the webhook triggers side effects like sending emails or provisioning access.

- [ ] **All relevant event types are handled.**
  For subscriptions, at minimum handle: `checkout.session.completed`,
  `customer.subscription.updated`, `customer.subscription.deleted`,
  `invoice.payment_failed`. Missing event types can leave subscription state
  out of sync. **High** if subscription state is only set on checkout and never
  updated on change/cancellation.

### Checkout & Pricing

- [ ] **Checkout Sessions are created server-side.**
  The price, quantity, and other parameters must come from the server, not the client.
  If the client sends a `price_id` and the server blindly uses it, a user could
  substitute a cheaper price. **High** if price comes from client without validation.

- [ ] **Price IDs are validated against an allowlist.**
  Even server-side, if the price ID comes from a client selection, validate it against
  known-good price IDs. **Medium**.

- [ ] **`success_url` and `cancel_url` are hardcoded or validated.**
  If these come from client input, an attacker could redirect post-checkout to a
  phishing page. **Medium**.

- [ ] **`client_reference_id` or metadata ties the session to the user.**
  After checkout completes, the webhook needs to know which user to provision.
  This should be set server-side from the authenticated user's ID, not from
  client input. **High** if the user ID comes from an unverified source.

### Key Management

- [ ] **`STRIPE_SECRET_KEY` is never in client-side code.**
  Grep for the secret key pattern (`sk_live_*` or `sk_test_*`) in any file that
  could reach the client. Also check that it's not in a `NEXT_PUBLIC_*` env var.
  **Critical** if exposed.

- [ ] **`STRIPE_WEBHOOK_SECRET` is set and not hardcoded.**
  Should come from environment variable, not committed to source. **High** if
  hardcoded in source.

- [ ] **Publishable key is used client-side (not secret key).**
  The `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` (`pk_live_*` / `pk_test_*`) is the only
  key that should appear in client code. **Info** — usually correct but worth verifying.

- [ ] **Test vs. live key separation.**
  Check that test keys aren't used in production configuration and vice versa.
  **Medium** if test keys appear in production env files.

### Subscription State

- [ ] **Subscription status is stored server-side and synced via webhooks.**
  If the app checks subscription status by calling the Stripe API on every request,
  it's slow but secure. If it caches status locally (in Supabase), that cache must
  be updated by webhooks. **High** if local cache exists but no webhook updates it.

- [ ] **Subscription checks happen server-side.**
  Client-side subscription checks can be bypassed. Premium features should be gated
  on the server (API routes, middleware, or RLS policies). **High** if gating is
  client-only.

- [ ] **Grace period / failed payment handling exists.**
  When `invoice.payment_failed` fires, does the app downgrade access or notify the user?
  **Medium** if payments can fail silently with continued access.

### Customer Portal & Billing

- [ ] **Customer portal session creation is authenticated.**
  The endpoint that creates a Stripe Customer Portal session must verify the requesting
  user matches the Stripe customer. **High** if any authenticated user can create a
  portal session for any customer.

- [ ] **No direct Stripe customer ID exposure to client.**
  The Stripe customer ID (`cus_*`) shouldn't be in client-visible API responses
  or URLs. It's not directly exploitable but reduces attack surface. **Low**.

## Common Patterns to Flag

```typescript
// BAD: No webhook signature verification
export async function POST(request: Request) {
  const event = await request.json(); // parsed, not raw!
  // processing event directly...
}

// BAD: Price ID from client without validation
export async function POST(request: Request) {
  const { priceId } = await request.json();
  const session = await stripe.checkout.sessions.create({
    line_items: [{ price: priceId, quantity: 1 }], // client controls price!
  });
}

// GOOD: Validated price ID
const ALLOWED_PRICES = ['price_xxx', 'price_yyy'];
export async function POST(request: Request) {
  const { priceId } = await request.json();
  if (!ALLOWED_PRICES.includes(priceId)) {
    return NextResponse.json({ error: 'Invalid price' }, { status: 400 });
  }
  // ...
}

// BAD: Subscription check client-side only
function PremiumFeature() {
  const { subscription } = useUser();
  if (subscription.status !== 'active') return <UpgradePrompt />;
  return <SecretContent />; // content is in the JS bundle regardless
}

// GOOD: Server-side gate
export async function GET(request: Request) {
  const user = await getAuthenticatedUser(request);
  const sub = await getSubscriptionStatus(user.id); // from DB, synced by webhooks
  if (sub.status !== 'active') {
    return NextResponse.json({ error: 'Subscription required' }, { status: 403 });
  }
  return NextResponse.json({ secretData: '...' });
}
```
