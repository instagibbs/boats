# Codex review: MR-5 commit 1 — "captaind: record the close outcome" (4 rounds to PASS)

Commit `381ce7a29` on `ark8-channels-stage1-close` (bark-stage1). Battery
at PASS: 18/18 server channel e2e (4 new), 119/119 server units,
postgres close-records suite, prechecks clean.

## Round 1: FAIL — 4 P1, 1 P2, 1 P3

1. **P1 capture-write failure loses the candidate while the close
   completes.** LDK broadcasts and queues `ChannelClosed` regardless of
   the broadcaster's outcome; a shutdown-and-return on write failure let
   the graceful-shutdown manager persist (channel removed, event
   pending) outlive the lost candidate → restart replays a close with no
   evidence, forever. → the broadcaster RETRIES the capture (blocking
   LDK's thread, persist-before-sign posture), requesting shutdown from
   the first failure.
2. **P1 "uniquely valid closing tx" is false.** A crash before the
   manager persisted the close restores the channel mid-negotiation;
   the reconnect renegotiates the SAME terminally-fixed balances into a
   differently-feed transaction — two legitimate fully-signed closing
   txs. → selection takes the LATEST-captured closing-shaped candidate
   (capture order = broadcast order; the completing tx always captures
   after everything it obsoletes), and the outcome record treats
   balances-equal/txid-different as an in-place REFRESH
   (`ChannelCloseOutcomeRecord::Refreshed`, loud log); only
   balance/keying differences remain the fail-closed contradiction.
3. **P1 a stale manager resurrects an outcome-bearing channel.** The
   outcome is durable before the ack, the manager persists
   asynchronously; a crash in between reloads the channel LIVE and a
   resumed channel could move the very balances the outcome fixed. →
   the terminal reconciler generalized to `reconcile_defunct_channels`:
   terminal rows AND outcome-bearing rows are logically force-closed,
   level-triggered, verified gone before peer acceptance. e2e restores
   the literal pre-close manager blob.
4. **P1 peer-forced zero-output close wedges the event queue.** Stock
   LDK's non-funder accepts a fee consuming the funder's whole balance
   (`force_close_avoidance_max_fee_satoshis` explicitly ignored for
   non-funders); with the server side sub-dust the closing tx is
   output-less. → selection tolerates 0..=2 outputs: the recorded
   balances come from the EVENT (exact under stage-1 invariants), the
   torched artifact is the funder's own fallback loss, and no stock-LDK
   non-funder fee cap exists (codex's preferred fix would reintroduce
   the fork). Driven end to end by computing the deterministic closing
   weight (674 WU) and landing the funder's fee in the sub-dust window.
5. **P2 terminal cleanup immediately undone; capture unindexed.** →
   capture restricted to live states + partial funding-outpoint index.
6. **P3 `closed_at` = replay-handling time.** → `closed_at` is the
   selected candidate's CAPTURE time; FK/composite constraints declined
   (this schema's integrity mechanism is the row-lock discipline, not
   referential actions) — accepted in R3.

## Round 2: FAIL — two refinements

- **P2 the retry's soundness argument crossed pools.** The capture rides
  the dedicated executor pool; a schedule exhausting it (blocked monitor
  writes) leaves the primary pool free to persist the removed-channel
  manager. → codex's own suggestion adopted: a **poisoned-capture
  latch** — the capture's bounded give-up (12 attempts) raises a shared
  flag and `ManagerStore::write` refuses every newer manager snapshot;
  the driver treats that as fatal, so the restart reloads a manager
  still holding the channel and the close renegotiates with a fresh
  capture. LDK's consistency read lock (spanning the broadcaster call)
  vs serialization's write lock makes in-flight writes safe.
- **P1 the live-only capture races the cosign boundary.** A pre-ready
  close can broadcast DURING admission's awaiting_upgrade→cosigned
  commit; a capture snapshot that excluded `awaiting_upgrade` misses it
  and the committed cosigned row's close wedges. → capture matches
  `awaiting_upgrade` too and takes `FOR UPDATE` on the matched rows,
  serializing against the admission commit and the terminal transition;
  the pre-cosign candidate cleanup restored.

## Round 3: FAIL — one residual

- **P2 the latch needs a durable baseline `ChannelPending` does not
  guarantee**: a peer establishing, cosigning AND closing before the
  driver's first post-ChannelPending persist leaves only a pre-channel
  snapshot; the poisoned restart force-closes the orphan monitor
  instead of renegotiating. A containment precondition at the upgrade
  cosign (durable blob names the channel) was implemented and REVERTED:
  it structurally breaks the seeded-row reference tests and couples
  admission liveness to driver persistence.

## Round 4: PASS

The residual is accepted as documented posture: it fails closed with no
false completion and no funds loss (the close is never reported
complete, the split never cosigned, the settlement degrades to the
spec-sanctioned unilateral fallback), the schedule needs the driver
starved for the channel's entire lifetime while the node otherwise
serves, and closing it would take exactly the durability-barrier class
this subsystem deliberately rejects. Codex: "The residual clears the P2
bar… The extreme scheduling requirement does not justify introducing
the rejected durability coupling."

## Post-review note: classification uniqueness (the CLN-bug question)

Is the "latest closing-shaped candidate" selection unique, or can an
adversary confuse it (cf. CLN's onchaind misclassification)? Verified
against LDK 0.2.4 source:

- The CLN bug's surface — classifying adversary-chosen CONFIRMED
  transactions by shape — does not exist here: the candidate set is
  exclusively our own LDK's broadcast hand-overs (the capturing
  broadcaster is the sole writer), every candidate carries our funding
  signature by construction, and everything that judges on-chain
  reality elsewhere in the MR matches exact recorded txids, never
  shapes.
- Within our own outbox, LDK signs the funding key on exactly two
  shapes, disjoint by construction: a commitment's input sequence and
  locktime carry `0x80`/`0x20` top bytes (the obscured commitment
  number fills only the low three bytes — no commitment number makes
  them MAX/zero), while the closing is `Sequence::MAX`/`LockTime::ZERO`
  (`chan_utils.rs` constants; both peers sign the SAME tx our LDK
  builds, and the peer's reach — its shutdown script, the fee — touches
  neither field). HTLC second-stage and justice transactions spend
  commitment outputs, never the funding.
- A "no P2A output ⇒ closing" belt was CONSIDERED AND REJECTED:
  `option_shutdown_anysegwit` makes the P2A script a legal shutdown
  script, so a malicious peer could put a P2A output inside a
  legitimate closing and wedge such a rule — the exact peer-controlled
  failure mode the zero-output tolerance avoids. (A peer paying its own
  balance to anyone-can-spend is self-harm, not our loss.)
  Sequence/locktime are precisely the fields a peer cannot reach.
- Either hypothetical misclassification direction is money-safe
  regardless: balances always come from the EVENT (the signed channel
  state), never the classified bytes, and downstream consumers match
  exact scripts (a commitment's CSV script never matches the wallet's
  shutdown P2WPKH — "nothing of ours", never a wrong claim).
