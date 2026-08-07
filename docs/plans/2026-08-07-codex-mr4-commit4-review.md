# Codex review record — MR-4 commit 4: "bark: the channel exit"

Commit: bark `2e33b2f1` (jj change `zutpxpkt`) on
`ark8-channels-stage1-client`, child of commit 3 (`32dc8cbe`).
Reviewer: codex, `model_reasoning_effort=max`, read-only sandbox.
Design contract: `2026-08-06-mr4-client-design.md` §8/§9.

**STATUS: NOT CONVERGED — surfaced to Greg after 4 rounds.** All 6 SDK
e2e pass against real captaind at every amendment (open trio + full
unilateral exit + deadline trigger + commitment-reorg recovery); 122
units green; workspace clean. But adversarial review has not reached
PASS, and the finding count plateaued (10 → 7 → 5 → 5) with round 4
including a regression the round-3 fix introduced and a recurrence of
a race believed closed in round 3. The exit's crash-safety / reorg /
LDK-event-concurrency semantics are the hard part of the whole MR and
warrant a design check before more iteration.

## Round history

- **R1 FAIL (10 findings, all adopted).** 5 P1: restart lost tail
  tracking; pending_claims could empty prematurely; sweeps not
  idempotent/channel-scoped; reorg could stay on the losing
  commitment; crash-after-terminal stranded bookkeeping. 5 P2: cancel
  not atomic; zero-valued movements; status omitted the bridge;
  incomplete pending/confirmation predicates; unvalidated close
  capture.
- **R2 FAIL (7 new).** Headline: sweeps had no broadcastable
  fee-anchor path (R1's CPFP-surface fix made them CPFP parents, but
  they carry no P2A); plus the pending-claims/async-persistence race,
  swept_at irreversibility, a drain-crash window, abandoned outputs
  not revisited post-Swept, generic cancel still reachable via
  REST/CLI, record-closed-before-movement window.
