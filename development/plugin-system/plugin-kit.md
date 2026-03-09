# TableProPluginKit Framework

`TableProPluginKit` is a shared framework linked by both the main app and all plugins. It defines the protocol contracts and transfer types that cross the plugin boundary.

## Protocols

### TableProPlugin

Base protocol for all plugins. Every plugin's principal class must conform to this.

```swift
public protocol TableProPlugin: AnyObject {
    static var pluginName: String { get }
    static var pluginVersion: String { get }
    static var pluginDescription: String { get }
    static var capabilities: [PluginCapability] { get }

    init()
}
```

All metadata is on the type itself (static properties), not on instances. The `init()` requirement enables `PluginManager` to instantiate plugins without knowing their concrete type.

### DriverPlugin

Extends `TableProPlugin` for database driver plugins.

```swift
public protocol DriverPlugin: TableProPlugin {
    static var databaseTypeId: String { get }
    static var databaseDisplayName: String { get }
    static var iconName: String { get }
    static var defaultPort: Int { get }
    static var additionalConnectionFields: [ConnectionField] { get }  // default: []
    static var additionalDatabaseTypeIds: [String] { get }            // default: []

    func createDriver(config: DriverConnectionConfig) -> any PluginDatabaseDriver
}
```

Key design points:

- **`databaseTypeId`**: Primary lookup key (e.g., `"MySQL"`, `"PostgreSQL"`). Must match the `DatabaseConnection.type` string used throughout the app.
- **`additionalDatabaseTypeIds`**: Allows one plugin to handle multiple database types. MySQL handles `"MariaDB"`, PostgreSQL handles `"Redshift"`.
- **`additionalConnectionFields`**: Extra fields shown in the connection dialog. SQL Server uses this for a schema field.
- **`createDriver(config:)`**: Factory method. Called each time a connection is opened.

### PluginDatabaseDriver

The main implementation protocol. This is what plugin authors spend most of their time on.

```swift
public protocol PluginDatabaseDriver: AnyObject, Sendable {
    // Connection lifecycle
    func connect() async throws
    func disconnect()
    func ping() async throws

    // Query execution
    func execute(query: String) async throws -> PluginQueryResult
    func fetchRowCount(query: String) async throws -> Int
    func fetchRows(query: String, offset: Int, limit: Int) async throws -> PluginQueryResult
    func executeParameterized(query: String, parameters: [String?]) async throws -> PluginQueryResult

    // Schema inspection
    func fetchTables(schema: String?) async throws -> [PluginTableInfo]
    func fetchColumns(table: String, schema: String?) async throws -> [PluginColumnInfo]
    func fetchIndexes(table: String, schema: String?) async throws -> [PluginIndexInfo]
    func fetchForeignKeys(table: String, schema: String?) async throws -> [PluginForeignKeyInfo]
    func fetchTableDDL(table: String, schema: String?) async throws -> String
    func fetchViewDefinition(view: String, schema: String?) async throws -> String
    func fetchTableMetadata(table: String, schema: String?) async throws -> PluginTableMetadata

    // Database/schema navigation
    func fetchDatabases() async throws -> [String]
    func fetchDatabaseMetadata(_ database: String) async throws -> PluginDatabaseMetadata
    func fetchSchemas() async throws -> [String]
    func switchSchema(to schema: String) async throws
    func switchDatabase(to database: String) async throws

    // Batch operations
    func fetchApproximateRowCount(table: String, schema: String?) async throws -> Int?
    func fetchAllColumns(schema: String?) async throws -> [String: [PluginColumnInfo]]
    func fetchAllForeignKeys(schema: String?) async throws -> [String: [PluginForeignKeyInfo]]
    func fetchAllDatabaseMetadata() async throws -> [PluginDatabaseMetadata]
    func fetchDependentTypes(table: String, schema: String?) async throws -> [(name: String, labels: [String])]
    func fetchDependentSequences(table: String, schema: String?) async throws -> [(name: String, ddl: String)]
    func createDatabase(name: String, charset: String, collation: String?) async throws

    // Transactions
    func beginTransaction() async throws
    func commitTransaction() async throws
    func rollbackTransaction() async throws

    // Execution control
    func cancelQuery() throws
    func applyQueryTimeout(_ seconds: Int) async throws

    // Properties
    var supportsSchemas: Bool { get }
    var supportsTransactions: Bool { get }
    var currentSchema: String? { get }
    var serverVersion: String? { get }
}
```

