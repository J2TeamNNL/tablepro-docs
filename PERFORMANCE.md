# TablePro Performance Audit

> Generated: 2026-02-25 | Updated: 2026-02-25 | Total issues: 60 | Fixed: 54 | Resolved: 5 | Deferred: 1

## Priority Matrix

| Priority | Count | Description |
|----------|-------|-------------|
| Critical | 3 | Data loss risk or unbounded memory growth |
| High | 11 | Measurable user-visible lag or O(n^2) paths |
| Medium | 26 | Wasteful but bounded; fix for polish |
| Low | 17 | Minor inefficiency; fix opportunistically |

---

## 1. Database Drivers

### DB-1 — CRITICAL: `mysql_store_result` buffers entire result set before row cap

- **File**: `Core/Database/MariaDBConnection.swift:526`
- **Category**: Memory
- **Status**: Fixed

`mysql_store_result` transfers the **entire** server result into client C-heap before Swift's 100k-row cap takes effect. A query returning 5M rows allocates gigabytes before the cap kicks in.

**Fix**: Switch to `mysql_use_result` (streaming mode). The `MariaDBStreamingResult` infrastructure already exists (line 1051) but is never used from `execute()`.

---

### DB-2 — HIGH: `PQexec` buffers entire result set before row cap

- **File**: `Core/Database/LibPQConnection.swift:367-369`
- **Category**: Memory
- **Status**: Deferred — requires PQsendQuery API rewrite (tracked as TODO in code)

Same issue as DB-1 for PostgreSQL. `PQexec` is synchronous, store-everything. The cap `min(numRows, maxRows)` only limits Swift array copying.

**Fix**: Use `PQsendQuery` + `PQsetSingleRowMode` for row-at-a-time streaming.

---

### DB-3 — HIGH: `[UInt8]` intermediate buffer doubles memory per field

- **File**: `Core/Database/MariaDBConnection.swift:624-633`, `Core/Database/LibPQConnection.swift:556-571`
- **Category**: Memory / CPU
- **Status**: Fixed

Both drivers create `[UInt8]` via `memcpy`, then create `String` from it — 2x memory per field value.

**Fix**: Initialize `String` directly from C pointer: `String(data: Data(bytes: fieldPtr, count: length), encoding: .utf8)`.

---

### DB-4 — HIGH: SQLite `fetchTableMetadata` does full `COUNT(*)` for every table

- **File**: `Core/Database/SQLiteDriver.swift:694-698`
- **Category**: CPU / I/O
- **Status**: Fixed

No fast approximate count in SQLite. Full sequential scan on every metadata panel open.

**Fix**: Use `sqlite_stat1` (if `ANALYZE` has been run) or skip `COUNT(*)` and return nil.

---

### DB-5 — HIGH: `fetchDatabaseMetadata` fires 2 sequential queries (MySQL + PostgreSQL)

- **File**: `Core/Database/MySQLDriver.swift:642-657`, `Core/Database/PostgreSQLDriver.swift:744-756`
- **Category**: Network
- **Status**: Fixed

Both queries hit the same system table with the same WHERE. Easily merged:
```sql
SELECT COUNT(*), SUM(DATA_LENGTH + INDEX_LENGTH)
FROM information_schema.TABLES WHERE TABLE_SCHEMA = '...'
```

---

### DB-6 — HIGH: PostgreSQL `fetchTableDDL` fires 4 sequential queries

- **File**: `Core/Database/PostgreSQLDriver.swift:554-622`
- **Category**: Network
- **Status**: Fixed

Columns, constraints, indexes — each a separate round-trip. Noticeable on high-latency connections.

**Fix**: Consolidate into multi-statement query or use `pg_get_tabledef` (PG13+).

---

### DB-7 — HIGH: `cancelCurrentQuery` opens a brand-new TCP connection

- **File**: `Core/Database/MariaDBConnection.swift:415-451`
- **Category**: Network / CPU
- **Status**: Fixed

