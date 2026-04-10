# Android Local Data Storage Audit

## What to Scan

1. **Room database**: Schema, what data is stored, encryption status
2. **DataStore preferences**: What key-value pairs are persisted
3. **File storage**: Cache dir, files dir, external storage usage
4. **Supabase SDK local cache**: Where the SDK stores offline data
5. **Backup configuration**: `backup_rules.xml`, `data_extraction_rules.xml`
6. **WebView cache**: If any WebView is used, local storage and cookies

## Checklist

### Room Database

- [ ] **Database is not encrypted.**
  By default, Room uses plaintext SQLite. On a rooted device or via ADB backup,
  the entire database is readable. Assess what's stored:
  - Screenshot metadata (filenames, timestamps, tags) — **Low** risk unencrypted
  - OCR text content — **Medium** if it captures sensitive screen content
  - User profile data, email — **Medium** unencrypted
  - Auth tokens or API keys in the database — **Critical** unencrypted
  
  If sensitive data is stored, recommend SQLCipher for Room or Android's
  file-based encryption. **Medium** overall for most apps.

- [ ] **Database WAL/journal files are in private storage.**
  Room by default writes to the app's private directory, which is fine. Verify
  no custom `Room.databaseBuilder()` path points to external/shared storage. **High**
  if database is on external storage.

- [ ] **Old database versions are migrated, not dropped.**
  `fallbackToDestructiveMigration()` deletes all data on schema change. While not
  a direct security issue, it can mask data handling problems. **Info**.

### DataStore / SharedPreferences

- [ ] **Audit all DataStore keys for sensitive data.**
  Common items stored in preferences:
  - User settings (theme, notification prefs) — **Info**, fine in plaintext
  - Auth tokens or session data — **Medium** if not encrypted
  - User ID, email — **Low** in plaintext
  - Onboarding state, feature flags — **Info**, fine

- [ ] **No SharedPreferences with `MODE_WORLD_READABLE`.**
  Deprecated but still possible. This makes prefs readable by other apps. **Critical**
  if found, though very rare in modern code.

- [ ] **Preference file names don't contain user identifiers.**
  File is stored in `/data/data/<package>/shared_prefs/`. If the filename itself
  contains a user email or ID, it's visible in directory listings. **Low**.

### File Storage

- [ ] **Screenshots and images are stored in app-private directories.**
  `context.filesDir` or `context.cacheDir` (internal) vs. external storage.
  If images are in external storage (`getExternalFilesDir()`), they're readable
  by any app with `READ_EXTERNAL_STORAGE` on older APIs. **Medium** if sensitive
  screenshots are stored externally on API < 30.

  Note: On API 30+ (scoped storage), external app-specific dirs are private.
  Since `minSdk = 31`, external app dirs are fine from a permissions standpoint,
  but still accessible via ADB or on rooted devices.

- [ ] **Cache directory is used for temporary files, not persistent sensitive data.**
  Cache can be cleared by the system at any time. If sensitive data needs to persist,
  it should be in `filesDir` (ideally encrypted). **Info**.

- [ ] **`FileProvider` paths are appropriately scoped.**
  Check `file_paths.xml` — it defines what directories the `FileProvider` can share.
  Overly broad paths (like root `/`) expose more than intended. **Medium** if the
  path declaration is too permissive.

- [ ] **Temporary files are cleaned up after use.**
  Screenshot processing intermediates, cropped images, ML Kit outputs should be
  deleted when no longer needed. Orphaned sensitive files increase exposure. **Low**.

### Backup & Data Extraction

- [ ] **`android:allowBackup="true"` — audit backup rules.**
  ADB backup can extract app data including databases and preferences.
  With `allowBackup="true"`, check:
  - `backup_rules.xml` — does it exclude sensitive files?
  - `data_extraction_rules.xml` — does it limit cloud backup scope?
  
  If these rules include the Room database or token storage, an attacker with
  physical access could extract them via `adb backup`. **Medium**.

- [ ] **`data_extraction_rules.xml` excludes sensitive directories.**
  This controls Android 12+ cloud backup. Should exclude:
  - Token/session storage files
  - Cached auth data
  - Temporary processing files
  
  **Medium** if sensitive data is included in cloud backup.

- [ ] **Consider `android:allowBackup="false"` for high-security apps.**
  If the app stores sensitive user content (screenshots of banking apps, personal
  messages, etc.), disabling backup entirely may be appropriate. **Info** — 
  trade-off between user convenience and security.

### ML Kit / Processing Outputs

- [ ] **OCR results are stored securely.**
  Text recognized from screenshots could contain anything — passwords, financial
  data, messages. If OCR text is stored in plaintext Room database or cache files,
  it's as sensitive as the original screenshot. **Medium**.

- [ ] **ML Kit models don't cache sensitive intermediates.**
  ML Kit may download models and cache processing data. Verify cache is in
  private storage. **Low** — ML Kit handles this correctly by default.

- [ ] **Image labeling results don't leak to analytics.**
  If labels/tags from `image-labeling` are sent to analytics or logging endpoints,
  they could reveal the content of users' screenshots. **Medium** if logged.

## Common Patterns to Flag

```kotlin
// BAD: Database on external storage
Room.databaseBuilder(
    context,
    AppDatabase::class.java,
    "${context.getExternalFilesDir(null)}/app.db"  // accessible via ADB
)

// GOOD: Internal storage (default)
Room.databaseBuilder(
    context,
    AppDatabase::class.java,
    "app.db"  // stored in /data/data/<package>/databases/
)

// BAD: Sensitive data in backup
// backup_rules.xml
<full-backup-content>
    <include domain="database" path="." />  // includes all databases
</full-backup-content>

// GOOD: Exclude sensitive data from backup
<full-backup-content>
    <exclude domain="database" path="auth.db" />
    <exclude domain="sharedpref" path="auth_prefs.xml" />
</full-backup-content>

// BAD: FileProvider with overly broad paths
// file_paths.xml
<paths>
    <root-path name="root" path="." />  // exposes everything!
</paths>

// GOOD: Scoped FileProvider paths
<paths>
    <cache-path name="shared_screenshots" path="shared/" />
</paths>
```