#### Minimum Required Methods

Most methods have default implementations in a protocol extension. The minimum a driver must implement:

- `connect()` / `disconnect()`
- `execute(query:)`
- `fetchTables(schema:)` / `fetchColumns(table:schema:)`
- `fetchIndexes(table:schema:)` / `fetchForeignKeys(table:schema:)`
- `fetchTableDDL(table:schema:)` / `fetchViewDefinition(view:schema:)`
- `fetchTableMetadata(table:schema:)`
- `fetchDatabases()` / `fetchDatabaseMetadata(_:)`

Everything else falls back to sensible defaults (e.g., `ping()` runs `SELECT 1`, `fetchRowCount()` wraps the query in `SELECT COUNT(*)`, `fetchRows()` appends `LIMIT/OFFSET`).

#### Methods with Default Implementations

| Method | Default behavior |
|--------|-----------------|
| `ping()` | Runs `SELECT 1` |
| `fetchRowCount(query:)` | Wraps in `SELECT COUNT(*) FROM (...) _t` |
| `fetchRows(query:offset:limit:)` | Appends `LIMIT/OFFSET` to query |
| `executeParameterized(query:parameters:)` | Replaces `?` placeholders with escaped values |
| `fetchSchemas()` | Returns `[]` |
| `switchSchema(to:)` | No-op |
| `switchDatabase(to:)` | Throws "This driver does not support database switching" |
| `createDatabase(name:charset:collation:)` | Throws NSError "createDatabase not supported" |
| `fetchApproximateRowCount(table:schema:)` | Returns `nil` |
| `fetchAllColumns(schema:)` | Iterates `fetchTables` + `fetchColumns` per table |
| `fetchAllForeignKeys(schema:)` | Iterates `fetchTables` + `fetchForeignKeys` per table |
| `fetchAllDatabaseMetadata()` | Iterates `fetchDatabases` + `fetchDatabaseMetadata` per DB |
| `fetchDependentTypes(table:schema:)` | Returns `[]` |
| `fetchDependentSequences(table:schema:)` | Returns `[]` |
| `beginTransaction()` / `commitTransaction()` / `rollbackTransaction()` | Runs `BEGIN` / `COMMIT` / `ROLLBACK` |
| `cancelQuery()` | No-op |
| `applyQueryTimeout(_:)` | No-op |
| `supportsSchemas` | `false` |
| `supportsTransactions` | `true` |
| `currentSchema` | `nil` |
| `serverVersion` | `nil` |

#### switchDatabase(to:) - Driver Overrides

The default implementation throws an error. Drivers that support database switching must override this method with database-specific logic:

| Driver | Override behavior |
|--------|-----------------|
| MySQL | Runs `USE \`escapedName\`` |
| MSSQL | Uses FreeTDS native `dbuse()` API |
| ClickHouse | Has its own override for ClickHouse database switching |
| PostgreSQL | Does not override -- database switching requires a full reconnect |
| Redis | Does not override -- not applicable |
| MongoDB | Does not override -- not applicable |

This is an optional override. Drivers only need to implement it if the database engine supports switching databases on an existing connection.

## Transfer Types

All data crossing the plugin boundary uses plain `Codable, Sendable` structs. No classes, no app-internal types.

### PluginQueryResult

```swift
public struct PluginQueryResult: Codable, Sendable {
    public let columns: [String]
    public let columnTypeNames: [String]
    public let rows: [[String?]]
    public let rowsAffected: Int
    public let executionTime: TimeInterval
}
```

