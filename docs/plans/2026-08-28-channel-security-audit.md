# ARK #8 channels stage-1 — Bitcoin-transaction-level security audit

Status: DRAFT. Triggered by the discovery that per-state JUSTICE
(punishing a broadcast revoked commitment) is unwired in stage-1 — a
regression from the reference impl `ark-channels-bridge-2026-06-18`. That
gap "calls into question everything," so this audit re-derives the whole
security story from first principles: **who can publish which Bitcoin
transaction, when, and what machinery answers it** — with each cell
graded WIRED? / TESTED? and diffed against the reference.

Method: FIVE parallel code audits — four on the transaction-family
*edges* (exit-chain+broadcaster / commitment justice / HTLC force-close
/ coop-close+downgrade+forfeit+expiry) and one on the state-machine
*states* (partial execution, abandonment/liveness, races, timelock
safety over the presigned tx DAG). Synthesized here, every gap verified
against code, then an adversarial design-stage codex pass. Combined
findings at the end.

**Two lenses applied to every gap:**
- **Altitude/normativity (spec):** the spec must state the OBLIGATION
  (a BOLT-style normative MUST), not the mechanism. Where it only
  *describes* a behavior as "unchanged / standard / as in BOLT-N," that
  is a description with no conformance cell and no test — a design
  blind spot. Sweep every such phrase around money movement.
- **Grade each gap:** `SPEC` (a normative MUST is missing, so no matrix
  cell / no test), `IMPL` (a regression — the reference wired it, the
  rewrite dropped it), or `BOTH`. Plus `WIRED?` and `TESTED?`.

The audit is organized so the state machine is first-class: the tx
families are the edges; the reachable on-chain states (which txs
confirmed, which timelocks elapsed) are the nodes; a complete audit
walks every node and asks whether every party's money is safe when the
counterparty is adversarial OR simply ABSENT.

## The parties and their capabilities

- **Client** (the wallet): holds the fully-signed bridge + genesis exit
  chain; the only party that can put the funding on-chain in the normal
  case (its unilateral exit: genesis → bridge → commitment). Holds its
  own commitments and the per-commitment secrets to punish the server.
- **Server** (captaind): the channel counterparty (LDK node). MAY retain
  the bridge; if it does, it can force-close by broadcasting
  bridge + commitment (spec `08-channels.md:288`). Holds the
  per-commitment secrets to punish the client. Also has expiry recourse
  (sweep the channel VTXO output) that needs neither bridge nor
  commitment.
- **Both** run LDK monitors that, fed the chain, build justice / HTLC
  claim transactions and hand them to their `BroadcasterInterface`.

## The transaction families (the spine)

For each: WHO can publish, WHEN, and the row is filled from the audits.

| # | Transaction | Who can publish | When |
|---|---|---|---|
| T1 | Genesis exit-chain (VTXO tree) txs | client | its unilateral exit |
| T2 | Bridge tx (funds the virtual funding outpoint) | client always; server IF it retained the bridge | exit / force-close |
| T3 | Holder commitment (own force-close) | either party (its own) | force-close |
| T4 | Counterparty commitment — LATEST | either party | force-close |
| T5 | Counterparty commitment — REVOKED (cheat) | either party (malicious) | force-close at an old state |
| T6 | HTLC-success / HTLC-timeout (second stage) | the party owning the branch (party-keyed CSV) | after the commitment confirms |
| T7 | Justice / penalty (revocation branch) | the wronged party | on seeing T5 (or T6 on a revoked commitment) |
| T8 | Cooperative closing tx | either party (spends the bridge) | cooperative close |
| T9 | Downgrade split RESPONSE tx (conflict-winning checkpoint) | client | PONR / chain-overrules-tombstone |
| T10 | Scope-transition FORFEIT of the old-scope VTXO | server | upgrade/downgrade teleport |
| T11 | Expiry sweep of the channel VTXO output | server | after channel expiry |
| T12 | Balance / to_local / to_remote sweeps | the owning party | after its commitment confirms + CSV |

