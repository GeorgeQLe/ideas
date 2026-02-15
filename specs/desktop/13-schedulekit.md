# ScheduleKit Desktop — 5-Framework Comparison

## Desktop Rationale

ScheduleKit (embeddable scheduling widget and calendar management tool) gains powerful advantages as a desktop application:

- **Native calendar integration (CalDAV):** Desktop apps can directly read and write to system calendars (macOS Calendar, Outlook, Thunderbird) via CalDAV and native APIs, enabling real-time availability checks without cloud API round-trips or OAuth token juggling.
- **System tray for upcoming meetings:** A persistent tray icon showing the next meeting countdown with one-click join links. Far faster than opening a browser to check your schedule.
- **Offline schedule management:** View, create, and modify appointments offline with full calendar access. Changes sync when connectivity returns — critical for professionals who schedule between meetings in areas with poor reception.
- **Notification center integration:** Native OS notifications for upcoming meetings, scheduling conflicts, and new booking requests that integrate with Do Not Disturb and Focus modes rather than fighting against them.
- **Drag-and-drop scheduling:** Native drag-and-drop for moving appointments between time slots with haptic-like precision and 60fps rendering — the calendar interaction model that web apps struggle to match.

---

## Desktop-Specific Features

- **System tray meeting countdown:** Next meeting name, time remaining, and one-click join link (Zoom/Meet/Teams) in the tray tooltip
- **Native calendar read/write:** Direct CalDAV and EventKit/Outlook COM access for real-time availability without cloud API
- **Global hotkey (Cmd+Shift+K):** Instant scheduling popup — select a time slot and create a meeting from any application
- **Meeting join buttons:** Click tray notification to auto-open Zoom/Meet/Teams meeting link at the scheduled time
- **Day planner widget:** Desktop widget showing today's schedule with color-coded calendar sources
- **Drag-and-drop rescheduling:** Native-feel drag to move appointments between time slots with smooth animation and conflict detection
- **Notification center actions:** Reply to scheduling requests directly from OS notifications (accept, decline, propose new time)
- **Auto-scheduling assistant:** Analyze open time slots across linked calendars and suggest optimal meeting times
- **Focus time protection:** Automatically block calendar time for focus work based on configurable rules (no meetings before 10am, 2-hour focus blocks)
- **Time zone overlay:** Display multiple time zones on the calendar view for scheduling across distributed teams

---

## Shared Packages

### core (Rust)

- **Availability engine:** Compute available time slots from multiple calendar sources with configurable buffer time, working hours, and timezone awareness
- **Conflict detector:** Real-time detection of scheduling conflicts, double-bookings, and travel time violations across all linked calendars
- **Scheduling optimizer:** Algorithm to find optimal meeting times for multiple participants considering preferences, time zones, and existing commitments
- **Recurrence engine:** RFC 5545 (iCalendar) compliant recurrence rule parsing and expansion for recurring meetings
- **Template engine:** Meeting type templates (30min/60min, required fields, default conferencing links, agenda templates)
- **Time zone handler:** IANA timezone database with DST-aware calculations for accurate cross-timezone scheduling

### api-client (Rust)

- **CalDAV client:** Full CalDAV protocol implementation (RFC 4791) for reading/writing to any CalDAV server (Google Calendar, iCloud, Fastmail, Nextcloud)
- **Google Calendar API:** REST client for Google Calendar with efficient sync (incremental sync tokens, push notifications)
- **Microsoft Graph API:** REST client for Outlook/Exchange calendars via Microsoft Graph
- **Booking page sync:** Push availability to ScheduleKit cloud for the embeddable booking widget; receive new bookings via webhook
- **Auth:** OAuth2 for Google/Microsoft, CalDAV basic/digest auth, JWT for ScheduleKit cloud

### data (SQLite)

- **Schema:** calendars (id, name, source, caldav_url, color, sync_token), events (id, calendar_id, title, start, end, rrule, location, conference_url, attendees_json, status), booking_types, availability_rules, settings
- **Offline events:** Full local copy of all linked calendar events with periodic sync; immediate local writes with background cloud push
- **Sync strategy:** ETag-based CalDAV sync for external calendars; CRDT merge for ScheduleKit bookings (concurrent bookings resolved by timestamp priority)

---

## Framework Implementations

### Electron

**Architecture:** Main process handles CalDAV sync, calendar polling, system tray with meeting countdown, and notification dispatch. Renderer process (React) provides the calendar UI with drag-and-drop scheduling, settings, and booking type configuration.

**Tech stack:**
- Renderer: React 18, FullCalendar (calendar component), TailwindCSS, react-beautiful-dnd (drag-and-drop scheduling), Zustand
- Main: tsdav (CalDAV client), node-cron (sync scheduling), Notification API
- Database: better-sqlite3 with Drizzle ORM
- Calendar parsing: ical.js (iCalendar format parsing)

**Native module needs:** better-sqlite3, electron-positioner (tray popup), keytar (credential storage)

**Bundle size:** ~175-230 MB

**Memory:** ~230-350 MB

