# NotificationCenter Refactor Plan

## Problem

TablePro uses ~40 custom `NotificationCenter` notifications for cross-component communication. This creates:

- **Untraceable coupling** — global broadcasts with no compile-time safety
- **Untyped payloads** — `userInfo` dictionaries with stringly-typed keys
- **Fan-out bugs** — `.refreshData` triggers ALL sidebar instances, not just the target connection
- **Overloaded semantics** — `databaseDidConnect` used both for "connection established" and "please reload sidebar"

## Target Architecture

Replace `NotificationCenter` with proper Swift patterns:

| Pattern                                 | When to Use                                               |
| --------------------------------------- | --------------------------------------------------------- |
| **`@Observable` + SwiftUI observation** | Settings, license status, any model already `@Observable` |
| **Direct method calls**                 | Parent → child, coordinator → owned viewmodel             |
| **`@FocusedValue`**                     | Menu/toolbar → active window's coordinator                |
| **Delegate/closure callbacks**          | 1:1 relationships (SSH tunnel → DatabaseManager)          |
| **Typed event bus (per-connection)**    | Multi-subscriber signals scoped to a connection ID        |

Keep `NotificationCenter` only for: AppKit → SwiftUI bridges where no shared reference exists (e.g., `NSApplicationDelegate` → SwiftUI scene).

---

## Phase 1: Remove Dead Notifications