## Refined threat model (from the audit)

Two facts change who can do what:
- **Only the client can broadcast the bridge (T2).** The server stores
  no bridge witness (verified: `git grep bridge_signature server/` is
  empty). So the funding reaches the chain ONLY via a client exit.
- **The server cannot force-close in production (see GAP-3).** Its
  `ChannelClose` CPFP is deliberately never relayed (WD-16) and the only
  force-close entry is a mainnet-refused dev seam.

Therefore **the server's commitment (T3/T4/T5) reaches the chain only by
the server racing its own retained commitment onto the client's bridge
during a client exit** (`obtain_commitment` adopts the first confirmed
foreign spend as `Theirs`, mod.rs:2327). That is an adversarial act. It
splits the findings into two families:

- **Family 1 — the client cannot defend against a racing/cheating
  server** (user funds at risk; the security-critical family).
- **Family 2 — the server cannot recover its own funds in force-close /
  client-abandonment** (server's own funds; operationally serious, not a
  user-custody violation).

## Findings — the gaps (5 audits, cross-verified, key claims re-checked)

> NOTE: this section is the ORIGINAL five-agent synthesis with its
> per-root-cause file:line detail. For authoritative SEVERITY and STATUS
> (post-codex), the **"Combined findings — FINAL"** section below
> supersedes it: GAP 8 is merged into GAP 1; GAP 2 and GAP 3 are HIGH;
> GAP 5 is LOW; GAP 9, R1, R2, R3, C1 are added there.

All five are REGRESSIONS from the reference `ark-channels-bridge-2026-06-18`,
all in the FORCE-CLOSE / on-chain-recovery machinery. Three root causes.

### Root cause A — client `QueueBroadcaster` captures, never relays; no exit-driver drain
`bark/src/channels/broadcaster.rs` writes every LDK broadcast to
`bark_ldk_broadcast_queue` and nothing reads it back (the table's own
test: "Nothing reads the capture on the recovery path"). LDK self-funds
(and hence hands to the plain broadcaster) everything that is NOT the
node's own commitment/HTLC — verified in vendored LDK
(`package.rs:1579-1587`: `requires_external_funding()` true only for
`HolderFundingOutput`/`HolderHTLCOutput`). So all of these are dropped:

- **GAP 1 [HIGH, user funds] — client JUSTICE unwired.** A server that
  races a REVOKED commitment onto the client's bridge is not punished;
  the client recovers only its old-state `to_remote` and loses every
  payment moved to it after that state. `ChannelSweepKind::Justice`
  (exit/models/states.rs:162) is never constructed. Spec §1300 gives
  justice zero response margin by design, so unwired = TOTAL, not slow.
  Comments in `mod.rs:2519-2522` / the broadcaster module doc falsely
  claim justice is "obtained from the durable close capture" — MASKING
  the gap.
- **GAP 4 [HIGH, user funds] — client cannot claim an HTLC off the
  server's commitment.** `CounterpartyOffered/ReceivedHTLCOutput` claims
  are self-funded → QueueBroadcaster → dropped. When the server's
  commitment wins the bridge race with an in-flight HTLC the client must
  claim (preimage or timeout), the client cannot get it on-chain at all.
  `claims.rs` handles only the holder's OWN 2nd-stage.
- **GAP 8 [HIGH, user funds] — client HTLC-output justice** (cheater's
  2nd-stage on a revoked commitment) — same drop; no stage-1 equivalent
  of the reference's `feed_watched_output_spends`.

**Fix:** port the reference's `exit_driver.rs` drain-and-relay of LDK's
in-memory broadcaster (tagging `Justice`), and narrow QueueBroadcaster's
no-relay rule to close-candidates only — exactly the shape the c3 server
`CaptureBroadcaster` fix already has (relay non-candidates).

