G1 verdict: **REWORK**. Baseline verified clean at `ea33bbf4f` over upstream `c5f37986b`.

## Findings

1. **Blocker / “nothing creates it” is false**

   **Citation:** [design:11-19](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:11), [server/src/arkoor.rs:36](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/arkoor.rs:36), [server/src/arkoor.rs:122](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/arkoor.rs:122), [lib/src/arkoor/mod.rs:415](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/arkoor/mod.rs:415), [08-channels.md:1164](/home/greg/bitcoin-dev/cleanroom/boats/08-channels.md:1164).

   **Why:** `cosign_oor` checks only input policy. The shared validator has no destination-policy gate, and the builder copies arbitrary destination policies into new VTXOs. Once `0x08` decodes, a generic request can create a `channel-funding` output without channel fields or bridge, then mark the ordinary input spent before signing. Lightning-pay and receive-claim also use this validator and permit arbitrary companion/destination policies ([server/src/ln/mod.rs:79](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/ln/mod.rs:79), [server/src/ln/mod.rs:735](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/ln/mod.rs:735)).

   **Fix:** In MR-1, make the shared pre-builder validator reject every `ChannelFunding` destination—normal and isolated—unless the later channel-admission path explicitly authorizes it. Also reject `ChannelFunding` inputs in generic cosign consumers. Add direct-arkoor and shared-Lightning tests proving rejection occurs before any DB/output/spent-state mutation.

2. **High / The taproot construction names the wrong MuSig operation**

   **Citation:** [design:37-43](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:37), [board.rs:53](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/board.rs:53), [tree/signed.rs:69](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/tree/signed.rs:69), [musig.rs:53](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/musig.rs:53).

   **Why:** The internal key is the **untweaked** `musig(A,S)`. Board construction uses `musig::combine_keys`, then `cosign_taproot` adds the one expiry leaf and performs the Taproot output tweak. `tweaked_key_agg` is appropriate later for key-path signing, not internal-key construction. Following the note literally would not reproduce the board output byte-for-byte.

   **Fix:** Specify exactly:

   ```rust
   let internal = musig::combine_keys([a, s]).x_only_public_key().0;
   cosign_taproot(internal, s, expiry_height)
   ```

   Use `tweaked_key_agg([A,S], spend_info.tap_tweak())` only for bridge signing. `clauses()` returns exactly one `TimelockSignClause`.

3. **High / Wire carrier invariants and preservation paths are underspecified**

   **Citation:** [design:88-104](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:88), [arkoor/mod.rs:169](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/arkoor/mod.rs:169), [arkoor/package.rs:33](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/arkoor/package.rs:33), [arkoor/package.rs:73](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/arkoor/package.rs:73).

   **Why:** Four independent `Option` fields permit invalid half-present requests/responses. Moreover, `with_vtxo`, `convert_vtxo`, and `set_vtxos` reconstruct requests; merely updating `convert.rs` does not ensure the extension survives the actual inbound path. Compiler fixes could silently fill these reconstructions with `None`.

   **Fix:** Use grouped internal carriers:

   - `Option<BridgeCosignRequest { channel_id: [u8; 32], pub_nonce }>`
   - `Option<BridgeCosignResponse { pub_nonce, partial_sig }>`

   Conversion must reject half-present or malformed protobuf pairs and preserve the group through every package transformation. Test protobuf → internal → `set_vtxos`/`convert_vtxo` → protobuf.

4. **Medium / The forced-match inventory is incomplete**

   **Citation:** [design:44-56](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:44).

   **Why:** The actual compiler-forced sites are:

   - `VtxoPolicyKind::Display` at [policy/mod.rs:170](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/vtxo/policy/mod.rs:170).
   - Six `VtxoPolicy` methods at [policy/mod.rs:720](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/vtxo/policy/mod.rs:720): `policy_type`, compatibility, arkoor key, user key, taproot, clauses.
   - Encoding at [vtxo/mod.rs:1025](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/vtxo/mod.rs:1025).
   - Bark destination eligibility, server input eligibility, round output validation, round telemetry, watchman policy.
   - The omitted historical Bark migration at [m0021_fix_lightning_movements.rs:49](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark/src/persist/sqlite/migrations/m0021_fix_lightning_movements.rs:49), whose safe arm is `false`.

   `FromStr` and the server decoder mirror are semantically required but not compiler-forced because they contain wildcard/tag matches.

   The watchman expiry-claim arm is correct: it waits until expiry, the signer supports `TimelockSign`, and claim construction is generic. It is not truly unreachable until finding 1 is fixed. Also audit non-exhaustive consumers: round inputs and offboard currently accept policies generically ([round/mod.rs:313](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/round/mod.rs:313), [offboards.rs:187](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/offboards.rs:187)).

