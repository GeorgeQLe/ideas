# DataPipe Desktop — 5-Framework Comparison

## Desktop Rationale

DataPipe (no-code visual ETL pipeline builder) is one of the strongest desktop candidates due to the inherent advantages of local data processing:

- **Local data processing — no cloud egress:** Sensitive data (PII, financial records, healthcare data) never leaves the machine. Eliminates cloud egress costs, data residency concerns, and the compliance overhead of transmitting regulated data through third-party servers.
- **Visual pipeline builder with native performance:** Dragging, connecting, and configuring dozens of pipeline nodes requires smooth 60fps rendering. Desktop apps leverage GPU-accelerated canvas rendering without browser sandbox overhead.
- **Handle large files locally:** Process multi-GB CSV, Parquet, and JSON files that would timeout or fail in browser-based ETL tools. Memory-mapped file access and streaming transforms enable datasets that exceed RAM.
- **Connect to local databases:** Direct TCP connections to on-premise PostgreSQL, MySQL, SQL Server, and MongoDB instances without exposing them to the internet or configuring SSH tunnels to a cloud service.
- **Schedule local jobs:** Cron-style job scheduling for recurring ETL pipelines that run on the local machine — useful for nightly data warehouse loads or periodic report generation without paying for cloud compute.

---

## Desktop-Specific Features

- **Visual node graph editor:** Drag-and-drop pipeline builder with source nodes, transform nodes, and destination nodes connected by data flow edges
- **Local database browser:** Connect to and browse local/network database schemas, preview tables, and select columns — all without leaving the app
- **File system source/destination:** Drag files or folders onto the canvas as data sources; write outputs directly to local directories
- **Live data preview:** Click any edge in the pipeline to see a live preview of the data flowing through that connection (first 1000 rows)
- **Memory-mapped large file processing:** Handle 10GB+ CSV and Parquet files using streaming transforms without loading entire files into RAM
- **Job scheduler:** Built-in cron-style scheduler for recurring pipeline executions with logging and failure alerts
- **Resource monitor:** Real-time CPU, memory, and disk I/O usage during pipeline execution displayed in the status bar
- **Local credential vault:** Store database passwords and API keys in the OS keychain (macOS Keychain, Windows Credential Manager) — never in plain text
- **Pipeline versioning:** Git-style version control for pipeline definitions with diff view and rollback
- **Export/import pipelines:** Share pipeline definitions as portable JSON files with teammates

---

## Shared Packages

### core (Rust)

- **Pipeline engine:** DAG execution engine with topological sort, parallel branch execution, backpressure handling, and graceful error recovery
- **Transform library:** Built-in transforms — filter, map, join, aggregate, pivot, unpivot, deduplicate, sort, window functions, regex extract, type cast
- **Connector framework:** Pluggable source/destination trait with implementations for CSV, JSON, Parquet, PostgreSQL, MySQL, SQLite, REST API, S3
- **Schema inference:** Automatic column type detection from file samples with confidence scoring; schema evolution handling for changing source data
- **Expression evaluator:** Safe expression language for custom transforms (e.g., `upper(trim(name))`, `amount * 1.1`, `date_parse(created_at, '%Y-%m-%d')`)
- **Streaming engine:** Arrow-based columnar data processing with configurable batch sizes for memory-efficient large file handling

### api-client (Rust)

- **Cloud sync:** Push pipeline definitions and run history to DataPipe cloud for team collaboration and remote monitoring
- **Connector marketplace:** Download community-built connectors (Salesforce, HubSpot, Stripe) from a registry
- **Auth:** License validation, team workspace auth with role-based pipeline access
- **Telemetry:** Optional anonymous usage analytics for pipeline patterns (which transforms are most used)

### data (SQLite)

- **Schema:** pipelines (id, name, definition_json, version, created, updated), runs (pipeline_id, status, started, completed, rows_processed, error_log), connections (type, host, credentials_ref), schedules, connector_registry
- **Run history:** Append-only run log with per-node execution stats (rows in, rows out, duration, errors) for pipeline optimization
- **Sync strategy:** Pipeline definitions use CRDT-style merge for collaborative editing; run history is local-only (too large to sync)

---

## Framework Implementations

### Electron

**Architecture:** Main process handles database connections (via native drivers), file system access, and job scheduling. Renderer process (React) provides the visual pipeline builder using React Flow, data preview tables, and monitoring dashboards.

**Tech stack:**
- Renderer: React 18, React Flow (node graph editor), AG Grid (data preview tables), TailwindCSS, Zustand
- Main: node-postgres, mysql2, better-sqlite3, node-cron (scheduler), chokidar (file watcher)
- Database: better-sqlite3 for app state; native drivers for pipeline sources
- Data processing: Node.js streams + worker_threads for parallel transforms

**Native module needs:** better-sqlite3, node-postgres (libpq bindings), duckdb-node (for Parquet/analytical queries), keytar (OS keychain)

