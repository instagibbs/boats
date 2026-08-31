# REWORK

Removing forwarding closes the two-leg forwarding failure, and the revised state machine, watch predicates, funding keys, and async processor are materially improved. Two critical gaps remain: terminal receipt can still require captaind to force-close and actualize the virtual funding, so D2/D3 are unsafe; and the release outbox is not fully linearized against live reorgs or durable `ChannelManager` persistence. Stock LDK also has no claimed pre-commit receive-floor hook.

## Closure table

| Finding | Status | Why |
|---|---|---|
| C1 | **PARTIAL** | Normal private-channel forwarding is rejected by [LDK’s outgoing-channel check](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/ln/channelmanager.rs:4923), but §3’s pre-commit terminal receive floor does not exist. |
| C2 | **OPEN** | Removing forwarding eliminates “outbound paid/inbound stuck,” but a terminal receiver that has exposed the preimage can still need the same bridge-retention and force-close machinery. |
| C3 | **PARTIAL** | The row lock, atomic commit set, and post-commit feed are correct; live reorg/feed ordering, feed-to-manager-persistence durability, and historical watch reconciliation remain unspecified. |
| F4 | **CLOSED** | The revised ordering matches `ChannelPending`: no Ark backing at `awaiting_upgrade` is valid, contradictory partial state is fatal, and quotas/timeouts are included. |
| F5 | **PARTIAL** | LDK-derived funding keys and redeem-script comparison are correct, but `ChannelPending` does not provide the claimed funding `TxOut` or amount. |
| F6 | **PARTIAL** | Persistence and final-type enforcement are added, but future LDK-generated aliases can still collide with an earlier synthetic SCID. |
| F7 | **PARTIAL** | Predicates, reorg metadata, and mandatory parent watcher are correct; already-confirmed prefixes/terminal spends can be missed when the row is inserted or armed. |
| F8 | **PARTIAL** | Foreign-input-safe signing and `ChannelCloseMinimum` are closed; the fee-bump reserve remains an assertion rather than an accounting policy. |
| F9 | **CLOSED** | `process_events_async` plus Postgres `KVStore` for the manager and custom monitor `Persist` matches the locked crate API. |
| F10 | **OPEN** | D5 remains TBD, and the proposed constant-package model is false. |
| F11 | **CLOSED** | The round-1 factual nits were corrected. |

## Findings

1. **Severity: Critical — D2/D3 remain unsafe for terminal HTLC receipt**  
   **Location:** [§3](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:193), [§6a](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:394), [§8 D2/D3](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:493), [spec deadline](/home/greg/bitcoin-dev/cleanroom/boats/08-channels.md:1076)  
   **Why:** `PaymentClaimed` follows durable monitor insertion of the preimage, not an irrevocable peer-side fulfill; LDK says it is generally available immediately after `claim_funds` ([channelmanager.rs](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/ln/channelmanager.rs:8686), [completion path](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/ln/channelmanager.rs:9448)). A client can disconnect then defer broadcasting its bridge/commitment until near CLTV expiry. Captaind’s captured commitment cannot spend the nonexistent bridge output, while the parent watch explicitly leaves bridge and commitment untouched ([§7](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:474)). The client can therefore create the same late preimage-versus-timeout race after captaind treated the payment as received. The expiry sweep is too late and only spends an actualized channel-VTXO output.  
   **Fix:** Either ship MR-3 with all production HTLC send/receive disabled, or retain the client-completed bridge at registration and add the `F`-deadline scheduler and actualization/fee-bump path in MR-3. Also explicitly run the expiry policy processor when channels are enabled; today the real processor is conditional on optional watchman configuration ([server startup](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/lib.rs:401)).

2. **Severity: Critical — the release gate still has reorg and crash holes**  
   **Location:** [§5](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:254), [§10](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:543), [LDK `Confirm` contract](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/chain/mod.rs:153)  
   **Why:** A remaining interleaving is: registration commits its release entry; the anchor is reorganized out before the entry is fed; the reorg has no injected confirmation to withdraw; the worker later feeds the now-stale anchor. LDK expressly forbids `transactions_confirmed` for a header no longer in the chain as of `best_block_updated` ([lines 161–163](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/chain/mod.rs:161)). Separately, the schema contains no outbox/delivery record, and feed success is not the same as durable manager persistence: the background processor writes the manager asynchronously ([processor](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-background-processor-0.2.3/src/lib.rs:1079)). A crash after feed/ack but before that write can restore an unfed manager with no retry. Finally, a final exit, ancestor sweep, or nonterminal prefix confirmed before watch insertion/arming leaves no watch record for late registration to inspect.  
   **Fix:** Give the release worker and reorg handler one serialized chain-generation gate; revalidate `anchor_hash` against the best chain immediately before feed; invalidate pending entries on reorg; and acknowledge delivery only after a durable manager-persistence barrier—or level-trigger reconciliation of every registered row. Add explicit outbox schema. At watch insertion and arming, reconcile the authoritative TxIndex/current chain under the same channel lock and immediately resolve or respond to already-seen transactions.

