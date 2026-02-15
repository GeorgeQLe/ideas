# StatusPing Desktop — 5-Framework Comparison

## Desktop Rationale

StatusPing (uptime monitoring dashboard) benefits enormously from a desktop presence:

- **System tray alerts:** Instant native OS notifications when a monitored service goes down — no browser tab to keep open, no email delay. The tray icon changes color (green/yellow/red) for at-a-glance status across all monitors.
- **Persistent dashboard:** Always-running monitoring dashboard accessible from the system tray. One click to see the full status overview without navigating to a web app or logging in again.
- **Native notifications with actions:** Rich OS notifications with "Acknowledge," "Open Incident," or "Snooze" action buttons — respond to incidents without opening the full app. Critical for on-call engineers who need sub-second response paths.
- **Network diagnostics from local machine:** Run ping, traceroute, DNS lookups, and curl checks directly from the user's machine to correlate monitoring data with local network conditions. Invaluable for debugging "is it down or is it just me" scenarios.
- **Offline incident management:** Draft incident reports, update status pages, and manage on-call schedules offline. Changes sync when connectivity returns — because internet outages are exactly when you need your monitoring tool most.

---

## Desktop-Specific Features

- **System tray status indicator:** Color-coded tray icon (green = all up, yellow = degraded, red = down) with tooltip showing monitor count and summary
- **Tray popup dashboard:** Compact status overview in a tray dropdown — shows all monitors with current status, response time sparkline, and uptime percentage
- **Native alert notifications:** OS notifications with action buttons (Acknowledge, Escalate, Snooze 30m) for immediate incident response without app switching
- **Local network diagnostics:** Built-in ping, traceroute, DNS lookup, and HTTP check tools that run from the user's machine with result history
- **Background monitoring agent:** Optional local monitoring probe that supplements cloud checks — detect issues visible only from the user's network or region
- **Global hotkey (Cmd+Shift+U):** Instant status overview popup from anywhere — critical during incident response
- **Menubar mini-stats:** Display response time or uptime percentage directly in the macOS menu bar text alongside the status icon
- **Incident timeline editor:** Offline-capable rich editor for composing incident updates and post-mortems with timestamps
- **Sound alerts:** Configurable audio alerts for critical incidents (distinct from OS notification sounds) with escalation patterns
- **Multi-account support:** Monitor services across multiple StatusPing accounts or organizations from a single desktop app

---

## Shared Packages

### core (Rust)

- **Monitor engine:** HTTP/HTTPS, TCP, DNS, ICMP ping, and keyword checks with configurable intervals, timeouts, retry policies, and alert thresholds
- **Alert evaluator:** Processes check results against alert rules (threshold breaches, consecutive failures, degraded response times, SSL certificate expiry)
- **Incident manager:** State machine for incidents (detecting -> triggered -> acknowledged -> investigating -> resolved) with timeline events and auto-resolution
- **Statistics calculator:** Compute uptime percentages, P50/P95/P99 response times, rolling averages, SLA compliance, and MTTR/MTTD metrics
- **Status page renderer:** Generate static HTML status pages from current monitor states and incident history with customizable branding

### api-client (Rust)

- **Cloud sync:** WebSocket connection for real-time monitor status updates from cloud probes; REST for configuration changes
- **Notification relay:** Push incidents to Slack, PagerDuty, OpsGenie, Discord, email, and SMS via cloud API
- **Auth:** API key-based auth for monitor agents, OAuth2 for dashboard users, team management with on-call rotation
- **Webhook handler:** Receive and parse incoming webhooks from integrated services (GitHub deployments, CI/CD pipelines, Vercel)

### data (SQLite)

- **Schema:** monitors (id, name, url, type, interval, status, last_check, response_time, ssl_expiry), incidents (id, monitor_id, status, started, resolved, updates[]), alert_rules, on_call_schedules, check_results, settings
- **Time-series storage:** Check results stored with timestamp indexing for efficient range queries and charting; automatic compaction after 90 days
- **Sync strategy:** Monitors and incidents sync bidirectionally; local check results upload in batches every 60 seconds to minimize bandwidth

