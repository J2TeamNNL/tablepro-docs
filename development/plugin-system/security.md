# Plugin Security Model

TablePro plugins are native macOS bundles (`.tableplugin`) loaded into the app process at runtime. This means plugins run with the same privileges as the app itself. This document describes the security mechanisms in place, what they protect against, and what they don't.

## Code Signing

### Built-in plugins

Plugins shipped inside the app bundle (in `Contents/PlugIns/`) are covered by the app's own code signature. macOS validates the app signature at launch, which transitively covers everything inside the bundle. No separate signature check is performed by `PluginManager`.

### User-installed plugins

Plugins installed to `~/Library/Application Support/TablePro/Plugins/` go through explicit code signature verification before loading. The flow:

1. `SecStaticCodeCreateWithPath` creates a `SecStaticCode` reference from the plugin bundle URL.
2. `createSigningRequirement()` builds a `SecRequirement` string:
   ```
   anchor apple generic and certificate leaf[subject.OU] = "TEAMID"
   ```
3. `SecStaticCodeCheckValidity` validates the bundle against this requirement, using the `kSecCSCheckAllArchitectures` flag to verify all architecture slices (arm64 + x86_64).

If verification fails, the plugin is rejected with a `PluginError.signatureInvalid` error. The user never gets the option to override this.

### Team ID pinning

The `signingTeamId` static property on `PluginManager` determines which Apple Developer Team ID is accepted. The requirement string pins to the leaf certificate's Organizational Unit (`subject.OU`), meaning only plugins signed by that specific team are accepted.

**Current status**: `signingTeamId` is set to the placeholder `"YOURTEAMID"`. This must be replaced with a real team identifier before user-installed plugin support ships.

### OSStatus error codes

When signature verification fails, `describeOSStatus()` maps Security framework codes to human-readable messages:

| OSStatus | Meaning |
|----------|---------|
| -67062 | Bundle is not signed |
| -67061 | Code signature is invalid |
| -67030 | Code signature has been modified or corrupted |
| -67013 | Signing certificate has expired |
| -67058 | Code signature is missing required fields |
| -67028 | Resource envelope has been modified |

Any other status code falls through to a generic "verification failed (OSStatus N)" message.

## Trust Levels

Plugins fall into four trust tiers based on how they were distributed:

| Level | Source | Signature check | What it means |
|-------|--------|----------------|---------------|
| **Built-in** | Shipped inside app bundle | App signature covers it | First-party code, maintained by the TablePro team |
| **Verified** | Downloaded from official marketplace | Team ID pinned signature check | Third-party code reviewed and signed by the TablePro team |
| **Community** | Downloaded from marketplace, signed by author | Author's Developer ID check | Third-party code signed by its developer, not reviewed by TablePro |
| **Sideloaded** | Manually placed in plugins directory | Team ID pinned signature check | Must still pass signature verification to load |

Built-in plugins cannot be uninstalled (`PluginError.cannotUninstallBuiltIn`). User-installed plugins can be enabled, disabled, or removed at any time.

## Threat Model

### What a malicious plugin CAN do

Plugins are native Mach-O bundles loaded via `NSBundle.load()` into the app's address space. Once loaded, a plugin has:

- **Full process access**: arbitrary Swift/ObjC code execution in the same process. A plugin can call any framework, swizzle methods, read process memory.
- **File system access**: read and write any file the app can access.
- **Network access**: open arbitrary network connections, send data anywhere.
- **Keychain access**: read Keychain items available to the app (connection passwords are stored in Keychain via `ConnectionStorage`).
- **Connection credentials**: plugins receive `DriverConnectionConfig` with plaintext host, port, username, password, and database name.

### What mitigations exist today

- **Code signature verification**: user-installed plugins must be signed with a specific Apple Developer ID. An unsigned or tampered bundle is rejected before `NSBundle.load()` is called.
- **Team ID pinning**: only plugins signed by the pinned team ID are accepted. A valid Apple Developer ID from a different team is rejected.
- **All-architectures check**: `kSecCSCheckAllArchitectures` prevents attacks that target only one architecture slice.
- **Conflict detection**: a user-installed plugin cannot shadow a built-in plugin's bundle ID (`PluginError.pluginConflict`).
- **User must explicitly install**: plugins don't auto-download. The user initiates installation from a `.zip` archive.
- **Version gating**: `TableProPluginKitVersion` and `TableProMinAppVersion` in Info.plist prevent loading plugins built against incompatible SDK versions.

### What mitigations are planned

