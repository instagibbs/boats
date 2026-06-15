# ARK #3: Boarding

Boarding moves on-chain funds onto the Ark protocol. The user funds an
on-chain output shared with the server, and the server cosigns a single exit
transaction that gives the user a VTXO of depth 1.

## Overview

```
   funding tx (user's on-chain wallet)
        │
        ▼
   funding output: cosign taproot (musig(A,S), S, expiry)     [chain anchor]
        │  (cosigned exit transaction, held off-chain)
        ▼
   VTXO output: pubkey policy (A), value = amount − fee
   P2A anchor: value = fee
```

The funding output's key path requires both parties, so neither can spend it
alone before expiry. The exit transaction is signed by both parties *before*
the funding transaction is broadcast, so the user never has funds locked in
an output it cannot recover.

## Transactions

### Funding output

The user constructs a transaction (any shape) with an output paying
`amount` to the **cosign taproot** (ARK #2):

* internal key: `musig(A, S)`
* leaf: timelock-sign `(expiry_height, S)`

where `expiry_height` is chosen by the user (typically current tip +
`vtxo_expiry_delta`) and `A` is a fresh user pubkey.

### Exit transaction

The exit transaction is the standard exit transaction of ARK #2 with:

* input: the funding outpoint, `nSequence = 0`, key-spend witness
* output 0: the `pubkey` policy output for `A`, value `amount − fee`
* output 1: P2A fee anchor, value `fee`

The resulting VTXO's `point` is output 0 of this transaction
(`BOARD_FUNDING_TX_VTXO_VOUT = 0`).

**Fee rule.** `fee` is the board fee charged by the server, taken from the
`board` entry of its published fee schedule (the `fees` parameter in ark
info, ARK #0):

```
fee = max(min_fee, base_fee + ppm * amount)
```

It is not carried in either message. Both parties compute it from `amount`
alone — the user when constructing the exit transaction, the server when
reconstructing it from the request fields — so the request need not include
it. The fee is therefore never negotiated: if the two sides used different
schedules they would derive different exit sighashes, and the server's
cosignature would simply fail to verify on the user's side (see Requirements,
below). Boarding has no separate fee-rejection step, unlike refresh (ARK #4)
and offboard (ARK #7), where the server validates the fee and rejects a
mismatch.

## Messages

### `board_cosign_request`

Sent by the user to the server:

| Field | Type | Meaning |
|---|---|---|
| `amount` | u64 sats | value of the funding output (gross, before fee) |
| `utxo` | outpoint | the funding outpoint |
| `expiry_height` | u32 | chosen expiry height |
| `user_pubkey` | pubkey | `A` |
| `pub_nonce` | musig_pub_nonce | the user's MuSig2 public nonce |

The user's nonce MUST be generated bound to the exit transaction's key-spend
sighash as the MuSig2 session message and to the tweaked aggregate key
`tweak(musig(A, S), taptweak)` where `taptweak` is the BIP-341 tweak of the
funding taproot.

Requirements (server):

* MUST reconstruct the exit transaction from the request fields and compute
  the same sighash.
* MUST refuse if `amount − fee` is zero or negative, if `amount` is below
  `min_board_amount` (or below `P2TR_DUST`), or above `max_vtxo_amount` if
  set.
* MUST validate `expiry_height` against its accepted range: no lower than
  the current chain tip and no higher than the tip plus `vtxo_expiry_delta`
  (plus a small buffer for tip skew; the reference allows 3 blocks).
* MUST respond with a `board_cosign_response` produced first-signer one-shot
  (ARK #0): a fresh nonce seeded with the exit sighash and an immediate
  partial signature, the secret nonce never stored, so that repeated
  requests cannot induce nonce reuse.

### `board_cosign_response`

| Field | Type | Meaning |
|---|---|---|
| `pub_nonce` | musig_pub_nonce | the server's public nonce |
| `partial_sig` | musig_partial_sig | the server's partial signature |

Requirements (user):

* MUST verify the server's partial signature against the exit sighash, the
  tweaked aggregate key, and the aggregate nonce before signing.
* MUST then produce its own partial signature and combine both into the final
  BIP-340 signature; it SHOULD additionally verify the final signature
  against the funding taproot output key.
* MUST NOT broadcast the funding transaction before holding a valid final
  exit signature.

The completed VTXO has a genesis of exactly one item:

```
transition:    cosigned, pubkeys = [A, S], signature = final sig
nb_outputs:    1
output_idx:    0
other_outputs: []
fee_amount:    fee
```

### `register_board_vtxo`

After the funding transaction has reached `required_board_confirmations`
confirmations, the user sends the completed VTXO (full encoding, ARK #2) to
the server.

Requirements (server):

* MUST validate the VTXO (ARK #2 validation) against the funding
  transaction.
* MUST verify the VTXO was built as in the board flow: exactly one genesis
  item, a matching server pubkey, and a recomputed exit txid and `point`
  matching the board construction. (The transition kind itself need not be
  inspected; any encoding that reconstructs the same exit transaction is
  equivalent.)
* MUST reject registration until the funding transaction has
  `required_board_confirmations` confirmations; the user retries later.
* MUST verify the funding output is still unspent (a spent funding output
  means a unilateral exit is already in progress).
* MUST treat registration idempotently: re-registering an
  already-registered VTXO succeeds without effect.

Requirements (user):

* SHOULD NOT treat the VTXO as spendable until the funding transaction has
  `required_board_confirmations` confirmations.
* MUST track the expiry height and refresh or exit before it.

## Security notes

* Because the server signs first-signer one-shot (ARK #0) — a fresh nonce
  per response, never stored or reused — it can safely serve repeated
  identical requests; the responses differ, but each is independently valid.
* The server learns the funding outpoint before it confirms; an aborted board
  (funding never broadcast) costs neither party anything since the cosigned
  exit transaction spends an output that never exists.
* If the server disappears after cosigning but before registration, the user
  exits unilaterally via the exit transaction (ARK #6).
