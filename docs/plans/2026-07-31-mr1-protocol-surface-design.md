# MR-1 design note (G1): the protocol surface — `channel-funding` policy, the bridge, the wire fields

**Status**: G1 rework (codex REWORK → this revision). 2026-07-31.
**Series position**: MR-1 — the protocol surface, stacking on the upstream
opener (the `bark-channels` release-contract crate). **Baseline**:
bark-stage1 `ark8-channels-stage1` @ `ea33bbf4` (= upstream master + the
opener). **Spec targets**: matrix IDs PV-1..5, PV-7, PV-8(field), PV-9,
PV-10, PV-11(bridge), BR-1..9, BR-14..17(construction), OP-23..24(wire
shape), OP-28(note) — see §5 for the construction-primitive-only scoping.

## 1. Scope and the (narrowed) inertness property

Types, construction, wire shape — **and the minimum admission gate that
makes decoding the new policy safe**. The G0/G1 reviews established that
"nothing creates it" is false without an explicit gate: once `0x08`
decodes, the *generic* arkoor path (and the shared Lightning pay /
receive-claim validator that reuses it) would accept a `channel-funding`
**destination** and mint a bridgeless channel VTXO — marking the input
spent before signing — because the shared validator gates only the *input*
policy, and the builder copies arbitrary destination policies through
(`lib/src/arkoor/mod.rs:415`, `server/src/arkoor.rs:36,122`,
`server/src/ln/mod.rs:79,735`). A bridgeless `channel-funding` VTXO
violates the spec's no-bridgeless rule (08-channels.md core), so MR-1 MUST
foreclose it.

The honest inertness claim is therefore: **previously valid non-channel
flows are byte/transaction-identical after this MR; every channel-shaped
request (a `channel-funding` destination anywhere, or a `channel-funding`
input in any generic consumer) is rejected before any DB write, output
creation, or spent-state mutation.** Nothing yet *authorizes* a
channel-funding destination — the upgrade admission path (MR-2, captaind
server) and the downgrade split (MR-4, close) add the only two authorized
entry points; until then the gate refuses all of them. `supports_channels`
stays `false`.

Not fully inert, and the note no longer claims otherwise: adding a public
`VtxoPolicy` variant and public proto/struct fields is a source-level API
change (breaks exhaustive matches / struct literals downstream), and
exposing `supports_channels: false` through bark-json changes observable
JSON (protobuf stays byte-identical — `false` is omitted). These are
recorded as API/changelog items, not behavior changes.

Non-goals (later MRs): upgrade/downgrade *authorization* and admission
checks, registration gating, watches, any captaind/bark runtime, the
ark-info advertisement *logic*, the attestation `channel_id` binding
(recorded first-release profile deviation).

## 2. The policy and the admission gate

### 2a. Policy type (`lib/src/vtxo/policy/mod.rs`, `lib/src/vtxo/mod.rs`)

- `ChannelFundingVtxoPolicy { user_pubkey: PublicKey }` following
  `PubkeyVtxoPolicy`; new `VtxoPolicy::ChannelFunding` variant (a
  *destination* policy guaranteeing `user_pubkey()`), mirrored into
  `ServerVtxoPolicy`'s decode per the mirror note at `vtxo/mod.rs:1084`.
- `VtxoPolicyKind::ChannelFunding`; Display/FromStr `"channel-funding"`.
- Tag `VTXO_POLICY_CHANNEL_FUNDING = 0x08` — next free byte and the
  spec-pinned byte. **Tag-race handling**: an upstream `0x08` allocation
  mid-review REOPENS G1 (spec + matrix + vectors + compatibility fixtures
  all move together), not a silent renumber; raise a tag reservation in
  the draft-MR dialogue.
- **Taproot (corrected per G1-F2)**: the internal key is the **untweaked**
  aggregate. Reuse the board/cosign constructors exactly:

  ```rust
  let internal = musig::combine_keys([user_pubkey, server_pubkey]).x_only_public_key().0;
  cosign_taproot(internal, server_pubkey, expiry_height)   // adds the one expiry leaf
  ```

  `cosign_taproot` (the board path, `lib/src/board.rs:53` via
  `lib/src/tree/signed.rs:69`) performs the taproot output tweak; do NOT
  use `tweaked_key_agg` for the internal key (that is for key-path signing,
  §3). `clauses()` returns exactly one `TimelockSignClause(expiry, S)` and
  **no** user exit leaf (PV-1..3). Pinned by a byte-equality test against
  a board funding output built from the same keys/expiry (PV-2).

### 2b. The admission gate (the one behavior this MR adds)

