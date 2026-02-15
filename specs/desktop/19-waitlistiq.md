# WaitlistIQ Desktop — 5-Framework Comparison

## Desktop Rationale

WaitlistIQ (smart waitlist management with analytics) benefits from a desktop version:

- **Offline waitlist management:** Import, review, and segment waitlist entries without internet. Critical during events or travel where connectivity is spotty — never lose track of signups.
- **Local analytics dashboard:** Process thousands of waitlist entries locally for instant trend analysis, conversion funnels, and cohort breakdowns without round-trip API latency.
- **Native notifications for signups:** Real-time OS-level notifications when high-value prospects join, when waitlist milestones are hit, or when referral chains activate.
- **CRM integration:** Connect to local CRM databases (HubSpot export, Salesforce CSV) and cross-reference waitlist entries without uploading data to another cloud service.
- **Export/import capabilities:** Drag-and-drop CSV/Excel files for bulk import, one-click export to mailing list tools, and direct clipboard integration for quick adds.

---

## Desktop-Specific Features

- **System tray signup counter:** Live badge showing total waitlist size and today's signup count
- **Native notification center:** Configurable alerts for new signups, milestone thresholds (100, 500, 1K), and VIP registrations
- **Drag-and-drop CSV import:** Drop spreadsheet files onto the app to bulk-import waitlist entries with column mapping
- **Local analytics engine:** Compute signup velocity, geographic distribution, referral attribution, and conversion funnels locally
- **Clipboard quick-add:** Copy an email address, press global hotkey to instantly add to waitlist
- **Multi-waitlist dashboard:** Manage multiple product waitlists from a single unified interface
- **CRM file sync:** Watch a folder for CRM exports and auto-correlate with waitlist entries
- **Scheduled reports:** Generate and export PDF/HTML reports on a schedule without browser interaction
- **Offline queue:** Collect signups from local forms or manual entry; batch-sync when online

---

## Shared Packages

### core (Rust)

- **Entry validator:** Email validation, deduplication detection, spam/bot filtering with configurable rules
- **Analytics engine:** Signup velocity calculation, cohort analysis, referral chain tracking, funnel metrics
- **Segmentation engine:** Rule-based and score-based segmentation (geography, referral count, signup source, custom fields)
- **Export formatter:** Generate CSV, Excel (via calamine), PDF reports, and HTML email templates
- **Referral tracker:** Build referral trees, calculate viral coefficients, identify top referrers
- **Scoring model:** Assign priority scores based on configurable weights (referral depth, engagement, profile completeness)

### api-client (Rust)

- **Cloud sync:** Bidirectional sync of waitlist entries with WaitlistIQ cloud; conflict resolution via server timestamp
- **Webhook relay:** Forward signup events to configured webhook endpoints (Slack, Zapier, custom)
- **Email service integration:** Trigger welcome emails and position-update emails via SendGrid/Postmark/Resend APIs
- **Auth:** API key management for widget embedding, OAuth2 for team access
- **Bulk operations:** Batch import/export endpoints with progress reporting and rollback on failure

### data (SQLite)

- **Schema:** waitlists, entries (email, name, referral_code, referrer_id, source, score, status, custom_fields JSON, created, synced), segments, analytics_snapshots, export_history, settings
- **Local-first:** All waitlist data stored locally; cloud sync is optional for hosted widget and team access
- **Full-text search:** FTS5 index on entry names, emails, and custom fields for instant filtering
- **Time-series:** Hourly/daily signup aggregates stored for fast chart rendering without re-scanning entries

---

## Framework Implementations

### Electron

**Architecture:** Main process handles file watching for CRM imports, system tray with signup counter, notification scheduling, and clipboard monitoring. Renderer process (React) for analytics dashboard, entry management, and segmentation UI.

**Tech stack:**
- Renderer: React 18, Recharts (analytics graphs), TailwindCSS, TanStack Table (entry lists), Zustand
- Main: chokidar (file watching), electron-store, node-cron (scheduled reports), clipboard module
- Database: better-sqlite3 with Drizzle ORM
- Export: pdfkit (PDF reports), exceljs (Excel export)

**Native module needs:** better-sqlite3, pdfkit, electron-positioner (tray popup)

