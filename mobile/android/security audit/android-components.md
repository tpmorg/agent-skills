# Android Components & IPC Audit

## What to Scan

1. **`AndroidManifest.xml`**: All `<activity>`, `<service>`, `<receiver>`, `<provider>` declarations
2. **Intent filters**: Which components are reachable from other apps
3. **Deep link handlers**: How incoming intents are processed
4. **Broadcast receivers**: What actions they respond to, exported status
5. **Content Providers**: URIs, permissions, exported status
6. **Pending Intents**: Mutable vs. immutable, implicit vs. explicit

## Key Concepts

**Exported components** are reachable by other apps on the device. A component is exported if:
- It has `android:exported="true"`, OR
- It has an `<intent-filter>` (implicitly exported on older target SDKs)

Since `targetSdk = 35`, Android requires explicit `exported` declarations. But intent filters
still make components reachable from the outside.

## Checklist

### Activities

- [ ] **`MainActivity` exported status is justified.**
  `MainActivity` with `android:exported="true"` is standard for the launcher activity.
  However, it has multiple intent filters which expand its attack surface:
  
  - Launcher filter — required, **Info**
  - `smartshots://` custom scheme — **Medium** (see deep link section)
  - `https://smartshotsai.com/` root path — **Medium** (OAuth callback, handles tokens)
  - `https://smartshotsai.com/oauth2callback` — **Info** (standard OAuth)
  - `https://app.smartshotsai.com/home` and `/screenshot` — **Info** (navigation only)

- [ ] **Activity `launchMode="singleTask"` is intentional.**
  `singleTask` means new intents are delivered to the existing instance via `onNewIntent()`.
  Verify that `onNewIntent()` validates incoming intent data. An attacker could send
  a crafted intent to the running activity. **Medium** if intent data is used without
  validation.

- [ ] **No other activities are unintentionally exported.**
  Scan for any activity besides `MainActivity` that has `exported="true"` or intent
  filters. **High** if found without justification.

### Services

- [ ] **`FloatingButtonService` is not exported.**
  `android:exported="false"` — correct. Another app cannot bind to or start this. **Info** ✓

- [ ] **`ScreenshotNotificationListenerService` has proper permission.**
  Protected by `android.permission.BIND_NOTIFICATION_LISTENER_SERVICE` — only the
  system can bind to it. `exported="false"` is also set. **Info** ✓

  Note: Notification Listener access is powerful — it can read all notifications.
  Verify the service only processes screenshot-related notifications and doesn't
  log or store notification content from other apps. **Medium** if it processes
  or stores non-screenshot notifications.

- [ ] **`ScreenshotSyncForegroundService` is not exported.**
  `android:exported="false"` and `foregroundServiceType="dataSync"` — correct. **Info** ✓

- [ ] **No bound services expose sensitive operations.**
  If any service uses `onBind()` to return an `IBinder`, check that it's not exported
  and doesn't accept untrusted input. **High** if exposed.

### Broadcast Receivers

- [ ] **`DailyNotificationReceiver` is not exported.**
  `android:exported="false"` — correct. Only the app itself can trigger it.
  But it has an intent filter for a custom action. Since `exported="false"`,
  external apps can't send this intent. **Info** ✓

- [ ] **`BootReceiver` IS exported — verify necessity.**
  `android:exported="true"` is required to receive `BOOT_COMPLETED`. This is
  standard and safe because:
  - `BOOT_COMPLETED` is a system broadcast
  - The receiver should verify the action before processing
  
  **Low** — verify the receiver checks `intent.action == BOOT_COMPLETED` and doesn't
  process other actions that could be sent by malicious apps.

- [ ] **Boot receiver doesn't perform sensitive operations directly.**
  It should only reschedule alarms, not process user data. **Medium** if it does
  more than scheduling.

### Content Providers

- [ ] **`FileProvider` is configured correctly.**
  `exported="false"` with `grantUriPermissions="true"` — this is the standard secure
  pattern. Only apps that receive an explicit grant (via Intent) can access files. **Info** ✓

- [ ] **`FileProvider` paths in `file_paths.xml` are scoped.**
  Check `@xml/file_paths` for overly broad declarations:
  - `<root-path path=".">` — **High**, exposes entire filesystem
  - `<files-path path=".">` — **Medium**, exposes all internal files
  - `<cache-path path="shared/">` — **Info**, properly scoped
  
  The path should only include directories that need to be shared.

- [ ] **`InitializationProvider` (WorkManager) is not exported.**
  `exported="false"` with `tools:node="merge"` — standard WorkManager setup. **Info** ✓

### Pending Intents

- [ ] **PendingIntents use `FLAG_IMMUTABLE` where possible.**
  Mutable PendingIntents (`FLAG_MUTABLE`) can be modified by the receiving app.
  Use `FLAG_IMMUTABLE` unless the intent specifically needs to be modified
  (e.g., inline reply actions). **Medium** if mutable PendingIntents are used
  where immutable would suffice.

  Check notification PendingIntents, alarm PendingIntents, and any PendingIntent
  passed to other apps.

- [ ] **PendingIntents use explicit intents.**
  Implicit PendingIntents (without a target component) could be intercepted by
  a malicious app. **Medium** if implicit PendingIntents are used.

### Deep Link Handling

- [ ] **Intent data is validated in `onNewIntent()` and `onCreate()`.**
  When a deep link arrives, the app receives the URL as `intent.data`. This data
  is attacker-controlled. Verify:
  - URL scheme is expected (https, smartshots)
  - Host matches expected domains
  - Path parameters are validated (no path traversal, no injection)
  - Query/fragment parameters are sanitized
  **High** if intent data is used in database queries or UI rendering without validation.

- [ ] **OAuth tokens from deep links are immediately consumed.**
  When `https://smartshotsai.com/#access_token=...` arrives, the token should be
  extracted, passed to Supabase SDK, and the URL should not be stored in navigation
  history or logged. **Medium** if tokens linger.

### Permissions

- [ ] **`SYSTEM_ALERT_WINDOW` usage is justified and documented.**
  This permission allows drawing over other apps (floating button). It's legitimate
  for this use case but is a sensitive permission that triggers Play Store review.
  **Info** — ensure it's documented in the app description.

- [ ] **`RECORD_AUDIO` usage is justified.**
  Not obviously related to screenshot management. If used for voice notes or
  transcription, it should be documented. If unused, remove it. **Low** if unused,
  **Info** if used and documented.

- [ ] **`SCHEDULE_EXACT_ALARM` alternatives considered.**
  `SCHEDULE_EXACT_ALARM` requires user approval on Android 13+. For daily
  notifications, `setInexactRepeating()` might suffice and doesn't need the
  permission. **Info**.

## Manifest Analysis Summary Template

```
Component Analysis:
- Activities: [count] total, [count] exported
  - [list exported activities and their intent filter surface]
- Services: [count] total, [count] exported
  - [list any exported services and their protections]
- Receivers: [count] total, [count] exported
  - [list exported receivers and what they respond to]
- Providers: [count] total, [count] exported
  - [list providers and their permissions]

Permission Analysis:
- Normal permissions: [list]
- Dangerous permissions: [list with justification status]
- Signature permissions: [list]
- Special permissions: [list, e.g., SYSTEM_ALERT_WINDOW]

Deep Link Surface:
- Custom schemes: [list]
- Verified App Links: [list domains and verification status]
- Unverified HTTPS links: [list any without autoVerify]
```
