# Verification verdict: REWORK

Seven findings are fixed, five are partial, and F1 remains fundamentally unpinned.

| Finding | Verdict | Verification |
|---|---|---|
| F1 | **REFUTED** | The P2A empty witness is correct—LDK explicitly requires no witness. But transaction validity remains vacuous; details below. |
| F2 | **CONFIRMED-FIXED** | The new test constructs the correct funding output with an empty input witness, stops the funder pump, and checks the panic text ([release_contract.rs:1015](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:1015), [release_contract.rs:1048](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:1048)). This would fail if v0.2.4’s empty-witness guard disappeared. |
| F3 | **CONFIRMED-FIXED** | The server type now comes from its own `ChannelPending` ([common/mod.rs:1430](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/common/mod.rs:1430)); both final `ChannelDetails.channel_type`s are asserted ([release_contract.rs:50](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:50)). |
| F4 | **CONFIRMED-FIXED** | Events are channel-ID matched, exact `ProcessingError` text is checked, removal/usability are asserted, and the bump commitment is tied to the funding outpoint ([release_contract.rs:554](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:554), [release_contract.rs:565](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:565), [release_contract.rs:578](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:578), [release_contract.rs:591](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:591)). |
| F5 | **CONFIRMED-FIXED** | Both queues are watched concurrently while unrelated events are drained ([common/mod.rs:1162](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/common/mod.rs:1162)); `is_channel_ready == false`, unusability, and the timer tick are covered ([release_contract.rs:93](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:93), [release_contract.rs:117](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:117)). |
| F6 | **PARTIAL** | Reverse payment is real. The ordering instrumentation is not sufficient: the broadcaster allocates its sequence number before inserting the transaction into captured storage ([common/mod.rs:249](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/common/mod.rs:249)). Thus `broadcast_seq < event_seq` can hold while capture has not completed. |
| F7 | **CONFIRMED-FIXED** | Historical headers are retained; `feed_historical` invokes only `transactions_confirmed` ([common/mod.rs:1063](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/common/mod.rs:1063)). The test advances the live tip, verifies height/hash do not regress, and pays afterward ([release_contract.rs:1075](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:1075), [release_contract.rs:1091](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:1091)). |
| F8 | **CONFIRMED-FIXED** | Reload reuses the seed while incrementing `starting_time_secs` ([common/mod.rs:559](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/common/mod.rs:559), [common/mod.rs:581](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/common/mod.rs:581)), satisfying v0.2.4’s uniqueness contract. One documentation correction remains below. |
| F9 | **PARTIAL** | The new node is dormant and the pump is aborted and awaited. However, snapshots are taken before quiescing ([common/mod.rs:551](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/common/mod.rs:551)), and the socket tasks are discarded rather than awaited ([common/mod.rs:912](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/common/mod.rs:912), [common/mod.rs:928](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/common/mod.rs:928)). In v0.2.4, `disconnect_socket` merely sets a flag; the transport closes later. The surviving peer is never observed disconnected. |
| F10 | **CONFIRMED-FIXED** | Server and client are genuinely reconstructed in sequence ([release_contract.rs:693](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:693), [release_contract.rs:738](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:738)); SCIDs and payments in both directions are checked after each restart. Subject to F9’s lifecycle race. |
| F11 | **PARTIAL** | Unannounced state, both inbound aliases, cross-peer alias equality, and a manually constructed alias route are real ([release_contract.rs:852](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:852), [release_contract.rs:863](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:863), [release_contract.rs:881](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:881)). But `negotiate_scid_privacy` remains at v0.2.4’s default `false` ([common/mod.rs:874](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/common/mod.rs:874)), and channel-type support is not asserted. The one-hop route therefore proves local alias lookup, not alias-only negotiation. No three-node test is requested. |
| F12 | **PARTIAL** | Pump awaiting and restore-on-drop are improvements ([common/mod.rs:510](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/common/mod.rs:510), [common/mod.rs:1213](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/common/mod.rs:1213)). The hook remains process-global; overlapping panic tests can restore hooks out of order. Stopping the custom pump also does not stop the independent `lightning-net-tokio` socket tasks. |
| F13 | **PARTIAL** | Per-item allowances, dependency cleanup, alias documentation, and insecure-value notes are fixed. Planning language remains in both new test files (“future … harness”) ([common/mod.rs:4](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/common/mod.rs:4), [release_contract.rs:4](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:4)). Module imports also remain mid-file ([common/mod.rs:1245](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/common/mod.rs:1245), [common/mod.rs:1497](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/common/mod.rs:1497)), contrary to project guidance. |

## F1 vacuity re-run

The empty P2A witness is not a dodge. LDK v0.2.4 explicitly says keyless anchors need no witness and leaves that input empty (`events/bump_transaction/mod.rs:139-145, 895-912`).

These regressions would still pass:

- An HTLC claim using version 2. The test checks lock time but never version ([release_contract.rs:483](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:483)); v0.2.4 requires version 3 for this channel type.
- An anchor descriptor pointing outside the commitment transaction. The child only has to spend the event-provided outpoint; that outpoint/value is never matched against an actual commitment output.
- An inflated `commitment_tx_fee_satoshis`, anchor value, or HTLC descriptor value. Package arithmetic trusts those event fields ([release_contract.rs:333](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:333), [release_contract.rs:456](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:456)).
- A duplicated wallet outpoint. Every occurrence is recognized and its value is counted again ([release_contract.rs:378](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:378), [release_contract.rs:500](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:500)).
- Garbage, nonempty wallet or HTLC witnesses. Assertions check only non-emptiness ([release_contract.rs:403](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:403), [release_contract.rs:510](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:510)).

Therefore “exact input sets,” “wallet inputs signed,” and “from-first-principles package fee” remain overclaims.

## New findings

- **Important:** Restart state is serialized before pump/transport fencing, allowing manager and monitor state to change during snapshotting.
- **Important:** The shared sequence records broadcast callback entry, not completed capture.
- **Minor:** The reload documentation says persisted `channel_keys_id` has “no dependency” on starting time ([common/mod.rs:532](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/common/mod.rs:532)); v0.2.4 embeds the original start seconds/nanos in that ID. It is independent only of the replacement instance’s start time.
- **Minor:** The alias route never asserts the alias differs from both synthesized real SCIDs, leaving a small additional vacuity.

## Commit hygiene and verified clean

- HEAD equals `ark8-channels-stage1`; the detached worktree is clean.
- Exactly two commits exist over `upstream/master`.
- The split remains structurally correct: scaffold/manifests first, tests only second.
- The empty-witness claim is now genuinely tested; eleven tests are present.
- The second message is not fully accurate because of the F1 and transport-ordering overclaims.
- `git diff --check` and both Rust indentation/blank-line prechecks pass.
- `fee_for_weight` exactly matches v0.2.4’s ceiling calculation.
- The historical-confirmation API use and manual `Route` structure are valid for v0.2.4.
- The prebuilt binary lists all eleven tests. Runtime verification could not proceed: the read-only sandbox rejects the localhost bind at [common/mod.rs:908](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/common/mod.rs:908) with `PermissionDenied`. No green runtime result is claimed.

**Final verdict: REWORK.** F1 still permits invalid on-chain transactions to satisfy the release contract; restart fencing, alias privacy, and panic-hook isolation also remain incomplete.