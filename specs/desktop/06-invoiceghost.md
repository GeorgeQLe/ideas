# InvoiceGhost Desktop — 5-Framework Comparison

## Desktop Rationale

InvoiceGhost (automated invoice/payment follow-up) delivers substantial value as a desktop application:

- **Desktop notifications for overdue payments:** Native OS notifications alert users immediately when invoices become overdue or when payment reminders need to be sent — no need to check a web dashboard constantly. Badge counts on the dock/taskbar icon show outstanding items at a glance.
- **Local invoice storage and security:** Sensitive financial documents (invoices, payment records, client information) stay on the user's machine by default. No mandatory cloud storage for confidential financial data — critical for businesses with data residency requirements.
- **Email drafting offline:** Compose, preview, and schedule follow-up emails without an internet connection. Emails queue locally and send when connectivity returns — freelancers often work from cafes and co-working spaces with spotty Wi-Fi.
- **Integration with local accounting software:** Interface with QuickBooks Desktop, Xero local exports, CSV/QIF files from banking software, and local PDF invoice files. Bridge the gap between accounting systems and payment follow-up.
- **PDF generation and management:** Generate professional invoices as PDFs natively, with local template storage, digital signatures, batch export, and direct printing support. No cloud rendering latency or per-document API costs.

---

## Desktop-Specific Features

- **Payment follow-up scheduler:** Automated escalating reminder emails (gentle at day 1, firm at day 7, final notice at day 14) triggered on configurable schedules per client
- **Native notifications:** OS alerts for overdue invoices, payment received confirmations, upcoming follow-up actions, and email delivery failures
- **Local PDF invoice generator:** Create professional PDF invoices from customizable templates without cloud rendering — includes digital signature and company logo support
- **Email composer:** Rich text email editor for drafting follow-up messages with invoice attachments, payment links, and personalized variables
- **Accounting software bridge:** Import invoices from QuickBooks Desktop via IIF/CSV, Xero CSV exports, FreshBooks exports, and generic CSV/OFX formats
- **Drag-and-drop invoice import:** Drop PDF invoices onto the app to OCR-extract details (amount, due date, client name, invoice number, line items)
- **System tray payment summary:** Tray icon with badge showing count of overdue invoices; tooltip shows total outstanding amount in default currency
- **Batch operations:** Select multiple invoices for bulk follow-up, bulk export to PDF, bulk status updates, or bulk archival
- **Offline email queue:** Compose and schedule emails offline; queue processes automatically when internet reconnects with delivery confirmation
- **Local encryption:** AES-256 encryption for stored financial data with master password or system keychain integration (macOS Keychain, Windows Credential Manager)

---

## Shared Packages

### core (Rust)

- **Invoice model:** Complete invoice representation (line items, tax calculations with multiple rates, discounts, payment terms, multi-currency support, recurring schedules)
- **Follow-up engine:** Rule-based scheduler that determines when to send reminders based on invoice age, payment terms, escalation rules, client payment history, and business day calculations
- **PDF generator:** Render professional invoices from templates using a Rust PDF library (printpdf) with custom branding, logos, digital signatures, and QR codes for payment links
- **Email template engine:** Handlebars-style templates for follow-up emails with variable substitution (client name, amount, due date, payment link, days overdue, invoice PDF attachment)
- **OCR parser:** Extract invoice fields from PDF files using Tesseract FFI or local ML model for automated data entry
- **Currency handler:** Multi-currency support with offline exchange rate caching, conversion, and locale-aware formatting

### api-client (Rust)

- **Email delivery:** SMTP client (lettre) for direct sending, or relay through SendGrid/Postmark API with delivery tracking and open/click analytics
- **Payment gateway:** Stripe/PayPal integration for generating payment links and tracking payment status in real-time via webhooks
- **Cloud sync:** Sync invoice data and follow-up history across devices for teams with conflict resolution
- **Auth:** JWT-based auth with team workspaces, client portal tokens for payment pages, API key management
- **Bank feed integration:** Plaid or open banking API for automatic payment matching against outstanding invoices

### data (SQLite)

