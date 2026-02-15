# LinkShelf Desktop — 5-Framework Comparison

## Desktop Rationale

LinkShelf (shared team bookmark manager) benefits enormously from a desktop companion application:

- **Browser extension companion:** A desktop app serves as the persistent backend for LinkShelf browser extensions, providing a local API server that extensions connect to for instant bookmark saving, tagging, and search — no round-trip to the cloud needed.
- **Global hotkey to save links:** Capture the current URL from any browser (or any app containing a URL) with a system-wide keyboard shortcut, bypassing the need to install per-browser extensions.
- **System tray quick access:** One-click access to recent bookmarks, pinned links, and team collections from the system tray — faster than opening a browser tab and navigating to a bookmarks page.
- **Offline browsing cache:** Automatically cache saved pages for offline reading. Store full-page snapshots (HTML + assets) locally so bookmarked content survives link rot and is available without internet.
- **Automatic screenshot and preview capture:** Desktop apps can programmatically capture page screenshots and extract metadata (title, description, Open Graph images) for rich bookmark previews without relying on a cloud rendering service.

---

## Desktop-Specific Features

- **Global hotkey (Cmd+Shift+B):** Save the current URL from the frontmost browser with auto-detected title and description
- **System tray quick-access panel:** Searchable dropdown showing recent bookmarks, pinned links, and team collections
- **Browser companion server:** Local HTTP/WebSocket server on localhost that browser extensions connect to for seamless save and search
- **Offline page cache:** Monolith-style single-file page archiving (HTML + inline CSS/images) for offline reading
- **Screenshot capture:** Headless browser (webview-based) captures full-page screenshots for visual bookmark previews
- **Smart clipboard detection:** Detect URLs on clipboard and offer to save them via a subtle notification
- **Drag-and-drop from browser:** Drag a URL from the browser address bar or a link from any page directly onto the tray icon or app window
- **Spotlight / Windows Search integration:** Bookmarks searchable from OS-level search (Spotlight on macOS, Windows Search)
- **Reading list widget:** Desktop widget showing a curated reading queue with estimated read times
- **Quick preview:** Hover over a bookmark in the tray panel to see a thumbnail preview and page excerpt

---

## Shared Packages

### core (Rust)

- **Bookmark parser:** Extract title, description, favicon, Open Graph metadata, and canonical URL from HTML content
- **Tag engine:** Auto-tagging based on URL patterns, content analysis, and user-defined rules; hierarchical tag taxonomy
- **Search engine:** Full-text search over bookmark titles, descriptions, tags, and cached page content with relevance scoring
- **Archive engine:** HTML content cleaning and single-file archiving (inline CSS, images as data URIs, remove scripts) for offline reading
- **Deduplication:** Detect and merge duplicate bookmarks across team members with URL normalization (strip tracking params, resolve redirects)
- **Import/export:** Parse bookmark files from Chrome, Firefox, Safari, Pocket, Raindrop.io, and Pinboard formats

### api-client (Rust)

- **Cloud sync:** Bidirectional bookmark sync with LinkShelf cloud; real-time team updates via WebSocket
- **Extension server:** Local HTTP + WebSocket server (localhost:XXXXX) providing REST API for browser extensions to save, search, and retrieve bookmarks
- **Auth:** JWT team auth with invite codes; OAuth2 for Google/GitHub SSO
- **Screenshot service:** Optional cloud fallback for page screenshots when local capture fails (e.g., JavaScript-heavy SPAs)

### data (SQLite)

- **Schema:** bookmarks (id, url, title, description, favicon_path, screenshot_path, tags[], collection_id, cached_html_path, created, updated, synced), collections, tags, team_shares, reading_queue, archive_cache
- **Full-text search:** FTS5 index on title, description, tags, and cached page text for instant bookmark search
- **Sync strategy:** CRDTs for team collections (concurrent bookmark adds are always merged); last-write-wins for individual bookmark metadata edits
- **Archive storage:** Cached HTML files stored on disk with path references in SQLite; configurable cache size limit with LRU eviction

---

## Framework Implementations

### Electron

**Architecture:** Main process handles the local extension server (Express/Fastify on localhost), screenshot capture (via built-in BrowserWindow), clipboard monitoring, and system tray. Renderer process (React) for bookmark manager, collection browser, and settings.

