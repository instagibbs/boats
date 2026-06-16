# ARK #2: VTXOs, Policies, and the Genesis Chain

This document specifies the structure, encoding, construction, and validation
of VTXOs. Everything else in this series builds on these definitions.

## Definitions

A **VTXO** ("virtual UTXO") represents off-chain funds and consists of:

* `amount` — the value of the VTXO in sats.
* `policy` — the output policy (spending conditions) of the VTXO output.
* `expiry_height` — the absolute block height at which the VTXO expires.
* `server_pubkey` — the server key `S` involved in all transitions.
* `exit_delta` — the relative-timelock delta (blocks) used in exit clauses.
* `anchor_point` — the **chain anchor**: the on-chain outpoint that must be
  confirmed for the VTXO to exist.
* `genesis` — an ordered list of **genesis items** describing the chain of
  pre-signed transactions (**exit transactions**) from the chain anchor to
  the VTXO's own output.
* `point` — the outpoint of the VTXO itself: the relevant output of the last
  exit transaction. The `point` is fully determined by the other fields; it
  serves as identifier and as a checksum of the genesis data.

The **VTXO ID** is the 36-byte encoding of `point`:
`txid (32 bytes, internal byte order) ‖ vout (u32, little-endian)`. Its text
representation is the standard outpoint form `txid:vout`.