The gate lives in the **one shared pre-builder validator**,
`validate_cosign_request` (`server/src/arkoor.rs:57`), which is the common
chokepoint of all three generic cosign modes — direct arkoor
(`cosign_oor`), Lightning-pay, and Lightning-receive-claim
(`server/src/ln/mod.rs:135,794`). Placing BOTH gates there — before builder
construction, before `execute_vtxo_tree_update`, before the `vtxos_in_flux`
spent-mark, and (critically for receive-claim) before its preimage is
durably settled (`ln/mod.rs:797`) — is what makes one gate cover every
path. (G1-re-review-F1: the earlier draft put the input check only in
`cosign_oor`, which Lightning-pay bypasses via `set_vtxos` → builder,
letting a crafted LN-pay spend a `ChannelFunding` input into an HTLC output.
Builder construction does NOT enforce `is_arkoor_compatible`
(`lib/src/arkoor/mod.rs:1132,1340`), so the shared validator is the only
safe chokepoint.)

- **Destination gate**: `validate_cosign_request` MUST reject any
  `ChannelFunding` destination — normal or isolated (the builder copies
  supplied policies into both, `arkoor/mod.rs:427,544`) — with a distinct
  `badarg`.
- **Input gate**: `validate_cosign_request` MUST reject a `ChannelFunding`
  *input* in every unauthorized mode. Also mirror the refusal in the
  non-arkoor consumers that don't route through this validator: round-input
  (`server/src/round/mod.rs:313`) and offboard
  (`server/src/offboards.rs:187`) admission (both currently accept policies
  generically).
- Client-side **destination** eligibility (`bark/src/arkoor.rs:77-82`)
  refuses too (input *selection* on the client is policy-agnostic today;
  this is a destination filter, not an input one).

### 2c. Forced-match inventory (compiler-enforced; each arm is the safe one)

Verified against the tree (G1-F4):
- `VtxoPolicyKind::Display` (`policy/mod.rs:170`).
- The six `VtxoPolicy` methods at `policy/mod.rs:720`: `policy_type`,
  `is_arkoor_compatible` (→ `false`), `arkoor_pubkey` (→ `None`),
  `user_pubkey` (→ the field), `taproot`, `clauses`.
- Encoding (`vtxo/mod.rs:1025`) — the `0x08 || 33-byte key` arm.
- The historical bark migration `m0021_fix_lightning_movements.rs:49` —
  safe arm `false`.
- Round-output construction/telemetry, watchman policy — watchman takes the
  **expiry-claim** arm (spec-correct terminal behavior EX-1; unreachable
  in-tree until a later MR creates such a VTXO, which is why the gate in
  §2b is what actually keeps it unreachable now).
- `FromStr` and the server decoder mirror are wildcard/tag matches (not
  compiler-forced) but semantically required — covered by tests.

## 3. The bridge (`lib/src/channel.rs`; precedent `lib/src/board.rs`)

Pure functions of `(channel-VTXO outpoint + amount + taproot spend info,
two BOLT-3 funding pubkeys, pinned_exit_delta)`:

- `bridge_tx(..) -> Transaction`: `nVersion=3`, `nLockTime=0`, one input
  (channel VTXO output, `Sequence::from_height(pinned_exit_delta)`, empty
  witness), out0 = `.to_p2wsh()` of
  `lightning::ln::chan_utils::make_funding_redeemscript(k1, k2)` (canonical
  BOLT-3, key-sorted; lib already deps `lightning` 0.2.4) with value = the
  full VTXO amount (0-fee), out1 = `bitcoin_ext::fee` P2A anchor (zero
  value). BR-1..6, PV-11. (G1 confirmed all of these correct.)
- `bridge_sighash(..) -> TapSighash`: BIP-341 key-path, `SIGHASH_DEFAULT`,
  `Prevouts::All(&[channel_vtxo_txout])`. BR-15/17.
- MuSig2 helpers over `lib/src/musig.rs`, using
  `tweaked_key_agg([A,S], spend_info.tap_tweak())` for the **signing**
  aggregate: nonce pair, partial-sign, partial-verify, and finalize →
  64-byte schnorr key-spend witness.
- Determinism (BR-9): construction is witness-independent, so both sides
  compute the same `bridge_txid` before any signature — the value MR-3
  later derives the permanent `channel_id` and funding outpoint
  (`bridge_txid:0`) from. Documented as a contract; MR-1 adds no
  `channel_id` helper (opaque bytes at this layer).

## 4. The wire (grouped carriers per G1-F3)

Loose `Option` fields let package transformations (`with_vtxo`,
`convert_vtxo`, `set_vtxos` in `lib/src/arkoor/{mod,package}.rs`) silently
drop half a pair. Use grouped carriers instead:

- proto: `ArkoorCosignRequest` += `optional bytes channel_id = 7;`
  `optional bytes bridge_pub_nonce = 8;`; `ArkoorCosignResponse` +=
  `optional bytes bridge_pub_nonce = 3;` `optional bytes
  bridge_partial_sig = 4;`; `ArkInfo` += `bool supports_channels = 21;`
  (bump the `last index` comment). Tags verified free.
- lib carriers: `Option<BridgeCosignRequest { channel_id: [u8;32],
  pub_nonce }>` on the request, `Option<BridgeCosignResponse { pub_nonce,
  partial_sig }>` on the response — one `Option` each, so a half-present
  pair is unrepresentable internally.
