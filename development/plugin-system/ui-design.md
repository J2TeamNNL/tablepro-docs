# UI Design: Plugin Management

The Settings > Plugins tab is implemented and shipping. Users can view all loaded plugins, enable/disable them, and install third-party plugins from `.zip` files.

## Settings > Plugins Tab

### Installed Plugins List

```
+---------------------------------------------------------------+
| Settings                                                       |
|---------------------------------------------------------------|
| General | Editor | Plugins | ...                               |
|---------------------------------------------------------------|
|                                                                |
|  Installed Plugins                                             |
|                                                                |
|  +-----------------------------------------------------------+|
|  | [icon] MySQL Driver             v1.0.0    Built-in   [ON] ||
|  | [icon] PostgreSQL Driver        v1.0.0    Built-in   [ON] ||
|  | [icon] SQLite Driver            v1.0.0    Built-in   [ON] ||
|  | [icon] SQL Server Driver        v1.0.0    Built-in   [ON] ||
|  | [icon] ClickHouse Driver        v1.0.0    Built-in   [ON] ||
|  | [icon] MongoDB Driver           v1.0.0    Built-in   [ON] ||
|  | [icon] Redis Driver             v1.0.0    Built-in   [ON] ||
|  | [icon] Oracle Driver            v1.0.0    Built-in   [ON] ||
|  +-----------------------------------------------------------+|
|                                                                |
|  [Install from File...]                                        |
|                                                                |
+---------------------------------------------------------------+
```

Each row displays:
- SF Symbol icon from `DriverPlugin.iconName`
- Plugin name and version string
- Source badge: blue "Built-in" for bundled plugins, green "User" for user-installed plugins
- Enable/disable Toggle

Clicking a row expands or collapses an inline detail section below it.

The "Install from File..." button opens an `NSOpenPanel` filtered to `.zip` files. A progress indicator is shown during extraction and validation.

### Plugin Detail Section

Clicking a plugin row expands a detail section inline:

```
+---------------------------------------------------------------+
|  MySQL Driver                                                  |
|---------------------------------------------------------------|
|                                                                |
|  Version:       1.0.0                                          |
|  Bundle ID:     com.TablePro.MySQLDriverPlugin                 |
|  Source:        Built-in                                        |
|                                                                |
|  Capabilities:  Database Driver                                |
|  Database Type: MySQL                                          |
|  Also handles:  MariaDB                                        |
|  Default Port:  3306                                           |
|                                                                |
|  Description:                                                  |
|  MySQL/MariaDB support via libmariadb                          |
|                                                                |
|                                     [Uninstall] (user only)    |
|                                                                |
+---------------------------------------------------------------+
```

Implementation details:
- Uses a SwiftUI `Form` with `.grouped` style.
- Fields shown: Version, Bundle ID, Source, Capabilities, Database Type, Also handles (if `additionalDatabaseTypeIds` is non-empty), Default Port, Description.
- The Uninstall button only appears for user-installed plugins. Clicking it shows a confirmation dialog before removal.
- Built-in plugins cannot be uninstalled; they can only be disabled via the toggle in the list row.

### Disabled State

When a plugin is disabled:
- The toggle shows OFF.
- The row appears dimmed.
- The database type is no longer available in the connection dialog.
- Existing connections using that type show an error on next connect attempt.

## Connection Dialog: Dynamic Fields

When creating a new connection, the dialog renders fields based on the selected driver plugin.

### Standard Fields (always shown)

```
+-----------------------------------------------+
|  New Connection                                |
|-----------------------------------------------|
|  Type:     [MySQL v]                           |
|  Name:     [                          ]        |
|  Host:     [localhost                 ]        |
|  Port:     [3306                      ]        |
|  Username: [root                      ]        |
|  Password: [********                  ]        |
|  Database: [mydb                      ]        |
+-----------------------------------------------+
```

### With Additional Fields (e.g., SQL Server)

```
+-----------------------------------------------+
|  New Connection                                |
|-----------------------------------------------|
|  Type:     [SQL Server v]                      |
|  Name:     [                          ]        |
|  Host:     [localhost                 ]        |
|  Port:     [1433                      ]        |
|  Username: [sa                        ]        |
|  Password: [********                  ]        |
|  Database: [master                    ]        |
|                                                |
|  --- Driver-specific ---                       |
|  Schema:   [dbo                       ]        |
+-----------------------------------------------+
```

The "Driver-specific" section is generated from `DriverPlugin.additionalConnectionFields`. Each `ConnectionField` maps to a text field (or secure text field if `isSecure` is true). Required fields show validation errors if left empty.

