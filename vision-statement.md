# Gplay — Vision Statement (Investor Pitch)

Every week, millions of amateur football sessions are organized the same way: a list in a WhatsApp group chat. To claim a spot, each player copies the entire list, pastes it, adds their name, and sends it back — hoping they copied the most recent version. When two players claim the last spot simultaneously, two conflicting lists exist and the group argues over who was first. When someone copies an outdated version, a name silently disappears. Payment tracking requires a second, parallel list with the same copy-paste mechanics — and the same failure modes. The result is a coordination process built on a broken primitive: a shared document with no single source of truth, edited by a dozen people at once, buried under hundreds of messages.

Gplay replaces this with a single live session page. The host creates a session — date, venue, cost per player, capacity — and shares one link. Players join with a single tap, no account required. Spots are allocated atomically and in order: simultaneous joins are resolved by the platform, not by debate, and overflow players are automatically queued on a bench list. Payment confirmation is built into the same view: players settle directly with the host through whatever channel they already use, mark themselves as paid, and the host verifies with one tap. Everyone sees the same real-time state — who's confirmed, who's unpaid, who's next in line — and when a player drops out, the next person on the bench is promoted and notified automatically.

Crucially, Gplay never touches money and doesn't try to replace the group chat — it integrates with it. It does one thing exceptionally well: it turns a fragile, error-prone social ritual into a reliable, transparent system of record. The session list is the wedge; the opportunity is to become the default coordination layer for amateur sports — where organization overhead, not demand, is what keeps games from happening.

**One-line version:**

> Gplay is the single source of truth for amateur sports sessions — one link replaces the copy-pasted WhatsApp list, with atomic spot allocation, built-in payment confirmation, and automatic bench management.

---

## Product decisions (locked)

- **Accounts:** Host must have an account. Participants join as guest (name only) or sign up.
- **Payments:** Always outside the app. Player taps "I have paid" (no notes/screenshots) → host confirms ✅ or rejects ❌. Disputes handled outside the app.
- **Cost:** Fixed split — total session cost ÷ spots, shown per person.
- **Overflow:** Bench/waitlist. When a confirmed participant cancels or is removed, the first benched player is auto-promoted and notified; they enter as unpaid and go through the normal payment flow.
- **Host powers:** Can remove any participant (including confirmed).
- **Out of scope (for now):** recurring sessions, team splitting, reminders, in-app payments.
