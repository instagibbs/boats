# DRAFT — upstream design issue for ark-bitcoin/bark (Greg posts; do not publish as-is)

Title: **RFC: ARK #8 channels — Lightning channels funded by channel VTXOs (open = upgrade, close = downgrade)**

---

## Summary

This proposes adding **channel VTXOs** to bark: a VTXO whose output funds a
real BOLT Lightning channel through one presigned "bridge" transaction, per
the ARK #8 spec. The channel lives entirely off-chain; the chain is touched
only on unilateral dispute. Payments between two bark clients of the same
captaind route as ordinary LN HTLCs (client ↔ captaind ↔ client), instantly.

```
channel VTXO output: taproot( musig(user, server) keypath, server expiry leaf )
     │ presigned bridge tx (off-chain; input nSequence = exit_delta; TRUC + P2A)
     ▼
bridge out0: stock BOLT-3 P2WSH 2-of-2 funding output
     │ commitment tx (off-chain unless force-closed; zero-fee commitments)
     ▼
to_local / to_remote / HTLCs
```

- **Open = upgrade**: an ARK #5 out-of-round *self-spend* of a `pubkey` VTXO
  into a new `channel-funding` VTXO; the server cosigns the transfer and the
  bridge in one exchange. Registration (the existing
  `RegisterVtxoTransactions` flow) is the point of no return and gates
  channel usability.
- **Close = downgrade**: an ordinary BOLT-2 cooperative close fixes the final
  balances; a server-sanctioned ARK #5 split of the channel VTXO then pays
  each side its close-fixed balance as plain `pubkey` VTXOs. Fully off-chain.
- **Unilateral exit** is always available without server cooperation:
  genesis chain → bridge (after `exit_delta`) → commitment → stock BOLT-3
  claims.

The Lightning layer is **stock LDK from crates.io** — `lightning` is already
a workspace dependency (currently locked 0.2.2; this bumps to 0.2.4). No
fork, no git pin. Manual/virtual funding uses
`unsafe_manual_funding_transaction_generated` + app-fed confirmations; the
channel type is `zero_fee_commitments` (same TRUC/P2A family as the exit
chain).

Spec: ARK #8 (`08-channels.md`) plus a stage-1 profile — link to the
published spec revision. [Greg: insert links]

## Why this is smaller than it sounds

The design deliberately rides machinery bark already has:

- **New policy variant** `channel-funding` (type byte `0x08`, the next free
  tag): policies are opaque bytes on the wire, so **zero proto changes** for
  the policy itself.
- **Wire surface**: two optional fields on the arkoor cosign part
  (`channel_id`, `bridge_pub_nonce`), the server's bridge nonce + partial on
  the response, and one ark-info bool (`supports_channels`). The downgrade
  split adds **no fields** — it is a plain ARK #5 request whose input
  carries the channel policy.
- **Reuses as-is**: the `Unregistered → Spendable` registration flow (the
  usability gate keys on it), the `vtxos_in_flux` + atomic spent-mark
  reservation (serializes the split against rounds/offboards), the
  `sync::ChainEventListener` + `watchman` action machinery (parent-exit
  watches and the expiry sweep are a new listener + policy arm), and the
  client's `WalletAction` checkpoint framework (open/close are new actions —
  crash-durability for free).

New surface, all gated:

- a `bark-channels` crate: the embedded-LDK harness (node assembly on
  `lightning` + `lightning-net-tokio` + `lightning-background-processor`,
  virtual-funding chain feed, capture-only broadcaster, external-input
  fee-bumping via `BumpTransactionEvent`);
- `server/src/channels/`: an `OptionalService` subsystem (the `watchman`
  config pattern), postgres tables for channel state + LDK persistence,
  cosign admission for the upgrade/downgrade variants — disjoint from
  `server/src/ln/` (the CLN payment backend is untouched);
- client channel open/close actions, REST `/channels`, `bark channel` CLI.

Default **off**; a captaind without `[channels] enabled = true` behaves
byte-identically to today (the whole existing suite runs green with the
subsystem on and off).

## Safety model (stage 1) and staging

Stage 1 deliberately excludes two spec mechanisms, with documented
consequences:

- **No teleport (channel refresh)**: a channel cannot outlive its VTXO's
  expiry in place; the lifecycle is downgrade → ordinary round refresh →
  re-upgrade, with depth headroom reserved at open so the cooperative close
  is always available. Long-lived in-place channels arrive with the teleport
  in a later stage.
- **No HTLC-success CSV** (the ARK channel type's script change): on-chain
  expired-HTLC ordering is a fee race rather than CSV-ordered. Mitigations
  now: a mandatory CLTV floor budgeting the full exit-actualization path
  (unroll depth + bridge CSV + confirmation margins) on every invoice,
  received HTLC, and forward; capped forwarding defaults. The CSV lands as
  its own upstream-LDK track in stage 2, at which point forwarding limits
  can relax.

Proposed MR staging (each independently mergeable, feature off until the
series completes; every commit builds — matching the per-commit CI):

1. **Protocol surface**: `channel-funding` policy + bridge construction in
   lib, proto fields, `supports_channels`. Pure, vector-tested.
2. **captaind**: the `bark-channels` harness crate + the `[channels]`
   subsystem + upgrade admission + registration gate + watches + expiry
   sweep.
3. **bark client**: open action + CLTV floor + unilateral exit + intra-ark
   payments end-to-end.
4. **Downgrade**: close-outcome recording + sanctioned split + symmetric
   watches.
5. **Surface**: REST/CLI/openapi + adversarial test sweep + operator docs.

## Asks

1. Appetite / placement: does an embedded-LDK channels subsystem behind
   `OptionalService` fit captaind, or would you rather see any part hosted
   differently?
2. Naming: `bark-channels` crate, `server/src/channels/`, `bark channel`
   CLI — collisions or preferences?
3. Interaction with in-flight work we should sequence around:
   `2026-05-enforcevtxoreg` (we *want* registration enforcement — our gate
   keys on it), `lightning-to-ark-lib`, `htlc-manager`,
   `2026-07-ldkpathfinding`.
4. MR chunking: is five the right cut, or would you prefer
   coarser/finer once MR-1 is in front of you?

[Greg: closing line / sign-off]
