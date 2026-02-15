# PulseBoard Desktop — 5-Framework Comparison

## Desktop Rationale

PulseBoard (remote team energy tracker) gains compelling advantages as a desktop app:

- **Persistent system tray widget:** One-click daily check-in from the menubar — no browser tab needed. Reduces friction from "open browser → navigate → click" to "click tray icon → tap emoji."
- **Always-visible energy status:** Optional always-on-top mini-widget showing team energy at a glance.
- **Desktop notifications:** Native OS notifications for burnout alerts, team energy drops, and check-in reminders.
- **Calendar integration:** Read native calendar events to correlate meeting load with energy scores.
- **Focus mode integration:** Detect macOS Focus / Windows Focus Assist status for context-aware check-ins.

---

## Desktop-Specific Features

- **Menubar check-in widget:** 1-5 energy scale + emoji in a compact dropdown from system tray
- **Always-on-top mini dashboard:** Tiny floating window with team energy heatmap
- **Native calendar read:** Access macOS EventKit / Windows Calendar API to show meeting load alongside energy data
- **Do Not Disturb awareness:** Suppress check-in reminders when user is in Focus mode
- **Desktop idle detection:** Detect when user is away; adjust check-in timing accordingly
- **Screenshot-free mood capture:** Quick keyboard shortcut (Cmd+Shift+P) to log mood without context-switching
- **Native notifications:** Rich notifications with inline reply for quick mood note
- **Login item:** Auto-start at login for seamless daily check-ins
- **Offline check-ins:** Record check-ins locally when offline; sync when reconnected

---

## Shared Packages

### core (Rust)

- **Check-in engine:** Validate energy scores (1-5), process emoji reactions, enforce one-check-in-per-day
- **Analytics calculator:** Compute team averages, trend lines, burnout risk scores, rolling 7/14/30 day windows
- **Burnout detection:** Algorithm evaluating consecutive low scores, declining trends, and meeting load correlation
- **Notification scheduler:** Determine when to prompt check-ins based on user timezone and preferences

### api-client (Rust)

- **Team sync:** Real-time WebSocket for live team energy updates; REST fallback for initial load
- **Check-in submission:** POST check-ins with offline queue and deduplication (idempotency key per user+date)
- **Slack/Teams relay:** Forward check-in reminders through Slack bot if configured
- **Auth:** JWT-based team auth with refresh tokens; team invite codes for onboarding

### data (SQLite)

- **Schema:** users, teams, checkins (user_id, date, energy, emoji, note, synced), team_settings, reminders
- **Local-first:** All personal check-in history stored locally; team data synced from server
- **Sync:** Check-ins sync up on submit; team aggregates pulled periodically (every 5 min) or via WebSocket

---

## Framework Implementations

### Electron

**Architecture:** Main process handles system tray, check-in reminders (node-cron), calendar access, and idle detection. Renderer process (React) for dashboard, trends, and settings.

**Tech stack:**
- Renderer: React 18, Recharts (trend graphs), TailwindCSS, Zustand
- Main: electron-tray, node-cron, electron-store, powerMonitor (idle detection)
- Database: better-sqlite3
- Calendar: ical.js for parsing exported calendars (no native access)

**Native module needs:** better-sqlite3, electron-positioner (tray popup placement)

**Bundle size:** ~160-200 MB

**Memory:** ~200-350 MB (full Chromium for what is essentially a tray widget)

**Pros:**
- Rich dashboard with web charting libraries (Recharts, D3, Chart.js)
- Full SaaS frontend code reuse for the dashboard view
- Tray popup can be a full mini web app with rich styling

**Cons:**
- 200+ MB and 200+ MB RAM for a check-in widget is absurdly heavy
- No native calendar access — must use ical export/import workaround
- powerMonitor idle detection is basic compared to native APIs
- Battery drain from background Chromium process on laptops

### Tauri

**Architecture:** Rust backend handles scheduling, system tray management, idle detection. Lightweight Svelte frontend for tray popup and dashboard window.

**Tech stack:**
- Frontend: Svelte 5, Chart.js (lightweight), TailwindCSS
- Backend: rusqlite, tokio (async runtime), notify-rust (notifications)
- Tray: tauri-plugin-positioner for tray popup placement

**Plugin needs:** tauri-plugin-notification, tauri-plugin-autostart, tauri-plugin-positioner

**Bundle size:** ~4-8 MB

**Memory:** ~25-50 MB (ideal for always-running tray app)

**Pros:**
- Tiny footprint perfect for an always-running tray widget
- Rust scheduling is efficient for daily reminder timers
- Minimal battery impact for background operation
- Fast tray popup render (system webview is already loaded)

**Cons:**
- No native calendar access from Rust (would need to call system commands or use ical files)
- Tray popup styling depends on system webview capabilities
- Less charting library variety than Electron (but Chart.js is sufficient)

### Flutter Desktop

**Architecture:** System tray via tray_manager plugin. Main window for dashboard. Riverpod for state management. Platform channels for calendar access on macOS.

**Tech stack:**
- UI: Flutter, fl_chart (charts), Riverpod
- Database: drift
- Calendar: platform channel → EventKit (macOS) / manual integration (Windows)
- Notifications: local_notifier