Every cancel creates a full TCP + MySQL handshake synchronously. Blocks calling thread for seconds on slow networks.

**Fix**: Use `mysql_kill()` on existing handle, or maintain a persistent control connection.

---

### DB-8 — MEDIUM: Health ping uses main driver, blocks user queries

- **File**: `Core/Database/DatabaseManager.swift:363-373`
- **Category**: Network
- **Status**: Fixed

Ping fires `SELECT 1` on the same driver used for user queries. A 25-second query blocks the 30-second ping, triggering false reconnect.

**Fix**: Use `activeMetadataDriver` for health pings.

---

### DB-9 — MEDIUM: Empty SELECT triggers extra `DESCRIBE` round-trip

- **File**: `Core/Database/MySQLDriver.swift:156-169`
- **Category**: Network
- **Status**: Fixed

Every zero-row SELECT fires `fetchColumnNames` (regex + DESCRIBE). Column metadata should already be available from `mysql_fetch_fields`.

---

### DB-10 — MEDIUM: `SQLiteDriver.stripLimitOffset` compiles regex on every call

- **File**: `Core/Database/SQLiteDriver.swift:718-733`
- **Category**: CPU
- **Status**: Fixed

MySQL/PostgreSQL drivers cache regex as `static let`. SQLite recompiles both patterns every pagination call.

**Fix**: Add `private static let limitRegex` / `offsetRegex` matching the other drivers.

---

### DB-11 — MEDIUM: `SQLiteDriver.fetchIndexes` N+1 PRAGMA queries

- **File**: `Core/Database/SQLiteDriver.swift:443-476`
- **Category**: CPU / I/O
- **Status**: Fixed

1 query for index list + 1 per index. 20 indexes = 21 queries through actor serialization.

**Fix**: Use `pragma_index_list` + `pragma_index_info` join in a single query.

---

### DB-12 — MEDIUM: SSH tunnel setup duplicated in connect + reconnect

- **File**: `Core/Database/DatabaseManager.swift:89-130 vs 466-500`
- **Category**: Maintainability
- **Status**: Fixed

Copy-pasted SSH/Keychain logic. Any fix to one must be manually mirrored.

**Fix**: Extract `buildEffectiveConnection(for:) async throws -> DatabaseConnection`.

---

### DB-13 — MEDIUM: PostgreSQL connection string injection with special characters

- **File**: `Core/Database/LibPQConnection.swift:187-220`
- **Category**: Security / Correctness
- **Status**: Fixed

Passwords with single quotes break the naively concatenated connection string.

**Fix**: Use `PQconnectdbParams` (keyword/value arrays) instead of `PQconnectdb`.

---

### DB-14 — LOW: Redundant `String(query)` copy in `executeQuerySync`

- **File**: `Core/Database/MariaDBConnection.swift:515-516`
- **Category**: Memory
- **Status**: Fixed

`let localQuery = String(query)` copies an already-owned String parameter. Same in `executeQueryStreaming` (line 947).

---

### DB-15 — LOW: `strdup` freed via Swift `.deallocate()` (formally UB)

- **File**: `Core/Database/LibPQConnection.swift:416-422`
- **Category**: Memory
- **Status**: Fixed

`strdup` uses C `malloc`; should be freed with `free()`, not Swift `.deallocate()`.

---

### DB-16 — LOW: Default `fetchAllColumns` N+1 (hits SQLiteDriver)

- **File**: `Core/Database/DatabaseDriver.swift:144-158`
- **Category**: CPU
- **Status**: Fixed

SQLite inherits the default N+1 implementation. 50 tables = 51 sequential queries.

**Fix**: Override in `SQLiteDriver` with `pragma_table_info` + `sqlite_master` join.

---

## 2. UI / Rendering

### UI-1 — HIGH: `QueryTab.==` is identity-only, causes full SwiftUI diffs