All cell values are stringified. The main app maps `columnTypeNames` to its internal `ColumnType` enum via `PluginDriverAdapter.mapColumnType()`.

### PluginColumnInfo

```swift
public struct PluginColumnInfo: Codable, Sendable {
    public let name: String
    public let dataType: String
    public let isNullable: Bool
    public let isPrimaryKey: Bool
    public let defaultValue: String?
    public let extra: String?       // e.g., "auto_increment"
    public let charset: String?
    public let collation: String?
    public let comment: String?
}
```

### PluginIndexInfo

```swift
public struct PluginIndexInfo: Codable, Sendable {
    public let name: String
    public let columns: [String]
    public let isUnique: Bool
    public let isPrimary: Bool
    public let type: String         // e.g., "BTREE", "HASH"
}
```

### PluginForeignKeyInfo

```swift
public struct PluginForeignKeyInfo: Codable, Sendable {
    public let name: String
    public let column: String
    public let referencedTable: String
    public let referencedColumn: String
    public let onDelete: String     // default: "NO ACTION"
    public let onUpdate: String     // default: "NO ACTION"
}
```

### PluginTableInfo

```swift
public struct PluginTableInfo: Codable, Sendable {
    public let name: String
    public let type: String         // "TABLE", "VIEW", "SYSTEM TABLE"
    public let rowCount: Int?
}
```

### PluginTableMetadata

```swift
public struct PluginTableMetadata: Codable, Sendable {
    public let tableName: String
    public let dataSize: Int64?
    public let indexSize: Int64?
    public let totalSize: Int64?
    public let rowCount: Int64?
    public let comment: String?
    public let engine: String?
}
```

### PluginDatabaseMetadata

```swift
public struct PluginDatabaseMetadata: Codable, Sendable {
    public let name: String
    public let tableCount: Int?
    public let sizeBytes: Int64?
    public let isSystemDatabase: Bool
}
```

## Connection Configuration

### DriverConnectionConfig

Passed to `createDriver(config:)`. Contains standard fields plus a dictionary for plugin-specific extras.

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

The `additionalFields` dictionary carries values from any `ConnectionField` entries declared by the plugin, plus internal fields like `driverVariant` (used by PostgreSQL to distinguish Redshift connections).

### ConnectionField

Declares a custom field in the connection dialog.

```swift
public struct ConnectionField: Codable, Sendable {
    public let id: String           // Key in additionalFields dictionary
    public let label: String        // Display label
    public let placeholder: String  // Placeholder text
    public let isRequired: Bool
    public let isSecure: Bool       // Renders as secure text field
    public let defaultValue: String?
}
```

## PluginCapability

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

A plugin declares its capabilities in `TableProPlugin.capabilities`. The `PluginManager` uses this to route registration. Currently only `.databaseDriver` triggers any registration logic.

## Multi-Type Support

A single plugin can handle multiple database types via `additionalDatabaseTypeIds`. The plugin is registered under all declared type IDs:

| Plugin | Primary ID | Additional IDs |
|--------|-----------|----------------|
| MySQLPlugin | `MySQL` | `MariaDB` |
| PostgreSQLPlugin | `PostgreSQL` | `Redshift` |

The PostgreSQL plugin uses `config.additionalFields["driverVariant"]` to decide whether to create a standard PostgreSQL driver or a Redshift-specific variant.

## Versioning

- **`TableProPluginKitVersion`** (Info.plist integer): Protocol version. Currently `1`. If a plugin declares a version higher than `PluginManager.currentPluginKitVersion`, loading is rejected.
- **`TableProMinAppVersion`** (Info.plist string, optional): Minimum app version. Compared via `.numeric` string comparison. Throws `appVersionTooOld(minimumRequired:currentApp:)` if the running app is older.
- **`pluginVersion`** (static property): Semver string for display purposes. Not enforced by the runtime.

The protocol version will increment when breaking changes are made to `PluginDatabaseDriver` or transfer types. Additive changes (new methods with default implementations) do not require a version bump.
