# ReviewRadar Desktop — 5-Framework Comparison

## Desktop Rationale

ReviewRadar (multi-platform review monitoring and response tool) is a natural fit for a desktop application:

- **Real-time review alerts:** System tray notifications the instant a new review appears on Google, Yelp, G2, Capterra, or the App Store — no browser polling needed. Response time to negative reviews directly impacts brand perception.
- **System tray monitoring dashboard:** A persistent tray icon with badge count of unresponded reviews lets customer success teams stay on top of incoming feedback without keeping a browser tab open all day.
- **Offline review drafting:** Compose thoughtful, polished responses to reviews while offline (flights, commutes), with responses queued for posting when connectivity returns.
- **Sentiment dashboard with native performance:** Real-time sentiment analysis visualizations, trend charts, and heatmaps render smoothly with native graphics — no browser performance ceiling on large review datasets.
- **Cross-platform review aggregation:** A single desktop window aggregates reviews from 10+ platforms with unified search, filtering, and response management — faster than switching between browser tabs.

---

## Desktop-Specific Features

- **System tray with review count:** Badge showing unresponded review count; color-coded (red for negative, yellow for neutral, green for positive)
- **Native push notifications:** Rich notifications showing star rating, platform, and review snippet — click to open response composer
- **Notification sound customization:** Different alert tones for 1-star vs. 5-star reviews; configurable urgency levels
- **Global hotkey (Cmd+Shift+R):** Instantly open the review response composer from any application
- **Desktop widgets:** macOS widget / Windows widget showing daily review summary and average rating trend
- **Offline response queue:** Draft and queue responses offline; batch-submit when reconnected with review-by-review confirmation
- **Screenshot capture:** Automatically capture review screenshots for documentation and dispute evidence
- **Multi-monitor support:** Detachable sentiment dashboard for a dedicated monitoring screen
- **Auto-start at login:** Begin monitoring immediately when the workstation boots; no manual launch needed
- **Scheduled digest:** Generate and display a daily review summary report at a configured time each morning

---

## Shared Packages

### core (Rust)

- **Sentiment analyzer:** Rule-based + ML sentiment scoring (positive/neutral/negative with confidence), aspect extraction (service, product, pricing, support)
- **Review normalizer:** Unified review struct from disparate platform APIs (Google, Yelp, G2, App Store, Trustpilot) with consistent star mapping and metadata
- **Response generator:** AI-powered response drafting engine with tone control (professional, friendly, apologetic), template variable substitution, and brand voice consistency
- **Alert engine:** Configurable rules for notification triggers (star threshold, keyword detection, sentiment score, platform priority)
- **Analytics calculator:** Aggregate metrics (average rating, response rate, sentiment trend, review velocity) with configurable time windows

### api-client (Rust)

- **Platform connectors:** REST clients for Google Business Profile API, Yelp Fusion API, G2 API, Apple App Store Connect, Trustpilot Business API, Capterra API
- **Review poller:** Efficient polling with ETags/last-modified for platforms without webhooks; webhook receiver for platforms that support them
- **Response poster:** Submit review responses back to each platform with platform-specific formatting and character limits
- **Auth:** OAuth2 flows for Google/Yelp, API key management for G2/Trustpilot, App Store Connect JWT
- **Cloud sync:** Push review data and analytics to ReviewRadar cloud for team dashboards; pull team response assignments

### data (SQLite)

- **Schema:** reviews (id, platform, rating, text, author, date, sentiment_score, response_status, response_text), platforms, notification_rules, response_templates, analytics_cache, sync_queue
- **Full-text search:** FTS5 index on review text and response text for instant search across all platforms
- **Sync strategy:** Reviews are append-only from platforms (no conflict); responses use optimistic locking with server-side arbitration for team assignments

---

## Framework Implementations

### Electron

**Architecture:** Main process handles review polling (scheduled HTTP requests), system tray management, notification dispatch, and background analytics computation. Renderer process (React) provides the review dashboard, sentiment charts, and response composer.

**Tech stack:**
- Renderer: React 18, Recharts (sentiment trend charts), TailwindCSS, Zustand, react-virtuoso (virtualized review list for large datasets)
- Main: node-cron (poll scheduling), electron-store (settings), Notification API
- Database: better-sqlite3 with Drizzle ORM
- AI: OpenAI SDK for response generation