**Bundle size:** ~170-230 MB

**Memory:** ~220-380 MB (higher with large analytics charts and entry tables loaded)

**Pros:**
- Recharts/D3 provide publication-quality analytics visualizations
- TanStack Table handles 100K+ entry lists with virtualization
- Full SaaS dashboard code reuse for analytics views
- Mature CSV/Excel parsing libraries in the npm ecosystem

**Cons:**
- Heavy bundle for what is primarily a data management tool
- Memory overhead grows linearly with loaded entry count in the renderer
- Scheduled report generation keeps full Chromium alive in background
- File watching + tray + Chromium = excessive resource usage for a background task

### Tauri

**Architecture:** Rust backend handles CSV/Excel parsing (calamine, csv crates), analytics computation, file watching (notify crate), and system tray management. Svelte frontend for dashboard, entry tables, and segmentation UI.

**Tech stack:**
- Frontend: Svelte 5, Chart.js (analytics), TailwindCSS, AG Grid (entry table)
- Backend: csv, calamine (Excel), rusqlite, notify (file watcher), tokio, lettre (email)
- Export: printpdf (PDF reports), rust_xlsxwriter (Excel export)

**Plugin needs:** tauri-plugin-notification, tauri-plugin-dialog, tauri-plugin-fs, tauri-plugin-clipboard-manager

**Bundle size:** ~6-12 MB

**Memory:** ~35-70 MB (efficient even with large datasets due to Rust backend processing)

**Pros:**
- Rust CSV/Excel parsing is 10-50x faster than JavaScript equivalents for bulk imports
- Analytics computation in Rust handles 500K+ entries without UI thread blocking
- Tiny bundle appropriate for a business tool distributed to marketing teams
- File watching via notify crate uses minimal resources for CRM folder monitoring
- Background analytics aggregation runs efficiently in Rust async tasks

**Cons:**
- Chart.js is less feature-rich than D3/Recharts for complex analytics
- AG Grid web version works but lacks some native table feel
- Rust PDF generation libraries are less mature than pdfkit
- Smaller ecosystem for email template rendering

### Flutter Desktop

**Architecture:** Dart UI with Riverpod state management. Heavy data processing delegated to Rust via flutter_rust_bridge (CSV parsing, analytics, scoring). Custom chart widgets for analytics.

**Tech stack:**
- UI: Flutter 3.x, fl_chart (analytics), Riverpod, DataTable2 (entry lists)
- Database: drift (SQLite)
- Import: flutter_rust_bridge -> calamine + csv crates
- Export: pdf package (Dart), excel package (Dart)

**Plugin needs:** tray_manager, local_notifier, file_picker, window_manager, desktop_drop

**Bundle size:** ~25-35 MB

**Memory:** ~100-170 MB

**Pros:**
- fl_chart provides beautiful animated analytics dashboards
- Hot reload excellent for iterating on complex dashboard layouts
- desktop_drop plugin enables drag-and-drop CSV import
- Single codebase extends to mobile app for checking waitlist on the go

**Cons:**
- Large data tables in Flutter desktop can be janky with 50K+ rows
- CSV/Excel parsing in Dart is slower than Rust — FFI bridge adds complexity
- Desktop file drag-and-drop less polished than native implementations
- PDF export libraries in Dart are less capable than native alternatives

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI views with @Observable models. Native NSTableView (via NSViewRepresentable) for high-performance entry lists. Charts framework for analytics. FileManager + NSMetadataQuery for folder watching.

