# MR-1 design note (G1): the protocol surface — `channel-funding` policy, the bridge, the wire fields

**Status**: G1 draft for codex + Greg. 2026-07-31.
**Baseline**: bark-stage1 `ark8-channels-stage1` (= upstream master + the
release-contract crate). **Spec targets**: matrix IDs PV-1..5, PV-7,
PV-8(field), PV-9, PV-10, PV-11(bridge), BR-1..9, BR-14..17(construction),
OP-23..24(wire shape), OP-28(note).

## 1. Scope and the inertness property

Types, construction, and wire shape only — **zero behavior change**. After
this MR a node behaves byte-identically to before: the new policy can be
encoded/decoded but nothing creates it; the bridge builder exists but
nothing calls it outside tests; the new proto fields are absent on every
real message (`supports_channels` stays `false` — the captaind MR flips it
when the subsystem exists). Every compiler-forced match arm on the new
policy takes the conservative/refusing branch, each commented with the
later MR that carves in. Reviewability claim for upstream: "this MR can be
merged and shipped with no observable effect."

Non-goals (later MRs): upgrade/downgrade admission, registration gating,
watches, any captaind/bark runtime, ark-info advertisement logic, the
attestation `channel_id` binding (recorded profile deviation).

## 2. The policy (`lib/src/vtxo/policy/mod.rs` + `lib/src/vtxo/mod.rs`)

- `ChannelFundingVtxoPolicy { user_pubkey: PublicKey }` — struct + impl
  following `PubkeyVtxoPolicy`'s pattern; new `VtxoPolicy::ChannelFunding`
  variant (it is a *destination* policy and guarantees `user_pubkey()`, so
  it belongs in the user-facing enum, mirrored into `ServerVtxoPolicy`'s
  decode per the explicit mirror note at `vtxo/mod.rs:1084`).
- `VtxoPolicyKind::ChannelFunding`, Display/FromStr `"channel-funding"`.
- Tag `VTXO_POLICY_CHANNEL_FUNDING = 0x08` — the next free byte AND the
  byte the spec pins. **Risk register**: if upstream mints `0x08` for
  something else mid-review, the spec renumbers with it; raise tag
  reservation in the draft-MR dialogue.
- `taproot()`/`clauses()`: internal key `musig(user_pubkey, server_pubkey)`
  (the existing `musig::tweaked_key_agg` path), exactly **one** leaf —
  `timelock-sign(expiry_height, S)` — and **no** user exit leaf (PV-3).
  PV-2 says this is the same construction as a board funding output:
  implement by **reusing the same clause constructors** the board/cosign
  taproot uses, and pin the equality in a test (same keys/expiry ⇒ same
  scriptpubkey) so drift is impossible rather than unlikely.
- Forced-match inventory (each site gets the refusing/conservative arm +
  a one-line pointer comment):
  - server arkoor input eligibility (`server/src/arkoor.rs:150-152`
    pattern): refuse — the sanctioned split arrives with the downgrade MR;
  - client send eligibility (`bark/src/arkoor.rs:77-82`): refuse;
  - round-output construction (`server/src/round/mod.rs`): refuse —
    round-issuable only under the refresh extension;
  - watchman action policy (`server/src/watchman/policy.rs`): treat like
    the other cosign-taproot expiry policies — **expiry-claim** — which is
    the spec-correct terminal behavior (EX-1) and unreachable until later
    MRs create such VTXOs;
  - telemetry/labels, `policy_type` column plumbing: mechanical.
  - `is_arkoor_compatible()` stays `false`; `arkoor_pubkey()` `None`. The
    upgrade (destination) and downgrade (input) get explicit entry points
    in their own MRs rather than blanket compatibility here.

## 3. The bridge (`lib/src/channel.rs`, precedent: `lib/src/board.rs`)

Pure functions of `(channel VTXO outpoint + amount + taproot spend info,
two BOLT-3 funding pubkeys, pinned_exit_delta)`:

