# REWORK

The scope cut is sound in principle: an MR-3 server that never originates or claims payments no longer needs the round-2 terminal-receipt floor or bridge-retention scheduler, and `F = depth + pinned_exit_delta + slack` is the correct stage-1 direction. It is not sound as written. Stock LDK cannot enforce the literal “no live HTLCs” invariant with the named setting, the on-disk r2 omits several claimed C3/F5/F7/F8 fixes, WD-16’s spec citation is wrong and conflicts with normative text, and expiry lacks an LDK terminal transition.

## Round-2 closure

| Finding | Status | Reason |
|---|---|---|
| C1 | **PARTIAL** | Private forwarding is disabled, but direct final-hop/keysend HTLCs remain accepted; no fail-all handler is specified. |
| C2 | **PARTIAL** | Never claiming receipts would remove the bridge/force-close obligation, but the receive and initial-push boundaries are not sealed. |
| C3 | **PARTIAL** | The row lock and atomic registration transaction are sound; chain-generation serialization, anchor revalidation, durable level-trigger reconciliation, and insertion reconciliation are absent. |
| F3 | **CLOSED** | The nonexistent pre-commit receive floor is no longer claimed or needed when receipts are never claimed. Literal pre-commit rejection would still require an LDK hook. |
| F4 | **CLOSED** | The durable `opening → awaiting_upgrade → cosigned → registered` order matches LDK events and includes quotas/timeouts. |
| F5 | **OPEN** | `funding_satoshis` is still not persisted, and r2 still incorrectly says `ChannelPending` supplies the funding output. |
| F6 | **PARTIAL** | The `2500..2²⁴` band and client-side privacy requirement are correct, but §5 and the schema do not consistently encode them. |
| F7 | **PARTIAL** | Predicates and reorg metadata are correct; current-chain reconciliation at insertion/arming remains absent. |
| F8 | **OPEN** | There is still no quantitative reserve accounting, and the remaining text still scopes it to HTLC/LDK bump events. |
| F10 | **PARTIAL** | The serial-confirmation model is corrected, but the actual config remains `<TBD>` rather than fixing the integer. |

## New/still-open findings

1. **Severity: Critical — “no live HTLCs” is not enforceable with the named LDK configuration**  
   **Location:** [r2 §1](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:35), [LDK forwarding option](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/util/config.rs:870), [keysend advertisement](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/ln/channelmanager.rs:15714), [final-hop acceptance](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/ln/channelmanager.rs:7961), [commit transition](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/ln/channel.rs:8726)  
   **Why:** `accept_forwards_to_priv_channels=false` only rejects forwarding over a private outgoing channel. LDK still advertises keysend and can commit a final-hop HTLC before emitting `PaymentClaimable`. Separately, r2 never rejects `PushMsat(nonzero)` or `DualFunded` from [`OpenChannelRequest`](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/events/mod.rs:1614), allowing value to move at open despite the “economically inert” claim and the spec’s zero-server-balance premise ([spec](/home/greg/bitcoin-dev/cleanroom/boats/08-channels.md:484)).  
   **Fix:** Require `PushMsat(0)` and reject `DualFunded`; expose no send/invoice APIs; never call `claim_funds`; immediately call [`fail_htlc_backwards`](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/ln/channelmanager.rs:8430) for every `PaymentClaimable`, including replay. Rename the guarantee to “no originated or claimed production payments.” Absolute zero committed HTLCs requires a pre-commit LDK hook/fork.

2. **Severity: Critical — the advertised C3 gate fixes are not in r2**  
   **Location:** [r2 gate](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:283), [one-shot outbox](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:300), [schema](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:591), [watch insertion](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:488)  
   **Why:** The file contains no chain-generation gate, pre-feed anchor revalidation, level-triggered registered-row reconciliation, or TxIndex reconciliation at insertion. The stale-anchor race still violates LDK’s [`Confirm` contract](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/chain/mod.rs:161), while feed success still precedes asynchronous manager persistence ([processor](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-background-processor-0.2.3/src/lib.rs:1079)).  
   **Fix:** Add the serialized chain generation/reorg gate, revalidate immediately before feed, level-trigger all registered rows until the manager snapshot is durable, add explicit delivery schema, and reconcile authoritative chain state while inserting/arming each watch.

3. **Severity: Important — WD-16 is a profile statement, not the cited normative invariant**  
   **Location:** [matrix WD-16](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-conformance-matrix.md:231), [user story](/home/greg/bitcoin-dev/cleanroom/boats/docs/channel-user-stories.md:149), [r2 derivation](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:199), [actual spec](/home/greg/bitcoin-dev/cleanroom/boats/08-channels.md:283)  
   **Why:** `08-channels.md:1605-1614` is unrelated liquidity-adjustment text. Current normative text expressly says the server **MAY** retain the completed bridge and then force-close. Holding only a partial prevents bridge broadcast under the selected exchange, but does not structurally prevent actualizing retained Ark levels; the parent-exit response does exactly that reactively. Stock LDK also logically force-closes on virtual-funding unconfirmation, as r2 acknowledges.  
   **Fix:** Correct the spec/matrix citation and either remove the spec’s retention MAY or describe D2/D3 as the MR-3 no-retention profile. State the invariant narrowly: “captaind never initiates an Ark bridge unroll.” The monitor claim and actualized-output expiry sweep are correct and neither is an unroll, but they are not an exhaustive list of reactive broadcasts.

