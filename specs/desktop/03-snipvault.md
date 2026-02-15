# SnipVault Desktop — 5-Framework Comparison

## Desktop Rationale

SnipVault (AI-powered code snippet manager) is arguably the strongest desktop candidate in the B2B SaaS lineup:

- **Global hotkey capture:** Save snippets from any application (terminal, browser, IDE, PDF viewer) with a system-wide keyboard shortcut — not just from a browser extension.
- **Clipboard monitoring:** Optionally watch clipboard for code patterns and offer to save automatically.
- **Offline semantic search:** Local embedding model enables instant, private snippet search without API calls.
- **System-wide paste:** Retrieve and paste snippets into any application via a Spotlight/Alfred-style launcher.
- **Direct IDE integration:** Tighter integration with VS Code, JetBrains via local IPC rather than HTTP extension APIs.

---

## Desktop-Specific Features

- **Global hotkey (Cmd+Shift+S):** Capture selected text from any app as a new snippet
- **Spotlight-style launcher (Cmd+Shift+V):** Search and paste snippets into any active application
- **Clipboard watcher:** Detect code patterns on clipboard, offer "Save to SnipVault?" notification
- **Local embedding index:** Run a small embedding model (all-MiniLM-L6 or similar) for offline semantic search
- **File system integration:** Drag-and-drop files to extract code, watch directories for new snippets
- **IDE companion:** Local WebSocket server that IDE extensions connect to for instant snippet access
- **Syntax detection:** Auto-detect language from clipboard content using tree-sitter
- **Multi-window:** Detachable snippet viewer for side-by-side reference while coding
- **Touch Bar (Mac):** Quick access to recent snippets
- **Template expansion:** Type abbreviation + trigger key to expand snippet inline (TextExpander-style)

---

## Shared Packages

### core (Rust)

- **Snippet parser:** Language detection via tree-sitter, syntax validation, formatting
- **Search engine:** BM25 full-text search + vector similarity search on local embeddings
- **Embedding generator:** Run ONNX embedding model (all-MiniLM-L6-v2, ~80MB) for semantic vectors
- **Tag inference:** Rule-based + ML tagging (language, framework, purpose, complexity)
- **Template engine:** Snippet template expansion with variable substitution and cursor positioning
- **Clipboard analyzer:** Pattern detection to identify code vs. prose vs. URLs

### api-client (Rust)

- **Cloud sync:** Bidirectional snippet sync with DriftLog cloud (for team sharing)
- **AI API:** Cloud-based AI for advanced tagging and snippet explanation (when online)
- **Auth:** API key or OAuth2 for team workspaces
- **Conflict resolution:** Three-way merge for snippets edited on multiple devices

### data (SQLite)

- **Schema:** snippets (id, title, content, language, tags[], folder_id, embedding_vector, created, updated, synced), folders, collections, search_index (FTS5), settings
- **Vector storage:** Store float32 embeddings in BLOB column; use sqlite-vss or manual cosine similarity
- **Sync:** CRDTs for concurrent edits on team snippets; last-write-wins for personal snippets

---

## Framework Implementations

### Electron

**Architecture:** Main process handles global shortcuts (globalShortcut), clipboard monitoring, system tray, and IDE WebSocket server. Renderer (React) for snippet browser, editor, and search. Secondary hidden window for Spotlight-style quick launcher.

**Tech stack:**
- Renderer: React 18, CodeMirror 6 (syntax highlighting), Fuse.js (fuzzy search), TailwindCSS
- Main: globalShortcut, clipboard module, WebSocket server (ws), better-sqlite3
- Search: better-sqlite3 FTS5 + onnxruntime-node for embeddings
- IDE: WebSocket IPC server on localhost

**Native module needs:** better-sqlite3, onnxruntime-node, node-global-key-listener (better global shortcuts)

**Bundle size:** ~200-280 MB (Chromium + ONNX runtime + embedding model)

**Memory:** ~300-500 MB (embedding model loaded in memory)

