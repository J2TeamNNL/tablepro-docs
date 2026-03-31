# TableProCore Package — Architecture Design

> Status: DRAFT — for review before implementation

## Overview

`TableProCore` is a new cross-platform Swift Package that provides shared business logic for both TablePro (macOS) and TablePro Mobile (iOS). It is written from scratch with clean architecture — not extracted from the macOS codebase.

**Principles:**
- Zero platform dependencies (Foundation + Swift stdlib only)
- Dependency injection — no `.shared` singletons
- Proper dependency direction: Core knows nothing about UI/platform
- Only abstract where implementations genuinely differ between platforms
- Keep PluginKit ↔ App type boundary (adapter pattern is intentional)

## Package Structure

```
Packages/TableProCore/
├── Package.swift
├── Sources/
│   ├── TableProPluginKit/     ← plugin protocols + transfer types (ABI boundary)
│   ├── TableProModels/        ← pure value types, zero deps except PluginKit
│   ├── TableProDatabase/      ← connection management, driver adapter
│   └── TableProQuery/         ← query building, filtering, parsing
└── Tests/
    ├── TableProModelsTests/
    ├── TableProDatabaseTests/
    └── TableProQueryTests/
```

Dependency graph:
```
TableProQuery ──→ TableProModels ──→ TableProPluginKit
                       ↑
TableProDatabase ──────┘
```

No cycles. Each target only depends downward.

---

## Module 1: TableProPluginKit

Mostly migrated from existing `Plugins/TableProPluginKit/`. Changes:

### Cleanup
- Remove `import SwiftUI` from `DriverPlugin.swift` — move `CompletionEntry` to a separate file that platforms can extend
- All types remain `Sendable` + `Codable` where applicable
- This is the **ABI boundary** — changes here require plugin recompilation

### Key Types (unchanged)
```swift
// Protocols
public protocol TableProPlugin { ... }
public protocol DriverPlugin: TableProPlugin { ... }
public protocol PluginDatabaseDriver: AnyObject, Sendable { ... }
public protocol ExportFormatPlugin: TableProPlugin { ... }
public protocol ImportFormatPlugin: TableProPlugin { ... }

// Transfer types (plugin → app)
public struct PluginQueryResult: Codable, Sendable { ... }
public struct PluginColumnInfo: Codable, Sendable { ... }
public struct PluginTableInfo: Codable, Sendable { ... }
public struct PluginIndexInfo: Codable, Sendable { ... }
public struct PluginForeignKeyInfo: Codable, Sendable { ... }
public struct PluginTableMetadata: Sendable { ... }
public struct PluginDatabaseMetadata: Sendable { ... }

// Config
public struct DriverConnectionConfig: Sendable { ... }
public struct ConnectionField: Codable, Sendable { ... }
public struct SQLDialectDescriptor: Sendable { ... }
```

### CompletionEntry Change
```swift
// OLD (in DriverPlugin.swift, requires SwiftUI):
// public struct CompletionEntry { var icon: Image ... }

// NEW: icon is a string identifier, platform resolves to Image/UIImage
public struct CompletionEntry: Sendable {
    public let label: String
    public let detail: String?
    public let iconName: String      // SF Symbol name — platform renders
    public let kind: CompletionKind

    public enum CompletionKind: String, Sendable {
        case keyword, function, table, column, schema, database, snippet
    }
}
```

---

## Module 2: TableProModels

Pure value types. No service calls, no platform types, no `@Observable`.

### DatabaseType

