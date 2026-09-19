# Flow 2 — Leave with auto-promotion (F3.1, F3.3)

A participant leaves; if they were confirmed and the bench is non-empty, the first benched player is promoted in the same transaction. The freed spot and the promotion are indivisible: no observer can ever see an empty spot with a non-empty bench.

## Steps

1. **App:** participant taps Leave, and the app calls the `leave` Cloud Function with the session ID. Identity comes from the auth token, never from client-supplied data.
2. **CF:** starts a transaction; reads the session doc and this user's participant doc. Atomic until commit.
3. **CF / validate:** session `status` is `open`; the participant doc exists and is active (`confirmed` or `benched`).
4. **CF / leaver:** set participant `status: left`. The `payment` value is untouched: a paid leaver's record survives until session end for the host (FR-13).
5. **CF / branch:**
   - leaver was `benched`: `benchCount -= 1`; skip to step 7 (no spot opened, no promotion)
   - leaver was `confirmed`: `confirmedCount -= 1`; continue
6. **CF / promote (only if the bench is non-empty):** find the benched doc with the lowest `order`, set `status: confirmed` (it keeps its own `order`, so it sorts into the confirmed list exactly where its join time places it); `confirmedCount += 1`, `benchCount -= 1`.
7. **CF / events:** append `leave` (actor = leaver); if a promotion happened, append `promote` (actor = system, target = promoted player).
8. **CF / commit:** all writes as one atomic unit; on conflict, retry from step 2.
9. **Firestore:** pushes the new state to every open client (about 1 second).
10. **All clients:** the leaver disappears from the list into the event log; the promoted player appears in the confirmed list at their sorted position; bench positions re-derive with no extra writes.

## Why it works

- **No double promotion.** "Who is first on the bench" is read fresh inside the transaction. If two confirmed players leave simultaneously, the second transaction's commit fails (the session doc changed), it retries, re-reads, and promotes the next benched player. The same player can never be promoted twice.
- **No cascade writes.** Bench positions are derived (index in the sorted bench), never stored. A promotion writes one document; every client re-sorts and the new bench positions fall out automatically.
- **No copied order.** The promoted player keeps their original `order`. Copying the leaver's number would duplicate an `order` value and falsify join history.

## Legend

- CF: Cloud Function (server-side code)
- atomic: all writes succeed together or none do
- derived: computed on screen from data already loaded, never stored
