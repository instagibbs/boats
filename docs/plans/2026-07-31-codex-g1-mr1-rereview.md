# G1 re-review

Final verdict: **REWORK**.

## Per-finding verdicts

1. **F1 — STILL OPEN**

   The destination half is correctly designed. `validate_cosign_request` is the shared chokepoint called by direct arkoor, Lightning-pay, and receive-claim ([arkoor.rs:165](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/arkoor.rs:165), [ln/mod.rs:135](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/ln/mod.rs:135), [ln/mod.rs:794](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/ln/mod.rs:794)). Placing the destination check before builder construction at [arkoor.rs:57](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/arkoor.rs:57) covers normal and isolated outputs. The builder really does copy supplied policies into both normal and isolated VTXOs ([arkoor/mod.rs:427](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/arkoor/mod.rs:427), [arkoor/mod.rs:544](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/arkoor/mod.rs:544)).

   But the input refusal is not shared. The proposed explicit arm is in direct `cosign_oor` only ([arkoor.rs:147](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/arkoor.rs:147)). Lightning-pay fetches arbitrary inputs, calls `set_vtxos`, and reaches the builder without any input-policy check ([ln/mod.rs:83](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/ln/mod.rs:83), [ln/mod.rs:102](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/ln/mod.rs:102)). Builder construction does not enforce `is_arkoor_compatible` ([arkoor/mod.rs:1132](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/arkoor/mod.rs:1132), [arkoor/mod.rs:1340](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/arkoor/mod.rs:1340)). A crafted Lightning-pay request can therefore spend a `ChannelFunding` input into an HTLC output.

   Required: reject `ChannelFunding` inputs in the shared validator for all unauthorized modes, and test Lightning-pay input refusal.

   Also remove the `validate_cosign_request / cosign_oor_with_builder` ambiguity. `cosign_oor_with_builder` is too late for receive-claim: the preimage is durably settled before that call ([ln/mod.rs:797](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/ln/mod.rs:797)). Add a receive-claim destination test proving rejection before that write.

2. **F2 — CONFIRMED-RESOLVED**

   The proposed construction is literally the board construction: `combine_keys([A,S]).x_only_public_key().0`, followed by `cosign_taproot` ([board.rs:53](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/board.rs:53), [signed.rs:69](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/tree/signed.rs:69)). `combine_keys` sorts compressed keys before aggregation ([musig.rs:43](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/musig.rs:43)). With equal amount, keys, server key, and expiry, the resulting funding `TxOut` is byte-identical.

   Minor code-level correction only: `tweaked_key_agg` expects `[u8; 32]`, so signing uses `spend_info.tap_tweak().to_byte_array()` ([musig.rs:57](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/musig.rs:57)).

3. **F3 — CONFIRMED-RESOLVED**

   Grouped request/response carriers make half-pairs unrepresentable internally, and the note now covers all reconstruction sites: `with_vtxo`, `convert_vtxo`, and `set_vtxos` ([arkoor/mod.rs:225](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/arkoor/mod.rs:225), [package.rs:33](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/arkoor/package.rs:33), [package.rs:73](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/arkoor/package.rs:73)). Existing absent-field traffic can map cleanly to `None`.

   Test wording should explicitly name `with_vtxo`, but that is polish.

4. **F4 — CONFIRMED-RESOLVED**

   The inventory now covers Display, all six exhaustive policy methods, encoding/mirror decoding, m0021, round construction/telemetry, generic input refusals, and watchman. The expiry-claim watchman arm is correct: it waits for expiry and the signer already supports `TimelockSign` ([watchman/policy.rs:115](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/watchman/policy.rs:115), [watchman/signer.rs:89](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/watchman/signer.rs:89)).

5. **F5 — STILL OPEN**

   The design note itself now labels the listed requirements as construction/schema primitives ([design:162](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:162)), and the requested bridge/signature/vector tests were added.

   However, the claimed matrix relabel did not happen. The matrix says the live owner source is the stage plan’s §7 table ([matrix:13](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-conformance-matrix.md:13)); that table still assigns `BR-1..9` to MR-1 and does not use “construction/schema primitive only” ([stage plan:498](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-bark-upstreaming-plan.md:498)). `OP-23..28` is still only broadly split between MR-1 and MR-2 ([stage plan:509](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-bark-upstreaming-plan.md:509)).

   The “legacy decoder posture” test also does not establish PV-9: an old-shape protobuf round-trip proves neither that the old policy decoder rejects `0x08` nor that rejection precedes mutation. The actual baseline decoder does reject it structurally ([vtxo/mod.rs:1084](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/vtxo/mod.rs:1084)), but the proposed test is mislabeled.

6. **F6 — CONFIRMED-RESOLVED**

   Inertness is now narrowed honestly, JSON/source-API effects are acknowledged, and the wire commit includes server `ArkInfo`, conversion, bark-json, and its field-completeness test ([design:26](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:26), [design:194](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:194)). Those are the actual forced consumers ([server/lib.rs:264](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/lib.rs:264), [bark-json:492](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-json/src/cli/mod.rs:492)).

7. **F7 — CONFIRMED-RESOLVED**

   The unknown vector is `0xff`, and an upstream `0x08` collision explicitly reopens G1 ([design:56](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:56), [design:179](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:179)). The baseline currently allocates only `0x00..0x07` ([vtxo/mod.rs:999](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/vtxo/mod.rs:999)).

## New findings

- **High — Lightning-pay input bypass:** described under F1. This is a protocol-safety defect, not polish.
- **Medium — Commit plan is not genuinely per-commit-buildable as scoped.** Commit 1 introduces the enum, so exhaustive matches in direct arkoor and Bark address eligibility must change immediately, yet commit 3 claims those input gates. If those arms move into commit 1, commits 1–2 still decode `0x08` before the shared destination gate lands. Put policy decoding, compiler-forced refusal arms, and the shared destination/input gate in the same commit.
- **Medium — Runtime-owner numbering is wrong.** The note says MR-3/MR-5 and calls MR-3 “server” ([design:31](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:31), [design:164](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:164)); the series plan assigns server open/admission to MR-2, client runtime to MR-3, and downgrade to MR-4 ([stage plan:369](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-bark-upstreaming-plan.md:369), [stage plan:397](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-bark-upstreaming-plan.md:397)).
- **Low — “Client send eligibility” only checks destination policy.** Input selection remains policy-agnostic at [arkoor_send.rs:244](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark/src/actions/arkoor_send.rs:244). Reword as destination eligibility or add an explicit input filter.

## Verified clean

- Exact board-output reproduction.
- LDK v0.2.4 funding script is public and sorts the two compressed funding keys.
- Request/response/ArkInfo protobuf tags are free.
- No existing policy is falsely rejected by the proposed exact-variant gate.
- Watchman expiry handling is correct.
- Both reviewed worktrees remained clean at `ea33bbf4`; LDK tag resolved to `3ff69ae3`.

**Verdict: REWORK.** The ownership/test corrections alone could be follow-up changes, but the Lightning-pay `ChannelFunding` input bypass and gate/commit placement must be fixed before implementation.