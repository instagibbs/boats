# ARK #8 stage 1 — codex test-coverage audit (raw record)

Invocation: codex exec, gpt-5.6-sol, reasoning=max, sandbox read-only, one
pass against bark-stage1 @ 5c245e272, executing the prompt at
2026-08-31-channel-coverage-audit-prompt.md. Verbatim report below.
Consolidation + verification deltas: 2026-08-31-coverage-audit-consolidated.md.

# Channels stage-1 test-coverage audit

## 1. The matrix

Target: `5c245e272`, the 19-commit stack `b3697e3c5..5c245e272`. The worktree was clean. Test-body counts matched the prompt: lifecycle 32, SDK channels 19, proxy 4, payments 5, server channels 10, server PostgreSQL channels 8, release contract 14, and feed barrier 5.

Rows combine postures only when reachability, obligation, and proving assertion are identical. “Offered” and “received” are relative to the client. Individual-HTLC tests are not credited to the `both` payload.

Citations:

- `S` — [08-channels.md](/home/greg/bitcoin-dev/cleanroom/boats/08-channels.md)
- `R` — [security remediation](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-31-channel-security-remediation.md)
- `C` — [bark_sdk/channels.rs](/home/greg/bitcoin-dev/cleanroom/bark-stage1/testing/tests/bark_sdk/channels.rs)
- `L` — [bark_sdk/lifecycle.rs](/home/greg/bitcoin-dev/cleanroom/bark-stage1/testing/tests/bark_sdk/lifecycle.rs)
- `PX` — [bark_sdk/channel_proxy.rs](/home/greg/bitcoin-dev/cleanroom/bark-stage1/testing/tests/bark_sdk/channel_proxy.rs)
- `P` — [bark_sdk/payments.rs](/home/greg/bitcoin-dev/cleanroom/bark-stage1/testing/tests/bark_sdk/payments.rs)
- `SC` — [server/channels.rs](/home/greg/bitcoin-dev/cleanroom/bark-stage1/testing/tests/server/channels.rs)
- `SD` — [server/postgres/channels.rs](/home/greg/bitcoin-dev/cleanroom/bark-stage1/testing/tests/server/postgres/channels.rs)
- `RC` — [release_contract.rs](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/release_contract.rs)
- `FB` — [feed_barrier.rs](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/feed_barrier.rs)
- `WP` — [watchman/policy.rs](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/src/watchman/policy.rs)

### N0–N4: operation, PONRs, and funding actualization

| State / payload | Party + posture | Obligation | Verdict | Proving test and binding assertion |
|---|---|---|---|---|
| N0, open before registration | Client, honest/restarted | Hold a complete bridge, remain unusable before registration, resume durable open; `S:391-439,487-497` | COVERED | `C::open_channel_restart_matrix`: every non-ready record is asserted unusable at `C:257-265`; restarts at Funding, Feeding, and Done end in one Ready usable channel at `C:267-277`. |
| N0, corrupt bridge partial | Client vs adversarial server | Refuse an invalid bridge before registration; `S:433-436` | COVERED | `lib/src/channel.rs::test_bridge_cosign_rejects_corrupted_partial`: aggregation rejects the mutated partial. This is the correct pure-cryptographic proving layer. |
| N0, malformed/tampered upgrade | Server vs adversarial client/proxy | Bind amount, owner, funding outpoint, headroom, and attestation before cosign; `S:441-469` | COVERED | `SC::open_by_upgrade_admission` binds the negotiated amount, self-spend key, reconstructed bridge, and headroom at `SC:195-227,325-397`; `PX::open_with_tampered_attestation_is_refused` asserts the proxy actually mutated the request and no channel reached Ready at `PX:333-367`. |
| N0, registration response lost/duplicated | Client, restarted/lossy transport | Replay the same registration and converge; `S:391-439` | COVERED | `PX::open_survives_lost_registration_response` asserts the loss occurred after the server commit and the open reaches Ready at `PX:293-314`. |
| N0, duplicate cosign/registration | Server, honest/restarted | Make retries idempotent without changing the SCID or duplicating the watch; `S:146-168,485-509` | COVERED | `PX::open_through_duplicating_proxy_is_idempotent` requires an acted-on registration and a present stable SCID at `PX:257-290`; `SC:297-323` asserts one watch row after re-cosign. |
| N0, outgoing payment pending | Client, crash at issue/outcome | Preserve payment identity and settle or fail backwards; `S:133-144,563-578` | COVERED | `L::committed_send_survives_crash_and_settles` binds the same payment to terminal settlement; `L::failed_send_retries_same_invoice` and `crash_cut_send_strands` bind retry/terminal state. |
| N0, incoming payment pending | Client, crash at claim binding | Preserve the claim CAS and fail closed; `S:133-144,1103-1119` | COVERED | `L::claim_binding_crash_fails_backwards_and_recovers` asserts the held binding, backward failure, retry, and final Received state at `L:572-664`. |
| N0, collect binding pending | Server, crash/restart | Durably bind one claim and readmit after the crash cut; `S:133-144,1103-1119` | COVERED | `L::collect_binding_crash_readmits` uses the server crash seam and binds the same hash to the recovered collect at `L:671-731`. |
| N1, upgrade input-chain prefix only | Server, honest/restarted | Treat a prefix as a cue, not terminal foreclosure; `S:498-509` | DARK | No HEAD test stops at an upgrade prefix and proves registration remains admissible. `SC::channel_parent_exit_watch_lifecycle` mines through the final before judging the watch. See P2-3. |
| N1, downgrade old-chain prefix only | Server, honest/restarted | Keep the split registrable; do not install `FALLBACK_WON`; `S:944-974` | COVERED | `SC::downgrade_prefix_confirmation_does_not_refuse_registration` confirms only the prefix, then asserts the complete group still registers and the watch remains armed at `SC:3287-3376`. |
| N2, upgrade input FINAL confirmed | Server, honest/restarted | Resolve the unarmed watch terminally and refuse late registration; `S:492-509` | COVERED | `SC::channel_parent_exit_watch_lifecycle` binds `input_final_confirmed` to the actual final txid and leaves the new chain unregistered at `SC:1532-1559`; the offline restart reaches terminal before activation at `SC:1622-1672`. |
| N2, downgrade FINAL first, split unregistered | Server, honest/restarted | Install monotonic `FALLBACK_WON`, refuse recognition, authenticate the refusal; `S:944-974` | COVERED | `SC::downgrade_watch_response_and_fallback_tombstone` binds the actual final evidence to `FALLBACK_WON`, no recognized leaves, and the authenticated group digest at `SC:3098-3136`. |
| N2, downgrade FINAL after split registration | Server, honest/restarted | Broadcast the exact registered response and resolve `responded`; `S:907-924` | COVERED | `SC::downgrade_watch_response_and_fallback_tombstone` asserts the expected response txid wins; `SC::downgrade_split_admission_and_group_registration` asserts both leaves become spendable atomically and the watch arms at `SC:2764-2805`. |
| N2, downgrade FINAL after split registration | Client, server absent/adversarial | Respond with the retained split, ungated on connectivity; `S:925-938` | COVERED | `C::downgrade_watch_responds_by_exit` stops the server, starts the exact leaf exit, confirms the retained `response_txid`, and closes with `Downgraded` at `C:630-668`. |
| N2, downgrade FINAL first, cooperative fallback | Client, honest/restarted | Resolve through the retained closing transaction; `S:595-652,894-905` | PARTIAL | `C::channel_close_fallback_won_rides_cooperative_tail` binds terminal `commitment.0 == closing_txid`, but recovery is only `claimed_sat > 0` at `C:480-484`. See P3-1. |
| N3, bridge in mempool | Client, honest then absent/restarted | Relay the exact bridge ungated and retain exit progress; `S:1013-1037` | COVERED | `L::urgent_htlc_deadline_close_is_relayed` and `server_recovers_from_a_vanished_client` both wait for the exact funding txid in the mempool at `L:1963-1969,2098-2104`; `L::exit_crash_restart_ladder` restarts in `ChannelBridge` at `L:2471-2480`. |
| N3, bridge only in mempool | Server, client absent/self restarted | Publish nothing until `gettxout(..., include_mempool=false)` says the funding is canonical; publish once afterward; `R:74-99` | PARTIAL | `L::server_recovers_from_a_vanished_client` reaches this posture, but never asserts that no commitment/package was submitted before `L:2115-2122` confirms the funding. It also does not bind a single publish attempt. See P2-5. |
| N4, bridge confirmed, server absent | Client, honest/restarted | Publish own commitment and claim its outputs; `S:1013-1043,2316-2321` | PARTIAL | `C::channel_unilateral_exit_end_to_end` reaches Closed/Exited and a successful movement at `C:927-943`, but does not bind recovered value to the commitment output. `L::exit_crash_restart_ladder` likewise ends with only `> 0`. See P3-1. |
| N4, bridge confirmed, client absent, server balance | Server, honest | Stalled policy must force-close, relay, mature, and recover `to_local`; `S:1055-1071,2253-2261` | COVERED | `L::server_recovers_from_a_vanished_client` finds the actual funding spender, derives `to_local` from it, and requires a new rounds-wallet UTXO whose transaction spends only that outpoint and carries at least 90% of its actual value at `L:2125-2242`. |
| N4, bridge confirmed, urgent HTLC | Server, honest | Force-close at the HTLC deadline even with stalled-close automation disabled; `S:1103-1119,R:74-78` | COVERED | `L::urgent_htlc_deadline_close_is_relayed` sets `stalled_close_after_blocks=0`, derives the real HTLC expiry, and asserts the exact funding outpoint becomes spent after the urgent rung at `L:1988-2010`. HTLC recovery itself is a separate N6 cell. |
| N4, stalled timer mid-absence | Server, self crashed/restarted | Preserve block-height absence without forgetting or inventing it; `R:95-99` | DARK | `L::server_recovers_from_a_vanished_client` uses a five-block timer but never restarts the server before it fires. See P1-5. |
| N4, close package first relay fails | Server, self crashed/restarted | Retain and retry the captured commitment package after LDK removes the channel; `R:74-91` | DARK | No test faults the first relay or restarts between attempts. The success-only stalled and urgent vectors cannot detect a lost retry queue. See P1-6. |
| N4, operator/protocol-fault close | Server, honest | Once LDK is force-closed, use the same canonical-funding publication path; `R:74-78` | COVERED | The operator trigger is bound by `L::theirs_commitment_sweeps` obtaining a real funding-spending commitment at `L:1402-1410`; the shared post-LDK publication path is proven by the urgent and stalled vectors. The reason does not select different publication machinery. |

