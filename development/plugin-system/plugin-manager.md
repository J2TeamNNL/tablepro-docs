# PluginManager

`PluginManager` is a `@MainActor @Observable` singleton that handles plugin discovery, loading, registration, and lifecycle management.

**File**: `TablePro/Core/Plugins/PluginManager.swift`

## Loading Flow

Called once at app startup via `PluginManager.shared.loadAllPlugins()`.

```mermaid
graph TD
    A[loadAllPlugins] --> B[Ensure ~/Library/.../Plugins/ exists]
    B --> C[Scan Contents/PlugIns/ - built-in]
    B --> D[Scan ~/Library/.../Plugins/ - user-installed]
    C --> E[For each .tableplugin]
    D --> E
    E --> F{Bundle valid?}
    F -->|No| G[Log error, skip]
    F -->|Yes| H{PluginKit version OK?}
    H -->|Too new| G
    H -->|OK| I{Min app version OK?}
    I -->|Too old| G
    I -->|OK| J{User-installed?}
    J -->|Yes| K{Code signature valid?}
    K -->|No| G
    K -->|Yes| L[Load bundle]
    J -->|No| L
    L --> M[Cast NSPrincipalClass to TableProPlugin]
    M --> N[Create PluginEntry]
    N --> O{Plugin enabled?}
    O -->|Yes| P[Instantiate + register capabilities]
    O -->|No| Q[Store entry only]
```

### Step-by-step