**Tech stack:**
- UI: SwiftUI, Charts framework (native), NSTableView for large lists
- Data: SwiftData with CloudKit sync
- Import: TabularData framework (Apple's native CSV), CoreXLSX (Excel)
- Export: PDFKit (native), CKHTML for email templates
- Notifications: UNUserNotificationCenter

**Bundle size:** ~6-10 MB

**Memory:** ~25-55 MB

**Pros:**
- Native NSTableView handles 1M+ rows with zero jank (virtual scrolling built in)
- Apple Charts framework renders crisp, interactive analytics charts
- TabularData framework provides fast, memory-efficient CSV parsing
- Native drag-and-drop, notification center, and Share Sheet integration
- Smallest memory footprint — ideal for always-running background monitoring
- Spotlight integration for searching waitlist entries from anywhere

**Cons:**
- macOS only — excludes Windows users (many marketing teams use Windows)
- CoreXLSX is less capable than calamine for complex Excel files
- Smaller community for business/marketing tool development on Swift
- No cross-platform code sharing with web dashboard

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform UI with Material 3 data tables. ViewModel + StateFlow for reactive analytics. Apache POI for Excel, OpenCSV for CSV. Ktor for API communication.

**Tech stack:**
- UI: Compose Desktop, Material 3 DataTable
- Database: SQLDelight
- Import: Apache POI (Excel), OpenCSV (CSV)
- Export: Apache PDFBox (PDF), Apache POI (Excel)
- HTTP: Ktor client
- Notifications: java.awt.TrayIcon.displayMessage

**Bundle size:** ~50-75 MB (+JRE)

**Memory:** ~170-280 MB

**Pros:**
- Apache POI is the most feature-complete Excel library in any language
- OpenCSV handles edge cases (encoding, quoting) that simpler parsers miss
- Compose DataTable with lazy columns handles large datasets
- Could share waitlist logic with Android companion app for mobile notifications

**Cons:**
- JVM startup time (~2s) feels slow for checking signup notifications
- High memory usage for a background monitoring tool
- AWT system tray notifications look non-native on all platforms
- Apache POI loads entire Excel files into memory — problematic for large imports

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 5 | 6 | 8 | 6 | 7 |
| Bundle size | 200 MB | 9 MB | 30 MB | 8 MB | 65 MB |
| Memory usage | 300 MB | 50 MB | 130 MB | 40 MB | 220 MB |
| Startup time | 2.5s | 0.7s | 1.1s | 0.4s | 2.0s |
| Native feel | 5/10 | 7/10 | 6/10 | 10/10 | 4/10 |
| Offline capability | 7/10 | 9/10 | 7/10 | 9/10 | 7/10 |
| Data import/export | 8/10 | 8/10 | 6/10 | 7/10 | 9/10 |
| Cross-platform | 9/10 | 9/10 | 9/10 | 2/10 | 8/10 |
| Best fit score | 6/10 | 9/10 | 6/10 | 7/10 | 5/10 |

---

## Recommended Framework

**Tauri** is the best fit for WaitlistIQ Desktop.

**Rationale:** WaitlistIQ is a lightweight dashboard tool that needs fast data processing for imports and analytics. Tauri's Rust backend delivers exceptional CSV/Excel parsing performance and can crunch analytics on 500K+ entries without blocking the UI. The 9 MB bundle is appropriate for distribution to marketing and growth teams who may not tolerate developer-tool-sized installs. Background file watching and tray notifications consume minimal resources.

**Runner-up:** Swift/SwiftUI for a Mac-exclusive version with native table performance and Charts framework, but the cross-platform requirement (marketing teams often use Windows) makes Tauri the practical choice.

---

## Monetization (Desktop)

- **Free tier:** Single waitlist, up to 500 entries, local-only analytics
- **Pro ($14/mo or $119/yr):** Unlimited waitlists, cloud sync, referral tracking, scheduled reports, webhook integrations
- **Team ($8/user/mo):** Shared waitlists, role-based access, team analytics, CRM folder sync
- **One-time license ($129):** Perpetual desktop license with 1 year of updates for solo founders
- **Distribution:** Direct download + Mac App Store (good for business procurement); skip Homebrew (non-developer audience)

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Waitlist CRUD, entry management UI, SQLite storage, CSV import with column mapping |
| 3-4 | Analytics dashboard (signup velocity, source breakdown, geographic distribution charts) |
| 5-6 | System tray with signup counter, native notifications for milestones and VIP signups |
| 7 | Referral tracking, entry scoring, segmentation engine, export to CSV/Excel/PDF |
| 8 | Cloud sync, auto-update, packaging (DMG/MSI/AppImage), beta testing with 5 waitlist owners |

**Post-MVP:** CRM folder watching, scheduled report generation, webhook integrations, email automation triggers, embeddable widget management from desktop
