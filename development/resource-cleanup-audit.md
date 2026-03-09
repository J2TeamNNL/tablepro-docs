# Resource Cleanup & Performance Audit

Comprehensive audit of resource leaks, connection lifecycle edge cases, and performance issues related to [#214](https://github.com/datlechin/TablePro/issues/214).

## Issues to Fix

### 1. URL filter notification observers never removed
**Severity:** Minor | **File:** `Views/Main/Extensions/MainContentCoordinator+URLFilter.swift`

`setupURLNotificationObservers()` (called from `init`) registers two block-based `NotificationCenter` observers but never stores the returned `NSObjectProtocol` tokens. They are never removed in `teardown()` or `deinit`. Each window open/close accumulates 2 orphaned observers.

Uses `[weak self]` so no retain cycle, but dead observers still fire and filter by `connectionId` — wasted work.

**Fix:** Store observer tokens and remove in `teardown()`.

### 2. Untracked metadata driver creation Tasks
**Severity:** Important | **File:** `Core/Database/DatabaseManager.swift:207, 677`

`connectToSession()` and `reconnectSession()` spawn fire-and-forget `Task {}` blocks to create metadata drivers. These tasks are never stored or cancelled. If `disconnectSession()` runs before the task completes, the task may assign a driver to a session that no longer exists (safe due to optional chaining) or — if a new session reuses the same UUID — to the wrong session.

**Fix:** Store metadata creation tasks and cancel them in `disconnectSession()`.

### 3. Misleading health monitor log message
**Severity:** Minor | **File:** `Core/Database/DatabaseManager.swift:542`

Log says "after 3 retries" but `ConnectionHealthMonitor` retries indefinitely with exponential backoff (capped at 120s). The "3" is incorrect.

**Fix:** Update log message.

### 4. Database connections not cleaned up on app termination
**Severity:** Important | **File:** `AppDelegate.swift:806`

`applicationWillTerminate` kills SSH tunnels via `terminateAllProcessesSync()` but does not disconnect database sessions. Health monitors, ping drivers, and database connections survive until the process exits. While the OS reclaims sockets, this skips graceful disconnect (e.g., PostgreSQL `PQfinish`, MySQL `mysql_close`).

**Fix:** Call `DatabaseManager.shared.disconnectAllSync()` or iterate sessions synchronously.

### 5. `localPortCandidates()` allocates 5001-element shuffled array
**Severity:** Minor | **File:** `Core/SSH/SSHTunnelManager.swift:249`

`Array(60000...65000).shuffled()` creates a ~40KB array per tunnel creation. A random start with wrap-around achieves the same anti-collision goal at O(1).

**Fix:** Use random offset iteration instead of full array shuffle.

## Completed Fixes (PR #217)

- [x] SSH tunnel not closed on window close — deterministic disconnect via `WindowLifecycleMonitor` + `.lastWindowDidClose` notification
- [x] SSH tunnel not closed on app termination — `terminateAllProcessesSync()` in `applicationWillTerminate`
- [x] `waitForProcessExit` hang on already-exited process — guard `isRunning`, timeout via `TaskGroup`, SIGKILL fallback
- [x] `closeAllTunnels()` not waiting for process exit — concurrent `TaskGroup` with timeout

## No Action Needed (Verified Correct)

- **NotificationCenter observers** in DatabaseManager, AppDelegate, WindowLifecycleMonitor, MainContentCoordinator (termination) — all properly stored and removed
- **NSEvent monitors** in VimKeyInterceptor, InlineSuggestionManager, SQLEditorCoordinator — dual cleanup (deinit + explicit uninstall)
- **Async Tasks** in MainContentCoordinator — all stored and cancelled in `teardown()`
- **ConnectionHealthMonitor** — `stopMonitoring()` cancels task and finishes continuation
- **Ping drivers** — disconnected in `stopHealthMonitor()`
- **SchemaProviderRegistry** — cleared in `disconnectSession()`
- **File handles** in ExportService, FileDecompressor — closed via `defer` blocks
- **Combine subscriptions** in SidebarViewModel — deallocated with ViewModel
- **URLSession instances** — all singletons with app lifetime, acceptable
