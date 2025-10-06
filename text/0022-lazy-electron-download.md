# RFC Template

- Start Date: 2025-09-18
- RFC PR: [electron/rfcs#0022](https://github.com/electron/rfcs/pull/0022)
- Electron Issues:
- Reference Implementation:
- Status: **Proposed**

# Lazy Electron Download (Remove postinstall Script)

## Summary

Remove the `postinstall` script from the `electron` npm package and instead download the Electron binary on-demand when the CLI is first spawned, improving security by allowing users to disable `postinstall` scripts while maintaining the current user experience.

## Motivation

The `postinstall` script in npm packages is a known attack vector in the npm ecosystem. Malicious packages can execute arbitrary code during installation, potentially compromising developer machines and CI/CD pipelines. Many security-conscious developers and organizations disable `postinstall` scripts entirely using `npm install --ignore-scripts` or `.npmrc` configuration. PNPM actually disables these scripts by default.

Currently, the Electron npm package downloads the Electron binary during the `postinstall` phase. While this provides a good initial experience, it forces users who want to disable `postinstall` scripts to either:
1. Accept the security risk of enabling postinstall scripts
2. Manually handle Electron binary downloads through custom tooling
3. Individually allowlist the Electron package (when this is supported)

By moving the download to occur lazily when the `electron` CLI is first invoked, we can:
- Allow security-conscious users to completely disable `postinstall` scripts
- Maintain a transparent experience for most users (binary downloads when they first run `electron`)
- Reduce the attack surface for supply chain attacks

## Guide-level explanation

For most Electron users, this change will be completely transparent. Instead of the Electron binary downloading when you run `npm install electron`, it will download the first time you run the `electron` command:

```bash
npm install electron
# No download happens during installation

npx electron .
# Electron binary downloads here on first run
# Subsequent runs use the cached binary
```

The download happens automatically and includes a progress indicator, just like the current `postinstall` experience.

### For users who disable postinstall scripts

If you currently disable `postinstall` scripts for security reasons (`npm install --ignore-scripts` or `ignore-scripts=true` in `.npmrc`), Electron will now work out of the box:

```bash
# In your .npmrc
ignore-scripts=true

npm install electron
# No download, no errors

npx electron .
# Downloads binary on first run
# Everything works as expected
```

### For edge cases requiring immediate download

Some use cases require the Electron binary to be available immediately after installation (e.g., custom build scripts, Docker container builds). For these scenarios, we should introduce a new `install-electron` CLI command:

```bash
npm install electron
npx install-electron
# Explicitly triggers the download
```

You can use this in your own `postinstall` script if you need the old behavior:

```json
{
  "scripts": {
    "postinstall": "install-electron"
  }
}
```

Or in Dockerfile builds:

```dockerfile
RUN npm install --ignore-scripts
RUN npx install-electron
```

## Reference-level explanation

### Implementation Details

1. **Remove postinstall script** from `package.json`:
   - Remove the `postinstall` script entry
   - Keep existing download utilities for reuse

2. **Modify CLI wrapper** (`cli.js` or equivalent):
   - Check if Electron binary exists at expected path
   - If not present, trigger download before spawning Electron process
   - Show progress indicator during download
   - Handle download errors gracefully with clear error messages

3. **Add `install-electron` binary**:
   - Create new executable that explicitly triggers download
   - Export it in `package.json` `bin` field
   - Reuse existing download logic from current `install.js`

4. **Path resolution**:
   - Keep current binary path resolution logic
   - Ensure download location matches existing conventions
   - Respect `ELECTRON_CUSTOM_DIR` and other environment variables

5. **Error handling**:
   - Clear error messages if download fails
   - Guidance on manual download if automatic download fails
   - Network error retry logic

### Code flow

```javascript
// Simplified example of CLI wrapper logic
async function main() {
  const { getElectronPath } = require('./path');
  const { download } = require('./download');

  let electronPath = getElectronPath();

  if (!fs.existsSync(electronPath)) {
    console.log('Electron binary not found, downloading...');
    await download();
    electronPath = getElectronPath();
  }

  // Spawn Electron process
  spawn(electronPath, process.argv.slice(2), { stdio: 'inherit' });
}
```

### Migration Path

This is a breaking change only for specific edge cases. The migration path is:

1. **Most users**: No action required - download happens on first run instead of install
2. **Users with custom build scripts expecting binary at install time**: Add `install-electron` to their build process
3. **Docker/CI environments**: Either accept lazy download or explicitly run `install-electron`

## Drawbacks

1. **First run delay**: Users will experience a download delay on first run instead of during installation. However, this is only a UX change, not a functional regression.

2. **Breaking change for some build pipelines**: Build systems that expect the binary to exist immediately after `npm install` will need to be updated to run `install-electron` explicitly.

3. **CI cache implications**: Some CI systems cache based on `npm install`. They may need to adjust caching strategies to include the lazy download step.

4. **Error visibility**: Download errors move from install-time to run-time, which could be surprising for some users (though error messages can guide them and it's probably better than electron not downloading doesn't break package installs).

## Alternative designs considered

1. **Keep postinstall but make it optional**: Would require environment variable configuration, less clean than lazy download

2. **Move to a separate package**: Creates confusion and fragments the ecosystem

3. **Require manual download**: Too much friction for the majority of users

4. **Download to global cache only**: Doesn't solve the `postinstall` security issue

### What if we don't do this?

- Security-conscious users continue to need workarounds
- Electron remains incompatible with `ignore-scripts=true`
- Supply chain attack surface remains larger than necessary

## Prior art

Several other tools have moved away from `postinstall` scripts:

1. **Playwright**: Moved to lazy download with `npx playwright install` for explicit installation
2. **Puppeteer**: Provides `PUPPETEER_SKIP_DOWNLOAD` and lazy download options
3. **esbuild**: Downloads native binaries on-demand with fallback mechanisms
4. **Cypress**: Supports `CYPRESS_INSTALL_BINARY=0` to skip postinstall download

These tools show that the ecosystem is moving toward explicit or lazy downloads rather than automatic `postinstall` scripts.
