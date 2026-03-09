# Developer Guide: Building a Plugin

This guide walks through creating a new database driver plugin. The SQLite plugin is the simplest reference implementation.

## Prerequisites

- Xcode (same version used to build TablePro)
- Access to the `TableProPluginKit` framework
- A code signing identity (required for user-installed plugins)

## 1. Create the Bundle Target

In Xcode, add a new target:

1. File > New > Target > macOS > Bundle
2. Set product name (e.g., `MyDBDriverPlugin`)
3. Set bundle extension to `tableplugin`
4. Link `TableProPluginKit.framework`

### Info.plist

Set these keys:

```xml
<key>NSPrincipalClass</key>
<string>MyDBPlugin</string>

<key>TableProPluginKitVersion</key>
<integer>1</integer>

<key>CFBundleIdentifier</key>
<string>com.example.mydb-driver</string>
```

Optionally set `TableProMinAppVersion` if your plugin uses APIs added in a specific app version.

## 2. Implement the Plugin Entry Point

Create the principal class. It must:
- Subclass `NSObject` (required for `NSPrincipalClass` loading)
- Conform to `TableProPlugin` and `DriverPlugin`
- Have a `required init()` (inherited from `NSObject`)

```swift
import Foundation
import TableProPluginKit

final class MyDBPlugin: NSObject, TableProPlugin, DriverPlugin {
    static let pluginName = "MyDB Driver"
    static let pluginVersion = "1.0.0"
    static let pluginDescription = "MyDB database support"
    static let capabilities: [PluginCapability] = [.databaseDriver]

    static let databaseTypeId = "MyDB"
    static let databaseDisplayName = "MyDB"
    static let iconName = "cylinder.fill"    // SF Symbol name
    static let defaultPort = 5555

    func createDriver(config: DriverConnectionConfig) -> any PluginDatabaseDriver {
        MyDBPluginDriver(config: config)
    }
}
```

### Optional: Additional Connection Fields

If your database needs extra connection parameters beyond host/port/user/pass/database:

```swift
static let additionalConnectionFields: [ConnectionField] = [
    ConnectionField(
        id: "myOption",
        label: "Custom Option",
        placeholder: "value",
        required: false,
        secure: false,
        defaultValue: "default"
    )
]
```

These values arrive in `config.additionalFields["myOption"]`.

### Optional: Multi-Type Support

If one driver handles multiple database types (e.g., MySQL also handles MariaDB):

```swift
static let additionalDatabaseTypeIds: [String] = ["MyDB-Variant"]
```

The plugin is registered under both `"MyDB"` and `"MyDB-Variant"`.

## 3. Implement PluginDatabaseDriver

This is the core of the plugin. Create a class conforming to `PluginDatabaseDriver`.

### Minimum Required Methods

```swift
final class MyDBPluginDriver: PluginDatabaseDriver, @unchecked Sendable {
    private let config: DriverConnectionConfig

    init(config: DriverConnectionConfig) {
        self.config = config
    }

    // -- Connection --

    func connect() async throws {
        // Open connection using config.host, config.port, etc.
    }

    func disconnect() {
        // Close connection
    }

    // -- Queries --

    func execute(query: String) async throws -> PluginQueryResult {
        // Run query, return results
        // All cell values must be stringified (String? per cell)
    }

    // -- Schema --

    func fetchTables(schema: String?) async throws -> [PluginTableInfo] { ... }
    func fetchColumns(table: String, schema: String?) async throws -> [PluginColumnInfo] { ... }
    func fetchIndexes(table: String, schema: String?) async throws -> [PluginIndexInfo] { ... }
    func fetchForeignKeys(table: String, schema: String?) async throws -> [PluginForeignKeyInfo] { ... }
    func fetchTableDDL(table: String, schema: String?) async throws -> String { ... }
    func fetchViewDefinition(view: String, schema: String?) async throws -> String { ... }
    func fetchTableMetadata(table: String, schema: String?) async throws -> PluginTableMetadata { ... }

    // -- Databases --

    func fetchDatabases() async throws -> [String] { ... }
    func fetchDatabaseMetadata(_ database: String) async throws -> PluginDatabaseMetadata { ... }
}
```

### Methods with Default Implementations

These have working defaults in the protocol extension. Override only if your database needs different behavior:

