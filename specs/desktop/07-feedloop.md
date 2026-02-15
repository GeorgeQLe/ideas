# FeedLoop Desktop — 5-Framework Comparison

## Desktop Rationale

FeedLoop (multi-channel feedback aggregator) is a strong desktop candidate due to its cross-application capture needs:

- **System tray quick capture:** One-click or global hotkey feedback capture from any context — no need to switch to a browser tab. Capture feedback while on a customer call, reading support email, or browsing Slack. The tray icon serves as a persistent inbox for incoming feedback.
- **Aggregate feedback offline:** Collect, tag, and organize feedback from multiple channels without internet. Process and categorize feedback during flights, commutes, or network outages. All data is available locally for analysis even when disconnected.
- **Local sentiment analysis:** Run lightweight NLP models on-device for real-time sentiment scoring without sending customer feedback to external APIs — critical for privacy-conscious teams handling PII in support conversations.
- **Screenshot annotation:** Native screenshot capture with annotation tools (arrows, highlights, blur sensitive info) for visual feedback. Integrates with OS screenshot APIs for seamless region capture without third-party tools.
- **Cross-app feedback capture:** Monitor clipboard, capture selections from any app, or use global hotkeys to pipe content from Slack, email, Zendesk, Intercom, or any other app directly into FeedLoop with source attribution.

---

## Desktop-Specific Features

- **Global hotkey capture (Cmd+Shift+K):** Capture selected text from any application as a new feedback item with automatic source app detection
- **System tray quick-add:** Click tray icon for a compact feedback entry form — title, channel, sentiment, customer, and notes in a focused popup
- **Screenshot annotation tool:** Native screen region capture with overlay annotation (arrows, rectangles, blur, text labels, numbering) — attach to feedback items as evidence
- **Clipboard monitoring:** Detect text copied from support tools (Zendesk, Intercom, email clients) and offer to import as feedback with smart parsing
- **Local sentiment analysis:** Run a lightweight transformer model (DistilBERT or similar, ~70 MB) for on-device sentiment scoring without API calls
- **Channel watchers:** Background monitors that poll configured email folders (IMAP), Slack channels, RSS feeds, or Twitter mentions for new feedback automatically
- **Offline categorization:** Tag, merge, prioritize, and categorize feedback without internet using locally-cached taxonomy and keyboard shortcuts
- **Cross-app drag-and-drop:** Drag text, images, or files from any application window into FeedLoop's drop zone for instant feedback capture
- **Multi-window views:** Detach feedback boards, individual items, or analytics views into separate windows for multi-monitor setups
- **Audio note capture:** Record voice memos as feedback items with optional local transcription via whisper.cpp — ideal for capturing verbal feedback from calls

---

## Shared Packages

### core (Rust)

- **Feedback normalizer:** Parse feedback from multiple sources (email, Slack messages, survey responses, support tickets, app reviews) into a unified schema with source metadata preservation
- **Sentiment analyzer:** On-device sentiment scoring using ONNX model (DistilBERT fine-tuned for customer feedback). Returns score (-1.0 to 1.0), label (positive/neutral/negative), and confidence.
- **Taxonomy engine:** Hierarchical categorization with auto-suggest based on content analysis, keyword matching, and historical pattern learning
- **Deduplication:** Fuzzy matching to detect duplicate feedback across channels using TF-IDF similarity scoring and configurable threshold
- **Analytics calculator:** Compute sentiment trends, category distributions, feedback velocity, customer satisfaction scores, and NPS estimates
- **Search engine:** Full-text search with filtering by channel, sentiment range, category, date range, customer, and assignee

### api-client (Rust)

- **Channel integrations:** Slack API (real-time events), Intercom API (conversations), Zendesk API (tickets), GitHub Issues API, email IMAP client (folder watching) for pulling feedback
- **Cloud sync:** Bidirectional sync of feedback items, tags, and categories with FeedLoop cloud for team collaboration and shared boards
- **Auth:** OAuth2 for channel integrations, JWT for team auth, API keys for webhook ingestion from custom sources
- **Webhook receiver:** Local HTTP server to receive incoming webhooks from configured services (Typeform, SurveyMonkey, custom forms)
- **Export API:** Generate CSV, JSON, or PDF reports of feedback data with charts for stakeholder sharing

### data (SQLite)

