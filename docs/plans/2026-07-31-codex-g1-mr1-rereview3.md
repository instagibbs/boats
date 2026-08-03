# Verdict: REWORK

The new builder/round approach is sound, but the claimed universal invariant misses one real VTXO mechanism: the board cosigner can presign a key-path spend of a future `ChannelFunding` outpoint.

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

4. **Other VTXO mechanisms — FAIL: board cosign bypass**

The design deliberately makes `ChannelFunding` byte-identical to board funding outputs ([design note:88](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:88)). That exposes this attack:

1. `request_board_cosign` accepts an arbitrary outpoint; no transaction or funding provenance is supplied ([ark.rs:171](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/rpcserver/ark.rs:171)).
2. `cosign_board` validates amount, fee and expiry, then signs that outpoint without checking its origin ([server/src/lib.rs:691](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/lib.rs:691)).
3. Board signing assumes exactly the same aggregate key, server expiry leaf, amount and expiry as `ChannelFunding` ([board.rs:44](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/board.rs:44)).
4. A user constructs a future upgrade arkoor transaction and knows its witness-independent txid/outpoint before obtaining arkoor signatures.
5. The user first obtains a board cosignature for that future outpoint, then performs the authorized MR-2 upgrade using matching key, amount and expiry.
6. Once the arkoor chain is published, the presigned board exit spends the `ChannelFunding` output immediately through its key path with `Sequence::ZERO` ([vtxo/mod.rs:314](/home/greg/bitcoin-dev/cleanroom/bark-stage1/lib/src/vtxo/mod.rs:314)), producing a user `Pubkey` VTXO instead of the balance-pinned bridge.

A “reject currently known ChannelFunding outpoints” lookup is insufficient: board signing can occur first. The fix must be race-safe in both orderings—for example, proven board-funding provenance or durable outpoint reservation mutually exclusive with future `ChannelFunding` creation. Add a regression using a predicted future upgrade outpoint.

Other audited mechanisms—VTXO pool issuance, LN settlement, expiry sweeps, offboard, and fixed-policy tree construction—do not provide a comparable bypass.

5. **MR-4 downgrade opt-in — PASS**

The sanctioned split is an OOR/arkoor spend, so passing `DowngradeInput` after split verification is coherent. It authorizes only the input half; MR-1 merely introduces the enum/check and passes `None` everywhere.

6. **Mechanical consistency — CHANGES REQUIRED**

- The PV-9 reframing in the note is correct: pre-channel rejection of `0x08` is a structural baseline fact, not a post-MR test.
- The §7 table is still stale: [line 496](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-bark-upstreaming-plan.md:496) calls PV-9 `T`; it should record baseline structural verification.
- PV-6 still says “shared-validator” and “all 3 generic paths” at [line 494](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-bark-upstreaming-plan.md:494).
- BR-7 is present; MR-3 ownership for BR-3/4/8 and OP-27 is now correct.
- Commit 1 can build atomically if it updates all library, bark, server-vtxopool, vector, and integration-test calls. The board safeguard must now land in that same decode-plus-invariant commit.

## Severity-ranked findings

1. **Blocker — board cosigning is a presigning oracle for future `ChannelFunding` outputs.**
2. **Minor — §7 PV-9 and PV-6 rows remain stale.**
3. **Minor — the note’s exhaustive arkoor-caller inventory omits server VTXO-pool delivery, although the builder invariant covers it.**

## Verified clean

- `boats`: clean at `a7f99611273c3344ff0a44f11b11a9eb3242734e`.
- Baseline: clean `../bark-stage1` at `ea33bbf4fedd28323cd2f6a57e6cb108819d6196`.
- LDK `v0.2.4` resolves to `b720a198…`; `make_funding_redeemscript` sorts the two serialized funding keys.
- No files were modified.

**Final verdict: REWORK.** Keep approach (b), but extend its perimeter to the board-cosigning mechanism with a race-safe prohibition.