### N5–N6: commitments, HTLC resolution, and justice

| State / payload | Party + posture | Obligation | Verdict | Proving test and binding assertion |
|---|---|---|---|---|
| N5, client commitment, no HTLC, client balance | Client, honest/restarted | Sweep delayed `to_local` and reach terminal; `S:1038-1040` | PARTIAL | `C::channel_unilateral_exit_end_to_end` and `L::exit_crash_restart_ladder` reach the correct states, but their value assertions are not tied to the actual `to_local`. See P3-1. |
| N5, client commitment, no HTLC, server balance | Server, honest | Sweep static `to_remote` into the server wallet; `S:1038-1040` | COVERED | `L::server_sweeps_its_balance_after_client_exit` derives the exact `to_remote`, finds a new rounds-wallet UTXO whose transaction spends it, and requires at least 90% of the actual value at `L:2352-2448`. |
| N5, latest server commitment, no HTLC/all-client | Client vs adversarial server race | Select the foreign winner and sweep client `to_remote`; `S:1013-1040` | COVERED | `L::coop_close_adopts_foreign_commitment` asserts terminal uses the exact foreign txid and recovers at least 90% of the largest actual output at `L:2632-2657`. `RC::test_counterparty_commitment_with_zero_local_output` pins the zero-server-output structure. |
| N5, latest server commitment, server balance | Server, client absent | Sweep delayed server `to_local`; `S:1055-1071` | COVERED | `L::server_recovers_from_a_vanished_client`, `L:2164-2242`, binds the actual delayed output to a rounds-wallet descendant. |
| N5, revoked server commitment, no HTLC | Client, honest victim | Relay justice before the cheater delay and recover both balances; `S:2316-2321,R:13-48` | COVERED | `L::client_punishes_revoked_server_close` reaches `ChannelClaimed`, requires every non-anchor output spent, and requires client recovery greater than the largest actual output, which is impossible without the cheater’s `to_local`, at `L:1582-1624`. |
| N5, revoked client commitment, no HTLC | Server, honest victim | Relay server-side justice and recover both balances; `S:2316-2321,R:13-48` | DARK | HEAD has no server-punishes-revoked-client vector. See P1-1. |
| N5, revoked server commitment with HTLC | Client, honest victim | Punish both balance and HTLC outputs; `S:2316-2321,R:13-48` | PARTIAL | `L::client_punishes_revoked_server_close_with_htlc` proves an HTLC-bearing revoked commitment, but `recovered > largest` can pass when balance justice succeeds and HTLC justice is absent; “all outputs spent” can be satisfied by the cheater. See P3-2. |
| N5, revoked client commitment with HTLC | Server, honest victim | Punish balance and HTLC outputs; `S:2316-2321,R:13-48` | DARK | No HEAD vector exports/mines a revoked client commitment or binds server recovery. See P1-1. |
| N6, client commitment, client-offered HTLC | Server, success/preimage branch | Directly claim before client timeout and land the value; `S:1038-1040,1103-1119` | PARTIAL | `L::force_close_server_claims_and_payer_scrapes` is this class—not a server-own commitment. It derives the HTLC outpoint and proves a spend before the client timeout at `L:300-333`, but not the server-owned landing value. See P3-2. |
| N6, client commitment, client-offered HTLC | Client, own timeout branch | Confirm presigned timeout, sweep its child, and account the value; `S:1038-1040` | PARTIAL | `L::blocked_htlc_claim_surfaces_and_clears` proves the exact HTLC output is spent and the durable marker clears at `L:2794-2821`, but does not follow the second-stage output to the wallet or terminal. See P3-2. |
| N6, server commitment, client-offered HTLC | Client, direct timeout branch | Reclaim after expiry; `S:1038-1040` | DARK | No test stages this commitment/direction/claimant class. See P1-3. |
| N6, server commitment, client-offered HTLC | Server, own presigned success branch | Confirm success and sweep the child; `S:1038-1040` | DARK | `L::urgent_htlc_deadline_close_is_relayed` proves only the commitment publication. See P1-3. |
| N6, client commitment, server-offered HTLC | Client, own presigned success branch | Confirm success and sweep the child; `S:1038-1040` | DARK | No HEAD test. See P1-3. |
| N6, client commitment, server-offered HTLC | Server, direct timeout branch | Reclaim after expiry; `S:1038-1040` | DARK | No HEAD test. See P1-3. |
| N6, server commitment, server-offered HTLC | Client, direct success branch | Claim with preimage and account the HTLC; `S:1038-1040` | COVERED | `L::client_claims_htlc_off_server_commitment` derives the actual HTLC outpoint, reaches terminal, and requires recovery greater than the actual client `to_remote`, which only this HTLC can supply, at `L:1812-1898`. |
| N6, server commitment, server-offered HTLC | Server, own presigned timeout branch | Confirm timeout and sweep the child; `S:1038-1040` | DARK | No HEAD test. See P1-3. |
| N6, client commitment, offered + received | Client, honest | Resolve own timeout and success second stages together | DARK | `close_under_pending_payments` drains off-chain; no on-chain vector carries both directions. See P1-3. |
| N6, client commitment, offered + received | Server, honest | Resolve direct success and timeout claims together | DARK | No on-chain `both` vector. See P1-3. |
| N6, server commitment, offered + received | Client, honest | Resolve direct timeout and success claims together | DARK | Only the success half is tested. See P1-3. |
| N6, server commitment, offered + received | Server, honest | Resolve own success and timeout second stages together | DARK | No HEAD vector. See P1-3. |
| N6, revoked server commitment, cheater success child confirmed | Client, honest victim | Punish the child’s delayed output | DARK | `_with_htlc` never mines a cheater second-stage transaction. See P1-2. |
| N6, revoked server commitment, cheater timeout child confirmed | Client, honest victim | Punish the child’s delayed output | DARK | No HEAD vector. See P1-2. |
| N6, revoked client commitment, cheater success child confirmed | Server, honest victim | Punish the child’s delayed output | DARK | No HEAD vector. See P1-2. |
| N6, revoked client commitment, cheater timeout child confirmed | Server, honest victim | Punish the child’s delayed output | DARK | No HEAD vector. See P1-2. |