- **File**: `Models/QueryTab.swift:453-455`
- **Category**: Rendering
- **Status**: Fixed

`lhs.id == rhs.id` means every `tabs` array write triggers full re-render of `ForEach(tabs)`. 3-5 mutations per query execution.

**Fix**: Implement value-based equality comparing UI-driving fields (title, isExecuting, errorMessage, resultVersion, metadataVersion, pagination, sortState, isPinned).

---

### UI-2 — HIGH: `extractTableName` compiles NSRegularExpression on every query

- **File**: `Views/Main/MainContentCoordinator.swift:726-734`
- **Category**: CPU
- **Status**: Fixed

Called twice per query execution. Regex compilation is expensive.

**Fix**: Cache as `private static let tableNameRegex`.

---

### UI-3 — MEDIUM: `ConnectionToolbarState.isExecuting.didSet` double-fires @Published

- **File**: `Models/ConnectionToolbarState.swift:138-147`
- **Category**: Rendering
- **Status**: Fixed

Setting `isExecuting` fires `objectWillChange` twice (once for itself, once for `connectionState` in didSet).

**Fix**: Batch into a single `setExecuting(_:)` method.

---

### UI-4 — MEDIUM: Three `onChange` handlers each rebuild `InMemoryRowProvider`

- **File**: `Views/Main/Child/MainEditorContentView.swift:132-157`
- **Category**: Rendering
- **Status**: Fixed

`resultVersion`, `metadataVersion`, `selectedTabId` changes each trigger separate `makeRowProvider`.

**Fix**: Coalesce into a single `RowProviderTrigger: Equatable` struct.

---

### UI-5 — MEDIUM: `currentChangeManager` creates Combine pipelines during body evaluation

- **File**: `Views/Main/Child/MainEditorContentView.swift:74-79`
- **Category**: Memory
- **Status**: Fixed

If `cachedChangeManager` is nil during body, `AnyChangeManager` with Combine subscriptions is created and immediately abandoned.

---

### UI-6 — MEDIUM: Column layout sync iterates all columns per `updateNSView`

- **File**: `Views/Results/DataGridView.swift:372-396`
- **Category**: CPU
- **Status**: Fixed

O(n_columns) loop runs on every `updateNSView`. With 30 columns on cursor-move frequency, this adds up.

**Fix**: Gate behind `coordinator.hasUserResizedColumns` flag.

---

### UI-7 — MEDIUM: Async column width write-back causes two-frame bounce

- **File**: `Views/Results/DataGridView.swift:361-364, 387-394`
- **Category**: Rendering
- **Status**: Fixed

`DispatchQueue.main.async { columnLayout.columnWidths = ... }` from inside `updateNSView` triggers a second SwiftUI render that re-enters the column sync loop.

**Fix**: Track `isWritingColumnLayout` flag to skip re-entry.

---

### UI-8 — MEDIUM: `calculateOptimalColumnWidth` runs CoreText measurement on main thread

- **File**: `Views/Results/DataGridCellFactory.swift:396-428`
- **Category**: CPU
- **Status**: Fixed

Up to 2,000 `NSString.size(withAttributes:)` calls (1k rows × 20 cols) on initial load.

**Fix**: Move to background `Task.detached`, apply widths back on MainActor.

---

### UI-9 — MEDIUM: Phase 2a/2b produce separate render cycles

- **File**: `Views/Main/MainContentCoordinator.swift:550-595`
- **Category**: Rendering
- **Status**: Resolved — already optimized by Phase 1 approximate count merge

Exact COUNT and enum values each write to `tabs[idx]` separately, causing 2 extra SwiftUI passes after Phase 1.

**Fix**: Coalesce Phase 2a and 2b into a single `MainActor.run`.

---

### UI-10 — MEDIUM: `flushPendingSave` synchronous on every tab switch