- **Schema:** feedback_items (id, source, channel, content, sentiment_score, sentiment_label, category_id, tags[], customer_id, assignee_id, priority, status, created, synced), categories, customers, channels, annotations (blob for screenshot data), audio_notes, settings
- **FTS index:** Full-text search on feedback content, customer names, tags, and notes for instant retrieval across thousands of items
- **Sync strategy:** Append-only for new feedback items; category/tag changes use last-write-wins with conflict detection and manual resolution queue
- **Blob storage:** Screenshots and audio notes stored as BLOBs with zstd compression; large files (>5 MB) stored as filesystem references

---

## Framework Implementations

### Electron

**Architecture:** Main process handles clipboard monitoring, global shortcuts, channel polling (IMAP, Slack WebSocket), and system tray. Renderer process (React) for the feedback dashboard, annotation editor, and analytics views. Hidden transparent window for screenshot capture overlay. Separate hidden window for tray popup.

**Tech stack:**
- Renderer: React 18, Konva.js (canvas annotation), Recharts (analytics), TailwindCSS, Zustand
- Main: electron-screenshots (screen capture), globalShortcut, clipboard module, imapflow (email polling), @slack/web-api
- Database: better-sqlite3
- AI: onnxruntime-node for sentiment analysis
- Audio: Web Audio API + whisper.cpp via node addon for local transcription

**Native module needs:** better-sqlite3, onnxruntime-node, node-screenshots, electron-positioner

**Bundle size:** ~210-280 MB (Chromium + ONNX runtime + sentiment model)

**Memory:** ~300-480 MB (canvas annotation + sentiment model loaded + channel watchers running)

**Pros:**
- Konva.js provides the most mature canvas annotation toolkit — arrows, shapes, text, blur, freehand drawing with undo/redo
- Recharts/D3 offer the best analytics visualization for sentiment trends, category distributions, and feedback heatmaps
- imapflow handles complex IMAP scenarios (OAuth2 email authentication, folder watching, IDLE support)
- Screen capture overlay as a transparent BrowserWindow is well-documented with proven cross-platform support
- Full SaaS dashboard code reuse for feedback boards and analytics views
- @slack/web-api provides official, well-maintained Slack integration

**Cons:**
- Very heavy for a tool that should run quietly in the background monitoring channels
- Multiple hidden windows (tray popup, screenshot overlay, channel watchers) multiply memory usage beyond estimates
- Clipboard monitoring via polling in main process is inefficient (checks every 500ms vs. event-driven)
- ONNX inference in Node.js has ~2x higher overhead than native Rust inference for same model
- globalShortcut can conflict with other Electron apps using the same key combination

### Tauri

**Architecture:** Rust backend handles clipboard monitoring (arboard), sentiment analysis (ort), channel polling (async tasks), and system tray. Svelte frontend for the feedback dashboard and quick-add panel. Separate webview window with canvas overlay for screenshot annotation.

**Tech stack:**
- Frontend: Svelte 5, Fabric.js (canvas annotation), Chart.js (analytics), TailwindCSS
- Backend: arboard (clipboard), ort (ONNX sentiment model), imap-rs (email polling), rusqlite, tokio, reqwest
- Screenshot: screenshots-rs or xcap crate for cross-platform screen capture
- Audio: cpal (audio capture) + whisper-rs (local transcription)

**Plugin needs:** tauri-plugin-global-shortcut, tauri-plugin-clipboard-manager, tauri-plugin-notification, tauri-plugin-positioner

**Bundle size:** ~10-18 MB (+ ~70 MB sentiment model, downloaded on first use)

**Memory:** ~45-100 MB (spikes to ~170 MB during sentiment analysis batch processing)

**Pros:**
- **Lightest background footprint** — channel watchers and clipboard monitoring use minimal resources in Rust async tasks
- ort provides native ONNX inference for sentiment scoring without bridge overhead — 2x faster than JS
- tokio async runtime handles concurrent channel polling (IMAP, Slack, webhooks) efficiently with task scheduling
- whisper-rs for local audio transcription is the fastest non-Apple option with GGML acceleration
- Tiny base bundle; sentiment model lazy-downloaded on first use with progress indication
- arboard clipboard monitoring can detect changes efficiently via pasteboard count comparison

**Cons:**
- Canvas annotation in Fabric.js within webview is less polished than Konva.js in Electron — fewer built-in tools
- Screenshot capture overlay requires careful multi-window management with transparency (platform-specific quirks)
- IMAP client libraries in Rust (imap crate) are less mature than Node.js imapflow — limited OAuth2 support
- Smaller ecosystem for rich annotation UI components — may need custom blur and arrow tools
- Slack integration requires manual WebSocket handling vs. official SDK

### Flutter Desktop

**Architecture:** Main window for feedback dashboard and analytics. Overlay window for screenshot annotation via CustomPainter. Riverpod for reactive state. Rust FFI via flutter_rust_bridge for sentiment analysis, channel polling, and audio transcription.