---

## Framework Implementations

### Electron

**Architecture:** Main process runs background check scheduler, system tray management, and notification dispatch. Renderer process (React) provides the full monitoring dashboard with real-time charts and incident management. Hidden renderer for tray popup with compact status view.

**Tech stack:**
- Renderer: React 18, Recharts (response time graphs), TailwindCSS, Zustand, date-fns
- Main: node-cron (check scheduler), electron-tray, node-notifier, net module (TCP checks)
- Database: better-sqlite3
- Network: axios (HTTP checks), raw-socket (ICMP), dns module (DNS checks), tls module (SSL checks)

**Native module needs:** better-sqlite3, raw-socket (for ICMP ping), electron-positioner

**Bundle size:** ~170-220 MB

**Memory:** ~230-380 MB (real-time charts with streaming data are memory-intensive)

**Pros:**
- Recharts/D3 provide the best real-time response time charts, sparklines, and heatmaps
- Node.js net/dns/tls modules handle network diagnostics natively without additional dependencies
- Full SaaS dashboard code can be reused directly for the monitoring views
- Tray popup as a mini web app allows rich status visualization with CSS animations
- WebSocket client in Node.js is mature for real-time cloud sync

**Cons:**
- 230+ MB RAM for a monitoring tray app is wasteful — this app should be invisible in the system
- Background Chromium process drains battery on laptops during on-call shifts (exactly when battery matters)
- ICMP ping requires raw-socket native module with elevated permissions on some systems
- Real-time chart rendering consumes CPU even when the dashboard is hidden behind other windows
- Notification delivery can be delayed by Chromium's event loop under load

### Tauri

**Architecture:** Rust backend handles all monitoring checks (reqwest for HTTP, trust-dns for DNS, surge-ping for ICMP), alert evaluation, and system tray. Lightweight Svelte frontend for the dashboard and tray popup. Checks run on dedicated tokio tasks.

**Tech stack:**
- Frontend: Svelte 5, Chart.js (lightweight charts), TailwindCSS
- Backend: reqwest (HTTP checks), trust-dns-resolver (DNS), surge-ping (ICMP), tokio (async scheduler), rusqlite
- Tray: tauri-plugin-positioner for tray popup alignment
- SSL: rustls for certificate inspection and expiry checking

**Plugin needs:** tauri-plugin-notification, tauri-plugin-autostart, tauri-plugin-positioner, tauri-plugin-shell (for traceroute)

**Bundle size:** ~5-10 MB

**Memory:** ~20-45 MB (ideal for always-running background app)

**Pros:**
- **Lightest footprint** — critical for an app that runs 24/7 during on-call shifts alongside other tools
- Rust async runtime (tokio) is ideal for concurrent HTTP/TCP/DNS/ICMP checks with thousands of tasks
- surge-ping provides raw ICMP without elevated permissions on most systems
- Background monitoring uses <10 MB additional memory beyond base footprint
- Battery-friendly: no rendering engine running when dashboard is closed — only Rust threads
- reqwest with connection pooling enables efficient high-frequency checks

**Cons:**
- Chart.js is adequate but less feature-rich than D3/Recharts for real-time streaming charts with annotations
- Tray popup styling limited by system webview capabilities on older OS versions
- Building traceroute in pure Rust is complex — may need to shell out to system traceroute binary
- Less community tooling for network diagnostic UIs (no pre-built traceroute visualization components)

### Flutter Desktop

**Architecture:** Main window for full dashboard. System tray via tray_manager. Riverpod for reactive state management. Rust FFI via flutter_rust_bridge for network checks. Custom chart widgets for response time visualization.

**Tech stack:**
- UI: Flutter 3.x, fl_chart (charts), Riverpod, tray_manager
- Database: drift (SQLite)
- Network: flutter_rust_bridge -> reqwest/surge-ping for monitoring checks
- Notifications: local_notifier with scheduled triggers