5. **Medium / The claimed matrix coverage exceeds what these tests establish**

   **Citation:** [design:106-117](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:106), [matrix:67-84](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-conformance-matrix.md:67), [stage plan:469-480](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-bark-upstreaming-plan.md:469).

   **Why:** Pure construction tests cannot discharge:

   - BR-3/8 source, storage, and “never reread live”.
   - BR-4 negotiated-amount equality.
   - BR-7’s commitment-input portion.
   - BR-14 key exchange/registry, BR-16 commitment signing.
   - BR-15 “once at open” or BR-17 session freshness, atomicity and retry behavior.
   - OP-23 identifier lookup semantics or OP-24 at-most-one-part admission.

   Additional missing pins include exact `0x08 || 33-byte-key` encoding, PV-4 key-role separation, full-VTXO generic round-trip/version rejection, true legacy protobuf/decoder compatibility, the no-bridgeless server rejection, pair-preservation, and final-script execution.

   **Fix:** Label those IDs explicitly as “construction/schema primitive only” and retain runtime ownership in MR-2/3. Add a `verify_tx` bridge test, assert one 64-byte DEFAULT witness, reject a corrupted partial signature, and test funding-key-order independence.

6. **Medium / “No observable effect” and commit-3 scope are inaccurate**

   **Citation:** [design:11-19](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:11), [design:95-97](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:95), [bark-json ArkInfo:28](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-json/src/cli/mod.rs:28), [field-completeness test:492](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-json/src/cli/mod.rs:492).

   **Why:** New `0x08` inputs necessarily change decode/rejection timing. Public enum variants and added public struct fields are source-breaking for exhaustive matches/struct literals. Exposing `supports_channels: false` through Bark JSON also changes observable JSON, although protobuf encoding can remain byte-identical because false is omitted. Commit 3 must additionally touch server `ArkInfo` construction, conversion fixtures, package transformations, and Bark JSON.

   **Fix:** Narrow inertness to “previously valid non-channel protocol flows remain byte/transaction-identical; new channel-shaped requests reject before mutation.” Expand commit scopes and add the required breaking/API changelog entries once the MR number exists.

7. **Low / Unknown-tag test reserves the next byte accidentally**

   **Citation:** [design:33-36](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:33), [design:111](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:111).

   **Why:** `0x08` is currently free and the reservation plan is sensible, but `0x09` is the likely next allocation and makes a brittle “unknown” vector.

   **Fix:** Use `0xff`. Any upstream `0x08` collision should reopen G1 and update spec, matrix, vectors and compatibility fixtures—not merely renumber code.

## Verified clean by task

1. **Correctness:** The intended PV-1..3 shape is correct after fixing the tweak wording. Exact board-byte reuse is possible through `combine_keys + cosign_taproot`. LDK’s public `make_funding_redeemscript` sorts keys and produces the canonical BOLT-3 2-of-2; `.to_p2wsh()`, full-value out0, zero-value P2A out1, version 3, locktime zero, and `Sequence::from_height(delta)` are correct. DEFAULT key-path sighash with `Prevouts::All` is correct.

2. **Match safety:** Proposed reject/false/None/user-key arms are safe. Watchman expiry-claim is the correct terminal fallback. No additional exhaustive Bark client match was found beyond address eligibility and the omitted migration.

3. **Wire:** Tags request 7/8, response 3/4 and ArkInfo 21 are free. `optional bytes` matches existing conventions. Resolved Prost 0.14.3 skips unknown fields ([prost derive:189](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/prost-derive-0.14.3/src/lib.rs:189)); it does not preserve them on re-encode. The attestation cannot absorb the new fields: it hashes only input ID, output count, each amount and policy ([attestations.rs:388](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/attestations.rs:388)). Reword the note: amount is committed separately; policy bytes commit the user key.

4. **Inertness:** Not clean until finding 1 is fixed. An old binary still rejects `0x08` during conversion before mutation, satisfying PV-9; the newly decoding but disabled MR-1 binary needs an explicit admission gate.

5. **Tests/commits:** Not clean as written. The three-commit split can build independently once the missing forced arms, package carriers, ArkInfo consumers and Bark JSON scope are included.

6. **Upstream fit:** `lib/src/channel.rs` is consistent with `board.rs`; `ark-lib` already directly depends on Lightning 0.2.4; the LDK constructor is public and appropriate. Main likely objections are the unsafe admission gap, public-API/changelog impact, incomplete carrier invariants, and overbroad conformance claims.

**Verdict: REWORK.** Findings 1–5 should be resolved in the design before implementation begins.