```swift
/// String-based struct for open extensibility.
/// All `switch` statements must include `default:`.
public struct DatabaseType: Hashable, Codable, Sendable, RawRepresentable {
    public let rawValue: String
    public init(rawValue: String) { self.rawValue = rawValue }

    // Known constants
    public static let mysql = DatabaseType(rawValue: "mysql")
    public static let mariadb = DatabaseType(rawValue: "mariadb")
    public static let postgresql = DatabaseType(rawValue: "postgresql")
    public static let sqlite = DatabaseType(rawValue: "sqlite")
    public static let redis = DatabaseType(rawValue: "redis")
    public static let mongodb = DatabaseType(rawValue: "mongodb")
    public static let clickhouse = DatabaseType(rawValue: "clickhouse")
    public static let mssql = DatabaseType(rawValue: "mssql")
    public static let oracle = DatabaseType(rawValue: "oracle")
    public static let duckdb = DatabaseType(rawValue: "duckdb")
    public static let cassandra = DatabaseType(rawValue: "cassandra")
    public static let redshift = DatabaseType(rawValue: "redshift")
    public static let etcd = DatabaseType(rawValue: "etcd")
    public static let cloudflareD1 = DatabaseType(rawValue: "cloudflared1")
    public static let dynamodb = DatabaseType(rawValue: "dynamodb")
    public static let bigquery = DatabaseType(rawValue: "bigquery")

    public static let allKnownTypes: [DatabaseType] = [
        .mysql, .mariadb, .postgresql, .sqlite, .redis, .mongodb,
        .clickhouse, .mssql, .oracle, .duckdb, .cassandra, .redshift,
        .etcd, .cloudflareD1, .dynamodb, .bigquery
    ]

    /// Plugin type ID for plugin lookup.
    /// Multi-type plugins: mariadb → "mysql", redshift → "postgresql"
    public var pluginTypeId: String {
        switch self {
        case .mariadb: return DatabaseType.mysql.rawValue
        case .redshift: return DatabaseType.postgresql.rawValue
        default: return rawValue
        }
    }
}

// NO iconImage, NO displayName, NO defaultPort, NO computed props calling services.
// UI metadata is queried via PluginMetadataProvider (see TableProDatabase module).
```

### DatabaseConnection

```swift
public struct DatabaseConnection: Identifiable, Codable, Sendable {
    public var id: UUID
    public var name: String
    public var type: DatabaseType
    public var host: String
    public var port: Int
    public var username: String
    public var database: String
    public var colorTag: String?              // color identifier, not Color/NSColor
    public var isReadOnly: Bool
    public var queryTimeoutSeconds: Int?
    public var additionalFields: [String: String]

    // SSH
    public var sshEnabled: Bool
    public var sshConfiguration: SSHConfiguration?

    // SSL
    public var sslEnabled: Bool
    public var sslConfiguration: SSLConfiguration?

    // Grouping
    public var groupId: UUID?
    public var sortOrder: Int

    public init(
        id: UUID = UUID(),
        name: String = "",
        type: DatabaseType = .mysql,
        host: String = "127.0.0.1",
        port: Int = 3306,
        username: String = "",
        database: String = "",
        colorTag: String? = nil,
        isReadOnly: Bool = false
    ) { ... }

    // NO password field — passwords live in SecureStore
    // NO NSImage, NO SwiftUI Color, NO @MainActor
    // NO displayColor, NO iconImage computed properties
}

public struct SSHConfiguration: Codable, Sendable {
    public var host: String
    public var port: Int
    public var username: String
    public var authMethod: SSHAuthMethod
    public var privateKeyPath: String?
    public var jumpHosts: [SSHJumpHost]

    public enum SSHAuthMethod: String, Codable, Sendable {
        case password, publicKey, agent
    }
}

public struct SSLConfiguration: Codable, Sendable {
    public var mode: SSLMode
    public var caCertificatePath: String?
    public var clientCertificatePath: String?
    public var clientKeyPath: String?

    public enum SSLMode: String, Codable, Sendable {
        case disable, require, verifyCa, verifyFull
    }
}
```

### Query Result Types (app-side, mapped from Plugin types via adapter)

