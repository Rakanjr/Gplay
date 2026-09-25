# Flow 3 — Participant paid / untap (F4.1)

The participant taps "I have paid" (`unpaid` to `pending`) or taps again to undo (`pending` back to `unpaid`). This is the simplest flow in the system: it writes a single document, has no contention (only the owner ever writes their own `payment` field), and touches no counters and no event log.

## Steps

1. **App:** participant taps "I have paid" (or taps again to undo), and the app calls the togglePayment Cloud Function with the session ID. Identity comes from the auth token.
2. **CF:** starts a transaction; reads the session doc and the participant doc. If the session doc does not exist, abort and return "session not found."
3. **CF / validate:** session `sessionStatus` is `open`; the participant doc exists and `userStatus` is `confirmed`. (The UI only shows payment controls to confirmed players, but the server re-verifies every UI assumption: a stale or tampered client could send the call for a benched player.)
4. **CF / validate transition:**
   - tap: current `payment` must be `unpaid`, set to `pending`
   - untap: current `payment` must be `pending`, set to `unpaid`
   - `paid`: abort and return "payment already confirmed by host" (the participant cannot touch `paid`; only the host's reject can move it, in Flow 4)
5. **CF:** set `paymentUpdatedAt` = server timestamp. No event-log entry (payment is visible inline as the badge; the log is membership-only). No counters (`paidCount` counts host-confirmed payments only, so `pending` does not touch it). No notifications (post-MVP F7; the live badge is the signal).
6. **CF / commit:** the single-document write commits atomically; on conflict, retry from step 2.
7. **Firestore:** pushes the new state to every open client (about 1 second).
8. **All clients:** the badge next to that name flips (`unpaid` / `pending`) for everyone; on the host's screen the confirm/reject buttons appear next to the pending badge.

## Why it works

- **Server-side validation of what the UI hides.** Payment controls are a confirmed-player feature; the server checks `userStatus: confirmed` itself rather than trusting the screen.
- **Transition validation makes taps idempotent.** A double-tap, a network retry, or a stale screen can never produce an illegal jump (for example `paid` back to `pending`); every call must start from a legal state.
- **The badge replaces a notification system.** The host learns about the pending payment because their screen is subscribed to the same live list (F2.5). No event entry, no push, no extra machinery.
- **Single-document write.** Unlike join/leave, nothing else must stay consistent with this change, so there is no multi-document transaction risk.

## Legend

- CF: Cloud Function (server-side code)
- atomic: all writes succeed together or none do
- badge: the inline payment indicator next to a name (unpaid / pending / paid)
