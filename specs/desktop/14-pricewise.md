# PriceWise Desktop — 5-Framework Comparison

## Desktop Rationale

PriceWise (AI-powered pricing optimizer) gains significant advantages as a desktop application:

- **Local price simulation:** Run complex pricing models and Monte Carlo simulations locally using full CPU/GPU resources without cloud compute costs or latency.
- **Sensitive data stays local:** Pricing strategies, margin data, and competitive intelligence never leave the machine — critical for enterprises where pricing is a guarded secret.
- **Offline what-if analysis:** Model pricing scenarios on flights, in meetings, or anywhere without internet. No risk of stale cloud sessions during critical pricing decisions.
- **Local database connectivity:** Connect directly to on-premise SQL Server, PostgreSQL, or MySQL databases containing sales history, inventory, and customer segments without exposing them through APIs.
- **Rich data visualization:** Native-performance rendering of complex pricing heatmaps, demand curves, elasticity charts, and multi-dimensional scenario comparisons that would lag in a browser.

---

## Desktop-Specific Features

- **Local database connector:** Direct ODBC/JDBC connections to on-premise sales databases (SQL Server, PostgreSQL, MySQL, Oracle)
- **What-if scenario workspace:** Side-by-side scenario comparison windows with live chart updates as parameters change
- **CSV/Excel bulk import:** Drag-and-drop pricing sheets, competitor price lists, and historical sales data
- **Pricing simulation engine:** Local Monte Carlo simulation using all CPU cores for demand elasticity modeling
- **System tray price alerts:** Background monitoring of competitor prices with native notifications on significant changes
- **Export to presentation:** One-click export of pricing analysis to PowerPoint/PDF for boardroom presentations
- **Keyboard-driven workflow:** Power-user shortcuts for rapid scenario creation, parameter tweaking, and chart navigation
- **Local AI inference:** Run pricing recommendation models locally (ONNX) for fully offline AI-powered suggestions
- **Scheduled data refresh:** Background sync from connected databases on configurable intervals
- **Multi-monitor support:** Detach charts and scenario panels to secondary monitors for expanded workspace

---

## Shared Packages

### core (Rust)

- **Pricing engine:** Calculate optimal prices using cost-plus, value-based, competitive, and dynamic pricing algorithms with configurable margin targets
- **Simulation runner:** Monte Carlo and sensitivity analysis across price points, volumes, and elasticity curves with multi-threaded execution via rayon
- **Demand modeler:** Fit demand curves (linear, log-linear, logistic) to historical sales data; estimate price elasticity of demand per segment
- **Scenario manager:** Create, clone, compare, and version pricing scenarios with parameter diffing and branching
- **AI recommender:** ONNX Runtime integration for local ML-based price recommendations trained on historical data
- **Data transformer:** Normalize and clean imported data from CSV, Excel, and database sources into unified schema
- **Report generator:** Build pricing analysis reports with charts, tables, and recommendations in structured format

### api-client (Rust)

- **Competitor price feeds:** Poll competitor pricing APIs or scraping endpoints; normalize into comparable price points
- **Cloud sync:** Push scenario snapshots to PriceWise cloud for team collaboration and approval workflows
- **Market data API:** Fetch commodity prices, exchange rates, and economic indicators for pricing context
- **Auth:** API key management for data feeds; OAuth2 for cloud platform; encrypted credential storage
- **Offline queue:** Buffer sync operations and API calls for replay on reconnect with conflict detection

### data (SQLite)

- **Schema:** products, price_points, scenarios, simulations, competitor_prices, sales_history, settings, sync_queue
- **Local-first:** All pricing data and scenarios stored locally; cloud sync is optional and selective
- **Analytical indexes:** Composite indexes on product+date+region for fast historical queries; FTS5 on product descriptions
- **Time-series storage:** Optimized storage for historical price and sales volume time-series data

---

## Framework Implementations

### Electron

**Architecture:** Main process handles database connections (via native ODBC bindings), background competitor monitoring, and file imports. Renderer process (React) powers the dashboard with D3.js/Recharts for complex pricing visualizations.

**Tech stack:**
- Renderer: React 18, D3.js + Recharts (charts), AG Grid (data tables), TailwindCSS, Zustand
- Main: better-sqlite3, node-odbc (database connectivity), node-cron (scheduled refreshes), electron-store
- Database: better-sqlite3 with Drizzle ORM
- AI: onnxruntime-node for local pricing models

**Native module needs:** better-sqlite3, node-odbc, onnxruntime-node, node-xlsx (Excel parsing)