**Tech stack:**
- Renderer: React 18, TailwindCSS, Zustand, react-virtuoso (bookmark list), Masonry layout (visual grid view)
- Main: Fastify (local extension API server), BrowserWindow (headless screenshot capture), clipboard module, globalShortcut
- Database: better-sqlite3 with Drizzle ORM
- Archive: readability (Mozilla's content extraction) + inline-assets

**Native module needs:** better-sqlite3, keytar (credentials)

**Bundle size:** ~170-220 MB

**Memory:** ~220-320 MB (higher with page cache and screenshot thumbnails loaded)

**Pros:**
- Built-in BrowserWindow provides the best page screenshot capability — full Chromium rendering for accurate captures
- Fastify in Node.js makes the extension companion server trivial to implement
- readability (Mozilla) is the gold standard for content extraction and clean reading view
- Can render cached offline pages in a BrowserView with full fidelity

**Cons:**
- Massive bundle for a utility that primarily lives in the system tray
- Background Chromium process for a bookmark manager is excessive
- Screenshot capture via BrowserWindow is memory-heavy (each capture spawns a renderer)
- Battery drain from keeping full Chromium alive for tray access

### Tauri

**Architecture:** Rust backend handles the extension companion server (axum on localhost), metadata extraction, archive creation, and system tray. Svelte frontend for bookmark browser, collection manager, and tray popup.

**Tech stack:**
- Frontend: Svelte 5, TailwindCSS, virtual scrolling (svelte-virtual-list)
- Backend: axum (local HTTP server for extensions), reqwest (metadata fetching), scraper (HTML parsing), rusqlite, tokio
- Archive: monolith-rs (single-file HTML archiving) or custom implementation with scraper + base64 inlining
- Screenshot: webview screenshot API or headless wry (limited)

**Plugin needs:** tauri-plugin-global-shortcut, tauri-plugin-clipboard-manager, tauri-plugin-notification, tauri-plugin-positioner, tauri-plugin-autostart

**Bundle size:** ~5-10 MB

**Memory:** ~25-45 MB (idle tray mode); ~60-90 MB (bookmark manager open with thumbnails)

**Pros:**
- Ultra-lightweight for an always-running tray utility — 25 MB idle is imperceptible
- axum provides a high-performance local server for browser extension communication
- scraper crate efficiently parses HTML metadata without spawning a full browser
- Fast tray popup with instant search (<100ms) feels native
- Minimal battery impact for background operation with clipboard monitoring

**Cons:**
- No built-in screenshot capture equivalent to Electron's BrowserWindow — limited to webview-based approaches or cloud fallback
- Offline cached pages rendered in system webview may have inconsistent fidelity across OS versions
- Less mature HTML archiving in Rust (monolith-rs exists but less proven than Mozilla's readability)
- Extension server requires manual CORS and security configuration

### Flutter Desktop

**Architecture:** Flutter main window for bookmark manager. Platform channels for system tray and global hotkeys. Background isolate for metadata fetching and archive creation. Local HTTP server via shelf package.

**Tech stack:**
- UI: Flutter 3.x, Riverpod, cached_network_image (thumbnails), flutter_staggered_grid_view (masonry layout)
- Database: drift (SQLite)
- Server: shelf (Dart HTTP server for extension API)
- Metadata: html package (HTML parsing), http (fetching)
- Screenshot: platform channel to native screenshot APIs

**Plugin needs:** tray_manager, hotkey_manager, local_notifier, window_manager, launch_at_startup, url_launcher

**Bundle size:** ~22-30 MB

**Memory:** ~80-140 MB

**Pros:**
- Beautiful masonry grid layout for visual bookmark browsing with smooth animations
- cached_network_image provides efficient thumbnail loading and caching
- Could extend to mobile app for saving and browsing bookmarks on the go
- shelf (Dart HTTP server) is lightweight for the extension companion server

**Cons:**
- tray_manager and hotkey_manager are less reliable than native implementations
- No built-in headless browser for screenshot capture — requires platform-specific solutions
- shelf HTTP server is less performant than axum or Fastify for the extension API
- Background metadata fetching in isolates adds complexity for HTML parsing

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI main window with NSStatusItem for menu bar quick access. WKWebView for page screenshots and offline cache rendering. NWListener for local extension companion server. Spotlight indexing via CoreSpotlight.

**Tech stack:**
- UI: SwiftUI, QuickLook (page previews), NSStatusItem + NSPopover (tray panel)
- Data: SwiftData
- Server: NWListener (Network framework) for local extension API
- Screenshot: WKWebView snapshot API
- Archive: WKWebView + WebArchive format (native macOS page archiving)
- Search: CoreSpotlight for OS-level bookmark searchability

**Bundle size:** ~5-9 MB

**Memory:** ~20-40 MB (idle); ~50-80 MB (manager open with previews)

**Pros:**
- **Best archive and offline reading:** WebArchive is macOS's native page archiving format — perfect fidelity
- CoreSpotlight integration makes bookmarks searchable from macOS Spotlight
- WKWebView snapshot provides high-quality page screenshots with minimal overhead
- NSStatusItem + NSPopover is the most polished tray experience on macOS
- QuickLook integration enables instant bookmark previews with spacebar
- Safari extension can communicate directly with the desktop app via App Groups

**Cons:**
- macOS only — no Windows/Linux support (excludes most teams)
- NWListener for the extension server is less ergonomic than Express/axum for REST APIs
- WebArchive format is macOS-proprietary (not portable to other platforms)
- Only works with Safari extension natively; Chrome/Firefox extensions need the HTTP server approach

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform for bookmark browser. AWT SystemTray for tray icon. Ktor for local extension companion server. JSoup for HTML parsing and metadata extraction.

**Tech stack:**
- UI: Compose Desktop, Material 3, Coil (image loading for thumbnails)
- Tray: java.awt.SystemTray
- Server: Ktor (embedded server for extension API)
- HTML: JSoup (parsing, cleaning, metadata extraction)
- Database: SQLDelight
- Screenshot: JavaFX WebView.snapshot() or Playwright Java

**Bundle size:** ~45-65 MB (+JRE)

**Memory:** ~150-240 MB

**Pros:**
- JSoup is the most mature HTML parsing library — handles malformed HTML gracefully, excellent metadata extraction
- Ktor embedded server is feature-rich for the extension companion API (routing, content negotiation, WebSocket)
- Coil image loading provides efficient thumbnail caching with disk and memory cache
- Could share bookmark management logic with Android companion app

**Cons:**
- AWT SystemTray provides a poor quick-access experience (no rich popup panel)
- JVM memory overhead is excessive for a lightweight utility app
- JavaFX WebView for screenshots adds significant complexity and bundle size
- Slow startup means delayed extension server availability after reboot

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 6 | 7 | 9 | 7 | 8 |
| Bundle size | 195 MB | 8 MB | 26 MB | 7 MB | 55 MB |
| Memory usage | 270 MB | 35 MB | 110 MB | 30 MB | 195 MB |
| Startup time | 2.5s | 0.6s | 1.1s | 0.3s | 1.9s |
| Native feel | 5/10 | 7/10 | 6/10 | 10/10 | 4/10 |
| Offline capability | 9/10 | 7/10 | 6/10 | 9/10 | 7/10 |
| Screenshot quality | 10/10 | 5/10 | 4/10 | 8/10 | 6/10 |
| Extension companion | 8/10 | 9/10 | 6/10 | 6/10 | 8/10 |
| Cross-platform | 9/10 | 9/10 | 9/10 | 2/10 | 8/10 |
| Best fit score | 6/10 | 9/10 | 5/10 | 7/10 | 4/10 |

---

## Recommended Framework

**Tauri** is the best fit for LinkShelf Desktop.

**Rationale:** LinkShelf is a lightweight utility that lives in the system tray 90% of the time, serving as a companion to browser extensions and a quick-access point for bookmarks. Tauri's 25 MB idle footprint is ideal for an always-running app that users barely notice. The Rust backend with axum provides a high-performance local server for browser extension communication, and the fast tray popup with instant search delivers the snappy experience users expect from a bookmark utility. The screenshot capture trade-off (no built-in BrowserWindow) is acceptable since cloud fallback or simpler Open Graph image extraction covers most use cases.

**Runner-up:** Electron if full-fidelity page screenshots and offline archive rendering are core requirements. The built-in BrowserWindow for headless rendering and Mozilla's readability for content extraction are unmatched, but the 270 MB memory footprint is hard to justify for a bookmark manager.

---

## Monetization (Desktop)

- **Free tier:** 500 bookmarks, local-only storage, basic tagging, single-browser extension support
- **Pro ($6/mo or $49/yr):** Unlimited bookmarks, offline page cache, full-text search across cached content, multi-browser extension sync
- **Team ($4/user/mo):** Shared collections, team tag taxonomy, bookmark assignments, activity feed
- **One-time license ($59):** Perpetual personal license with 1 year of updates (appeals to users tired of bookmark SaaS churn)
- **Distribution:** Direct download + browser extension stores (Chrome Web Store, Firefox Add-ons, Safari Extensions Gallery)

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Bookmark CRUD, URL metadata extraction (title, description, favicon), SQLite persistence, basic tag management |
| 3-4 | System tray quick-access panel with search, global hotkey to save current URL, clipboard URL detection |
| 5-6 | Local extension companion server (HTTP + WebSocket), Chrome extension integration, real-time sync between extension and desktop |
| 7 | Offline page archiving, full-text search across cached content, collection management, import from Chrome/Firefox |
| 8 | Auto-update, packaging (DMG/MSI/AppImage), auto-start at login, cloud sync for team features, beta launch |

**Post-MVP:** Firefox and Safari extensions, page screenshot capture, reading queue with estimated read times, team shared collections, Spotlight/Windows Search integration, smart auto-tagging, mobile companion app
