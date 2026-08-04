# REWORK

The confirmation-injection gate itself is real in stock LDK 0.2.4, and most of the twelve admission checks—including the one-`exit_delta` floor and mandatory split headroom—match the spec and landed code. The design is not safe to implement yet, however: its forwarding configuration cannot enforce the claimed HTLC invariants, D2/D3 leave captaind unable to meet the mandatory per-HTLC force-close deadline, registration is not linearized against the parent-exit watcher, and the proposed channel state machine contradicts the actual LDK event order.

## Findings

1. **Severity: Critical**  
   **Location:** [§1:48](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:48), [§3:160](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:160), [§8:399](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:399), [spec:1076](/home/greg/bitcoin-dev/cleanroom/boats/08-channels.md:1076), [LDK forwarding check](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/ln/channel.rs:10830)  
   **Why:** D4 cannot be implemented with the knobs named. LDK checks an incoming forward against the **outgoing channel’s** `ChannelConfig::cltv_expiry_delta`, not the incoming channel’s `F`. Setting each channel to its own floor therefore fails for high-`F` incoming channel A forwarded over lower-`F` channel B. The named `max_htlc_value_in_flight_msat`, per-HTLC maximum, and `accept_forwards` fields do not exist: stock LDK exposes an inbound percentage cap, HTLC count, and [`accept_forwards_to_priv_channels`](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/util/config.rs:870). It has no arbitrary absolute per-HTLC cap. Also, captaind is not guaranteed to be non-terminal: LDK always advertises keysend ([channelmanager.rs:15714](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/ln/channelmanager.rs:15714)), accepts spontaneous payloads, and exposes them only after the HTLC is irrevocably committed. A post-event assertion cannot enforce I-6b.  
   **Fix:** Define a static `forwarding_cltv_ceiling`; configure **every outgoing channel** to that value before release and refuse any open whose `F` exceeds it. Avoid runtime ceiling increases because LDK temporarily accepts the prior config. Translate exposure limits into the real percentage/count fields and bound channel capacity if an absolute cap is required. Add an LDK-level final-hop CLTV hook—likely stage-2 fork work—or explicitly keep the subsystem non-operating until it exists.

2. **Severity: Critical**  
   **Location:** [§8 D2/D3:377](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:377), [spec I-6d:1086](/home/greg/bitcoin-dev/cleanroom/boats/08-channels.md:1086), [spec expiry recourse:1036](/home/greg/bitcoin-dev/cleanroom/boats/08-channels.md:1036)  
   **Why:** A forwarding captaind can have an unresolved incoming leg after it has paid the outgoing leg. It then must force-close the incoming channel by the `F` deadline. D2 assigns that duty only to clients, while D3 discards the completed bridge. Calling LDK force-close is insufficient: its commitment spends an off-chain bridge output that captaind cannot actualize without the fully signed bridge and genesis chain. The claimed fallback—“the sweep already recovers the whole output”—is only defined when the channel-VTXO output was already actualized. This exposes a real tension between I-6d and the spec’s otherwise-optional BR-12/13/WD-16 posture.  
   **Fix:** For a forwarding server, make bridge retention mandatory: upload and atomically retain the completed bridge at registration, retain the signed genesis chain, and add a scheduler using LDK’s exposed pending-HTLC expiries. Specify how captaind actualizes bridge plus commitment. Otherwise forwarding must remain disabled. Resolve the corresponding spec/profile contradiction explicitly.

3. **Severity: Critical**  
   **Location:** [§5:239](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:239), [§10:449](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:449), [current registration](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/lib.rs:855), [concurrent listeners](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/sync/mod.rs:35)  
   **Why:** Late-registration refusal is not linearized against block processing. Registration can check “final exit not confirmed,” then the listener can confirm and resolve the unarmed watch, after which registration commits and releases `ChannelReady` against backing already being clawed back. The note also does not require the signed levels, registration marker, and watch arm to commit in one transaction. A crash between marker and feed can strand a channel unless explicitly reconciled.  
   **Fix:** Make registration and chain-event resolution contend on one DB row/advisory lock. In one transaction: validate the complete signed set, verify no terminal chain-derived resolution, persist signatures, mark registered, and arm the watch. Feed only after commit through an idempotent outbox/reconciler; startup must catch up the real chain first and then release committed registrations. Feed both `ChannelManager` and `ChainMonitor`; historical confirmations must not regress `best_block_updated`. Persist resolution reason, block hash/height, and reverse it on reorg. Replace “suspend operation” with the actual profile behavior: stock LDK force-closes on funding unconfirmation ([release test](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs:604)).