- **File**: `Views/Main/Extensions/MainContentCoordinator+TabSwitch.swift:43`
- **Category**: CPU / I/O
- **Status**: Fixed

JSON-encoding all tabs on MainActor during rapid keyboard tab switching.

**Fix**: Debounce flush (<50ms since last save → skip).

---

### UI-11 — MEDIUM: `TableProTabSmart` 14 individual field writes trigger 14 CoW copies

- **File**: `Models/QueryTab.swift:588-606`
- **Category**: Memory
- **Status**: Fixed

Each `tabs[selectedIndex].X = Y` triggers a CoW copy of the entire tabs array.

**Fix**: Build local `var tab`, mutate all fields, write back once.

---

### UI-12 — MEDIUM: `NumberFormatter` allocated per status bar render

- **File**: `Views/Main/Child/MainStatusBarView.swift:114-124`
- **Category**: Memory / CPU
- **Status**: Fixed

`NumberFormatter()` involves locale resolution. Called on every body evaluation.

**Fix**: Use `private static let decimalFormatter`.

---

### UI-13 — MEDIUM: Status bar re-renders on every selection change via `QueryTab` identity equality

- **File**: `Views/Main/Child/MainStatusBarView.swift:12-13`
- **Category**: Rendering
- **Status**: Resolved — addressed by UI-1 value-based equality fix

`tab: QueryTab?` argument uses identity-only `==`, so any tab mutation forces re-render.

**Fix**: Fix UI-1 first. Then extract `rowInfoText` to a dedicated view with explicit Equatable inputs.

---

### UI-14 — MEDIUM: `EditorTabBar` re-renders entire tab list on tab switch

- **File**: `Views/Editor/EditorTabBar.swift:25-40`
- **Category**: Rendering
- **Status**: Resolved — addressed by UI-1 value-based equality fix

`isSelected` changes for two tabs but `ForEach` re-evaluates all items due to identity-only equality.

**Fix**: Depends on UI-1 (proper `QueryTab` equality).

---

### UI-15 — LOW: `trimmingCharacters` on full query per render

- **File**: `Views/Editor/QueryEditorView.swift:94`
- **Category**: CPU
- **Status**: Fixed

`.disabled(queryText.trimmingCharacters(in: .whitespacesAndNewlines).isEmpty)` walks the whole string.

**Fix**: Use `appState.hasQueryText` which is already maintained.

---

### UI-16 — LOW: Settings read before identity early-return in `updateNSView`

- **File**: `Views/Results/DataGridView.swift:171-177`
- **Category**: CPU
- **Status**: Fixed

Move identity check above settings read.

---

### UI-17 — LOW: `cleanupSortCache` fires @Published unconditionally

- **File**: `Views/Main/MainContentCoordinator.swift:96-103`
- **Category**: Rendering
- **Status**: Fixed

Writes to `querySortCache` even when filter is a no-op.

**Fix**: Only write if count differs.

---

### UI-18 — LOW: `frameDidChangeNotification` posted on every keystroke

- **File**: `Views/Editor/SQLEditorCoordinator.swift:50-67`
- **Category**: CPU
- **Status**: Fixed

---

### UI-19 — LOW: Full-string `==` check before text replacement in `SQLEditorView`

- **File**: `Views/Editor/SQLEditorView.swift:56-62`
- **Category**: CPU
- **Status**: Fixed

**Fix**: Use NSString length comparison as fast pre-check.

---

## 3. Services & Storage

### SVC-1 — CRITICAL: `SQLFileParser` uses Swift String.Index in inner loop — O(n^2)

- **File**: `Core/Utilities/SQLFileParser.swift:83-204`
- **Category**: CPU
- **Status**: Fixed

`buffer.index(after: index)` and `buffer[index]` on bridged NSStrings = O(n) per character access. 500MB SQL file = O(n^2). This is the exact pattern flagged in CLAUDE.md as a critical pitfall.

**Fix**: Convert to `NSString` + `character(at:)` for O(1) access.

