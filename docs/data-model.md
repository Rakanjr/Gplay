# Gplay — Data Model

The single source of truth for what Firestore stores. Every field here is justified by a requirement in `docs/features.md`; if it's not here, it's derived (computed on screen), not stored.

**Core principle:** the phone displays, the server decides. Clients read documents and call Cloud Functions; only Cloud Functions (and scheduled jobs) write sessions, participants, and events.

---

## Design Rules (locked)

1. **Store events, compute math.** Anything that *happened* (a join, a close, a rejection) is stored. Anything that is a pure function of stored data (cost per person, counts on an open list, "ended" display state) is derived on screen — it can never go stale.
2. **State vs. event log.** Every entity's document holds *current state* only. History lives in the append-only `events` sub-collection (the activity feed). FR-9 "never silent" is satisfied by the event log, not by ghost entries in the list.
3. **Denormalize only summaries you need without loading the detail.** The session document carries counters for preview screens (host home); the open session page derives its counts from the participant snapshot it already holds.
4. **Strict order comes from a server counter, not timestamps.** `order` is assigned from `joinCounter` inside the join transaction — no ties, ever. `joinedAt` is kept for display/audit only.

---

## Collections

### `users/{uid}`

One document per account — hosts (email+password) and guests (anonymous) alike. `uid` is the Firebase Auth ID.

| Field | Type | Written by | Notes |
|---|---|---|---|
| `displayName` | string | client (once) | remembered across sessions (F2.2) |
| `isHost` | bool | client (on sign-up) | guests upgrade later by merging accounts |
| `createdAt` | timestamp | client (first write) | |

### `sessions/{sessionId}`

| Field | Type | Written by | Notes |
|---|---|---|---|
| `hostId` | uid | create fn | |
| `hostName` | string | create fn | denormalized for the "X's session has started" screens (LR-2/3) |
| `venue` | string | create/edit fn | |
| `startTime` | timestamp | create/edit fn | |
| `capacity` | int | create/edit fn | edit fn rejects capacity < `confirmedCount` (F5.2) |
| `totalCost` | number | create/edit fn | |
| `sessionStatus` | enum | **server only** | `open` · `closed` · `cancelled` |
| `closedAt` | timestamp? | **server only** | set on manual close/cancel |
| `confirmedCount` | int | **server only** (transaction) | for preview screens |
| `benchCount` | int | **server only** (transaction) | for preview screens |
| `paidCount` | int | **server only** (transaction) | for "7/12 paid" header & previews (F4.4) |
| `joinCounter` | int | **server only** (transaction) | hands out strict join order |
| `createdAt` | timestamp | create fn | |

**Derived, never stored:** `costPerPerson = totalCost ÷ capacity` (FR-3 — updates live on edit for free). The "has started / has ended" screens (LR-2/3) are derived from `sessionStatus` + `startTime` (+2h); a `closed`/`cancelled` session can never be re-derived from time alone, which is why `sessionStatus` is stored.

### `sessions/{sessionId}/participants/{uid}`

The link between a user and a session. One doc per membership; document ID is the user's `uid` (a user appears at most once per session).

| Field | Type | Written by | Notes |
|---|---|---|---|
| `displayName` | string | join fn | snapshot of the name at join time |
| `userStatus` | enum | **server only** | `confirmed` · `benched` · `left` · `removed` |
| `order` | int | join fn (transaction) | from `joinCounter`; strict, tie-free ordering |
| `joinedAt` | timestamp | join fn | display/audit |
| `payment` | enum | **server only** | `unpaid` · `pending` · `paid` |
| `paymentUpdatedAt` | timestamp? | payment fns | |

Rules:
- The visible list = `userStatus IN (confirmed, benched)`, sorted by `order` within each group. Bench position number = index in that sorted bench (derived, never stored).
- `left` / `removed` docs are hidden from the list but **persist until session end** (FR-13 dispute record, FR-9 audit). Both may re-join: the doc resets to active with a **new** `order` (back of the queue) — no jumping back to an old spot.
- Payment controls are shown only for `confirmed` players. Promotion (F3.3) and host-bench (F3.5) carry `payment` over unchanged.
- A rejected payment is not a status: `payment` just returns to `unpaid` (badge flips back; post-MVP F7.3 adds a push).

### `sessions/{sessionId}/events/{eventId}`

Append-only activity feed of **membership changes only** — rendered on the session page below the list. **Server writes, everyone reads. Never updated, never deleted** (until whole-session deletion at end). Payment changes are not logged here; they are visible inline as badges.

| Field | Type | Notes |
|---|---|---|
| `type` | enum | `join` · `leave` · `remove` · `promote` · `bench` |
| `actorUid` / `actorName` | uid / string | who did it (the host, or the participant themselves) |
| `targetUid` / `targetName` | uid / string? | who it happened to (remove/bench/promote/payment decisions) |
| `at` | timestamp | server timestamp |

Display strings are composed on the client from `type` + names, e.g. `remove` → "Khalid was removed by the host · 18:50".

---

## Who may write what (preview of security rules)

| Collection | App (client) can write | Cloud Functions (server) can write |
|---|---|---|
| users | own document only | yes |
| sessions | no | only this way |
| participants | no | only this way, inside transactions |
| events | no | only this way, append-only (never edited or deleted) |

This table is why FR-5 and NFR-3 hold: there is no client-side path to a list mutation at all.

## Data lifecycle (from tech-stack.md, unchanged)

- Anonymous auth account: deleted on session end/cancel, on leave-with-zero-other-active-sessions, or by scheduled cleanup.
- Participant docs (incl. `left`/`removed`) and events: deleted with the session at end.
- Session end = whole `sessions/{id}` tree deleted (scheduled job).
