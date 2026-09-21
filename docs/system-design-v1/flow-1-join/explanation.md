# Flow 1 — Join (FR-5, NFR-3)

The join is the heart of P1: two simultaneous joins can never collide, lose a name, or produce two "last spots." The guarantee comes from running the entire decision inside one server-side transaction.

## Steps

1. **App:** participant taps Join, enters name, and the app calls the `join` Cloud Function with the session ID and name.
2. **Cloud Function:** starts a transaction and reads the session document. Everything until the commit is atomic. If the document does not exist, return "session not found."
3. **CF / validate:**
   - session `sessionStatus` must be `open`, otherwise return "session closed"
   - no active participant doc with this uid in this session, otherwise return "already joined" (one active membership per user per session; membership in other sessions is unaffected)
4. **CF / assign:**
   - if `confirmedCount < capacity`: `userStatus: confirmed`, `confirmedCount += 1`
   - else: `userStatus: benched`, `benchCount += 1`
5. **CF / order:** `order = joinCounter + 1`; `joinCounter += 1`. Strict, tie-free join order, independent of network timing.
6. **CF / write participant:**
   - fresh join: create the doc (name, `userStatus`, `order`, `joinedAt` = server timestamp, `payment: unpaid`)
   - re-join (a `left`/`removed` doc with this uid exists): update that doc with the new `userStatus`, a new `order`, a new `joinedAt`, and `payment` reset to `unpaid` (a new join is a new commitment)
7. **CF / event:** append one `events` entry: `type: join`, actor = this user, server timestamp. This is the activity-feed line.
8. **CF / commit:** counters, participant doc, and event commit as one atomic unit. On conflict the transaction retries from step 2.
9. **Firestore:** pushes the new state to every open client (about 1 second).
10. **All clients:** list re-sorts by `order`, event log gains a line, counts re-derive from the snapshot.

## Why it works

- The capacity check and the spot assignment happen inside the same atomic transaction, so simultaneous joins serialize: the second transaction always sees the state the first one committed.
- Join order comes from `joinCounter`, an integer incremented inside the lock, never from timestamps (which can tie) or network timing (which is arbitrary).
- Document IDs are unique per collection, and the participant doc ID is the user's uid, so a user can never hold two entries in the same session. Re-joins reactivate the existing doc at the back of the queue.

## Legend

- CF: Cloud Function (server-side code)
- atomic: all writes succeed together or none do
- transaction: a locked read-modify-write operation; Firestore retries it automatically if the data changed underneath it