- **Schema:** clients (id, name, email, company, address, payment_terms, notes), invoices (id, client_id, items[], subtotal, tax, total, currency, status, due_date, paid_date, paid_amount), follow_ups (id, invoice_id, type, scheduled, sent, opened, clicked), email_templates, payments, recurring_schedules, settings
- **Encryption:** Sensitive fields (amounts, client details, bank info) encrypted at rest using SQLCipher or application-level AES-256-GCM
- **Sync strategy:** Invoice status syncs bidirectionally; follow-up history is append-only to prevent conflicts across devices
- **Audit log:** Every action (email sent, status change, payment recorded, invoice edited) logged with timestamp for compliance and dispute resolution

---

## Framework Implementations

### Electron

**Architecture:** Main process handles email sending (nodemailer), PDF generation, notification scheduling, and system tray. Renderer process (React) provides the invoice dashboard, email composer, and PDF preview. Hidden background process for follow-up scheduler that runs independently of UI.

**Tech stack:**
- Renderer: React 18, TipTap (rich text email editor), react-pdf (inline preview), TailwindCSS, Zustand
- Main: nodemailer (SMTP), pdfmake or puppeteer (PDF generation from HTML templates), node-cron (follow-up scheduler)
- Database: better-sqlite3 with sqlcipher bindings (encryption)
- Preview: react-pdf for inline PDF viewing with zoom and page navigation
- OCR: tesseract.js for invoice scanning and field extraction

**Native module needs:** better-sqlite3, keytar (system keychain for SMTP credentials), node-notifier

**Bundle size:** ~200-270 MB (Chromium + PDF rendering + email libraries)

**Memory:** ~260-420 MB (TipTap editor + PDF preview + email queue manager)

**Pros:**
- TipTap is the best rich text email editor — supports templates, variables, HTML email output, and email client preview modes
- react-pdf provides excellent inline PDF preview with zoom, search, and page navigation
- nodemailer is the most mature Node.js email library with extensive SMTP support, OAuth2, and delivery tracking
- Full SaaS dashboard reuse for invoice management views — minimal new UI development
- puppeteer (bundled Chromium) can render pixel-perfect PDF invoices from HTML/CSS templates with no design limitations

**Cons:**
- Heaviest bundle — PDF generation via puppeteer adds another Chromium instance (~80 MB)
- Memory-intensive for what is often a background follow-up scheduling tool
- Storing financial data in an Electron app raises security perception concerns (though actual security can be solid with sqlcipher)
- nodemailer SMTP connections can be flaky with certain providers; error handling and retry logic is complex
- Two Chromium instances (main app + puppeteer) compete for memory

### Tauri

**Architecture:** Rust backend handles PDF generation (printpdf), email sending (lettre crate), follow-up scheduling (tokio timers), encryption (aes-gcm crate), OCR, and system tray. Svelte frontend for invoice management, email composer, and dashboard.

**Tech stack:**
- Frontend: Svelte 5, Tiptap-lite or custom rich text editor, TailwindCSS, svelte-pdf (preview)
- Backend: lettre (SMTP email with TLS), printpdf (PDF generation), aes-gcm (encryption), rusqlite, tokio, tesseract-rs (OCR)
- Keychain: tauri-plugin-stronghold or system keychain access for SMTP credentials

**Plugin needs:** tauri-plugin-notification, tauri-plugin-dialog, tauri-plugin-fs, tauri-plugin-stronghold (secure storage)

**Bundle size:** ~8-16 MB

**Memory:** ~40-80 MB (spikes during PDF generation or OCR processing)

**Pros:**
- lettre crate provides robust, async SMTP email sending with TLS, DKIM signing, and OAuth2 auth
- printpdf generates PDFs natively without headless browser overhead — single-digit millisecond generation
- aes-gcm encryption in Rust is fast, auditable, and uses constant-time operations — better security story than JS-based encryption
- tauri-plugin-stronghold provides encrypted storage for SMTP credentials and API keys using military-grade encryption
- Tiny footprint for what is often a background scheduler — users forget it is running

**Cons:**
- Rich text email editor options in Svelte/web are less polished than TipTap-in-React (Svelte TipTap wrappers exist but are thinner)
- PDF template design requires custom Rust code (no visual PDF template builder or HTML-to-PDF conversion)
- Invoice PDF layouts are more complex to build in printpdf vs. HTML-to-PDF tools — tables, logos, and alignment require manual positioning
- OCR via tesseract-rs requires bundling Tesseract binaries (~15 MB) or assuming system installation