**Tech stack:**
- UI: Flutter 3.x, CustomPainter (annotation canvas), fl_chart (analytics), Riverpod
- Database: drift (SQLite)
- AI: flutter_rust_bridge -> ort for sentiment scoring
- Screenshot: screen_capturer + custom transparent overlay window for region selection
- Audio: record package + flutter_rust_bridge -> whisper-rs for transcription

**Plugin needs:** tray_manager, local_notifier, hotkey_manager, window_manager, screen_capturer, desktop_drop

**Bundle size:** ~30-42 MB (+ sentiment model downloaded separately)

**Memory:** ~110-180 MB

**Pros:**
- CustomPainter provides powerful canvas rendering for annotation tools with smooth gesture input and hit testing
- fl_chart creates beautiful, animated sentiment trend visualizations with interactive tooltips
- Could extend to mobile app for capturing feedback from phone (photo annotation, voice notes from field teams)
- Hot reload speeds up iteration on the annotation UI and dashboard layouts
- desktop_drop plugin enables native drag-and-drop from other applications

**Cons:**
- Screen capture overlay is complex in Flutter desktop — transparent fullscreen window with custom compositing
- Global hotkey support (hotkey_manager) is less reliable than native implementations, especially on Linux
- Channel polling (IMAP, Slack API) requires Rust FFI bridge adding build and debug complexity
- System tray quick-add UX is limited by tray_manager capabilities — no rich popup on Windows
- CustomPainter annotation lacks pre-built tools (blur, arrow heads) — must implement from geometry primitives

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI views with NSStatusItem for system tray. ScreenCaptureKit for native screenshot capture. Markup-style overlay for annotation. CoreML for sentiment analysis on Apple Neural Engine. AVFoundation for audio capture.

**Tech stack:**
- UI: SwiftUI, PencilKit or custom NSView (annotation), Charts framework (analytics), NSStatusItem + NSPopover
- Data: SwiftData with CloudKit sync option
- Screenshot: ScreenCaptureKit (macOS 12.3+) with SCContentFilter for region selection
- AI: CoreML for sentiment analysis (ANE-accelerated on Apple Silicon — most power efficient)
- Clipboard: NSPasteboard.general.changeCount observation (event-driven, not polling)
- Audio: AVAudioRecorder + Speech framework (SFSpeechRecognizer for live transcription)

**Bundle size:** ~8-14 MB (+ CoreML model ~30 MB, can be thinned for architecture)

**Memory:** ~30-60 MB

**Pros:**
- **ScreenCaptureKit** provides the most polished screenshot capture experience on macOS with system picker UI
- CoreML sentiment analysis runs on Apple Neural Engine — fastest, most power-efficient inference available
- Native NSPasteboard observation is the proper macOS clipboard monitoring API — event-driven, zero polling overhead
- Speech framework provides excellent on-device transcription for voice notes with speaker diarization
- Services menu integration: select text in any app -> right-click -> "Add to FeedLoop" (zero-friction capture)
- Handoff: start categorizing feedback on Mac, continue on iPhone
- Focus filters: suppress feedback notifications during Focus modes

**Cons:**
- macOS only — excludes Windows/Linux team members (product and support teams are often cross-platform)
- PencilKit is primarily designed for freehand drawing, not structured annotation (arrows, numbered callouts, blur)
- No cross-platform code sharing for feedback processing logic
- ScreenCaptureKit requires macOS 12.3+ (excludes macOS 11 users)
- Smaller community building feedback/annotation tools in Swift

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform for dashboard. AWT for system tray and clipboard monitoring. Java Robot for screenshot capture. DJL for sentiment analysis. Ktor for channel integrations. Jakarta Mail for IMAP.

**Tech stack:**
- UI: Compose Desktop, Material 3, Compose Canvas (annotation overlay)
- Tray: java.awt.SystemTray with TrayIcon
- Screenshot: java.awt.Robot for screen capture + Compose Canvas overlay for annotation
- Database: SQLDelight
- AI: DJL with ONNX Runtime backend (sentiment model)
- Email: Jakarta Mail (IMAP polling with IDLE support)
- HTTP: Ktor client for Slack, Intercom, Zendesk APIs

**Bundle size:** ~60-85 MB (+JRE + sentiment model)

**Memory:** ~200-350 MB (JVM + sentiment model loaded + IMAP connections maintained)

