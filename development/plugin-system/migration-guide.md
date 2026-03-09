# PluginKit Migration Guide

This guide covers how to handle breaking changes when TableProPluginKit versions bump. If you maintain a third-party plugin, read this whenever you upgrade to a new TablePro release.

## Versioning Policy

TableProPluginKit uses a single integer version (`TableProPluginKitVersion`) declared in each plugin's `Info.plist`. This version tracks binary-incompatible changes to protocols and transfer types.

**Current version: `1`**

Rules:

- **Additive changes do not bump the version.** New methods on `PluginDatabaseDriver` with default implementations, new optional fields on transfer types with default init values - these are backwards-compatible. Your plugin keeps working without changes.
- **Breaking changes bump the version.** Removing methods, changing method signatures, renaming protocol requirements, changing transfer type field types or removing fields - these require a version bump.
- **Plugins declaring a version higher than the app supports are rejected at load time.** If your plugin has `TableProPluginKitVersion = 3` but the app only supports up to `2`, it won't load. The app logs: `incompatibleVersion(required: 3, current: 2)`.
- **The version only goes up.** There are no minor versions or patch levels. Each bump means "review the migration steps below."

## When to Update Your Plugin

Existing plugins continue to work after a PluginKit version bump until the old version is deprecated and removed (which will be announced at least one major release in advance).

Here's what to check on each TablePro release:

1. Read the release notes for any PluginKit version changes.
2. If the version bumped, find the corresponding section below for step-by-step migration.
3. Update `TableProPluginKitVersion` in your `Info.plist` to the new version.
4. Rebuild against the new `TableProPluginKit.framework`.
5. Test with the target TablePro version.

### Info.plist Keys

| Key | Type | Description |
|-----|------|-------------|
| `TableProPluginKitVersion` | Integer | Which PluginKit protocol version this plugin targets. Must be <= the app's `PluginManager.currentPluginKitVersion`. |
| `TableProMinAppVersion` | String | Minimum TablePro app version required (e.g., `"0.15.0"`). Optional. If set, the app rejects the plugin when running an older version. |

## Version 1 (Current - Baseline)

This is the initial PluginKit release. No migration needed.

### Protocols

**`TableProPlugin`** - base protocol for all plugins:

```swift
public protocol TableProPlugin: AnyObject {
    static var pluginName: String { get }
    static var pluginVersion: String { get }
    static var pluginDescription: String { get }
    static var capabilities: [PluginCapability] { get }
    init()
}
```

**`DriverPlugin`** - entry point for database driver plugins:

```swift
public protocol DriverPlugin: TableProPlugin {
    static var databaseTypeId: String { get }         // Required
    static var databaseDisplayName: String { get }    // Required
    static var iconName: String { get }               // Required
    static var defaultPort: Int { get }               // Required
    static var additionalConnectionFields: [ConnectionField] { get }  // Default: []
    static var additionalDatabaseTypeIds: [String] { get }            // Default: []
    func createDriver(config: DriverConnectionConfig) -> any PluginDatabaseDriver
}
```

**`PluginDatabaseDriver`** - the driver implementation protocol. Required methods (no default):

- `connect() async throws`
- `disconnect()`
- `execute(query:) async throws -> PluginQueryResult`
- `fetchTables(schema:) async throws -> [PluginTableInfo]`
- `fetchColumns(table:schema:) async throws -> [PluginColumnInfo]`
- `fetchIndexes(table:schema:) async throws -> [PluginIndexInfo]`
- `fetchForeignKeys(table:schema:) async throws -> [PluginForeignKeyInfo]`
- `fetchTableDDL(table:schema:) async throws -> String`
- `fetchViewDefinition(view:schema:) async throws -> String`
- `fetchTableMetadata(table:schema:) async throws -> PluginTableMetadata`
- `fetchDatabases() async throws -> [String]`
- `fetchDatabaseMetadata(_:) async throws -> PluginDatabaseMetadata`

All other methods have default implementations. See `PluginDatabaseDriver.swift` for the full list and defaults.

### Transfer Types

All transfer types are `Codable` and `Sendable`:

| Type | Fields |
|------|--------|
| `PluginQueryResult` | `columns: [String]`, `columnTypeNames: [String]`, `rows: [[String?]]`, `rowsAffected: Int`, `executionTime: TimeInterval` |
| `PluginColumnInfo` | `name`, `dataType`, `isNullable`, `isPrimaryKey`, `defaultValue?`, `extra?`, `charset?`, `collation?`, `comment?` |
| `PluginTableInfo` | `name`, `type`, `rowCount?` |
| `PluginIndexInfo` | See source |
| `PluginForeignKeyInfo` | See source |
| `PluginTableMetadata` | See source |
| `PluginDatabaseMetadata` | See source |
| `DriverConnectionConfig` | `host`, `port`, `username`, `password`, `database`, `additionalFields: [String: String]` |
| `ConnectionField` | `id`, `label`, `placeholder`, `isRequired`, `isSecure`, `defaultValue?` |
| `PluginCapability` | Enum: `.databaseDriver`, `.exportFormat`, `.importFormat`, `.sqlDialect`, `.aiProvider`, `.cellRenderer`, `.sidebarPanel` |

## Migration Template - Version N to N+1

Future version bumps will add a section here following this format:

```
## Version N to Version N+1

Released in TablePro vX.Y.Z.

### Breaking Changes

- `methodX(old:)` renamed to `methodX(new:)`
- `TransferTypeY.fieldZ` type changed from String to Int
- `removedMethod()` removed (use `replacementMethod()` instead)

### New Required Methods (no default)

- `newMethod()` - what it does, how to implement it

### New Optional Methods (have defaults)

- `optionalMethod()` - default behavior, when you'd want to override

### Transfer Type Changes

- `PluginQueryResult` added field `newField: Type` (default value: X)
- `PluginColumnInfo.oldField` renamed to `newFieldName`

### Migration Steps

1. Update `TableProPluginKitVersion` to `N+1` in Info.plist
2. Rename `methodX(old:)` to `methodX(new:)`
3. Update `TransferTypeY.fieldZ` from String to Int
4. Implement `newMethod()`
5. Rebuild and test

### Before / After

// Before (version N)
func methodX(old: String) async throws -> Result { ... }

// After (version N+1)
func methodX(new: String) async throws -> Result { ... }
```

## Compatibility Matrix

| PluginKit Version | Minimum App Version | Status |
|-------------------|---------------------|--------|
| 1 | 0.15.0 | Current |

This table will be updated with each version bump.

## Best Practices for Forward Compatibility

- **Only import TableProPluginKit.** Never import the main `TablePro` app target or reference its internal types. The plugin boundary is the PluginKit framework.
- **Implement all protocol methods explicitly.** Don't rely on default implementations staying the same across versions. If a default changes behavior, your plugin won't notice unless you override it.
- **Keep `TableProMinAppVersion` as low as possible.** This maximizes the range of app versions your plugin works with. Only bump it when you actually need a feature from a newer app version.
- **Don't depend on undocumented behavior.** If a default implementation uses `SELECT COUNT(*) FROM (query) _t` for `fetchRowCount`, don't assume that exact SQL. Implement your own if the default doesn't work for your database.
- **Test against both the minimum and latest app versions.** The minimum ensures backwards compatibility; the latest catches any deprecation warnings.
- **All driver classes must be `Sendable`.** `PluginDatabaseDriver` requires `AnyObject & Sendable`. Use actors or proper synchronization for mutable state.
- **Return `PluginQueryResult.empty` for no-op results.** Don't construct zero-valued results manually when there's a static `.empty` available.
