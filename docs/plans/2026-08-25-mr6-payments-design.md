# MR-6 — payments (scoping/design note, pre-G1)

The payments slot, re-slotted per the plan's 2026-08-09 addendum: the
descoped client stage 3 (floors, acceptance, scheduling, the intra-ark
payment) PLUS the four accumulated debts, in one MR, debts first. Code
waits for Greg's go after the codex review of this note.

## Ratified scope answers (Greg, 2026-08-25)

1. **One MR, debts first**: the on-chain HTLC resolution machinery and
   the accounting generalizations land BEFORE the commit that enables
   HTLC acceptance — nothing is acceptable before it is resolvable. The
   moment HTLCs can sit in commitments, an HTLC-bearing force-close is
   reachable; the machinery must already be there.
2. **Strictly intra-ark**: A → captaind → B over channel VTXOs on
   captaind's LDK node. No peering with the CLN subsystem, no external
   bolt11 in either direction this MR.
3. **Collect leg: caps + document**: stage 1 ships stock BOLT-3 scripts;
   the server's preimage claim on a client-offered HTLC is an
   adversary-timed fee race until the funding actualizes (the
   2026-08-06 finding). The ratified §3.7 exposure bounds (small
   `max_htlc_value_in_flight`, per-HTLC caps, floor `F`, kill switch)
   are the stage-1 answer; the operator guide states the residual
   plainly. The party-keyed script templates stay stage-2.

## Profile constraints this MR implements (already ratified at G0)