| Method | Default Behavior |
|--------|-----------------|
| `ping()` | Runs `SELECT 1` |
| `fetchRowCount(query:)` | Wraps query in `SELECT COUNT(*) FROM (...)` |
| `fetchRows(query:offset:limit:)` | Appends `LIMIT N OFFSET M` |
| `executeParameterized(query:parameters:)` | String-replaces `?` placeholders |
| `beginTransaction()` | Runs `BEGIN` |
| `commitTransaction()` | Runs `COMMIT` |
| `rollbackTransaction()` | Runs `ROLLBACK` |
| `switchDatabase(to:)` | Throws "unsupported" error |
| `cancelQuery()` | No-op |
| `applyQueryTimeout(_:)` | No-op |
| `fetchAllColumns(schema:)` | Iterates `fetchTables` + `fetchColumns` per table |
| `fetchAllForeignKeys(schema:)` | Iterates `fetchTables` + `fetchForeignKeys` per table |
| `fetchAllDatabaseMetadata()` | Iterates `fetchDatabases` + `fetchDatabaseMetadata` per db |

**`switchDatabase(to:)` note**: The default implementation throws an "unsupported" error. Drivers that support database switching must override it with their own logic. For reference:
- MySQL overrides with backtick-escaped `USE \`name\`` syntax.
- MSSQL overrides using the native FreeTDS API (not a SQL statement).
- ClickHouse has its own override that reconnects with the new database in the URL.

### Concurrency

`PluginDatabaseDriver` requires `Sendable` conformance. Common patterns:

- **Actor isolation**: Use a private actor to wrap the native connection handle (see SQLite plugin's `SQLiteConnectionActor`).
- **`@unchecked Sendable`**: If you manage thread safety manually with locks, mark the class `@unchecked Sendable`.
- **NSLock for interrupt handles**: For `cancelQuery()`, store the connection handle behind an `NSLock` so it can be accessed from any thread.

## 4. Column Type Names

Return raw type name strings in `PluginQueryResult.columnTypeNames`. The app maps these to its internal `ColumnType` enum. Recognized names include:

`BOOL`, `INT`, `INTEGER`, `BIGINT`, `SMALLINT`, `TINYINT`, `FLOAT`, `DOUBLE`, `DECIMAL`, `NUMERIC`, `REAL`, `DATE`, `DATETIME`, `TIMESTAMP`, `TIME`, `JSON`, `JSONB`, `BLOB`, `BYTEA`, `BINARY`, `GEOMETRY`, `POINT`, `LINESTRING`, `POLYGON`, `ENUM`, `SET`.

Unrecognized type names map to `.text`, which is a safe fallback.

## 5. Build and Test

### Build the Plugin

The plugin target produces a `.tableplugin` bundle. Ensure:

- It builds as a Universal Binary (arm64 + x86_64) for distribution.
- The `TableProPluginKit` framework is linked (not embedded -- it ships with the app).

### Testing

For unit tests, use the inline-copy pattern: copy the plugin's source files into the test target rather than loading the bundle dynamically. This avoids bundle-loading complexity in test runs.

```swift
// In your test target:
// 1. Add MyDBPluginDriver.swift to the test target's Compile Sources
// 2. Test the driver directly

func testConnect() async throws {
    let config = DriverConnectionConfig(
        host: "localhost",
        port: 5555,
        username: "test",
        password: "test",
        database: "testdb"
    )
    let driver = MyDBPluginDriver(config: config)
    try await driver.connect()
    // assertions...
    driver.disconnect()
}
```

Note: `DatabaseDriverFactory.createDriver` now throws (rather than calling `fatalError`) when a plugin is not found. For tests that need a `DatabaseDriver` without loading real plugin bundles, use a `StubDriver` mock that conforms to `DatabaseDriver` directly. This avoids the need to have `.tableplugin` bundles available in the test environment.

### Manual Testing

1. Build the plugin target.
2. Copy the `.tableplugin` bundle to `~/Library/Application Support/TablePro/Plugins/`.
3. Launch TablePro. Check the log for `"Loaded plugin 'MyDB Driver'"`.
4. Create a connection using your database type.
5. Alternatively, install via Settings > Plugins: click "Install from File...", select a `.zip` containing your `.tableplugin` bundle, and verify it appears in the plugin list.

For built-in plugin development, the plugin target is embedded in the app bundle automatically via Xcode's "Embed Without Signing" build phase.

## Reference: SQLite Plugin Structure

The SQLite plugin (`Plugins/SQLiteDriverPlugin/`) is the simplest driver and a good starting point:

```
SQLiteDriverPlugin/
  SQLitePlugin.swift     # Entry point + driver implementation (single file)
```

Key patterns to copy:
- `NSObject` subclass for the entry point
- Actor-based connection wrapper for thread safety
- `NSLock` for the interrupt handle
- Raw result struct to pass data out of the actor
- `stripLimitOffset` helper for pagination
- Error enum conforming to `LocalizedError`
