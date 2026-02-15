# OnboardFlow Desktop — 5-Framework Comparison

## Desktop Rationale

OnboardFlow (automated employee onboarding platform) delivers substantial value as a desktop application for IT administrators and HR teams:

- **IT admin command center:** A persistent desktop dashboard lets IT admins manage onboarding tasks, provision accounts, and track new hire progress without juggling browser tabs — critical during high-volume onboarding days.
- **Local document management:** Onboarding involves tax forms, NDAs, equipment agreements, and ID verification. A desktop app can store, preview, and process documents locally with native file system access and drag-and-drop.
- **Offline onboarding checklists:** Remote office managers or facilities staff in areas with unreliable internet can complete onboarding checklists offline, syncing when connectivity returns.
- **System integration (AD/LDAP):** Desktop apps can directly interface with on-premise Active Directory, LDAP, and SCIM endpoints for user provisioning without routing through a cloud proxy.
- **Native notifications for pending tasks:** OS-level notifications for overdue tasks, incomplete paperwork, and approaching start dates ensure nothing falls through the cracks during busy hiring periods.

---

## Desktop-Specific Features

- **Onboarding dashboard:** Kanban-style board showing all new hires organized by stage (pre-boarding, day-one, week-one, complete)
- **Document scanner integration:** Connect to local scanners/cameras for ID verification and physical document digitization
- **Active Directory / LDAP connector:** Direct LDAP bind for account provisioning, group assignment, and password generation without cloud middleware
- **Drag-and-drop document collection:** Drop signed PDFs, photos, or scanned docs directly into a new hire's profile folder
- **System tray reminders:** Badge count of pending onboarding tasks; click to see today's priority list
- **Template editor:** Rich WYSIWYG editor for onboarding email templates, welcome packets, and checklist templates with native print support
- **Local PDF generation:** Generate offer letters, equipment checklists, and onboarding packets as PDFs locally using native rendering
- **Bulk import:** Drag CSV/Excel files to batch-create new hire profiles from HRIS exports
- **Calendar sync:** Create orientation meetings and training sessions in native calendar apps (Outlook, macOS Calendar)
- **Equipment tracking:** Barcode/QR scanner integration for tracking laptop and badge assignments

---

## Shared Packages

### core (Rust)

- **Workflow engine:** State machine for onboarding stages (offer-accepted, documents-pending, IT-provisioned, day-one, complete) with configurable transitions and auto-triggers
- **Document processor:** PDF parsing, form field extraction, digital signature verification, and template merge for offer letters
- **Checklist engine:** Hierarchical task management with dependencies, due date calculation relative to start date, and completion tracking
- **Provisioning logic:** User account creation payloads for AD/LDAP/SCIM, email distribution list assignments, and permission templates
- **Compliance validator:** Ensure all required documents (I-9, W-4, NDA) are collected before marking onboarding complete; jurisdiction-aware rules

### api-client (Rust)

- **HRIS integration:** REST/GraphQL clients for BambooHR, Workday, Gusto, Rippling for bidirectional employee data sync
- **LDAP client:** Native LDAP v3 bind, search, and modify operations for Active Directory and OpenLDAP provisioning
- **Cloud sync:** Push onboarding progress to OnboardFlow cloud for manager visibility; pull new hire data from HRIS webhooks
- **Auth:** SAML/SSO integration for enterprise environments, JWT token management with RBAC (admin, HR, IT, manager roles)
- **Notification relay:** Send onboarding reminders via email (SMTP), Slack, and Teams when desktop notifications are insufficient

### data (SQLite)

- **Schema:** employees, onboarding_workflows, tasks, documents, equipment_assignments, provisioning_log, templates, settings, audit_trail
- **Document storage:** File paths to locally stored documents with metadata (type, status, uploaded_by, verified); binary blobs for small files
- **Sync strategy:** Conflict-free merge on task completion (once complete, always complete); last-write-wins for employee profile edits with server-side audit log
- **Compliance audit:** Immutable append-only audit trail table for all onboarding actions, exportable for HR compliance reviews

---

## Framework Implementations

### Electron

**Architecture:** Main process handles LDAP connections, file system operations (document storage, PDF generation), and system tray notifications. Renderer process (React) provides the onboarding dashboard, document viewer, and template editor.

**Tech stack:**
- Renderer: React 18, TailwindCSS, react-pdf (document viewer), TipTap (rich text editor for templates), react-beautiful-dnd (kanban board)
- Main: ldapjs (LDAP client), pdfkit (PDF generation), chokidar (file watcher), node-cron (scheduled reminders)
- Database: better-sqlite3 with Drizzle ORM
- Calendar: ical-generator for .ics file creation

