# MR-6 commit 5 — payments enablement: codex review record

Commit: `07af4febc` "channel payments enabled: the claim seam, the HTLC
deadline rung, the collect leg" (bark-stage1,
ark8-channels-stage1-close). Reviewer: codex (read-only, pinned LDK
0.2.4 citations required). Verdict: **PASS at round 15** — the longest
convergence of the MR (c4 took 13), fitting for the commit that flips
acceptance ON. Battery green on the final tree and every intermediate
amend (632+ units / bark-channels / SDK channels 19 / server channel
22); the round-2 misfire was caught by the e2e battery itself.

## What the commit does

- **The claim seam (client)**: claiming is where a preimage becomes an
  obligation the chain may have to decide; everything below a refusal
  fails backwards. Order: unknown even custom TLVs (plain claim_funds
  would auto-fail them silently, wedging any binding) → purpose
  (bolt11/keysend only) → exactly one receiving channel (complete-MPP
  explicit fail-backwards) → claim-admissible record (ready|registered)
  → pinned floor F with the expiry reconstructed as
  claim_deadline + HTLC_FAIL_BACK_BUFFER, measured against a FRESH
  chain view → single-claim journal discipline → the CLAIM-BINDING CAS
  (pending→claiming bound to the claimable's LDK payment id) as the
  last step before claim_funds.
- **The freshness lease**: the claim seam refuses to measure an HTLC's
  remaining lifetime against a stale view. The lease is granted ONLY by
  a completed, QUIESCENT watch pass (a bare height proves nothing about
  spends a scan never delivered); DATED from the pass's tip fetch (a
  slow pass grants an already-stale lease); BOUND to a watch-registry
  generation (every Filter registration bumps it on both sides of its
  store write — channel A's pass must never vouch for freshly opened
  channel B); stamped by BOTH clocks (monotonic against wall jumps,
  wall against suspend, future stamps stale). A stale lease FAILS
  BACKWARDS — never replays — so it cannot hold the driver's drain
  barrier hostage to the sync that would renew it.
- **The HTLC deadline rung**: level-triggered (remaining ≤ F), before
  cooperative rungs, non-cancelable via exit_origin='htlc_deadline'.
  Obligations from LDK's own live HTLC surface
  (ChannelDetails::pending_{in,out}bound_htlcs — monitor balances fold
  live inbound claimed HTLCs into the closing total): every live
  outbound except peer-fulfilled; inbound with fulfill-in-flight or
  Committed-but-claimed (journal `received` — the wedged-peer case);
  dust excluded (the chain cannot decide it). Fires on
  ready|registered|negotiating|outcome_ready|fallback_only; escalates a
  RUNNING ready-origin exit in place; cancellation independently
  re-checks live obligations.
- **The peer-close level trigger**: an observed NON-cooperative close
  starts the exit NOW (`peer_close`, non-cancelable) — LDK delists a
  force-closed channel instantly, and waiting for the VTXO hard line
  could hand a live HTLC's timeout race to the peer. Cooperativeness is
  a durable column beside the close marker (m0053) — reasons LIE across
  crashes: a completed cooperative close whose event was never handled
  resurfaces as ProcessingError, so a non-cooperative reason first
  PROBES for the captured closing candidate (typed: decode/storage
  errors replay, only proven absence classifies; balance-less
  completions fall back loudly) — both sides, the server inside its
  locked close transaction. The close marker write atomically escalates
  a running ready-origin exit; cancellation re-checks every close fact
  inside its own transaction.
- **STRANDED (posture correction over c4)**: a pending send untracked
  by LDK at restart is never-issued OR settled-and-revoked-away —
  indistinguishable — so it strands: not failed (a retry could
  double-pay), not replaceable, no movement booked, surfaced via the
  new read-only journal listing. An authoritative terminal event still
  resolves it (PaymentFailed → failed/replaceable; sent dominates).
- **The collect leg (server)**: ChannelCollectInvoice (admin service,
  F* as min_final) + a journaled single-claim discipline (V61: hash
  pending → claiming-bound-to-payment-id → received) + the same claim
  seam checks (own bolt11 invoices only, one part, pinned floor +
  conformance, fresh bitcoind height per claim, TLV screen). WD-16
  stands: the on-chain preimage race exists only after the client
  actualizes, where server claim funding answers in the budgeted slack.