- **R3 FAIL (5).** Sweeps still tracked in the manager (R1 fix not
  reconciled with R2's direct-broadcast); the terminal race persisted;
  queue unbounded; winner-replacement didn't unmark/untrack; cancel
  TOCTOU + bridge not untracked. Addressed with a **coherent
  transaction-ownership restructure** (below) rather than more
  per-symptom patches.
- **R4 FAIL (3 High, 2 Medium).** See "Open findings."

## The transaction-ownership model (as of R3's restructure)

- **Bridge + commitment**: the only P2A-anchored parents — tracked in
  the exit transaction manager, broadcast/CPFP'd via
  `exits_needing_cpfp`. The commitment is obtained from the durable
  close-event capture (`bark_ldk_channel_close_event`), never the
  broadcast queue. Both re-tracked from persisted state on restart.
- **Sweeps** (`to_local` / static-payment): self-fee-paying (LDK's
  `spend_spendable_outputs` at the cached rate) — never tracked, never
  a CPFP parent, never in the queue. `pump` builds exactly one attempt
  per output, persists it (m0045 `sweep_txid`/`sweep_tx`), and returns
  it; the `ChannelCommitment` state broadcasts it directly each tick
  and tracks its confirmation hash-bound.
- **`pump`**: advances the best block, (attempted) synchronous
  monitor-event drain, builds/returns sweeps, marks confirmed queue
  rows drained to bound the queue.

## Open findings (R4)

1. **HIGH — the terminal barrier is still not real.** `pump`'s
   `drain_monitor_events` calls `ChannelMonitor::process_pending_events_async`,
   which **early-returns under contention** (LDK's per-monitor
   event-processing guard) when the daemon's background driver already
   holds it, and it cannot surface a `ReplayEvent` persist failure. So
   the intended "descriptor is durable before `pending_claims` reads"
   guarantee does not hold; a maturing output can be absent from both
   the monitor view (LDK handed it to the event) and the descriptor
   table, letting `ChannelSwept`/`ChannelClaimed` finalize without it.
   *This is the R2/R3 race; the R3 sync-drain fix was defeated by the
   LDK concurrency detail.* Candidate fix (needs a design call):
   **remove pump's drain entirely and rely on the background driver
   (the reliable single owner) + the descriptor-inclusion in
   `pending_claims`**, sound under the invariant that the node driver
   runs whenever an exit progresses (a channel exit cannot
   `obtain_commitment` without the node). Alternative: make swept-ness
   a function of the persisted exit state rather than a separate
   `swept_at` column (see finding 3), so the claims view needs no
   race-free descriptor persistence.
**Update (post-Greg): findings 2 and 5 fixed** (bridge kept out of
`exit_txids` and untracked explicitly on cancel; generic cancel refuses
on a channel-entry read error). Commit re-squashed to `1c643a61`; exit
e2e re-verified green. Findings 1, 3, 4 remain for the design
discussion below.

2. **HIGH — regression: the bridge in `exit_txids`.** R3's fix added
   the bridge txid to `txids` so cancellation untracks it, but
   `Start`/`Processing` walk every `exit_txid`, so CPFP now offers the
   bridge before its CSV delta and cancellation becomes non-cancelable
   too early. Fix: keep the bridge OUT of `exit_txids`; untrack it
   explicitly in `cancel_channel_exit` (derive its txid from the
   payload). Mechanical.
3. **HIGH — `swept_at` marked before the state persists.** The sweep
   loop calls `mark_outpoints_swept` before the executor persists the
   `ChannelSwept`/`ChannelCommitment` state carrying that sweep; a
   crash in the window filters the descriptor out as swept while the
   state lacks the sweep bytes — unrecoverable. Fix (design-ish): make
   swept-ness derive from the persisted state's confirmed sweeps, not
   a separate column; or persist the sweep entry in the state BEFORE
   marking the descriptor.
4. **MEDIUM — sweeps fixed at their first feerate, no RBF.** A sweep
   built and persisted at the cached rate is rebroadcast verbatim; a
   later fee-floor rise can stall it. *Judgment call for stage 1:* a
   `to_local` sweep has no competing spender once its CSV matures (it
   is the user's own delayed output), so "stuck at low fee" means
   "confirms slowly," not "funds lost" — arguably an acceptable
   documented residual for stage 1, or add a rebuild-on-rate-rise
   path. Needs Greg's ship-bar call.
5. **MEDIUM — generic cancel swallows a channel-entry read error.**
   `Exit::cancel_exit`'s `if let Ok(Some(entry))` treats a read error
   as "not a channel" and proceeds, which would strand the channel
   record. Fix: a read error refuses. Mechanical.

## R4 affirmatively-clean

The manager ownership split, restart tracking of bridge/commitment,
sweep exclusion from CPFP, winner-replacement cleanup, and start/cancel
lock serialization all check out.

## F1+F3 design consult (codex, `model_reasoning_effort=max`) — UNSOUND

The proposed unified fix (delete pump's drain; terminal/claims gate a
pure function of durable descriptors ∪ self-clearing monitor − confirmed
state sweeps; drop `swept_at`) was assessed **UNSOUND** — the F3 half is
directionally right, but the F1 half rests on an invariant that is
false:

- **"Node exists" ≠ "driver is persisting."** The channel driver is a
  separate daemon task; exit progress only checks the node can be
  constructed, not that `run_driver` is alive or the stop token is
  uncancelled. It can be stopped/panicked, not started in manual-sync
  mode, or absent on a daemon-less wallet, while exit progress
  continues.
- **`ReplayEvent` is not propagated.** LDK's
  `ChainMonitor::process_pending_events_async` catches a persist
  failure, re-queues the event, and only *notifies* — it does not
  surface the error or cancel the node. So a persistent SQLite/disk
  failure leaves: the monitor balance gone (event generated), no
  descriptor row, a still-alive-looking driver replaying forever, and
  exit progress free to reach terminal. There is no sub-second
  durability bound (fallback timers are 10s/30s; a busy handler delays
  arbitrarily), and "recomputed each tick" is not atomic with the
  terminal write (a TOCTOU remains).
- **Monitor balances are not fully self-clearing.** In 0.2.4 they can
  linger until `ANTI_REORG_DELAY` (e.g. `ClaimableAwaitingConfirmations`,
  `MaybeTimeoutClaimableHTLC`) — usually just delaying terminality
  (safe), but the balance can also vanish when the event is *queued*,
  before the descriptor is durable, so the monitor side does not repair
  the race.

**Sound direction codex proposes** (bigger than the original plan):
1. The node exposes driver health + last event-persistence failure;
   **exit progress fails closed** when the driver is stopped/unhealthy.
2. The terminal transition **waits on an event-processing
   flush/acknowledgement watermark** after the relevant best-block
   advance — a notifier or same-daemon relationship is insufficient.
3. Every descriptor is a terminal **obligation**; "abandoned" may
   suppress sweep *attempts* but must not suppress terminal accounting.
4. Subtract only inputs from **canonical, confirmed** sweeps; preserve a
   durable spent-outpoint mapping through terminal (`ChannelClaimed`
   currently keeps sweep txids but not their inputs).
5. Optionally use LDK's historical `get_spendable_outputs` as a recovery
   backstop.

This is a meaningful design addition (a durable event-persistence
watermark + driver-health gating of exit progress), in the money path —
escalated to Greg for a scope/timing decision rather than implemented.

## Refined terminal consult (codex) + the hardening built

Two follow-up consults refined the fix. The first (the F1+F3 unified
plan) was UNSOUND. The second, framed around the STAGE-1 no-HTLC
property, confirmed the no-late-descriptor argument holds for the
reachable stage-1 unilateral path and that the DEEPLY_CONFIRMED +
recovering-terminal recheck is a sufficient barrier — but pinned the
real remaining hole: **driver liveness**. `!stop.is_cancelled()` does
not prove `run_driver` is running (a manual-sync daemon never spawns
it; a freshly loaded node is dormant), so a dormant driver persists no
descriptor and the exit can reach `ChannelClaimed` vacuously.

**Hardening implemented (F1+F3+F4 together):**
- **Fail-closed persistence.** SpendableOutputs and ChannelClose
  persist failures cancel the node's stop token (not just
  `ReplayEvent`), so a durable store failure makes the node unhealthy.
- **Real driver liveness.** A `driver_running` flag is set for the
  whole of `run_driver`; `is_healthy() = driver_running && !cancelled`.
- **Health-gated exit progress**, rechecked at the terminal
  transition (`channel_terminal_ready`), not only at entry — a dormant
  or stopped node makes the exit fail closed (retryable), never
  terminalize.
- **State-derived swept-ness (F3).** Dropped the `swept_at` column
  usage and the mark/unmark machinery; the terminal gate computes
  owed = (opaque self-clearing monitor balances ∪ every persisted
  descriptor outpoint) − (descriptor outpoints spent by a CONFIRMED
  sweep in the exit state). Abandoned descriptors are obligations
  (they suppress sweep attempts, not accounting). Both sides durable,
  no mark-before-persist window.
- **F4 residual documented.** Sweeps are fixed at their first feerate
  (no RBF); a `to_local` sweep has no competing spender, so a stale
  fee only slows confirmation — accepted stage-1 residual.
- Removed pump's contended-no-op synchronous drain; the background
  driver is the sole reliable descriptor persister, and the liveness
  gate makes trusting it sound.

Stage-1 scope note baked into the code: the no-late-descriptor
property holds because stage 1 has no HTLCs; the payments milestone
must revisit the terminal barrier (an explicit event-persistence
watermark) when late-maturing HTLC outputs exist.

## Round 5 — FAIL (one High): the CSV async-persistence race survives

Codex confirmed the hardening's other parts sound: the manual-sync
false-terminal is fixed (health-gated entry + terminal recheck), the
state-derived accounting is correct (all descriptors incl. abandoned
are obligations, confirmed-sweep inputs subtract matching descriptor
outpoints, repeated reads harmless).

The remaining High: `driver_running = true` only proves the processor
future has not RETURNED — not that a given `SpendableOutputs` was
persisted. At CSV the monitor drops the balance the instant it queues
the event; `pump` deliberately does not drain it; so between queue and
the background driver's persist, `pending_claims` is empty,
`channel_terminal_ready` passes, and the exit can reach
`ChannelSwept`/`ChannelClaimed` before the descriptor lands.
`ChannelClaimed` never re-evaluates. **Stage-1's no-HTLC property does
NOT save this — the single `to_local` descriptor is itself generated
asynchronously at CSV.** An event-persistence acknowledgement/watermark
is required; liveness alone is insufficient.

### The fork (money-path design decision — escalated to Greg)

Closing it soundly is a choice, not a mechanical fix:

- **(A) General event-persistence watermark.** Prove the driver
  persisted all monitor events through the relevant height before the
  terminal check trusts emptiness. Clean and general, but LDK's
  `process_events_async` drains the monitor opaquely (no per-pass hook,
  and a second drainer early-returns under `is_processing_pending_events`),
  so this needs the client to OWN monitor-event draining (a driver
  restructure) or a durable per-height processed-watermark. Real work,
  in the money path.

- **(B) Positive-sweep gate (stage-1).** Terminal requires the exit to
  have CONFIRMED-swept the client's commitment output(s), not merely
  observed an empty claim view — an empty-sweeps `ChannelSwept` never
  reaches `ChannelClaimed`. Sound for stage 1 because the commitment
  has exactly one non-dust client output (no HTLCs; the client funds a
  min-size channel and makes no payments, so `to_local` is non-dust).
  Simpler; leans on documented stage-1 assumptions; the payments
  milestone still needs (A).

Recommendation: **(B) for stage 1**, with the assumption documented and
(A) tracked as the payments-milestone prerequisite — unless we'd rather
invest in (A) now to avoid revisiting the exit's terminal barrier.

## Round 6 — (A) implemented: the event-persistence watermark

Greg chose **(A)**. Implemented as a durable *expected-maturity ledger*
that rests on one verified LDK invariant: the `ChannelMonitor` is local
and matures a CSV output — dropping its `ClaimableAwaitingConfirmations`
balance and enqueuing the async `SpendableOutputs` — **only** when we
feed it the crossing block via `best_block_updated`; never on wall-clock.

So `snapshot_expected_maturities` reads every monitor's awaiting balances
and records them durably **immediately before every `best_block_updated`**
(the three advance sites: `pump`, `advance_chain_view`,
`feed_bridge_confirmation`) — capturing each maturing output at least
once while its balance is still reported. `channel_terminal_ready` then
adds to `owed`: `descriptor_count < count_expected_maturities(channel)`.
An output that has matured but whose descriptor the driver has not yet
persisted keeps the exit out of terminal until the descriptor lands
(then, unchanged, until a confirmed sweep spends it). A record-persist
failure propagates (`advance_chain_view`/`feed_bridge_confirmation` now
return `Result`; callers `?`) so a DB failure fails closed.

New table `bark_ldk_expected_maturity` (migration m0046).

Two refinements the first e2e run forced (both real, both fixed):

- **Amount-keyed, not height-keyed.** A commitment reorg re-confirms the
  SAME output at a fresh maturity height. A height key recorded a second
  row for a maturity that never yields its own descriptor →
  `descriptor_count < expected` forever → terminal wedged (the reorg-
  recovery e2e hung). Keyed by `(channel_id, amount_sat)` the reorg re-
  announcement dedups; stage-1's single client output per channel means
  distinct outputs cannot collide on amount.
- **Height gate on the advance.** `pump` snapshots + advances only when
  the fetched tip is strictly above `current_best_block()` — the daemon
  drives `pump` far more often than blocks are mined, and snapshotting on
  every no-op pump is wasted monitor-lock contention.

Documented residuals (stage-1-acceptable, payments-milestone follow-ups):
- Two distinct outputs of identical amount would collapse to one ledger
  row (over-eager). Unreachable in stage 1 (one client output/channel).
- A *different* commitment winning a reorg (adversarial concurrent
  counterparty close) leaves the losing amount recorded → terminal
  wedges. **Liveness only** — funds safe, exit keeps trying, no false
  terminal. Fixable later by clearing the ledger on winner-change /
  commitment-unconfirmation.

Battery at the amended commit (`0e50d364`): 122 bark units + migrations
green; all 6 channel SDK e2e green — the 4 fast ones under nextest, the 2
heavy exit tests (unilateral, reorg-recovery) via `cargo test` to
completion (~6 min each; they exceed nextest's 180s slow-timeout purely
from bdk/bitcoind sync latency over ~250 blocks of CSV maturity + 100-
block finality — a pre-existing environment characteristic that hits the
base commit equally, not a watermark regression). `open_channel_restart_
matrix` is a pre-existing peer-reconnect-vs-readiness flake under heavy
parallel contention (passes alone).

### R6 verdict: FAIL — the watermark misses late `transactions_confirmed`

Codex verified the normal path fixed (snapshot → advance → expected=1
blocks same-tick terminal; persist failures propagate; amount dedup
sound for stage 1; same-output reorgs dedup; different-winner wedge =
conservative liveness only) — but found the invariant itself incomplete,
and the finding checks out against the LDK source:

`transactions_confirmed` creates `MaturingOutput` entries then calls
`block_confirmed`, which drains maturity against **`self.best_block`**
(not the fed height — `debug_assert!(best_block.height >= conf_height)`
explicitly anticipates historical feeds). So a commitment confirmation
fed when the monitor's best block is already past CSV maturity creates
AND drains the descriptor event **inside the same call** — no awaiting
balance ever spans a snapshot boundary, so the ledger records nothing.

Reachable: force-close confirms at h → crash before the exit feeds it →
wallet down > CSV blocks → restart (`advance_chain_view` catches the
monitor to tip; snapshot sees only `ClaimableOnChannelClose`) → the exit
re-feeds the historical confirmation (`feed_confirmation` →
`transactions_confirmed`, un-snapshotted by design — a pre-call snapshot
cannot observe an output first discovered by that call) → instant
maturation → `expected=0`, `pending_claims` momentarily empty → false
terminal with the to_local unswept. Same money consequence as R5.

Codex's SUGGESTION (solicited): stage 1 — require a confirmed client-
output sweep as a positive terminal barrier (the single non-dust client
output makes it concrete); general — serialize confirmation delivery
through the sole driver and await a durable drain acknowledgement
(the payments-milestone restructure).

Decision point (escalated to Greg): the fork re-opens. Buildable-(A)
(the pre-advance ledger) cannot cover late discovery without leaning on
the stage-1 single-output property somewhere; airtight-(A) requires the
driver restructure.

### R6 resolution — Greg chose the stage-1 floor (codex's suggestion)

Options presented: (1) floor of one — `expected = max(ledger, 1)` at the
terminal gate; (2) feed-time `ClaimableOnChannelClose` recording (same
stage-1 reliance, more moving parts); (3) the full general restructure
now. Greg picked **(1)**.

Implemented: `channel_terminal_ready` floors the ledger count at one —
terminal now unconditionally requires at least one persisted client-
output descriptor AND (unchanged subtraction) a confirmed sweep spending
it. The ledger keeps the general >1 pre-advance coverage. The floor's
assumption — exactly one non-dust client output per commitment — is
structurally enforced end to end: the client hardcodes `push_msat=0` at
`create_channel`, the server refuses non-zero push at the open event,
stage 1 refuses payments both sides, and LDK's own dust/reserve
machinery refuses a dust funder balance at open. The general fix (the
node owning confirmation delivery with a durable drain acknowledgement)
stays documented as the payments-milestone prerequisite.

Battery re-run at the floored commit: 122 units; 4 fast e2e green under
nextest (restart_matrix's earlier one-off was parallel-contention flake
— passes alone and in this run); the 2 heavy exit tests re-run to
completion via `cargo test`. Codex round 7 dispatched to verify.

### R7 verdict: PASS — commit 4 converged (7 rounds)

Codex walked the R6 scenario against the floored commit: after restart
the ledger is empty and the historical `transactions_confirmed`
immediately matures the output, but `expected=max(0,1)=1 >
descriptor_count=0` keeps terminal false until the descriptor persists,
then the descriptor stays owed until its confirmed sweep — "crashes at
any intermediate ordering remain fail-closed." No legitimate stage-1
wedge: the client supplies the full amount, push is zero, payments are
refused, and LDK rejects channel values below 1,000 sat, so the client
output exists and is non-dust. `.max(1)` is a no-op whenever the ledger
has entries; dedup/reorg/abandoned behaviors unchanged. "The only new
contract is intentional: the floor assumes stage 1 always has one
client output… future multi-output/zero-output stages would need an
output-specific obligation count" — the documented payments-milestone
follow-up (with the node-owned confirmation-delivery restructure).

No R1–R6 conclusion disturbed. **VERDICT: PASS.**
