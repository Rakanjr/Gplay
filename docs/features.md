# Gplay — Feature Requirements & Feature Breakdown

## Table 1 — Feature Requirements

### Functional Requirements

| Req ID | Requirement | Problem |
|--------|-------------|---------|
| FR-1 | Host creates a session: date/time, venue, total cost, capacity (spots) | — |
| FR-2 | App generates a unique shareable link per session | — |
| FR-3 | App computes and displays cost per person (total ÷ spots) | — |
| FR-4 | Opening the link shows the live session list: confirmed players (in join order), bench (in join order), each with payment status | P1 |
| FR-5 | Joining is a single tap + name entry. Server assigns the spot atomically and in order — two simultaneous joins can never produce two "last spots" or lose a name | P1 |
| FR-6 | If the session is full, the joiner is placed on the bench automatically | P1 |
| FR-7 | When a confirmed player leaves or is removed, the first benched player is auto-promoted and notified | P1 |
| FR-8 | Every mutation (join, leave, remove, promote, payment change) is applied server-side as a single source of truth; all open clients update in real time (no refresh, no copy-paste) | P1 |
| FR-9 | Host can remove any participant; removal is visible, never silent | P1 |
| FR-10 | Participant taps "I have paid" → status becomes pending confirmation | P2 |
| FR-11 | Host taps ✅ (confirm → paid) or ❌ (reject → back to unpaid) | P2 |
| FR-12 | Payment status is shown inline next to each name — there is no second list anywhere | P2 |
| FR-13 | No money moves through the app. No screenshots, no notes. Disputes happen outside | P2 |
| FR-14 | Host sign-up/login (email + password for MVP) | — |
| FR-15 | Guests join with name only; a lightweight device token keeps their identity stable across app restarts | — |

### Non-Functional Requirements

| Req ID | Requirement |
|--------|-------------|
| NFR-1 | Multiplatform, mobile-first: one codebase → iOS + Android. Everyone joins through the app — no web version of the session page |
| NFR-2 | Real-time: list updates visible to all viewers within ~1 second |
| NFR-3 | Concurrency-safe: spot allocation handled by server-side transaction, never client-side |
| NFR-4 | Offline-tolerant reads: last known list visible offline; writes require connectivity |
| NFR-5 | Simplicity budget: a guest must be able to join in under 15 seconds from tapping the link |

### Session Lifecycle Rules (derived, applies to FR-1/FR-4)

| Rule ID | Rule |
|---------|------|
| LR-1 | Before start time: list fully visible, joins open |
| LR-2 | At start time (or when host manually closes/cancels): the list disappears — the entire page shows only "[Host name]'s session has already started" |
| LR-3 | After start time + 2 hours: page shows only "[Host name]'s session has ended" |

---

## Table 2 — Feature Breakdown

| Feature ID | Feature Description | Derived From | Priority |
|------------|---------------------|--------------|----------|
| **F1 — Session Creation & Sharing** | | | |
| F1.1 | Host sign-up / login (email + password) | FR-14 | High |
| F1.2 | Create session form (date/time, venue, total cost, capacity, with validation) | FR-1 | High |
| F1.3 | Cost per person, computed and shown on the session (updates live on edit) | FR-3 | Medium |
| F1.4 | Unique shareable deep link per session (opens app, or App/Play Store if not installed) | FR-2, NFR-1 | High |
| **F2 — The Live List (P1)** | | | |
| F2.1 | Session list view: session details, confirmed players in join order, bench in join order | FR-4 | High |
| F2.2 | Guest identity: silent device token on first open + display name remembered across sessions | FR-15 | High |
| F2.3 | Join action: one tap + name, server transaction assigns the next spot atomically | FR-5, NFR-3 | High |
| F2.4 | Auto-bench: joiner placed on bench with position when session is full | FR-6 | Medium |
| F2.5 | Real-time sync: every open screen updates within ~1s, no refresh | FR-8, NFR-2 | High |
| **F3 — Leave, Remove & Auto-Promotion** | | | |
| F3.1 | Participant leave: frees the spot, list updates on all devices | — (list integrity) | High |
| F3.2 | Host remove: host removes any participant, visible to everyone, never silent | FR-9 | High |
| F3.3 | Auto-promotion: first benched player moves up in the same transaction when a spot opens; push notification deferred post-MVP (live list is the notification) | FR-7 | Medium |
| **F4 — Payments (P2)** | | | |
| F4.1 | "I have paid": participant taps → status unpaid → pending (can untap) | FR-10 | High |
| F4.2 | Host confirm / reject: ✅ → paid, ❌ → back to unpaid | FR-11 | High |
| F4.3 | Inline payment badge next to every name (⚪ unpaid / 🟡 pending / 🟢 paid) — no second list | FR-12, FR-13 | High |
| F4.4 | Paid counter header on the session page ("7/12 paid") | — (host convenience) | Low |
| **F5 — Host Session Management** | | | |
| F5.1 | Host home screen: my upcoming sessions on top, past below | FR-1 (supporting) | High |
| F5.2 | Edit session details (time/venue/cost/capacity), changes live everywhere; capacity can't drop below current confirmed count | FR-1 (supporting), FR-8 | Medium |
| F5.3 | Session lifecycle & join deadline: before start = list visible + joins open; at start time or manual close = page shows only "[Host name]'s session has already started"; after +2h = only "[Host name]'s session has ended" | LR-1, LR-2, LR-3 | High |
| F5.4 | Duplicate session: pre-filled create form from a past session | — (host convenience) | Low |
| **F6 — Guest Experience Polish** | | | |
| F6.1 | Offline reads: last known list visible without connection; writes blocked with clear message | NFR-4 | Low |
| F6.2 | Loading, empty, and error states with plain-language text on every screen | — (UX quality) | Low |
| F6.3 | 15-second join pass: measure and cut taps on the link → joined path | NFR-5 | Low |

---

## Build Order

All High-priority features first — that is the true MVP (14 features):

```
F1.1 → F1.2 → F1.4 → F2.1 → F2.2 → F2.3 → F2.5
→ F3.1 → F3.2 → F4.1 → F4.2 → F4.3 → F5.1 → F5.3
```

Then all Medium, then all Low.

## Explicitly Out of Scope (MVP)

Recurring sessions, team splitting, reminders, in-app payments, chat, screenshots/notes on payments, web version of the session page, marketing/explainer website.