```swift
public struct QueryResult: Sendable {
    public let columns: [ColumnInfo]
    public let rows: [[String?]]
    public let rowsAffected: Int
    public let executionTime: TimeInterval
    public let isTruncated: Bool
    public let statusMessage: String?
}

public struct ColumnInfo: Sendable, Identifiable {
    public let id: String                    // column name as ID
    public let name: String
    public let typeName: String
    public let isPrimaryKey: Bool
    public let isNullable: Bool
    public let defaultValue: String?
    public let comment: String?
    public let characterMaxLength: Int?
    public let ordinalPosition: Int
}

public struct TableInfo: Hashable, Sendable, Identifiable {
    public var id: String { name }
    public let name: String
    public let type: TableKind
    public let rowCount: Int?
    public let dataSize: Int?
    public let comment: String?

    public enum TableKind: String, Sendable {
        case table, view, materializedView, systemTable, sequence
    }
}

public struct IndexInfo: Sendable { ... }
public struct ForeignKeyInfo: Sendable { ... }

public enum ConnectionStatus: Sendable {
    case disconnected
    case connecting
    case connected
    case error(String)
}

public struct DatabaseError: Error, LocalizedError, Sendable {
    public let code: Int?
    public let message: String
    public let sqlState: String?
    public var errorDescription: String? { message }
}
```

### Filter Types

```swift
public struct TableFilter: Identifiable, Codable, Sendable {
    public var id: UUID
    public var columnName: String
    public var filterOperator: FilterOperator
    public var value: String
    public var secondValue: String           // for BETWEEN
    public var isEnabled: Bool
    public var rawSQL: String?               // for raw SQL filter mode

    public var isValid: Bool { ... }

    public static let rawSQLColumn = "__raw_sql__"
}

public enum FilterOperator: String, Codable, Sendable {
    case equal, notEqual
    case greaterThan, greaterThanOrEqual
    case lessThan, lessThanOrEqual
    case like, notLike
    case isNull, isNotNull
    case `in`, notIn
    case between
    case contains, startsWith, endsWith
}

public enum FilterLogicMode: String, Codable, Sendable {
    case and = "AND"
    case or = "OR"
}
```

### Tab / Pagination / Sort Types

```swift
public struct PaginationState: Codable, Sendable {
    public var pageSize: Int
    public var currentPage: Int
    public var totalRows: Int?

    public var currentOffset: Int { currentPage * pageSize }
    public var hasNextPage: Bool { ... }

    public mutating func reset() { currentPage = 0; totalRows = nil }
}

public struct SortState: Codable, Sendable {
    public var columns: [SortColumn]
    public var isSorting: Bool { !columns.isEmpty }

    public mutating func toggle(column: String) { ... }
    public mutating func clear() { columns = [] }
}

public struct SortColumn: Codable, Sendable {
    public let name: String
    public let ascending: Bool
}
```

### Schema Types

```swift
public struct ColumnDefinition: Codable, Sendable { ... }
public struct IndexDefinition: Codable, Sendable { ... }
public struct ForeignKeyDefinition: Codable, Sendable { ... }
public struct CreateTableOptions: Codable, Sendable { ... }
```

---

## Module 3: TableProDatabase

Connection management + driver adapter. Depends on Models + PluginKit.

### Platform Injection Protocols

Only 3 protocols — where macOS and iOS genuinely differ:

```swift
/// Loads driver plugins. macOS: Bundle.load(). iOS: static registration.
public protocol PluginLoader: Sendable {
    func availablePlugins() -> [any DriverPlugin]
    func driverPlugin(for typeId: String) -> (any DriverPlugin)?
}

/// Creates SSH tunnels. macOS: SSHTunnelManager. iOS: nil (not supported).
public protocol SSHProvider: Sendable {
    func createTunnel(
        config: SSHConfiguration,
        remoteHost: String,
        remotePort: Int
    ) async throws -> SSHTunnel
    func closeTunnel(for connectionId: UUID) async throws
}

public struct SSHTunnel: Sendable {
    public let localHost: String
    public let localPort: Int
}

/// Secure credential storage. Both platforms: Keychain (Security.framework).
/// Protocol exists because test mocking needs it.
public protocol SecureStore: Sendable {
    func store(_ value: String, forKey key: String) throws
    func retrieve(forKey key: String) throws -> String?
    func delete(forKey key: String) throws
}
```