## Error Handling

Errors during plugin install and uninstall are handled with native SwiftUI and AppKit alerts:

- **Install errors**: Shown via a SwiftUI `.alert` modifier on the settings view. Error cases include:
  - Invalid bundle (missing `NSPrincipalClass`, wrong extension, no `DriverPlugin` conformance)
  - Signature verification failed
  - Plugin conflict (a plugin with the same bundle ID is already installed)
  - App version too old (`TableProMinAppVersion` exceeds current app version)
- **Uninstall confirmation**: Uses `AlertHelper.confirmDestructive` to show a confirmation dialog before removing a user-installed plugin.
- **Runtime load failures**: Logged via OSLog. If a plugin fails to load at startup, it is skipped and the remaining plugins continue loading.

## Browse Tab (Phase 3)

The Plugins settings pane uses a segmented picker to switch between Installed and Browse sub-tabs.

```
+---------------------------------------------------------------+
| Settings                                                       |
|---------------------------------------------------------------|
| General | Editor | Plugins | ...                               |
|---------------------------------------------------------------|
|                                                                |
|           [ Installed | Browse ]   (segmented picker)          |
|                                                                |
+---------------------------------------------------------------+
```

When "Browse" is selected, the view shows the registry contents fetched from GitHub.

### Browse Tab Layout

```
+---------------------------------------------------------------+
|  [Search plugins...                                    ]       |
|                                                                |
|  [All] [Database Drivers] [Export Formats] [Themes]            |
|                                                                |
|  +-----------------------------------------------------------+|
|  | [icon] CockroachDB Driver  ✓  v0.1.0  by dev    [Install]||
|  | [icon] DuckDB Driver         v0.2.0  by dev    [Install]  ||
|  | [icon] Parquet Export      ✓  v1.0.0  by dev  [Installed]  ||
|  +-----------------------------------------------------------+|
|                                                                |
+---------------------------------------------------------------+
```

- Search bar filters the plugin list by name and description (local client-side filtering).
- Category filter chips sit below the search bar. Tapping a chip filters the list to that category. "All" shows everything.
- The plugin list scrolls vertically. Each entry is a `RegistryPluginRow`.

### RegistryPluginRow

Each row displays:
- SF Symbol icon from the registry manifest's `iconName` field
- Plugin name, with a checkmark badge inline if the plugin has verified trust level
- Version string and author name (e.g., "v0.1.0  by dev")
- Action button on the trailing edge (see install flow states below)

Clicking a row expands a `RegistryPluginDetailView` inline below it, showing:
- Description text (multi-line, truncated with a "Show More" toggle if longer than 3 lines)
- Category label
- Compatibility info (minimum app version required)
- Homepage link (opens in default browser)

### Install Flow States

The action button on each row transitions through these states:

| State | Button | Behavior |
|-------|--------|----------|
| Not installed | "Install" (blue) | Starts streaming download from the registry URL |
| Downloading | Progress bar with percentage | `PluginInstallTracker` updates progress; cancellable |
| Installing | Spinner with "Installing..." | Zip extraction, signature check, bundle load via `installPlugin(from:)` |
| Completed | "Installed" (gray, disabled) | Plugin now appears in the Installed tab |
| Failed | "Retry" (red) | Resets to downloading state on tap |

`PluginInstallTracker` holds per-plugin state keyed by bundle ID. It publishes state changes so the row updates reactively.

Download and install steps:
1. Streaming download with `URLSession` data task, tracking bytes received vs. expected content length.
2. SHA-256 checksum of the downloaded zip verified against the manifest's `checksum` field.
3. On checksum match, delegates to the existing `installPlugin(from:)` path (zip extraction, code signature verification, bundle loading).
4. On failure, the row shows the error inline and switches to the Retry state.

### Error, Loading, and Empty States

- **Loading**: A `ProgressView` spinner centered in the Browse tab while the initial registry fetch is in progress.
- **Fetch error**: A centered message with the error description and a "Try Again" button that re-triggers `RegistryClient.fetchManifest()`.
- **Offline fallback**: If the network request fails but a cached manifest exists in UserDefaults, the cached data is shown with a subtle "Showing cached data" label below the search bar.
- **Empty search results**: "No plugins match your search." text centered in the list area.
- **Incompatible plugin**: If a plugin's `minAppVersion` exceeds the current app version, the Install button is replaced with "Requires vX.Y.Z" in gray text. The detail view explains the version requirement.