**Plugin needs:** tray_manager, local_notifier, window_manager, launch_at_startup

**Bundle size:** ~25-35 MB

**Memory:** ~90-160 MB

**Pros:**
- fl_chart provides smooth, animated response time graphs with gesture interaction (pinch zoom, pan)
- Custom widgets can create unique status visualizations (heatmaps, sparklines, uptime calendars)
- Could extend to mobile monitoring app for on-call engineers checking status from their phone
- Hot reload for rapid dashboard layout iteration during development
- Riverpod streams map well to real-time check result data flow

**Cons:**
- System tray support via tray_manager is less mature than Electron or Tauri — limited icon options
- Network diagnostics (ICMP, traceroute) require Rust FFI bridge adding build complexity
- Always-running background monitoring adds overhead from Flutter engine even when no UI is visible
- No built-in way to show stats in macOS menu bar text (requires platform channel to NSStatusItem)
- tray_manager does not support tray popup windows on all platforms

### Swift/SwiftUI (macOS)

**Architecture:** Pure SwiftUI with NSStatusItem for menu bar app. Network framework for monitoring checks. UserNotifications for alerts. Swift Charts for visualization. Designed as a menu bar utility with optional detachable dashboard window.

**Tech stack:**
- UI: SwiftUI, Charts framework (native), NSStatusItem + NSPopover for menu bar
- Data: SwiftData with lightweight migration support
- Network: NWConnection (TCP), URLSession (HTTP/HTTPS), dnssd (DNS), SimplePing (ICMP)
- Notifications: UNUserNotificationCenter with actionable notifications and categories
- Background: BackgroundTasks framework + DispatchSourceTimer for check scheduling
- SSL: Security.framework for certificate chain validation

**Bundle size:** ~6-10 MB

**Memory:** ~18-35 MB

**Pros:**
- **Best menu bar integration** — display live response time or status emoji directly in menu bar text (NSStatusItem.button.title)
- NWConnection and URLSession provide the most reliable macOS network stack with built-in retry and caching
- Actionable notifications with custom actions (Acknowledge, Snooze) are native and appear in Notification Center
- Apple Charts renders beautiful, performant response time graphs with accessibility built in
- Lowest memory and battery usage — essential for 24/7 on-call monitoring without impacting other work
- Widgets: macOS desktop widget showing current status of critical monitors on the desktop
- Siri Shortcuts: "Hey Siri, what's my server status?" via App Intents

**Cons:**
- macOS only — excludes Windows/Linux users (DevOps teams often use Linux and many SREs are on Ubuntu)
- SimplePing for ICMP is limited compared to surge-ping — no TTL control or raw packet access
- No cross-platform code reuse for monitoring logic
- Smaller community for network monitoring tools in Swift compared to Go/Rust ecosystems
- dnssd API is more complex than simple DNS resolution libraries

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform for dashboard. AWT SystemTray for tray icon. Ktor for HTTP checks, dnsjava for DNS lookups. ViewModel + StateFlow for reactive updates. Check scheduler via Kotlin coroutines.

**Tech stack:**
- UI: Compose Desktop, Material 3, custom Compose Canvas charting
- Tray: java.awt.SystemTray with TrayIcon
- Database: SQLDelight with typed queries
- Network: Ktor client (HTTP/HTTPS), dnsjava (DNS), Java InetAddress (ICMP)
- Charts: Compose Canvas-based custom charting or JFreeChart via interop
- HTTP: Ktor client with connection pooling

**Bundle size:** ~50-70 MB (+JRE)

**Memory:** ~170-280 MB

**Pros:**
- Ktor client supports HTTP/2 and WebSocket for efficient monitoring and real-time cloud sync
- dnsjava is the most comprehensive Java DNS library — supports all record types and DNSSEC
- Could share monitoring logic with Android companion app for mobile on-call status checking
- Kotlin coroutines with Flow map well to streaming check results and reactive UI updates
- Strong ecosystem for scheduling (Quartz, kotlinx-coroutines-scheduled)

