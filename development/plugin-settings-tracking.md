# Plugin Settings — Progress Tracking

Analysis date: 2026-03-11 (updated).

## Current State Summary

The plugin settings system has two dimensions:

1. **Plugin management** (Settings > Plugins) — enable/disable, install/uninstall. Fully working.
2. **Per-plugin configuration** — export plugins expose `optionsView()` with persistent settings via `PluginSettingsStorage`. Import plugin (SQL) has options UI but no persistence. Driver plugins have zero configurable settings.

Plugin enable/disable state lives in `UserDefaults["com.TablePro.disabledPlugins"]` (namespaced, with legacy key migration).

---

## Plugin Management UI

| Feature                                                  | Status | File                                                | Notes                                                       |
| -------------------------------------------------------- | ------ | --------------------------------------------------- | ----------------------------------------------------------- |
| Installed plugins list with toggle                       | Done   | `Views/Settings/Plugins/InstalledPluginsView.swift` | One row per `PluginEntry`, inline detail expansion          |
| Enable/disable toggle (live)                             | Done   | `Core/Plugins/PluginManager.swift:365`              | Immediate capability register/unregister, no restart needed |
| Plugin detail (version, bundle ID, source, capabilities) | Done   | `Views/Settings/Plugins/InstalledPluginsView.swift` | Shown on row expansion                                      |
| Install from file (.tableplugin, .zip)                   | Done   | `Views/Settings/Plugins/InstalledPluginsView.swift` | NSOpenPanel + drag-and-drop                                 |
| Uninstall user plugins                                   | Done   | `Views/Settings/Plugins/InstalledPluginsView.swift` | Destructive button with AlertHelper.confirmDestructive      |
| Restart-required banner                                  | Done   | `Views/Settings/Plugins/InstalledPluginsView.swift` | Orange dismissible banner after uninstall                   |
| Browse registry                                          | Done   | `Views/Settings/Plugins/BrowsePluginsView.swift`    | Remote manifest from GitHub, search + category filter       |
| Registry install with progress                           | Done   | `Views/Settings/Plugins/RegistryPluginRow.swift`    | Download + SHA-256 verification, multi-phase progress       |
| Contextual install prompt (connection flow)              | Done   | `Views/Connection/PluginInstallModifier.swift`      | Alert when opening DB type with missing driver              |
| Code signature verification                              | Done   | `Core/Plugins/PluginManager.swift:558`              | Team ID `D7HJ5TFYCU` for user-installed plugins            |

## Per-Plugin Configuration

### Export Plugins

| Plugin | Options Model                                                                                               | Options View            | Persistence                          | Status |
| ------ | ----------------------------------------------------------------------------------------------------------- | ----------------------- | ------------------------------------ | ------ |
| CSV    | `CSVExportOptions` — delimiter, quoting, null handling, formula sanitize, line breaks, decimals, header row | `CSVExportOptionsView`  | `PluginSettingsStorage` (persisted)  | Done   |
| XLSX   | `XLSXExportOptions` — header row, null handling                                                             | `XLSXExportOptionsView` | `PluginSettingsStorage` (persisted)  | Done   |
| JSON   | `JSONExportOptions` — pretty print, null values, preserve-as-strings                                        | `JSONExportOptionsView` | `PluginSettingsStorage` (persisted)  | Done   |
| SQL    | `SQLExportOptions` — gzip, batch size                                                                       | `SQLExportOptionsView`  | `PluginSettingsStorage` (persisted)  | Done   |
| MQL    | `MQLExportOptions`                                                                                          | `MQLExportOptionsView`  | `PluginSettingsStorage` (persisted)  | Done   |

All export plugins use `PluginSettingsStorage(pluginId:)` which stores options in `UserDefaults` keyed as `com.TablePro.plugin.<id>.settings`, encoded via `JSONEncoder`.

### Import Plugins

| Plugin     | Options Model    | Options View           | Persistence | Status                                           |
| ---------- | ---------------- | ---------------------- | ----------- | ------------------------------------------------ |
| SQL Import | `SQLImportOptions` (Codable struct) | `SQLImportOptionsView` | `PluginSettingsStorage` (persisted)  | Done |

### Driver Plugins

All 8 driver plugins have zero per-plugin settings. No `optionsView()`, no configuration struct.

---

## Known Issues & Gaps

### High Priority

None — previously tracked high-priority issues have been resolved.

### Medium Priority

None — previously tracked medium-priority issues have been resolved.

### Low Priority

| Issue                        | Description                                                                                                    | Impact                                        |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------- | --------------------------------------------- |
| No settings protocol in SDK  | `TableProPluginKit` has no `PluginSettingsProtocol` for plugins to declare persistent preferences              | Third-party plugins can't define settings     |