- `convert.rs` MUST reject a half-present or malformed protobuf pair
  (not silently `None` it), and the group MUST survive every package
  transformation — tested protobuf → internal → `set_vtxos`/`convert_vtxo`
  → protobuf.
- Attestation (OP-28, reworded per G1-F3-clean): the attestation message
  hashes only input id, output count, per-output amount, and per-output
  policy bytes (`lib/src/attestations.rs:388`) — it provably cannot absorb
  the new request fields. `channel_id` rides deliberately unbound (recorded
  profile deviation); the amount is committed separately and the policy
  bytes commit the channel-funding `user_pubkey`; a tampered `channel_id`
  fails at bridge partial-signature verification (MR-3).

## 5. Tests and the construction-primitive scoping (G1-F5)

These matrix IDs are dischargeable HERE only as **construction/schema
primitives**; their runtime obligations stay owned by the captaind MR
(MR-2, server admission) and the bark MR (MR-3, client), and the plan's §7
table is labeled accordingly:
- BR-3/BR-8 (source/storage/never-reread-live), BR-4 (negotiated-amount
  equality), BR-7 (commitment-input half), BR-14 (key exchange/registry),
  BR-16 (commitment signing), BR-15/BR-17 (once-at-open / session
  freshness / atomicity / retry), OP-23 (identifier lookup), OP-24
  (at-most-one-part admission) — MR-1 pins only the tx/schema shape they
  build on.

Tests (upstream style; unit in-file, vectors in
`lib/src/test_util/vectors.rs`):

| Pin | Test |
|---|---|
| exact `0x08 \|\| 33-byte-key` encoding + decode round-trip + kind strings | policy unit + vector |
| unknown-tag rejection: a **`0xff`** blob (not `0x09` — the likely next allocation) | decode-failure unit |
| taproot shape, no exit leaf, key-role separation (PV-1..4) | scriptpubkey hex vector, fixed keys/expiry |
| board-construction byte-equality (PV-2) | same-inputs equality test |
| bridge shape + `verify_tx` (BR-1..6, PV-11) | field asserts + consensus `verify` of the funding-input spend |
| determinism (BR-9) | two constructions byte-identical; pinned txid vector |
| sighash + MuSig2 round-trip (BR-15/17) | sign both halves, aggregate, assert **one 64-byte DEFAULT witness**, schnorr-verify vs the tweaked output key; **reject a corrupted partial**; **funding-key-order independence** |
| **destination gate** (the new behavior): direct arkoor AND shared Lightning-pay both reject a `ChannelFunding` destination **before any DB/output/spent-state mutation** | server unit/integration |
| generic **input** refusal (arkoor/round/offboard) | server unit |
| proto optionality + pair preservation (PV-10, OP-23..24 shape) | convert round-trips ±channel fields, and through `set_vtxos`/`convert_vtxo` |
| pre-channel decoder rejects `0x08` (PV-9) | assert the policy decoder (`decode_vtxo_policy`) errors on a `0x08` blob — the structural unknown-tag rejection at `vtxo/mod.rs:1084` — i.e. a channel VTXO cannot be misread as a non-channel one; plus unknown proto fields skipped (prost 0.14 skips-not-preserves) |

Plus whole-workspace `cargo check --all --tests --examples` green, the
opener's release-contract suite untouched and green.

## 6. Commit plan (each builds workspace-wide)

Decoding `0x08` and the admission gate MUST land in the **same commit**
(G1-re-review): the moment the policy is decodable, the shared validator
must already refuse it, or the intermediate tree has the very bypass this
MR closes. The compiler-forced refusal arms (§2c) land with the enum in
that same commit for the same reason.

1. `lib, server: the channel-funding VTXO policy and its admission gate` —
   policy type + taproot (combine_keys+cosign_taproot) + encoding + the
   full forced-match inventory (incl. m0021) + the shared-validator
   destination & input gates + the round/offboard input refusals + the
   client destination filter + policy/vector tests + the before-mutation
   gate tests for direct arkoor, Lightning-pay, and receive-claim.
2. `lib: the presigned bridge transaction and its cosign helpers` —
   `channel.rs` + bridge/sighash/musig + `verify_tx` and the sign tests.
3. `server-rpc, bark: optional channel fields on the arkoor cosign
   exchange; supports_channels in ark info` — proto + grouped carriers +
   convert (half-present rejection) + package-preservation tests + server
   `ArkInfo` construction (still `false`) + bark-json field + its
   field-completeness test. Carries the API/changelog note.

## 7. Open items for review

1. Module name `lib/src/channel.rs` (board.rs precedent argues top-level).
2. Whether the gate should instead defer to the captaind MR (MR-2) —
   decided NO: decoding `0x08` without the gate is unsafe, so the gate
   ships in the same *commit* that makes the policy decodable (commit 1).
3. `optional` proto3 presence vs bare fields — `optional` for explicit
   presence (matches `max_vtxo_amount`).