### Root cause B — server has no `Event::SpendableOutputs` handler
`server/src/channels/event.rs:989-995` catch-all; no `OutputSweeper`.

- **GAP 2 [MEDIUM, server funds] — server never sweeps its own
  to_local / to_remote / successful-HTLC-claim outputs.** Dropped at
  maturity on any force-close. LDK forgets the descriptor at emission;
  the server never persists it. Contradicts `doc/channel-payments.md`
  ("rides the server's own HTLC/BALANCE claim machinery" — the balance
  half does not exist). Key-recoverable in principle, no stage-1 path.
  **This bites even in the NORMAL client exit:** the server's balance is
  a `to_remote` `StaticPaymentOutput` on the client's commitment that it
  only needs to sweep — and can't. Scoped to a server that carries
  balance/HTLCs (WD-16: a zero-balance server MAY leave it open).

### Root cause C — server has no production force-close + withholds the real bridge from LDK
- **GAP 3 [MEDIUM, server funds] — the WD-16 post-actualization
  force-close is not built in production.** Distinct from the stage-2
  retained-bridge early force-close (submitting the bridge, gated on the
  CSV Ark channel type — correctly absent). GAP-3 is the STANDARD BOLT
  force-close the spec makes a stage-1 MUST once the client has
  actualized the funding: WD-16 / §1065 — "MUST eventually force-close
  to recoup them ... whenever the server carries balance or unresolved
  HTLCs." Not implemented: the server withholds the bridge txid from the
  LDK feed permanently ("terminal included", mod.rs:88-97,291-315) so it
  never notices the client's on-chain bridge; the parent-exit watcher
  no-ops when the spender IS the bridge (watch.rs:109); every
  force-close call site is a pre-registration refusal / defunct-reap /
  the dev seam; and the production admin RPC `channel_force_close` →
  `dev_force_close` → `ensure!(dev_seams)` → REFUSED ON MAINNET
  (admin.rs:155, mod.rs:604 + startup guard). REGRESSION: the reference
  had a real `force_close_channel` RPC (no dev gate). Bites the
  client-actualizes-then-abandons case; GAP-2 then also blocks the
  follow-up sweep. Scoped to a balance-carrying server (zero-balance MAY
  leave it open). [Severity MEDIUM not HIGH: server's OWN funds, and a
  zero-balance server is unaffected — but it is a stated stage-1 MUST.]

### Test / coverage
- **GAP 5 [MEDIUM] — reorg coverage narrow.** Only commitment-reorg is
  tested; bridge/genesis/coop-close/sweep/winner-flip reorgs are
  code-only. Latent.
- **The reference has a 4-quadrant e2e matrix stage-1 entirely lacks:**
  `barkd_{server,client}_punishes_dishonest_{client,server}_close`
  (+`_with_pending_htlc`). Stage-1 covers only the honest, zero-HTLC
  quadrants (`theirs_commitment_sweeps` is explicitly HTLC-free). The
  reference git history shows this exact bug was hit and fixed there
  ("mark client justice test as ignored pending broadcast fix").
- **Server-side revoked column (client cheats): plausibly wired
  (CaptureBroadcaster relays non-candidates), UNTESTED.**

## NOT gaps (reconciled — do not raise as findings)

- **Stock LDK / no party-keyed HTLC CSV** (§1288-1305): a RATIFIED
  core-only decision (docs 2026-08-25, 2026-08-06), compensated by the
  exposure caps (`MAX_ACCEPTED_HTLCS=6`, in-flight caps). Consequence is
  a fee-race bounded by the caps, not a deterministic window.
- **Forfeit absence** (spec §2148-2162): the forfeit is a REFRESH
  mechanism; stage-1 is core-only (no in-place refresh), defensively
  enforced (round admission rejects a ChannelFunding VTXO). Downgrade
  (nSequence=0 split) and upgrade (parent-exit watch) carry the
  revoked-scope safety property. Scope gap, not a safety hole.