**Native module needs:** better-sqlite3, electron-positioner (tray popup), node-notifier (richer notifications on Linux)

**Bundle size:** ~170-220 MB

**Memory:** ~220-350 MB (higher with large review datasets and charts rendered)

**Pros:**
- Recharts/D3 ecosystem provides the richest sentiment visualization options
- react-virtuoso handles 100K+ review lists with smooth scrolling
- Full SaaS web frontend code reuse for dashboard components
- Rich notification support with custom HTML tray popups

**Cons:**
- Heavy resource usage for what is primarily a monitoring/notification tool
- Background polling keeps full Chromium alive, draining battery on laptops
- Tray popup latency (200-400ms) feels sluggish for quick review checks
- Over-engineered for a tool that spends 95% of its time minimized to tray

### Tauri

**Architecture:** Rust backend handles all review polling (reqwest + tokio intervals), sentiment analysis, notification dispatch, and system tray. Svelte frontend for the review dashboard, response composer, and settings.

**Tech stack:**
- Frontend: Svelte 5, Chart.js (sentiment charts), TailwindCSS, virtual list (svelte-virtual-list)
- Backend: reqwest (HTTP polling), rusqlite, tokio (async scheduling), notify-rust (native notifications)
- Sentiment: rust-bert or candle for local sentiment analysis; reqwest to cloud AI APIs

**Plugin needs:** tauri-plugin-notification, tauri-plugin-positioner, tauri-plugin-autostart, tauri-plugin-store

**Bundle size:** ~6-12 MB

**Memory:** ~30-55 MB (idle monitoring mode); ~80-120 MB (dashboard active with charts)

**Pros:**
- Ultra-lightweight for a monitoring tool — 30 MB idle is perfect for always-running tray apps
- Rust async polling with tokio is extremely efficient for concurrent multi-platform API requests
- Minimal battery drain during background monitoring (no Chromium process)
- Fast tray popup for quick review glances (<100ms render)
- Native notification integration feels like a first-class OS experience

**Cons:**
- Chart.js is less feature-rich than D3/Recharts for complex sentiment visualizations
- System webview rendering may vary slightly across OS versions for the dashboard
- Smaller ecosystem for rich text editing (response composer is simpler)
- Less mature virtual scrolling for very large review lists

### Flutter Desktop

**Architecture:** Main window for review dashboard. Platform channels for native notifications and system tray. Riverpod for state management. Background isolate for review polling.

**Tech stack:**
- UI: Flutter 3.x, fl_chart (sentiment visualizations), Riverpod, flutter_quill (response editor)
- Database: drift (SQLite)
- HTTP: dio (API polling)
- Notifications: local_notifier

**Plugin needs:** tray_manager, local_notifier, window_manager, launch_at_startup

**Bundle size:** ~24-32 MB

**Memory:** ~90-160 MB

**Pros:**
- fl_chart provides animated, interactive sentiment trend charts with smooth transitions
- Background isolate keeps UI responsive during heavy analytics computation
- Could extend to mobile app for on-the-go review monitoring and quick responses
- Custom rendering allows branded review cards matching each platform's visual identity

**Cons:**
- tray_manager is less mature — limited tray popup customization
- Background polling in Flutter desktop requires careful isolate management
- Rich text response composer (flutter_quill) less polished than web alternatives
- Higher memory usage than Tauri for background monitoring

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI main window with @Observable view models. NSStatusItem for menu bar monitoring. BackgroundTasks framework for scheduled polling. UserNotifications with rich notification content.

**Tech stack:**
- UI: SwiftUI, Charts framework (native), NSStatusItem + NSPopover
- Data: SwiftData
- HTTP: URLSession with background configuration for polling
- Notifications: UNUserNotificationCenter with UNNotificationContentExtension (rich previews)
- AI: URLSession to cloud APIs for response generation

**Bundle size:** ~6-10 MB

**Memory:** ~20-40 MB (idle); ~60-90 MB (dashboard active)

**Pros:**
- **Best notification experience:** Rich notification extensions show star rating, review snippet, and quick-response actions directly in Notification Center
- NSStatusItem provides the most polished system tray experience on macOS
- Native Charts framework renders sentiment trends with minimal memory overhead
- WidgetKit integration shows daily review summary on macOS desktop widgets
- Background URLSession polling works even when app is in background/suspended
- Lowest battery impact for always-on monitoring

