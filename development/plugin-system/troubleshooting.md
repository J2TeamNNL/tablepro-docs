# Plugin Troubleshooting Guide

Common issues when developing and installing TablePro plugins, with solutions derived from the actual error paths in `PluginManager` and `PluginError`.

---

## Bundle Won't Load

### "Cannot create bundle from X"

`Bundle(url:)` returned `nil`. The `.tableplugin` directory structure is wrong.

**Fix:**
- Verify your bundle has the correct layout:
  ```
  MyPlugin.tableplugin/
    Contents/
      Info.plist
      MacOS/
        MyPlugin          ← compiled binary
  ```
- Check that `Info.plist` contains `NSPrincipalClass` pointing to your plugin class.
- The bundle must have a `.tableplugin` extension. `PluginManager` skips anything else.

### "Bundle failed to load executable"

`bundle.load()` returned `false`. The binary exists but macOS refused to load it.

**Fix:**
- **Architecture mismatch.** If you built arm64-only and the user runs on Intel (or vice versa), the load fails silently. Build as Universal Binary: `lipo -create arm64/MyPlugin x86_64/MyPlugin -output MyPlugin`.
- **Missing linked frameworks.** Your plugin must link against `TableProPluginKit.framework`, not embed it. If the framework isn't found at load time, `dlopen` fails.
- **Check Console.app** for `dyld` errors. Filter by your plugin name. Common causes: missing `@rpath`, unresolved symbols, wrong install name.
- Run `file MyPlugin.tableplugin/Contents/MacOS/MyPlugin` to confirm the binary contains both architectures.

### "Principal class does not conform to TableProPlugin"

`bundle.principalClass` loaded, but the cast to `TableProPlugin.Type` failed.

**Fix:**
- Your principal class must subclass `NSObject` AND conform to `TableProPlugin`.
- The `NSPrincipalClass` value in `Info.plist` must match the class name exactly. For Swift classes, use the unqualified name (no module prefix) if the class is `@objc`-compatible.
- If your class is pure Swift without `@objc`, you need the module-qualified name: `MyPluginModule.MyPluginClass`.

---

## Signature Verification Fails

Signature checks only apply to user-installed plugins (under `~/Library/Application Support/TablePro/Plugins/`). Built-in plugins bundled with the app skip this check.

### "bundle is not signed" (OSStatus -67062)

The plugin has no code signature at all.

**Fix:**
- Sign with a valid Apple Developer ID: `codesign --sign "Developer ID Application: Your Name (TEAMID)" --deep MyPlugin.tableplugin`
- Ad-hoc signatures (`codesign -s -`) are not accepted for user-installed plugins.

### "code signature is invalid" (OSStatus -67061)

A signature exists but doesn't validate.

**Fix:**
- Re-sign the bundle. This often happens when the binary was modified after signing (e.g., `install_name_tool` or `strip` ran post-signing).
- Verify with: `codesign -v --deep --strict MyPlugin.tableplugin`

### "code signature has been modified or corrupted" (OSStatus -67030)

Files in the bundle changed after signing.

**Fix:**
- Do all modifications (adding resources, fixing rpaths) *before* signing.
- Re-sign the entire bundle after any change.

### "signing certificate has expired" (OSStatus -67013)

**Fix:**
- Renew your Apple Developer certificate at developer.apple.com, download the new cert, and re-sign.

### "resource envelope has been modified" (OSStatus -67028)

A file in `Contents/Resources/` was added, removed, or changed after signing.

**Fix:**
- Finalize all resources before signing. Re-sign if you need to change anything.

### "code signature is missing required fields" (OSStatus -67058)

The signature exists but is incomplete.

**Fix:**
- Re-sign with `--deep` to ensure all nested code is signed.
- Check that your signing identity is a full Developer ID, not a self-signed cert.

### Team ID mismatch

The signature is valid, but the Team Identifier doesn't match what `PluginManager` expects. The app checks `certificate leaf[subject.OU]` against a configured team ID.

**Fix:**
- Check your plugin's team ID: `codesign -dvvv MyPlugin.tableplugin 2>&1 | grep TeamIdentifier`
- The team ID must match `PluginManager.signingTeamId`. Contact the TablePro team if you need your team ID allowlisted.

---

## Version Compatibility

### "Plugin requires PluginKit version X, but app provides version Y"

Your plugin's `TableProPluginKitVersion` in `Info.plist` is higher than `PluginManager.currentPluginKitVersion` (currently `1`).

**Fix:**
- Rebuild your plugin against the version of `TableProPluginKit` that ships with the target app version.
- If you set `TableProPluginKitVersion` manually, lower it to match. But only do this if your plugin genuinely doesn't use newer API.

### "Plugin requires app version X or later, but current version is Y"

Your `TableProMinAppVersion` in `Info.plist` is newer than the running app.

**Fix:**
- Update the app to the required version, or lower `TableProMinAppVersion` in your plugin's `Info.plist` if the dependency isn't real.
- Version comparison uses `.numeric` ordering, so `1.10.0` > `1.9.0`.

---

## Plugin Conflicts