---

### SVC-2 — HIGH: XLSX export accumulates entire sheet XML in memory

- **File**: `Core/Services/XLSXWriter.swift:77-99`
- **Category**: Memory
- **Status**: Fixed

500k rows × 20 columns = 200-500 MB of XML in RAM. Peak memory is 2x (sheet data + ZIP output).

**Fix**: Stream worksheet XML to temp files, assemble ZIP from files.

---

### SVC-3 — HIGH: Import records history entry for every SQL statement

- **File**: `Core/Services/ImportService.swift:177-203`
- **Category**: I/O / Memory
- **Status**: Fixed

50k-row dump = 50k history entries, 500 cleanup checks, 50k notification callbacks.

**Fix**: Add `isImportBatch` flag to skip per-statement history. Post notification once at end.

---

### SVC-4 — HIGH: Tab persistence writes on main thread

- **File**: `Core/Services/TabPersistenceService.swift:82-104`
- **Category**: I/O
- **Status**: Fixed

`saveTabsImmediately` and `handleWindowClose` do JSON encoding + atomic file write on MainActor.

**Fix**: Use `Task.detached(priority: .utility)` for the actual write.

---

### SVC-5 — MEDIUM: Import double-parses file (count + execute)

- **File**: `Core/Services/ImportService.swift:102-160`
- **Category**: I/O / CPU
- **Status**: Fixed

500MB dump parsed twice. 10-30 seconds of extra time.

**Fix**: Use indeterminate progress, or cache statements from first parse.

---

### SVC-6 — MEDIUM: `saveLastQueryDebounced` writes on main actor

- **File**: `Core/Services/TabPersistenceService.swift:257-268`
- **Category**: I/O
- **Status**: Fixed

After debounce delay, file I/O still runs on MainActor.

**Fix**: Hop to background thread after debounce guard passes.

---

### SVC-7 — MEDIUM: No WAL mode on SQLite history DB

- **File**: `Core/Storage/QueryHistoryStorage.swift:86-92`
- **Category**: I/O
- **Status**: Fixed

Default journal mode (DELETE) = exclusive lock + fsync per insert. Bottleneck during import.

**Fix**: `PRAGMA journal_mode=WAL; PRAGMA synchronous=NORMAL;` after `sqlite3_open`.

---

### SVC-8 — MEDIUM: History cleanup: COUNT + DELETE without transaction

- **File**: `Core/Storage/QueryHistoryStorage.swift:462-486`
- **Category**: I/O
- **Status**: Fixed

Two statements with separate implicit transactions and journal flushes.

**Fix**: Wrap in `BEGIN IMMEDIATE; ... COMMIT;`.

---

### SVC-9 — MEDIUM: FTS5 MATCH input not sanitized

- **File**: `Core/Storage/QueryHistoryStorage.swift:275-279`
- **Category**: CPU / Correctness
- **Status**: Fixed

`select *` causes FTS5 parse error — silent empty results.

**Fix**: Escape FTS5 special characters or wrap in double quotes.

---

### SVC-10 — MEDIUM: JSON export: tiny individual `FileHandle.write` calls per field

- **File**: `Core/Services/ExportService.swift:614-642`
- **Category**: I/O / CPU
- **Status**: Fixed

50 cols × 100k rows × 5 writes = 25M syscalls.

**Fix**: Buffer entire row then write once (matching CSV path).

---

### SVC-11 — MEDIUM: `SQLFormatterService.uppercaseKeywords` mutates string in-place per match

- **File**: `Core/Services/SQLFormatterService.swift:332-343`
- **Category**: CPU / Memory
- **Status**: Fixed

Each `replaceSubrange` can trigger a String copy. Hundreds of mutations on large SQL.

**Fix**: Use `NSRegularExpression.stringByReplacingMatches` or `NSMutableString`.

---

### SVC-12 — MEDIUM: `rebuildChangeIndex()` is O(n) after every single deletion

