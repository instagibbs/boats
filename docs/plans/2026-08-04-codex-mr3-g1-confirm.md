# **PASS-WITH-CHANGES**

No blocking design error or stock-LDK assumption remains; implementation can begin. The no-payment profile, C3 gate, WD-16 profile, funding reconstruction, SCID allocation, terminal close, reserve model, and `F` calculation are sound. Remaining changes are code-start precision: keysend means captaind may transiently know a preimage despite never claiming it, and §6a retains one stale statement about commitments.

## Closure table

| Prior finding | Status | Confirmation |
|---|---|---|
| C1 | **PARTIAL** | The enforceable no-origination/no-claim behavior landed, including fail-all handling, but stale categorical “holds no HTLCs/preimages” wording remains. [Note §3](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:243) |
| C2 | **CLOSED** | `push_msat == 0`, dual-funding rejection, no send/invoice API, and never `claim_funds` seal every production value-acquisition boundary. [Note §3](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:244), [LDK open event](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/events/mod.rs:1634) |
| C3 | **CLOSED** | The note now has the shared chain-generation guard, immediate anchor revalidation, level-triggered feed reconciliation, manager-persistence barrier, and startup ordering. [Note §5](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:320), [LDK Confirm contract](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/chain/mod.rs:153), [async persistence](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-background-processor-0.2.3/src/lib.rs:1079) |
| F3 | **CLOSED** | The nonexistent pre-commit hook is no longer claimed; the note explicitly concedes momentary commitment on stock LDK. [Note §3](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:254), [LDK commit transition](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/ln/channel.rs:8726) |
| F4 | **CLOSED** | The durable `opening → awaiting_upgrade → cosigned → registered → terminal` order matches the actual event sequence and includes timeout/quota handling. [Note §6b](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:499) |
| F5 | **CLOSED** | `funding_satoshis` comes from `OpenChannelRequest`; outpoint, final type and redeem script come from `ChannelPending`; together they derive the canonical `TxOut`. [Note §2c](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:150), [LDK ChannelPending](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/events/mod.rs:1394) |
| F6 | **CLOSED** | The normative allocator and schema use `2500..2²⁴`, `UNIQUE(height, tx_index)`, fixed vout 0, persisted collision bumps, and client/funder `scid_privacy`. [Note §5](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:384), [schema](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:703), [LDK fake-SCID range](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/util/scid_utils.rs:85) |
| F7 | **CLOSED** | Both watch insertion and arming reconcile authoritative current-chain state under the channel lock. [Note §7](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:564) |
| F8 | **CLOSED** | The reserve scales per live channel to one worst-case parent-exit response plus one expiry sweep, with maximum feerate/weight, confirmed UTXOs, stable rebump identity, and terminal release. [Note §11.9](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:737) |
| F10 | **CLOSED** | `cltv_claim_slack = 18`; defaults yield `100 + 144 + 18 = 262`, below the variant’s `100 + 288 + 18 = 406`. [Note D5](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:644), [current defaults](/home/greg/bitcoin-dev/cleanroom/bark-stage1/server/captaind.default.toml:24), [LDK 18-block target](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/chain/channelmonitor.rs:276) |
| Finding 1 — origination/claim enforcement | **PARTIAL** | The mechanism is sufficient: LDK never auto-claims and `fail_htlc_backwards` removes the claimable payment. But keysend’s `PaymentClaimable` contains the sender-generated preimage, so “never holds a preimage” is literally false; the correct invariant is never claims or fulfills with it. [LDK fail API](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/ln/channelmanager.rs:8430), [keysend purpose](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/events/mod.rs:162) |
| Finding 2 — C3 gate | **CLOSED** | All four missing durability/reorg mechanisms landed and satisfy `Confirm` ordering. [Note §5](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:337) |
| Finding 3 — WD-16 | **CLOSED** | Current spec really makes bridge retention/early force-close optional; MR-3 validly selects no retention. The parent response actualizes transfer levels but never the bridge. [Spec bridge MAY](/home/greg/bitcoin-dev/cleanroom/boats/08-channels.md:283), [spec parent response](/home/greg/bitcoin-dev/cleanroom/boats/08-channels.md:507), [matrix WD-16](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-conformance-matrix.md:231) |
| Finding 4 — funding output | **CLOSED** | Corrected to two-event assembly and persisted amount-before-accept. [Note §2c](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:156) |
| Finding 5 — expiry terminal transition | **CLOSED** | The note specifies durable terminal state, stock logical force-close, capture/suppression, manager persistence, cleanup, and restart coverage. Stock 0.2.4 exposes only the broadcasting force-close API. [Note §8](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:608), [LDK force-close API](/home/greg/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/lightning-0.2.4/src/ln/channelmanager.rs:4686) |
| Finding 6 — fee reserve | **CLOSED** | Correctly scoped away from production HTLC bumps to the parent-exit response and expiry sweep; only concrete sizing remains. [Note §11.9](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:737) |
| Finding 7 — harness/success CSV | **PARTIAL** | The success-CSV mechanism claim is fully deleted, and §3 correctly describes commitment updates; §6a still incorrectly says “no commitment, second stage.” [Correct §3 text](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:264), [stale §6a text](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:488), [harness revoke warning](/home/greg/bitcoin-dev/cleanroom/bark-stage1/bark-channels/tests/common/mod.rs:1700) |
| Finding 8 — SCID consistency | **CLOSED** | Normative section, schema, and decision summary now agree. [Note §5](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:384) |

## Remaining findings

No **[blocking]** findings.

- **[code-start] Severity: Important — terminal-transition crash barrier**  
  **Location:** [§8](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:608)  
  **Why:** `terminal_at` and the manager snapshot are separate durable writes.  
  **Fix:** Reconcile terminal rows level-triggeredly until the restored manager no longer contains the channel; inject crashes before/after force-close and manager persistence.

- **[code-start] Severity: Important — reserve parameters**  
  **Location:** [config placeholders](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:678)  
  **Why:** Exact package weights, maximum feerate and satoshi reserve remain intentionally unset.  
  **Fix:** Pin them before enabling admissions; retain the expiry tranche after a parent-response confirmation and suppress LDK bump events generated by logical terminal closes.

- **[code-start] Severity: Minor — categorical HTLC wording**  
  **Location:** [§1](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:36), [§3](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:233)  
  **Why:** Stock LDK can commit an HTLC and expose a keysend preimage even though captaind never claims it.  
  **Fix:** Consistently say “originates and claims no production payments”; remove “holds no HTLCs/preimages.”

- **[code-start] Severity: Minor — stale harness/outbox wording**  
  **Location:** [§6a](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:488), [module table](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-08-04-mr3-captaind-channels-design.md:139)  
  **Why:** Cooperative settlement signs commitments/HTLC transactions, and the release mechanism is a reconciler rather than a one-shot outbox.  
  **Fix:** Say none are broadcast and rename the residual “outbox” references.

**Verified sound:** no-claim/no-origination profile; no-retention bridge invariant; C3 reorg-and-persistence gate; two-event funding `TxOut`; segregated persistent SCIDs; logical terminal force-close; parent/expiry reserve scope; `262 < 406`; and no load-bearing success-CSV mechanism claim.