4. **Severity: Important — funding-output persistence remains impossible as specified**  
   **Location:** [r2 §2c](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:143), [state](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:449), [schema](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:595), [`ChannelPending`](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/events/mod.rs:1394)  
   **Why:** `ChannelPending` supplies an outpoint and optional redeem script/type, not a value or `TxOut`.  
   **Fix:** Before acceptance persist `{temporary_channel_id, counterparty, funding_satoshis, channel_negotiation_type, proposed_type}` from `OpenChannelRequest`; then derive the funding `TxOut` from the persisted amount and final P2WSH script.

5. **Severity: Important — expiry has no LDK terminal transition**  
   **Location:** [r2 expiry](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:515), [LDK close API](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/ln/channelmanager.rs:4686)  
   **Why:** Once an expiry/ancestor sweep makes the virtual funding impossible, LDK can otherwise retain an active channel forever. LDK 0.2.4 has no public abandon-without-broadcast API.  
   **Fix:** Specify a durable Ark-terminal transition that withdraws the synthetic confirmation or otherwise closes LDK, captures and suppresses the impossible commitment broadcast, persists the manager, and releases watches/reserves. Add expiry-plus-restart coverage. This is a logical LDK force-close, not an Ark unroll.

6. **Severity: Important — F8 remains unaccounted and mis-scoped**  
   **Location:** [r2 hardening](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:621), [reserve claim](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:630)  
   **Why:** The text still mentions “HTLC bump” and `BumpTransactionEvent`, with no amount, feerate ceiling, concurrency bound, UTXO reservation, rebump identity, or release rule. MR-3’s real server liabilities are parent-exit response packages and expiry sweeps.  
   **Fix:** Define weights, maximum admitted feerate, maximum concurrent claims, confirmed-wallet-UTXO reservation, reuse across rebump, and terminal release for those two operations.

7. **Severity: Important — the harness and success-CSV statements are factually wrong**  
   **Location:** [r2 harness](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:429), [LDK commitment signing](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/ln/channel.rs:12908), [harness final-revoke warning](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/common/mod.rs:1700), [spec success CSV](/home/greg/bitcoin-dev/cleanroom/boats/08-channels.md:1256)  
   **Why:** A cooperative HTLC updates and signs commitments and HTLC second-stage transactions; after `PaymentClaimed`/`PaymentSent`, the final revoke can still be in flight. Clean settlement means none is broadcast and no CSV path executes—not that commitments are untouched. Also, the Ark success CSV delays the preimage path so the **offerer’s timeout wins**; it does not make a non-unrolling preimage-holder win.  
   **Fix:** Correct both claims. Either use readiness/capacity as the MR-3 operability test or explicitly admit the test-only live-HTLC interval and wait through final revocation. Do not present success CSV alone as making terminal server receipt safe.

8. **Severity: Important — SCID requirements remain internally inconsistent**  
   **Location:** [normative SCID section](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:329), [decision summary](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:734), [schema](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:596)  
   **Why:** The summary correctly reserves `2500..2²⁴`, but §5 only requires `≥1`; the schema’s uniqueness constraint references an undeclared `vout`.  
   **Fix:** Put the band in the normative allocator and either add `vout` or use `UNIQUE(height, tx_index)` because funding vout is fixed at zero.

## Verified sound

- The defined exchange leaves captaind only a bridge partial and the client completes the bridge ([spec ordering](/home/greg/bitcoin-dev/cleanroom/boats/08-channels.md:410)).
- No-retention is allowed by the current spec’s `MAY`, and empty open/close/exit/expiry requires no HTLC.
- Monitor claims after client force-close and an expiry-leaf sweep of an already-on-chain channel VTXO are valid and are not bridge unrolls.
- Normal private-channel forwarding is disabled by `accept_forwards_to_priv_channels=false`.
- F4 ordering, armed/unarmed watch predicates, and rich reorg metadata are sound.
- The client-side `negotiate_scid_privacy` requirement and `2500..2²⁴` alias-separation band are sound.

## Definitive `F` answer

A genesis level needs **one confirmation, not 18 confirmations**, before its child level can proceed. TRUC permits only a one-parent/one-child unconfirmed topology; each zero-fee level and its P2A child therefore confirms as one package, after which the next level is immediately spendable. Neither [BIP 431](https://github.com/bitcoin/bips/blob/master/bip-0431.mediawiki), [Core v29 TRUC policy](https://github.com/bitcoin/bitcoin/blob/v29.0/src/policy/truc_policy.h), nor [`submitpackage`](https://bitcoincore.org/en/doc/29.0.0/rpc/rawtransactions/submitpackage/) requires deeper burial. Bark likewise advances on plain `TxStatus::Confirmed` ([exit engine](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark/src/exit/progress/mod.rs:89)).

Set:

`cltv_claim_slack = 18`

Therefore, with current `D=100` and `d=144`:

- Stage 1: `F = D + d + 18 = 262`
- Ark-channel variant: `D + 2d + 18 = 406`

The smaller direction is correct: stage 1 has no success-CSV delay. Eighteen is an operational margin, consistent with LDK’s [`MAX_BLOCKS_FOR_CONF`](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/chain/channelmonitor.rs:276); it assumes competitive next-block CPFP and rebumping, not a consensus guarantee under unbounded congestion.