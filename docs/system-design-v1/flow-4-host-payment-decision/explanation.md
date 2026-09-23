# Flow 4 — Host confirms or rejects payment (F4.2)

The host taps Confirm (`pending` to `paid`) or Reject (`pending`/`paid` back to `unpaid`) on a player's payment. This is the first flow restricted to a specific caller (the host), and the first flow that writes two documents in one transaction: the participant's `payment` plus the session's `paidCount`.

## Steps

1. **App:** host taps Confirm or Reject next to a `pending` badge, and the app calls the payment-decision Cloud Function with `sessionId`, `targetUid`, and the `action` (`confirm` or `reject`). Inference is impossible here — from `pending` the host has two legal moves — so the action travels with the call. Caller identity comes from the auth token, never from the client payload.
2. **CF:** reject unsigned callers; the auth token yields `actorUid`.
3. **CF:** starts a transaction; reads the session doc and the target participant doc (`sessions/{sessionId}/participants/{targetUid}`). All reads before any write. If the session doc does not exist, abort and return "session not found."
4. **CF / validate session:** `sessionStatus` must be `open`, and `session.hostId` must equal `actorUid` — host-ness is a fact about the session doc, not the auth token and not the `isHost` flag on the user doc (that flag only marks host-type accounts). A mismatch aborts and returns "only the host."
5. **CF / validate target:** the participant doc must exist and `userStatus` must be `confirmed`. Payment controls belong to confirmed players; the server re-verifies the UI assumption rather than trusting the screen.
6. **CF / validate transition:**
   - confirm: current `payment` must be `pending`, set to `paid`. If already `paid`, return silently (goal state already true; retry-safe). If `unpaid`, abort and return "player hasn't marked as paid."
   - reject: current `payment` must be `pending` or `paid`, set to `unpaid` — this is the only path that reverses a confirmed payment (mistake-correction). If already `unpaid`, return silently (also covers the player-untap race).
7. **CF:** writes the participant doc (`payment`, `paymentUpdatedAt` = server timestamp) and the session doc (`paidCount`: +1 on confirm, −1 on reject-from-`paid`, unchanged on reject-from-`pending`). Silent-success returns write nothing. No event-log entry (the log is membership-only; payment is visible inline as the badge). No notification (post-MVP F7; the live badge is the signal).
8. **CF / commit:** the two-document write commits atomically; on conflict, retry from step 3.
9. **Firestore:** pushes the new state to every open client (about 1 second).
10. **All clients:** the badge flips to `paid` (confirm) or back to `unpaid` (reject), and the "7/12 paid" header (F4.4) moves with `paidCount`.

## Why it works

- **Host authorization is a session fact.** `session.hostId == actorUid` is checked inside the transaction, against the same snapshot the commit is built on. The `isHost` field on `users/{uid}` means "host-type account," and the auth token carries identity only — neither can answer "host of this session."
- **Explicit action, no inference.** Flow 3 could infer the transition because each state had exactly one legal move. From `pending` the host has two; the client must say which.
- **First two-document transaction.** The badge (`participants.payment`) and the header counter (`sessions.paidCount`) change together or not at all — they can never disagree.
- **Counter delta follows the transition.** Rejecting something never counted (`pending` back to `unpaid`) must not touch `paidCount`; only `pending` to `paid` adds and only `paid` to `unpaid` subtracts.
- **Silent success when the goal state is already true.** Confirm-on-`paid` and reject-on-`unpaid` return without writing, which makes retries and the player-untap race safe. Abort when the host's understanding is wrong in a way that matters (confirm on `unpaid`).
- **The badge replaces the machinery.** The live Firestore subscription carries the result to every client; no event entry, no push, no extra moving parts.

## Legend

- CF: Cloud Function (server-side code)
- actorUid / targetUid: the caller (from the auth token) / the player the action applies to (from the client payload)
- atomic: both writes succeed together or neither does
- badge: the inline payment indicator next to a name (unpaid / pending / paid)
- silent success: return without writing when the goal state is already true