**Pros:**
- Jakarta Mail (formerly JavaMail) is the most reliable IMAP client library in any ecosystem — handles IDLE, OAuth2, and encoding edge cases
- DJL provides comprehensive ML inference with automatic backend selection (ONNX, PyTorch, TensorFlow)
- Compose Canvas is flexible for building custom annotation tools with path operations
- Could share feedback logic with Android companion app for mobile feedback capture
- Strong ecosystem for text analysis (Stanford CoreNLP, OpenNLP, Apache Lucene for search)

**Cons:**
- Highest memory usage — inappropriate for an always-running background aggregator on most laptops
- java.awt.Robot screenshot capture is basic (full screen only, no region picker) and OS-dependent
- AWT SystemTray provides minimal tray functionality — no popup window, no badge, basic icons
- JVM startup delay (~2s) when launching quick capture from global hotkey defeats the purpose of quick capture
- AWT clipboard monitoring is polling-based and less reliable than native APIs

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 7 | 8 | 9 | 7 | 10 |
| Bundle size | 250 MB | 14 MB | 36 MB | 11 MB | 75 MB |
| Memory usage | 380 MB | 70 MB | 140 MB | 45 MB | 270 MB |
| Startup time | 2.7s | 0.7s | 1.2s | 0.4s | 2.1s |
| Native feel | 5/10 | 6/10 | 6/10 | 10/10 | 4/10 |
| Offline capability | 7/10 | 9/10 | 7/10 | 8/10 | 7/10 |
| Annotation quality | 9/10 | 6/10 | 7/10 | 8/10 | 5/10 |
| Cross-platform | 9/10 | 9/10 | 9/10 | 2/10 | 8/10 |
| Best fit score | 6/10 | 8/10 | 6/10 | 7/10 | 4/10 |

---

## Recommended Framework

**Tauri** is the best fit for FeedLoop Desktop.

**Rationale:** FeedLoop is fundamentally a background aggregator that should run quietly while monitoring multiple channels and capturing feedback on demand via system tray and global hotkeys. Tauri's minimal resource footprint (70 MB memory) is critical when the app is running alongside Slack, email clients, Zendesk, and other support tools that product and support teams use daily. The Rust backend with tokio excels at concurrent channel polling (IMAP, Slack WebSocket, webhook server) without blocking, and native ONNX inference via ort provides efficient sentiment scoring. While Electron offers better annotation tools (Konva.js), Fabric.js in Tauri's webview is sufficient for the core annotation use case, and the resource savings are decisive for an always-running background tool.

**Runner-up:** Electron, if screenshot annotation is a primary differentiator and the team wants the richest canvas editing experience with blur, arrows, and freehand drawing. Konva.js and the transparent overlay window pattern are significantly more mature in Electron. Worth choosing if annotation quality drives user acquisition.

---

## Monetization (Desktop)

- **Free tier:** 1 channel (manual entry only), 50 feedback items/month, basic sentiment analysis (rule-based), local storage only
- **Pro ($16/mo or $129/yr):** Unlimited channels, unlimited items, ML-powered sentiment analysis, multi-channel integrations (Slack, email, Zendesk), cloud sync, analytics dashboard
- **Team ($10/user/mo):** Shared feedback boards, role-based access, customer segmentation, trend alerts, export to Jira/Linear/Notion, assignment workflows
- **Enterprise ($25/user/mo):** SSO, audit log, custom integrations via API, dedicated support, on-premise deployment option, data retention policies
- **Distribution:** Direct download, Homebrew. Consider Mac App Store for non-technical product teams who discover tools there. Windows Store for enterprise IT approval workflows.

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Feedback item CRUD (manual entry, import from clipboard), SQLite storage with FTS5, categorization with tags and custom taxonomy hierarchy |
| 3-4 | System tray quick-add with global hotkey (Cmd+Shift+K), clipboard monitoring for text capture with smart parsing, basic sentiment scoring (rule-based keyword matching) |
| 5-6 | Screenshot capture with region selection and annotation (arrows, rectangles, blur, text labels), attach annotations to feedback items, drag-and-drop import from other apps |
| 7 | Analytics dashboard (sentiment trends over time, category distribution pie/bar charts, feedback velocity sparkline), full-text search with channel/sentiment/date filters |
| 8 | Auto-update, packaging (DMG/MSI/AppImage), Slack channel integration (first automated channel connector), beta testing with product teams |

**Post-MVP:** ONNX-powered sentiment model, IMAP email channel watcher, Zendesk/Intercom/GitHub integrations, audio notes with whisper.cpp transcription, team sync with shared boards, export to Jira/Linear, customer profiles with feedback history, cross-channel deduplication, NPS calculation, mobile companion app