- **Marketplace review process**: reviewing plugin code and behavior before listing in an official marketplace.
- **Runtime sandboxing**: restricting plugin capabilities using macOS sandbox profiles or XPC (future work).
- **Capability declarations**: enforcing that a plugin declaring only `.databaseDriver` cannot access export or AI APIs.

### What the system does NOT protect against

- A validly-signed plugin from a **compromised developer account**. If an attacker obtains the signing key, they can produce plugins that pass verification.
- **Supply chain attacks** on plugin dependencies. A plugin linking against a compromised C library will pass signature checks if the final bundle is signed.
- **Runtime misbehavior** by a signed plugin. Once loaded, there is no monitoring of what the plugin code actually does.
- A plugin that **exfiltrates connection credentials** it legitimately receives via `DriverConnectionConfig`.

## Plugin Capabilities and Access

The `PluginCapability` enum declares what a plugin intends to provide:

```swift
public enum PluginCapability: Int, Codable, Sendable {
    case databaseDriver
    case exportFormat
    case importFormat
    case sqlDialect
    case aiProvider
    case cellRenderer
    case sidebarPanel
}
```

These are currently **declarations only**, not enforced restrictions. A plugin declaring `.databaseDriver` has the same runtime access as one declaring `.aiProvider`. There is no sandbox boundary between capability types.

Database driver plugins receive connection credentials via `DriverConnectionConfig`:

```swift
public struct DriverConnectionConfig: Sendable {
    public let host: String
    public let port: Int
    public let username: String
    public let password: String
    public let database: String
    public let additionalFields: [String: String]
}
```

The password is passed in plaintext. This is necessary for the plugin to establish a database connection, but it means any loaded plugin has access to the credentials.

## Bundle Integrity

Several checks run before a plugin's executable code is loaded:

1. **NSBundle creation**: `Bundle(url:)` validates basic bundle structure (correct directory layout, Info.plist present).
2. **PluginKit version check**: `TableProPluginKitVersion` in Info.plist must be less than or equal to `PluginManager.currentPluginKitVersion`. A plugin built against a newer SDK is rejected with `PluginError.incompatibleVersion`.
3. **App version check**: if `TableProMinAppVersion` is set in Info.plist, the current app version is compared. If the app is older, loading fails with `PluginError.appVersionTooOld`.
4. **Code signature verification** (user-installed only): as described above.
5. **Principal class check**: `bundle.principalClass` must conform to `TableProPlugin`. If not, the plugin is rejected with `PluginError.invalidBundle`.
6. **Architecture verification**: `kSecCSCheckAllArchitectures` ensures all Mach-O slices in a Universal Binary are validly signed.

During installation (via `installPlugin(from:)`), the signature is verified on the extracted bundle *before* copying it to the user plugins directory. A plugin that fails verification is never persisted.

## Recommendations for Plugin Developers

- **Always code-sign** with a valid Apple Developer ID certificate. Unsigned plugins will be rejected.
- **Build as Universal Binary** (arm64 + x86_64) to work on both Apple Silicon and Intel Macs. The `kSecCSCheckAllArchitectures` flag validates all slices.
- **Don't store secrets in the plugin bundle**. Anything inside the `.tableplugin` directory is readable by anyone with file access.
- **Use HTTPS for all network connections**. Database protocols that don't support TLS should document this clearly.
- **Minimize dependencies** to reduce attack surface. Every linked library is a potential vulnerability.
- **Set `TableProPluginKitVersion` and `TableProMinAppVersion`** in your Info.plist to prevent your plugin from loading in incompatible environments.
- **Use a unique bundle identifier**. Your plugin cannot share a bundle ID with a built-in plugin.

## Known Limitations

- **`signingTeamId` is a placeholder**: set to `"YOURTEAMID"`. Must be replaced with the actual Apple Developer Team ID before sideloaded plugin support ships. Until then, no user-installed plugins will pass verification.
- **`Bundle.unload()` is unreliable on macOS**: when a user-installed plugin is uninstalled, `PluginManager` calls `bundle.unload()`, but Apple's documentation notes this may not actually unload the code. The plugin's executable code may remain mapped in memory until the app restarts.
- **No runtime sandboxing**: plugins run in the same process with the same entitlements as the app. There is no XPC boundary, no sandbox profile, no capability enforcement.
- **No capability restrictions**: the `PluginCapability` enum is advisory. A plugin declaring only `.databaseDriver` has the same runtime access as the host app.
- **Credentials in plaintext**: `DriverConnectionConfig` passes database passwords as plain `String` values. There is no way for the app to restrict which plugins see which credentials.