**Pros:**
- FullCalendar is the most feature-rich calendar component available — week/day/month views, drag-and-drop, resource scheduling
- tsdav provides a well-tested CalDAV client for JavaScript
- Extensive time zone handling libraries (Luxon, date-fns-tz)
- Web-based calendar UI allows maximum customization of the scheduling experience

**Cons:**
- Heavy bundle and memory for an app that primarily shows a tray countdown and handles notifications
- FullCalendar's drag-and-drop feels web-like, not native (no inertia, no snap-to-grid smoothness)
- No native calendar access — must use CalDAV even for local calendar accounts
- Background Chromium process for tray meeting countdown wastes resources

### Tauri

**Architecture:** Rust backend handles CalDAV sync (reqwest + quick-xml for WebDAV), system tray meeting countdown, notification scheduling, and availability computation. Svelte frontend for calendar view and scheduling interface.

**Tech stack:**
- Frontend: Svelte 5, @event-calendar/core (Svelte calendar component), TailwindCSS, date-fns
- Backend: reqwest (CalDAV HTTP), quick-xml (WebDAV/iCalendar XML parsing), rusqlite, tokio, icalendar-rs (iCal parsing)
- Notifications: tauri-plugin-notification

**Plugin needs:** tauri-plugin-notification, tauri-plugin-global-shortcut, tauri-plugin-autostart, tauri-plugin-positioner, tauri-plugin-store

**Bundle size:** ~6-12 MB

**Memory:** ~30-55 MB (idle tray); ~70-110 MB (calendar view open)

**Pros:**
- Lightweight tray app perfect for always-running meeting countdown
- Rust CalDAV implementation with reqwest + quick-xml is fast and memory-efficient
- icalendar-rs handles RFC 5545 recurrence rules natively in Rust
- Fast startup ensures quick access to scheduling via global hotkey
- Minimal battery impact for background calendar sync and notifications

**Cons:**
- @event-calendar/core (Svelte) is less feature-rich than FullCalendar
- No native calendar access — must implement CalDAV from scratch or use system commands
- Drag-and-drop rescheduling in system webview may feel less smooth than native
- CalDAV implementation in Rust requires significant protocol knowledge

### Flutter Desktop

**Architecture:** Flutter UI with custom calendar widget. Platform channels for native calendar access (EventKit on macOS, COM on Windows). Riverpod for state management. Background isolate for CalDAV sync.

**Tech stack:**
- UI: Flutter 3.x, table_calendar or syncfusion_flutter_calendar, Riverpod
- Database: drift (SQLite)
- Calendar: platform channel to EventKit (macOS) / Windows Calendar COM API
- CalDAV: shelf_caldav or custom implementation via http + xml packages
- Notifications: local_notifier with scheduled notifications

**Plugin needs:** tray_manager, local_notifier, window_manager, launch_at_startup, hotkey_manager

**Bundle size:** ~24-33 MB

**Memory:** ~90-155 MB

**Pros:**
- syncfusion_flutter_calendar provides a polished calendar with day/week/month/agenda views and drag-and-drop
- Platform channels can access native calendar stores directly on macOS (EventKit)
- Custom gesture handling enables smooth drag-and-drop rescheduling with animation
- Could extend to mobile app for on-the-go scheduling

**Cons:**
- Platform channel calendar access requires separate implementations for macOS and Windows
- Syncfusion components have separate licensing costs
- CalDAV implementation in Dart is immature (shelf_caldav is limited)
- tray_manager meeting countdown is less polished than native tray implementations

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI views with @Observable view models. EventKit for direct calendar read/write. NSStatusItem for menu bar meeting countdown. UNUserNotificationCenter for meeting alerts. WidgetKit for desktop day planner widget.

**Tech stack:**
- UI: SwiftUI, native calendar views (FSCalendar via UIKit bridge or custom SwiftUI calendar)
- Data: SwiftData
- Calendar: EventKit (full read/write access to all macOS calendars — iCloud, Google, Exchange, CalDAV)
- Notifications: UNUserNotificationCenter with UNNotificationAction (accept/decline inline)
- Widget: WidgetKit for macOS desktop widget
- Conference: URLSession + universal links for meeting join

**Bundle size:** ~6-10 MB

**Memory:** ~20-40 MB (idle); ~45-75 MB (calendar view active)

**Pros:**
- **Best calendar integration by far:** EventKit provides direct, permission-based access to ALL system calendars (iCloud, Google, Exchange, CalDAV) without implementing each protocol separately
- Actionable notifications with accept/decline/propose buttons directly in Notification Center
- WidgetKit enables a native macOS desktop widget showing today's schedule
- NSStatusItem tray icon with meeting countdown is the most polished implementation on macOS
- Focus mode integration: suppress notifications during scheduled focus time
- Siri Shortcuts: "Schedule a meeting with [person]" voice scheduling
- Universal Links: auto-detect and launch Zoom/Meet/Teams from meeting URLs
- Lowest resource usage for always-running tray app