- **File**: `Core/ChangeTracking/DataChangeManager.swift:56-61`
- **Category**: CPU
- **Status**: Fixed

1,000 row deletions = O(n^2) total work.

**Fix**: Maintain `changeIndex` incrementally instead of full rebuild.

---

### SVC-13 — MEDIUM: `SQLContextAnalyzer.removeStringsAndComments` — 4 regex passes

- **File**: `Core/Autocomplete/SQLContextAnalyzer.swift:955-983`
- **Category**: CPU
- **Status**: Fixed

4 sequential `stringByReplacingMatches` passes on every autocomplete trigger.

**Fix**: Combine into single alternation regex.

---

### SVC-14 — MEDIUM: `SQLContextAnalyzer.isInsideComment` — two O(n) loops

- **File**: `Core/Autocomplete/SQLContextAnalyzer.swift:714-735`
- **Category**: CPU
- **Status**: Fixed

Counts `/*` and `*/` in separate loops. Called per autocomplete event.

**Fix**: Single-pass state machine.

---

### SVC-15 — MEDIUM: Synchronous Keychain reads may block main thread

- **File**: `Core/Storage/ConnectionStorage.swift:173-332`
- **Category**: I/O
- **Status**: Resolved — callers already off main thread

`SecItemCopyMatching` can block 100-500ms when Keychain is locked or Secure Enclave is involved.

**Fix**: Wrap in `Task.detached(priority: .userInitiated)`.

---

### SVC-16 — LOW: `escapeJSONString` uses Swift grapheme-cluster iteration

- **File**: `Core/Services/ExportService.swift:679`
- **Category**: CPU
- **Status**: Fixed

**Fix**: Iterate `string.utf8` bytes (all JSON escapes are single-byte ASCII).

---

### SVC-17 — LOW: `isSQLFunctionExpression` rebuilds `[String]` array per call

- **File**: `Core/ChangeTracking/SQLStatementGenerator.swift:337-358`
- **Category**: Memory / CPU
- **Status**: Fixed

**Fix**: Use `static let sqlFunctionExpressions: Set<String>`.

---

### SVC-18 — LOW: UUID generated per string literal placeholder in formatter

- **File**: `Core/Services/SQLFormatterService.swift:257`
- **Category**: CPU
- **Status**: Fixed

**Fix**: Use incrementing integer counter instead of UUID.

---

### SVC-19 — LOW: `recordRowDeletion` uses linear `removeAll` when changeIndex allows O(1)

- **File**: `Core/ChangeTracking/DataChangeManager.swift:254`
- **Category**: CPU
- **Status**: Fixed

---

### SVC-20 — LOW: `AppSettingsStorage` creates new JSONDecoder/Encoder per read/write

- **File**: `Core/Storage/AppSettingsStorage.swift:165-183`
- **Category**: Memory / CPU
- **Status**: Fixed

**Fix**: Cache as private stored properties.

---

### SVC-21 — LOW: `AppSettingsManager.dataGrid` didSet saves validated but stores original

- **File**: `Core/Storage/AppSettingsManager.swift:48-57`
- **Category**: Correctness
- **Status**: Fixed

In-memory value differs from persisted value after validation.

---

### SVC-22 — LOW: `FilterSettingsStorage.clearAllLastFilters` loads full plist

- **File**: `Core/Storage/FilterSettingsStorage.swift:165`
- **Category**: Memory / I/O
- **Status**: Fixed

---

### SVC-23 — LOW: `SecItemDelete` return code ignored in `savePassword`

- **File**: `Core/Storage/ConnectionStorage.swift:148-169`
- **Category**: Correctness
- **Status**: Fixed

**Fix**: Use `SecItemUpdate` for existing items.

---

### SVC-24 — LOW: Character-by-character string building in `detectFunctionContext`