**Plugin needs:** tray_manager, local_notifier, window_manager, launch_at_startup

**Bundle size:** ~22-30 MB

**Memory:** ~80-150 MB

**Pros:**
- fl_chart provides beautiful, animated trend visualizations
- Can extend to mobile app (team members check in from phone)
- Platform channels can access native calendar APIs on macOS
- Hot reload excellent for iterating on the tray popup UI

**Cons:**
- System tray support (tray_manager) is less mature than Electron/Tauri
- Always-on-top mini widget is tricky in Flutter desktop
- Platform channel boilerplate for calendar access on each OS
- Heavier than Tauri for a background app

### Swift/SwiftUI (macOS)

**Architecture:** Pure SwiftUI with @Observable models. NSStatusItem for menu bar widget. EventKit for native calendar access. UserNotifications for alerts.

**Tech stack:**
- UI: SwiftUI, Charts framework (native), NSStatusItem + NSPopover
- Data: SwiftData with CloudKit sync
- Calendar: EventKit (direct access to macOS Calendar)
- Notifications: UNUserNotificationCenter
- Idle: NSWorkspace.shared for idle/sleep detection

**Bundle size:** ~5-8 MB

**Memory:** ~20-40 MB

**Pros:**
- **Best menu bar experience** — NSStatusItem + NSPopover is the native macOS pattern
- Native EventKit gives direct, permission-based access to all calendars
- Apple Charts framework renders beautiful native trend charts
- Lowest memory and battery usage — ideal for always-running app
- macOS Focus mode detection via NSWorkspace
- SwiftData + CloudKit sync for cross-device (Mac + iPhone) check-ins
- Widgets for macOS desktop widget showing team energy

**Cons:**
- macOS only — excludes Windows/Linux team members
- iCloud sync instead of custom backend (works for Apple-only teams)
- Smaller market (team tools must be cross-platform)

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform for dashboard. AWT SystemTray for tray icon. ViewModel + StateFlow for reactive state. Ktor for API.

**Tech stack:**
- UI: Compose Desktop, Material 3
- Tray: java.awt.SystemTray
- Database: SQLDelight
- HTTP: Ktor client
- Notifications: java.awt.TrayIcon.displayMessage (basic)

**Bundle size:** ~45-65 MB (+JRE)

**Memory:** ~150-250 MB

**Pros:**
- Could share code with Android companion app for mobile check-ins
- Compose animations are smooth for trend chart transitions
- SQLDelight provides type-safe cross-platform data layer
- Kotlin coroutines map well to periodic check-in scheduling

**Cons:**
- AWT SystemTray is the worst tray implementation (no popup, basic icons, ugly notifications)
- JVM memory overhead is wasteful for a background widget
- No native calendar access without JNI to platform APIs
- Slow startup time for a tray utility

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 5 | 6 | 7 | 5 | 8 |
| Bundle size | 180 MB | 6 MB | 25 MB | 6 MB | 55 MB |
| Memory usage | 280 MB | 35 MB | 110 MB | 30 MB | 200 MB |
| Startup time | 2.5s | 0.6s | 1.0s | 0.3s | 1.8s |
| Native feel | 4/10 | 6/10 | 6/10 | 10/10 | 3/10 |
| Offline capability | 7/10 | 8/10 | 7/10 | 9/10 | 7/10 |
| Calendar integration | 3/10 | 4/10 | 7/10 | 10/10 | 3/10 |
| Cross-platform | 9/10 | 9/10 | 9/10 | 2/10 | 8/10 |
| Best fit score | 5/10 | 8/10 | 7/10 | 9/10 | 4/10 |

---

## Recommended Framework

**Tauri** for cross-platform, **Swift/SwiftUI** for Mac-only.

**Rationale:** PulseBoard desktop is fundamentally a system tray widget with an occasional dashboard. Resource efficiency matters enormously for always-running background apps. Tauri at 35 MB RAM is acceptable; Electron at 280 MB for a mood tracker is not. Swift wins on macOS (native tray, calendar, notifications) but the cross-platform requirement for team tools makes Tauri the practical choice.

**Strategy:** Build Tauri first for cross-platform coverage. If Mac users are >60% of base, add a native Swift menu bar companion app for premium Mac experience.

---

## Monetization (Desktop)

- **Free tier:** Personal check-ins, local history, no team sync
- **Team ($4/user/mo):** Team dashboard, sync, burnout alerts, calendar correlation
- **Enterprise ($8/user/mo):** SSO, advanced analytics, API access, custom branding
- **Desktop premium (+$2/user/mo):** Native calendar integration, offline mode, always-on-top widget
- **App Store:** Distribute via Mac App Store (good for enterprise IT approval) + direct download
- **No perpetual license** — team tools require ongoing server costs, subscription model fits

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | System tray check-in widget (1-5 scale + emoji), local SQLite storage |
| 3-4 | Team dashboard with trend charts, WebSocket sync for real-time team data |
| 5 | Native notifications for check-in reminders and burnout alerts |
| 6 | Idle detection, auto-start at login, offline check-in queue |
| 7 | Settings UI, team management, onboarding flow |
| 8 | Auto-update, packaging, beta testing with 2-3 teams |

**Post-MVP:** Calendar integration, always-on-top mini widget, Focus mode detection, macOS Widgets