**Pros:**
- CodeMirror 6 provides excellent multi-language syntax highlighting
- globalShortcut API is well-tested for system-wide hotkeys
- Can reuse SaaS web frontend components directly
- WebSocket server for IDE integration is trivial in Node.js

**Cons:**
- Very heavy for a utility that should feel lightweight and instant
- Spotlight-style popup has 200-500ms latency (Chromium window creation)
- Clipboard monitoring in main process is polling-based (not event-driven)
- Running ONNX in Node.js has higher overhead than native

### Tauri

**Architecture:** Rust backend handles global shortcuts, clipboard monitoring (arboard crate), embedding inference (ort crate), and system tray. Svelte frontend for snippet browser. Separate small window for quick launcher popup.

**Tech stack:**
- Frontend: Svelte 5, CodeMirror 6, TailwindCSS
- Backend: arboard (clipboard), ort (ONNX Runtime), rusqlite, tree-sitter, tokio
- Search: tantivy (Rust full-text search) + ort for vector search

**Plugin needs:** tauri-plugin-global-shortcut, tauri-plugin-clipboard-manager, tauri-plugin-notification

**Bundle size:** ~10-20 MB (+ ~80 MB embedding model, downloaded separately)

**Memory:** ~60-150 MB (embedding model loaded on demand)

**Pros:**
- **Fastest search:** tantivy + native ONNX Runtime = sub-10ms semantic search
- Clipboard monitoring via arboard is event-driven on macOS (pasteboard count polling, but efficient)
- tree-sitter runs natively in Rust — best language detection performance
- Quick launcher popup renders in <100ms (system webview is pre-loaded)
- Tiny base bundle; embedding model can be lazy-downloaded on first use

**Cons:**
- Global shortcut reliability varies across Linux desktop environments
- Separate window management for launcher popup requires careful Tauri window API usage
- Less mature ecosystem for rich text editing components

### Flutter Desktop

**Architecture:** Main window for snippet browser. Overlay window for quick launcher. Platform channels for global shortcuts and clipboard access. Rust FFI for embedding inference.

**Tech stack:**
- UI: Flutter, flutter_highlight (syntax), Riverpod, window_manager
- Database: drift
- Search: flutter_rust_bridge → tantivy + ort
- Clipboard: platform channels → native APIs

**Plugin needs:** hotkey_manager, tray_manager, window_manager, flutter_rust_bridge

**Bundle size:** ~30-45 MB (+ embedding model)

**Memory:** ~120-200 MB

**Pros:**
- Beautiful custom snippet cards with smooth animations
- Hot reload for rapid UI iteration on the launcher popup
- Could extend to mobile snippet viewer app
- Custom rendering allows unique snippet visualization (syntax heatmaps, usage graphs)

**Cons:**
- Global hotkey support (hotkey_manager) is less reliable than native implementations
- Quick launcher overlay window is complex to implement in Flutter desktop
- Text rendering for code is less crisp than native or web (though improving)
- No built-in equivalent to CodeMirror; must build or wrap

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI main window + NSPanel for quick launcher (floating, non-activating). NSEvent global monitor for hotkeys. NSPasteboard observation for clipboard. CoreML/MLX for embeddings.

**Tech stack:**
- UI: SwiftUI, native NSTextView with syntax highlighting (Highlightr)
- Data: SwiftData
- Search: SearchKit (macOS native FTS) + CoreML/MLX for vector search
- Clipboard: NSPasteboard.general.changeCount observation
- Hotkey: MASShortcut or Carbon hotkey API

**Bundle size:** ~8-15 MB (+ CoreML model ~40 MB, smaller than ONNX)

**Memory:** ~40-80 MB (CoreML manages model memory efficiently)

**Pros:**
- **Best quick launcher experience:** NSPanel with .nonactivatingPanel style is the native macOS way (like Spotlight)
- CoreML embedding inference is fastest on Apple Silicon (ANE acceleration)
- NSPasteboard observation is the proper macOS clipboard monitoring API
- Services menu integration: select text in any app → right-click → "Save to SnipVault"
- Spotlight integration: snippets searchable from macOS Spotlight
- Native keyboard shortcut handling (most reliable global hotkeys)