### DatabaseDriver Protocol (app-side)

```swift
/// App-side driver protocol. PluginDriverAdapter bridges PluginDatabaseDriver → this.
public protocol DatabaseDriver: AnyObject, Sendable {
    // Connection lifecycle
    func connect() async throws
    func disconnect() async throws
    func ping() async throws -> Bool

    // Query execution
    func execute(query: String) async throws -> QueryResult
    func cancelCurrentQuery() async throws

    // Schema
    func fetchTables(schema: String?) async throws -> [TableInfo]
    func fetchColumns(table: String, schema: String?) async throws -> [ColumnInfo]
    func fetchIndexes(table: String, schema: String?) async throws -> [IndexInfo]
    func fetchForeignKeys(table: String, schema: String?) async throws -> [ForeignKeyInfo]
    func fetchDatabases() async throws -> [String]

    // Database/Schema switching
    func switchDatabase(to name: String) async throws
    var supportsSchemas: Bool { get }
    func switchSchema(to name: String) async throws
    var currentSchema: String? { get }

    // Transactions
    var supportsTransactions: Bool { get }
    func beginTransaction() async throws
    func commitTransaction() async throws
    func rollbackTransaction() async throws

    // Metadata
    var serverVersion: String? { get }
}
```

### PluginDriverAdapter

Maps `PluginDatabaseDriver` → `DatabaseDriver`. Same role as current adapter but written clean.

```swift
public final class PluginDriverAdapter: DatabaseDriver {
    private let pluginDriver: any PluginDatabaseDriver
    private let connection: DatabaseConnection

    public init(connection: DatabaseConnection, pluginDriver: any PluginDatabaseDriver) {
        self.connection = connection
        self.pluginDriver = pluginDriver
    }

    public func execute(query: String) async throws -> QueryResult {
        let pluginResult = try await pluginDriver.execute(query: query)
        return QueryResult(from: pluginResult)    // map Plugin types → App types
    }

    public func fetchTables(schema: String?) async throws -> [TableInfo] {
        let pluginTables = try await pluginDriver.fetchTables(schema: schema)
        return pluginTables.map { TableInfo(from: $0) }
    }

    // ... etc — clean mapping, no leaked internals
}

// Clean mapping extensions
extension QueryResult {
    init(from plugin: PluginQueryResult) { ... }
}

extension TableInfo {
    init(from plugin: PluginTableInfo) { ... }
}
```

### ConnectionManager

Concrete class (not protocol — 95% shared logic). Platform differences injected.

