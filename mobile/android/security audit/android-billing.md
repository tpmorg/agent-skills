# Android Google Play Billing Audit

## What to Scan

1. **Billing client setup**: `BillingClient` configuration and connection handling
2. **Purchase flow**: How purchases are initiated and acknowledged
3. **Purchase verification**: Client-side vs. server-side validation
4. **Subscription state**: How subscription status is determined and cached
5. **Entitlement logic**: How premium features are gated
6. **Receipt/token handling**: What happens with purchase tokens

## Checklist

### Purchase Verification

- [ ] **Purchases are verified server-side.**
  The #1 billing security issue: client-only verification. The Google Play Billing
  Library confirms a purchase happened, but a modified APK or rooted device can
  fake this. Real verification requires:
  
  1. Client sends purchase token to your server
  2. Server calls Google Play Developer API to validate the token
  3. Server grants entitlement only after validation
  
  If verification is client-only (check `purchaseState == PURCHASED` and done),
  it's **Critical**. If there's no server verification at all, any user with a
  patched APK gets free premium access.

- [ ] **Server-side verification uses Google Play Developer API, not just receipt parsing.**
  The purchase token should be verified via:
  `androidpublisher.purchases.subscriptions.get` (subscriptions) or
  `androidpublisher.purchases.products.get` (one-time purchases).
  
  Just checking the local receipt JSON signature is better than nothing but can
  be bypassed. **High** if only local signature verification.

- [ ] **Purchase acknowledgment happens after server verification.**
  Unacknowledged purchases are refunded after 3 days. The flow should be:
  1. Purchase completes on client
  2. Send token to server for verification
  3. Server verifies and grants entitlement
  4. Client acknowledges purchase (`acknowledgePurchase()`)
  
  If acknowledgment happens before server verification, the user has premium access
  even if verification later fails. **Medium**.

### Subscription State

- [ ] **Subscription status is authoritative from the server, not the client.**
  The server (your Next.js API / Supabase) should be the source of truth for
  subscription status. The client can cache status for offline UX, but must
  re-validate with the server on each app launch or before accessing premium features.
  
  **High** if subscription status is stored only in Room/DataStore and never
  re-verified server-side.

- [ ] **Server receives Google Play Real-Time Developer Notifications (RTDN).**
  For subscription lifecycle events (renewal, cancellation, grace period, hold),
  the server should receive push notifications from Google Play. Without RTDN,
  the server only knows subscription status when the client reports it — which
  can be spoofed. **High** if RTDN isn't set up.

- [ ] **Grace period and account hold are handled.**
  When a payment fails, Google Play may enter a grace period (user keeps access)
  or account hold (user loses access). The app should reflect these states.
  **Medium** if payment failure states aren't handled.

- [ ] **Client-side subscription cache has an expiry.**
  If the app caches "user is premium" locally, that cache should expire (e.g., after
  24 hours) and re-check with the server. Without expiry, a user could cancel their
  subscription but keep premium access on the device until the cache is manually
  cleared. **Medium**.

### Entitlement Logic

- [ ] **Premium features are gated on the server, not just the client.**
  If premium features involve server-side data (AI analysis, cloud sync, etc.),
  the server should enforce the subscription check. Client-side gating is a UX
  convenience only. **High** if server endpoints serve premium data without
  checking subscription status.

- [ ] **No premium features or content are accessible by modifying client code.**
  If premium content is bundled in the APK and hidden by a client-side check,
  it's extractable. Content that should be premium-only should be fetched from
  the server after authorization. **Medium** for code/UI features, **High** for
  premium data/content.

### Token Handling

- [ ] **Purchase tokens are transmitted securely to the server.**
  Tokens should be sent over HTTPS with user authentication. **Medium** if sent
  without auth (allows token replay from a different account).

- [ ] **Purchase tokens are treated as sensitive data.**
  Don't log tokens, store them in plaintext preferences, or include them in
  analytics events. They can be used to query purchase details via Google's API.
  **Low** if logged.

- [ ] **Consumed purchases can't be replayed.**
  For consumable products, once consumed, the token should be marked as used
  server-side. Sending the same token again should not grant another entitlement.
  **High** if tokens aren't tracked server-side.

### BillingClient Configuration

- [ ] **`BillingClient` connection is handled gracefully.**
  Disconnection, retries, and error states should be handled without crashing or
  failing open (granting access when billing status is unknown). **Medium** if the
  app defaults to "premium" when billing connection fails.

- [ ] **`PurchasesUpdatedListener` handles all states.**
  Must handle: `OK`, `USER_CANCELED`, `ITEM_ALREADY_OWNED`, `ERROR`, etc.
  `ITEM_ALREADY_OWNED` in particular should trigger a restore flow, not an error.
  **Low** if error states aren't fully handled.

- [ ] **`queryPurchasesAsync()` is called on app launch.**
  This catches purchases made on another device, restored purchases, and purchases
  that weren't properly acknowledged. Without this, users may lose access after
  reinstalling. **Medium** for UX, **Low** for security.

## Common Patterns to Flag

```kotlin
// CRITICAL: Client-only verification
purchasesUpdatedListener = PurchasesUpdatedListener { result, purchases ->
    if (result.responseCode == BillingResponseCode.OK) {
        purchases?.forEach { purchase ->
            if (purchase.purchaseState == PurchaseState.PURCHASED) {
                // BAD: Just setting local state without server verification
                userPrefs.setPremium(true)
                purchase.acknowledge()
            }
        }
    }
}

// GOOD: Server verification before acknowledgment
purchasesUpdatedListener = PurchasesUpdatedListener { result, purchases ->
    if (result.responseCode == BillingResponseCode.OK) {
        purchases?.forEach { purchase ->
            if (purchase.purchaseState == PurchaseState.PURCHASED) {
                // Send to server for verification
                viewModel.verifyPurchase(
                    token = purchase.purchaseToken,
                    productId = purchase.products.first()
                )
                // Acknowledgment happens after server confirms
            }
        }
    }
}

// BAD: Failing open when billing connection drops
override fun onBillingServiceDisconnected() {
    // If we can't check, assume premium
    _isPremium.value = true  // BAD
}

// GOOD: Failing closed
override fun onBillingServiceDisconnected() {
    // Maintain last known state, retry connection
    retryConnection()
    // Don't upgrade access on failure
}
```

## Integration with Web Audit

If the app uses the same Supabase backend as the web app:

- Verify the web API has an endpoint for purchase token verification
- Check that Supabase has a `subscriptions` or `user_subscriptions` table
- Verify RLS on subscription tables (users shouldn't be able to update their own
  subscription status directly)
- Cross-reference with the Stripe audit if the web app uses Stripe — ensure both
  billing systems update the same source of truth
