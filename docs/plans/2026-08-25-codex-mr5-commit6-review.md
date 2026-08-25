# Codex review: MR-5 commit 6 — "bark: never bid past the stake" (PARKED at 8 rounds, P2-converged)

Commit `cac12a382` "bark: never bid past the stake [FOR UPSTREAM:
independent of channels]" on `ark8-channels-stage1-close` (bark-stage1),
atop the trivial fixture commit `f5905ffbc` ("server-rpc, bark-json:
complete the ArkInfo test fixtures" — fold targets at the next stack
rewrite: `425de13ea` for the server-rpc roundtrip fixture and the
bark-json `ark_info_base`, `19c278036` for `baseline_channel_info`; both
are under the pushed tip today).

**Stack rewritten 2026-08-25 (Greg's call):** the `gettxspendingprevout`
rival probe was excised from commit 4 IN HISTORY (`daa645d03` →
`5872d8537`, blind escalation only from birth), so this commit contains
no removal — only the completed no-external-pricing rule. c5/fixtures
re-picked (`21eebb436`/`f5905ffbc`); the final tree verified
byte-identical to the fully-tested pre-rewrite tip, and commit 4' passed
its own battery. Battery at the tip: units 595/595, sdk channels 18/18,
server channels 22/22.

Origin: Greg's fee-burn question ("wouldn't want a counterparty to
overbid, and we RBF theirs and burn a ton of funds") plus his follow-up
call to REMOVE the `gettxspendingprevout` rival probe ("I don't think
it's particularly safe to keep spending beyond fast based on external
inputs").

## Round 1: FAIL — 2 P1, 3 P2

1. **P1 the ceiling was the package's face value, not the wallet's
   stake** (strangers' tree branches, the peer's commitment balance). →
   an explicit `stake` parameter on the builder; exit requests carry the
   exit's claimable value; the downgrade response carries OUR leaves.
2. **P1 ceding was sticky** (the escalation counter never reset; a climb
   past the ceiling pinned every later tick above it). → ceding RESETS
   the escalation — abandons the climb, not the work; rejected bids cost
   nothing.
3. **P2 a ceded downgrade left a dead association blocking scope-end
   forever.** → full retraction (association deleted) when the recorded
   child is dead.
4. **P2 the RBF minimum charged parent+child bandwidth** though only the
   child is replaced. → child-only (rule 4 uses the replacement's size).
5. **P2 the estimation walk and its anchor-only test parents broke.** →
   value-carrying fixtures; stake threading came later.

## Round 2: FAIL — 4 P1, 2 P2

1. **P1 per-parent ceiling × N-level chain = N × stake.** → the chain's
   committed child fees deduct from every other parent's stake.
2. **P1 stakes were not live recovery** (channel exits at full funding —
   deliberately KEPT, with the rationale made explicit: the local claim
   set is not enumerable pre-commitment-confirmation and an understated
   stake strands time-critical HTLC claims; downgrade summed
   historically-owned leaves incl. Spent). → downgrade stake = live
   leaves only (not-yet-installed = fully exposed).
3. **P1 `mempool_ancestor_info` double-counted the queried child**
   (`ancestorsize` already includes it) — the understated standing rate
   re-triggered replacement at a constant target, RATCHETING REAL FEES;
   and the downgrade replaced its standing child with a plain
   `Effective` build (reject/cede livelock below the stake). → both
   fixed; accepted residuals: the builder's 1 sat/vB incremental
   bandwidth, the ancestor-inclusive rule-3 fee (bounded, never
   rejectable).
4. **P1 a small sibling's ceded bid reset a shared parent's
   escalation.** → requests MERGE per parent (stakes aggregate; one
   counter touch per parent per tick).
5. **P2 retraction unsafe** (eviction failure still retracted). →
   retraction requires a CLEAN eviction.
6. **P2 the estimator knowingly described a path live execution
   refuses.** → per-parent stakes through the walk; `uneconomic_txs`
   surfaced.

## Round 3: FAIL — 2 P1, 2 P2, 1 P3

1. **P1 the budget was in-memory only** (a restart restored the full
   stake) **and counted third-party children's fees as ours** (an
   attacker's expensive anchor child could zero our stake and stall the
   exit — a griefing vector the round-2 deduction itself introduced). →
   the builder returns the fee it commits; wallet-built children persist
   it with their association (`wallet_fee_sat`, migration 48; `None` on
   store PRESERVES); the deduction reads only `Wallet`-origin
   associations.
2. **P1 the retraction could erase a concurrently stored successor**
   (the generic leaf exits also manage the response parent). →
   compare-and-delete on the exact observed dead child.
3. **P2 shared fees deducted once per sibling, then summed.** → the
   deduction moved after the merge, component-level, txid-deduped.
4. **P2 estimator gaps** (gross stakes at one producer — documented
   residual; the walk hard-errored on a delivered-ceiling refusal). →
   the walk cedes per parent (canonically priced, `Option<Transaction>`
   children keep parent order).
5. **P3 stale openapi.json.** → regenerated (also picking up the
   `ExitChannelCooperativeClosingState` schema stale since the commit-3
   work); `bark/schema.sql` regenerated for migration 48.

(The round-3 numbered fixes above describe the state as of round 3;
rounds 4–8 below refined several of them — the durable budget, the
origin gating and the CAS machinery supersede the round-3 versions.)

## Round 4: FAIL — 3 P1, 2 P2

1. **P1 confirmed ancestors vanished from the budget** (only bumpable
   parents were recorded, and the in-memory fee info was lost on
   restart). → the builder RETURNS the fee it commits; wallet-built
   children persist it with their association (`wallet_fee_sat`,
   migration 48; `None` on store preserves only for the SAME child);
   every chain level joins the component's parent set.
2. **P1 third-party children still priced our replacements** (rule-3
   minimums from any standing child's fee). → `RbfRequirement` carries
   `wallet_child`; an under-target NON-wallet child is replaced BLIND at
   the target — never priced against.
3. **P1 the retraction could erase a concurrent successor.** →
   compare-and-delete on the exact observed dead child.
4. **P2 cross-child fee inheritance** through the preserve-on-NULL
   upsert. → identity-gated (CASE on child bytes / txid).
5. **P2 ceded walk parents priced at the bare parent deficit.** →
   canonical pricing (parent + canonical child weight).

## Round 5: FAIL — 2 P1, 3 P2

1. **P1 the DOWNGRADE path still priced any standing child** (the
   origin branch had only gone into the exit path). → branch on the
   association's stored origin; the stand-down also resets the
   escalation.
2. **P1 the stand-down compared against the ESCALATED bid** — a
   target-paying child discovered after one rejection got replaced,
   re-ratcheting. → stand-down FIRST against the unescalated target;
   a sufficient standing package resets the counter.
3. **P2 the rollback was check-then-write.** → guarded (round 6 made it
   atomic).
4. **P2 the estimator built Rbf from any child's fee.** → both producer
   sites mirror the live origin rule.
5. **P2 `uneconomic_txs` vanished past a funding shortfall.** → the
   walk classifies the remaining parents canonically before returning.

## Rounds 6–8: FAIL — concurrency narrowing (1 P1, then P2-only)

- R6 **P1 the guarded restore was still TOCTOU** → a new persister
  primitive `replace_exit_child_tx(expected_current: Option<Txid>, …) →
  bool`: an ATOMIC compare-and-swap in each backend (sqlite one
  transaction; adaptor one critical section), with shared test-suite
  coverage (CAS miss/hit, expect-absent miss/install).
- R6 **P2 the estimator's second producer missed the stand-down** →
  mirrors the live rules exactly.
- R7 **P2 candidate INSTALLATION was read-then-write** → the install is
  a CAS on exactly the association read moments before; a miss YIELDS
  the whole attempt (nothing broadcast).
- R8 **P2 a neutral yield ratcheted the bid** → a tri-state broadcast
  outcome; a yield maps to the AMBIGUOUS hand-off (evict for input
  safety, never escalate).
- R8 **P2 the post-broadcast persist could overwrite a concurrent
  writer** → REMOVED: the pre-broadcast CAS install is the sole durable
  write; post-accept adoption is memory-only.

## PARKED (Greg's call, 2026-08-25)

The review was parked after the round-8 fixes landed: the rounds had
converged (no P1 since round 5; rounds 7–8 were narrowing concurrency
P2s, both fixed), the commit's failure domain is fees (never principal),
and it is flagged for UPSTREAM consideration where it gets independent
review anyway. No round 9 was run; any residual findings ride with the
upstream submission.

Also: the 659-sat server admission e2e vectors moved onto 661-sat
(floor-compliant) channels with a new direct-RPC open-door floor-refusal
vector (the commit-5 floor had made the seeded 659 channel unreachable —
caught by the battery, not by review); the ~170 s fallback-won close e2e
joined the long-channel-e2e nextest timeout override.

## Upstream consideration (Greg's call)

This commit is FLAGGED FOR UPSTREAM independently of channels (an
`UPSTREAM NOTE` trailer rides in the commit message): the fee-burn
exposure predates the channels work and applies to every unilateral
exit — no economic ceiling, RBF minimums priced from whatever child
stood on the anyone-can-bump anchor (a third party's bulky
high-absolute-fee child dragged every replacement up to its fee), and
the `ancestorsize` double-count re-triggering replacement at a constant
target. The fixes land in the shared builder and exit driver, not the
channel code.

## Key design decisions (for the arc record)

- **The bid is never priced off external input.** The rival probe
  (`gettxspendingprevout`) is GONE — a rival's advertised feerate is
  attacker-controllable (P2A anchors are anyone-can-bump). Rivals are
  outbid only through blind escalation (own rejections, one
  incrementalfee per attempt), bounded by the ceiling.
- **The economic ceiling is intrinsic and stake-scoped**: the builder
  refuses any child fee above min(caller stake, delivered value), at
  both fee-growth points. Losing past that point is strictly cheaper
  than winning; ceding degrades to the closing fallback (same
  close-fixed balances) or the chain-overrules rescue.
- **The budget is durable, wallet-scoped, component-level**: wallet-paid
  child fees persist with their associations; an exit chain is ONE
  recovery; shared parents get one merged request, one counter, one
  aggregated stake.
- **Ceding abandons the escalation, not the work** (counter resets;
  sawtooth bidding — only an ACCEPTED bid ever pays), and a fully dead
  launch is retracted by compare-and-delete so it cannot hold a
  record's scope-end hostage.
- **Accepted residuals**: builder's 1 sat/vB incremental bandwidth
  (higher node settings surface as rejections; blind escalation
  recovers); rule-3 uses the ancestor-inclusive fee (overpays by the
  parent's own fee — zero for exits/responses); channel exits bound at
  funding value (HTLC-claim liveness over stake tightness); the
  estimator's gross-stake divergence (under-counts `uneconomic_txs`
  near the boundary; fee totals stay conservative).

## Follow-up fold (2026-08-25, from the surface-work design review)

Folded into this commit (rewritten `cac12a382` → `f13336286`): the
downgrade response's REPLACEMENT children now use the same
candidate-before-broadcast discipline as the generic manager — CAS
install over exactly the standing child read, yield on a concurrent
writer, CAS rollback to the standing child on rejection. Closes the
accepted-then-crash window that left an input-locking replacement
unnamed until the anchor-spent probe (the review's P1-3; the mirror of
this commit's own round-7/8 work, now applied to the channels path
too).

Final rewritten stack: c3 `671e7d635`, c4 `16e7101ba`, c5 `d0b9336e4`,
fixtures `7e6a41611`, tip `f13336286`. Tree verified byte-identical to
the tested pre-fold state; per-commit batteries re-run at c3/c4/tip.