**Bundle size:** ~200-280 MB

**Memory:** ~300-500 MB base (can spike to 1-2 GB during large pipeline runs)

**Pros:**
- React Flow is the most mature, feature-rich node graph editor available — drag-and-drop, zoom, minimap, custom node types
- AG Grid handles million-row data previews with virtualized scrolling
- Largest ecosystem of database drivers (pg, mysql2, mssql, mongodb)
- worker_threads enable parallel pipeline execution in Node.js

**Cons:**
- High base memory compounds with data processing memory — a 2GB pipeline run on top of 300MB base is problematic
- JavaScript data transforms are 5-10x slower than Rust/C++ for large datasets
- Node.js streams have higher overhead than native streaming for multi-GB files
- Chromium rendering can stutter with 50+ nodes on the canvas

### Tauri

**Architecture:** Rust backend handles all data processing (Arrow-based streaming), database connections, file I/O, and job scheduling. Svelte frontend provides the visual pipeline builder with a custom canvas-based node editor.

**Tech stack:**
- Frontend: Svelte 5, Svelvet (node graph editor) or custom canvas renderer, TailwindCSS, AG Grid (data preview)
- Backend: arrow-rs (columnar data), connectorx (database reads), datafusion (SQL transforms), rusqlite, tokio, keyring (OS credential store)
- File formats: csv, parquet, serde_json

**Plugin needs:** tauri-plugin-fs, tauri-plugin-dialog, tauri-plugin-shell, tauri-plugin-store

**Bundle size:** ~15-25 MB

**Memory:** ~60-120 MB base; pipeline processing uses Rust's memory-mapped I/O for minimal overhead

**Pros:**
- **Best data processing performance:** arrow-rs + datafusion provide production-grade columnar processing (same engine as Apache DataFusion)
- Memory-mapped file I/O in Rust processes 10GB+ files without loading into RAM
- connectorx reads from PostgreSQL/MySQL 3-10x faster than Node.js or Python drivers
- Tiny base memory leaves maximum headroom for actual data processing
- tokio enables true parallel pipeline branch execution

**Cons:**
- Svelvet (Svelte node graph) is less mature than React Flow — may need custom canvas work
- AG Grid or similar data preview table adds web dependency overhead
- Complex visual pipeline builder requires significant frontend development regardless of framework
- Debugging Rust data transforms is harder than JavaScript during development

### Flutter Desktop

**Architecture:** Flutter UI with custom canvas-based pipeline editor. flutter_rust_bridge for data processing engine (arrow-rs). Platform channels for database connections and file system access.

**Tech stack:**
- UI: Flutter 3.x, custom CustomPainter (pipeline canvas), data_table_2 (data preview), Riverpod
- Database: drift (app state); flutter_rust_bridge to native database drivers
- Data processing: flutter_rust_bridge to arrow-rs + datafusion
- Scheduling: background isolate with timer-based cron

**Plugin needs:** window_manager, file_picker, desktop_drop, flutter_rust_bridge

**Bundle size:** ~30-45 MB

**Memory:** ~120-200 MB base

**Pros:**
- CustomPainter provides GPU-accelerated canvas for smooth pipeline editing with 100+ nodes
- Flutter's gesture system handles complex drag-and-drop, zoom, and pan natively
- Could extend to tablet app for pipeline monitoring (not building, but viewing runs)
- Hot reload accelerates iteration on the complex visual editor

**Cons:**
- No equivalent to React Flow — must build the entire node graph editor from scratch (significant effort)
- data_table_2 cannot handle million-row previews like AG Grid
- All data processing must cross FFI bridge (flutter_rust_bridge), adding latency for live data previews
- Desktop file drag-and-drop is less smooth than web or native implementations

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI views with AppKit interop for the canvas-based pipeline editor (NSView with Core Animation layers). Native database connections via Swift libraries. Grand Central Dispatch for parallel pipeline execution.