### N7–N10: closing, expiry, terminal accounting, and money-changing reorgs

| State / payload | Party + posture | Obligation | Verdict | Proving test and binding assertion |
|---|---|---|---|---|
| N7, normal close, all-client balance | Client, honest/restarted | Register the exact split and close without a live old-scope exit; `S:549-652,721-831` | COVERED | `C::channel_close_settles_offchain` asserts exact pre-fee balances, a 400,000-sat owned leaf, total spendable balance, and no live exit at `C:346-377`. |
| N7, normal close, both balances | Client, honest | Preserve the payment-adjusted exact balance | COVERED | `P::collect_pay_and_close_composition` binds payment movement to exact post-close balances at `P:98-143`. |
| N7, normal close, sub-dust server side | Client, honest | Preserve the exact short side while returning the user leaf | COVERED | `L::sub_dust_close_side_still_settles` asserts the exact 400,000 msat server balance and exact client leaf amount at `L:3080-3091`. |
| N7, normal close, both/odd-sat/sub-dust | Server, honest/restarted | Record exact pre-fee balances and register both branches atomically; `S:721-855` | COVERED | `SC::downgrade_split_admission_and_group_registration` derives the floor-plus-user-remainder, asserts exact client/server leaf amounts, and makes both spendable atomically at `SC:2609-2729,2764-2805`. |
| N7, closing outcome recorded | Server, self restarted | Retain backing, funding outpoint, balances, closing txid, and watch after LDK forgets the channel; `S:613-652` | COVERED | `SC::cooperative_close_outcome_recorded` binds all fields to the actual closing transaction and re-reads them after restart at `SC:2036-2117`. |
| N7, output-less closing artifact | Client, honest | Settle using pre-fee split balances despite unusable fallback artifact | COVERED | `C::channel_close_zero_output_artifact_still_settles` asserts empty-artifact economics do not change the exact split balance at `C:498-522`. |
| N7, output-less closing artifact | Server, honest | Record the pre-fee outcome without wedging the event queue | COVERED | `SC::cooperative_close_zero_output_recorded` asserts the actual closing transaction is output-less while the retained outcome remains exact at `SC:2168-2195`. |
| N7, closing pending, foreign commitment wins before Swept | Client vs adversarial server | Reselect the actual funding winner | COVERED | `L::coop_close_adopts_foreign_commitment`, `L:2610-2657`, binds the exact winner txid and actual recovered output value. |
| N7/N9/N10, closing reached `ChannelSwept`, then alternative commitment wins after reorg | Client, honest/restarted | Demote, reselect, and claim the new winner | DARK | `C::channel_exit_recovers_from_commitment_reorg` may reach Swept but re-confirms the same commitment and never supplies an alternative winner or value assertion. See P1-7. |
| N8, pre-bridge expiry, split unregistered | Server, client absent | Use the expiry leaf at `max(bridge_mature, expiry)` and recover server-owned value; `S:238-241,1045-1053` | PARTIAL | `WP::channel_funding_*` pins `Wait` and exact `Claim` deadlines at `WP:939-991`; `C::channel_expiry_race_server_claims` proves the tree anchor disappears, and `SC:1738-1773` proves terminal foreclosure, but neither binds recovered value to the server. See P3-3. |
| N8, client returns after losing expiry race | Client, honest | Never fabricate Swept/Claimed or closed accounting | COVERED | `C::channel_expiry_race_server_claims` checks every live state and append-only history for forbidden terminals and requires the record remain Exiting at `C:1260-1307`. |
| N8, registered split, old chain actualized | Server, honest | Choose `Progress{bridge_mature−1}` and confirm the exact response | COVERED | `WP::channel_funding_progresses_a_registered_split` asserts the exact txid and deadline at `WP:998-1022`; `SC::downgrade_watch_response_and_fallback_tombstone` confirms that response. |
| N8, registered split, old chain actualized | Client, server absent | Exit the exact owned split leaf | COVERED | `C::downgrade_watch_responds_by_exit`, `C:630-668`, binds the retained response txid and terminal close resolution. |
| N9, client own-commitment sweeps and restart ladder | Client, self restarted | Resume Commitment and Swept states and recover actual value | PARTIAL | State reachability is strong: `L::exit_crash_restart_ladder` asserts pre-terminal Commitment and Swept before each restart at `L:2482-2526`; value is only `recovered > 0` and `claimed_sat > 0` at `L:2528-2547`. See P3-1. |
| N9, server static/delayed outputs | Server, honest | Persist descriptors and sweep into its rounds wallet | COVERED | `L::server_sweeps_its_balance_after_client_exit` and `server_recovers_from_a_vanished_client` both bind actual commitment outpoints to confirmed rounds-wallet descendants and actual values at `L:2196-2242,2402-2448`. |
| N9, V63 server claim lock after restart | Server, self restarted | Re-arm wallet input locks so a round cannot steal claim funding | COVERED | `L::force_close_server_claims_and_payer_scrapes` restarts before the HTLC claim, runs a concurrent round, and still confirms the exact claim outpoint at `L:307-355`. Disabling lock re-arm makes the claim double-spend and timeout. |
| N9, V64 sweep row after persist/before broadcast or RBF | Server, self restarted | Resume, fee-bump, track superseded txids, and preserve the descriptor | DARK | Happy-path provenance vectors never restart or inspect a V64 row. No test mentions `superseded_txids`. See P1-4. |
| N9, closed/tombstoned channel, stale manager | Server, adversarial stale DB/restarted | Never resurrect the live channel | COVERED | `SC::stale_manager_cannot_resurrect_a_closed_channel` binds the retained terminal row against the stale manager at `SC:2369-2466`; `SD::channel_downgrade_group_records` pins monotonic registration/fallback exclusion. |
| N10, downgrade tombstone reorg | Server, honest/restarted | Reopen the watch but never reopen off-chain recognition | COVERED | `SC::downgrade_watch_response_and_fallback_tombstone` keeps `FALLBACK_WON` after invalidation at `SC:3254-3280`; `SD:903-913` independently asserts watch reopening does not clear the tombstone. |
| N10, server sweep confirmation reorg before depth 100 | Server, self restarted | Restore/rebroadcast the V64 obligation rather than retire it | DARK | No V64 reorg test. See P1-4. |