**Status:** Done (PR [#281](https://github.com/datlechin/TablePro/pull/281))

Removed 11 notifications that had no sender OR no subscriber. Pure cleanup, zero risk.

### Removed (defined + observed, never posted):

- [x] `formatQueryRequested` — subscriber in `QueryEditorView`, no sender
- [x] `sendAIPrompt` — subscriber in `AIChatPanelView`, no sender
- [x] `reconnectDatabase` — subscriber in `MainContentCommandActions`, no sender
- [x] `refreshAll` — subscribers in `SidebarViewModel` + `MainContentCommandActions`, no sender
- [x] `connectionHealthStateChanged` — defined in `AppNotifications.swift`, no sender or subscriber
- [x] `applyAllFilters` — subscriber in `MainContentCommandActions`, no sender
- [x] `duplicateFilter` — subscriber in `MainContentCommandActions`, no sender
- [x] `removeFilter` — subscriber in `MainContentCommandActions`, no sender
- [x] `deselectConnection` — subscriber in `ContentView`, no sender

### Removed (posted, never observed):

- [x] `licenseStatusDidChange` — posted by `LicenseManager` (which is `@Observable`, consumers already use observation)
- [x] `pluginStateDidChange` — posted by `PluginManager`, no subscriber

Also removed cascading dead code: `handleRefreshAll`, `handleReconnect`, filter broadcast handlers, `DiscardAction.refreshAll` enum case. Net -139 lines across 12 files.

---

## Phase 2: Replace Settings Notifications with `@Observable`

**Status:** Done (PR [#282](https://github.com/datlechin/TablePro/pull/282))

Removed 7 notification names and `SettingsChangeInfo` infrastructure. Replaced Combine subscriber with direct call, converted 2 SwiftUI subscribers to `@Observable` observation.

### Removed (dead — zero subscribers):

- [x] `appearanceSettingsDidChange`
- [x] `generalSettingsDidChange`
- [x] `tabSettingsDidChange`
- [x] `keyboardSettingsDidChange`
- [x] `aiSettingsDidChange`
- [x] `settingsDidChange` (generic catch-all)

### Converted:

- [x] `historySettingsDidChange` — replaced Combine subscriber in `QueryHistoryManager` with direct `applySettingsChange()` call from `AppSettingsManager`
- [x] `editorSettingsDidChange` (SwiftUI) — `SQLEditorView` uses `.onChange(of: AppSettingsManager.shared.editor)`, `QueryEditorView` reads `@Observable` directly in `body`

### Kept (AppKit bridges):

- [x] `dataGridSettingsDidChange` — `DataGridView` (AppKit), `DataGridCellFactory` (AppKit)
- [x] `editorSettingsDidChange` — `SQLEditorCoordinator` (AppKit)
- [x] `accessibilityTextSizeDidChange` — system event bridge

Also removed: `SettingsChangeInfo` struct, `Notification.settingsChangeInfo` extension, `import Combine` from `QueryHistoryManager`. Net -120 lines.

---

## Phase 3: Replace Data Refresh with Direct Coordinator-to-Sidebar Calls

**Status:** Done (PR [#283](https://github.com/datlechin/TablePro/pull/283))

Gave the coordinator a direct `weak var sidebarViewModel` reference, replacing 12 global broadcasts with scoped `reloadSidebar()` calls. Fixed `.databaseDidConnect` abuse in save/discard paths. Sidebar reloads are now per-window instead of global.

### What changed:

- [x] `MainContentCoordinator` — added `weak var sidebarViewModel` + `reloadSidebar()` method
- [x] `SidebarView` — accepts `coordinator` parameter, wires `coordinator.sidebarViewModel = viewModel` on appear
- [x] `ContentView` — passes `coordinator: sessionState.coordinator` to `SidebarView`
- [x] `+Navigation.swift` — replaced all 10 `.refreshData` posts with `reloadSidebar()`
- [x] `+SaveChanges.swift` — replaced `.databaseDidConnect` abuse with `reloadSidebar()`
- [x] `+Discard.swift` — replaced `.databaseDidConnect` abuse with `reloadSidebar()`
- [x] `SidebarViewModel` — removed `Publishers.Merge` subscription for `.databaseDidConnect`/`.refreshData`
- [x] `MainContentCommandActions` — chained `coordinator?.reloadSidebar()` into `handleRefreshData()` and `handleDatabaseDidConnect()` so menu/toolbar/import/DatabaseManager signals still reach the sidebar

### What's kept:

- `.refreshData` — still posted by menu (Cmd+R), toolbar, `ImportDialog`, `DatabaseManager.applySchemaChanges()`. Flows through `MainContentCommandActions.handleRefreshData()` → chains `reloadSidebar()`.
- `.databaseDidConnect` — still posted by `DatabaseManager` (legitimate). Flows through `MainContentCommandActions.handleDatabaseDidConnect()` → chains `reloadSidebar()`. `AppDelegate` subscribers kept for file queue draining.

---

## Phase 4: Replace Sidebar Action Notifications with `@FocusedValue`

**Status:** Not started

Menu items post global notifications to reach the sidebar. These should use `@FocusedValue` to call the active window's sidebar directly.

- [ ] `copyTableNames` — menu → `SidebarViewModel.copySelectedTableNames()`
- [ ] `truncateTables` — menu → `SidebarViewModel.batchToggleTruncate()`
- [ ] `clearSelection` — menu → `SidebarViewModel.selectedTables.removeAll()`
- [ ] `showAllTables` — menu → coordinator action
- [ ] `showTableStructure` — sidebar context menu → coordinator
- [ ] `editViewDefinition` — sidebar context menu → coordinator
- [ ] `createView` — sidebar context menu → coordinator
- [ ] `exportTables` — sidebar context menu → coordinator
- [ ] `importTables` — sidebar context menu → coordinator

### Pattern:

```swift
// Define focused value
struct SidebarActionsKey: FocusedValueKey {
    typealias Value = SidebarViewModel
}

extension FocusedValues {
    var sidebarActions: SidebarViewModel? { ... }
}

// In SidebarView
.focusedValue(\.sidebarActions, viewModel)

// In menu
Button("Copy Table Names") {
    focusedSidebarActions?.copySelectedTableNames()
}
```

---

## Phase 5: Replace Structure View Notifications with Coordinator Pattern

**Status:** Not started

`MainContentCommandActions` routes commands to `TableStructureView` via notifications because the structure view is deeply embedded and not directly accessible.

### Notifications to replace:

- [ ] `copySelectedRows` (structure path)
- [ ] `pasteRows` (structure path)
- [ ] `undoChange` (structure path)
- [ ] `redoChange` (structure path)
- [ ] `saveStructureChanges`
- [ ] `previewStructureSQL`

### Strategy:

Create a `StructureViewActions` protocol/class that `TableStructureView` registers with the coordinator. The coordinator calls methods directly instead of broadcasting.

---

## Phase 6: Replace Editor/AI Notifications with Direct References

**Status:** Not started

### Editor notifications:

- [ ] `loadQueryIntoEditor` — posted by `HistoryPanelView`, `QuickSwitcher+`. Coordinator should have a method `loadQueryIntoEditor(_:)` called directly.
- [ ] `insertQueryFromAI` — posted by `AIChatCodeBlockView`. AI panel needs a callback/delegate to the coordinator.
- [ ] `newQueryTab` — posted by `HistoryPanelView`. Direct coordinator call.
- [ ] `explainQuery` — posted by `QueryEditorView`. Direct coordinator call.

### AI notifications:

- [ ] `aiFixError` — posted by coordinator, received by `AIChatPanelView`. Could use a shared `AIChatViewModel` reference or `@FocusedValue`.
- [ ] `aiExplainSelection` — posted by editor context menu, received by AI panel. Same approach.
- [ ] `aiOptimizeSelection` — same.

---

## Phase 7: Replace Window Lifecycle Notifications

**Status:** Not started

### Keep (AppKit → SwiftUI bridge, no alternative):

- `openMainWindow` — `AppDelegate` → SwiftUI `openWindow`
- `openWelcomeWindow` — same
- `mainWindowWillClose` — `NSWindowDelegate` → tab persistence

### Replace:

- [ ] `lastWindowDidClose` — `WindowLifecycleMonitor` → `DatabaseManager`. Use a direct callback/delegate.
- [ ] `sshTunnelDied` — `SSHTunnelManager` → `DatabaseManager`. Use a closure callback set at tunnel creation.
- [ ] `connectionUpdated` — `ConnectionFormView` → `WelcomeWindowView`. Use `@Observable ConnectionStorage`.
- [ ] `newConnection` — menu → welcome/content view. Use `@FocusedValue` or `@Environment(\.openWindow)`.

---

## Phase 8: Replace Deep-Link Notifications

**Status:** Not started

- [ ] `openSQLFiles` — `AppDelegate` → `MainContentCommandActions`. Keep notification (legitimate AppKit → SwiftUI bridge).
- [ ] `switchSchemaFromURL` — `AppDelegate` → coordinator. Keep or use a coordinator lookup by connectionId.
- [ ] `applyURLFilter` — `AppDelegate` → coordinator. Same.

---

## Priority Order

1. **Phase 1** (dead code removal) — zero risk, immediate cleanup
2. **Phase 3** (data refresh scoping) — fixes the actual sidebar bug, biggest architectural win
3. **Phase 4** (sidebar actions via @FocusedValue) — clean menu routing
4. **Phase 5** (structure view) — removes the most confusing notification routing
5. **Phase 6** (editor/AI) — cleaner inter-panel communication
6. **Phase 2** (settings) — partial, only SwiftUI consumers
7. **Phase 7** (window lifecycle) — lower priority, partially legitimate
8. **Phase 8** (deep-link) — mostly keep as-is

## Metrics

| Metric                                       | Before | Current | Target                                           |
| -------------------------------------------- | ------ | ------- | ------------------------------------------------ |
| Custom notification names                    | 62     | ~33     | ~15 (AppKit bridges + settings for AppKit views) |
| Dead notifications                           | 11     | 0       | 0                                                |
| Global broadcasts without connection scoping | 3      | 1       | 0                                                |
| `userInfo` dictionary payloads               | ~8     | ~7      | 0 (typed APIs)                                   |