### Resolved (since initial analysis)

| Issue                                        | Resolution                                                                                              |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| Export options not persisted                  | All 5 export plugins now use `PluginSettingsStorage` with `Codable` options models                      |
| `disabledPlugins` key not namespaced         | Now uses `"com.TablePro.disabledPlugins"` with legacy key migration (`PluginManager.swift:15-16,50-58`) |
| 4 dead capability types                      | Removed — `PluginCapability` now only has 3 cases: `databaseDriver`, `exportFormat`, `importFormat`     |
| `PluginInstallTracker.markInstalling()` unused | Now called in `BrowsePluginsView.swift:185` when download fraction reaches 1.0                         |
| SQL import options not persisted             | `SQLImportOptions` converted to `Codable` struct with `PluginSettingsStorage` persistence               |
| `additionalConnectionFields` hardcoded       | Connection form Advanced tab now dynamically renders fields from `DriverPlugin.additionalConnectionFields` with `ConnectionField.FieldType` support (text, secure, dropdown) |
| No driver plugin settings UI                 | `DriverPlugin.settingsView()` protocol method added with `nil` default; rendered in InstalledPluginsView |
| Hardcoded registry URL                       | `RegistryClient` now reads custom URL from UserDefaults with ETag invalidation on URL change            |
| `needsRestart` not persisted                 | Backed by UserDefaults, cleared on next plugin load cycle                                               |

---

## Recommended Next Steps

### Step 1 — Persist SQL import options

- Add `Codable` conformance to `SQLImportOptions`
- Add `PluginSettingsStorage` integration (same pattern as export plugins)

### Step 2 — Dynamic connection fields rendering

- Refactor `ConnectionFormView` Advanced tab to iterate `additionalConnectionFields` from `DriverPlugin` instead of hardcoding per-database sections
- Removes need for form changes when a plugin adds new fields

### Step 3 — Plugin settings protocol (SDK v2)

- Add `PluginSettingsProtocol` to `TableProPluginKit` with `settingsView() -> AnyView?` and `persistSettings()`/`loadSettings()`
- Render in Settings > Plugins detail expansion for plugins that implement it
- Driver plugins can then expose timeout, SSL, query behavior settings

---

## Key Files

| Component                      | Path                                                             |
| ------------------------------ | ---------------------------------------------------------------- |
| Settings tab container         | `TablePro/Views/Settings/PluginsSettingsView.swift`              |
| Installed list + toggle        | `TablePro/Views/Settings/Plugins/InstalledPluginsView.swift`     |
| Browse registry                | `TablePro/Views/Settings/Plugins/BrowsePluginsView.swift`        |
| Registry row + install         | `TablePro/Views/Settings/Plugins/RegistryPluginRow.swift`        |
| Registry detail                | `TablePro/Views/Settings/Plugins/RegistryPluginDetailView.swift` |
| PluginManager                  | `TablePro/Core/Plugins/PluginManager.swift`                      |
| Registry extension             | `TablePro/Core/Plugins/Registry/PluginManager+Registry.swift`    |
| Registry client                | `TablePro/Core/Plugins/Registry/RegistryClient.swift`            |
| Registry models                | `TablePro/Core/Plugins/Registry/RegistryModels.swift`            |
| Install tracker                | `TablePro/Core/Plugins/Registry/PluginInstallTracker.swift`      |
| Download count service         | `TablePro/Core/Plugins/Registry/DownloadCountService.swift`      |
| Plugin models                  | `TablePro/Core/Plugins/PluginModels.swift`                       |
| Plugin settings storage        | `Plugins/TableProPluginKit/PluginSettingsStorage.swift`          |
| Connection install prompt      | `TablePro/Views/Connection/PluginInstallModifier.swift`          |
| SDK — base protocol            | `Plugins/TableProPluginKit/TableProPlugin.swift`                 |
| SDK — driver protocol          | `Plugins/TableProPluginKit/DriverPlugin.swift`                   |
| SDK — export protocol          | `Plugins/TableProPluginKit/ExportFormatPlugin.swift`             |
| SDK — import protocol          | `Plugins/TableProPluginKit/ImportFormatPlugin.swift`             |
| SDK — capabilities             | `Plugins/TableProPluginKit/PluginCapability.swift`               |
| SDK — connection fields        | `Plugins/TableProPluginKit/ConnectionField.swift`                |
| CSV export (representative)    | `Plugins/CSVExportPlugin/CSVExportPlugin.swift`                  |
| SQL import plugin              | `Plugins/SQLImportPlugin/SQLImportPlugin.swift`                  |
| Connection form (adv. fields)  | `TablePro/Views/Connection/ConnectionFormView.swift`             |
