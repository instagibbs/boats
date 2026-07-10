# ARK #7: Offboarding

Offboarding moves funds from the Ark protocol back on-chain cooperatively:
the server pays the user's on-chain destination from its own wallet, and the
user forfeits the input VTXOs to the server. Unlike an emergency exit
(ARK #6), an offboard costs a single on-chain output and needs no waiting
period — but it requires a cooperating server.

The mechanism is a **connector swap**: the user's forfeit transactions each
require a *connector* output that only exists if the server's offboard
transaction is in the chain. The forfeits the server collects are therefore
only valid in a history where the user has been paid, making the swap atomic
without hash locks or round participation.

## Overview

1. The user sends an offboard request (destination, amount, fee rate) plus
   the input VTXO ids and ownership attestations.
2. The server builds an unsigned **offboard transaction** from its on-chain
   wallet, paying the destination and creating a connector output, and
   returns it together with one forfeit-cosign nonce per input.
3. The user validates the transaction, signs one **forfeit transaction** per
   input (each spending that input VTXO *and* a connector), and returns its
   nonces and partial signatures.
4. The server verifies and completes the forfeit signatures, records the
   completed outcome (see `finish_offboard`), then broadcasts and returns
   the signed transaction to the user.
5. The user verifies the signed transaction has the same txid it forfeited
   against, and broadcasts it as well.

If the server aborts before step 4, the user has lost nothing: the forfeits
spend a connector that will never exist. If the server broadcasts the
offboard transaction, the user is paid on-chain regardless of any later
server behavior.

## Offboard request

| Field | Type | Meaning |
|---|---|---|
| `script_pubkey` | script | the on-chain destination |
| `net_amount` | u64 sats | the amount the destination output will carry |
| `deduct_fees_from_gross_amount` | bool | whether fees come out of the input total (true) or were added on top by the user (false) |
| `fee_rate` | sat/kWU | the fee rate the user used to compute fees |

The destination output (`script_pubkey`, `net_amount`) MUST be a standard,
non-dust output: a P2PKH, P2SH, P2WPKH, P2WSH, or P2TR output at or above its
script type's dust threshold (546, 540, 294, 330, and 330 sats respectively),
or a bare `OP_RETURN` of at most 83 bytes. The reference rejects all other
script types (bare multisig, pay-to-anchor, unknown witness versions).

The user SHOULD obtain the server's current offboard fee rate through the
dedicated query — a request that returns a single fee rate in sat/kvB — and
use it as `fee_rate`; note that the published rate is denominated in sat/kvB
while the request field is sat/kWU (the conversion is
`sat/kWU = sat/kvB ÷ 4`). The server accepts a requested rate that does not
exceed any regular-target estimate it published within a recent window
(15 minutes in the reference implementation), and rejects higher rates.

**Fee rule.** The offboard fee is, per the `offboard` entry of the server's
published fee schedule (ARK #0):

```
fee = base_fee
    + fee_rate * (fixed_additional_vb + len(script_pubkey) in vb)
    + ppm-expiry fee over the input VTXOs
```

The weight component rounds up (ceiling division of the per-kWU product);
each ppm term rounds down. The ppm-expiry fee walks the input VTXOs in
input order, charging each VTXO's amount — capped by what remains of a
chargeable total: the gross input sum when `deduct_fees_from_gross_amount`
is true, `net_amount` otherwise — at the ppm rate selected by the VTXO's
remaining blocks until expiry.

The server MUST verify that the sum of the input VTXO amounts equals
`net_amount` plus the fee *exactly*; any surplus or deficit is rejected, so
an implementation must reproduce the fee computation (including rounding)
bit-for-bit.

## Transactions

### Offboard transaction

Built and funded by the server's wallet. Its required outputs:

* output 0 (`OFFBOARD_TX_OFFBOARD_VOUT`): the destination output —
  `net_amount` to `script_pubkey`;
* output 1 (`OFFBOARD_TX_CONNECTOR_VOUT`): the **connector output** —
  value `n * P2TR_DUST` (where `n` is the number of input VTXOs), paying a
  key-spend-only P2TR for a fresh per-offboard *connector key* held by the
  server.

Further outputs (change) and all inputs are at the server's discretion.

Requirements (user): before signing any forfeit, the user MUST verify
output 0 matches its request exactly and output 1's value is exactly
`n * P2TR_DUST`. The connector scriptPubkey is not otherwise verifiable —
the connector key is the server's — so the user simply adopts output 1's
scriptPubkey when deriving the fanout transaction and the forfeits.

### Connector fanout transaction (`n > 1`)

When there is more than one input VTXO, the single connector output is split
into one connector per input by a deterministic fanout transaction:

* `nVersion = 3`, `nLockTime = 0`
* one input: offboard transaction output 1, `nSequence = 0xFFFFFFFF`,
  key-spend witness (connector key)
* `n` outputs of `P2TR_DUST` each, all paying the connector scriptPubkey
* a zero-value P2A fee anchor

Connector `i` (for input VTXO `i`) is output `i` of this transaction. When
`n = 1`, the offboard transaction's output 1 is used as the connector
directly and no fanout transaction exists. Both sides derive the fanout
transaction independently from the unsigned offboard transaction; it is
never exchanged.

### Forfeit transactions

One per input VTXO `v`, with connector outpoint `c` as derived above:

* `nVersion = 3`, `nLockTime = 0`
* input 0: `v.point`, `nSequence = 0xFFFFFFFF`, key-spend witness — the
  2-of-2 MuSig2 signature of `musig(user, S)` tweaked with `v`'s output
  taproot tweak
* input 1: `c`, `nSequence = 0xFFFFFFFF`, key-spend witness — signed by the
  server's connector key alone
* output 0: value `v.amount + P2TR_DUST` (the forfeited funds plus the
  consumed connector dust), paying a key-spend-only P2TR for the server key
* output 1: zero-value P2A fee anchor

Both signatures use the BIP-341 key-spend sighash (`SIGHASH_DEFAULT`) with
both prevouts provided. The prevouts are `v`'s output and the connector
prevout: for `n = 1` this is the offboard transaction's output 1; for
`n > 1` it is the fanout output actually being spent — value `P2TR_DUST`,
paying the connector scriptPubkey. Signers MUST NOT use the offboard
transaction's combined connector output (value `n * P2TR_DUST`) as the
prevout when `n > 1`: the sighash commits to all prevout amounts, and a
signature committing to the combined value is consensus-invalid when the
forfeit spends the fanout output.

The user signs first, first-signer one-shot (ARK #0), having received the
server's nonces in `prepare_offboard`. The server completes each aggregate
signature and MUST verify it against `v`'s taproot output key before
treating the input as forfeited; it MUST NOT sign or broadcast the offboard
transaction before holding valid forfeit signatures for all inputs.

## Messages

### `prepare_offboard`

| Field | Type |
|---|---|
| `offboard` | the offboard request (above) |
| `input_vtxo_ids` | list of vtxo_id |
| `attestations` | one schnorr_sig per input, in input order |

Each attestation is a BIP-340 signature by the corresponding input VTXO's
user key over the message:

```
sha256(
  "Ark offboard request challenge  "    (32 bytes, ASCII; two trailing spaces)
‖ destination txout                     (Bitcoin consensus encoding:
                                         value ‖ script length ‖ script)
‖ nb_inputs                             (u32, little-endian)
‖ each input vtxo_id                    (36 bytes)
)
```

Note the message is the same for every input; each input's key signs it
separately.

Requirements (server):

* MUST verify every attestation against the corresponding input's user
  pubkey, and that the inputs are known, unspent, and not banned. (There is
  no explicit expiry check in this path; the server's protection against an
  expired input is its right to sweep the backing funds after expiry, though
  expiry does also enter the fee via the ppm-expiry component.)
* MUST reject a request that names the same input VTXO more than once, and
  MUST reject a request whose attestation count does not equal its input
  count.
* MUST validate the request (standardness, fee rate freshness, fee rule
  above). MAY reject a destination `script_pubkey` that matches a
  server-configured address blocklist.
* MUST generate one fresh MuSig2 nonce per input, in input order.
* Locks the input VTXOs for the lifetime of the pending offboard session and
  rejects a `prepare_offboard` whose inputs overlap an already-pending
  session, so a client MUST complete or abandon one offboard of a given input
  before preparing another.

Response:

| Field | Type |
|---|---|
| `offboard_tx` | the unsigned offboard transaction (consensus encoding) |
| `forfeit_cosign_nonces` | one musig_pub_nonce per input, in input order |

### `finish_offboard`

Sent by the user after validating the offboard transaction and signing the
forfeits:

| Field | Type |
|---|---|
| `offboard_txid` | txid of the unsigned offboard transaction |
| `user_nonces` | one musig_pub_nonce per input |
| `partial_signatures` | one musig_partial_sig per input |

Response:

| Field | Type |
|---|---|
| `signed_offboard_tx` | the fully signed offboard transaction |

`finish_offboard` is idempotent by `offboard_txid`. The server MUST serialize
processing for a prepared session: exactly one request may consume its MuSig2
secret nonces, and a concurrent request receives an explicit *in progress*
result rather than being mistaken for an unknown session. This prevents using
one server nonce in two signing sessions. A completed retry does not sign
again; it returns the already stored outcome below.

A `finish_offboard` the server has answered with success — or whose offboard
transaction it has broadcast — is **completed**, and completion survives
restarts: a byte-identical retry MUST return the same `signed_offboard_tx`,
however much later it arrives, and a broadcast or wallet failure is retried
from the completed outcome, never by preparing a new offboard. The server MUST
retain the outcome — the signed offboard transaction, the connector-bound
forfeits, and the input set — until the input VTXOs and payout are resolved.
(In practice: the outcome commits before the server broadcasts, replies, or
discards session state.) A prepared session with no completed outcome and no
request in progress MAY expire after a timeout.

*Completed* and *not accepted* are mutually exclusive, and remain so across
restarts. A finish attempt that fails before completing MUST leave the server
with no actionable forfeit from that attempt. Once no request is in progress,
the server MAY answer an unknown `offboard_txid` with an authenticated,
definitive **not accepted** result — distinct from a transport error and from
*in progress* — which asserts that the finish never completed and no forfeit
from it will ever be used, so the client may discard that finish intent and
return to `Prepared`. A server MUST NOT be able to reach a state in which it
emits *not accepted* for an offboard it in fact completed.

Requirements (user):

* From its first attempt to send `finish_offboard`, and across any crash or
  restart, the user MUST remain able to replay that exact request (same
  `offboard_txid`, nonces, and partial signatures) and to recognize its payout.
  Once sending may have occurred, a missing response is an **unknown
  outcome**, not a failed offboard: the user MUST replay only that
  byte-identical request and monitor the known `offboard_txid`. It MUST NOT
  start a new `prepare_offboard` for those inputs or attempt another spend of
  them.
* The sole exception is an authenticated, definitive *not accepted* result as
  defined above. Only that result permits the user to clear the finish intent
  and return to `Prepared`; timeout, disconnection, or an unknown transport
  outcome does not.
* MUST verify the signed transaction's txid equals the txid it forfeited
  against; a different txid would invalidate the connector-bound forfeits
  while still spending nothing of the user's — but the user MUST NOT treat
  the offboard as complete in that case.
* SHOULD broadcast the signed transaction itself and verify its own mempool
  accepts it (protecting against a server double-spending its inputs).
* MUST consider the input VTXOs spent from the moment its finish intent is
  eligible to be sent. Before that transition, an abandoned prepared session
  has sent no forfeit and the user may safely choose another offboard or a
  unilateral exit.
* SHOULD track the transaction to its required confirmation depth before
  considering the offboard final.

## Server sweeping (informative)

After the offboard confirms, the server holds, per input, a fully signed
forfeit transaction it can broadcast if the user ever attempts to use the
forfeited VTXO (e.g. by publishing its exit chain). The connector outputs
exist solely to bind those forfeits to the offboard transaction; once the
input VTXOs have expired they are no longer needed, and the server sweeps
the connector dust after expiry plus a safety margin (144 blocks in the
reference implementation) — each per-input connector after its own input's
expiry, and the combined fanout root after the latest input expiry.