- **Cooperative close, downgrade/PONR/FALLBACK_WON, pre-bridge expiry
  sweep:** wired + real chain-race/reorg tested. No gaps.
- **Detection:** sound both sides (client `sync_channel_watch` scans all
  watched outputs; server full-block feed) — except the server's own
  withheld bridge (GAP-3).
- **Exit ordering (tree→bridge→commitment):** sound — BIP-68
  `nSequence=exit_delta` + client state-machine discipline.
- **Server submitting the bridge (retained-bridge EARLY force-close of
  an idle channel):** STAGE-2, gated on the CSV Ark channel protection;
  implemented in neither stage-1 nor the reference. Not a gap. (Distinct
  from GAP-3, the post-actualization standard force-close.)

## Severity summary

- **Family 1 (user funds, security-critical): GAP 1, GAP 4, GAP 8** —
  the client cannot defend against a server that races a
  revoked/HTLC-bearing commitment onto the client's bridge during a
  client exit. One root cause (client broadcaster). REGRESSION.
- **Family 2 (server's own funds, stated stage-1 MUST): GAP 2 (+GAP 3
  for abandonment)** — a balance-carrying server cannot recover its
  collect-leg funds on any client exit. REGRESSION.
- **Coverage: GAP 5** (reorg tests narrow) + the entire 4-quadrant
  dishonest-close test matrix the reference has and stage-1 lacks.

## Combined findings (audit + codex adversarial pass — FINAL)

Codex (design-stage adversarial, xhigh) confirmed the central threat model
and the three root causes, added one state-machine gap the five agents
missed, merged GAP 8, regraded three, and reframed two "not-gaps" as
bounded residual risks. Verified the load-bearing changes myself (GAP 9
reselect; the LDK `requires_external_funding` scope). Final list:

### Money-safety gaps (must decide before ship)

- **GAP 1 [HIGH, user funds] — client JUSTICE unwired** (root cause A).
  Now ABSORBS the old GAP 8: HTLC-output justice (cheater's 2nd-stage)
  is detected fine by the real Filter watch, but its penalty rides the
  same dead `QueueBroadcaster` — so it is GAP 1, not a separate
  detection gap (codex correction).
- **GAP 4 [HIGH, user funds] — client can't claim an HTLC off the
  server's commitment** (root cause A). Gated on the adversarial server
  racing a retained commitment onto the client's bridge (confirmed: no
  honest production path lands a server commitment on-chain).
- **GAP 2 [HIGH ↑ from MED, server funds] — server never sweeps its own
  to_local/to_remote** (root cause B). DETERMINISTIC on an ordinary
  client exit (its `to_remote` `StaticPaymentOutput` hits the
  `SpendableOutputs` catch-all), no malice. Regraded up: material funds,
  service-custody.
- **GAP 3 [HIGH ↑ from MED, server funds] — no production
  post-actualization force-close** (root cause C). Material funds
  indefinitely locked; the admin RPC is a mainnet-refused dev seam.
- **GAP 9 [MED alone / HIGH combined] NEW — cooperative-close state
  never adopts a foreign commitment winner.** VERIFIED:
  `ExitChannelCooperativeClosingState` (exit/progress/states.rs:364-460)
  tracks only its own `closing_txid`; it never calls `obtain_commitment`
  / reselects, unlike `ExitChannelCommitmentState` (:613-685). If the
  server races a retained commitment that confirms first, the client
  rebroadcasts a doomed close forever and never resolves — and if that
  commitment is revoked, GAP 1 also drops the justice. Fix rides root
  cause A's area (client exit machinery).

### Residual protocol risks (bounded/ratified — must be EXPLICITLY owned, not assumed safe)

- **R1 [HIGH residual, bounded] — no-CSV HTLC fee-race.** Codex regrade
  of the "stock-LDK = safe" not-gap: with no relative-locktime on the
  claim branches, a preimage-holder's direct success claim that does not
  confirm before the HTLC's CLTV can be beaten by the counterparty's
  higher-fee timeout claim → HTLC lost. Bounded by the exposure caps
  (6 HTLCs, in-flight cap) PER CHANNEL, multipliable across channels; NOT
  a consensus-enforced window. The ratified core-only decision must own
  this as a bounded residual, not treat capped exposure as no-risk.