**Tech stack:**
- UI: SwiftUI + NSView (custom pipeline canvas with CALayers), NSTableView (data preview)
- Data: SwiftData (app state)
- Database: PostgresNIO, MySQLNIO (SwiftNIO-based async drivers)
- File formats: TabularData framework (Apple's built-in CSV/JSON parser), custom Parquet reader
- Scheduling: launchd integration for system-level job scheduling
- Keychain: Security framework for credential storage

**Bundle size:** ~10-18 MB

**Memory:** ~40-80 MB base; GCD manages processing memory efficiently

**Pros:**
- Core Animation layers provide the smoothest canvas rendering for the pipeline editor (hardware-accelerated compositing)
- TabularData framework handles CSV/JSON parsing natively with good performance
- launchd integration enables true system-level scheduling that survives app restarts
- Security framework provides the most secure credential storage via macOS Keychain
- GCD parallel execution maps naturally to pipeline DAG branch parallelism

**Cons:**
- macOS only — excludes data engineers on Linux (a significant portion of the target audience)
- No Parquet support in TabularData — requires custom reader or C library bridging
- Smaller ecosystem of database drivers (PostgresNIO/MySQLNIO less mature than libpq/mysqlclient)
- Building a custom pipeline canvas in NSView/CALayer is significant engineering effort

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform for UI with custom Canvas composable for pipeline editor. JDBC for database connections. Apache Arrow Java for columnar processing. Coroutines for parallel execution.

**Tech stack:**
- UI: Compose Desktop, custom Canvas (pipeline editor), Material 3
- Database connections: JDBC (PostgreSQL, MySQL, SQL Server, Oracle — broadest driver support)
- Data processing: Apache Arrow Java, Apache Spark (optional for heavy processing)
- File formats: Apache Parquet (Java), opencsv
- Scheduling: Quartz Scheduler
- HTTP: Ktor client

**Bundle size:** ~65-100 MB (+JRE)

**Memory:** ~200-400 MB base (JVM + Arrow buffers)

**Pros:**
- **Broadest database support:** JDBC connects to virtually every database (Oracle, DB2, Teradata, Snowflake) — critical for enterprise ETL
- Apache Arrow Java and Parquet libraries are production-grade (same libraries used in Apache Spark)
- Quartz Scheduler is the most feature-complete job scheduler available (cron, dependencies, retry policies)
- Kotlin coroutines with structured concurrency map elegantly to pipeline DAG execution
- Compose Canvas API is capable of building complex node editors

**Cons:**
- Highest memory footprint — problematic when pipeline processing itself needs significant RAM
- JVM startup delay means scheduled pipeline runs have a 2-3s overhead
- Compose Canvas is less mature than web canvas libraries for interactive graph editing
- JVM garbage collection pauses can cause pipeline editor UI stuttering with large graphs

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 8 | 10 | 12 | 11 | 10 |
| Bundle size | 240 MB | 20 MB | 38 MB | 14 MB | 82 MB |
| Memory usage | 400 MB | 90 MB | 160 MB | 60 MB | 300 MB |
| Startup time | 3.0s | 1.0s | 1.4s | 0.6s | 2.5s |
| Native feel | 5/10 | 6/10 | 6/10 | 9/10 | 5/10 |
| Offline capability | 8/10 | 9/10 | 7/10 | 9/10 | 8/10 |
| Data processing perf | 5/10 | 10/10 | 8/10 | 7/10 | 8/10 |
| Database connectivity | 8/10 | 7/10 | 5/10 | 5/10 | 10/10 |
| Visual editor quality | 9/10 | 7/10 | 6/10 | 7/10 | 6/10 |
| Cross-platform | 9/10 | 9/10 | 9/10 | 2/10 | 8/10 |
| Best fit score | 7/10 | 8/10 | 5/10 | 4/10 | 6/10 |

---

## Recommended Framework

**Tauri** is the best fit for DataPipe Desktop, with Electron as a close alternative.

**Rationale:** DataPipe's core value proposition is local data processing, which demands maximum memory headroom for actual data work. Tauri's 90 MB base versus Electron's 400 MB means significantly more RAM available for processing multi-GB datasets. The Rust backend with arrow-rs and datafusion provides production-grade columnar processing — the same engine that powers Apache DataFusion. The main trade-off is a less mature visual node graph editor (Svelvet vs React Flow), which requires more frontend investment, but the data processing performance advantage is decisive for an ETL tool.

**Runner-up:** Electron, if the visual pipeline editor quality is the top priority. React Flow is substantially more mature than any Svelte or Rust alternative, and for teams that prioritize the drag-and-drop builder experience over raw data processing performance, Electron delivers a faster path to a polished UI.

---

## Monetization (Desktop)

- **Free tier:** 3 pipelines, local file sources only (CSV/JSON), basic transforms, no scheduling
- **Professional ($39/mo):** Unlimited pipelines, all database connectors, job scheduler, cloud sync, Parquet support
- **Team ($19/user/mo):** Shared pipeline library, collaborative editing, team credential vault, run history sharing
- **Enterprise ($99/mo flat):** On-premise connector marketplace, SSO, audit logging, custom connector SDK, priority support
- **Connector packs ($9/mo each):** Premium connectors for Salesforce, Snowflake, BigQuery, Databricks sold as add-on modules

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Visual node graph editor with drag-and-drop (source, transform, destination nodes), pipeline definition persistence |
| 3-4 | CSV/JSON file source and destination nodes, basic transforms (filter, map, sort), live data preview on edges |
| 5-6 | PostgreSQL and MySQL connectors, schema browser, column selection, credential storage in OS keychain |
| 7 | Pipeline execution engine with streaming processing, progress indicators, error handling and retry |
| 8 | Job scheduler, run history with per-node stats, auto-update, packaging, beta launch |

**Post-MVP:** Parquet support, additional database connectors (SQL Server, MongoDB, Snowflake), custom transform expressions, pipeline versioning, team collaboration, connector marketplace, Spark integration for heavy workloads
