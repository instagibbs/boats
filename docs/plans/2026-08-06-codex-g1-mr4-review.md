# Codex review record — MR-4 (bark client) G1 design note

**Subject**: `2026-08-06-mr4-client-design.md`, the client-lifecycle MR
(embedded LDK node in bark, ChannelOpen action, channel unilateral
exit; payments/downgrade/surface deliberately out with owners).
**Arc**: six rounds, max effort, read-the-actual-crates instruction —
REWORK ×5 → **PASS-WITH-CHANGES** (both changes applied in place).
Finding counts by round: 2C/6I/1M → 1C/6I/1M → 1C/4I/1M → 6 blockers →
6 completions → 2 clarifications.

## What the rounds forced into the design (the arc's value)

1. **The exit graft became an architecture instead of an enumeration.**
   Rounds 1–4 kept finding wallet-store couplings in the generic exit
   machine (hydration, per-tick reloads, status/cancel/drain, claim
   and CPFP filters, the reconcile). The converged design: an
   `ExitVtxoSource` seam through which every VTXO load routes
   (grep-checkable rule), exactly two generic-state branches — the
   `Processing` delta derivation (a `ChannelFunding` VTXO has **no
   user-signable clause**; the generic clause lookup would fail before
   the tail was ever reached — round 5's confirmation) and the
   `AwaitingDelta → ChannelBridge` handoff — and a record-based
   terminal reconcile that never touches the store.
2. **Crash-recovery became contracts, not prose.** The ChannelOpen
   checkpoint carries the complete deterministic plan (byte-identical
   package reconstruction ⇒ the server's same-backing re-cosign is
   reachable from every crash, with nonces volatile by design);
   establishment split into two persisted substeps around
   `FundingGenerationReady` (the bridge txid depends on the negotiated
   funding keys — round 2 caught the note's overclaim); re-entry
   inspects the reloaded manager by `user_channel_id` and
   adopts/validates/persists the channel id before resuming; the two
   compound persister operations (`create_action_with_locks`,
   `initiate_channel_exit`/`cancel_channel_exit`) carry explicit
   replay/CAS semantics. A reconciling orphan-lock sweep was proposed
   and **rejected in-review** (it races live action creation); the
   atomic method replaced it.
3. **LDK-reality corrections.** KeysManager signer derivation is
   reload-stable on 0.2.4 (settles the old branch's override dance —
   not ported); `FundingTxBroadcastSafe` is consumed as a deliberate
   no-op; `SpendableOutputs` and the complete
   `BumpTransaction::ChannelClose` payload (keyed by `claim_id`,
   idempotent upsert, identity columns) are persisted before
   acknowledgement — 0.2.4 does not redeliver; `pending_claims()` is
   monitor-backed (`get_claimable_balances`), because LDK knows of
   maturing outputs before any descriptor is emitted; `ChannelClosed`
   is a durable signal, never terminality; exit finality
   (`DEEPLY_CONFIRMED` → `ChannelClaimed`) and LDK monitor archival
   are separate clocks.
4. **Two open-time obligations discovered in review**: the exit-fee
   reserve check (P2A anchors are fee attachment points, not sources —
   a zero-balance wallet has no usable exit; refuse by default,
   necessary-not-sufficient stated honestly with its ongoing re-check
   and residual) and the advertised `channel_claim_slack` (a local
   default can disagree with admission). Plus the wire gap: `ArkInfo`
   gains `channel_node_id` / `channel_addresses` (from a new
   `advertise_addresses` config — the bound `0.0.0.0` is not dialable)
   / `channel_claim_slack`, and the epoch-race retry signal moves to
   `Code::Aborted`.

## Verified-clean across rounds

The eager re-key ordering, funder-side witness-completeness self-feed,
designated channel type equality, admission package shape (against the
24-check server contract), the `Code::Aborted` choice, the SCID
banding, and the Processing-branch placement were each explicitly
confirmed against the repository and the lightning 0.2.4 source.

## Status

G1 complete. Implementation awaits Greg's go (standing rule), with the
M4 = lifecycle / M5 = downgrade / M6 = payments sequencing ratification
embedded as note §13.1.
