# Verdict: REWORK

The new builder/round approach is sound, but the design must additionally domain-separate `ChannelFunding`'s output construction from a board funding output's: a distinct VTXO type with distinct spending rules should not reuse another type's exact taproot construction.

## Task-by-task verdict

1. **Arkoor chokepoint — PASS**

Every production arkoor construction funnels through `ArkoorBuilder`:

| Flow | Construction |
|---|---|
| Direct arkoor, including offboard split helper | [bark/src/arkoor.rs:145](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark/src/arkoor.rs:145) |
| Lightning pay | [pay.rs:407](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark/src/actions/lightning/pay.rs:407) |
| Lightning revocation | [pay.rs:653](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark/src/actions/lightning/pay.rs:653) |
| Lightning receive-claim | [receive.rs:512](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark/src/actions/lightning/receive.rs:512) |
| Server reconstruction for all cosign requests | [server/src/arkoor.rs:57](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/arkoor.rs:57) |
| VTXO-pool delivery—an additional production caller omitted from the note | [server/src/vtxopool.rs:326](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/vtxopool.rs:326) |

All package constructors converge on `ArkoorBuilder::new_isolate_dust`; package/server reconstruction converges on `ArkoorBuilder::from_cosign_request`, which calls the same central `new` ([package.rs:192](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/arkoor/package.rs:192), [package.rs:276](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/arkoor/package.rs:276), [package.rs:404](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/arkoor/package.rs:404), [mod.rs:1339](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/arkoor/mod.rs:1339)).

The remaining grep hits are library unit tests, vectors, and integration tests. No production server/bark path mints or spends through arkoor without this funnel.

2. **Required parameter / fail-closed behavior — PASS**

The central constructor receives:

- The complete input VTXO.
- Normal destinations.
- Isolated destinations.

Dust isolation always calls the same central constructor after partitioning, so neither half is hidden ([mod.rs:1044](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/arkoor/mod.rs:1044), [mod.rs:1132](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/arkoor/mod.rs:1132)). `from_cosign_request` also funnels through it. The only direct struct reconstruction is a private typestate transition carrying already-checked fields.

With all constructor variants—including both `from_cosign_request` variants—taking the required argument, omission fails compilation. `UpgradeOutput` cannot authorize an input and `DowngradeInput` cannot authorize an output.

3. **Round shared chokepoint — PASS**

- Self-signed participation calls `validate_payment_amounts` before registration: [round/mod.rs:499](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/round/mod.rs:499).
- Delegated participation calls the same function before its database write: [round/mod.rs:1874](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/round/mod.rs:1874).
- The validator iterates every fetched input and every output: [round/mod.rs:849](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/round/mod.rs:849).
- Tree construction consumes only admitted `all_outputs`: [round/mod.rs:743](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/round/mod.rs:743).
- Forfeit processing requires inputs belonging to the stored, previously validated participation: [round/forfeit.rs:79](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/round/forfeit.rs:79).

There is no second delegated database writer or alternate tree/forfeit admission path.

4. **Other VTXO mechanisms — domain-separation gap**

The design had `ChannelFunding` share a board funding output's exact taproot
construction ([design note:88](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:88)). A distinct VTXO type with distinct spending rules must not reuse another type's exact output construction — a cosignature produced over one type's output should not be able to validate against another's. Fix: give channel-funding its own domain-separated (non-colliding) taproot; a structural, stateless property. See the design note §2d for the mechanism (a constant unspendable domain-marker leaf). Other audited mechanisms — VTXO pool issuance, LN settlement, expiry sweeps, offboard, fixed-policy tree construction — need no comparable change.

5. **MR-4 downgrade opt-in — PASS**

The sanctioned split is an OOR/arkoor spend, so passing `DowngradeInput` after split verification is coherent. It authorizes only the input half; MR-1 merely introduces the enum/check and passes `None` everywhere.

6. **Mechanical consistency — CHANGES REQUIRED**

- The PV-9 reframing in the note is correct: pre-channel rejection of `0x08` is a structural baseline fact, not a post-MR test.
- The §7 table is still stale: [line 496](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-bark-upstreaming-plan.md:496) calls PV-9 `T`; it should record baseline structural verification.
- PV-6 still says “shared-validator” and “all 3 generic paths” at [line 494](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-bark-upstreaming-plan.md:494).
- BR-7 is present; MR-3 ownership for BR-3/4/8 and OP-27 is now correct.
- Commit 1 can build atomically if it updates all library, bark, server-vtxopool, vector, and integration-test calls. The board safeguard must now land in that same decode-plus-invariant commit.

## Severity-ranked findings

1. **Blocker — `ChannelFunding` must be domain-separated from a board funding output** (distinct VTXO type → distinct, non-colliding taproot construction; §2d).
2. **Minor — §7 PV-9 and PV-6 rows remain stale.**
3. **Minor — the note’s exhaustive arkoor-caller inventory omits server VTXO-pool delivery, although the builder invariant covers it.**

## Verified clean

- `boats`: clean at `a7f99611273c3344ff0a44f11b11a9eb3242734e`.
- Baseline: clean `../bark-stage1` at `ea33bbf4fedd28323cd2f6a57e6cb108819d6196`.
- LDK `v0.2.4` resolves to `b720a198…`; `make_funding_redeemscript` sorts the two serialized funding keys.
- No files were modified.

**Final verdict: REWORK.** Keep approach (b), and add domain separation of the channel-funding taproot from a board funding output (distinct type → distinct construction).