**Bundle size:** ~200-280 MB (Chromium + ONNX Runtime + charting libraries)

**Memory:** ~350-550 MB (large datasets + D3 chart rendering + data grids)

**Pros:**
- D3.js ecosystem provides the richest data visualization options — pricing heatmaps, demand curves, elasticity charts
- AG Grid handles 100K+ row pricing tables with virtualized scrolling out of the box
- Maximum reuse from PriceWise SaaS web frontend
- Mature Excel/CSV parsing libraries in the Node.js ecosystem

**Cons:**
- Heavy memory footprint when multiple scenario windows are open with active charts
- Chromium overhead is substantial for a data-intensive analytical tool
- Native database connectivity (ODBC) requires fragile native module compilation across OS versions
- Bundle size feels excessive for enterprise IT deployment via MDM or SCCM
- Monte Carlo simulation in JavaScript is 4-8x slower than Rust for equivalent workloads

### Tauri

**Architecture:** Rust backend handles all data processing — database connections, simulation engine, data import/export. Svelte frontend renders charts via Chart.js or ECharts and data tables.

**Tech stack:**
- Frontend: Svelte 5, ECharts (rich charting), TanStack Table (data grids), TailwindCSS
- Backend: rusqlite, sqlx (external DB connectivity), rayon (parallel simulation), ort (ONNX), calamine (Excel parsing)
- Simulation: rayon for multi-threaded Monte Carlo, ndarray for matrix operations

**Plugin needs:** tauri-plugin-dialog (file picker), tauri-plugin-notification, tauri-plugin-fs, tauri-plugin-shell

**Bundle size:** ~12-22 MB (without ONNX model; +50-200 MB with bundled pricing model)

**Memory:** ~80-180 MB (spikes during large simulations)

**Pros:**
- Rayon enables true multi-threaded simulation — 4-8x faster Monte Carlo than JavaScript
- Rust's sqlx provides async database connectivity without native module compilation headaches
- Tiny bundle size is appealing for enterprise deployment and IT approval
- Memory-efficient data processing; can handle million-row datasets without swapping

**Cons:**
- ECharts/Chart.js in webview is less flexible than D3.js for custom pricing visualizations
- Data grid options in Svelte are less mature than AG Grid for Electron
- Building custom chart interactions (brushing, linking, cross-filtering) requires more frontend effort
- Excel parsing in Rust (calamine) covers fewer edge cases than SheetJS
- Webview rendering of 50+ charts simultaneously may need careful performance optimization

### Flutter Desktop

**Architecture:** Dart UI with Riverpod state management. Heavy computation offloaded to Rust via flutter_rust_bridge. Custom chart widgets using CustomPainter for pricing visualizations.

**Tech stack:**
- UI: Flutter 3.x, fl_chart + custom CustomPainter charts, Riverpod, DataTable2
- Database: drift (SQLite), sqflite_common_ffi
- Compute: flutter_rust_bridge → simulation engine in Rust
- Import: excel_dart for spreadsheet parsing

**Plugin needs:** file_picker, window_manager, desktop_drop, local_notifier

**Bundle size:** ~28-40 MB

**Memory:** ~150-250 MB (custom chart rendering is memory-intensive)

**Pros:**
- CustomPainter allows pixel-perfect custom pricing charts with smooth animations
- Hot reload is excellent for iterating on complex chart layouts and interactions
- Single codebase could extend to a tablet companion app for boardroom presentations
- Smooth chart animations enhance the what-if scenario comparison experience

**Cons:**
- No equivalent to AG Grid — must build or heavily customize data table for large datasets
- flutter_rust_bridge adds FFI overhead to every simulation call and complicates debugging
- Desktop text input in data entry forms still has subtle issues with selection and IME
- Chart ecosystem (fl_chart) is less feature-rich than D3.js or ECharts for specialized financial charts

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI views with @Observable view models. Swift Charts for native visualizations. Core Data or SwiftData for persistence. Accelerate framework for numerical computation.

**Tech stack:**
- UI: SwiftUI, Swift Charts, NSTableView (via AppKit interop for large tables)
- Data: SwiftData with CloudKit sync
- Compute: Accelerate framework (vecLib, vDSP) for matrix operations, Grand Central Dispatch for parallel simulation
- Import: CoreXLSX for Excel parsing
- AI: CoreML for on-device pricing models

**Bundle size:** ~10-18 MB

**Memory:** ~50-100 MB

**Pros:**
- Swift Charts provides beautiful, native-feeling data visualizations with accessibility built in
- Accelerate framework gives near-C performance for matrix operations and statistical computations
- CoreML models run efficiently on Apple Neural Engine for AI pricing recommendations
- Best multi-window experience — native macOS window management for scenario comparison
- Smallest memory footprint even with large datasets loaded