### Timelock and invariant cells

| State / payload | Party + posture | Obligation | Verdict | Proving test and binding assertion |
|---|---|---|---|---|
| N0, runway exactly `F` / `F+1` | Client vs adversarial server | Independently refuse unless runway is greater than `F`; `S:441-463` | DARK | No client test ages the selected input to this boundary. Server refusal would mask a missing client guard. See P1-8. |
| N0, runway exactly `F` / `F+1` | Server vs adversarial client | Refuse at `F`, accept at `F+1`; `S:441-463` | PARTIAL | `SC::open_by_upgrade_admission` proves the guard at runway 30 with floor 34 (`SC:558-587`), but not its exact boundary. See P2-2. |
| N0, minimum funding and split headroom | Client vs adversarial server | Independently enforce 660 sats, depth headroom, and pinned exit delta; `S:441-471,781-823,2218-2230` | PARTIAL | `C::open_channel_refusals` reaches the 659-sat refusal and `C::channel_reopen_ladder_ends_at_the_open_headroom_guard` reaches the headroom refusal, but neither proves the refusal occurred before a server RPC; no mismatch test exists for `pinned_exit_delta`. See P2-1. |
| N0/N7, minimum funding/headroom | Server vs adversarial client | Refuse 659 sats, accept the sanctioned 660-sat shape, enforce open and split headroom | COVERED | `SC::downgrade_split_admission_and_group_registration` rejects the 659-sat open at `SC:2855-2887`, admits the 660-sat shape, and rejects missing split headroom at `SC:2687-2703`; open headroom refusal is also bound at `SC:219-227`. |
| N0, client exit-fee reserve absent | Client, honest/adversarial server | Refuse an open that confirmed coins cannot fee-bump; price the estimate at `fast`; `S:178-185,R:108-119` | DARK | `L::exit_funding_reports_an_unaffordable_exit` opens while funded and only proves the on-read status flips after a later drain. It does not exercise open refusal or the `fast` source. See P1-9. |
| N0, exit-funding status after wallet drain | Client, honest | Recompute affordability on read | COVERED | `L::exit_funding_reports_an_unaffordable_exit` asserts a positive funded estimate, drains confirmed coins, then requires `fundable == false` at `L:2276-2314`. |
| N0, channels plus `daemon_manual_sync` | Client, self configured/restarted | Refuse startup because chain-safety duties would never tick; `R:121-122` | DARK | Existing manual-sync tests run without Ark channels. See P1-10. |
| Operating, close-lead/hard-line ordering | Client, honest/server absent | Prefer cooperative close at `hard+close_lead`, then exit at hard line; prohibit cancel/resurrection | COVERED | `C::channel_deadline_prefers_cooperative_close` mines exactly to the cooperative threshold and asserts no exit at `C:533-553`; `C::close_inside_hard_line_never_crosses_ponr` and `manual_close_and_cancel_refused_inside_hard_line` bind the hard fallback state at `C:750-861`; `L::deadline_close_then_reopen` binds the treadmill. |
| Operating, HTLC deadline at exactly `F` | Client, peer absent | Force an uncancelable exit no later than exactly `F` remaining; `S:1103-1119` | PARTIAL | `L::htlc_deadline_forces_uncancelable_exit` proves the rung and cause, but mines in six-block jumps at `L:1111-1132`; an off-by-one or five-block-late trigger survives. See P2-4. |
| Receiver HTLC at `F` / `F−1` | Client vs adversarial sender | Accept at `F`, refuse below it and on absent/overflow deadline | COVERED | `bark/src/channels/mod.rs::claim_floor_boundary_is_exact` asserts `F`, `F−1`, absent, and overflow at lines 5046-5059; `P::keysend_at_floor_claims` supplies the public at-floor path at `P:323-349`. |
| Receiver HTLC at `F` / `F−1` | Server vs adversarial client | Enforce the live channel’s pinned floor | DARK | No server event test injects an exact final CLTV below its pinned floor. See P1-11. |
| Forwarded HTLC differential `F_in` / `F_in−1` | Server vs adversarial route | Reject an unsafe incoming-outgoing differential | DARK | Happy forwarding tests do not control or inspect the exact CLTV differential. See P1-11. |
| Floor carrier overflow | Client | Checked `u16` refusal | COVERED | `bark/src/channels/payments.rs::cltv_floor_is_the_pinned_trio_sum` asserts the trio sum and overflow refusal at lines 681-684. |
| Live server floor profile changed/below worst floor | Server, self restarted/misconfigured | Fail startup; never weaken existing channel floors; `S:1089-1119` | DARK | The guards at `server/src/channels/mod.rs:208-242` have no test. See P1-11. |
| N8, ChannelFunding before/at expiry and late bridge | Server | `Wait`, then `Claim{max(bridge_mature,expiry)}` | COVERED | `WP::channel_funding_waits_before_expiry`, `claims_at_expiry_with_mature_bridge`, and `claim_deadline_is_bridge_maturity_after_late_exit` assert exact actions/deadlines at `WP:939-991`. |
| N8, registered split response | Server | `Progress{bridge_mature−1}`; unsigned split remains expiry-governed | COVERED | `WP::channel_funding_progresses_a_registered_split` and `never_progresses_an_unsigned_split`, `WP:998-1043`. |
| N7, downgrade within six-block expiry band | Server vs client racing expiry | Refuse at both cosign and final registration; `S:869-974` | DARK | `DOWNGRADE_EXPIRY_MARGIN` has no test at either gate. See P1-12. |
| N9, client terminal at depth 99/100 | Client, honest/restarted | Stay Swept through 99 and become Claimed only at 100 | PARTIAL | Many e2e tests eventually reach `ChannelClaimed`, but none assert the 99/100 boundary. See P2-6. |
| N9, server sweep retirement at depth 99/100 | Server, honest/restarted | Retain V64 through 99; retire at 100 | DARK | No test reaches the V64 retirement boundary. See P1-4. |
| Bridge CSV versus commitment | Both parties | Bridge `nSequence=pinned_exit_delta`; commitment adds no Ark delay | COVERED | `lib/src/channel.rs::test_bridge_tx_shape_and_vector` asserts the bridge input, funding output, P2A, and sequence; `RC::test_force_close_bump_and_htlc_resolution` binds immediate commitment spend and BOLT second stages. |
| Generic round consumption of ChannelFunding | Server vs adversarial client | Reject ChannelFunding as a round input | COVERED | `server/src/round/mod.rs::test_register_payment_channel_funding_input_rejected` asserts the policy refusal at lines 2131-2142. |

