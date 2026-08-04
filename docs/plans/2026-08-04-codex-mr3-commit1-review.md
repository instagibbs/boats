# Codex review record — MR-3 commit 1 (bark-channels production node)

**Subject**: the first commit of the captaind-channels MR (internal MR-3,
part 3 of the upstream series): "bark-channels: promote the harness into
the production embedded-LDK node", on bookmark
`ark8-channels-stage1-captaind` (parent = protocol-surface tip `1ddde4ad`).
Commit hashes across the arc: `eb96c997` (round 1) → `459f4dcd` (round 2)
→ **`d7bab25c` (round 3, final)** — one commit, amended in place.

**Scope reviewed**: node assembly over type-erased LDK seams
(build/fail-fast reload), production `UserConfig` + designated-channel-type
gate, the `lightning-background-processor` 0.2.3 async driver (async
`KVStore` manager persistence, `CancellationToken` shutdown, future boxed
at the reveal site), and the fee-bump UTXO reserve ledger. Server glue
(postgres `Persist`/`KVStore`, `db_executor`, nursery broadcaster) is
deliberately absent — design note §2b homes it in the scaffold commit.

**Process**: 3 codex rounds (`codex exec --sandbox read-only`, max
reasoning), fixes folded into the commit between rounds. Round prompts and
verdicts under the session scratchpad; verbatim verdicts summarized here.

## Round 1 — REWORK (4 Important / 4 Minor; all verified real, all fixed)

1. **I — reserve did not deduplicate input outpoints**: a repeated
   `OutPoint` could fund both tranches, then double-release. → dedup before
   selection + regression test.
2. **I — greedy largest-first selection refused feasible partitions** (its
   45k/40k/20k vs 50k+30k example) and conflated that with aggregate
   insufficiency. → bounded exhaustive subset search (top-16 candidates,
   prefer fewer inputs then less overshoot, conservative refusal past the
   cap) + distinct `Unpartitionable` error + adversarial tests.
3. **I — `NetworkGraph` hardcoded Regtest** (harness leftover). → network
   flows from `ChainParameters`; `reload` gained an explicit network param.
4. **I — "no scorer" doc overclaim**: a `FixedPenaltyScorer` exists for the
   router's signature. → docs narrowed to "no gossip, no graph-based
   pathfinding; the fixed-zero-penalty scorer is never updated/persisted".
5. **M — KVStore double wrote synchronously at call time**, pinning nothing
   asynchronous. → writes complete inside their future behind a holdable
   gate; driver test proves the exit path awaits the final manager write.
6. **M — legacy-anchors negative test vacuous.** → bit now also rides on
   the otherwise-designated set.
7. **M — no fail-fast / multi-monitor reload coverage.** → new test:
   corrupt manager panics, one corrupt monitor among two panics, honest
   reload restores both channels with both monitors watched, dormant.
8. **M — stale "harness" crate docs/description.** → rewritten.

Round-1 verified-clean list included reload fail-fast semantics, the
quiesce/async-durability documentation, the driver composition (boxing +
the `'static` no-gossip phantom judged sound), and — notably — the then
blocklist-shaped channel-type gate and SCID handling ("correct").

## The gate rework (user-triggered, between rounds 1 and 2)

Greg's question — "is `is_designated_channel_type` the type we REQUIRE?" —
exposed two weaknesses codex had blessed:

- The gate was a **blocklist** (require zero-fee-commitments, refuse three
  named bits); the design note (§11.8 "accept only the designated stock
  type") prescribes an **allowlist**. Rework: `designated_channel_type()`
  + equality, refusing any other bit without naming it.
- **`option_scid_alias` must be REQUIRED**, not merely tolerated: spec §5
  makes it a MUST on the final type, only the funder can propose the bit,
  so the acceptor requiring it enforces the funder's obligation (two-sided
  checks); functionally, the synthesized real SCIDs are node-local fiction
  the peers disagree on, so an alias-less counterparty could legally put a
  real SCID on the wire.

The new integration pin (gate must accept what real negotiation produces)
failed on first run and surfaced a genuine LDK fact: **the negotiated
zero-fee-commitments type does NOT carry `static_remote_key`** (unlike the
legacy anchor types) — the designated set is exactly
`{anchor_zero_fee_commitments, scid_privacy}`.

Process lesson: the round-1 prompt stated the invariant at config level
("scid alias is negotiated") rather than requirement level ("the gate must
require the bit in the negotiated type") — codex verified what it was
told. Later prompts state spec-level obligations.

## Round 2 — REWORK (8/8 round-1 findings CLOSED; gate confirmed; 1 Important + 5 Minor new)

Gate confirmation: LDK 0.2.4 clears explicit `static_remote_key` when
negotiating zero-fee commitments; `Features::PartialEq` treats trailing
zero bytes as equal (no representation-normalization hazard); requiring
`scid_privacy` judged appropriate.

New findings (all verified real, all fixed):

1. **I — refusal path could overflow-panic on `Amount` arithmetic.** →
   checked totals with a distinct `AmountOverflow` error before anything
   locks (later partial sums are subsets of the validated totals);
   `Amount::MAX` test with zero locks after.
2. **M — subset search excluded mask 0** (zero parent requirement wrongly
   `Unpartitionable`). → search from 0; empty-tranche test.
3. **M — gated KVStore double violated same-key call ordering** under
   concurrent futures. → call-time generations + tombstoned removals; a
   stale write can never clobber or resurrect a newer state.
4. **M — reload test's "pumps stopped = quiesced" claim false**
   (`lightning-net-tokio` processes on its own). → full both-transport
   quiesce before snapshot. (Same fallacy had already bitten the driver
   test live: the gated shutdown window let a lagging final
   `revoke_and_ack` advance the monitors past the exit-time manager
   snapshot, wedging the post-reload channel — fixed by quiescing the
   transport before the gate dance; quiesce-before-snapshot exercised for
   real.)
5. **M — taproot negative vacuous under the equality gate.** → taproot as
   the sole extra bit on the designated set.
6. **M — comment misattributed acceptor-side scid-alias support** to
   `negotiate_scid_privacy` (LDK advertises acceptor support
   unconditionally; the flag drives the funder's proposal). → corrected.

## Round 3 — PASS

All six round-2 findings CLOSED; "no new findings or amendment
regressions". (Sandbox could not rerun the socket-based tests — denied
localhost binds — closure supported by inspection + the supplied green
runs.)

## Final state and verification

- Commit **`d7bab25c`** on `ark8-channels-stage1-captaind` (single
  logical commit; series: opener → protocol → this).
- `cargo check --all --tests` green; `cargo clippy -p ark-lib --tests` and
  `-p bark-channels --tests` clean (0 warnings); upstream unit recipe
  (`cargo nextest run --workspace --exclude ark-testing`) **521/521**
  (30 bark-channels tests: 11 original release-contract + 4→6 config +
  8→12 reserve + driver e2e + fail-fast reload); timing-sensitive e2e
  tests stable across repeated runs.
- Owed later: changelog file keyed to the MR number at MR-open; the
  concrete `fee_bump_reserve_sat`/max-feerate defaults land with the
  scaffold commit's config surface, sized from the real package weights.