**Cons:**
- macOS only — excludes Windows users who make up a large share of enterprise pricing teams
- Swift Charts is less customizable than D3.js for highly specialized pricing visualizations
- NSTableView interop is clunky for large data grids within SwiftUI
- No direct ODBC support — connecting to enterprise databases requires third-party libraries
- Smaller ecosystem for financial data processing compared to JVM or Node.js

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform UI with ViewModel + StateFlow. JFreeChart or Compose-based charting. JDBC for database connectivity. Apache Commons Math for statistical computation.

**Tech stack:**
- UI: Compose Desktop, Material 3, lets-plot (Kotlin charting library)
- Database: SQLDelight (local), Exposed/JDBC (external database connectivity)
- Compute: Apache Commons Math, Kotlin coroutines for parallel simulation
- Import: Apache POI (Excel), Kotlin CSV
- HTTP: Ktor client

**Bundle size:** ~55-80 MB (+JRE)

**Memory:** ~250-400 MB (JVM overhead + dataset in memory)

**Pros:**
- JDBC provides the widest database connectivity — every enterprise database has a JDBC driver
- Apache POI is the most comprehensive Excel library (reads/writes .xlsx, .xls, formulas, pivot tables)
- Apache Commons Math offers battle-tested statistical and optimization algorithms
- Kotlin coroutines map naturally to parallel simulation workloads

**Cons:**
- JVM memory overhead makes the app feel heavy for a desktop tool
- Compose charting ecosystem is immature compared to web charting libraries
- Startup time (~2-3s) is noticeable when launching for a quick price check
- lets-plot is functional but lacks the polish and interactivity of D3.js or ECharts
- JRE bundling adds deployment complexity for enterprise IT teams

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 7 | 9 | 10 | 8 | 9 |
| Bundle size | 240 MB | 18 MB | 35 MB | 14 MB | 68 MB |
| Memory usage | 450 MB | 130 MB | 200 MB | 75 MB | 320 MB |
| Startup time | 2.8s | 0.9s | 1.4s | 0.5s | 2.5s |
| Native feel | 5/10 | 6/10 | 6/10 | 10/10 | 5/10 |
| Offline capability | 8/10 | 9/10 | 7/10 | 8/10 | 8/10 |
| Data visualization | 10/10 | 7/10 | 6/10 | 8/10 | 5/10 |
| Cross-platform | 9/10 | 9/10 | 9/10 | 2/10 | 8/10 |
| Best fit score | 8/10 | 7/10 | 5/10 | 6/10 | 6/10 |

---

## Recommended Framework

**Electron** is the best fit for PriceWise Desktop.

**Rationale:** PriceWise is a data-heavy analytical tool where visualization quality directly impacts user trust and decision-making. D3.js and AG Grid are unmatched for rendering complex pricing heatmaps, elasticity curves, and 100K-row pricing tables — these are the core UX differentiators. The heavier bundle and memory cost are acceptable for an enterprise tool that runs on powerful workstations, not developer laptops.

**Runner-up:** Tauri if targeting smaller businesses where lightweight deployment matters. The Rust simulation engine (rayon + ndarray) actually outperforms Electron on raw computation, but the visualization gap is the deciding factor.

---

## Monetization (Desktop)

- **Free tier:** Local-only mode — up to 3 products, basic cost-plus pricing, CSV import
- **Pro ($39/mo or $349/yr):** Unlimited products, AI pricing recommendations, demand modeling, scenario comparison, Excel import/export
- **Enterprise ($99/mo per seat):** Direct database connectors, team collaboration, approval workflows, SSO, audit trail
- **One-time license ($499):** Perpetual desktop license with 1 year of updates for consultants who use it project-by-project
- **Distribution:** Direct download with license key; MSI installer for enterprise IT; optional Mac App Store presence

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Product/price data model, CSV import, basic data grid with sorting/filtering |
| 3-4 | Pricing charts (demand curve, margin waterfall), what-if scenario creation with parameter sliders |
| 5-6 | Monte Carlo simulation engine, scenario comparison view, sensitivity analysis charts |
| 7 | AI pricing recommendations (cloud API), competitor price tracking, export to PDF/PowerPoint |
| 8 | Auto-update, packaging (DMG/MSI/AppImage), onboarding wizard, beta testing with 3-5 pilot customers |

**Post-MVP:** Local database connectors (ODBC), local AI inference, team collaboration, approval workflows, historical price tracking dashboard