- Floor `F` everywhere (§3.2, quantifiers pinned in 08-channels "The
  force-close deadline"): received HTLC checks the ACTUAL receiving
  channel's `F`; an unbound invoice advertises the MAXIMUM `F` across
  eligible receiving channels; keysend covered per-HTLC (no blanket
  reject); forwarding requires `incoming_cltv − outgoing_cltv ≥ F_in`
  (the INCOMING scope's floor); force-close no later than an unresolved
  HTLC having exactly `F` remaining. Checked arithmetic,
  u16-representability where it feeds LDK config.
- Forwarding ON when channels are enabled, capped, kill-switched
  (§3.7); all stage-1 forwarding is intra-ark by construction.
- Stock LDK, no fork, no type extension (non-goals §1).

## G1 round 1 (codex, 2026-08-25): FAIL — 7 P1, 5 P2, all folded in

The revision below supersedes the original cut. The load-bearing
corrections: a plain output-rekey of the expectation ledger STRANDS
OUTPUTS (pinned LDK balances expose amount/height/source, never an
outpoint) — terminality must come from a serialized feed/drain barrier
("no opaque LDK balances and every persisted descriptor resolved"), not
output counting; no durable payment state exists while the handlers
discard PaymentSent/Failed/Claimed/Forwarded (a crash can lose a sent
preimage and double-pay on retry); a freshly opened channel gives
captaind ZERO outbound liquidity (push_msat hardcoded 0, push rejected
at acceptance) so A→captaind→B needs a B→captaind collect first; stock
LDK enforces the OUTGOING channel's cltv_expiry_delta, not the pinned
incoming-F rule; LDK's CLTV_CLAIM_BUFFER force-closes only
inbound-with-preimage — the outer scheduler is the SOLE early
protection for offered/forwarded timeouts; LDK's HTLC bump handler
batches descriptors and adds wallet inputs — the CPFP stake model
cannot price it per-event; forwarding caps are NEGOTIATED at open —
turning them on cannot retrofit live channels.

Stage-1 postures adopted from the round (flag for ratification at
go/no-go): **MPP disabled** (superseded in round 4 by "single-HTLC
claiming enforced" — `basic_mpp` is undisableable) (custom invoice — per-part
expiry is not visible at PaymentClaimable time; decoded-invoice
conformance tests since LDK pads final CLTV by 3); **one uniform F
profile** for all forwarding channels, persisted per server channel (a
config change must never weaken an existing channel's protection);
**cap-profile capability check fail-closed** — forwarding refuses on any
live channel without the required negotiated caps (pre-release: no such
channel exists, the check is cheap insurance); **HTLC-cause exits are
non-cancellable** (rides the new exit-origin machinery) and the HTLC
rung is level-triggered (`remaining ≤ F`), evaluated before cooperative
rungs in every pre-PONR state.

## Proposed commits (each independently green; round-1 cut superseded)

**c1 — the serialized feed/drain barrier + the new terminal predicate.**
One owner feeds both ChannelManager and monitors, drains both event
queues, waits for every handler write to commit, and only then durably
acknowledges the chain update. Terminality becomes "no opaque LDK
balances AND every persisted descriptor resolved" — no output counting,
no `.max(1)` floor, no rekey-by-outpoint (pinned LDK balances expose
amount/height/source only). Legacy amount rows get dual-read until a
full-drain watermark (they cannot be backfilled). Behavior-neutral for
HTLC-less channels; tests: equal-valued outputs, crashes at each
feed/drain boundary, late delivery, reorgs.

**c2 — the durable client watch feed + counterparty observation.**
The client seam through LDK's `Filter`: a durable registry of
registered outpoints/scripts, raw spender-transaction delivery (a
boolean spent-probe cannot carry a preimage), block hash/cursor +
reorg state, and the register-during-processing rescan. Captaind keeps
its existing full-block feed (no redundant scanner); esplora/bitcoind
paths get batching. `Theirs`/`StaticOutput` sweeps land HERE, gated
with the feed (observation and selection must not disagree —
`obtain_commitment`'s `Ours` assumption falls with it). Tests:
pre-existing counterparty confirmation, reorg Theirs→Ours, same-block
descendants, dust/zero local output.

**c3 — durable HTLC claim jobs + batch-aware fee policy.**
Keyed by LDK `ClaimId`, carrying the actual must-spend prevouts and
descriptor set (the bump handler batches descriptors, splits across
transactions, adds wallet inputs, creates delayed outputs needing a
later sweep). Each batch budgeted from VERIFIED prevout values, wallet
contribution capped, the delayed-output sweep reserved, wallet-input
locks and replacements persisted. The economic ceiling is informed by
the claimed value but the per-event CPFP stake model is NOT reused
verbatim. The client fee estimator learns real targets (it currently
flattens all of them — sound only while nothing is time-sensitive).
Relay path for these claims included.

> **c3 AMENDED 2026-08-26 (Greg): the durable-job design above is
> SUPERSEDED by the trust-LDK design.** Eight implementation-review
> rounds showed the durable ledger re-implementing mempool RBF
> arbitration beside LDK's own claim state machine, sprouting corners
> faster than they closed. Ratified replacement: the ChannelMonitor is
> the durable claim state (it regenerates events with fresh
> descriptors/targets every block until the chain settles the claim);
> mempools arbitrate between attempts; bark keeps NO attempt ledger,
> relay loop or durable selections — each event is built, registered
> with the wallet, broadcast and forgotten. What bark keeps is policy:
> the budget (contribution ≤ claimed; actual fee + sweep reserve ≤
> claimed), the LDK-mirror selection arithmetic, in-memory
> run-lifetime coin locks + our-txid tracking (so bumps re-spend their
> own coins and user payments are never reclassified), and ONE durable
> table — the blocked operator marker. The estimator work stands as
> written. Full rationale, LDK/BDK citations and both review arcs:
> `2026-08-26-codex-mr6-commit3-review.md`. The close-side asymmetry is
> deliberate: commitments/anchors stay on bark's exit CPFP machinery
> (they live inside VTXO exit chains LDK cannot fee-manage).

**c4 — payment state + policy, still refusing payments.**
The durable payment journal keyed by `PaymentId`/idempotency key
(hash, direction, msat amounts/fees, invoice/keysend data,
preimage/secret, state, movement id); event handling commits journal +
ledger effects BEFORE event acknowledgement; `Sent` dominates a late
`Failed`; restart resumes by payment id. Movements gain send/receive
kinds with exact msat metadata.

> **c4 shaping note (2026-08-26, following the c3 trust ratification):
> lean on LDK's payment idempotence before adding bark machinery.**
> `send_payment` with a caller-chosen deterministic `PaymentId` refuses
> duplicates (`DuplicatePayment`), so a crash-replayed send command is
> safe without a separate command outbox — derive the PaymentId, replay
> the command on restart, let LDK dedup. The journal carries what is
> genuinely bark's: msat movements for the balance predicate and F*
> accounting, not a shadow of LDK's payment state machine.
>
> **AMENDED at c4 convergence (2026-08-26, Greg's call, codex r9): no
> automatic re-issue at all.** The replay-on-restart clause above was
> implemented and then deleted — four patches in, codex kept finding
> re-issue lifecycle gaps (the same smell as c3's durable ledger). The
> shipped posture: at node construction (quiescent), a pending send row
> whose id LDK does not track is marked FAILED — the crash cut the send
> before it entered LDK — and the user retries explicitly (failed rows
> are replaceable in place, refused while LDK still tracks the id). A
> wallet must not silently pay an invoice minutes after a restart, and
> everything LDK tracks is LDK's: its events settle the journal. See
> the commit-4 review record for the full convergence. Embedded-node invoice/pay/keysend
library APIs (route hints supply captaind→B, the sender prepends its
own channel — no gossip; keysend via an intra-ark route descriptor).
The uniform-F profile persisted per server channel; the cap-profile
capability check (fail-closed on non-conforming live channels).
Acceptance still OFF.

**c5 — liquidity, receive policy, scheduling: acceptance turns ON.**
The B→captaind collect/top-up operation (captaind invoice +
PaymentClaimable handling — the ratified collect-leg model provides
captaind's outbound liquidity); per-HTLC received-acceptance against
the receiving channel's `F` (keysend included; single-HTLC claiming
enforced — `basic_mpp` remains advertised, multi-part payments are
never claimed and time out); the level-triggered, NON-CANCELLABLE HTLC
deadline rung
(`remaining ≤ F`, before cooperative rungs, in every pre-PONR state;
persists its cause through the exit-origin machinery); forwarding
config from the uniform F profile. Only then does acceptance flip.

**c6 — the payments e2e.**
B→captaind collect; A→captaind→B; below-floor rejection (decoded-invoice
CLTV, keysend vector); forwarding-delta refusal; crash/replay of the
payment journal at every event boundary; the HTLC-bearing force-close
(offered and received) to terminal state; the backwards preimage claim;
batched-claim fees; legacy-cap refusal vector; MR-4's deferred
composition e2e (open → pay → downgrade → re-upgrade); and the MR-5
residuals payments unlock — a REAL sub-dust close side and
balance-bearing closes.

## G1 round 2 (codex, 2026-08-25): FAIL — 5 P1 remaining, folded in below

1. **The floor's stock-LDK seam is CLAIM-time, not commitment-time.** An
   HTLC is irrevocably committed before any final-hop hook runs, so no
   pre-commitment refusal exists on stock LDK. The pinned requirement's
   INTENT — never hold a CLAIMABLE obligation with less than `F`
   remaining — is enforced at the only sanctioned seam: for invoices,
   LDK's own `min_final_cltv_expiry_delta` check fails a short HTLC
   backwards before `PaymentClaimable`; for keysend (and as
   defense-in-depth for invoices), the `PaymentClaimable` handler checks
   the receiving channel's floor and FAILS BACKWARDS below it — the
   preimage never enters `claim_funds`, so no monitor holds it, no
   forced close triggers, and the sender reclaims at timeout. A failed
   HTLC transiently rides the commitment; that exposure carries no
   claim obligation and is bounded by the per-HTLC caps. The spec's
   "received-HTLC acceptance" maps to CLAIMING.
2. **Uniform `F*` is immutable while any channel exists.** LDK honors a
   channel's previous config for a grace window after updates, so even
   tighten-only rollouts leave a weaker-delta window. Stage 1 kills the
   class: `F*` is fixed at the first channel's creation; a differing
   config refuses startup while channels exist. Channel admission
   requires `F_c ≤ F*`.
3. **The barrier awaits MANAGER durability.** Handler completion does
   not make the async-written manager blob durable (driver.rs states
   it); the drain acknowledgement waits for manager persistence too.
   Crash cuts: manager-durable/cursor-old replays events idempotently;
   cursor-durable/manager-old is made unrepresentable (the cursor write
   waits).
4. **The journal is a full exactly-once protocol**: a durable command
   OUTBOX commits the `PaymentId`-keyed intent BEFORE `send_payment` is
   invoked or an invoice is returned (a send-accepted/journal-absent
   crash is unrepresentable); payment-hash single-use is enforced
   separately (LDK permits duplicate `PaymentClaimable` per hash — the
   later one is failed); `PaymentForwarded` (no `PaymentId`, duplicates
   possible) is recorded at-least-once as TELEMETRY in stage 1 — its
   fee ledger effects are reconciliation-derived from channel balances,
   never event-summed.
5. **The cap capability is a concrete predicate**: per channel,
   negotiated `max_accepted_htlcs ≤ N_cap`,
   `max_htlc_value_in_flight_msat ≤ min(V_cap, p% × capacity)` in BOTH
   directions (ours by handshake config, theirs verified at accept),
   dust limit within profile bounds; outbound aggregate exposure is
   bounded by `per-HTLC cap × max_accepted_htlcs` (LDK offers no
   outbound-total knob). A live channel is conforming iff every
   inequality holds on its NEGOTIATED values; otherwise forwarding
   refuses.

P2 folds: persisted descriptors and claim jobs carry block provenance
with reversible active/orphaned/resolved states (a `Theirs→Ours` reorg
must not wedge terminality); the collect-leg e2e matrix comes from the
finding itself — all four commitment-owner × HTLC-direction
combinations, direct vs second-stage resolution, late bridge, trimming
on either commitment, restart, reorg; the surface note's "no
forwarding" line is corrected (forwarding is ON, capped — §3.7), and
the operator-guide residual statement is REQUIRED BEFORE acceptance
flips (it moves from MR-7 into this MR's c5).

## G1 round 3 (codex, 2026-08-25): FAIL — premises VERIFIED, spec gaps folded

Round 3 verified every load-bearing stock-LDK premise with line-level
citations: the claim seam (keysend onion decoding stores the preimage in
routing state only; monitors receive it only on claiming; failing
backwards never supplies it; monitor force-close requires a stored
preimage), the invoice `min_final` rejection ordering (before
claimability), the config grace fallback, the async manager write, and
`PaymentForwarded`'s identifier-less duplicates. Remaining gaps, folded:

1. **`must_drive_onchain` (P1)**: `fail_htlc_backwards` removes
   claimability but the HTLC lingers in both commitments until the
   removal handshake — the non-cancellable HTLC rung must not exit on
   it. The rung's predicate is persisted and EXCLUDES final-hop
   failed/unclaimed HTLCs and incomplete MPP parts; it is evaluated
   only after event draining. Test: a peer withholding the removal
   handshake (the rung must hold; LDK's own timeout machinery fails the
   channel if the peer never completes).
2. **The exposure profile completed (P1)**: `H_cap = V_cap` explicitly
   (LDK's knob is the aggregate in-flight limit; one HTLC may consume
   it all — the per-HTLC and aggregate bounds coincide by definition);
   dust exposure bounded by LDK's `max_dust_htlc_exposure` (the
   negotiated `dust_limit_satoshis` is only the trimming threshold);
   the kill switch is a named config (`channel_forwarding_enabled`)
   read per-forward, with a c6 test that flipping it stops forwards
   without touching acceptance.
3. **MPP cannot be disabled at the receiver (P2)**: pinned LDK always
   advertises `basic_mpp` and aggregates parts before
   `PaymentClaimable` (keysend MPP included). Posture restated: the
   claim handler requires EXACTLY ONE `receiving_channel_ids` entry;
   multi-part payments are never claimed and time out at LDK's
   three-tick expiry (sender refunded). Hostile invoice-MPP and
   keysend-MPP vectors in c6.
4. **Capability evidence is persisted at `OpenChannelRequest` time
   (P2)**, before `accept_inbound_channel` — the event is not
   re-derivable and `ChannelDetails` does not expose negotiated
   count/dust later. Unknown evidence = nonconforming. Pre-MR channels
   (no immutable-F-at-open proof) are refused for forwarding —
   close-and-reopen, never retrofit via `update_channel_config`.
5. **Orphaned-descriptor terminality (P2)**: `orphaned` DISCHARGES the
   current-chain obligation (terminality counts only `active`
   descriptors unresolved); a reorg that reactivates an orphaned
   descriptor reopens the exit's non-terminal state — terminal states
   already re-validate block hashes each tick (the MR-4/5 recovering-
   terminal pattern), so reactivation rides that machinery.
6. **The operator-guide residual statement is IN c5** (the "explicitly
   out" line below is corrected — only the FULL operator guide is
   MR-7).
7. **c6 gains the command-side crash cuts** (per the c4 amendment:
   journal-before-send, crash-cut sends failing at construction,
   failed-row replacement):
   monitor-durable/manager-old, invoice-durable/before-return, plus
   duplicate-payment-hash vectors; and the floor boundary is tested at
   exactly `F` (accepted) vs `F − 1` (failed), the defense-in-depth
   check reconstructing the single HTLC's expiry as
   `claim_deadline + HTLC_FAIL_BACK_BUFFER` with checked arithmetic,
   failing closed on `None`.

## G1 round 4 (codex, 2026-08-25): FAIL — one P1, folded

**Capability gating covers ACCEPTANCE, not just forwarding**: a legacy
channel's negotiated inbound caps are live for direct and
incomplete-MPP HTLCs before any claim-time hook, and cannot be
retrofitted. Acceptance flips ONLY when zero live unknown/nonconforming
RECEIVING channels exist — close-and-reopen ENFORCED at c5 startup
(refusal to enable, not advice), with direct-HTLC and incomplete-MPP
legacy-cap vectors in c6. Wording corrected throughout: not "MPP
disabled" but "single-HTLC claiming enforced; `basic_mpp` remains
advertised".

## G1 round 5 (codex, 2026-08-25): FAIL — one P2, folded

Complete vs incomplete MPP distinguished: LDK's three-tick timeout
covers only INCOMPLETE part sets — a complete multi-part payment
surfaces `PaymentClaimable` and is retained. c5's handler therefore
FAILS a complete multipart claimable BACKWARDS explicitly (the
one-entry length check is the condition, the fail is the action);
internal sends pin `max_path_count = 1` (advertised `basic_mpp`
otherwise permits up to ten paths); c6 distinguishes complete-MPP
immediate fail-backwards from incomplete-MPP tick expiry.

## Design decisions to ratify (Greg) — revised after G1 round 1

1. **One scheduler** owns "the chain must decide by height H" for both
   expiry rungs and HTLC deadlines — with the corrections: the HTLC
   rung is level-triggered (`≤ F`), runs before cooperative rungs in
   every pre-PONR state, and its exit is NON-CANCELLABLE (LDK's
   `CLTV_CLAIM_BUFFER` covers only inbound-with-preimage; the outer
   scheduler is the sole protection for offered/forwarded timeouts).
2. **The client watch seam is LDK's `Filter`** with a durable registry
   and raw-transaction delivery; the server keeps its full-block feed.
3. **HTLC claim fees**: durable `ClaimId`-keyed claim jobs with
   batch-level budgets from verified prevouts — informed by, not
   identical to, the CPFP stake model.
4. **No separate payments flag**, PROVIDED the fail-closed cap/F
   capability check precedes enablement.
5. **Single-HTLC claiming enforced in stage 1** (`basic_mpp` remains
   advertised — undisableable on pinned LDK; multi-part payments are
   never claimed and time out; per-part expiry is invisible at claim
   time).
6. **One uniform F profile (`F*`), immutable while any channel exists**;
   admission requires `F_c ≤ F*`; a differing config refuses startup.
7. **Floor enforcement at the claim seam**: invoices via
   `min_final_cltv_expiry_delta`, everything via a fail-backwards check
   at `PaymentClaimable` — below-floor HTLCs are never claimed, so no
   claim obligation ever exists under `F`.
8. **`PaymentForwarded` is stage-1 telemetry**: at-least-once recorded,
   fee accounting reconciliation-derived, never event-summed.

## Conformance mapping (six commits)

Floor F quantifiers (I-4/5/6): the uniform-`F*` persistence and
invoice/receive plumbing → c4; claim-time enforcement + scheduling →
c5; vectors → c6. The collect-leg residual statement (operator guide)
→ c5, BEFORE acceptance flips. The MR-4 composition claim
(open→pay→downgrade→re-upgrade) → c6. RG/IB unchanged.

## Explicitly out (this MR)

External LN bridging (channel ↔ CLN/outside); the Ark channel type +
party-keyed templates (stage 2); teleport/refresh; the forwarding-race
ordering mechanism (§3.6); MR-7 surface work (REST/CLI and the FULL
operator guide — the collect-leg residual STATEMENT itself lands in
this MR's c5, before acceptance flips).
