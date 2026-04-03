# TablePro Mobile — Release Roadmap

## Current Status: Pre-Release (33% ready)

Core database functionality complete. Missing App Store requirements, polish, and testing.

---

## Phase 1: Critical Fixes (Must do before ANY release)

**Estimated: 1 day**

### Blocking Bugs
- [ ] Fix iOS deployment target (26.4 → 17.0)
- [ ] Remove debug `print()` statements → replace with `os.Logger`
- [ ] Fix PostgreSQL SSL mode — make dynamic based on `sslEnabled` flag (currently hardcoded `disable`)
- [ ] Fix Keychain access group — remove hardcoded Team ID, use `$(AppIdentifierPrefix)`

### App Store Requirements
- [ ] Design app icon (all sizes for iPhone + iPad)
- [ ] Create launch screen (storyboard or SwiftUI)
- [ ] Write privacy policy (required for iCloud/CloudKit)
- [ ] Configure App Store metadata (description, keywords, category)
- [ ] Prepare screenshots (iPhone 6.7", 6.5", iPad 12.9")
- [ ] Set correct bundle ID + signing for distribution

---

## Phase 2: Essential Polish (Required for quality 1.0)

**Estimated: 3-4 days**

### Missing Screens
- [ ] Settings screen — pagination size, default database type, clear data, sync toggle
- [ ] About screen — version, links (website, privacy policy, support), credits
- [ ] Onboarding — first-launch welcome with "Sync from iCloud" or "Add Connection" guidance

### Error Handling
- [ ] Improve connection error messages — distinguish auth failure, timeout, network unreachable, SSH failure
- [ ] Add recovery suggestions ("Check your password", "Verify SSH host", "Enable VPN")
- [ ] Show toast/banner for transient errors instead of replacing entire screen

### SSL Support
- [ ] MySQL SSL — pass SSL config to `mysql_options()` (MYSQL_OPT_SSL_CA/CERT/KEY)
- [ ] PostgreSQL SSL — map `SSLConfiguration.mode` to `sslmode=` in connection string
- [ ] Redis SSL — use `redisCreateSSLContextWithOptions()` + `redisInitiateSSLWithContext()`

### Query Editor
- [ ] Persist query history across sessions (save to file, not just @State)
- [ ] Syntax highlighting — color SQL keywords (SELECT, FROM, WHERE) with regex or tree-sitter-highlight
- [ ] Multiple queries support — split by `;` and execute sequentially

### Data Display
- [ ] Table structure view — show columns, types, primary keys, indexes, foreign keys
- [ ] Database switching — picker UI to switch between databases
- [ ] Schema switching — for PostgreSQL, picker to switch schemas
- [ ] Row count badge on tables — show approximate count in table list

---

## Phase 3: User-Expected Features (Quality of life)

**Estimated: 1 week**

### Export
- [ ] Export query results as CSV
- [ ] Export query results as JSON
- [ ] Copy row as INSERT SQL statement
- [ ] Share sheet integration (AirDrop, Files, etc.)

### Advanced Query
- [ ] Multiple query tabs (tab bar above editor)
- [ ] Saved queries / favorites (persist to file)
- [ ] Query templates per database type

### Data Browsing
- [ ] Sort by column tap (UI-based, not SQL only)
- [ ] Filter bar on data browser (simple column + operator + value)
- [ ] Column visibility toggle (hide/show columns)

### Connections
- [ ] Duplicate connection
- [ ] Connection groups (folders, synced from macOS)
- [ ] Connection color tags
- [ ] Connection status indicators (connected/disconnected dot)
- [ ] Prompt for password on connect if Keychain empty (instead of silent failure)

---

## Phase 4: Testing & Accessibility

**Estimated: 3-4 days**

### Testing
- [ ] Unit tests for SQLBuilder (SQL injection edge cases)
- [ ] Unit tests for DatabaseType normalization
- [ ] Unit tests for SyncRecordMapper (field mapping)
- [ ] Unit tests for SSHConfiguration decoder (macOS compatibility)
- [ ] Integration test: SQLite CRUD cycle
- [ ] Integration test: CloudKit sync mock

### Accessibility
- [ ] VoiceOver labels on all interactive elements
- [ ] Dynamic Type support verification
- [ ] Reduced Motion support
- [ ] Minimum touch target 44pt verification

### Localization
- [ ] Extract all user-facing strings to Localizable.strings
- [ ] Vietnamese localization (primary market)
- [ ] English localization (default)

---

## Phase 5: Advanced Features (Post-1.0)

**Not required for initial release**

- [ ] Redis key browser (tree view with namespaces)
- [ ] MongoDB support (cross-compile libmongoc)
- [ ] MSSQL support (cross-compile FreeTDS)
- [ ] ClickHouse support (HTTP API, no C lib needed)
- [ ] SSH jump hosts
- [ ] SSH keyboard-interactive + TOTP
- [ ] iCloud Drive database file sync (SQLite files)
- [ ] Shortcuts/Siri integration ("Run query on production")
- [ ] Widget (connection status, last query time)
- [ ] Apple Watch companion (connection status)
- [ ] Push notifications via CloudKit subscriptions (real-time sync)

---

## Release Timeline

| Milestone | Duration | Features |
|-----------|----------|----------|
| **Alpha** (internal) | Done | Core DB, SSH, sync |
| **Phase 1** — Critical fixes | 1 day | Bugs, App Store requirements |
| **Phase 2** — Essential polish | 3-4 days | Settings, onboarding, SSL, error handling |
| **Beta** (TestFlight) | After Phase 2 | Internal testing, feedback |
| **Phase 3** — QoL features | 1 week | Export, advanced query, filters |
| **Phase 4** — Testing & a11y | 3-4 days | Tests, VoiceOver, localization |
| **1.0 Release** | After Phase 4 | App Store submission |
| **Phase 5** — Advanced | Ongoing | MongoDB, MSSQL, widgets |

**Estimated total to 1.0: 2-3 weeks**

---

## Architecture Notes

### What's solid (don't change)
- Actor-based drivers with dedicated thread for SSH relay
- TableProCore shared package (Models, Database, Query, Sync)
- CloudKit sync with CKRecord passthrough for macOS compatibility
- NavigationSplitView for iPad adaptive layout

### What needs attention
- SSH tunnel thread safety (NSLock + actor hybrid) — works but complex
- ConnectionFormView — largest file (500+ lines), consider splitting
- QueryEditorView — history should persist, editor needs syntax highlight
- All `Text()` with user data MUST use `Text(verbatim:)` — check any new views