1. **Directory setup**: Creates `~/Library/Application Support/TablePro/Plugins/` if missing.
2. **Scan**: Lists all `.tableplugin` items in both the built-in and user directories.
3. **Bundle creation**: `Bundle(url:)` -- fails if the bundle structure is invalid.
4. **Version check**: Reads `TableProPluginKitVersion` from Info.plist. Rejects if higher than `PluginManager.currentPluginKitVersion` (currently `1`). Throws `incompatibleVersion(required:current:)`.
5. **App version check**: If `TableProMinAppVersion` is set, compares against the running app version using `.numeric` string comparison. Throws `appVersionTooOld(minimumRequired:currentApp:)` with the actual version strings if the app is too old.
6. **Code signature** (user-installed only): Calls `verifyCodeSignature(bundle:)` which throws `PluginError.signatureInvalid(detail:)` on failure. See [Code Signature Verification](#code-signature-verification).
7. **Load executable**: `bundle.load()` -- loads the Mach-O binary into the process.
8. **Principal class**: Casts `bundle.principalClass` to `TableProPlugin.Type`.
9. **Entry creation**: Stores a `PluginEntry` with metadata (name, version, source, capabilities, enabled state).
10. **Registration**: If enabled, instantiates the class and calls `registerCapabilities()`.

The `loadPlugin(at:source:)` method is `@discardableResult` and returns the `PluginEntry` directly.

## Registration

### Capability-Gated Registration

Registration checks the plugin's declared `capabilities` array before registering for each protocol:

```swift
let declared = Set(type(of: instance).capabilities)

if let driver = instance as? any DriverPlugin {
    if !declared.contains(.databaseDriver) {
        // Log warning, but register anyway (lenient mode)
    }
    let typeId = type(of: driver).databaseTypeId
    driverPlugins[typeId] = driver
    for additionalId in type(of: driver).additionalDatabaseTypeIds {
        driverPlugins[additionalId] = driver
    }
}
```

The `driverPlugins` dictionary maps type ID strings to `DriverPlugin` instances. It is a regular `@MainActor`-isolated property on `PluginManager`. `DatabaseDriverFactory` is also `@MainActor`, ensuring safe access to `driverPlugins` without data races.

### Capability Validation

`validateCapabilityDeclarations()` runs at load time for each plugin and checks both directions:

- Plugin declares `.databaseDriver` but doesn't conform to `DriverPlugin` -- logs warning
- Plugin declares `.exportFormat` but doesn't conform to `ExportFormatPlugin` -- logs warning

This catches configuration mistakes early. Registration is lenient (proceeds with a warning) to avoid breaking third-party plugins that forget to declare capabilities.

### Unregistration

When a plugin is disabled, `unregisterCapabilities(pluginId:)` removes **all** entries from `driverPlugins` that were registered by that plugin. This includes both the primary type ID and any additional type IDs:

```swift
private func unregisterCapabilities(pluginId: String) {
    driverPlugins = driverPlugins.filter { _, value in
        guard let entry = plugins.first(where: { $0.id == pluginId }) else { return true }
        if let principalClass = entry.bundle.principalClass as? any DriverPlugin.Type {
            let allTypeIds = Set([principalClass.databaseTypeId] + principalClass.additionalDatabaseTypeIds)
            return !allTypeIds.contains(type(of: value).databaseTypeId)
        }
        return true
    }
}
```

The `Set` ensures both the primary ID and all `additionalDatabaseTypeIds` are removed in one pass. For example, disabling the MySQL plugin removes both `"MySQL"` and `"MariaDB"` entries.

## Enable / Disable

```swift
func setEnabled(_ enabled: Bool, pluginId: String)
```

- Updates the `PluginEntry.isEnabled` flag.
- Persists the disabled set to `UserDefaults` under the key `"disabledPlugins"`.
- If enabling: instantiates the plugin class and registers capabilities.
- If disabling: unregisters capabilities (removes from `driverPlugins`).
- Posts `Notification.Name.pluginStateDidChange` with `userInfo: ["pluginId": pluginId]`.

Disabled plugins remain in the `plugins` array and their bundles stay loaded in memory. They just have no registered capabilities.

## Install / Uninstall

### installPlugin(from:)

```swift
func installPlugin(from url: URL) async throws -> PluginEntry
```

Entry point for all plugin installation. Guards against concurrent installs via the `isInstalling` property (rejects with `installFailed` if already in progress). Routes by file extension:

- `.tableplugin` → `installBundle(from:)` (synchronous)
- `.zip` → `installFromZip(from:)` (async, extracts via `ditto`)

Called from three places:
- **File picker** (`InstalledPluginsView.installFromFile`) — accepts both `.zip` and `.tableplugin`
- **Drag-and-drop** (`InstalledPluginsView.onDrop`) — validates extension before calling
- **Finder double-click** (`AppDelegate.handlePluginInstall`) — handles `.tableplugin` files opened via macOS file association

#### installBundle(from:)

Direct `.tableplugin` bundle installation:

1. `Bundle(url:)` to validate structure.
2. `verifyCodeSignature(bundle:)`.
3. Checks for built-in conflict (same bundle ID → `pluginConflict`).
4. `replaceExistingPlugin(bundleId:)` to cleanly remove any existing version.
5. Copies to `userPluginsDir` (skips copy if source and destination are the same path via `standardizedFileURL` comparison).
6. `loadPlugin(at:source: .userInstalled)`.

#### installFromZip(from:)

Archive-based installation:

1. Extracts `.zip` to a temp directory using `/usr/bin/ditto` (async via `withCheckedThrowingContinuation`).
2. Finds the first `.tableplugin` bundle in extracted contents.
3. `verifyCodeSignature(bundle:)`.
4. Checks for built-in conflict.
5. `replaceExistingPlugin(bundleId:)` to cleanly remove any existing version.
6. Copies to `userPluginsDir`.
7. `loadPlugin(at:source: .userInstalled)`.

If a plugin with the same filename already exists in the user plugins directory, it is replaced.

#### replaceExistingPlugin(bundleId:)

Shared helper called by both install paths. If a plugin with the given bundle ID is already loaded:

1. `unregisterCapabilities(pluginId:)` — must run first since it reads from `plugins` to find the principal class.
2. `bundle.unload()`.
3. Removes entry from `plugins` array.

#### isInstalling

`private(set) var isInstalling = false` — observable property set by `installPlugin(from:)`. The UI observes this to disable the install button and show a spinner. Also prevents concurrent installs from different entry points (file picker, drag-and-drop, Finder).

### uninstallPlugin(id:)

```swift
func uninstallPlugin(id: String) throws
```

1. Finds the plugin entry by ID.
2. Rejects if the plugin is built-in (`PluginError.cannotUninstallBuiltIn`).
3. Unregisters capabilities.
4. Calls `bundle.unload()`.
5. Removes the entry from the `plugins` array.
6. Deletes the `.tableplugin` directory from disk.
7. Removes from the disabled plugins set.
8. Sets `needsRestart = true` (the UI shows a restart recommendation banner).

## Code Signature Verification

User-installed plugins must pass macOS code signature validation pinned to the app's team ID:

```swift
private func verifyCodeSignature(bundle: Bundle) throws
```

Uses Security.framework:

1. `SecStaticCodeCreateWithPath` to get a `SecStaticCode` reference. Throws `signatureInvalid(detail:)` if this fails.
2. `createSigningRequirement()` builds a `SecRequirement` pinned to the team ID: `anchor apple generic and certificate leaf[subject.OU] = "D7HJ5TFYCU"`.
3. `SecStaticCodeCheckValidity` with `kSecCSCheckAllArchitectures` and the team ID requirement. Throws `signatureInvalid(detail:)` if validation fails.

The `signingTeamId` static property holds the TablePro team identifier (`D7HJ5TFYCU`).

Error details come from `describeOSStatus(_:)`, which maps common Security.framework codes to readable strings:

| OSStatus | Description |
|----------|-------------|
| -67062 | bundle is not signed |
| -67061 | code signature is invalid |
| -67030 | code signature has been modified or corrupted |
| -67013 | signing certificate has expired |
| -67058 | code signature is missing required fields |
| -67028 | resource envelope has been modified |
| other | verification failed (OSStatus N) |

Built-in plugins skip this check because they are covered by the app's own code signature.

## Data Model

### PluginEntry

```swift
struct PluginEntry: Identifiable {
    let id: String              // Bundle identifier or filename
    let bundle: Bundle
    let url: URL
    let source: PluginSource    // .builtIn or .userInstalled
    let name: String
    let version: String
    let pluginDescription: String
    let capabilities: [PluginCapability]
    var isEnabled: Bool
}
```

### PluginSource

```swift
enum PluginSource {
    case builtIn
    case userInstalled
}
```

### PluginError

| Case | When |
|------|------|
| `invalidBundle(String)` | Bundle cannot be created or loaded |
| `signatureInvalid(detail: String)` | Code signature check failed, with human-readable OSStatus description |
| `checksumMismatch` | Future: content hash verification |
| `incompatibleVersion(required:current:)` | PluginKit version too new for the running app |
| `appVersionTooOld(minimumRequired:currentApp:)` | App version is older than the plugin's `TableProMinAppVersion` |
| `pluginConflict(existingName: String)` | User-installed plugin has the same bundle ID as a built-in plugin |
| `cannotUninstallBuiltIn` | Tried to uninstall a built-in plugin |
| `notFound` | Plugin ID not in registry |
| `noCompatibleBinary` | Future: universal binary check |
| `installFailed(String)` | Archive extraction or copy failed |

## PluginDriverAdapter

**File**: `TablePro/Core/Plugins/PluginDriverAdapter.swift`

Bridges `PluginDatabaseDriver` (plugin side) to `DatabaseDriver` (app side). This is where transfer types are mapped:

- `PluginQueryResult` -> `QueryResult` (includes column type mapping from raw type name strings to `ColumnType` enum)
- `PluginColumnInfo` -> `ColumnInfo`
- `PluginIndexInfo` -> `IndexInfo`
- `PluginForeignKeyInfo` -> `ForeignKeyInfo`
- `PluginTableInfo` -> `TableInfo`
- `PluginTableMetadata` -> `TableMetadata`
- `PluginDatabaseMetadata` -> `DatabaseMetadata`

The adapter also conforms to `SchemaSwitchable` and delegates `switchSchema(to:)` / `switchDatabase(to:)` to the plugin driver.

### Column Type Mapping

`PluginDriverAdapter.mapColumnType(rawTypeName:)` converts raw type name strings from plugins into the app's `ColumnType` enum. The mapping handles common SQL type names:

- `BOOL*` -> `.boolean`
- `INT`, `INTEGER`, `BIGINT`, etc. -> `.integer`
- `FLOAT`, `DOUBLE`, `DECIMAL`, etc. -> `.decimal`
- `DATE` -> `.date`
- `TIMESTAMP*` -> `.timestamp`
- `JSON`, `JSONB` -> `.json`
- `BLOB`, `BYTEA`, `BINARY` -> `.blob`
- `ENUM*` -> `.enumType`
- `GEOMETRY`, `POINT`, etc. -> `.spatial`
- Everything else -> `.text`