No in-scope safety decision is justified from mempool contents. Mempool inspection in the tests above is staging only; the one insufficiently proven condition is the negative, chain-only funding-canonical gate identified as P2-5.

## 2. Findings, ranked

### P1-1 — Server-side justice is dark for revoked client commitments, with and without HTLCs

- **Cell:** N5, client commitment is revoked; server is the honest victim; payload none or HTLC.
- **Reachability:** Open normally, move balance or commit an HTLC, preserve an old client holder commitment, advance the channel so it becomes revoked, actualize the bridge, then mine that old commitment.
- **Consequence:** If server monitor relay or justice signing is missing, the client waits out its CSV and takes an obsolete balance; with an HTLC it may also take the revoked HTLC value. The server loses actual channel funds.
- **Nearest evidence:** `L::client_punishes_revoked_server_close[_with_htlc]` proves only the opposite victim role. `L::server_sweeps_its_balance_after_client_exit` uses a latest client commitment, so no justice is required.
- **Proposed vectors:** `server_punishes_revoked_client_close` and `server_punishes_revoked_client_close_with_htlc` in `testing/tests/bark_sdk/lifecycle.rs`.
- **Seams:** **NEW** test-only client holder-commitment export seam, symmetric with server `channel_export_commitment`. It is necessary because HEAD exposes no old fully signed client commitment.
- **Assertions:** Derive the revoked balance and HTLC outpoints from the mined transaction; require server justice transactions to spend those exact outpoints before the client CSV path; require confirmed rounds-wallet descendants carrying the corresponding actual values less measured fees; require the server channel/sweep rows to reach terminal.
- **Runtime:** 600–900 seconds each.

### P1-2 — Justice after the cheater confirms an HTLC second stage is dark in both victim roles

- **Cell:** N6, revoked commitment plus a confirmed cheater HTLC-success or HTLC-timeout child; victim client or server.
- **Reachability:** The preceding P1-1 setup, but mine the cheater’s valid second-stage transaction before allowing victim justice to run.
- **Consequence:** Parent-output justice now conflicts with a confirmed child. If the monitor does not follow and punish the child’s delayed output, the cheater takes the HTLC after its delay.
- **Nearest evidence:** `L::client_punishes_revoked_server_close_with_htlc` mines only the revoked commitment. It lets justice spend the commitment’s HTLC output directly and never exercises child tracking.
- **Proposed vectors:** `client_punishes_revoked_server_htlc_second_stage_{success,timeout}` and `server_punishes_revoked_client_htlc_second_stage_{success,timeout}` in `lifecycle.rs`.
- **Seams:** **NEW** test-only export of the fully signed old HTLC-resolution child, for both holder roles. Direct `generateblock` already exists.
- **Assertions:** Mine the revoked commitment and chosen child; derive the child’s delayed output; require a victim-owned justice transaction to spend that exact child outpoint before the cheater delay; trace the actual value into the victim wallet; require terminal state.
- **Runtime:** 600–900 seconds per party/direction vector.

### P1-3 — Five latest-state HTLC claim classes and every bidirectional on-chain payload are dark

- **Cells:** Client direct timeout and server own success off a server commitment; client own success and server direct timeout off a client commitment; server own timeout off a server commitment; and all four claimant/commitment combinations with offered and received HTLCs together.
- **Reachability:** Existing payment hold seams can commit each direction. Existing client exit and server commitment-export/direct-mining seams select either commitment class.
- **Consequence:** An eligible claimant can fail to publish or finish its branch, transferring the HTLC to the counterparty or leaving a second-stage output indefinitely locked.
- **Nearest evidence:** `force_close_server_claims_and_payer_scrapes` covers server direct success off the **client** commitment; `client_claims_htlc_off_server_commitment` covers client direct success off the server commitment; `blocked_htlc_claim_surfaces_and_clears` reaches only the first stage of client timeout. None proves the dark classes or coexistence.
- **Proposed vectors:** `client_commitment_client_resolves_bidirectional_htlcs`, `client_commitment_server_resolves_bidirectional_htlcs`, `server_commitment_client_resolves_bidirectional_htlcs`, and `server_commitment_server_resolves_bidirectional_htlcs` in `lifecycle.rs`.
- **Seams:** Existing claim-binding/pass holds, pending-expiry reader, commitment export, client exit, and direct mining; no new product seam.
- **Assertions:** Put one success-entitled and one timeout-entitled HTLC on the selected commitment. For each, bind the resolution transaction to the exact commitment outpoint, follow any second-stage output into the entitled wallet, compare against transaction-derived value, and require channel/payment terminal states.
- **Runtime:** 600–900 seconds each.

### P1-4 — Server V64 sweep persistence, RBF, reorg, and deep retirement are dark

- **Cell:** N9/N10, a real `channel_spendable_output` exists and the server crashes after persistence, replaces an attempt, sees a superseded candidate win, or loses a confirmation before depth 100.
- **Reachability:** Both strong server-balance e2e tests already create real static or delayed descriptors.
- **Consequence:** A descriptor or replacement can be forgotten, retired at one confirmation, or left pointing at an orphan. Server funds remain locked or are lost.
- **Nearest evidence:** `server_sweeps_its_balance_after_client_exit` and `server_recovers_from_a_vanished_client` prove one happy attempt. `FB::test_feed_ack_holds_for_matured_spendable_output` proves persist-before-ack, but no test reloads or mutates a V64 attempt row.
- **Proposed vectors:** `server_channel_sweep_survives_restart_before_broadcast` and `server_channel_sweep_rbf_reorg_retires_at_depth_100` in `lifecycle.rs`.
- **Seams:** **NEW** server test hold after `set_channel_sweep_attempt` and before broadcast, plus deterministic fee-rise/first-broadcast-failure control. This is the otherwise unobservable durable boundary.
- **Assertions:** Bind the stored descriptor to the actual commitment outpoint; after restart require broadcast of a transaction spending it; require replacement to spend the same outpoint and record the old txid as superseded; invalidate the winning block and require row reopening/rebroadcast; assert retention at depth 99, retirement at 100, and a rounds-wallet descendant carrying actual value.
- **Runtime:** 600–900 seconds each.

### P1-5 — The stalled-close absence timer is dark across server restart