- **R2 [HIGH residual] — expiry-boundary fee-race.** If the bridge+CPFP
  is still in the mempool as `expiry_height` arrives, the server's valid
  expiry sweep of the original VTXO output can RBF-replace the bridge
  package (the bridge's nSequence is RBF-signaling) → the client loses
  the WHOLE channel VTXO. Mitigated by the `F` operational lead (exit
  early; the arithmetic looks conservative, no off-by-one found) but the
  code cedes CPFP escalation once fees exceed the stake and cannot force
  miner preference. Safe once the bridge confirms; exposed before.
- **R3 [HIGH residual] NEW (codex r2) — the exit-fee reserve is
  advisory, unreserved, and underpriced — which makes R2 reachable in
  realistic conditions.** `channel_open_exit_reserve_sat`
  (bark/src/channels/mod.rs:3065,3100-3135,4775-4780; config.rs:225-243)
  is a POINT-IN-TIME balance check at open, not an earmark: later spends
  or concurrent channel opens consume it; the estimate uses `regular`
  fees while a time-critical exit needs `fast`; and an underfunded exit
  proceeds anyway. `daemon_manual_sync` also suppresses the deadline
  automation. So a drained/underfunded/manual-sync wallet can leave the
  bridge package unconfirmed at expiry → R2's whole-VTXO loss.
  Fix candidates: durably earmark the reserve, price it at `fast`,
  refuse/alert on underfunding, and don't let manual-sync silently
  disable deadline exits.

### Conformance / lower

- **C1 [MED conformance] — `static_remote_key` profile mismatch.** Spec
  says core-only uses `zero_fee_commitments` WITH `static_remote_key`;
  stage-1 rejects that bit (bark-channels/config.rs:81-102). Either
  stage-1 is nonconforming or the spec needs correction. No custody
  loss (the exact allowlist blocks mixed-type negotiation).
- **GAP 5 [LOW ↓ from MED] — reorg coverage debt.** The impl handles
  canonicality/unconfirmation/replacement/reselect; the one real
  omission is GAP 9's branch. Tests are narrow.
- **Dust [LOW]** — a sub-dust `to_remote`/HTLC trimmed to fee is a
  bounded loss (~≤330 sat), standard LN behavior.
- **P2A pinning / Esplora [LOW]** — no theft (a rival child only
  consumes the anchor; the parent is value-bearing); minor: the Esplora
  path lacks the Core-version capability check the bitcoind path has.

### Confirmed NOT gaps
Downgrade/no-forfeit safety intact (single-input checkpointed split,
nSequence=0, old scope retired only post-decision, parent-exit watch
armed through reorg); ordering sound; coop-close/downgrade-PONR/
FALLBACK_WON/pre-bridge expiry wired+tested; server submitting the
bridge = stage-2.

### Compounding
In ONE revoked-commitment close the client can lose its `to_local`
balance AND multiple HTLC amounts — all converge on GAP 1's dead relay.
Fixing GAP 3 without GAP 2 still leaves server balances unclaimable.

### Remediation → root cause (for the fold-in plan)
- **Root cause A (client relay)** fixes GAP 1, GAP 4, old-GAP-8, and
  GAP 9's reselect — mirror the c3 server `CaptureBroadcaster` (relay
  non-close-candidates) on the client `QueueBroadcaster`, and add the
  coop-close reselect. Stock-LDK-safe, no fork.
- **Root cause B** — add the server `Event::SpendableOutputs` sweep.
- **Root cause C** — add a production server force-close for the WD-16
  post-actualization recovery.
- **R1, R2, C1** — requirements decisions (bound/accept/document/spec-
  reconcile), not necessarily code.
