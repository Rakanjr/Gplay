# Gplay — Tech Stack & Rationale

One choice per layer. This document explains what each component is and why Gplay specifically needs it.

---

## Core

| Layer | Choice |
|---|---|
| Framework | Flutter |
| Language | Dart |
| Backend | Firebase |

### Flutter (framework)

**What it is:** A Google toolkit for building phone apps. You write the app's screens and logic once, and Flutter compiles it into both an iOS app and an Android app.

**Why Gplay needs it:** Sports groups always mix iPhone and Android users. The alternative — Swift for iOS + Kotlin for Android — means two languages, two codebases, every feature and bug handled twice. One codebase is the difference between shipping and never finishing.

### Dart (language)

**What it is:** The language Flutter apps are written in. Not a separate choice — Flutter = Dart.

**Why it's fine:** Readable, forgiving, hard to write catastrophic bugs in. You learn exactly one language for the entire app.

### Firebase (backend)

**What it is:** A set of ready-made server services run by Google (database, auth, server logic, push notifications).

**Why Gplay needs it:** P1 demands "exactly one list, stored server-side." If the list lives in a WhatsApp message it's a copy; if it lives on a server everyone connects to, it's *the* list. Firebase provides that server without building, renting, or maintaining one.

---

## Firebase Services

### Cloud Firestore (database)

**What it is:** Where all data lives — sessions, player names, payment statuses.

**Why Gplay needs it:** Its live listeners push changes to every open app the instant they happen — ~3 lines of code and every screen updates within a second. That is F2.5 / FR-8 (the P1 fix) nearly for free; with most other databases it's a significant project. Built-in offline cache covers F6.1 / NFR-4 at zero extra work.

### Firebase Authentication (login)

**What it is:** Ready-made secure login. Passwords are stored and hashed by Google; the app never touches them.

**How Gplay uses it:**
- **Hosts (F1.1):** email + password accounts.
- **Guests (F2.2):** anonymous sign-in — on first app open, Firebase silently creates an invisible account tied to the device. No form, no password, no email. This is the "lightweight device token" from FR-15: it lets a guest leave a session they joined and keeps their name remembered. Same auth system as hosts, so a guest who later becomes a host can upgrade/merge accounts.

### Cloud Functions (server logic)

**What it is:** Small pieces of app code that run on Google's servers, triggered by events (e.g., someone taps Join).

**Why Gplay needs it — the most important decision in the stack:**

FR-5's guarantee (two simultaneous joins can never collide or lose a name) is impossible to enforce on phones alone: two devices can both read "one spot left" in the same millisecond and both write "I'm in." The fix must live in one place everyone goes through — the server.

Joins, leaves, auto-promotions, and payment confirms run as **transactions** inside Cloud Functions: a locked, all-or-nothing operation (check capacity → assign spot or bench → write → done). Simultaneous requests queue; the second sees the true state.

Firestore technically allows transactions from the device, but a phone app can be tampered with to bypass capacity rules. Logic inside a Cloud Function cannot be bypassed, because that code never lives on anyone's phone.

### Cloud Messaging / FCM (push notifications)

**What it is:** Google's system for sending notifications to phones even when the app is closed.

**Why:** Post-MVP "You've been promoted from the bench" alerts (F3.3). For MVP the live list is the notification. Free and already part of Firebase, so adding later is easy.

### Firebase Hosting (one verification file)

**What it is:** Static file hosting at your own domain (e.g., `gplay.app`).

**Why — this is NOT a web version of the app:** Apple and Google require one small verification file hosted at your domain to prove ownership before they enable deep links. That file is Hosting's entire job. (Plus a minimal landing page for the not-installed case, built with F1.4.)

---

## Flutter Libraries

| Package | Purpose |
|---|---|
| `firebase_core`, `cloud_firestore`, `firebase_auth`, `cloud_functions` | Official bridges from Dart to each Firebase service — one-line calls instead of raw network code |
| `flutter_riverpod` | State management — see below |
| `go_router` | Navigation between screens; URL-aware, handles half of deep linking |
| `app_links` | Catches incoming deep links and hands the URL to the app |

### Riverpod (state management)

**What it is:** "State" = the data currently shown on screen. State management is the plumbing that answers: when data changes, how do the right screens redraw?

**Why Gplay needs it:** The whole app is "data changes → screens update." A join on phone A must redraw the list on phone B; a host's must flip a badge everywhere. Riverpod pairs directly with Firestore's live updates: data flows in, screens rebuild automatically. Flutter's built-in `setState` falls apart across a multi-screen app; Riverpod is the most popular serious choice, well documented, and well known to every tutorial and AI tool — which matters when learning.

---

## Deep Linking (F1.4): Apple Universal Links + Android App Links

**Problem:** Someone taps `gplay.app/s/x7k2p` in WhatsApp. App installed → open straight to that session. Not installed → go to the store page.

**How:** The native systems Apple and Google built for exactly this. Host a verification file at the domain (Firebase Hosting), register the domain in the app, and the phone's OS routes the link before any browser opens. `go_router` + `app_links` handle the Flutter side.

**Why not Firebase Dynamic Links:** the old standard answer — Google shut it down in August 2025. The native replacement has no third-party middleman, no deprecation risk, no cost.

---

## Tooling

| Tool | Role |
|---|---|
| VS Code + Flutter extension | The editor. Adds run button, device picker, error highlighting, hot reload (code change visible on device in under a second) |
| Xcode | Installed, never opened — provides Apple's compilers Flutter calls behind the scenes |
| Android Studio | Installed, never opened — provides the Android SDK and emulators |
| `flutter doctor` | Built-in health check: prints exactly what's installed, what's missing, and the fix for each. Run until everything is good |
| git + GitHub | Version control (already set up) |
| Paper sketch | Design — 3 screens don't justify Figma; hot reload makes the app itself the design tool |

---

## Data Lifecycle Rules (privacy)

Two distinct pieces of guest data, treated differently:

1. **Anonymous auth account** — deleted when: the session ends or is cancelled; the guest leaves and is in zero other active sessions; or a scheduled cleanup wipes anonymous accounts older than X days (covers app uninstalls).
2. **List entry (name in the session)** — stays until the session ends. Required by FR-9 (removal is visible, never silent — a vanished name is the P1 bug) and FR-13 (disputes happen outside — the host needs the payment record to point to). When a guest leaves, their entry shows "Name — left" until session end, then all session data is deleted.

Privacy exposure is minimal by design: the only personal data a guest ever provides is a first name.

---

## Explicitly NOT in the Stack

- **No local database (SQLite etc.)** — Firestore's offline cache covers offline reads. Two databases = sync bugs, the exact disease this app cures.
- **No custom server / VPS** — Cloud Functions cover all server logic.
- **No payment SDK (Stripe etc.)** — FR-13: money never touches the app. Also avoids app-store payment rules and financial liability.
- **No web version of the session page** — everyone joins through the app.
- **No Figma for MVP** — paper sketch, then build directly.

## Cost Note

Development and MVP scale (hundreds of users, a few weekly games) run at $0 on Firebase's free allowances. Cloud Functions require the pay-as-you-go Blaze plan, but its free tier covers this scale — a budget alert will be configured on day one as a safety net.
