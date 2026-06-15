# ARK #0: Overview and Conventions

This document series specifies the Ark protocol as implemented by Second's
`ark-bitcoin` codebase (server: `captaind`, client: `bark`). The series is
written in the style of the Lightning Network BOLTs: each document is a
normative specification precise enough that an independent party can build a
compatible client or server without reading the reference implementation.

The specifications are transport-agnostic. Messages are specified by their
semantic content and binary object encodings; the reference implementation
carries them over gRPC/protobuf, but any transport that delivers the same
objects is conformant. See each document's "Messages" sections for the
abstract message definitions.

## Table of contents

* [ARK #0: Overview and Conventions](00-overview.md) (this document)
* [ARK #1: Protocol Object Encoding](01-encoding.md)
* [ARK #2: VTXOs, Policies, and the Genesis Chain](02-vtxo.md)
* [ARK #3: Boarding](03-boarding.md)
* [ARK #4: Rounds](04-rounds.md)
* [ARK #5: Out-of-round (arkoor) Payments](05-arkoor.md)
* [ARK #6: Emergency Exit](06-exit.md)
* [ARK #7: Offboarding](07-offboard.md)

The following areas are part of the implemented protocol but are out of scope
for this series at this time: Lightning send/receive (HTLC VTXOs are specified
structurally in ARK #2, but the payment flows are not), the mailbox delivery
protocol, and the Ark address format (see `docs/addresses.md` and
`docs/mailbox.md`).

## Introduction

The Ark protocol is a bitcoin layer-2 system in which a coordinating party,
the *Ark server*, enables fast and cheap off-chain payments between users
while every user retains the ability to unilaterally recover their bitcoin
on-chain without any cooperation (the *emergency exit*).

Funds on the Ark protocol are represented as *VTXOs* (virtual UTXOs). A VTXO
is defined by:

1. a confirmed on-chain output (its *chain anchor*), and
2. a chain of pre-signed off-chain transactions (its *genesis*) that, when
   broadcast, create a real on-chain UTXO controlled by the user.

The user can always broadcast this chain; the protocols below are designed so
that at every intermediate state, broadcasting is safe for honest parties.

There are five ways VTXOs are created and destroyed:

* **Boarding** (ARK #3): a user locks an on-chain UTXO into a 2-of-2 with the
  server and obtains a VTXO.
* **Rounds** (ARK #4): users forfeit existing VTXOs and receive fresh ones
  issued from a new on-chain *round transaction* funding a tree of pre-signed
  transactions. Rounds are used to refresh VTXOs before they expire and to
  acquire VTXOs with a shallower exit.
* **Out-of-round (arkoor) payments** (ARK #5): a user forfeits a VTXO and the
  server cosigns new VTXOs for one or more recipients, instantly and without
  any on-chain footprint, at the cost of growing the exit chain and adding a
  trust assumption until the next refresh.
* **Emergency exit** (ARK #6): a user broadcasts the genesis chain of a VTXO
  and claims the resulting UTXO on-chain.
* **Offboarding** (ARK #7): a user forfeits VTXOs and the server pays the
  user's on-chain destination, atomically bound through a connector swap.

### Trust model

* The server can never steal funds from a user who exits in time. Every VTXO
  carries a complete, fully signed exit chain.
* Every VTXO has an *expiry height*. After expiry, the server may sweep the
  on-chain funds backing the VTXO. Users MUST refresh or exit before expiry.
* VTXOs received out-of-round carry a weaker guarantee: the server cosigned
  the transition, and a malicious server colluding with the sender could have
  double-signed. Recipients SHOULD refresh arkoor VTXOs in a round to restore
  the full guarantee.

## Actors and keys

* **User** (denoted `A` for Alice, `B` for Bob): holds a *user pubkey* per
  VTXO, used both in output policies and to cosign transitions.
* **Server** (denoted `S`): holds the long-lived *server pubkey* published in
  the server info, plus ephemeral *cosign keys* used in round trees.

`musig(K1, K2, ...)` denotes MuSig2 (BIP-327) key aggregation of the listed
keys, with the keys sorted by their 33-byte compressed encodings before
aggregation (KeySort). Where a taproot output is constructed, the aggregate
key serves as internal key and is tweaked per BIP-341 (`taptweak`); script
trees are specified per construction in ARK #2.

## Common parameters

A server publishes its parameters ("ark info"). The parameters relevant to
this series:

| Parameter | Type | Meaning |
|---|---|---|
| `network` | string | bitcoin network the server operates on |
| `server_pubkey` | pubkey | the server key `S` |
| `round_interval` | seconds | the interval between rounds |
| `nb_round_nonces` | u32 | number of cosign nonces provided per requested VTXO in a round (ARK #4) |
| `vtxo_exit_delta` | u16 | relative-timelock delta (blocks) on every exit clause |
| `vtxo_expiry_delta` | u16 | blocks from issuance until a new VTXO expires |
| `max_vtxo_amount` | optional sats | maximum amount of a single VTXO |
| `required_board_confirmations` | u32 | confirmations before a board VTXO is registered |
| `min_board_amount` | sats | minimum board amount the server advertises; the *enforced* minimum is `max(min_board_amount, P2TR_DUST)`, so a server MAY advertise a value below `P2TR_DUST` and clients MUST treat `P2TR_DUST` as the effective floor |
| `max_vtxo_exit_depth` | u16 | maximum genesis chain length the server will extend |
| `fees` | schedule | the fee schedule for boards, refreshes, etc. |

In addition to these static parameters, the server publishes its current
offboard fee rate through a dedicated query (see ARK #7).

### Fee schedule

The `fees` parameter is a *fee schedule*: a set of per-operation entries the
server publishes in ark info. Like the rest of ark info it is conveyed
out-of-band — it is not one of the protocol-encoded objects of ARK #1 — so
its fields are referred to by name here, and a consumer ignores entries for
operations it does not implement. The entries used by this series are `board`
(ARK #3), `refresh` (ARK #4), and `offboard` (ARK #7); a schedule MAY carry
further entries (e.g. for Lightning) outside its scope.

Each operation's **Fee rule**, in its own document, gives the exact formula
that combines these fields; this section defines only their structure, types,
and shared semantics.

**Shared semantics.**

* Amounts (`min_fee`, `base_fee`) are in satoshis.
* A `ppm` is a parts-per-million rate: an integer where `10_000` = 1%.
  Applied to a chargeable amount it yields `floor(amount × ppm / 1_000_000)`.
* `base_fee` is a flat amount added once per operation; `min_fee`, where
  present, is a floor the total is raised to.
* An on-chain `fee_rate` (sat/kWU) is *not* part of the schedule; where an
  operation needs one (offboard) it is supplied per request (ARK #7).

**`ppm_expiry_table`** (used by `refresh` and `offboard`) is a list of
entries, each:

| Field | Type | Meaning |
|---|---|---|
| `expiry_blocks_threshold` | u32 (blocks) | lower bound of an expiry band |
| `ppm` | ppm | rate charged to a VTXO falling in that band |

For a VTXO with `R` blocks until expiry, the applicable entry is the one with
the greatest `expiry_blocks_threshold` not exceeding `R`; if no threshold is
`≤ R`, that VTXO is charged nothing. The table MUST be sorted by
`expiry_blocks_threshold` ascending and its `ppm` values MUST be
non-decreasing along that order. The monotonicity is deliberate: a small
disagreement about the chain tip (hence about `R`) can then only shift a VTXO
to an adjacent band and never lowers its rate, keeping the fee robust to tip
skew.

**Entries.** `board`:

| Field | Type | Meaning |
|---|---|---|
| `min_fee` | sats | floor on the total board fee |
| `base_fee` | sats | flat per-board addend |
| `ppm` | ppm | rate on the board `amount` |

`refresh`:

| Field | Type | Meaning |
|---|---|---|
| `base_fee` | sats | flat per-refresh addend |
| `ppm_expiry_table` | list (above) | expiry-based rate over the input VTXOs |

`offboard`:

| Field | Type | Meaning |
|---|---|---|
| `base_fee` | sats | flat per-offboard addend |
| `fixed_additional_vb` | u64 (vbytes) | vbytes charged on top of the destination output size, at the request's `fee_rate` |
| `ppm_expiry_table` | list (above) | expiry-based rate over the input VTXOs |

## Conventions

* The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", and "MAY" are to
  be interpreted as described in RFC 2119.
* All amounts are in satoshis. All integers in object encodings are
  little-endian unless explicitly stated otherwise (some signature messages
  use big-endian; this is called out where it occurs).
* All transactions created by this protocol that are intended to be held
  off-chain use transaction version 3 (TRUC, BIP-431) and carry a
  pay-to-anchor (P2A) output for fee-bumping. The P2A output is the standard
  output with `scriptPubkey = OP_1 <0x4e73>` (`51024e73`).
* `P2TR_DUST` denotes the dust threshold of a P2TR output: 330 sats.
* `sha256(x)` is the SHA-256 hash; `ripemd160(x)` is RIPEMD-160;
  `hash160(x) = ripemd160(sha256(x))`.
* All signatures are BIP-340 Schnorr signatures. All sighashes are BIP-341
  taproot sighashes with sighash type `SIGHASH_DEFAULT` (0x00) and all
  prevouts provided.
* Several flows have one party produce its MuSig2 cosignature *first-signer
  one-shot*: holding all counterparty public nonces, the party generates a
  fresh nonce (seeding its generation with the sighash message), immediately
  produces its partial signature, and discards the secret nonce. The
  signature is **not** deterministic — repeated identical requests yield
  fresh nonces and different partial signatures — but the pattern is safe to
  serve repeatedly because a secret nonce is never stored or reused.
* "Block delta" is a relative timelock measured in blocks, encoded as `u16`.
  "Block height" is an absolute block height, encoded as `u32`. Bounds on
  these values are specified in ARK #1.
* The server publishes a fee **schedule** (the `fees` parameter above) that
  sets the *protocol fee* for each operation: board (ARK #3), refresh
  (ARK #4), and offboard (ARK #7). Both parties compute an operation's fee
  from this schedule and the operation's own amounts rather than transmitting
  the fee as a standalone field; each operation states its exact **Fee rule**
  and how (or whether) the server validates it. These protocol fees are paid
  to the server and are distinct from the on-chain miner fees carried by every
  transaction's P2A anchor (ARK #2, ARK #6).

## Vocabulary

| Term | Meaning |
|---|---|
| Ark server | the coordinator (do not use "ASP") |
| VTXO | virtual UTXO, the off-chain representation of funds |
| chain anchor | the on-chain output a VTXO's exit chain spends from |
| genesis | the chain of transitions from chain anchor to the VTXO output |
| board | move on-chain funds onto the Ark protocol |
| offboard | cooperatively move funds back on-chain |
| connector | a dust output binding a forfeit to a specific transaction |
| in-round transaction | a transfer executed inside a round |
| out-of-round (arkoor) transaction | a transfer cosigned outside of rounds |
| checkpoint | an intermediate output isolating arkoor transfers (ARK #5) |
| forfeit | the act of granting the server the input VTXO when receiving new ones |
| emergency exit | unilateral recovery of a VTXO on-chain |
| expiry | the height after which the server may sweep a VTXO's backing funds |
| hArk | the hash-locked round participation protocol (ARK #4) |