**Native module needs:** ldapjs (LDAP bindings), better-sqlite3, node-usb (barcode scanner), electron-pdf-window

**Bundle size:** ~180-240 MB

**Memory:** ~250-400 MB (higher with multiple document previews open)

**Pros:**
- TipTap provides an excellent WYSIWYG template editor matching Google Docs quality
- react-pdf renders documents inline without external viewer dependencies
- ldapjs is mature and well-documented for Active Directory integration
- react-beautiful-dnd delivers a polished kanban drag-and-drop experience

**Cons:**
- Heavy bundle for an admin tool that runs alongside other enterprise apps
- PDF generation via pdfkit is slower than native alternatives
- LDAP operations in Node.js are single-threaded, blocking during large directory queries
- Memory overhead compounds when multiple admin users run it on the same terminal server

### Tauri

**Architecture:** Rust backend handles LDAP connections (ldap3 crate), PDF generation (printpdf), file management, and background task scheduling. Svelte frontend provides the dashboard UI, document viewer, and template editor.

**Tech stack:**
- Frontend: Svelte 5, TailwindCSS, PDF.js (document viewer), TipTap (template editor), svelte-dnd-action (kanban)
- Backend: ldap3 (async LDAP), printpdf, rusqlite, tokio, lettre (SMTP for email notifications)
- Notifications: tauri-plugin-notification

**Plugin needs:** tauri-plugin-notification, tauri-plugin-dialog (file picker), tauri-plugin-fs, tauri-plugin-shell

**Bundle size:** ~10-18 MB

**Memory:** ~50-90 MB (spikes to ~130 MB during bulk document processing)

**Pros:**
- ldap3 crate is async and non-blocking — handles large AD queries without freezing the UI
- printpdf generates PDFs natively in Rust with minimal overhead
- Tiny footprint is ideal for enterprise environments where IT admins run many tools simultaneously
- Rust backend can process bulk CSV imports of 10K+ employees without memory issues

**Cons:**
- TipTap on system webview may render slightly differently across OS versions
- PDF.js in a webview is less integrated than Electron's native PDF handling
- LDAP testing and debugging is harder without Node.js ecosystem tooling
- Fewer pre-built enterprise UI components compared to React ecosystem

### Flutter Desktop

**Architecture:** Flutter UI with Riverpod state management. Platform channels for LDAP access (calling system ldapsearch or native libraries). flutter_rust_bridge for PDF generation and document processing.

**Tech stack:**
- UI: Flutter 3.x, Riverpod, syncfusion_flutter_pdfviewer, flutter_quill (rich text editor)
- Database: drift (SQLite)
- LDAP: platform channel to native LDAP libraries (DirectoryServices on macOS, Win32 LDAP on Windows)
- PDF: flutter_rust_bridge to printpdf (Rust)

**Plugin needs:** tray_manager, local_notifier, file_picker, window_manager, desktop_drop

**Bundle size:** ~28-38 MB

**Memory:** ~110-180 MB

**Pros:**
- Syncfusion PDF viewer is feature-rich with annotation support (useful for document review)
- flutter_quill provides a decent rich text editor for template creation
- Could extend to a mobile app for managers to approve onboarding tasks on the go
- Beautiful animated transitions between onboarding stages

**Cons:**
- LDAP integration requires platform-specific channel implementations for each OS
- Syncfusion components have separate licensing costs
- Desktop drag-and-drop (desktop_drop) is less polished than web-based alternatives
- No native print dialog integration — must generate PDF first, then open in system viewer

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI views with @Observable view models. OpenDirectory framework for native macOS directory services (Active Directory binding). PDFKit for document viewing and generation. EventKit for calendar integration.

**Tech stack:**
- UI: SwiftUI, PDFKit (native PDF), NSTextView (rich text editing)
- Data: SwiftData
- Directory: OpenDirectory.framework (native AD/LDAP on macOS)
- Calendar: EventKit for orientation meeting scheduling
- Notifications: UNUserNotificationCenter with actionable notifications

**Bundle size:** ~8-14 MB

**Memory:** ~35-70 MB

**Pros:**
- OpenDirectory framework provides the best Active Directory integration on macOS — native Kerberos auth, group policy reading
- PDFKit is the native macOS PDF engine — fastest rendering, annotation support, digital signatures
- EventKit creates calendar events directly in system calendar
- Native print support with full macOS print dialog and preview
- Lowest resource usage for enterprise environments

