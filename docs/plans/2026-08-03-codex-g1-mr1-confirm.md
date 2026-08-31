## Finding

1. **Medium — normative Taproot construction remains stale and underspecified.**

   **Citation:** The note adds a placeholder marker and only an example constant ([§2a](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:88), [§2d](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:204)), while the specs still require the board-identical, one-leaf construction ([02-vtxo.md](/home/greg/bitcoin-dev/cleanroom/boats/02-vtxo.md:165), [08-channels.md](/home/greg/bitcoin-dev/cleanroom/boats/08-channels.md:204), [matrix PV-2/PV-3](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-conformance-matrix.md:67), [stage plan](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-bark-upstreaming-plan.md:154)). Commit 1 does not mention these updates ([design §6](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:305)).

   **Why:** `<marker>` and “e.g. a tagged hash” do not determine identical output keys across implementations or ensure consensus-level unspendability. The inequality test alone would not catch two implementations choosing different markers—or a marker script that leaves true on the stack.

   **Fix:** Normatively pin the exact marker construction: leaf version, complete script bytes and constant derivation, and two depth-1 leaves. A fixed `OP_RETURN <exact constant>` TapScript is one valid choice. Update `02-vtxo.md`, `08-channels.md`, matrix PV-2/PV-3, diagrams/stage plan, and commit 1. Add tests for:

   - Exact marker/root/scriptPubKey vector.
   - 65-byte expiry control block containing the marker sibling.
   - Successful stock `TimelockSign` expiry spend.
   - Failed attempted marker-leaf spend.

## Verified clean by task

1. **Touch-nothing-shared — verified clean.** `VtxoClause` is closed ([clause.rs](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/vtxo/policy/clause.rs:379)); the watchman matches it and calls the stock witness builders ([signer.rs](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/watchman/signer.rs:134)). Control blocks come from `vtxo.output_taproot()` ([clause.rs](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/vtxo/policy/clause.rs:29), [vtxo/mod.rs](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/vtxo/mod.rs:531)), not a one-leaf template. The client signer is equally generic. The channel policy must return its existing `TimelockSign` from `clauses()`, but the marker requires no clause, enum, or signer change. Claim (c) is confirmed.

2. **Mechanism — verified clean, subject to pinning above.** Board funding uses the one-leaf `cosign_taproot` ([signed.rs](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/tree/signed.rs:69), [board.rs](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/board.rs:53)). Adding a second leaf changes the Merkle root and therefore the TapTweak/output key under normal hash assumptions. Board and channel signatures use different tweaked keys and prevout-script sighashes. Key-path signing still consumes one `tap_tweak()` through unchanged `tweaked_key_agg` ([musig.rs](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/musig.rs:57)). The expiry sweep remains valid, and control-block accounting automatically grows by exactly 32 witness bytes/32 wu ([clause.rs](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/vtxo/policy/clause.rs:148)).

3. **Spec consistency — not clean.** Finding 1.

4. **Internal note consistency — verified clean.** §2a, §2d, §5’s inequality, and §6 commit 1 all select the marker-leaf mechanism. “Mutate,” “one-leaf,” “+6 wu,” and EC-tweak occur only as explicitly rejected alternatives; no pending/primary residue remains. Domain separation and the builder admission invariant are complementary and non-conflicting.

5. **Regression — verified clean.** The four client flows still converge on the package builder ([direct](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark/src/arkoor.rs:145), [pay](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark/src/actions/lightning/pay.rs:407), [revocation](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark/src/actions/lightning/pay.rs:653), [receive](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark/src/actions/lightning/receive.rs:512)); server reconstruction reaches the same central constructor. Both round paths call `validate_payment_amounts` before mutation ([interactive](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/round/mod.rs:499), [delegated](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/round/mod.rs:1874)). Offboard refusal, grouped carriers, forced-match inventory, and atomic commit-1 placement remain intact. Commit 1 remains buildable after adding the normative documents/tests above.

`boats` is clean at `68936858`; baseline is clean and exactly `ark8-channels-stage1` at `ea33bbf4`. LDK `v0.2.4` peeled to `b720a198`; its funding-script constructor remains public and key-sorting.

**Verdict: PASS-WITH-CHANGES.** The construction is correct; the exact marker and normative spec updates must land before implementation.