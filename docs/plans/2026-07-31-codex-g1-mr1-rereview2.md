# Second re-review

## Prior-open-item verdicts

| Item | Verdict |
|---|---|
| Lightning-pay input bypass | **Closed for the three named cosign modes.** Direct arkoor, Lightning-pay, and receive-claim all call `validate_cosign_request`; receive-claim calls it before `settle` ([ln/mod.rs:794](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/ln/mod.rs:794)). |
| No residual mint/spend path | **Still open — blocker.** See finding 1. |
| Collapsed commit plan | Correct ordering, but commit 1 is not safe/test-green until finding 1 and PV-9 are corrected. |
| Runtime numbering | **Correct:** upgrade MR-2; downgrade MR-4. |
| PV-9 relabel | Baseline claim correct, proposed MR-1 test incorrect. |
| §7 labels | Partially corrected; ownership gaps remain. |
| Destination-filter wording | **Correct.** |

## Findings

1. **High — round admission still permits a ChannelFunding spend/mint path.**

   The proposed round-input refusal is pinned to `check_fetch_round_input_vtxos` ([design:98](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:98)), which only serves self-signed participation ([round/mod.rs:472](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/round/mod.rs:472)).

   Delegated participation independently fetches inputs, performs only policy-agnostic `check_spendable`, then stores the participation ([round/mod.rs:1836](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/round/mod.rs:1836), [round/mod.rs:1851](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/round/mod.rs:1851), [round/mod.rs:1878](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/round/mod.rs:1878)). This is exploitable: delegated attestations verify against `vtxo.user_pubkey()` ([attestations.rs:145](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/attestations.rs:145)), and the generic forfeit signs the same `[A,S]` key path ([forfeit.rs:149](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/forfeit.rs:149)).

   Additionally, the new round-output match arm must explicitly reject ChannelFunding; otherwise arbitrary output requests flow into `VtxoTreeSpec` and mint a bridgeless VTXO ([round/mod.rs:743](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/round/mod.rs:743), [round/mod.rs:877](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/round/mod.rs:877)).

   Required fix: reject ChannelFunding inputs and outputs inside shared round `validate_payment_amounts`, called by both interactive and delegated paths, and test both modes. Keep the separate offboard input refusal.

2. **Medium — PV-9’s proposed test cannot exist after commit 1.**

   The baseline decoder genuinely rejects `0x08` through its wildcard error arm ([vtxo/mod.rs:1088](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/vtxo/mod.rs:1088)). But after MR-1 adds the `0x08` decode arm, the same `decode_vtxo_policy` cannot be tested as rejecting it, contrary to [design:204](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:204).

   Make PV-9 a cross-version/baseline compatibility test, or classify it as structurally verified against `ea33bbf4`. The MR-1 disabled-admission test is separate.

3. **Medium — §7 ownership remains inconsistent.**

   - BR-7 disappeared entirely when `BR-1..9` was split.
   - BR-3/4/8 list only MR-2 runtime ownership, although the note says MR-2/MR-3 ([plan:499](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-bark-upstreaming-plan.md:499)).
   - OP-23..28 lists only MR-2 runtime, while OP-27 is explicitly implemented by MR-3 ([plan:403](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-bark-upstreaming-plan.md:403)).
   - PV-6 attribution to MR-1 is now correct, but its test wording should include delegated round and offboard coverage.

4. **Low — explicitly require the Lightning-pay ChannelFunding-input regression.**

   The test table names Lightning-pay only for destination refusal and lists input refusal as arkoor/round/offboard ([design:201](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:201)). The exact previously blocking LN-pay input case should be named.

## Verified clean

- `validate_cosign_request` has full input and normal/isolated destination policies available before builder construction.
- The three named paths reach it before DB/spent-state mutation; receive-claim reaches it before preimage settlement.
- Exact-variant refusal does not disturb Pubkey or existing HTLC flows.
- An additional caller, Lightning revocation, also uses the validator and already restricts inputs/outputs to its intended policies.
- The server-only VTXO-pool Arkoor builder uses fixed Pool inputs, ServerHtlcRecv destination, and Pubkey change; it is not an adversarial bypass.
- Offboard genuinely needs its own refusal.
- Taproot construction, LDK v0.2.4 funding-key sorting, grouped carriers, tags, corrected MR numbering, and destination-filter wording remain clean.
- `boats` and `bark-stage1` remained clean at `fd809eb` and `ea33bbf4`, respectively.

**Final verdict: REWORK.** The shared-validator Lightning blocker is fixed, but the delegated-round spend and round-output mint perimeter is a genuine remaining protocol-safety defect.