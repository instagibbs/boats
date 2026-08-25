# Codex review: MR-5 commit 4 — "bark: the symmetric watch and the deadline rungs" (10 rounds to PASS)

Commit `daa645d03` on `ark8-channels-stage1-close` (bark-stage1);
rewritten 2026-08-25 to `5872d8537` — the rival probe excised in history,
see the historical note under Key design decisions and the commit-6
record. The final stack tree is unchanged. Battery
at PASS: sdk channels 18/18 (incl. the cooperative deadline rung with no
exit started, the full respond-by-exit race — server DEAD — to the
on-chain checkpoint and the scope-ended record, the reopen ladder to the
open headroom guard, and the hard-line PONR-never-crossed test), server
channels 22/22, units 412/412.

## Round 1: FAIL — 4 P1, 1 P2, 1 P3

1. **P1 server disconnection disabled the safety watch** (the watch and
   rungs lived inside the connectivity-gated channels sync). → the
   chain-safety duties (watch, rungs, exit reconcile) run UNGATED in the
   daemon loop and in the manual `Wallet::sync`; the respond e2e stops
   the server before the respond phase.
2. **P1 a sanctioned sub-dust user share could not respond, and one dust
   record starved every later watch.** → the RESPONSE is broadcast
   DIRECTLY: the retained conflict-winning checkpoint, P2A-bumped —
   never dependent on leaf standardness or store state; records handled
   in isolation; generic leaf exits remain (downgraded-only, ≥ dust) as
   the claims' bookkeeping.
3. **P1 a losing replacement was recorded as the accepted child**
   (InsufficientReplacementFee classified as benign conflict), freezing
   escalation. → only success or exact AlreadyKnown attaches;
   replacement-fee failures evict and escalate; the escalation increment
   is the node's `incrementalfee` (ceil-converted; 1 sat/vB fallback).
4. **P1 the close driver could cross the PONR after the hard line.** →
   the rungs run before the (gated) close driver each tick, and
   `run_split_cosign` re-checks a FRESH tip immediately before the
   point of no return — inside the hard line the crossing is REFUSED
   (e2e pins record-ends-exiting with no retained split).
5. **P2 every accepted child was replaced every tick.** → the standing
   package carries PROVENANCE: our own accepted child stands unless the
   target rate rose above it; only rejections or visible rivals
   escalate.
6. **P3 unchecked hard-line arithmetic.** → saturating sums.

## Round 2: FAIL — 3 P1, 1 P2, 1 P3

Ordering and lifecycle: the rungs moved BEFORE the gated close driver in
the loop; `tip_now` (uncached) feeds the rungs and the PONR recheck; the
response package became durably managed (persisted child association,
own-child provenance pricing, parent-confirmed rebuilds); ambiguity no
longer escalates the bid; `incrementalfee` ceil.

## Round 3: FAIL — 1 P1, 2 P2

The scope-end additionally requires the RESPONSE settled (else the
package keeps driving even past leaf scope); parent-confirmed rebuilds
got typed single-tx rejection handling (`broadcast_tx_typed`, shared
classifier); persistence failures compensate with eviction (the wallet
application precedes the fallible persist).

## Round 4: FAIL — 1 P1, 1 P2

At `registration_pending` the response broadcasts only while the FINAL
is unconfirmed (a fresh response after it can never qualify under the
server's same-block rule — it would only strand the closing fallback);
THE CHAIN OVERRULES THE TOMBSTONE: a response that confirmed despite
FALLBACK_WON installs the owned leaves (`downgraded`) — in the refusal
arm, the replay arm, and the watch; a rejected replacement never
clobbers the standing association; settlement gains the anchor-spent
probe for association-less confirmed children.

## Round 5: FAIL — 1 P1, 3 P2

The rescue reaches `exiting` records too (watched while a retained
downgrade exists; the install CAS widened) and runs under the exit lock
(suspending/untracking the live fallback whose bridge the response
spent); the scope-end uses the settlement probe; the probe uses the
cross-backend confirmed-only spend check; movement resolution
centralized (`resolve_close_movement`, stable action id, produced
leaves + effective balance).

## Rounds 6–9: FAIL — crash-window closure

The first (and any dead-predecessor-successor) child association is
persisted BEFORE the wallet store — no crash leaves a wallet-held,
input-locking child no durable record names; a NotFound standing child
is EVICTED before its successor is built (single-UTXO starvation); the
`Closed` replay is gated on the RECORDED resolution; the close
movement's verdict is FULLY deferred — the rescue resolves it
Successful, the terminal exit reconcilers fail it exactly once `exited`
is recorded (both reconcilers; pending-only, idempotent); the fee view
refreshes ungated before the safety duties (never a target frozen at
disconnect time).

## Round 10: PASS — no findings

## Key design decisions (for the arc record)

- **The symmetric watch is chain-safety machinery**: ungated on server
  connectivity, fee view refreshed first, run before any gated close
  advancement, manual sync included.
- **Respond-by-exit, two-layered**: the retained conflict-winning
  checkpoint broadcasts DIRECTLY (P2A-bumped, escalating, standardness-
  and store-independent, `registration_pending` on); the generic leaf
  exits carry the claims where the leaves are wallet VTXOs.
- **Race discipline**: unregistered, respond only while the final is
  unconfirmed (the same-block rule); registered, the armed-watch window
  applies. THE CHAIN OVERRULES THE TOMBSTONE everywhere FALLBACK_WON is
  handled.
- **Deadline rungs**: cooperative rung at `hard + channel_close_lead`
  (144 default); the hard line (fresh-tip, uncached) owns `ready`, the
  pre-settlement phases and the sanctioned fallback; no rung past the
  PONR; the PONR itself re-checks a fresh tip and refuses inside the
  hard line.
- **Outbid, never wait**: blind escalation per unaccepted attempt
  (node-derived increment) — never gated on local conflict visibility,
  and never priced off a conflicting package's advertised feerate
  (attacker-controllable input); our own accepted child stands until the
  target rises; ambiguity never escalates; every non-attached child is
  evicted before the lock releases.
  *(Historical note: as reviewed, this commit also floored the bid just
  above a VISIBLE rival's package via `gettxspendingprevout`. Greg
  rejected pricing off external inputs; the probe was excised from this
  commit in the stack rewrite of 2026-08-25 — see the commit-6 record —
  so the committed history never carries it.)*
- **Deferred movement verdicts**: a close's movement resolves only where
  the resolution is recorded (rescue → Successful; terminal exit
  reconcilers → Failed on `exited`; `Closed` replay resolution-gated).
- **The reopen ladder ends at the OPEN's headroom guard** (+1 upgrade,
  +2 close under the pinned bound) — the close-side depth refusal is
  defense-in-depth, unreachable through sanctioned opens; the ladder
  ends in plain Ark balance, never a stuck channel.
- **Accepted residuals**: no e2e drives a real sub-dust user share
  (unreachable without payments), the F5 own-child-skip and the rescue
  orderings are machinery-verified only.

## Follow-up fold (2026-08-25, from the surface-work design review)

Folded into this commit (rewritten `5872d8537` → `16e7101ba`): the
MANUAL doors now enforce the hard fallback line on a FRESH tip exactly
like the rungs — `close_channel` refuses before `shutdown` (P1: the
manual API could destroy a usable channel when cooperative settlement
was already forbidden at the PONR anyway), and `cancel_channel_exit`
refuses inside the line (a restored channel could neither close nor
re-enter its exit in time). Two e2e vectors added: the manual refusals
(daemon stopped so the doors face the fresh tip themselves), and the
close-origin exit's cancel refusal at the end of the hard-line race
test.
