# DRAFT — GitLab draft-MR description (Greg posts; replace [bracketed] items)

Suggested title: `Draft: bark-channels — release-contract tests for Lightning channels on channel VTXOs (ARK #8)`
Source: `gsanders87/bark:ark8-channels-stage1` → target: `ark-bitcoin/bark:master`

---

## What this is

The opening piece of a staged series implementing **ARK #8 channels**:
Lightning channels whose funding output lives in a VTXO's presigned,
never-broadcast exit chain (a `channel-funding` VTXO plus one presigned
"bridge" transaction whose out0 is a stock BOLT-3 P2WSH funding output).
Open = an ARK #5 self-spend ("upgrade"); close = a BOLT co-op close settled
by a sanctioned split ("downgrade"); payments between two bark clients of
one captaind route as ordinary LN HTLCs, instantly. The chain is touched
only on unilateral dispute.

Spec: [link to 08-channels.md at the published revision — the document is
layered: a self-contained core lifecycle, with the dedicated channel type
and the refresh/teleport as later extensions; this series implements the
core plus its first-release profile].

**This MR deliberately contains zero protocol changes.** It adds one
workspace crate, `bark-channels`, whose integration tests pin — against the
exact crates.io `lightning` release in the lockfile — every LDK behavior
the feature depends on, plus the lockfile bump `0.2.2 → 0.2.4` (inside the
existing `lightning = "0.2.0"` requirement; 0.2.3 fixed a
zero-fee-commitment reserve bug and the `ZeroConf` feature accessors on our
path).

## Why start here

The feature's load-bearing claim is that **stock, released LDK** suffices
for a channel whose funding transaction never exists on-chain: manual
funding (`unsafe_manual_funding_transaction_generated`, funder-only),
application-fed confirmations at real heights, a capture-only broadcaster,
`zero_fee_commitments`, and external-input fee-bumping via
`BumpTransactionEvent`. These eleven tests prove that composition
empirically and then stay on as release tripwires: a future `lightning`
version bump that changes any pinned behavior fails here first, at the bump
commit. Highlights:

- channel usability is gated on the application feeding the funding
  confirmation — the mechanism the series later uses to gate `ChannelReady`
  on Ark-side registration;
- the confirmation can be fed at an already-historical height (the real
  case: the VTXO's chain anchor long predates the open) — depth is computed
  against the live best block, which never regresses;
- force-close, anchor CPFP, and HTLC claims are asserted against
  in-test-known data (derived prevout fees, descriptor-vs-commitment
  cross-checks, cryptographic verification of the wallet's own signatures);
- the synthesized funding position (a virtual funding occupies no block
  position, so the short channel id is node-local fiction): same-height
  uniqueness, restart stability across either side's reload, peers need
  not agree, aliases carry all wire uses, and the duplicate-position assert
  is pinned as fatal.

## Where the series goes (five MRs, each independently mergeable, feature
off by default until the set is complete — all on stock released LDK; the
spec's channel-type and refresh extensions are later, separate arcs)

1. **This MR** — the harness crate's release-contract tests.
2. Protocol surface: the `channel-funding` VTXO policy (tag `0x08`, opaque
   policy bytes — no proto changes) + canonical bridge construction in
   `lib`; two optional fields on the arkoor cosign exchange
   (`channel_id`, `bridge_pub_nonce` + the server's bridge nonce/partial on
   the response); `supports_channels` in ark info.
3. captaind: the embedded-LDK subsystem behind an `OptionalService`
   `[channels]` config (the `watchman` pattern) — upgrade admission,
   registration-gated readiness, parent-exit watches riding
   `ChainEventListener`, expiry sweep as a `watchman` policy arm.
4. bark client: open action (the `WalletAction` framework), CLTV floor,
   unilateral exit through the existing exit subsystem, intra-ark payments.
5. Close-by-downgrade end to end; then REST/CLI surface and an adversarial
   sweep.

It reuses what the repo already has — `RegisterVtxoTransactions` and the
`Unregistered → Spendable` gate, the `vtxos_in_flux` atomic reservation,
`sync::ChainEventListener`/`watchman`, the `WalletAction` checkpoints — and
adds no new Ark RPC.

## Feedback wanted at this stage

1. **Appetite and placement**: an embedded-LDK channels subsystem in
   captaind behind `OptionalService` — right shape, or hosted differently?
2. **Naming**: `bark-channels` crate, `server/src/channels/`,
   `bark channel` CLI.
3. **Sequencing** against in-flight work we want to compose with rather
   than collide: `2026-05-enforcevtxoreg` (our readiness gate keys on
   registration enforcement), `lightning-to-ark-lib`, `htlc-manager`,
   `2026-07-ldkpathfinding`.
4. **Chunking**: is the five-stage cut above the right review granularity,
   or would you prefer it coarser/finer?

Test-review nits are welcome too, but the questions above are the ones this
draft exists to ask. Changelog entry follows once the MR number exists.

[Greg: sign-off / any Second-internal context]