4. **Severity: Important**  
   **Location:** [§6:316](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:316), [§11.3:478](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:478), [actual event order](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/common/mod.rs:1603)  
   **Why:** Every valid upgrade reaches `ChannelPending` before the Ark cosign. Section 11.3 nevertheless says incomplete Ark state at `ChannelPending` must force-close, while §6 creates `channel_state` only after cosign. Implemented literally, every valid open closes. Reference commit `0595bb594` explicitly fixed this by tolerating **no** Ark state while force-closing only partial/inconsistent state. “All refusals have no state mutation” is also inaccurate: an LDK pending channel already exists and consumes resources.  
   **Fix:** Specify a durable state machine: `opening(temp_id)` → `awaiting_upgrade(permanent_id, funding_txo, redeem_script, final_type)` → `cosigned(backing/watch-unarmed)` → `registered/released`. No Ark state at the expected pre-cosign stage is valid; partial or contradictory state is fatal. Add pending-open quotas, timeout/cleanup, and ensure the gate lands before or with the first commit that accepts opens.

5. **Severity: Important**  
   **Location:** [§2c:108](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:108), [spec funding keys:107](/home/greg/bitcoin-dev/cleanroom/boats/08-channels.md:107), [KeysManager derivation](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/sign/mod.rs:2158)  
   **Why:** The note simultaneously says the server’s long-term `S` is the BOLT funding key and that BOLT keys are distinct from `A/S`. The former violates BR-14. The cited old-branch override was stale at branch tip; its signer deliberately discarded the override. Stock `KeysManager` correctly derives a unique per-channel funding key.  
   **Fix:** Use ordinary LDK-derived funding keys. Persist the canonical `funding_redeem_script` and output supplied by [`ChannelPending`](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/events/mod.rs:1394), then compare bridge output 0 directly against it; do not introduce a long-term-key signer override.

6. **Severity: Important**  
   **Location:** [§5 SCID:228](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:228), [SCID encoding](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/util/scid_utils.rs:12), [collision panic](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/ln/channelmanager.rs:3464)  
   **Why:** Bridge-txid-derived `tx_index` is the correct lever, but the storage and collision domain are missing. LDK inserts both aliases and real SCIDs into the same map and panics on either collision. The schema stores no synthetic height/index/bump. The release test cited as restart evidence confirms an already-ready manager; LDK serializes its SCID and ignores the repeated feed, so it does not prove allocator recovery before first release. The alias test explicitly shows that `option_scid_alias` was **not negotiated**.  
   **Fix:** Persist `{anchor_hash, height, tx_index, collision_bump}` transactionally before feeding, with a unique full-SCID constraint and checked wrap. Check against every local real SCID and alias. Set `negotiate_scid_privacy=true`, require the final `ChannelPending` type to support it, and add crash-before-first-feed plus concurrent-allocation tests.

7. **Severity: Important**  
   **Location:** [§7:323](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:323), [spec WD-2:495](/home/greg/bitcoin-dev/cleanroom/boats/08-channels.md:495), [schema:454](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:454)  
   **Why:** Arming on registration is correct, but resolution is underspecified. An unregistered watch must not resolve on a mere prefix; only the final input-exit confirmation or an ancestor expiry sweep makes later registration terminal. `resolved_at` alone cannot identify or unwind a reorg. “Embedded or standalone watchmand” also leaves a money-safety service optional: channels can be enabled while the current `watchman` service is disabled.  
   **Fix:** Define resolution predicates precisely and persist `resolution_reason`, spending txid, block hash, and height; reopen chain-derived resolutions above a fork. Treat ancestor-sweep resolution as a late-registration refusal too. Choose an always-on embedded watcher for MR3, or make channel startup depend on a positively healthy standalone watcher.

8. **Severity: Important**  
   **Location:** [§11:465](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:465), [LDK zero-fee requirements](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/util/config.rs:190), [BumpTransaction contract](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/events/mod.rs:1677)  
   **Why:** Stock 0.2.4 supplies the expected external-wallet `BumpTransaction` path, but the note has no confirmed fee-reserve admission/accounting at manual channel acceptance. It also carries only half of reference fix `6edd047f7`: mixed bump PSBTs must sign only wallet-owned inputs, leaving the LDK channel input for the channel signer. Reference fix `6c954e14f`, pinning `ChannelCloseMinimum` to the relay floor to prevent cooperative-close negotiation wedges, is omitted.  
   **Fix:** Add a documented fee-bump reserve policy checked before `accept_inbound_channel`, foreign-input-safe PSBT signing, and the relay-floor `ChannelCloseMinimum` mapping. State the Core v29+/TRUC relay requirement. All eight listed hardening items are otherwise relevant; none is inherently teleport-only.

