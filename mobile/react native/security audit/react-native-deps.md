# React Native Dependencies & Supply Chain Audit

## What to Scan

1. **`package.json`**: direct dependencies, scripts, patch-package usage
2. **Lockfiles**: `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`
3. **Native modules**: packages with custom iOS/Android code
4. **Build tooling**: Metro plugins, Babel plugins, codegen, postinstall scripts

## Checklist

### Dependency Risk

- [ ] **Dependencies are actively maintained and reasonably current.**
  Stale auth, storage, or crypto libraries deserve extra scrutiny. **Medium** if
  abandoned packages sit in critical paths.

- [ ] **High-risk packages are minimized.**
  Libraries that handle auth, encryption, WebViews, file access, or native bridges
  should be carefully chosen and few in number. **Info** or **Medium**.

- [ ] **Known-vulnerable packages are identified.**
  Use lockfile-based review and advisories where available. **High** if vulnerable
  packages affect exposed runtime paths.

### Build / Script Safety

- [ ] **Install/build scripts are intentional.**
  `postinstall`, codegen, patch-package, and native setup scripts should be reviewed.
  **Medium** if opaque or surprising.

- [ ] **Private registries and package sources are trusted.**
  Custom tarballs, git dependencies, or local patches increase supply-chain risk.
  **Medium** if present without clear reason.

- [ ] **Debug-only packages do not leak into production behavior.**
  Tooling like Flipper, dev clients, or inspection plugins should not broaden the
  release attack surface. **Low** to **Medium**.

## Common Patterns to Flag

```json
{
  "scripts": {
    "postinstall": "node scripts/download-unverified-binary.js"
  }
}
```