```swift
public final class ConnectionManager: @unchecked Sendable {
    private let pluginLoader: PluginLoader
    private let secureStore: SecureStore
    private let sshProvider: SSHProvider?

    // State
    private var sessions: [UUID: ConnectionSession] = [:]

    public init(
        pluginLoader: PluginLoader,
        secureStore: SecureStore,
        sshProvider: SSHProvider? = nil
    ) {
        self.pluginLoader = pluginLoader
        self.secureStore = secureStore
        self.sshProvider = sshProvider
    }

    /// Connect to a database. Returns session on success.
    public func connect(_ connection: DatabaseConnection) async throws -> ConnectionSession {
        // 1. Resolve password from SecureStore
        let password = try secureStore.retrieve(forKey: connection.id.uuidString)

        // 2. Set up SSH tunnel if needed
        var effectiveHost = connection.host
        var effectivePort = connection.port
        if connection.sshEnabled, let ssh = connection.sshConfiguration, let provider = sshProvider {
            let tunnel = try await provider.createTunnel(
                config: ssh,
                remoteHost: connection.host,
                remotePort: connection.port
            )
            effectiveHost = tunnel.localHost
            effectivePort = tunnel.localPort
        }

        // 3. Create driver via plugin
        guard let plugin = pluginLoader.driverPlugin(for: connection.type.pluginTypeId) else {
            throw ConnectionError.pluginNotFound(connection.type.rawValue)
        }
        let config = DriverConnectionConfig(
            host: effectiveHost,
            port: effectivePort,
            user: connection.username,
            password: password ?? "",
            database: connection.database,
            additionalFields: connection.additionalFields
        )
        let pluginDriver = plugin.createDriver(config: config)

        // 4. Connect
        let driver = PluginDriverAdapter(connection: connection, pluginDriver: pluginDriver)
        try await driver.connect()

        // 5. Create session
        let session = ConnectionSession(
            connectionId: connection.id,
            driver: driver,
            activeDatabase: connection.database,
            status: .connected
        )
        sessions[connection.id] = session
        return session
    }

    public func disconnect(_ connectionId: UUID) async throws {
        guard let session = sessions[connectionId] else { return }
        try await session.driver.disconnect()
        if let sshProvider, session.connection.sshEnabled {
            try await sshProvider.closeTunnel(for: connectionId)
        }
        sessions.removeValue(forKey: connectionId)
    }

    public func session(for connectionId: UUID) -> ConnectionSession? {
        sessions[connectionId]
    }
}

public enum ConnectionError: Error, LocalizedError {
    case pluginNotFound(String)
    case notConnected
    case sshNotSupported

    public var errorDescription: String? {
        switch self {
        case .pluginNotFound(let type): return "No driver plugin for database type: \(type)"
        case .notConnected: return "Not connected to database"
        case .sshNotSupported: return "SSH tunneling is not available on this platform"
        }
    }
}
```

### ConnectionSession

```swift
public struct ConnectionSession: Sendable {
    public let connectionId: UUID
    public let driver: any DatabaseDriver
    public var activeDatabase: String
    public var currentSchema: String?
    public var status: ConnectionStatus
    public var tables: [TableInfo]

    // NO UI state (isExpanded, selectedTable, etc.)
    // NO @Observable — platform layer wraps this in observable if needed
}
```

### PluginMetadataProvider

Replaces the current pattern where `DatabaseType` computed properties call `PluginMetadataRegistry.shared`.

```swift
/// Provides metadata about database types from loaded plugins.
/// Replaces computed properties on DatabaseType that called singletons.
public final class PluginMetadataProvider: Sendable {
    private let pluginLoader: PluginLoader

    public init(pluginLoader: PluginLoader) {
        self.pluginLoader = pluginLoader
    }

    public func displayName(for type: DatabaseType) -> String {
        plugin(for: type)?.databaseDisplayName ?? type.rawValue.capitalized
    }

    public func defaultPort(for type: DatabaseType) -> Int {
        plugin(for: type)?.defaultPort ?? 3306
    }

    public func iconName(for type: DatabaseType) -> String {
        plugin(for: type)?.iconName ?? "server.rack"
    }

    public func supportsSSH(for type: DatabaseType) -> Bool {
        plugin(for: type)?.supportsSSH ?? false
    }

    public func supportsSSL(for type: DatabaseType) -> Bool {
        plugin(for: type)?.supportsSSL ?? false
    }

    public func sqlDialect(for type: DatabaseType) -> SQLDialectDescriptor? {
        plugin(for: type)?.sqlDialect
    }

    // ... other metadata queries

    private func plugin(for type: DatabaseType) -> (any DriverPlugin)? {
        pluginLoader.driverPlugin(for: type.pluginTypeId)
    }
}
```

---

## Module 4: TableProQuery

Query building, filtering, SQL generation. Pure logic, depends on Models + PluginKit.

### TableQueryBuilder