### "A built-in plugin 'X' already provides this bundle ID"

Your plugin's `CFBundleIdentifier` collides with a built-in plugin. The app blocks user-installed plugins from overriding built-in ones.

**Fix:**
- Change your `CFBundleIdentifier` to something unique (e.g., `com.yourcompany.tablepro.myplugin`).
- You cannot replace built-in drivers (MySQL, PostgreSQL, SQLite, ClickHouse, MSSQL, MongoDB, Redis, Oracle) with user-installed plugins.

---

## Installation Issues

### "No .tableplugin bundle found in archive"

The ZIP file doesn't contain a `.tableplugin` bundle at the top level.

**Fix:**
- Package your plugin so the ZIP extracts to `MyPlugin.tableplugin/` directly, not nested inside another folder.
- The installer uses `ditto -xk` and looks for the first item with `.tableplugin` extension in the extracted directory.

### "Failed to extract archive (ditto exit code N)"

The ZIP file is corrupted or not a valid ZIP.

**Fix:**
- Re-create the archive. Use `ditto -ck --keepParent MyPlugin.tableplugin MyPlugin.zip` for best compatibility.

---

## Runtime Issues

### Plugin loads but database type doesn't appear

The plugin loaded successfully (check Console.app for "Loaded plugin" log), but the database type isn't available in the connection dialog.

**Fix:**
- Verify `databaseTypeId` on your `DriverPlugin` matches a `DatabaseType` case that the app knows about. For custom database types, the app must explicitly support the type in its enum.
- Check that your plugin declares `.databaseDriver` in its `capabilities` array.
- If your plugin serves multiple database types, implement `additionalDatabaseTypeIds` on your `DriverPlugin` conformance.
- Make sure the plugin isn't disabled. Check `UserDefaults` key `disabledPlugins` or the Plugins preference pane.

### Connection fails immediately after plugin loads

`createDriver(config:)` returns a driver, but `connect()` throws.

**Fix:**
- Check Console.app filtered to the "PluginDriverAdapter" category. The adapter logs connection errors.
- Verify your `PluginDatabaseDriver.connect()` implementation handles the connection config correctly (host, port, credentials, SSL settings).
- The `PluginDriverAdapter` sets status to `.error(message)` on failure. The error message propagates to the UI.

### Plugin is disabled and won't re-enable

**Fix:**
- The disable state is stored in `UserDefaults` under the `disabledPlugins` key as a string array of bundle IDs.
- To manually clear: `defaults delete com.TablePro disabledPlugins` (or remove your specific bundle ID from the array).

---

## SourceKit / Xcode Indexing Noise

### "No such module 'TableProPluginKit'"

This is an Xcode indexing issue, not a real build error. SourceKit sometimes can't resolve cross-target module imports.

**Fix:**
- Build with `xcodebuild` to confirm the project compiles.
- Clean derived data if it persists: `rm -rf ~/Library/Developer/Xcode/DerivedData/TablePro-*`
- The project uses `objectVersion 77` (PBXFileSystemSynchronizedRootGroup), which can confuse older Xcode indexing.

### "Cannot find type 'PluginQueryResult' in scope"

Same indexing noise. Types from `TableProPluginKit` (like `PluginQueryResult`, `PluginColumnInfo`, `PluginTableInfo`) sometimes aren't visible to SourceKit in plugin targets.

**Fix:**
- Build with `xcodebuild` to verify. If it builds, ignore the SourceKit errors.

---

## Testing

### Can't load plugin bundles in unit tests

Plugin bundles require the full app context: framework search paths, code signing, runtime bundle loading. The test runner doesn't provide this.

**Fix:**
- Don't call `PluginManager.loadAllPlugins()` in tests. Plugin bundles aren't available in the test runner.
- Use `StubDriver` mocks that implement `DatabaseDriver` protocol directly.
- To test plugin source code, add the plugin's Swift files to the test target (inline-copy pattern) so you can test logic without bundle loading.

### DatabaseDriverFactory.createDriver throws in tests

The factory throws when a plugin isn't loaded (it no longer calls `fatalError`).

**Fix:**
- Tests should not go through `DatabaseDriverFactory`. Use `StubDriver` mocks instead.
- If you must test factory behavior, mock the `PluginManager.driverPlugins` dictionary.

---

## Debugging Tips

- **Console.app**: Filter by subsystem `com.TablePro`. Two categories matter:
  - `PluginManager` - load, register, enable, disable events
  - `PluginDriverAdapter` - adapter-level errors during query execution and schema operations
- **Code signature inspection**: `codesign -dvvv MyPlugin.tableplugin 2>&1` shows signing identity, team ID, entitlements, and flags.
- **Binary architecture**: `file MyPlugin.tableplugin/Contents/MacOS/MyPlugin` shows which architectures the binary contains.
- **dyld debugging**: Set `DYLD_PRINT_LIBRARIES=1` in Xcode scheme environment variables to see all library loads at launch.
- **Plugin directories**:
  - Built-in: `TablePro.app/Contents/PlugIns/`
  - User-installed: `~/Library/Application Support/TablePro/Plugins/`