- `bridge_tx(..) -> Transaction`: `nVersion=3`, `nLockTime=0`, one input
  (the channel VTXO output, `nSequence = pinned_exit_delta`, empty
  witness), out0 = P2WSH of
  `lightning::ln::chan_utils::make_funding_redeemscript(k1, k2)` (the
  canonical BOLT-3 script — lib already depends on `lightning`, and using
  LDK's own constructor makes cross-stack drift impossible) with value =
  the full VTXO amount (0-fee), out1 = `bitcoin_ext::fee::fee_anchor()`
  (P2A, zero value). BR-1..6.
- `bridge_sighash(..) -> TapSighash`: BIP-341 key-path, `SIGHASH_DEFAULT`,
  all prevouts (= the single channel-VTXO TxOut). BR-15/17.
- MuSig2 helpers over the existing `lib/src/musig.rs` surface:
  nonce pair, partial sign, partial verify, and finalize-to-witness
  (aggregate → 64-byte schnorr keyspend witness), all against the
  policy's tweaked aggregate key.
- Determinism is the point (BR-9): construction is witness-independent, so
  both sides compute the same `bridge_txid` before any signature exists —
  that txid is what later MRs derive the permanent `channel_id` and the
  funding outpoint (`bridge_txid:0`) from. A doc comment states this
  contract; MR-1 adds no `channel_id` helper (it is opaque bytes at this
  layer).

## 4. The wire (`server-rpc/protos/bark_server.proto` + `convert.rs` + lib structs)

- `ArkoorCosignRequest` += `optional bytes channel_id = 7;` and
  `optional bytes bridge_pub_nonce = 8;` (tags 1–6 in use). Field comments
  carry the semantics + spec pointer: presence marks the part as an
  upgrade; the server cosigns transfer and bridge in one exchange; a peer
  that does not implement channels ignores them (PV-10).
- `ArkoorCosignResponse` += `optional bytes bridge_pub_nonce = 3;` and
  `optional bytes bridge_partial_sig = 4;`.
- `ArkInfo` += `bool supports_channels = 21;` (+ bump the `last index`
  comment). Client-side `ArkInfo` struct gains the field; nothing reads it
  yet beyond exposure.
- Rust carriers: the lib `ArkoorCosignRequest`/response structs gain
  `Option<..>` fields threaded through `server-rpc/src/convert.rs`
  round-trips. **Attestation note (OP-28/OP-29)**: the attestation message
  computation is untouched — `channel_id` rides deliberately unbound (the
  recorded first-release profile deviation); the destination's policy
  bytes already commit the channel-funding `user_pubkey`/amount, and a
  tampered `channel_id` fails at bridge partial-signature verification.

## 5. Tests (upstream style; unit tests in-file, vectors in `lib/src/test_util/vectors.rs` where fitting)

| Pin | Test |
|---|---|
| encode/decode round-trip + kind strings | policy unit tests |
| unknown-tag rejection (PV-7/PV-9): a `0x09` blob AND a pre-channel decoder posture vs `0x08` | decode-failure unit test |
| taproot construction + no-exit-leaf (PV-1..3) | known-vector scriptpubkey hex, fixed keys/expiry |
| board-construction equality (PV-2) | same-inputs taproot equality test |
| bridge shape (BR-1..6, PV-11) | field-by-field asserts on `bridge_tx` |
| determinism (BR-9) | two independent constructions byte-identical; pinned txid vector |
| sighash + MuSig2 round-trip (BR-15/17) | sign both halves, aggregate, schnorr-verify vs the tweaked output key |
| proto optionality (PV-10, OP-23..24 shape) | convert.rs round-trips with and without the channel fields |

Plus: whole-workspace `cargo check --all --tests --examples` green (the
inertness claim), and the release-contract suite untouched and green.

## 6. Commit plan (within the one MR)

1. `lib: add the channel-funding VTXO policy` — policy + forced arms +
   policy tests.
2. `lib: the presigned bridge transaction and its cosign helpers` —
   `channel.rs` + bridge tests.
3. `server-rpc: optional channel fields on the arkoor cosign exchange;
   supports_channels in ark info` — proto + convert + carrier structs +
   round-trip tests.

Each commit builds workspace-wide (their per-commit CI).

## 7. Open items for review

1. Module name `lib/src/channel.rs` (vs `vtxo/channel.rs`) — board.rs
   precedent argues top-level.
2. `optional` proto3 presence-tracking vs bare fields for the new bytes
   fields — `optional` chosen for explicit presence (matches
   `max_vtxo_amount`'s precedent).
3. Whether the ArkInfo field should wait for the captaind MR instead —
   included here because the client-side refusal test (PV-8) wants the
   field to exist, and a permanently-false bool is inert.