- **Cell:** N4, bridge confirmed, client absent, server has balance; restart at `stalled_close_after_blocks−1`.
- **Reachability:** `server_recovers_from_a_vanished_client` already uses a five-block compressed policy and a vanished client.
- **Consequence:** Forgetting absence on each restart can defer recovery indefinitely; inventing absence can force-close a healthy channel.
- **Nearest evidence:** The existing vector never restarts before the timer fires.
- **Proposed vector:** `stalled_close_timer_survives_restart_without_inventing_absence` in `lifecycle.rs`.
- **Seams:** Existing compressed configuration, server restart, and bridge staging; no new seam.
- **Assertions:** At threshold−1 restart and assert funding remains unspent; mine one block and require the exact server commitment and provenance-bound recovery. A connected control channel must remain unspent through the same restart/blocks.
- **Runtime:** 600–900 seconds.

### P1-6 — Server close-package retry after a failed relay and restart is dark

- **Cell:** N4, LDK has force-closed after funding actualization; first commitment-package submission fails; server restarts.
- **Reachability:** Use either WD-16 or the urgent HTLC setup, then fault the first submission.
- **Consequence:** LDK removes the channel from its live list immediately. If the durable candidate/retry path is wrong, the commitment and HTLC claims are stranded permanently.
- **Nearest evidence:** The stalled and urgent vectors exercise only successful first relay.
- **Proposed vector:** `server_force_close_package_retries_after_restart` in `lifecycle.rs`.
- **Seams:** **NEW** fail/hold-next package-submission seam at the chain broadcaster.
- **Assertions:** Require the first failure, a durable candidate tied to the exact funding outpoint, restart, eventual exact funding spender, and provenance-bound recovery of server balance and any HTLC.
- **Runtime:** 600–900 seconds.

### P1-7 — Alternative-winner reselect after `ChannelSwept` is dark

- **Cell:** N7→N9→N10, cooperative closing/sweep confirmed, reorg removes them, and a server commitment becomes the funding winner.
- **Reachability:** Existing close, server commitment export, `invalidateblock`, and direct-mining seams suffice.
- **Consequence:** Without `ChannelSwept` revalidation/demotion, the client can terminalize an orphaned sweep and never claim its real foreign-commitment output.
- **Nearest evidence:** `coop_close_adopts_foreign_commitment` changes winner before Swept. `channel_exit_recovers_from_commitment_reorg` permits Swept but only reconfirms the same commitment and asserts no value.
- **Proposed vector:** `channel_swept_reorg_reselects_foreign_winner` in `lifecycle.rs`.
- **Seams:** Existing seams only.
- **Assertions:** Require an exact precondition of `ChannelSwept` over the closing tx; invalidate its block and descendants; mine the exported foreign commitment; require a recorded demotion, terminal selection of the exact foreign txid, and wallet recovery of its actual client output.
- **Runtime:** 600–900 seconds.

### P1-8 — Independent client runway admission is dark

- **Cell:** N0, selected input has runway `F`; server is adversarial and willing to cosign.
- **Reachability:** Age a real boarded VTXO to `F` and interpose the existing Ark RPC proxy.
- **Consequence:** With the client guard disabled, the channel opens already inside its exit discipline; the server expiry sweep can take the whole VTXO before the bridge confirms.
- **Nearest evidence:** `SC::open_by_upgrade_admission` proves only the server guard. No client test reaches the boundary.
- **Proposed vector:** `client_open_refuses_runway_at_f_and_accepts_f_plus_one` in `channel_proxy.rs`.
- **Seams:** Existing `ArkRpcProxy`; add only a test-local request counter.
- **Assertions:** At `F`, assert the client refusal and zero upgrade RPCs. At `F+1`, assert an RPC occurs and the open reaches Ready. The zero-RPC assertion prevents the honest server from masking a missing client guard.
- **Runtime:** 200–600 seconds.

### P1-9 — Client open-time exit-reserve refusal is dark

- **Cell:** N0, client lacks confirmed fee coins required for its exit chain.
- **Reachability:** Configure an explicit reserve or drain confirmed on-chain coins before calling `open_channel`.
- **Consequence:** The client can open a channel it cannot CPFP; at the deadline the bridge stalls and the expiry sweep takes the whole VTXO.
- **Nearest evidence:** `exit_funding_reports_an_unaffordable_exit` tests only the status surface after an already-open channel is drained.
- **Proposed vector:** `client_open_refuses_unfunded_exit_reserve` in `channels.rs`, plus `exit_reserve_uses_fast_confirmed_balance` at the reserve-calculation layer.
- **Seams:** Existing wallet drain and RPC proxy. A deterministic fee-source test double is **NEW** only for distinguishing `fast` from `regular`.
- **Assertions:** Assert refusal before any upgrade RPC with insufficient confirmed balance; fund the exact computed reserve and assert admission. The calculation test must give `fast` and `regular` different values and require the former.
- **Runtime:** 200–600 seconds integration; under 30 seconds for the calculation test.

### P1-10 — Channels under manual-sync mode have no coverage for the mandatory startup refusal

- **Cell:** N0–N8, channels configured while `daemon_manual_sync=true`.
- **Reachability:** Ordinary barkd configuration; no test seam is needed.
- **Consequence:** If the guard regresses, watch, claim, and deadline duties never tick; the server can take the whole channel at expiry.
- **Nearest evidence:** Existing manual-sync vectors do not enable Ark channels.
- **Proposed vector:** `channels_refuse_manual_sync_daemon` in `testing/tests/barkd/channels.rs`.
- **Seams:** None.
- **Assertions:** Starting barkd with both settings must fail with the specific incompatibility before opening/listening; the normal-channel configuration must start.
- **Runtime:** Under 200 seconds.

### P1-11 — Server receiver/forwarder floors and live-profile immutability are dark

- **Cells:** Receiver final CLTV `F−1`; forwarder differential `F_in−1`; startup with profile below the worst floor; restart with a profile different from live channels.
- **Reachability:** A hostile payer/router supplies the short CLTV; an operator changes the profile across restart.
- **Consequence:** The server can accept or forward an HTLC whose upstream claim cannot become possible before expiry, or silently weaken protection on live channels, losing the forwarded amount.
- **Nearest evidence:** `keysend_at_floor_claims` proves the Bark receiver path. Ordinary forwarding and quarantine tests neither inject exact server CLTVs nor exercise the startup guards.
- **Proposed vectors:** `server_receiver_and_forwarder_reject_floor_minus_one` in `testing/tests/server/channels.rs` and `server_refuses_changed_live_htlc_floor_profile` in the server startup suite.
- **Seams:** **NEW** exact-CLTV injection below LDK’s random shadow padding; justified because the public sender cannot deterministically create `F−1`. No new seam for configuration restart.
- **Assertions:** At `F−1`, require refusal before claim/forward and eventual payer refund with no outbound HTLC; at `F`, require settlement. Startup tests must fail on below-worst and changed-live profiles and succeed on the original exact profile.
- **Runtime:** 200–600 seconds for CLTV integration; under 200 seconds for startup cases.

### P1-12 — `DOWNGRADE_EXPIRY_MARGIN` is dark at cosign and registration

- **Cell:** N7/N8, split cosign or final registration with six blocks remaining.
- **Reachability:** Seed a real completed close, age its backing, then invoke the existing direct cosign and registration paths.
- **Consequence:** Recognized leaves can be installed over a backing whose ancestor expiry sweep has already become confirmable; neither the response nor rollback repairs that recognition.
- **Nearest evidence:** Split admission and expiry tests never enter the six-block band.
- **Proposed vector:** `downgrade_expiry_margin_refuses_cosign_and_registration` in `testing/tests/server/channels.rs`.
- **Seams:** Existing direct RPC, held registration, and block generation.
- **Assertions:** With seven blocks remaining, cosign/register must succeed; with six, each gate must refuse, leave no new spent mark or recognized leaf, and retain the unilateral fallback.
- **Runtime:** 200–600 seconds.