9. **Severity: Important**  
   **Location:** [§2c D1:114](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:114), [async processor signature](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-background-processor-0.2.3/src/lib.rs:910)  
   **Why:** D1 is answerable now. The locked background processor is 0.2.3. Its async processor composes with a custom synchronous `ChainMonitor::Persist` and an external shutdown-capable sleeper, and already handles pending forwards and manager persistence. But manager persistence requires an async `KVStore`; this conflicts with “NOT a KVStore shim.”  
   **Fix:** Adopt `process_events_async` and add a small Postgres `KVStore` mapping for the singleton manager while retaining the custom monitor `Persist`. If a KVStore adapter is rejected, choose the hand-written loop explicitly.

10. **Severity: Important**  
    **Location:** [§8 D5:409](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:409), [LDK confirmation budget](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/chain/channelmonitor.rs:277), [current depth default](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/captaind.default.toml:26)  
    **Why:** Leaving the margin TBD is a code-start blocker. `18` alone covers only one transaction under LDK’s own confirmation assumption, while the unified HTLC path includes genesis levels, bridge, commitment, and HTLC second stage.  
    **Fix:** Define an explicit operational confirmation target `C=18`. A conservative depth-aware default is  
    `slack = (C−1)·D + 3·C + 3`,  
    where `D` genesis confirmations already contribute one block each to `F`, the three `C`s cover bridge/commitment/HTLC, and `3` is processing grace. With the current `D=100`, use `cltv_claim_slack=1757`, giving default `F=2001` at `exit_delta=144`. Validate checked arithmetic and document that no finite value covers unbounded congestion.

11. **Severity: Minor**  
    **Location:** [§4:189](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:189), [§6:257](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:257), [§9:434](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:434), [§12:510](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:510)  
    **Why:** Several factual statements should be corrected: captaind does not “hold its own `ChannelReady` from step 1”—LDK has not generated it; the part-4 client must also pass `UpgradeOutput` to construct the transfer, so captaind is not the sole caller; PV-6 concerns authorized **spending** via `DowngradeInput`, not upgrade creation; reserve `0` is clamped to at least 1000 sat by LDK; `KeysManager` does not need `ldk_virtual_fundings`; and “no onion routing” should read “no gossip/pathfinding,” since direct Lightning forwarding still processes onion payloads.  
    **Fix:** Correct these statements and the §12 requirement mapping.

## Verified sound

- The registration gate primitive works: with ordinary `accept_inbound_channel`, `minimum_depth ≥ 1`, and no `Confirm` feed, neither timer ticks nor peer messages produce `ChannelReady`; feeding the confirmation does. The release test and [`check_get_channel_ready`](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/ln/channel.rs:11093) agree.

- Manual funding is funder-only. Only the client calls `unsafe_manual_funding_transaction_generated`; captaind learns the outpoint from `funding_created` and can reach readiness solely through confirmation injection.

- SCID synthesis through `(anchor height, synthetic tx_index, vout)` is correct; the index is 24-bit and bridge-txid derivation plus persisted collision probing is appropriate.

- Stock `zero_fee_commitments` produces the expected external-wallet `BumpTransaction::{ChannelClose, HTLCResolution}` flow. The existing release test proves the v3/P2A package behavior.

- Admission checks 1–12 cover the server halves of OP-2..5, OP-14..16, OP-23..26, BR-4/18, and DA-6/7. The `max−2` headroom MUST is correctly retained.

- The `1× pinned_exit_delta` formula is correct. The removed second delta belonged to the excluded custom HTLC-success CSV; it must not be carried into stage 1.

- Baseline code claims are correct: generic arkoor uses `ChannelAuthorization::None` and rejects `ChannelFunding`; builder/package shape-bounding is present; the watchman expiry arm already routes `ChannelFunding` to `decide_action_expiry`; `OptionalService` currently has only the production `watchman` user; bridge helpers exist; V55 is next and `just dump-server-sql-schema` is the repository convention.

- No-refresh stage 1 has a safe cooperative terminal path if mandatory split headroom and the proactive downgrade → round-refresh → re-upgrade lifecycle are enforced.

## Answers to D1–D5

- **D1:** Use async `lightning-background-processor` 0.2.3 plus a Postgres `KVStore` for the manager; retain custom monitor `Persist`.

- **D2:** Reject as written. A forwarding server needs the I-6d force-close scheduler.

- **D3:** Retain the client-completed bridge at registration; close-outcome-only is incompatible with D2/D4.

- **D4:** Use a static global forwarding CLTV ceiling and real LDK percentage/count caps. Add an LDK hook for final-hop/keysend enforcement.

- **D5:** Use depth-derived `1757` for the current `D=100` default, from `(18−1)·D + 3·18 + 3`; document and test the assumption.