3. **Severity: Important — the terminal receive floor is not pre-commit**  
   **Location:** [§3 claim](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:197), [LDK commit transition](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/ln/channel.rs:8726), [invoice check](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/ln/channelmanager.rs:7921)  
   **Why:** LDK promotes the inbound HTLC to `Committed` before queuing it for onion/final-hop processing. The custom invoice delta is then checked post-commit. It is real only when supplied to `create_inbound_payment(..., Some(F))` ([API](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/ln/channelmanager.rs:13271)); setting the invoice field alone merely instructs the sender. Keysend has no corresponding receiver-created secret, and LDK always advertises keysend ([features](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/ln/channelmanager.rs:15714)).  
   **Fix:** Remove the pre-commit claim. Stock MR-3 must pass `Some(max_F)` into inbound-payment creation, advertise the same maximum across all eligible receiving channels, and fail every spontaneous payment without claiming it. A genuine pre-commit per-channel floor requires an LDK hook/fork.

4. **Severity: Important — funding-output persistence does not match the event API**  
   **Location:** [§2c](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:135), [state machine](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:404), [schema](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:553)  
   **Why:** `ChannelPending` supplies the outpoint, optional final type, and optional redeem script—not a funding output/value ([event](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/events/mod.rs:1394)). The amount exists earlier on `OpenChannelRequest` ([event](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/events/mod.rs:1614)), but the proposed state/schema does not persist it.  
   **Fix:** Persist `funding_satoshis`, temporary ID, counterparty, and requested type before acceptance; require fresh `ChannelPending` type/script to be `Some`; derive the canonical `TxOut` as `(funding_satoshis, P2WSH(redeem_script))`; then compare bridge out0 and outpoint.

5. **Severity: Important — SCID collision coverage is not future-complete**  
   **Location:** [§5 SCID](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:300), [schema](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:554)  
   **Why:** LDK’s future alias allocator checks only its private alias set ([allocator](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/ln/channelmanager.rs:4054)), not previously injected synthetic SCIDs; a later collision panics during `ChannelReady` insertion ([insertion](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/ln/channelmanager.rs:3464)). Also, `negotiate_scid_privacy` affects outbound private channels, so the client/funder must set it; captaind’s setting alone cannot negotiate this inbound channel.  
   **Fix:** Allocate synthetic indices only from `2500..2²⁴`; LDK aliases use `0..2499` ([fake-SCID domain](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/util/scid_utils.rs:87)). Continue checking existing real SCIDs. Add the omitted `vout` column—or make uniqueness `(height, tx_index)` because funding vout is fixed at zero—and explicitly configure the test/client funder. Requiring final `ChannelPending` support remains correct.

6. **Severity: Important — fee-bump reserve remains undefined**  
   **Location:** [§11.9](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:588), [LDK reserve requirement](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/util/config.rs:197)  
   **Why:** “Can always fund” has no reserve amount, feerate ceiling, concurrent-channel accounting, `claim_id` reuse, or release rule. LDK can emit repeated commitment and HTLC bump requests ([bump contract](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/events/bump_transaction/mod.rs:147)).  
   **Fix:** Define reserve weight and maximum-feerate assumptions, reserve confirmed UTXOs per live claim/channel at acceptance, reuse them across rebump events, and release only after terminal confirmation. Foreign-input-safe signing and the 253 sat/kw close floor are otherwise correct.

7. **Severity: Important — D5’s package model is false**  
   **Location:** [§8 D5](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:504), [spec exit order](/home/greg/bitcoin-dev/cleanroom/boats/08-channels.md:1012), [current exit engine](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark/src/exit/progress/states.rs:148)  
   **Why:** Each genesis level waits for confirmed inputs, and each zero-fee v3 transaction needs its own 1-parent/1-child CPFP package. `submitpackage` does not allow its parents to depend on one another ([Core RPC documentation](https://bitcoincore.org/en/doc/28.0.0/rpc/rawtransactions/submitpackage/)). LDK additionally emits HTLC-resolution work only after the commitment has one confirmation ([bump contract](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/events/bump_transaction/mod.rs:205)). The path is serial, not one package.

## Verified sound

- With every channel private, `accept_forwards_to_priv_channels=false`, `accept_intercept_htlcs=false`, and no manual intercept forwarding, stock LDK performs no normal relay.
- Therefore captaind cannot enter the forwarding-specific “outbound paid, inbound stuck” state.
- A cooperatively fulfilled endpoint HTLC remains off-chain; no commitment, HTLC second stage, or deferred success-CSV path is touched.
- The one-row registration/watch lock and single registration transaction are sound for events observed after the watch exists.
- Feeding both `ChannelManager` and `ChainMonitor` at the real anchor height without regressing `best_block_updated` is correct.
- The revised F4 state ordering and LDK-derived per-channel funding keys are sound.
- The unarmed final-exit/ancestor-sweep predicates, armed-prefix response, and reorg metadata are correct.
- A negotiated final type with `scid_privacy` does place the alias on `channel_ready` ([LDK](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/ln/channel.rs:11152)).
- D1, foreign-input-safe bump signing, and `ChannelCloseMinimum = 253 sat/kw` are sound.

## D5 recommendation

Use the serial model with `C=18`:

`cltv_claim_slack = (C−1)·D + 3C + 3 = 17D + 57`.

At current `D=100`, set **`cltv_claim_slack = 1757`**, giving **`F = 100 + 144 + 1757 = 2001`**. Derive or validate the minimum against configured `D`; no finite value protects against unbounded congestion.