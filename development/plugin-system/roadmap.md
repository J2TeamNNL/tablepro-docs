# Plugin System Roadmap

## Phase 0: Foundation - COMPLETE

Laid the groundwork for the plugin system.

**Delivered:**
- `TableProPluginKit` shared framework with all protocols and transfer types
- `PluginManager` singleton with bundle loading, version checking, code signature verification
- `PluginDriverAdapter` bridging plugin drivers to the app's `DatabaseDriver` protocol
- `PluginEntry`, `PluginSource`, `PluginError` data model
- Oracle driver as proof-of-concept plugin (OCI stub)

## Phase 1: Built-in Plugins - COMPLETE

Extracted all database drivers from the main app into plugin bundles.

**Delivered:**
- 8 driver plugins: MySQL, PostgreSQL, SQLite, SQL Server, ClickHouse, MongoDB, Redis, Oracle
- Multi-type support: MySQL handles MariaDB, PostgreSQL handles Redshift
- Custom connection fields (SQL Server schema field)
- `DatabaseManager` factory simplified to plugin lookup
- Removed all direct driver imports from main app target
- C bridge headers moved into respective plugin bundles (CMariaDB, CLibPQ, CFreeTDS, CLibMongoc, CRedis, COracle)

## Phase 2: Sideload - COMPLETE

Users can install third-party plugins from `.zip` files via Settings.

**Delivered:**
- Settings > Plugins tab (`PluginsSettingsView.swift`) with installed plugin list, enable/disable toggles, detail view
- "Install from File..." flow: `NSOpenPanel` -> zip extraction via `ditto` (async, non-blocking) -> code signature verification -> bundle loading
- Uninstall for user-installed plugins with confirmation dialog
- Team-pinned code signature verification using `SecRequirement` (not just any valid cert)
- Detailed error reporting: `signatureInvalid(detail:)`, `pluginConflict`, `appVersionTooOld`
- SwiftUI `.alert`-based error presentation (replaced unreliable `NSApp.keyWindow`)
- `DatabaseDriverFactory.createDriver` now throws instead of `fatalError` -- graceful error when plugin missing
- Thread-safe `driverPlugins` access (removed `nonisolated(unsafe)`, made factory `@MainActor`)
- Plugin tests: `PluginModelsTests` (7 tests), `ExportServiceStateTests` rewritten with `StubDriver` mock

## Phase 3: Marketplace - COMPLETE

GitHub-based plugin registry with in-app discovery.

**Delivered:**
- RegistryClient fetches flat JSON manifest from `datlechin/tablepro-plugins` GitHub repo
- ETag/If-None-Match caching with UserDefaults offline fallback
- Browse tab in Settings > Plugins with search bar and category filter chips
- One-click install from registry: streaming download with progress, SHA-256 checksum verification, delegates to existing installPlugin(from:)
- PluginInstallTracker for per-plugin download/install state (downloading, installing, completed, failed)
- RegistryPluginRow with verified badge, author info, and contextual Install/Installed/Retry button
- RegistryPluginDetailView with expandable summary, category, compatibility info, and homepage link
- New error types: downloadFailed, incompatibleWithCurrentApp

## Phase 4: Auto-Updates

Version checking and update notifications for installed plugins.

**Scope:**
- Periodic version check against the registry
- Badge on Settings > Plugins when updates are available
- One-click update (download + replace bundle)
- Update tab showing changelog diff
- Opt-out per plugin

**Dependencies:** Phase 3 (requires registry with version metadata).

## Phase 5: Export Plugins - COMPLETE

Extracted all 5 built-in export formats into plugin bundles.

**Delivered:**
- `ExportFormatPlugin` protocol in TableProPluginKit with `formatId`, `export()`, `optionsView()`, `perTableOptionColumns`
- `PluginExportDataSource` protocol bridging `DatabaseDriver` for plugin data access
- `PluginExportProgress` thread-safe progress reporter with cancellation and UI throttling
- `PluginExportUtilities` shared helpers (JSON escaping, SQL comment sanitization, UTF-8 encoding)
- 5 export plugin bundles: CSVExport, JSONExport, SQLExport, XLSXExport, MQLExport
- Each plugin provides its own SwiftUI options view via `optionsView() -> AnyView?`
- Generic per-table option columns (SQL: Structure/Drop/Data, MQL: Drop/Indexes/Data)
- Dynamic format picker in Export dialog, filtered by database type compatibility
- `ExportDataSourceAdapter` bridges `DatabaseDriver` to `PluginExportDataSource`
- `ExportService` simplified to thin orchestrator delegating to plugins
- Removed 11 format-specific files from main app (5 ExportService extensions, XLSXWriter, 5 options views)

### Theme Plugins (Future)

**Scope:**
- JSON-based theme definitions (no executable code)
- Colors for editor, data grid, sidebar, toolbar
- Font overrides
- Stored in `~/Library/Application Support/TablePro/Themes/`
- Theme picker in Settings > Editor
- Shareable as single `.json` files

## Phase 6: Cell Renderers + Developer Portal

### Cell Renderers (Tier 1-2)

**Scope:**
- Custom rendering for specific column types (e.g., image preview, map view, color swatch)
- Tier 1: JSON config mapping column type patterns to built-in renderers
- Tier 2: JavaScript-based custom renderers in a `JSContext`

### Developer Portal

**Scope:**
- Documentation site for plugin developers
- Plugin submission and review workflow
- Code signing certificate distribution
- SDK download with Xcode project templates
- Example plugins repository

## Known Limitations / Tech Debt

Issues identified during Phase 2 implementation that should be addressed in future phases:

1. **Team ID placeholder**: `PluginManager.signingTeamId` is set to `"YOURTEAMID"` -- must be replaced with the actual Apple Developer Team ID before shipping sideloaded plugin support to users.

2. **`Bundle.unload()` unreliability**: macOS `Bundle.unload()` is not guaranteed to actually unload code. Disabled/uninstalled plugins may leave code in memory until app restart.

3. **No hot-reload**: Enabling a previously disabled plugin re-instantiates the class but doesn't reconnect existing sessions using that driver.

4. **`executeParameterized` default is SQL injection-adjacent**: The default implementation does string replacement of `?` placeholders, which relies on single-quote escaping. Drivers should override with native prepared statements.

5. **`PluginDriverAdapter.beginTransaction` uses hardcoded SQL**: Sends `BEGIN` regardless of database type. Drivers that need different transaction syntax (e.g., Oracle's implicit transactions) must handle this at the plugin level.

6. **No plugin dependency resolution**: Plugins cannot declare dependencies on other plugins. Each plugin must be self-contained.

7. **Single-zip install format**: Only `.zip` archives supported. No support for direct `.tableplugin` bundle drag-and-drop.

## Timeline

| Phase | Status | Dependencies |
|-------|--------|-------------|
| 0: Foundation | Done | -- |
| 1: Built-in Plugins | Done | Phase 0 |
| 2: Sideload | **Done** | Phase 1 |
| 3: Marketplace | **Done** | Phase 2 |
| 4: Auto-Updates | Next | Phase 3 |
| 5: Export Plugins | **Done** | Phase 2 |
| 6: Renderers + Portal | Planned | Phase 3, 5 |

Phases 5 and 3 can proceed in parallel now that Phase 2 is complete. Phase 5 (themes specifically) has no dependency on Phase 3 since themes are local files.