The **exit depth** of a VTXO is the number of genesis items (equal to the
number of exit transactions). A server publishes `max_vtxo_exit_depth` and
MUST refuse to cosign transitions spending an input VTXO whose exit depth
has already reached this value. Note this bounds the *input*: because an
input at depth `max_vtxo_exit_depth` or greater is refused, the deepest
acceptable input sits at `max_vtxo_exit_depth − 1`, so a checkpointed arkoor
transfer — which adds two genesis items (ARK #5) — can produce a resulting
VTXO whose depth exceeds the published value by at most one. Clients SHOULD
refresh well before reaching it.

A VTXO whose `point` equals its `anchor_point` has an empty genesis and is a
virtual representation of an on-chain UTXO. A VTXO whose `point` differs from
its `anchor_point` MUST have a non-empty genesis.

## Script primitives

All output constructions below use the following tapscripts. `<n>` denotes a
minimally-encoded script number push. CSV values are encoded as the consensus
sequence value of a block-height relative lock (`Sequence::from_height`); CLTV
values as the absolute block height.

* **delayed-sign** `(delta, PK)`:
  `<delta> OP_CSV OP_DROP <PK> OP_CHECKSIG`
  Spending input MUST set `nSequence` to the height-based relative lock
  `delta`. Witness (script spend): `[<sig>, <script>, <control block>]`.

* **timelock-sign** `(height, PK)`:
  `<height> OP_CLTV OP_DROP <PK> OP_CHECKSIG`
  Spending transaction MUST set `nLockTime = height` (and a non-final
  sequence). Witness: `[<sig>, <script>, <control block>]`.

* **delay-timelock-sign** `(delta, height, PK)`:
  `<height> OP_CLTV OP_DROP <delta> OP_CSV OP_DROP <PK> OP_CHECKSIG`
  Both locktime and sequence requirements above apply.

* **hash-sign** `(H, PK)`:
  `OP_HASH160 <ripemd160(H)> OP_EQUALVERIFY <PK> OP_CHECKSIG`
  where `H` is a SHA-256 hash. The spender reveals the 32-byte preimage `p`
  with `sha256(p) = H`; the script verifies `hash160(p) = ripemd160(H)`.
  Witness: `[<sig>, <preimage>, <script>, <control block>]`.

* **hash-delay-sign** `(H, delta, PK)`:
  `<delta> OP_CSV OP_DROP OP_HASH160 <ripemd160(H)> OP_EQUALVERIFY <PK> OP_CHECKSIG`
  Witness: `[<sig>, <preimage>, <script>, <control block>]`, `nSequence` set
  as for delayed-sign.

All `<PK>` pushes are 32-byte x-only keys. Where a clause names a compressed
public key, its x-only form is used in the script.

`musig(...)` denotes BIP-327 key aggregation with KeySort (keys sorted by
their 33-byte compressed encodings). Taproot outputs are built per BIP-341:
the listed internal key is tweaked with the merkle root of the listed script
leaves. Where two leaves are listed, both are at depth 1 (their branch hash
is order-independent); a single leaf is at depth 0.

### Shared taproot constructions

* **cosign taproot** `(agg_pk, S, expiry_height)` — used by board funding
  outputs (ARK #3) and round-tree internal node outputs (ARK #4):
  * internal key: `agg_pk`
  * leaf: timelock-sign `(expiry_height, S)`

* **leaf-cosign taproot** `(A, S, expiry_height, unlock_hash)` — used by
  round-tree leaf outputs (ARK #4):
  * internal key: `musig(A, S)`
  * leaf 1: timelock-sign `(expiry_height, S)` (the *expiry clause*)
  * leaf 2: hash-sign `(unlock_hash, musig(A, S))` (the *unlock clause*)

## VTXO policies

The policy determines the scriptPubkey of the VTXO output via a taproot
construction parameterized by `(server_pubkey, exit_delta, expiry_height)`.

### User-facing policies

These may appear as the `policy` of a user's VTXO and in VTXO requests.
`pubkey` is the general-purpose policy: it is accepted as a round
(signed-tree) output and as an arkoor-spendable input (ARK #4, ARK #5).
`channel-funding` (the Lightning-channel funding output, ARK #8) is accepted
as a round output only in the channel-refresh flow and is not
arkoor-spendable. The HTLC policies arise only in the Lightning send/receive
flows (out of scope for this series) and are neither valid round outputs nor
arkoor inputs.

#### `pubkey` (type byte `0x00`)

Fields: `user_pubkey` (33 bytes).

* internal key: `musig(A, S)` — cooperative spend path
* leaf: delayed-sign `(exit_delta, A)` — unilateral claim after exit delay

#### `server-htlc-send` (type byte `0x01`)

An HTLC from user to server, used to fund outgoing Lightning payments.

Fields: `user_pubkey` (33) ‖ `payment_hash` (32) ‖ `htlc_expiry` (u32).

* internal key: `musig(A, S)` — cooperative spend/revocation path
* leaf 1: hash-delay-sign `(payment_hash, exit_delta, S)` — server claims
  with the preimage
* leaf 2: delay-timelock-sign `(2 * exit_delta, htlc_expiry, A)` — user
  reclaims after expiry; the doubled delay gives the server time to respond
  with the preimage path

#### `server-htlc-recv` (type byte `0x02`)

An HTLC from server to user, used for incoming Lightning payments.

Fields: `user_pubkey` (33) ‖ `payment_hash` (32) ‖ `htlc_expiry` (u32) ‖
`htlc_expiry_delta` (u16).

* internal key: `musig(A, S)` — cooperative claim (user reveals preimage)
* leaf 1: delay-timelock-sign `(exit_delta, htlc_expiry, S)` — server
  reclaims after expiry
* leaf 2: hash-delay-sign `(payment_hash, htlc_expiry_delta + exit_delta, A)`
  — user claims with the preimage; the longer delay gives the server time to
  use its expiry path if the user exits too late

#### `channel-funding` (type byte `0x08`)

The funding output of a Lightning channel built on Ark (ARK #8). The VTXO's
own output *is* the channel's BOLT-3 funding outpoint.

Fields: `holder_funding_pubkey` (33) ‖ `counterparty_funding_pubkey` (33).

The two fields are the channel's BOLT-3 funding pubkeys. They are pinned to the
channel parties' Ark keys: `holder_funding_pubkey` is the user key `A` and
`counterparty_funding_pubkey` is the server key `S` (ARK #8). As for `pubkey`,
`A` is the client's own key — but here it is the funding key of the channel,
not an exit-leaf key (a `channel-funding` output has no exit leaf). Both pubkeys
are encoded explicitly rather than derived from `(A, S)`: the holder key is a
per-channel client key, and carrying the counterparty key keeps the policy
well-defined if per-channel server keys are introduced later.

* internal key: `musig(holder_funding_pubkey, counterparty_funding_pubkey)` —
  the cooperative 2-of-2; this is both the path the off-chain Lightning
  commitment transaction spends and the path used to forfeit, refresh, or
  offboard the VTXO
* leaf: timelock-sign `(expiry_height, S)` — the server sweeps after expiry

This is structurally the **cosign taproot** `(musig(A, S), S, expiry_height)`,
with `musig(A, S)` = `musig(holder_funding_pubkey, counterparty_funding_pubkey)`
(see "Shared taproot constructions"). Unlike `pubkey`, it carries no user
unilateral-exit clause: a channel VTXO's unilateral exit is timelocked by the
Lightning commitment transaction's input `nSequence` (ARK #8), not by a VTXO
leaf.

> **Target vs. reference.** The `timelock-sign(expiry_height, S)` leaf is the
> *target* construction. The reference implementation currently builds this
> output *keyspend-only* — internal key only, with an empty script tree — on
> both the Ark and the Lightning sides; adding the expiry leaf is the pending
> `asp-leaf-sweep-gap` fix (ARK #8) and changes the funding `scriptPubkey` on
> both sides together. The reference also currently encodes this policy under
> type byte `0x06`; this series assigns it `0x08`, since `0x06` is
> `hark-forfeit` here.

### Server-internal policies

These appear as outputs of intermediate transactions inside a VTXO's genesis
and in server-side accounting. They never appear as the policy of a VTXO
delivered to a user, but users MUST be able to construct them to derive and
validate exit transactions.

#### `checkpoint` (type byte `0x03`)

Fields: `user_pubkey` (33).

* internal key: `musig(A, S)` — used to cosign the follow-up arkoor tx
* leaf: timelock-sign `(expiry_height, S)` — server sweeps after expiry

#### `expiry` (type byte `0x04`)

Fields: `internal_key` (32, x-only).

* internal key: `internal_key` (an aggregate cosign key)
* leaf: timelock-sign `(expiry_height, S)` — server sweeps after expiry

#### `hark-leaf` (type byte `0x05`)

The policy of round-tree leaf outputs; identical in structure to the
leaf-cosign taproot above.

Fields: `user_pubkey` (33) ‖ `unlock_hash` (32).

#### `hark-forfeit` (type byte `0x06`)

The policy of the output of a hash-locked forfeit transaction (ARK #4).

Fields: `user_pubkey` (33) ‖ `unlock_hash` (32).

* internal key: `musig(A, S)`
* leaf 1: hash-sign `(unlock_hash, S)` — server claims by revealing the
  unlock preimage
* leaf 2: delayed-sign `(exit_delta, A)` — user recovers if the server never
  reveals the preimage

#### `server-owned` (type byte `0x07`)

No fields. Key-spend-only taproot with internal key `S` (x-only) and no
script tree.

## Genesis transitions

Each genesis item contains a **transition** describing how its exit
transaction spends the previous output: the policy of the *parent* output and
the witness satisfying it. There are three transition kinds.

### `cosigned` (type byte `0x01`)

Spends a cosign-taproot output via key spend.

Fields:

```
vec<pubkey>:        pubkeys     (the cosigners; MUST be non-empty)
option<schnorr_sig>: signature
```

* Parent output: cosign taproot `(musig(pubkeys), S, expiry_height)`. Note
  `pubkeys` already includes the server's cosign key where applicable; `S`
  appears separately only in the expiry leaf.
* Witness: key spend, `[<signature>]`.
* Signature: BIP-341 key-spend sighash (`SIGHASH_DEFAULT`, all prevouts) of
  the exit transaction, verified against the taproot output key.

This is the transition out of a board funding output (cosigners
`[A, S]`, ARK #3) and out of round-tree internal node outputs (ARK #4).

### `arkoor` (type byte `0x02`)

Spends a taproot output via key spend of a tweaked aggregate key.

Fields:

```
vec<pubkey>:        client_cosigners   (excludes the server key)
taptweak:           tap_tweak
option<schnorr_sig>: signature
```

* Parent output key: `tweak(musig(client_cosigners ‖ S), tap_tweak)` — the
  x-only tweaked aggregate of the client cosigners plus the server key, where
  `tap_tweak` is the BIP-341 tweak of the parent policy's taproot. The `‖`
  denotes set membership only: `musig` applies KeySort (ARK #0), so the
  concatenation order does not affect the aggregate. The parent scriptPubkey
  is `OP_1 <output key>`.
* Witness: key spend, `[<signature>]`.
* Signature: BIP-341 key-spend sighash of the exit transaction, verified
  against the parent output key.

This is the transition used by out-of-round payments (ARK #5): both the spend
of the input VTXO into a checkpoint and the spend of a checkpoint into the
final output are `arkoor` transitions.

### `hash-locked-cosigned` (type byte `0x03`)

Spends a leaf-cosign-taproot output via script spend of the unlock clause.

Fields:

```
pubkey:             user_pubkey
option<schnorr_sig>: signature
unlock:             1 byte: 0x00 ‖ preimage (32 bytes)
                         or 0x01 ‖ unlock_hash (32 bytes)
```

* Parent output: leaf-cosign taproot
  `(user_pubkey, S, expiry_height, unlock_hash)` where
  `unlock_hash = sha256(preimage)` when the preimage is known.
* Witness: script spend of the unlock clause:
  `[<signature>, <preimage>, <unlock script>, <control block>]`.
* Signature: a MuSig2 aggregate signature for `musig(user_pubkey, S)`
  (untweaked — this key is the script-path key, not the taproot output key)
  over the BIP-341 script-spend sighash of the exit transaction for the
  unlock-clause leaf.

This is the final transition of VTXOs issued in rounds (ARK #4). A VTXO with
this transition is incomplete until both the cosignature and the preimage are
known; see ARK #4 for how they are obtained.

## Exit transactions

The exit transaction for genesis item `i` is constructed as:

* `nVersion = 3`, `nLockTime = 0`
* one input:
  * `prevout`: the chain anchor (for `i = 0`) or output `output_idx` of exit
    transaction `i − 1`
  * `scriptSig` empty, `nSequence = 0`
  * witness: per the item's transition (empty if signatures/preimages are
    missing)
* outputs, in order:
  * `other_outputs[0 .. output_idx]`
  * the **next output** (see below) at index `output_idx`
  * `other_outputs[output_idx ..]`
  * a P2A fee anchor with value `fee_amount`

The **next output** of item `i` is:

* if item `i + 1` exists: the parent output implied by item `i + 1`'s
  transition (see "Genesis transitions"), with value equal to the running
  amount (below);
* otherwise (last item): the VTXO's own output — `policy` evaluated with
  `(amount, server_pubkey, exit_delta, expiry_height)`.

**Amount flow.** Define `other_sum(i) = fee_amount(i) + Σ value(other_outputs(i))`.
The value entering item 0 is the chain anchor output's value; the value
entering item `i + 1` is the value entering item `i` minus `other_sum(i)`.
The value entering the last item, minus its `other_sum`, MUST equal the
VTXO's `amount`. Equivalently, the chain anchor value MUST equal
`amount + Σ_i other_sum(i)`. Exit transactions pay no on-chain fee from
their non-anchor outputs; all fees come from the P2A anchor (via its
`fee_amount` and/or CPFP, see ARK #6).

The signed weight of a 1-input, 1-output-plus-anchor exit transaction with a
key-spend witness is 124 vB (`EXIT_TX_WEIGHT`); the P2A anchor output itself
is 13 vB (`FEE_ANCHOR_WEIGHT`) and is spendable with a 1-WU empty witness.

## VTXO encoding

A VTXO is encoded as (version 2):

```
u16:          version (= 2)
u64:          amount (sats)
u32:          expiry_height
pubkey:       server_pubkey
u16:          exit_delta
outpoint:     anchor_point
genesis:      see below
policy:       type byte + fields, see "VTXO policies"
outpoint:     point
```

The genesis section:

```
compact_size: nb_items
per item:
  transition:  type byte + fields, see "Genesis transitions"
  u8:          nb_outputs        (= 1 + len(other_outputs); MUST be >= 1)
  u8:          output_idx
  txout * (nb_outputs - 1): other_outputs
  u64:         fee_amount (sats)
```

`other_outputs` exclude both the transition output itself and the P2A fee
anchor; together with `output_idx` and `fee_amount` they fully determine the
exit transaction.

Version 1 differs only in omitting `fee_amount` (implied zero). Encoders MUST
emit version 2; decoders MUST accept versions 1 and 2 and reject others. The
top-level `expiry_height` and `exit_delta` are subject to the bounds in ARK #1
and are checked at decode time, as are the `htlc_expiry` height and
`htlc_expiry_delta` of the HTLC policies; every `other_outputs` txout is
additionally subject to the `MAX_SCRIPT_PUBKEY_SIZE` bound (ARK #1).

A decoder that retains the genesis (producing a *full* VTXO) MUST reject:

* a non-empty genesis when `point == anchor_point`;
* an empty genesis when `point != anchor_point`;
* a `cosigned` transition with an empty pubkey list;
* a genesis item with `nb_outputs = 0`;
* a genesis item with `output_idx >= nb_outputs`. Otherwise `point` would
  reference a sibling `other_output` or the P2A fee anchor (an
  anyone-can-spend output) rather than the transition's own output, while the
  exit transaction — built by clamping `output_idx` to the placement index —
  would still carry valid signatures. This breaks the invariant that `point`
  is fully determined by the genesis data: a cosigner of a transfer could
  otherwise hand a counterparty a VTXO that passes validation but whose
  `point` does not identify its funds.

A *bare* VTXO is the same encoding with an empty genesis section
(`nb_items = 0`); it identifies a VTXO without carrying its exit data. The
two encodings are interchangeable for readers that ignore the genesis.

## Validation

Upon receiving a VTXO together with its chain anchor transaction, a party
MUST validate it before treating it as funds:

1. The chain anchor transaction's txid MUST equal `anchor_point.txid`, and it
   MUST have an output at `anchor_point.vout`.
2. If the genesis is empty: the anchor output MUST exactly equal the txout
   implied by the VTXO's own policy and amount, and `point` MUST equal
   `anchor_point`.
3. Otherwise: the anchor output MUST exactly equal (value and scriptPubkey)
   the parent output implied by the first genesis item's transition, with
   value `amount + Σ other_sum(i)`. Any arithmetic overflow is invalid.
4. Construct each exit transaction in order. For each item:
   * the running amount MUST NOT underflow;
   * the transition's signature(s) MUST be present and valid for the
     constructed transaction's sighash (as specified per transition kind);
     for `hash-locked-cosigned`, the preimage MUST be present and match.
5. The outpoint `(txid of last exit tx, output_idx of last item)` MUST equal
   `point`.

A structural variant of this validation that skips signature and preimage
checks ("validate unsigned") is used while a VTXO is still being signed
(ARK #4).

Additionally, a party SHOULD verify that the VTXO is *standard*: its own
output and all `other_outputs` of every genesis item must be standard
outputs, so that the exit chain is relayable on the public network.

Finally, the chain anchor transaction MUST be confirmed (to the party's
required depth) before the VTXO is considered final.