**Cons:**
- JVM memory overhead makes it inappropriate for a 24/7 background monitoring app (170 MB idle is excessive)
- AWT SystemTray provides minimal tray functionality (basic icon, no rich popup, no color-coded status)
- Java InetAddress.isReachable for ICMP is unreliable and often falls back to TCP on non-root systems
- Slow startup time (~2s) for a tool that needs to be available immediately during incidents
- No way to display stats in macOS menu bar text via AWT

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 6 | 7 | 8 | 6 | 8 |
| Bundle size | 200 MB | 8 MB | 30 MB | 8 MB | 60 MB |
| Memory usage | 300 MB | 30 MB | 120 MB | 25 MB | 220 MB |
| Startup time | 2.5s | 0.5s | 1.1s | 0.3s | 1.9s |
| Native feel | 4/10 | 7/10 | 5/10 | 10/10 | 3/10 |
| Offline capability | 7/10 | 9/10 | 7/10 | 8/10 | 7/10 |
| Network diagnostics | 7/10 | 9/10 | 6/10 | 8/10 | 6/10 |
| Cross-platform | 9/10 | 9/10 | 9/10 | 2/10 | 8/10 |
| Best fit score | 5/10 | 9/10 | 6/10 | 8/10 | 4/10 |

---

## Recommended Framework

**Tauri** is the best fit for StatusPing Desktop.

**Rationale:** StatusPing is fundamentally a background tray application that must run 24/7 during on-call shifts with minimal resource consumption. Tauri's 30 MB memory footprint vs. Electron's 300 MB is not just a nice-to-have — it directly impacts battery life and system responsiveness for engineers who have many monitoring tools running simultaneously. The Rust backend with tokio provides the ideal async runtime for concurrent HTTP/TCP/DNS/ICMP checks across potentially hundreds of monitors, and the lightweight Svelte frontend renders an adequate dashboard when needed without consuming resources when hidden.

**Runner-up:** Swift/SwiftUI for a Mac-only version. The native menu bar integration (showing live stats in menu bar text), macOS Widgets for desktop status monitoring, and Siri Shortcuts integration would be the premium experience for macOS-based DevOps teams. Consider building this as a companion app if Mac users exceed 50% of the user base.

---

## Monetization (Desktop)

- **Free tier:** Up to 5 monitors, 1-minute check interval, local notifications, 24-hour check result history
- **Pro ($19/mo or $149/yr):** Unlimited monitors, 10-second intervals, SMS/Slack/PagerDuty alerts, 90-day history, status pages, SSL monitoring
- **Team ($12/user/mo):** Multi-user dashboard, on-call scheduling with rotation, incident management, runbook integration, post-mortem templates
- **Desktop agent add-on (free with Pro):** Local monitoring probe supplements cloud checks — detects network-specific issues and provides multi-region perspective
- **Distribution:** Direct download, Homebrew, Chocolatey. Skip app stores — DevOps engineers prefer direct installation and CLI tooling. Provide `brew install statusping` and `choco install statusping` packages.

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | HTTP/HTTPS monitor engine (Rust core), check scheduling with configurable intervals and timeouts, SQLite storage for check results with timestamp indexing |
| 3-4 | System tray with color-coded status icon, tray popup dashboard showing all monitors with status and response time, native OS notifications on status change |
| 5-6 | Response time charts (sparklines and full history), uptime percentage calculation, incident detection and state machine (triggered -> acknowledged -> resolved) |
| 7 | Local network diagnostics (ping, DNS lookup, traceroute), auto-start at login, notification action buttons (Acknowledge, Snooze), sound alerts |
| 8 | Auto-update, packaging (DMG/MSI/AppImage), cloud sync for monitor configuration import, beta testing with on-call teams |

**Post-MVP:** TCP/DNS/ICMP monitors, SSL certificate expiry monitoring, status page generator, on-call scheduling with rotation, Slack/PagerDuty integration, multi-account support, macOS Widgets, incident post-mortem templates, SLA reporting