**Cons:**
- macOS only — most enterprise IT teams run Windows (dealbreaker for many orgs)
- SwiftUI table view for large employee lists is less mature than web data grids
- No Windows Active Directory management (the primary use case)
- Limited to macOS directory services; no direct Windows AD controller management

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform for UI. UnboundID LDAP SDK for directory operations. Apache PDFBox for PDF processing. SQLDelight for persistence.

**Tech stack:**
- UI: Compose Desktop, Material 3, PDFBox (via Swing interop for viewer)
- LDAP: UnboundID LDAP SDK (pure Java, excellent AD support)
- Database: SQLDelight
- PDF: Apache PDFBox (generation and viewing)
- HTTP: Ktor client
- Calendar: ical4j for .ics generation

**Bundle size:** ~55-80 MB (+JRE)

**Memory:** ~180-320 MB (JVM + LDAP connection pool + PDFBox rendering)

**Pros:**
- UnboundID LDAP SDK is the most mature, feature-complete LDAP library available (used by Ping Identity, ForgeRock)
- Apache PDFBox handles complex PDF forms, digital signatures, and accessibility tags
- JVM ecosystem has the strongest enterprise integration libraries (SAML, SCIM, OAuth2)
- Could share onboarding logic with Android companion app for manager approvals

**Cons:**
- Highest memory footprint — problematic for terminal server environments where multiple admins share a machine
- JVM startup time (~2.5s) feels sluggish for a tool opened frequently during onboarding rushes
- Compose Desktop table components less polished than web data grids
- PDFBox rendering via Swing interop creates visual inconsistency with Compose UI

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 7 | 9 | 10 | 8 | 9 |
| Bundle size | 210 MB | 14 MB | 33 MB | 11 MB | 68 MB |
| Memory usage | 320 MB | 70 MB | 145 MB | 50 MB | 250 MB |
| Startup time | 2.8s | 0.9s | 1.3s | 0.5s | 2.5s |
| Native feel | 5/10 | 6/10 | 6/10 | 10/10 | 5/10 |
| Offline capability | 7/10 | 8/10 | 7/10 | 8/10 | 7/10 |
| LDAP/AD integration | 7/10 | 7/10 | 5/10 | 8/10 | 9/10 |
| Document handling | 8/10 | 7/10 | 7/10 | 9/10 | 8/10 |
| Cross-platform | 9/10 | 9/10 | 9/10 | 2/10 | 8/10 |
| Best fit score | 8/10 | 7/10 | 5/10 | 4/10 | 6/10 |

---

## Recommended Framework

**Electron** is the best fit for OnboardFlow Desktop.

**Rationale:** OnboardFlow is a rich enterprise dashboard that prioritizes UI completeness over lightweight footprint. IT admins need a polished kanban board, inline document previews, WYSIWYG template editing, and robust LDAP integration — all areas where Electron's mature ecosystem (TipTap, react-pdf, react-beautiful-dnd, ldapjs) delivers the fastest path to a production-quality experience. Enterprise IT teams expect heavier desktop apps and typically run them on well-provisioned machines.

**Runner-up:** Tauri if resource efficiency is critical (e.g., terminal server deployments where multiple admins share hardware), though the trade-off is a less polished document handling and template editing experience.

---

## Monetization (Desktop)

- **Starter (free):** Up to 5 active onboardings, basic checklists, local-only storage
- **Professional ($29/mo per admin seat):** Unlimited onboardings, LDAP/AD provisioning, document management, cloud sync for manager visibility
- **Enterprise ($79/mo per admin seat):** SAML SSO, SCIM provisioning, compliance audit trail export, custom workflow builder, priority support
- **On-premise license ($5,000/yr):** Self-hosted cloud sync server for organizations that cannot use SaaS; includes desktop licenses for all admins
- **Implementation services ($2,500 one-time):** Custom workflow setup, AD schema mapping, document template design for large enterprises

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | New hire profile creation, onboarding checklist engine with stage transitions, SQLite persistence |
| 3-4 | Dashboard kanban board (drag between stages), document upload/preview, bulk CSV import |
| 5-6 | LDAP/AD connector for account provisioning, email template editor, notification system |
| 7 | System tray with pending task count, calendar integration (.ics export), offline queue |
| 8 | Auto-update, packaging (MSI/DMG/AppImage), compliance audit trail export, beta testing |

**Post-MVP:** SCIM provisioning, SAML SSO, equipment barcode tracking, mobile manager approval app, custom workflow builder, HRIS bidirectional sync