**Cons:**
- macOS only — excludes Windows and Linux users entirely
- EventKit is macOS-specific; no equivalent cross-platform API
- SwiftUI calendar views are less mature than web alternatives (FullCalendar)
- Custom SwiftUI drag-and-drop for time slot selection requires significant implementation effort

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform for calendar UI. AWT SystemTray for tray icon. ical4j for iCalendar parsing. Ktor for CalDAV HTTP operations. SQLDelight for persistence.

**Tech stack:**
- UI: Compose Desktop, Material 3, custom Calendar composable
- Tray: java.awt.SystemTray with TrayIcon
- CalDAV: ical4j (iCalendar parsing) + Ktor (WebDAV HTTP)
- Database: SQLDelight
- HTTP: Ktor client
- Notifications: java.awt.TrayIcon.displayMessage (basic) or JNA to native APIs

**Bundle size:** ~50-70 MB (+JRE)

**Memory:** ~160-260 MB

**Pros:**
- ical4j is the most comprehensive iCalendar library available — full RFC 5545 compliance including complex recurrence, VTIMEZONE, and VFREEBUSY
- Ktor provides elegant coroutine-based CalDAV sync with proper WebDAV support
- Could share scheduling logic with Android app for mobile booking management
- Java time API (java.time) provides excellent timezone handling out of the box

**Cons:**
- AWT tray icon provides a poor meeting countdown experience (no rich tooltip, ugly notifications)
- JVM startup delay means the meeting countdown is unavailable for 2-3s after login
- Building a full calendar view in Compose Canvas is significant engineering effort
- JVM memory overhead is excessive for a scheduling utility that should feel lightweight

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 7 | 9 | 9 | 6 | 10 |
| Bundle size | 200 MB | 9 MB | 28 MB | 8 MB | 60 MB |
| Memory usage | 290 MB | 45 MB | 120 MB | 30 MB | 210 MB |
| Startup time | 2.6s | 0.7s | 1.2s | 0.3s | 2.2s |
| Native feel | 5/10 | 6/10 | 6/10 | 10/10 | 4/10 |
| Offline capability | 7/10 | 8/10 | 7/10 | 9/10 | 7/10 |
| Calendar integration | 5/10 | 6/10 | 7/10 | 10/10 | 7/10 |
| Notification quality | 6/10 | 7/10 | 5/10 | 10/10 | 3/10 |
| Drag-and-drop UX | 7/10 | 6/10 | 7/10 | 8/10 | 5/10 |
| Cross-platform | 9/10 | 9/10 | 9/10 | 2/10 | 8/10 |
| Best fit score | 6/10 | 7/10 | 6/10 | 9/10 | 4/10 |

---

## Recommended Framework

**Swift/SwiftUI** is the best fit for ScheduleKit Desktop on macOS. **Tauri** is recommended for cross-platform distribution.

**Rationale:** Calendar integration is ScheduleKit's core differentiator, and EventKit on macOS provides unmatched access to all system calendars without implementing CalDAV, Google Calendar API, or Microsoft Graph separately. The native notification actions (accept/decline from Notification Center), WidgetKit desktop widget, and Focus mode awareness create a scheduling experience that no cross-platform framework can replicate. For organizations that need Windows/Linux support, Tauri provides the best alternative with its lightweight footprint and efficient Rust-based CalDAV implementation.

**Strategy:** Build Swift/SwiftUI for macOS first (fastest path to the best calendar experience, largest professional desktop market). Follow with a Tauri cross-platform version for Windows/Linux, sharing the Rust core (availability engine, CalDAV client, recurrence parser) between both apps.

**Runner-up:** Tauri as standalone cross-platform choice if building only one version. The trade-off is implementing CalDAV protocol in Rust rather than leveraging EventKit, but the lightweight footprint and cross-platform coverage make it viable.

---

## Monetization (Desktop)

- **Free tier:** 1 calendar source, basic availability view, manual scheduling, tray meeting countdown
- **Pro ($12/mo or $99/yr):** Unlimited calendar sources, booking page, AI scheduling assistant, focus time protection, offline mode
- **Team ($8/user/mo):** Team availability overlay, round-robin scheduling, shared booking pages, meeting analytics
- **Enterprise ($15/user/mo):** SSO, Exchange/Office 365 admin integration, room booking, custom branding, API access
- **Distribution:** Mac App Store (EventKit requires App Store or notarization; App Store simplifies enterprise MDM deployment) + direct download for Tauri cross-platform version

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Calendar view (day/week/month), EventKit integration for reading system calendars, event display with color coding |
| 3-4 | System tray meeting countdown, native notifications for upcoming meetings with join-link buttons, global hotkey quick scheduling |
| 5-6 | Availability computation from linked calendars, booking page sync with ScheduleKit cloud, new event creation with drag-and-drop |
| 7 | Offline mode with local event cache, CalDAV sync for non-Apple calendars, conflict detection and alerts |
| 8 | Auto-update, packaging (DMG for macOS, MSI/AppImage for Tauri), settings UI, timezone handling, beta launch |

**Post-MVP:** WidgetKit desktop widget, AI scheduling assistant, focus time protection, team availability overlay, Google Calendar and Outlook direct API sync, Siri Shortcuts integration, meeting analytics dashboard, room booking
