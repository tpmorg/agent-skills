# React Native Permissions & Native Surface Audit

## What to Scan

1. **Requested permissions**: camera, microphone, photos, contacts, location, notifications
2. **Permission libraries**: Expo permissions, `react-native-permissions`, native modules
3. **Native manifests**: `AndroidManifest.xml`, `Info.plist`
4. **Native modules and bridges**: custom modules, exposed native capabilities

## Checklist

### Permission Scope

- [ ] **Only necessary permissions are requested.**
  Unused permissions expand the attack surface and create privacy risk. **Low** or
  **Medium** depending on sensitivity.

- [ ] **Permissions are requested just-in-time, not preemptively.**
  Broad startup permission prompts reduce trust and can expose unnecessary access.
  **Info** to **Low**.

- [ ] **Permission rationale matches actual use.**
  Misleading or overly broad usage descriptions can hide risky collection behavior.
  **Low** if inaccurate, **Medium** if it masks unexpected access.

### Native Surface Exposure

- [ ] **Custom native modules do not expose privileged actions without checks.**
  Any bridge that writes files, accesses device identifiers, or invokes native APIs
  should have narrowly defined inputs. **High** if overbroad.

- [ ] **Clipboard, screenshots, and background tasks are handled intentionally.**
  Sensitive data copied to the clipboard, exposed in screenshots, or processed in the
  background can leak unexpectedly. **Medium** if unmanaged for sensitive apps.

- [ ] **Dev-only tooling is removed or disabled in release.**
  Debug menus, inspection tools, or verbose native logging should not remain active.
  **Medium** if release builds can expose internal state.

## Common Patterns to Flag

```ts
// BAD: Asking for location on first app launch without feature context
await request(PERMISSIONS.IOS.LOCATION_ALWAYS);

// BETTER: Request only when the feature is invoked
```