### P2-1 — Client minimum-funding, headroom, and pinned-delta refusals are not isolated from server enforcement

- **Cell:** N0 client admission with 659 sats, excessive exit depth, or mismatched advertised/input `exit_delta`.
- **Reachability:** Existing open-refusal and reopen-ladder flows; mutate ArkInfo for the delta case.
- **Consequence:** A malicious server can lead the client into a channel with no cooperative settlement or a mismatched profile. Unilateral exit remains available, so this is bounded rather than a dark custody cell.
- **Nearest evidence:** Current tests assert an error but not that no server request occurred; there is no delta mismatch vector.
- **Proposed vector:** `client_open_admission_refuses_before_rpc` in `channel_proxy.rs`.
- **Seams:** Existing proxy with test-local request counter/ArkInfo mutation.
- **Assertions:** Each invalid case must fail with zero upgrade RPCs; the exact valid boundary must issue the RPC and open.
- **Runtime:** 200–600 seconds.

### P2-2 — Server runway’s exact `F`/`F+1` boundary is unpinned

- **Cell:** N0 server admission at remaining runway `F` and `F+1`.
- **Reachability:** The existing aged-scope admission test.
- **Consequence:** An off-by-one can reject one safe height or admit one unsafe height; the broad guard itself is proven.
- **Nearest evidence:** `SC::open_by_upgrade_admission` tests only 30 against floor 34.
- **Proposed vector:** Extend `open_by_upgrade_admission` with exact `F` refusal and `F+1` acceptance.
- **Seams:** Existing.
- **Assertions:** Bind daemon tip, decoded expiry, computed floor, refusal at equality, and successful cosign one block earlier.
- **Runtime:** Existing 200–600-second vector; small incremental cost.

### P2-3 — Upgrade prefix non-foreclosure is unproven

- **Cell:** N1, only an upgrade input’s old-scope prefix is confirmed.
- **Reachability:** Use a checkpointed/multi-level input, mine all but its final transaction, then retry registration.
- **Consequence:** A premature terminal decision can abort a valid open and force recovery through the old input; no permanent loss is required.
- **Nearest evidence:** The parent-watch test mines through the final; only the downgrade has a prefix-specific vector.
- **Proposed vector:** `upgrade_prefix_confirmation_does_not_refuse_registration` in `testing/tests/server/channels.rs`.
- **Seams:** Existing direct mining and registration.
- **Assertions:** Assert the watch unresolved after the prefix, registration succeeds, the retained response remains armed, and no terminal resolution was written.
- **Runtime:** 200–600 seconds.

### P2-4 — The client HTLC-deadline rung is not boundary-exact

- **Cell:** Operating channel with an unresolved claimed HTLC at `F+1` and then `F`.
- **Reachability:** Existing `htlc_deadline_forces_uncancelable_exit` setup.
- **Consequence:** The implementation could close several blocks late while the current six-block sampling still passes.
- **Nearest evidence:** The vector proves cause and uncancelability but advances six blocks per check.
- **Proposed vector:** Tighten the same vector.
- **Seams:** Existing pending-expiry reader.
- **Assertions:** At `expiry−F−1`, require Ready and no exit entry; mine one block, run one maintenance pass, then require Exiting with the durable HTLC cause and refused cancellation.
- **Runtime:** Existing 200–600-second vector.

### P2-5 — Funding-canonical and publish-once behavior is not negatively bound

- **Cell:** N3→N4, server has already force-closed internally while the bridge is only in the mempool.
- **Reachability:** The existing urgent and vanished-client setups.
- **Consequence:** Publishing early can lock rounds-wallet coins behind an invalid package; repeated publication can churn funding and obscure retries.
- **Nearest evidence:** WD-16 confirms the bridge before observing the funding spend, but never asserts absence of the commitment/package beforehand.
- **Proposed vector:** Strengthen `server_recovers_from_a_vanished_client` as `server_close_waits_for_canonical_funding_and_publishes_once`.
- **Seams:** Existing mempool inspection and compressed stalled policy.
- **Assertions:** While only the bridge is in the mempool, run close passes/restart and require no commitment or anchor child in the mempool and no package coin lock. After confirmation, require one distinct funding-spending commitment candidate and successful recovery.
- **Runtime:** 600–900 seconds.

### P2-6 — Client terminal depth 99/100 is unpinned

- **Cell:** N9, all client sweeps confirmed at depth 99, then 100.
- **Reachability:** Existing unilateral exit.
- **Consequence:** Early terminal accounting can survive current tests yet become false after a money-changing reorg.
- **Nearest evidence:** Tests mine until `ChannelClaimed` without observing the preceding exact depth.
- **Proposed vector:** `client_channel_claimed_only_at_depth_100` in `channels.rs`.
- **Seams:** Existing block generation and exit-state inspection.
- **Assertions:** At depth 99 require `ChannelSwept`, nonterminal record, and no successful movement; one block later require `ChannelClaimed`, Closed/Exited, and exact output provenance.
- **Runtime:** 600–900 seconds.

### P3-1 — Seven client recovery vectors contain state-only or `> 0` illicit-pass assertions

- **Cells:** N5/N7/N9 client recovery through own commitment, foreign commitment, cooperative fallback, restart, reorg, and split-over-tombstone.
- **Reachability:** All are already reached through sanctioned flows.
- **Consequence:** A CPFP change output, tiny unrelated receipt, or accounting-only terminal can satisfy the assertion while the channel output remains unclaimed.
- **Nearest evidence and proposed strengthening:**

| Existing vector | Weak assertion | Required replacement |
|---|---|---|
| `C::channel_unilateral_exit_end_to_end` | `MovementStatus::Successful` at `C:939-943` | Derive the actual client commitment output, require a wallet-owned descendant spending it, compare recovered value with the actual output less measured fees, then retain the terminal assertions. |
| `L::theirs_commitment_sweeps` | `claimed.claimed_sat > 0` at `L:1462` | Derive the client `to_remote` from `theirs`; require its exact outpoint in the sweep ancestry and at least the actual output less fee in the wallet. |
| `L::exit_crash_restart_ladder` | `recovered > 0` and `claimed_sat > 0` at `L:2540-2546` | Bind all expected client-owned commitment outputs to wallet descendants across the restarts and compare with their transaction-derived sum. |
| `C::channel_close_fallback_won_rides_cooperative_tail` | `claimed_sat > 0` at `C:484` | Locate the actual client shutdown output of `closing_txid`, require the exact outpoint’s wallet descendant, and compare actual value. |
| `L::peer_cooperative_close_adopted` | Exact closing txid only at `L:431-434` | Add shutdown-output provenance and value before accepting `ChannelClaimed`. |
| `C::channel_exit_recovers_from_commitment_reorg` | New block/state and terminal only at `C:1005-1021` | Require the re-confirmed commitment’s exact client output to reach the wallet on the new branch. |
| `L::chain_overrules_tombstone_installs_leaves` | Record merely becomes Downgraded at `L:3033-3051` | Bind the retained signed user leaf IDs and amounts to the wallet’s spendable/exited VTXOs; exclude the server leaf. |