```swift
public struct TableQueryBuilder: Sendable {
    private let dialect: SQLDialectDescriptor?
    private let pluginDriver: (any PluginDatabaseDriver)?

    public init(dialect: SQLDialectDescriptor? = nil, pluginDriver: (any PluginDatabaseDriver)? = nil) {
        self.dialect = dialect
        self.pluginDriver = pluginDriver
    }

    /// Build a base SELECT query for browsing a table.
    public func buildBrowseQuery(
        tableName: String,
        sortState: SortState = SortState(),
        limit: Int,
        offset: Int
    ) -> String { ... }

    /// Build a filtered SELECT query.
    public func buildFilteredQuery(
        tableName: String,
        filters: [TableFilter],
        logicMode: FilterLogicMode = .and,
        sortState: SortState = SortState(),
        limit: Int,
        offset: Int
    ) -> String { ... }
}
```

### FilterSQLGenerator

```swift
public struct FilterSQLGenerator: Sendable {
    private let dialect: SQLDialectDescriptor

    public init(dialect: SQLDialectDescriptor) {
        self.dialect = dialect
    }

    public func generateWhereClause(
        from filters: [TableFilter],
        logicMode: FilterLogicMode
    ) -> String { ... }
}
```

### SQLStatementGenerator

```swift
/// Generates INSERT/UPDATE/DELETE statements from row changes.
public struct SQLStatementGenerator: Sendable {
    private let dialect: SQLDialectDescriptor

    public func generateInsert(table: String, columns: [String], values: [String?]) -> String { ... }
    public func generateUpdate(table: String, changes: [String: String?], where: [String: String]) -> String { ... }
    public func generateDelete(table: String, where: [String: String]) -> String { ... }
}
```

### RowParser

```swift
public protocol RowDataParser: Sendable {
    func parse(text: String, columns: [String]) throws -> [[String?]]
}

public struct TSVRowParser: RowDataParser { ... }
public struct CSVRowParser: RowDataParser { ... }
```

---

## Platform Integration

### macOS (TablePro.xcodeproj)

```swift
// macOS-specific implementations
final class BundlePluginLoader: PluginLoader {
    func availablePlugins() -> [any DriverPlugin] {
        // Load .tableplugin bundles from app bundle + user plugins directory
    }
}

final class SSHTunnelProvider: SSHProvider {
    func createTunnel(...) async throws -> SSHTunnel {
        // Existing SSHTunnelManager logic
    }
}

final class KeychainStore: SecureStore {
    // Security.framework Keychain access
}

// DatabaseType UI extensions (macOS only)
extension DatabaseType {
    var iconImage: Image { ... }       // SwiftUI Image from SF Symbol
    var displayColor: Color { ... }    // SwiftUI Color
}
```

### iOS (TableProMobile.xcodeproj)

```swift
// iOS-specific implementations
final class StaticPluginLoader: PluginLoader {
    func availablePlugins() -> [any DriverPlugin] {
        // Return compiled-in plugins: MySQL, PostgreSQL, SQLite, Redis
        [MySQLDriverPlugin(), PostgreSQLDriverPlugin(), SQLiteDriverPlugin(), RedisDriverPlugin()]
    }
}

// No SSHProvider — pass nil to ConnectionManager

final class KeychainStore: SecureStore {
    // Same Security.framework — works on iOS too
}

// DatabaseType UI extensions (iOS only)
extension DatabaseType {
    var iconImage: Image { ... }       // Same SF Symbols, SwiftUI Image
    var displayColor: Color { ... }
}
```

---

## Migration Strategy

```
Phase 1: Write TableProCore (new package, clean code)
         Tests pass independently

Phase 2: iOS app depends on TableProCore
         Build iOS app

Phase 3: macOS app gradually migrates to TableProCore
         Module by module, behind feature flags if needed
         Old code removed only after migration verified
```

macOS app continues working with existing code throughout. Zero risk.
