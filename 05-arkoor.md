# ARK #5: Out-of-round (arkoor) Payments

An out-of-round transaction transfers a VTXO's funds to new VTXOs instantly,
with no on-chain footprint and no round. The sender and the server cosign new
virtual transactions extending the input VTXO's genesis chain.

Arkoor transfers are weaker than in-round transfers: the recipient must trust
that the server did not collude with the sender to double-sign the input.
Recipients SHOULD refresh received arkoor VTXOs in a round (ARK #4).
Each transfer also deepens the exit chain; the server MUST refuse to cosign
when an input VTXO's exit depth has already reached `max_vtxo_exit_depth`
(ARK #2).

## Checkpoint transactions

Naively, each arkoor transfer would append one transaction spending the input
VTXO directly to the recipients' outputs. This has two problems: a recipient
that partially exits forces the server to publish the whole shared chain to
claim forfeits, and one recipient's exit pollutes every co-recipient's chain.

Instead, each transfer routes through a **checkpoint transaction**. Every
checkpoint output uses the `checkpoint` policy (ARK #2):

* internal key: `musig(sender, S)` — used to cosign the follow-up transaction
* leaf: timelock-sign `(expiry_height, S)` — server sweeps after expiry

If a recipient exits, the server broadcasts only the checkpoint transaction;
other recipients' VTXOs are then anchored by the checkpoint outputs, which
the server can still sweep after expiry. Use of the checkpoint is signalled
per part (`use_checkpoint`) and MUST be uniform across all parts of a
package; servers SHOULD require checkpoints (the reference server rejects
`use_checkpoint = false`).

## Transactions

A single-input arkoor transfer takes one input VTXO and a list of
destinations, each `(total_amount, policy)` with a user-facing policy
(ARK #2). Destinations are split into *normal outputs* (`m ≥ 1`) and
*isolated outputs* (`k ≥ 0`, used for dust, see below).

All transactions below have `nVersion = 3`, `nLockTime = 0`, single input
with `nSequence = 0`, and a zero-value P2A fee anchor as last output. They
are held off-chain (becoming part of the new VTXOs' genesis chains) and pay
no fees.

**Balance rule**: the sum of all destination amounts MUST exactly equal the
input VTXO's amount. (Exact equality is required for the transactions to be
standard with an ephemeral zero-value anchor.)

### With checkpoint (`use_checkpoint = true`)

* **Checkpoint tx** — spends the input VTXO's `point` by key spend of the
  input's policy taproot (an `arkoor` transition, ARK #2). Outputs, in
  order: one checkpoint-policy output per normal destination (value
  `total_amount`), then if `k > 0` one combined checkpoint-policy output of
  value `Σ isolated amounts` (the *dust isolation output*), then the fee
  anchor. All checkpoint outputs share the same scriptPubkey
  (checkpoint policy of the *sender's* user pubkey).
* **Arkoor txs** — one per normal destination `j`: spends checkpoint output
  `j` by key spend (an `arkoor` transition over the checkpoint policy
  taproot), with a single output paying destination `j`'s policy and amount,
  plus the fee anchor. The new VTXO's `point` is output 0.
* **Isolation fanout tx** (only if `k > 0`) — spends the dust isolation
  output by key spend, with one output per isolated destination (policy and
  amount), plus the fee anchor.

### Without checkpoint (`use_checkpoint = false`)

* **Arkoor tx** — a single transaction spending the input VTXO's `point` by
  key spend, with one output per normal destination (policy, amount), then
  if `k > 0` one combined checkpoint-policy dust isolation output, then the
  fee anchor. New VTXO `point`s are the corresponding output indices.
* **Isolation fanout tx** as above, spending the dust isolation output.

### Dust isolation

Outputs below `P2TR_DUST` (330 sats) cannot stand alone on-chain. When a
transfer mixes dust and non-dust destinations, the dust destinations MUST be
moved behind a single combined output (which MUST itself be ≥ `P2TR_DUST`)
and fanned out by the isolation fanout transaction, whenever isolation is
possible. When the dust sum is already ≥ `P2TR_DUST`, the combined output
stands on its own. When the dust sum is below `P2TR_DUST`, isolation requires
borrowing the deficit from a non-dust destination: if the non-dust
destinations total at least `2 * P2TR_DUST`, the sender splits one that can
spare the deficit (leaving at least `P2TR_DUST`) and routes the split-off
piece through the combined output. When the non-dust total is below
`2 * P2TR_DUST` (so any split would itself create new dust), or when no single
destination can spare the deficit, the sender MAY mix without isolation; this
never trips the server's mirror rule below, because no output then reaches
`2 * P2TR_DUST`.

The server enforces the mirror rule per output list (normal and isolated
checked separately): a list mixing an output below `P2TR_DUST` with an
output of at least `2 * P2TR_DUST` MUST be rejected — such a mix is always
avoidable by splitting the larger output.

## Signing

Let the transactions requiring signatures, in order, be:

1. the checkpoint tx (if any),
2. each arkoor tx in destination order (one tx when no checkpoint),
3. the isolation fanout tx (if any).

So `nb_sigs = 1 + m + (k>0 ? 1 : 0)` with a checkpoint, and
`1 + (k>0 ? 1 : 0)` without.

Each signature is a 2-of-2 MuSig2 key-spend signature by the sender's user
key and the server key over the transaction's BIP-341 key-spend sighash
(`SIGHASH_DEFAULT`, single prevout):

* the first signature (spending the input VTXO) uses
  `musig(sender, S)` tweaked with the *input VTXO's* policy taproot tweak;
* every other signature uses `musig(sender, S)` tweaked with the
  *checkpoint policy's* taproot tweak (this also applies to the fanout
  signature in the no-checkpoint case, since the dust isolation output uses
  the checkpoint policy).

The server signs first, first-signer one-shot (ARK #0), so the user
completes the signatures and the server never holds fully signed
transactions the user has not seen.

## Messages

### `arkoor_cosign_request`

One *part* per input VTXO; parts MAY be batched into a package (see below).

| Field | Type |
|---|---|
| `input` | vtxo_id (the server resolves the full VTXO) |
| `outputs` | list of (`total_amount`, `policy`) — normal destinations |
| `isolated_outputs` | list of (`total_amount`, `policy`) |
| `use_checkpoint` | bool |
| `user_pub_nonces` | `nb_sigs` MuSig2 public nonces, in signing order |
| `attestation` | schnorr_sig |

The attestation is a BIP-340 signature by the input VTXO's user key over:

```
sha256(
  "arkoor cosign attestation       "   (32 bytes, ASCII)
‖ vtxo_id                              (36 bytes)
‖ nb_outputs                           (u32, little-endian)
‖ for each output (normal then isolated):
    total_amount                       (u64, little-endian)
  ‖ policy                             (protocol encoding)
)
```

Requirements (server):

* MUST verify the attestation against the input VTXO's user pubkey.
* MUST verify the input exists, is unspent, and is not banned.
* MUST verify the input has an arkoor-compatible policy for this endpoint
  (`pubkey`); HTLC policies are not accepted here. (Other flows — e.g.
  Lightning — reuse the same checkpoint/arkoor transaction machinery with
  HTLC inputs, but the generic arkoor endpoint specified in this document
  accepts only `pubkey` inputs. One further input is admitted through this
  endpoint with no new fields: a `channel-funding` input (ARK #8) is accepted
  exactly when the request is the sanctioned **downgrade split** of a
  recorded completed channel close, verified per ARK #8's downgrade admission
  rules; every other arkoor spend of a `channel-funding` input MUST be
  rejected. Destination *output* policies are not
  otherwise restricted by this endpoint, with one exception: a
  `channel-funding` destination (ARK #8) MUST NOT be cosigned except through
  the ARK #8 channel variant of this request, together with its bridge — a
  `channel-funding` output carries no user spending clause at all, so without
  a cosigned bridge the funds would be stranded behind server cooperation
  until the expiry sweep.)
* MUST reject an input whose own exit transaction has already been seen
  on-chain or in the mempool (an input being exited unilaterally MUST NOT be
  arkoor-spent). There is no explicit expiry check; the server's protection
  against an expired input is its right to sweep the backing funds after
  expiry (ARK #4).
* MUST verify the balance rule and dust rules above.
* MUST verify the input's exit depth has not reached `max_vtxo_exit_depth`.
  (No per-output `max_vtxo_amount` check is needed: outputs cannot exceed
  the input's amount, so the bound is preserved by induction.)
* MUST reconstruct the transactions exactly as specified and respond with a
  cosignature per sighash, produced first-signer one-shot (ARK #0).
* MUST persist the input VTXO as spent *before* signing (signing twice over
  the same input with different outputs is theft-enabling; see Security
  notes). The spent-mark records the part's **operation identity** — the
  input, the ordered normal and isolated destination lists, and
  `use_checkpoint` (plus any extension fields, e.g. ARK #8's `channel_id`); a
  package's identity is the ordered list of its parts'. Retries are judged
  against that identity, not against request bytes: a request for a spent
  input whose identity matches MUST be answered with a **fresh signing
  session** over the same reconstructed transactions — fresh server nonces
  and partials, against the fresh sender nonces of the retry (below) — and
  one whose identity differs MUST be rejected. Re-signing the identical
  transactions under fresh nonces yields only additional signatures over the
  same txids, so no conflicting spend can result. A byte-identical duplicate
  is not a retry but a transport retransmission within the same signing
  session — a conforming retry always carries fresh nonces, so it is never
  byte-identical. A server MUST NOT open a fresh signing session for a
  byte-identical duplicate — pairing the sender's already-used public nonces
  with new server nonces is precisely the cross-session nonce reuse the
  retry rule exists to prevent — it MUST either replay its stored response
  verbatim or reject the duplicate; a sender MUST NOT rely on replay.

### `arkoor_cosign_response`

Per part:

| Field | Type |
|---|---|
| `server_pub_nonces` | `nb_sigs` public nonces, signing order |
| `server_partial_sigs` | `nb_sigs` partial signatures, signing order |

Requirements (sender):

* MUST verify every server partial signature (BIP-327 partial verification
  against the corresponding sighash, tweak, and nonces) before producing its
  own partial signatures.
* On any retry — a lost, timed-out, or unverifiable response — MUST discard
  the session's secret nonces and generate fresh ones: a secret nonce never
  participates in more than one signing session (the nonce discipline of
  ARK #4 failure handling). The operation-identity rule above is what makes
  the retry answerable: the server re-signs the same transactions in a fresh
  session, so a sender that has lost its secret nonces mid-exchange recovers
  by the same path.
* Completes each final signature, embeds them in the new VTXOs' genesis
  items, and delivers the new VTXOs to the recipients (delivery — e.g. the
  mailbox protocol — is out of scope for this document).

### Registering the signed transactions

The server cosigns and persists the input as spent, but holds the resulting
output VTXOs as *unregistered* until it receives the fully signed off-chain
transactions. After completing the signatures the sender MUST upload the
signed virtual transactions (at minimum the checkpoint transaction) to the
server; a recipient SHOULD likewise upload them on receipt. Two consequences
follow for an interoperating implementation:

* An unregistered VTXO is not accepted as the input to a later transfer
  (the server's spendability check rejects it), so a recipient that never
  uploads its chain holds funds the server will refuse to cosign onward.
* The server can only broadcast the checkpoint transaction in response to a
  recipient's exit (Security notes, below) if it holds that signed
  transaction. Until the chain is registered, the checkpoint-sweeping
  guarantee does not hold.

### Packages

A transfer that needs more than one input VTXO is expressed as a *package*:
a list of parts, one per input, each with its own destination set (the
sender splits the logical destinations across inputs so that each part
balances). The package request/response is simply the list of part
requests/responses, and the server MUST treat the package atomically
(cosign all parts or none). Parts MUST reference distinct input VTXOs. The
server MAY bound the number of parts per package (the reference server does
not).

## The new VTXOs

Each output VTXO extends the input VTXO's genesis with:

* with checkpoint — two items:
  1. `arkoor` transition (cosigners `[sender]`, tweak = input policy taproot
     tweak, checkpoint signature); `output_idx` = the destination's
     checkpoint output index; `other_outputs` = the checkpoint tx's other
     outputs (excluding the anchor); `fee_amount = 0`;
  2. `arkoor` transition (cosigners `[sender]`, tweak = checkpoint policy
     taproot tweak, arkoor signature); `output_idx = 0`, no other outputs.
* without checkpoint — one item: `arkoor` transition (input tweak, arkoor
  signature); `output_idx` = the destination index; `other_outputs` = the
  arkoor tx's other outputs.
* isolated destinations get the corresponding two items routed through the
  dust isolation output and the fanout transaction.

The new VTXOs keep the input's `expiry_height`, `server_pubkey`,
`exit_delta`, and `anchor_point`.

Requirements (recipient):

* MUST fully validate received VTXOs (ARK #2) against the chain anchor
  transaction, and verify the chain anchor's confirmation.
* SHOULD check with the server that the VTXO is not already spent, and
  SHOULD verify the exit depth leaves room to spend it.
* SHOULD refresh the VTXO in a round before relying on it long-term, and
  MUST refresh or exit before `expiry_height`.

## Security notes

* **Double-sign trust**: nothing on-chain prevents the server from cosigning
  two conflicting arkoor spends of the same VTXO. The recipient's protection
  is the server's honesty plus the sender's published exit chain; a refresh
  (ARK #4) replaces this with the round guarantee. This is the protocol's
  deliberate trade-off for instant payments.
* **Checkpoint sweeping**: after a recipient's partial exit, the server
  broadcasts the checkpoint transaction; co-recipients' chains remain intact
  and the server's sweep rights (expiry leaves) are preserved. Servers SHOULD
  bound the number of destinations per input (and thereby the checkpoint
  transaction's outputs) to bound their worst-case response cost; the
  reference default is 4.
* The sender learns the recipients' new VTXO ids and policies; there is no
  sender/recipient unlinkability inside one transfer.
