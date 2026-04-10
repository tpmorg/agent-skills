# Android Dependency & Supply Chain Audit

## What to Scan

1. **`build.gradle.kts`**: Direct dependencies and versions
2. **Version catalog**: `libs.versions.toml` for centralized version management
3. **Transitive dependencies**: `./gradlew dependencies` output
4. **ProGuard/R8 rules**: `proguard-rules.pro` — what's kept, what's stripped
5. **Gradle plugins**: Plugin versions and sources
6. **Build script security**: `local.properties` handling, build-time code execution

## Checklist

### Known Vulnerabilities

- [ ] **Check pinned dependency versions against known CVEs.**
  For each significant dependency, check if the version has known vulnerabilities:
  
  Key dependencies to check (from the Gradle file):
  - `com.google.mlkit:text-recognition:16.0.1`
  - `com.google.mlkit:image-labeling:17.0.9`
  - `com.google.mlkit:object-detection:17.0.2`
  - `com.jakewharton.timber:timber:5.0.1`
  - `androidx.browser:browser:1.8.0`
  - OkHttp version (via version catalog)
  - Retrofit version (via version catalog)
  - Ktor version (via version catalog)
  - Supabase BOM version (via version catalog)
  - Hilt version (via version catalog)
  
  **High** for any dependency with a known exploitable CVE. **Medium** for CVEs
  that are theoretical or require unusual conditions.

  Note: This check requires web access or a local vulnerability database.
  Flag as a manual step if automated scanning isn't available.

- [ ] **Dependencies use stable versions, not snapshots or alphas.**
  Snapshot and alpha builds may have undiscovered vulnerabilities and aren't
  guaranteed to be reproducible. **Low** if alpha versions are in the dependency tree.

- [ ] **BOM (Bill of Materials) is used for Compose and Supabase.**
  BOMs ensure consistent, tested version combinations. Present for both
  `androidx.compose.bom` and `supabase.bom` — **Info** ✓. Verify the BOM versions
  are recent.

### Dependency Scope

- [ ] **Debug-only dependencies aren't in release builds.**
  Check for dependencies that should be `debugImplementation` but are `implementation`:
  - `androidx.ui.tooling` — should be `debugImplementation` (correctly set ✓)
  - `androidx.ui.test.manifest` — should be `debugImplementation` (correctly set ✓)
  - Any logging interceptors, debug drawers, or testing tools
  **Medium** if debug tools are included in release.

- [ ] **Test dependencies are properly scoped.**
  `testImplementation` and `androidTestImplementation` don't ship in the APK.
  Verify no test frameworks are in `implementation`. **Low** if misscoped.

- [ ] **`kapt` dependencies are compile-time only.**
  Annotation processors (Room compiler, Hilt compiler) should only run at compile
  time. They're correctly using `kapt` scope. **Info** ✓

### ProGuard / R8

- [ ] **R8 is enabled for release builds.**
  `isMinifyEnabled = true` — confirmed. This obfuscates code, making reverse
  engineering harder. **Info** ✓

- [ ] **`isShrinkResources = true` for release.**
  Removes unused resources. **Info** ✓

- [ ] **ProGuard rules don't keep too much.**
  Check `proguard-rules.pro` for overly broad `-keep` rules:
  - `-keep class * { *; }` — defeats the purpose entirely, **Medium**
  - `-keepnames class **` — keeps all class names, reduces obfuscation, **Low**
  - Keeping entire model/data classes is fine and necessary for serialization

- [ ] **Serialization classes are correctly kept.**
  Kotlin serialization, Gson, and Retrofit require certain classes to be kept.
  Missing keep rules cause runtime crashes. Not a security issue, but verify
  correctness. **Info**.

- [ ] **ProGuard mapping file is saved for crash reporting.**
  `mapping.txt` is needed to deobfuscate stack traces. It should be uploaded to
  Play Console or crash reporting service, not committed to the repo. **Info**.

### Build Script Security

- [ ] **`local.properties` is loaded safely.**
  The build script loads `local.properties` with a file existence check — correct.
  Properties are accessed with defaults for missing values — correct. **Info** ✓

- [ ] **No remote script execution in build files.**
  Check for `apply from: "https://..."` or `buildscript` blocks that download
  scripts from external URLs. **Critical** if found — allows build-time code injection.

- [ ] **Plugin sources are official repositories.**
  Verify all plugins come from `google()`, `mavenCentral()`, or Gradle Plugin Portal.
  **High** if custom/unknown repositories are used.

- [ ] **Gradle wrapper version is current.**
  `gradle-wrapper.properties` should use a recent, non-vulnerable Gradle version.
  **Medium** if significantly outdated.

- [ ] **Gradle wrapper checksum is verified.**
  `distributionSha256Sum` in `gradle-wrapper.properties` prevents supply chain
  attacks via tampered Gradle distributions. **Low** if missing.

### Sensitive Dependency Behaviors

- [ ] **Timber logging is disabled or filtered in production.**
  Timber with a `DebugTree` logs everything. In release builds, either plant no
  tree or plant a tree that only logs warnings/errors without sensitive data.
  **Medium** if `DebugTree` is planted in release.

  ```kotlin
  // Check Application.onCreate():
  if (BuildConfig.DEBUG) {
      Timber.plant(Timber.DebugTree())
  }
  // No tree in release = no logging = good
  ```

- [ ] **ML Kit doesn't phone home with user data.**
  Google ML Kit (on-device) processes data locally by default. However, model
  download and usage analytics may be sent to Google. Check if there's opt-out
  configuration. **Info** — generally acceptable, but note in privacy assessment.

- [ ] **Supabase SDK default logging level is appropriate.**
  The Supabase Kotlin SDK can log requests at various levels. Check the client
  builder configuration for logging settings. **Medium** if verbose in production.

## Dependency Audit Template

```
Dependency Audit Results:
| Dependency | Version | Latest | CVEs | Status |
|---|---|---|---|---|
| OkHttp | x.y.z | a.b.c | CVE-XXXX-YYYY | Update recommended |
| Ktor | x.y.z | a.b.c | None known | OK |
| ... | ... | ... | ... | ... |

ProGuard Analysis:
- R8 enabled: Yes/No
- Resource shrinking: Yes/No
- Custom rules: [count] rules
- Overly broad rules: [list any concerning patterns]

Build Security:
- Local properties handling: OK / Issue
- Plugin sources: All official / [list unofficial]
- Gradle wrapper: version X.Y, checksum: verified/missing
```
