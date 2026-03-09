# Plugin System

TablePro uses a native-bundle plugin architecture to load database drivers (and eventually other extensions) at runtime. Each plugin is a `.tableplugin` bundle that links the `TableProPluginKit` framework and exposes a principal class conforming to `TableProPlugin`.

Phase 0 (foundation), Phase 1 (all 8 built-in drivers extracted), and Phase 2 (sideload install/uninstall via Settings > Plugins tab) are all complete. The system is live and shipping.

Code review improvements applied during Phase 2: data race fix on plugin registry access, async process execution for install/uninstall operations, team-pinned `SecRequirement` signature verification, and proper error propagation throughout the plugin lifecycle.

## Documents

| Document | Description |
|----------|-------------|
| [architecture.md](architecture.md) | Three-tier model, extension points, trust levels, directory layout |
| [plugin-kit.md](plugin-kit.md) | TableProPluginKit framework: protocols, transfer types, versioning |
| [plugin-manager.md](plugin-manager.md) | PluginManager singleton: loading, registration, enable/disable, install/uninstall |
| [developer-guide.md](developer-guide.md) | How to build a new plugin from scratch |
| [ui-design.md](ui-design.md) | Settings tab wireframes and implementation |
| [roadmap.md](roadmap.md) | Phased rollout plan (Phase 0-6) |
| [security.md](security.md) | Code signing model, threat model, trust levels, known limitations |
| [troubleshooting.md](troubleshooting.md) | Common errors, signature failures, SourceKit noise, testing tips |
| [migration-guide.md](migration-guide.md) | Versioning policy, compatibility matrix, migration templates |

## File Map

```
Plugins/
  TableProPluginKit/          # Shared framework (linked by all plugins + main app)
    TableProPlugin.swift        # Base plugin protocol
    DriverPlugin.swift          # Driver extension protocol
    PluginDatabaseDriver.swift  # Driver implementation protocol (50+ methods)
    PluginCapability.swift      # Capability enum
    DriverConnectionConfig.swift # Connection config passed to createDriver()
    ConnectionField.swift       # Custom connection dialog fields
    PluginQueryResult.swift     # Query result transfer type
    PluginColumnInfo.swift      # Column metadata
    PluginIndexInfo.swift       # Index metadata
    PluginForeignKeyInfo.swift  # Foreign key metadata
    PluginTableInfo.swift       # Table list entry
    PluginTableMetadata.swift   # Table stats (size, row count, engine)
    PluginDatabaseMetadata.swift # Database stats
    ArrayExtension.swift        # Safe subscript helper
    MongoShellParser.swift      # Shared MongoDB shell parsing utilities

  ClickHouseDriverPlugin/     # ClickHouse driver
  MongoDBDriverPlugin/        # MongoDB driver
  MSSQLDriverPlugin/          # SQL Server driver (FreeTDS)
  MySQLDriverPlugin/          # MySQL/MariaDB driver (libmariadb)
  OracleDriverPlugin/         # Oracle driver (OCI stub)
  PostgreSQLDriverPlugin/     # PostgreSQL/Redshift driver (libpq)
  RedisDriverPlugin/          # Redis driver
  SQLiteDriverPlugin/         # SQLite driver (Foundation sqlite3)

TablePro/Core/Plugins/        # Main app infrastructure
  PluginManager.swift           # Singleton: load, register, enable/disable, install/uninstall
  PluginDriverAdapter.swift     # Bridges PluginDatabaseDriver -> DatabaseDriver protocol
  PluginModels.swift            # PluginEntry, PluginSource
  PluginError.swift             # Error types for plugin operations

TablePro/Views/Settings/
  PluginsSettingsView.swift     # Settings > Plugins tab UI (list, enable/disable, install/uninstall)
```

## Current Driver Plugins

| Plugin | Type ID | Additional IDs | C Library | Port |
|--------|---------|----------------|-----------|------|
| MySQL | `MySQL` | `MariaDB` | libmariadb | 3306 |
| PostgreSQL | `PostgreSQL` | `Redshift` | libpq | 5432 |
| SQLite | `SQLite` | -- | sqlite3 | 0 |
| SQL Server | `SQL Server` | -- | FreeTDS | 1433 |
| ClickHouse | `ClickHouse` | -- | HTTP API | 8123 |
| MongoDB | `MongoDB` | -- | libmongoc | 27017 |
| Redis | `Redis` | -- | hiredis | 6379 |
| Oracle | `Oracle` | -- | OCI stub | 1521 |