- **Forwarding + the acceptance gate**: accept_forwards_to_priv_channels
  = channels.channel_forwarding_enabled (restart-applied kill switch),
  behind the startup gate: every accepting row (cosigned|registered, no
  close outcome) must prove channel_caps_conforming (absent evidence =
  nonconforming); offenders QUARANTINE the run — forwarding off, claims
  refused on the offender — rather than refusing startup (which would
  deadlock the close-and-reopen remedy). Manual-sync mode refuses to
  start over a channel-configured wallet. The daemon interval clamps
  under the freshness bound. Operator residual statement:
  doc/channel-payments.md.
- **Registered = claim-capable and exit-capable**: LDK can be ready
  before the record flips (the flip is Ark-connectivity-gated), so
  `registered` joins every rung/exit path, and the open action
  RECONCILES with a forced exit that takes its record (local
  bookkeeping lands; the ready flip yields with the movement finished).
- **Invoice eligibility matches the seam**: floor over claim-admissible
  records only; creation requires an LDK-ready channel among those
  records' own ids.

## Round history (15 rounds, ~30 findings)

r1 (7): ChannelDetails-based rung (monitor balances blind to live
inbound); collect journal; exiting-escalation + cancel recheck; fresh
tip at the seams; gate NULL-skip; closed-row startup wedge → quarantine;
keysend insert error typing. r2 (1): peer-close level trigger — the
first draft's delisted-without-marker heuristic misfired on a
cooperative close mid-capture (caught by the SDK e2e battery) and was
replaced by the durable cooperativeness column. r3 (3): the
cooperative-completion probe (reasons lie across crashes); STRANDED
replaces fail-at-construction (double-pay via settled-and-forgotten
sends); in-transaction cancel checks + marker-write escalation. r4 (5):
stranded vs m0052's CHECK (caught only by review — the migration was
extended in place, pre-release posture); cancel vs completed cooperative
close; typed client probe; the server-side probe; the journal listing
surface. r5 (1): client probe txid integrity. r6 (1): PaymentFailed
resolves stranded. r7 (1): freshness bound on the observed tip. r8 (3):
monotonic clock; pre-barrier stamp (superseded in r9); interval clamp.
r9 (3): completed-pass lease + STALE-FAILS-BACKWARDS (dissolving the
barrier deadlock structurally); dual clocks (Linux suspend); manual-sync
refusal. r10 (3): quiescent-only grant; the generation binding;
Registered exit-capability. r11 (3): double-sided generation bumps; the
open-action reconcile; the claim-binding CAS (LDK's claim-completion gap
permits a second same-hash claimable). r12 (2): the even-TLV screen;
invoice eligibility. r13 (1): per-record readiness. r14 (1): lease dated
from the tip fetch. r15: PASS.

## Key decisions for Greg's review

1. **Stale chain view ⇒ fail backwards, never replay** — the safe
   direction (sender refunds and retries) and the structural fix for
   the lease-vs-barrier deadlock class.
2. **The freshness lease** (quiescent pass, tip-fetch dating,
   generation binding, dual clocks) — the claim seam's chain view is a
   proof-carrying artifact, not a cached height.
3. **STRANDED** — c4's fail-at-construction corrected: ambiguous
   crash-cut sends neither fail nor auto-retry; explicit user
   resolution via the journal listing; authoritative events still
   resolve.
4. **Quarantine over startup refusal** for nonconforming channels
   (matches the ratified "refusal to enable, not advice" and keeps the
   close-and-reopen remedy reachable).
5. **peer_close as a forced exit origin** + the cooperative-completion
   probe (closing candidates as the authoritative witness against lying
   closure reasons).
6. **The claim-binding CAS** on both journals (pending→claiming keyed
   by LDK payment id) — LDK's own dedup ends at claim completion; a
   paying-twice payer must not be claimed twice.
7. **Even-TLV screen before any binding** (plain claim_funds silently
   auto-fails such payments).
8. **Registered is a first-class protected state**; the open action
   reconciles with forced exits instead of wedging.
9. Manual-sync + channels refuses; the daemon interval clamps under
   the freshness bound.

## Carried to c6 (e2e)

Everything this record pinned statically wants vectors: the floor
boundary (F vs F−1), the claim-binding race, hostile MPP and keysend,
TLV refusal, the stranded lifecycle (crash cuts), the peer-close
trigger and the cooperative-probe reclassification, the HTLC deadline
rung firing (wedged-peer Committed-claimed case), collect end-to-end +
repeat-payment refusal, quarantine behavior, plus the c6 list already
in the design note (A→captaind→B, crash/replay at every boundary,
HTLC-bearing force-closes, composition, MR-5 residuals).
