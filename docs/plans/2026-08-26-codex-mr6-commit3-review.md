# Codex review: MR-6 commit 3 — HTLC claim funding (design pivot + 5 rounds to PASS)

Commit: `bark: HTLC claim funding, riding LDK's own claim machinery`
(bark-stage1 `f1229bd0f` at PASS).

This commit was REDESIGNED mid-review at Greg's direction. The review
record therefore has two arcs: the abandoned durable-ledger design
(8 rounds, never passed) and the trust-LDK design that replaced it
(5 rounds to PASS). The pivot itself is the most important artifact.

Batteries at PASS: workspace units 627/627 (incl. 14 claim tests),
bark-channels 23/23, bark-sdk channels 19/19, server channel 22/22.

## The design pivot

The original c3 contract (design note rev6) specified durable
ClaimId-keyed jobs with deterministic, store-committed coin selections.
Eight codex rounds hardened it — ~30 accepted findings — and each round
kept surfacing new corners of the same shape: attempt lineage vs
coverage, RBF floors vs LDK's 5×-capped ratchet, batch-boundary
selection ids, victim invalidation. The root cause, surfaced by Greg's
question ("why don't we just trust LDK to hand us the right package at
the required feerates?"): the durable ledger re-implemented mempool
semantics beside LDK's own claim state machine.

What LDK already owns (all verified against pinned 0.2.4):

- The ChannelMonitor persists claim intent and REGENERATES a fresh
  `HTLCResolution` event — current descriptor set, current target — on
  connected blocks (timer-gated at 15/3/1 blocks by deadline proximity,
  `package.rs:1431`) and on `rebroadcast_pending_claims`
  (`chainmonitor.rs:900`, queue-only — see trust-r1 below), shrinking
  the set as pieces resolve and stopping when the chain settles it.
- Mempools arbitrate between successive attempts: every attempt spends
  the same HTLC outpoints, so any two are RBF rivals by construction,
  and either confirming is ours. Deactivating/retaining/refusing our own
  predecessors — the durable design's whole conflict layer — duplicated
  that arbitration, badly: LDK's target ratchet is capped at 5× the
  live estimate (`package.rs:1553`), so "the ratchet will clear the RBF
  floor" was refuted, and every repair grew the corner count.

The trust design: bark keeps NO attempt ledger, no relay loop, no
replacement bookkeeping, no durable selections. Each regenerated event
is handled once — build, register with the BDK wallet, broadcast,
forget. A crash forgets the in-memory locks; the next regeneration
re-selects fresh; a rival of a still-mempooled earlier attempt is
mempool-arbitrated churn, never loss. This is the upstream-intended
integration shape. The commit shrank by roughly half.

What bark still adds is POLICY, not machinery:

1. **The budget** (the design note's stake-ceiling posture): wallet
   contribution ≤ claimed value; actual fee + sweep reserve ≤ claimed
   value, priced on the real change decision (sub-dust burn included)
   with a size-identical probe script so refusals burn no address.
2. **LDK's own arithmetic**, mirrored exactly in the
   `CoinSelectionSource` (weights, eligibility ladder incl. forced
   conflicting spends, change math, `max_tx_weight` window,
   `bump_transaction/mod.rs:461-668`) so LDK's builder assertions hold
   and a selection failure means what LDK expects (shrink and retry,
   `mod.rs:1078`).
3. **Operator visibility**: one durable table — a blocked marker set
   when a pass leaves any claimed output unfunded, cleared by the first
   fully-covering pass. LDK's targets only ratchet upward; blocked is an
   operator state, not a wait.

## Trust-design round trail (5 rounds)

R1 (2 P1/1 P2): `rebroadcast_pending_claims` only QUEUES events (the
driver moves them on its next drain) and block regeneration is
timer-gated — a claim could miss its last pre-timeout pass → the claim
pass now runs an EMPTY feed/drain barrier after the nudge, so the same
pass handles what it provoked. Sibling batches could force-steal each
other's fee coins (LDK derives per-batch utxo ids, `mod.rs:1092`) while
any nonempty capture cleared the marker → per-pass used-set excluded on
EVERY rung, shared-wallet-input validation belt, coverage-based marker.
Budget burn aborted instead of trying a change-producing coin.

R2 (3 P1): registered attempts hid their coins from same-id RBF (BDK's
`list_unspent` filters spent-by-unconfirmed) → `claim_bump_utxos`:
confirmed coins whose EVERY canonical spender is unconfirmed, folded
back only when lock-known; dead locks pruned by the same sweep.
With-change breaches also retry; the search bound became the candidate
count.

R3 (3 P1/1 P2): smallest-only eviction wedged claims needing a
largest-eviction; immature coinbase outputs funded consensus-invalid
claims (both wallet views now filter maturity, matching BDK's own
builder, `tx_builder.rs:656`); marker persistence `?` could suppress
already-built claims (registration/broadcast now precede marker writes,
which only log); a STALE lock could reclassify an ordinary pending send
as bump funding → the tracker records every broadcast claim txid, and
fold-back requires the coin's unconfirmed spender to be OURS.

R4 (1 P1): the both-direction BFS frontier exhausted its linear cap
before deep chains completed (twelve burning near-dust coins picked in
multiples hid a large coin at depth 11) → two deterministic SPINES
first (pure smallest-eviction, pure largest-eviction, marching at the
plan's front), off-spine branches seeding a deduplicated mixed BFS
tail; budget 3×candidates+3. Regression test pins the deep chain.

R5: PASS ("both spines complete before BFS; 3n+3 leaves ≥ n+1 BFS
attempts; the expect is unreachable").

## The abandoned durable-design arc (8 rounds, compressed)

R1 (10 P1/2 P2): multi-batch txs discarded; irreversible terminality;
payload-change races; max_tx_weight ignored; LDK weight-contract
violations; estimate-only budget; locks invisible to the wallet;
"waiting for lower rates" unrecoverable; genesis-to-tip scans;
predecessor attempts misclassified. R2 (10 P1/1 P2): RBF coin reuse;
reuse bypassing weight/budget; consumed-coin reuse after re-batching;
sibling coin sharing; victim invalidation; crash windows; settle
atomicity; reopen semantics; stale-wallet gating; partial-batch
markers. R3 (4 P1/2 P2): payload bumps losing the RBF coin; CAS gaps;
send-vs-claim races; reorged coin creation; retained-tail discards;
address burn. R4 (4 P1/1 P2): RBF absolute-fee floor; dead coins
retried once; probe floors vs reorged commitments; orphaned batch-id
selections. R5-R7: the retention-of-conflicting-predecessors design
sprouted fee-set, lock and coverage corners each round. R8 (1 P1/1 P2):
stale shrunk-payload attempts poisoning floor and coverage; forced
steals bypassing the floor. At that point the trust question was asked
and answered.

Design lesson, recorded for the arc: the durable machinery was hardened
honestly and still diverged, because its FOUNDATION duplicated a
consensus mechanism (mempool RBF arbitration + LDK's regeneration) that
cannot be safely mirrored piecemeal. The related asymmetry stands
deliberately: channel-close commitments/anchors stay on bark's exit
CPFP machinery because they live inside VTXO exit chains LDK cannot
fee-manage; HTLC claim txs are ordinary wallet transactions, which is
exactly why trusting LDK works for them.

## Accepted residuals (on the record)

- Post-crash selection churn / self-rivalry (mempool-arbitrated), and
  post-crash fold-back refusal of pre-crash attempts' coins until those
  attempts confirm or BDK eviction returns the coins.
- No durable claim journal beyond the blocked marker.
- Wallet coins stranded under an unconfirmed attempt until it or a
  rival confirms (upstream-identical posture).
- A blocked-marker row lingering after an externally-resolved claim
  (cleaned with the channel).
- The wallet-wide send-vs-claim reservation layer (cross-component
  stage-1 residual; `register_tx` + the tracker narrow it).
- Startup-tick latency bounded by the daemon interval.
- The budget search is a bounded heuristic (spines + BFS); the blocked
  marker is the honest fallback.
- Full-drive e2e defers to the payments e2e commit (needs the
  server-relay seam); this commit pins selection, tracker, marker and
  arithmetic at unit level (14 tests).
- c2 side-note surfaced by the trust audit: the durable Filter
  registration table is belt (monitors re-register all watches on every
  reload) — a later simplification candidate, not worth re-litigating a
  hardened layer now.