**Cons:**
- macOS only — excludes Windows/Linux developers
- Highlightr (syntax highlighting) covers fewer languages than CodeMirror
- No web code reuse
- SwiftData migrations less mature than SQLite tooling

### Kotlin Multiplatform

**Architecture:** Compose Desktop for UI. JNativeHook for global hotkeys. AWT for system tray and clipboard. DJL for embedding inference.

**Tech stack:**
- UI: Compose Desktop, RSyntaxTextArea (syntax highlighting via interop)
- Hotkeys: JNativeHook (global key listener)
- Database: SQLDelight
- Search: Apache Lucene (FTS) + DJL/ONNX Runtime Java (embeddings)
- Clipboard: java.awt.datatransfer.Clipboard

**Bundle size:** ~60-90 MB (+ JRE + embedding model)

**Memory:** ~200-400 MB (JVM + embedding model)

**Pros:**
- Apache Lucene is the most battle-tested search engine (powers Elasticsearch)
- JNativeHook is reliable cross-platform global hotkey library
- DJL provides good ML inference abstractions
- Could share snippet management logic with Android app

**Cons:**
- Highest memory usage — inappropriate for an always-running utility
- Quick launcher popup via Compose window has noticeable JVM-related latency
- AWT clipboard API is basic (no rich monitoring)
- Compose text editing less mature than CodeMirror

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 6 | 7 | 9 | 7 | 9 |
| Bundle size | 240 MB | 15 MB | 35 MB | 12 MB | 75 MB |
| Memory usage | 400 MB | 100 MB | 160 MB | 60 MB | 300 MB |
| Startup time | 2.5s | 0.7s | 1.2s | 0.3s | 2.2s |
| Native feel | 5/10 | 6/10 | 5/10 | 10/10 | 4/10 |
| Offline capability | 8/10 | 9/10 | 7/10 | 9/10 | 8/10 |
| Search performance | 7/10 | 9/10 | 7/10 | 9/10 | 8/10 |
| Cross-platform | 9/10 | 9/10 | 9/10 | 2/10 | 8/10 |
| Best fit score | 6/10 | 9/10 | 5/10 | 8/10 | 4/10 |

---

## Recommended Framework

**Tauri** is the best fit for SnipVault Desktop.

**Rationale:** SnipVault is a developer power tool that needs to be lightning-fast, always running, and lightweight. Developers invoke it dozens of times per day via global hotkeys — latency must be <200ms. Tauri delivers the best balance: tiny bundle that devs respect, native-speed search (tantivy + ort), reliable global shortcuts, and cross-platform coverage. The Rust backend naturally handles tree-sitter parsing and ONNX inference without FFI bridges.

**Runner-up:** Swift/SwiftUI for a Mac-exclusive version. The NSPanel quick launcher and CoreML inference on Apple Silicon would be the absolute best experience on macOS.

---

## Monetization (Desktop)

- **Free:** 100 snippets, local only, basic search
- **Pro ($8/mo or $69/yr):** Unlimited snippets, semantic search, cloud sync, team sharing
- **One-time license ($79):** Perpetual personal use, 1 year updates (appeals to devs who hate subscriptions)
- **Team ($5/user/mo):** Shared snippet libraries, permissions, audit log
- **Distribution:** Direct download + Homebrew. Skip Mac App Store (sandboxing restricts global hotkeys and clipboard monitoring).

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Snippet CRUD, SQLite storage, syntax highlighting (CodeMirror 6), language detection |
| 3-4 | Global hotkey capture, clipboard monitoring, quick launcher popup with fuzzy search |
| 5 | Full-text search (tantivy/FTS5), tag management, folder organization |
| 6 | Local embedding model integration, semantic search |
| 7 | Cloud sync, system tray, auto-start, IDE WebSocket server |
| 8 | Auto-update, packaging, import from Gist/snippets.dev, beta launch |

**Post-MVP:** Template expansion, team sharing, browser extension companion, mobile viewer