**Cons:**
- macOS only — excludes Windows users (many customer success teams use Windows)
- No cross-platform code sharing with potential Android/Windows clients
- Smaller community for enterprise-grade charting customization
- Limited to Apple ecosystem for push notification relay

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform for dashboard UI. AWT SystemTray for tray icon. ViewModel + StateFlow for reactive state. Ktor client for API polling. Coroutines for background scheduling.

**Tech stack:**
- UI: Compose Desktop, Material 3, Vico (charting library for Compose)
- Tray: java.awt.SystemTray + TrayIcon
- Database: SQLDelight
- HTTP: Ktor client with coroutine-based polling
- Notifications: java.awt.TrayIcon.displayMessage (basic) or JNA to native APIs

**Bundle size:** ~50-70 MB (+JRE)

**Memory:** ~160-270 MB

**Pros:**
- Ktor + coroutines provide elegant concurrent polling across multiple review platforms
- Vico charting library integrates natively with Compose for smooth sentiment charts
- SQLDelight cross-platform persistence could share data layer with Android app
- Strong HTTP client ecosystem (Ktor, OkHttp) for reliable API polling

**Cons:**
- AWT tray notifications are visually ugly and lack rich content (no star ratings, no quick actions)
- JVM memory overhead is wasteful for a background monitoring tool
- Slow startup (~2s) means delayed monitoring if app restarts
- Compose Desktop text input for response drafting is less polished than web editors

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 6 | 7 | 8 | 6 | 8 |
| Bundle size | 195 MB | 9 MB | 28 MB | 8 MB | 60 MB |
| Memory usage | 280 MB | 45 MB | 125 MB | 30 MB | 210 MB |
| Startup time | 2.5s | 0.7s | 1.1s | 0.3s | 2.0s |
| Native feel | 5/10 | 7/10 | 6/10 | 10/10 | 4/10 |
| Offline capability | 7/10 | 9/10 | 7/10 | 8/10 | 7/10 |
| Notification quality | 6/10 | 8/10 | 5/10 | 10/10 | 3/10 |
| Real-time monitoring | 7/10 | 9/10 | 7/10 | 8/10 | 7/10 |
| Cross-platform | 9/10 | 9/10 | 9/10 | 2/10 | 8/10 |
| Best fit score | 6/10 | 9/10 | 5/10 | 7/10 | 4/10 |

---

## Recommended Framework

**Tauri** is the best fit for ReviewRadar Desktop.

**Rationale:** ReviewRadar is fundamentally a monitoring tool that runs in the system tray 95% of the time, periodically polling review APIs and dispatching notifications. Resource efficiency is paramount — Tauri's 45 MB idle memory versus Electron's 280 MB is the difference between a tool users leave running and one they quit to save battery. The Rust backend with tokio provides the most efficient concurrent API polling, and native notifications via notify-rust feel like first-class OS alerts.

**Runner-up:** Swift/SwiftUI if targeting macOS-only deployments. The rich notification extensions with inline review responses and WidgetKit integration provide the best single-platform experience, but most customer success teams need cross-platform support.

---

## Monetization (Desktop)

- **Free tier:** Monitor 1 platform (Google only), 50 reviews/month, basic notifications
- **Pro ($24/mo):** All platforms, unlimited reviews, sentiment analytics, AI response drafts, offline mode
- **Team ($12/user/mo):** Shared review queue, response assignments, team analytics, approval workflows
- **Agency ($49/mo per client):** Multi-location monitoring, white-label reports, client-specific response templates
- **Distribution:** Direct download for all platforms; Mac App Store for enterprise MDM deployment

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Review ingestion from Google Business Profile and Yelp APIs, unified review data model, SQLite persistence |
| 3-4 | System tray with unresponded count, native notifications for new reviews, review list UI with filtering |
| 5-6 | Sentiment analysis (cloud API), response composer with AI-drafted suggestions, response posting back to platforms |
| 7 | Sentiment trend dashboard, analytics charts, offline response queue, auto-start at login |
| 8 | Auto-update, packaging (DMG/MSI/AppImage), notification customization settings, beta launch |

**Post-MVP:** Additional platforms (G2, Capterra, Trustpilot, App Store), team response assignments, desktop widgets, screenshot capture, scheduled digest reports, local sentiment model