- **Proposed vectors:** Strengthen the existing vectors under their current names.
- **Seams:** Existing transaction and wallet inspection only.
- **Runtime:** No new vector class; existing 200–900-second runs.

### P3-2 — Three HTLC/justice vectors stop before provenance-bound final value

- **Cells:** Server direct success off client commitment; client justice on an HTLC-bearing revoked server commitment; client timeout second-stage.
- **Consequence:** An opposing spend or an unswept second-stage output can satisfy the present assertion.
- **Nearest evidence and proposed strengthening:**

| Existing vector | Weak assertion | Required replacement |
|---|---|---|
| `L::force_close_server_claims_and_payer_scrapes` | Exact HTLC outpoint merely becomes spent at `L:318-333` | Locate its spender, prove the preimage branch, follow any child, and require a server rounds-wallet descendant carrying the actual HTLC value less fees. |
| `L::client_punishes_revoked_server_close_with_htlc` | Every output spent plus `recovered > largest` at `L:1742-1773` | Identify the HTLC outpoint separately; require a client justice input spending it, and compare client recovery against the actual client balance + cheater balance + HTLC outputs less fees. |
| `L::blocked_htlc_claim_surfaces_and_clears` | Marker clears and first-stage HTLC outpoint is spent at `L:2794-2821` | Parse that spender, derive its delayed child, mature and sweep the child into the client wallet, then require `ChannelClaimed`. |

- **Proposed vectors:** Strengthen the existing vectors.
- **Seams:** Existing.
- **Runtime:** Existing 600–900-second class.

### P3-3 — Server expiry recovery is asserted as disappearance, not server-owned value

- **Cell:** N8, client dark, pre-bridge expiry sweep.
- **Reachability:** Already reached by `channel_expiry_race_server_claims` and the expiry arm of `channel_parent_exit_watch_lifecycle`.
- **Consequence:** A wrong-value or wrong-destination claim can spend the observed anchor and terminalize the row while the server does not recover the channel value.
- **Nearest evidence:** `C:1247-1257` checks only that the anchor disappears; `SC:1746-1773` checks only that the row becomes terminal. Generic watchman vectors assert `ClaimBroadcast` construction, not confirmed server provenance.
- **Proposed vector:** `channel_expiry_sweep_pays_server_by_provenance` in `testing/tests/server/channels.rs`.
- **Seams:** Existing channel expiry setup and wallet/transaction RPCs.
- **Assertions:** Trace the actual expired ChannelFunding tree into the confirmed claim, require a server-wallet descendant, compare with the transaction-derived reclaimable value less fees, and require the channel row terminal.
- **Runtime:** 200–600 seconds.

## 3. Adequately covered

- Canonical bridge construction, corrupted partial rejection, zero-fee/P2A shape, and independence of bridge CSV from commitment delay are strongly pinned.
- Virtual-funding release is gated on durable registration, and reload watches every monitor; the matured-spendable-output feed barrier is persist-before-ack.
- Open action restart, lossy response retry, duplicate cosign/registration, and stable synthetic SCID behavior are strongly bound.
- Upgrade FINAL handling, exact retained response, late-registration refusal, and offline startup catch-up are strongly covered.
- Downgrade shape, exact per-key totals, odd-satoshi assignment, sub-dust isolation, group-atomic registration, and response txids are strongly covered.
- Cooperative-close outcome capture and retention bind backing, funding outpoint, exact pre-fee balances, and closing txid across server restart.
- All-client, payment-adjusted, sub-dust, and output-less cooperative settlements have strong exact-balance assertions.
- Close PONR restart, lost group-registration response, hard-line refusal, cooperative preference, and uncancelable exit causes are strongly covered apart from the exact HTLC threshold.
- Client receiving-floor acceptance/refusal and checked `u16` arithmetic are pinned at the proper unit/e2e layers.
- ChannelFunding watchman `Wait`, `Claim`, `Progress`, and unsigned-split actions have exact deadline unit coverage.
- Client downgrade response remains live with the server absent and confirms the exact retained response.
- WD-16 proves the successful server force-close path and provenance-bound delayed-output recovery.
- Ordinary client exits prove provenance-bound server `to_remote` recovery.
- Latest foreign commitment adoption before Swept has exact winner and client-value proof.
- Client no-HTLC justice is mutation-strong.
- Client direct HTLC-success off a server commitment is mutation-strong.
- V63 claim-lock re-arm is bound by a concurrent round and exact eventual claim.
- Server tombstone monotonicity, reorg behavior, and stale-manager non-resurrection are strongly covered.
- Live `exit_funding` affordability is correctly proven to be recomputed after confirmed wallet funds disappear.
- Generic round admission rejects ChannelFunding inputs at the correct pure validation layer.

## 4. Consciously not worth testing

- Stage-2 CSV channel scripts, response windows, retained-bridge early server force-close, and refresh/teleport quiescence have no stage-1 cells.
- R1’s no-CSV HTLC fee race and R2’s expiry-boundary miner race are ratified accepted risks; only their floor/deadline/reserve mitigations were audited.
- Reorg breadth that does not change a funding winner or money decision remains the ratified GAP 5 debt.
- Dust trimming beyond the normative 660-sat/sub-dust isolation rules, P2A pinning, and Esplora capability remain ratified punts.
- A server commitment before a real bridge is structurally unreachable in core stage 1 except through an adversarial race during client exit or a post-bridge server force-close; only those paths were audited.
- A server with no balance and no unresolved HTLC may leave an actualized channel open indefinitely by specification; no force-close test is required for that cell.
- No recovery obligation is assigned to the party deliberately cheating with a revoked transaction.
- A trimmed or zero-valued party output creates no sweep duty for that party; `RC::test_counterparty_commitment_with_zero_local_output` is the sufficient structural proving layer.
- A separate wallet e2e for every release-contract transaction shape would duplicate a pure invariant already pinned at the LDK layer.
- Justice wall-clock scheduling versus the 144-block `to_self_delay` is an operational watch-liveness assumption; deterministic transaction tests can prove the justice path but cannot certify daemon uptime.
- Separate publication tests for stalled, operator, urgent, and protocol-fault closure reasons are not required once reason entry and the shared post-LDK publication path are each proven; restart/retry behavior remains a finding because it is different machinery.
- A literal process-restart duplicate for every downgrade tombstone transition is not required: durable PostgreSQL monotonicity, watch reopening, stale-manager restart, and authenticated integration behavior jointly prove it.
- Client action on an old-scope prefix before the contested FINAL is consequence-free; the material server non-foreclosure gap is listed as P2-3.
- The documented CLN/bip321 failures and load flakes are environment defects, not channel coverage findings.
- No fence or ratified disposition requires a scope challenge.

## 5. Verdict

**FAIL.** The stage-1 stack has twelve P1 coverage clusters: server-side justice, cheater second-stage justice, ordinary HTLC claim quadrants, server V64 recovery, stalled-timer restart, close-package retry, post-Swept winner reselect, client runway admission, client exit-reserve admission, manual-sync refusal, server HTLC-floor/profile enforcement, and downgrade-expiry-margin enforcement.