### Flutter Desktop

**Architecture:** Flutter UI with Riverpod state. Platform channels for email sending and system keychain access. Rust FFI for PDF generation and encryption. WebView for email HTML preview. Native print dialog integration.

**Tech stack:**
- UI: Flutter 3.x, Riverpod, flutter_quill (rich text editor), printing (system print dialog), syncfusion_flutter_pdf
- Database: drift with SQLCipher FFI for encrypted storage
- Email: flutter_rust_bridge -> lettre, or platform channel -> system email client
- PDF: syncfusion_flutter_pdf or pdf package for generation, pdfx for preview

**Plugin needs:** tray_manager, local_notifier, file_picker, printing, window_manager

**Bundle size:** ~30-42 MB

**Memory:** ~100-170 MB

**Pros:**
- syncfusion_flutter_pdf provides comprehensive PDF generation with tables, images, barcodes, and digital signatures
- flutter_quill is a capable rich text editor for email composition with extensible toolbar
- Printing package enables direct system print dialog for invoices — users can print without exporting first
- Could extend to mobile app for checking invoice status on-the-go and receiving payment notifications
- Consistent UI across platforms reduces design effort

**Cons:**
- Email sending requires FFI bridge or platform channels — no native Dart SMTP library suitable for desktop
- syncfusion_flutter_pdf has licensing costs for commercial use (community license available for small revenue)
- System tray support is less mature than Electron or Tauri — no badge counts on all platforms
- Encryption at the database level requires SQLCipher FFI integration adding native build complexity
- flutter_quill HTML export is less reliable than TipTap for email-compatible HTML

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI views with @Observable models. PDFKit for native PDF generation and preview. NSSharingService for email integration. Keychain Services for credential storage. SwiftData with encrypted store for financial data.

**Tech stack:**
- UI: SwiftUI, native NSTextView (rich text editor with HTML output), PDFKit (PDF rendering and generation)
- Data: SwiftData with encrypted store via FileProtection
- Email: NSSharingService (system email client integration), or SwiftSMTP for direct SMTP sending
- Security: Keychain Services for credentials, CryptoKit (AES-GCM encryption) for data
- Notifications: UNUserNotificationCenter with scheduled triggers and custom categories

**Bundle size:** ~8-12 MB

**Memory:** ~30-55 MB

**Pros:**
- PDFKit provides the most polished native PDF generation and preview experience with annotation support
- Keychain Services is the gold standard for secure credential storage — hardware-backed on Apple Silicon
- CryptoKit offers hardware-accelerated encryption on Apple Silicon via Secure Enclave
- NSSharingService integrates with user's configured email client (Mail, Outlook, Spark) — no SMTP config needed
- Scheduled UNUserNotificationCenter triggers work even when the app is backgrounded or terminated
- Share Extension: forward invoice PDFs from Mail.app directly into InvoiceGhost for tracking
- Quick Look integration: preview InvoiceGhost invoices from Finder

**Cons:**
- macOS only — excludes Windows users (many small businesses and freelancers are Windows-based)
- SwiftSMTP for direct sending is less mature than lettre or nodemailer — limited OAuth2 support
- No cross-platform code sharing for the financial logic (tax calculations, currency handling)
- Smaller ecosystem for accounting integrations (QuickBooks, Xero SDKs are primarily JS/Python)

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform for dashboard and email editor. JavaMail for SMTP. iText or Apache PDFBox for PDF generation. SQLDelight with SQLCipher for encrypted data.

**Tech stack:**
- UI: Compose Desktop, Material 3, Compose rich text editor (Halilibo or custom)
- Email: JavaMail (javax.mail / Jakarta Mail) — the most mature JVM email library
- PDF: Apache PDFBox (reading/OCR) + iText (generation with tables, images, signatures)
- Database: SQLDelight with SQLCipher driver for encryption
- Security: Java KeyStore, javax.crypto for AES encryption
- HTTP: Ktor client for payment gateway and cloud sync

**Bundle size:** ~55-80 MB (+JRE)

**Memory:** ~180-300 MB

