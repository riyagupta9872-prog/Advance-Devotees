# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

**Sakhi Sang** — a Sunday-class attendance and devotee management PWA for a Krishna devotee community. Stack: **vanilla JS + Firebase (Firestore) + PWA**. No build tools, no npm, no framework.

To run: open `index.html` in a browser or use any static file server (`python -m http.server`, VS Code Live Server). Firebase is configured inline.

## Architecture

### Module Map

| File | Responsibility |
|------|---------------|
| `config.js` | Firebase init, `AppState` singleton, `DevoteeCache` (90s TTL), date/format helpers, toast/modal utilities |
| `db.js` | All Firestore CRUD — devotees, sessions, attendance, calling, events, books, donations; batch writes capped at 400 |
| `excel.js` | Import/export via `xlsx-js-style`; duplicate detection (name+mobile fuzzy match), batched writes |
| `js/ui-core.js` | Auth, role-based UI (`applyRoleUI()`), admin modals, tab switching |
| `js/ui-home.js` | Dashboard: attendance report, books, donations |
| `js/ui-devotees.js` | Devotee directory and profiles |
| `js/ui-calling.js` | Weekly calling coordination and submission tracking |
| `js/ui-attendance.js` | Session attendance marking |
| `js/ui-analytics.js` | KPI dashboard, care tabs, team reports, trends |
| `js/ui-activities.js` | Events and registrations |
| `sw.js` | Service worker — version string **must be bumped** on each deploy |

### State & Reactivity

`AppState` in `config.js` is the single source of truth (auth user, role, session filter, team filter, callingBy filter).

**Only mutate filters via `dispatchFilters()`** — it fires the `filtersChanged` custom event that all `load*()` functions subscribe to. Never mutate `AppState.filters` directly.

Each tab's `load*()` function must be async and idempotent — called on every `filtersChanged` event.

### Auth & Roles

| Role | Access |
|------|--------|
| `superAdmin` | Full app, all teams, user management |
| `teamAdmin` | Own team data only; team name locked in filters |
| `serviceDevotee` | Calling tab + Attendance tab only |

Bootstrap: first user to sign up auto-becomes `superAdmin`. Subsequent users create a signup request; superAdmin approves/rejects from "Sign-up Requests" modal.

Special case: "Login as Attendance Service Devotee" checkbox at login overrides role to `serviceDevotee` for that browser session only (stored in `sessionStorage`).

### Firestore Conventions

- Doc IDs for sessions/weekly data: `YYYY-MM-DD` string (always IST — use `toLocalDateStr()`, never raw `new Date().toISOString()`)
- Soft-delete: `isActive: false`; never hard-delete
- Dual timestamps on writes: `updatedAt` (server `FieldValue.serverTimestamp()`) + `updatedAtClient` (ISO string)
- Audit trails: `profileChanges/`, `callingStatusChanges/`, `callingSubmissions/`
- `camelCase` in JS; Firestore stores `snake_case` — convert via `toCamel()` / `toSnake()` in `db.js`

### Key Collections

`devotees`, `sessions`, `attendanceRecords`, `callingStatus`, `callingSubmissions`, `callingStatusChanges`, `users`, `signupRequests`, `personalMeetings`, `bookDistributions`, `donations`, `registrations`, `services`, `events`, `eventDevotees`, `settings`, `profileChanges`, `callingWeekHistory`

### Business Logic Highlights

**Calling status values**: `Yes`, `No`, `Shift`, `Not Interested` (plus festival variants `online_class`, `festival_calling`, `not_interested_now`)

**Calling submission**: A facilitator submitting their week locks edits and counts in reports. Only submitted callers appear in calling counts; unsubmitted show as "Not Submitted".

**Care logic**:
- Absent this week = missed latest session but attended 1+ of prior 4
- Absent past 2 weeks = missed last 2 but attended 1+ of prior 3
- Inactive auto-flag = missed last 3 sessions; auto-clears on attendance

**Attendance time-coding** (background color on attendance cards):
- Before 12:30 → no highlight
- 12:30–12:45 → light pink (`#ffcdd2`)
- 12:45–1:00 → darker pink (`#ef9a9a`)
- After 1:00 → dark red (`#c62828`) with white text

**Teams** (hardcoded): Champaklata, Chitralekha, Indulekha, Lalita, Nilachal, Other, Rangadevi, Sudevi, Tungavidya, Vishakha

**Devotee status hierarchy**: `Most Serious` > `Serious` > `Expected to be Serious` (default, stored as `"ETS"`) > `New Devotee` > `Inactive`

### PWA / Service Worker

Cache strategies in `sw.js`:
- Firebase/Firestore URLs → network-only (never cached)
- App JS/HTML → network-first (fresh on deploy, offline fallback)
- Fonts, CSS, CDN libs → cache-first

**Always bump the version string** (currently `v233`) in `sw.js` when changing any cached asset. Users won't pick up JS/CSS changes otherwise.

### UI Conventions

- Navigation = show/hide via `.hidden` class; no routing
- Modals: IDs end with `-modal`; close via `closeModal(id)`
- Toasts: `showToast(message, type)` — type is `success | error | warning | info`
- CSS tokens: primary green `#1a5c3a`, saffron accent; semantic vars `--color-success`, `--color-danger`, `--color-warning`; fonts Cinzel (headings) + Nunito/DM Sans (body)
- Picker controls: `.picker-wrap > .picker-input + .picker-menu`

## When Adding Features

1. New tab needs: a `load*()` async function, a `filtersChanged` listener, and registration in `ui-core.js`'s tab switcher.
2. Functions called from HTML `onclick=` must be on `window` (e.g., `window.myFunction = async function() {}`).
3. No npm, no build step — add libraries via CDN `<script>` tags in `index.html`.
4. Bump `sw.js` version after any HTML/JS/CSS change users need to receive immediately.
5. Batch Firestore writes in 400-doc chunks; use `FieldValue.serverTimestamp()` + `updatedAtClient` ISO string on every write.
