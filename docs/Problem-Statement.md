
# Problem Statement

Amateur sports sessions are organized through copy-pasted lists in group chats (typically WhatsApp). To claim a spot, each player copies the entire list, pastes it, adds their name, and sends it back — hoping they copied the most recent version. This produces two recurring failures:

**P1**:Silent name loss. Someone copies an outdated version of the list and a previously-registered name silently disappears. There is no single source of truth, so "the real list" is whichever copy each player happens to have.

**P2**:Parallel payment tracking. Payment is tracked with a second, parallel copy-paste list — with the identical failure modes (outdated copies, silent loss, conflicting versions).