**Pros:**
- JavaMail / Jakarta Mail is the most battle-tested email library in any ecosystem — handles SMTP edge cases, OAuth2, MIME encoding, and attachment handling that other libraries miss
- Apache PDFBox + iText is the most comprehensive PDF manipulation stack (generate, read, OCR, merge, sign, flatten, watermark)
- javax.crypto provides FIPS-compliant encryption options suitable for financial data compliance
- Could share invoice management logic with Android companion app for mobile invoice tracking
- Strong ecosystem for financial data handling (Apache Commons, JSR-354 Money API, ICU for currency formatting)

**Cons:**
- JVM overhead is excessive for a background follow-up scheduler that should run invisibly
- Compose rich text editor is less mature than TipTap or flutter_quill — limited formatting options
- iText has AGPL licensing — commercial license ($2,500+/yr) required for closed-source use
- Slow startup for a tool that should be ready instantly when a payment notification arrives
- Material 3 design may feel out of place for a financial/business application

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 7 | 8 | 9 | 7 | 9 |
| Bundle size | 240 MB | 12 MB | 36 MB | 10 MB | 70 MB |
| Memory usage | 340 MB | 60 MB | 130 MB | 40 MB | 240 MB |
| Startup time | 2.6s | 0.8s | 1.2s | 0.4s | 2.1s |
| Native feel | 5/10 | 6/10 | 6/10 | 10/10 | 5/10 |
| Offline capability | 8/10 | 9/10 | 7/10 | 8/10 | 8/10 |
| Email/PDF quality | 9/10 | 7/10 | 7/10 | 8/10 | 9/10 |
| Cross-platform | 9/10 | 9/10 | 9/10 | 2/10 | 8/10 |
| Best fit score | 8/10 | 7/10 | 6/10 | 7/10 | 6/10 |

---

## Recommended Framework

**Electron** is the best fit for InvoiceGhost Desktop.

**Rationale:** InvoiceGhost is a document-centric application where the quality of the email composer and PDF rendering directly impacts user experience and professional perception. TipTap provides the richest email editing experience with template variables and email-client preview modes, react-pdf offers excellent inline PDF preview, and puppeteer-based HTML-to-PDF rendering produces the most visually polished invoices with minimal template development effort. Unlike monitoring or developer tools where lightweight footprint is critical, InvoiceGhost users (freelancers, small business owners) are less sensitive to bundle size and more sensitive to document quality and ease of use.

**Runner-up:** Tauri, if the team can invest in building custom PDF templates with printpdf and accepts a simpler email editor. The security story (Rust encryption, Stronghold secure storage, constant-time crypto) is genuinely compelling for financial data, and the lighter footprint benefits users running the app alongside accounting software.

---

## Monetization (Desktop)

- **Free tier:** Up to 5 clients, 10 invoices/month, basic email templates, manual follow-ups only, local storage
- **Pro ($24/mo or $199/yr):** Unlimited clients and invoices, automated follow-up sequences with escalation, payment link generation (Stripe/PayPal), custom PDF templates, OCR import
- **Business ($49/mo):** Multi-user with permissions, client portal, recurring invoices, bank feed integration (Plaid), expense tracking, aging reports
- **One-time license ($199):** Perpetual desktop license with 1 year of updates — appeals strongly to freelancers and small businesses who dislike recurring SaaS costs
- **Distribution:** Direct download and Mac App Store. Windows Store for enterprise discoverability. All builds notarized and code-signed for trust — essential for financial software.

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Invoice CRUD (client management, line items, tax, discounts, payment terms), SQLite storage with encryption, PDF invoice generation from 3 bundled templates |
| 3-4 | Email composer with rich text editor, SMTP configuration and testing, follow-up email templates with variable substitution (client name, amount, due date) |
| 5-6 | Automated follow-up scheduler (gentle -> firm -> final notice on configurable day offsets), system tray with overdue count badge, native notifications for due/overdue invoices |
| 7 | Payment tracking (manual mark-as-paid, Stripe payment link generation), dashboard with outstanding/overdue/paid overview, aging report |
| 8 | Auto-update, packaging (DMG/MSI), import from CSV/QuickBooks IIF, offline email queue with retry, beta testing with freelancers |

**Post-MVP:** OCR invoice import from PDF, bank feed integration (Plaid), recurring invoices with auto-send, client payment portal, multi-currency support, digital signatures, expense tracking, QuickBooks Online sync, Xero integration