- **File**: `Core/Autocomplete/SQLContextAnalyzer.swift:612`
- **Category**: CPU / Memory
- **Status**: Fixed

**Fix**: Track `wordStart`/`wordEnd` indices, extract once via `NSString.substring(with:)`.

---

### SVC-25 — LOW: TabStateStorage migration runs synchronously at first launch

- **File**: `Core/Storage/TabStateStorage.swift:193-249`
- **Category**: I/O
- **Status**: Resolved — one-time millisecond-scale migration, acceptable

---

## Recommended Fix Order

### Phase 1 — Critical + Quick Wins (1-2 days)

| ID | Issue | Effort |
|----|-------|--------|
| SVC-1 | SQLFileParser O(n^2) string indexing | Medium |
| UI-1 | QueryTab value-based equality | Low |
| UI-2 | Cache extractTableName regex | Low |
| DB-10 | Cache SQLiteDriver regex | Low |
| UI-12 | Static NumberFormatter | Low |
| UI-11 | Batch TableProTabSmart field writes | Low |
| SVC-17 | Static Set for SQL function expressions | Low |
| UI-15 | Use appState.hasQueryText | Low |

### Phase 2 — High-Impact Fixes (3-5 days)

| ID | Issue | Effort |
|----|-------|--------|
| DB-1 | MySQL streaming results | High |
| DB-2 | PostgreSQL streaming results | High |
| DB-3 | Eliminate intermediate byte buffer | Low |
| SVC-4 | Tab persistence off main thread | Low |
| SVC-3 | Skip history during import | Low |
| SVC-7 | WAL mode for history DB | Low |
| DB-5 | Merge fetchDatabaseMetadata queries | Low |
| DB-7 | Fix cancel query connection | Medium |
| UI-4 | Coalesce onChange handlers | Medium |
| UI-8 | Background column width calculation | Medium |

### Phase 3 — Polish (1 week)

| ID | Issue | Effort |
|----|-------|--------|
| SVC-2 | Streaming XLSX export | High |
| DB-6 | Consolidate PostgreSQL DDL queries | Medium |
| DB-8 | Health ping on metadata driver | Low |
| UI-3 | Batch toolbar state | Low |
| UI-6 | Gate column sync behind user resize | Low |
| UI-9 | Coalesce Phase 2a/2b renders | Medium |
| SVC-5 | Eliminate double parse in import | Medium |
| SVC-10 | Buffer JSON export writes | Low |
| SVC-12 | Incremental change index | Medium |
| SVC-13 | Single-pass string/comment removal | Low |

---

## Already Fixed (Previous Audits)

The following were identified and fixed in previous performance audits:

- RowBuffer reference wrapper for QueryTab (MEM-1/2)
- Index-based sort cache (MEM-3)
- Streaming XLSX with inline strings (MEM-4/15) — partial, SVC-2 remains
- Driver-level row limits cap at 100K (MEM-5)
- Weak driver reference in SQLSchemaProvider (MEM-9)
- Undo stack depth cap (MEM-10)
- Dictionary-based tab pending changes (MEM-11)
- Removed unicodeScalars.map in drivers (CPU-1/2)
- Cached 100+ regex in SQLFormatterService (CPU-3/5/8/9/10)
- O(1) change lookup index (CPU-11)
- Batch fetchAllColumns via INFORMATION_SCHEMA (DAT-4)
- Phase 2 metadata cache check (NET-1)
- connect_timeout for LibPQ (NET-2)
- Driver-level cancelQuery (NET-3)
- Throttled history cleanup (IO-1)
- `(string as NSString).length` replacing `string.count` across codebase
- `allowsNonContiguousLayout = true` on NSLayoutManager
- Viewport-only syntax highlighting with scroll-based lazy expansion
- Debounced text binding (100ms) in EditorCoordinator
- TabSnapshot Equatable caching in NativeTabBar
- Approximate row count for instant pagination (DB drivers + Phase 1 merge)
