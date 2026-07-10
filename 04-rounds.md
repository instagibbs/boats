# ARK #4: Rounds

A round lets a set of users atomically forfeit existing VTXOs and receive
freshly issued ones, anchored in a single new on-chain transaction (the
**round transaction**). Rounds restore full exit depth and reset the trust
assumptions accumulated by out-of-round transfers.

The round protocol issues new VTXOs as leaves of a **VTXO tree** of
pre-signed transactions, and exchanges the users' forfeits for an **unlock
preimage** through a hash-lock, making the swap atomic. This hash-locked swap
protocol is referred to as *hArk*.

## Overview

1. The server announces a round attempt (with an anti-replay challenge).
2. Participants submit their inputs (with ownership attestations) and their
   requested outputs. The server replies with an `unlock_hash` whose preimage
   only it knows.
3. The server proposes a VTXO tree covering all requests, funded by output 0
   of an unsigned round transaction. Interactive participants cosign the tree
   branches leading to their leaves.
4. The server combines all signatures, signs and broadcasts the round
   transaction, and publishes the signed tree.
5. Each participant builds its new VTXOs from the tree. The leaf output of
   each new VTXO is locked by `unlock_hash`, so the VTXOs are not yet
   spendable.
6. The participant obtains the server's cosignature on each leaf transaction,
   then sends the server signed **forfeit transactions** for all its inputs.
   These forfeits are themselves only claimable by the server if it reveals
   the unlock preimage.
7. The server returns the unlock preimage; the participant's new VTXOs are
   now complete.

If the server withholds the preimage after receiving forfeits, the user can
publish its (incomplete) exit chain; the server can only claim the forfeited
funds by revealing the preimage on-chain, which simultaneously unlocks the
user's new VTXOs. If the server never reveals it, the user recovers the
forfeited funds through the forfeit output's exit clause.

There are two participation modes sharing steps 5–7:

* **Interactive**: the participant generates a fresh *cosign key* per
  requested leaf and cosigns its tree branches itself (steps 3–4).
* **Delegated**: the participant only submits the request; the server's own
  cosign keys sign the tree. The participant polls for the result. This mode
  requires no liveness during tree signing.

## The VTXO tree

### Tree shape

The tree is a radix-4 tree over `n` leaves (one leaf per requested VTXO),
constructed deterministically from the ordered leaf list:

```
nodes := [leaf 0, leaf 1, ..., leaf n−1]
cursor := 0
while cursor < len(nodes) − 1:
    children := nodes[cursor .. min(cursor+4, len(nodes))]
    append new internal node with those children
    cursor += len(children)
```

The last node appended is the root. Note that later internal nodes can take
both leaves and earlier internal nodes as children. Nodes are indexed by
their position in `nodes` (leaves first, then internal nodes in creation
order); this indexing is used everywhere below. A node's *internal index* is
its index minus `n`. A node's *level* is 0 for leaves and
`1 + max(child levels)` otherwise; its *internal level* is its level minus 1.

### Tree specification (`vtxo_tree_spec`)

All the data needed to derive the unsigned tree:

```
u8:           version (= 2)
u32:          expiry_height
pubkey:       server_pubkey
u16:          exit_delta
vec<pubkey>:  global_cosign_pubkeys
vec<leaf_spec>: vtxos       (MUST be non-empty)
```

where each `leaf_spec` is:

```
policy:          the requested output policy        (ARK #2)
u64:             amount (sats)
option<pubkey>:  cosign_pubkey   (present for interactive participants)
sha256:          unlock_hash
```

`global_cosign_pubkeys` are cosigners that sign *every* internal node — in
rounds this is the server's ephemeral round cosign key; in single-party trees
it is the user's cosign key followed by the server's.

A tree has at least two leaves: the radix-4 construction, the cosign-taproot
funding output, and the leaf-cosign leaf outputs are only jointly well-defined
for `n ≥ 2` (for `n = 1` the root would be the leaf, yet the funding output is
a cosign taproot the leaf transaction cannot spend). A server MAY pad a
round's leaf list to reach this minimum and to fill the tree; the reference
pads to a minimum of two leaves using dummy leaves with an unspendable pubkey,
no cosign key, and an `unlock_hash` for which no preimage is known.
Participants MUST tolerate leaves that are not theirs.

### Aggregate cosign keys

For each node, define its aggregate cosign key:

* leaf `i`: `musig(user_pubkey(leaf i), server_pubkey)` — the internal key of
  the leaf-cosign taproot (ARK #2);
* internal node: `musig(cosign pubkeys of all leaves under the node that have
  one ‖ global_cosign_pubkeys)`.

### Funding output

The tree is funded by a single output (the round transaction's output 0,
`ROUND_TX_VTXO_TREE_VOUT = 0`):

* value: the sum of all leaf amounts
* scriptPubkey: **cosign taproot** `(root aggregate key, S, expiry_height)`
  where the root aggregate key is `musig(all leaf cosign pubkeys ‖
  global_cosign_pubkeys)` (note: over *all* leaves, equal to the root node's
  aggregate key).

The expiry leaf in every node output allows the server to sweep the entire
remaining tree on-chain once the round expires.

### Node transactions

Every node has exactly one transaction. All node transactions have
`nVersion = 3`, `nLockTime = 0`, and a single input with `nSequence = 0`
whose prevout is the parent node's corresponding output (the root spends the
funding output).

* **Internal node tx** — one output per child, in child order, followed by a
  zero-value P2A fee anchor:
  * child is a leaf: the **leaf-cosign taproot** output
    `(user_pubkey(leaf), S, expiry_height, unlock_hash(leaf))` with the leaf
    amount;
  * child is internal: the **cosign taproot** output
    `(child aggregate key, S, expiry_height)` with value equal to the sum of
    the child's own outputs.

  The input is satisfied by key spend with a MuSig2 signature of the node's
  aggregate cosign key (tweaked per its cosign taproot).

* **Leaf tx** — one output paying the requested `policy` with the leaf
  amount, plus a zero-value P2A fee anchor. The input spends the parent's
  leaf-cosign output via the script-spend **unlock clause**
  (`hash-sign(unlock_hash, musig(user, S))`, ARK #2); its witness requires
  both the aggregate signature and the unlock preimage.

Node transactions pay no fee; round fees are collected as the difference
between forfeited input amounts and issued output amounts.

**Fee rule.** The refresh fee is, per the `refresh` entry of the server's
published fee schedule (ARK #0):

```
fee = base_fee + ppm-expiry fee over the input VTXOs
```

The user pays it implicitly, by requesting outputs that sum to less than its
forfeited inputs; the server validates that the shortfall covers the computed
fee (see Requirements below). The check is `inputs − outputs ≥ fee`, not
equality, so a user computing against a slightly stale schedule overpays
rather than being rejected.

### Tree signing (interactive)

Sighashes: for every internal node, the BIP-341 key-spend sighash of its
transaction, with the single prevout being the parent output (or the funding
output for the root).

Each interactive participant generates `nb_round_nonces` MuSig2 nonce pairs
per requested leaf — one for each *internal level* its branch may pass
through (internal level 0 = nodes all of whose children are leaves; see
"Tree shape"). For each internal node, the aggregate nonce is computed
from:

* for each leaf under the node that has a cosign pubkey: that participant's
  nonce at index equal to the node's internal level;
* for each global cosigner: its nonce at index equal to the node's internal
  index.

The participant produces one partial signature per internal node on the
branch from its leaf to the root (ordered leaf to root), each over the node's
sighash with the tweak of the node's cosign taproot. The server MUST verify
every received partial signature (BIP-327 partial verification) before
accepting it, and the participant MUST verify the server's/global cosigners'
partial signatures or, equivalently, the final signatures.

The final per-node signatures are the BIP-327 aggregation of all partial
signatures for that node, valid against the node's taproot output key.

### Signed tree (`signed_vtxo_tree_spec`)

```
u8:              version (= 2)
vtxo_tree_spec:  spec
outpoint:        utxo            (the funding outpoint)
vec<schnorr_sig>: cosign_sigs    (per internal node, leaves-to-root order)
```

(Version 1, accepted for decoding only, encodes the signature count as `u32`
instead of compact_size.)

### Building the output VTXOs

The VTXO for leaf `i` has:

* `amount`, `policy` from the leaf spec; `expiry_height`, `server_pubkey`,
  `exit_delta` from the tree spec; `anchor_point` = the funding outpoint;
* genesis items, root to leaf:
  * for each internal node on the branch (root first): a `cosigned`
    transition with `pubkeys` = the node's cosigners (leaf cosign pubkeys
    under the node followed by the global cosign pubkeys) and the node's
    final signature; `output_idx` = the child index leading toward the leaf;
    `other_outputs` = the node tx's outputs excluding that child output and
    the fee anchor; `fee_amount = 0`;
  * the final item: a `hash-locked-cosigned` transition with the leaf's
    `user_pubkey`, no signature yet, and the `unlock_hash`; `output_idx = 0`,
    no other outputs, `fee_amount = 0`;
* `point` = output 0 of the leaf tx.

Such a VTXO validates structurally ("validate unsigned") but is not complete
until the leaf cosignature and unlock preimage are added (below).

A `channel-funding` leaf (the channel-refresh flow, ARK #8) is built
identically: only the leaf transaction's output policy differs. The
leaf-cosign taproot still uses the leaf's `user_pubkey` (`A`) — the user's
cooperative key, exactly as for any other policy — and the final genesis item is
the same `hash-locked-cosigned` transition.

## Messages

### Round events

The server publishes an ordered event stream; clients subscribe and MUST be
able to (re)synchronise to the latest event. The reference server primes a
new subscription with the most recent event and additionally exposes a query
for the latest round event; events MAY be dropped for a subscriber that lags.

* **`round_attempt`** — `round_seq` (u64), `attempt_seq` (u64),
  `challenge` (32 bytes, fresh random per attempt).
  A round may go through multiple attempts (e.g. when participants drop
  out); each new attempt restarts submission with a new challenge.
* **`vtxo_proposal`** — `round_seq`, `attempt_seq`,
  `vtxos_spec` (`vtxo_tree_spec`), `unsigned_round_tx` (transaction),
  `cosign_agg_nonces` (one aggregate nonce per internal node,
  leaves-to-root).
* **`round_finished`** — `round_seq`, `attempt_seq`,
  `signed_round_tx` (transaction),
  `cosign_sigs` (one signature per internal node, leaves-to-root).
* **`round_failed`** — `round_seq`.

### `submit_payment` (interactive participation)

Sent after observing `round_attempt`:

| Field | Type |
|---|---|
| `input_vtxos` | list of (`vtxo_id`, `attestation`) |
| `vtxo_requests` | list of (`policy`, `amount`, `cosign_pubkey`, `pub_nonces`) |
| `unblinded_mailbox_id` | optional bytes (delivery hint, out of scope) |

Each input's `attestation` is a BIP-340 signature by the input VTXO's user
key over the message:

```
sha256(
  "Ark round input ownership proof "   (32 bytes, ASCII)
‖ challenge                            (32 bytes)
‖ vtxo_id                              (36 bytes)
‖ nb_requests                          (u64, big-endian)
‖ for each request:
    amount                             (u64, big-endian)
  ‖ policy                             (protocol encoding)
  ‖ cosign_pubkey                      (33 bytes)
)
```

This proves ownership of the input and binds it to this attempt's challenge
and to the exact set of requested outputs.

Requirements (server):

* MUST verify each attestation against the input VTXO's user pubkey.
* MUST verify the inputs are known, unspent, not banned, and not already
  being exited on-chain. (There is no explicit expiry check in this path; the
  server's protection against an expired input is its right to sweep the
  backing funds after expiry, see "Sweeping".)
* MUST reject a request with no outputs, and MUST reject a request that names
  the same input VTXO more than once.
* MUST verify the input/output balance covers the refresh fee:
  `inputs − outputs ≥ fee`, with `fee` per the Fee rule above.
* MUST verify each requested policy is a `pubkey` policy — or a
  `channel-funding` policy, accepted only in a *delegated* channel-refresh
  participation (ARK #8), never in an interactive `submit_payment`; HTLC
  policies are not valid round outputs — and each amount is at least
  `P2TR_DUST` (and at most `max_vtxo_amount` if set).
* MUST reject a `cosign_pubkey` already registered by another request in the
  same attempt (cosign pubkeys MUST be unique within a round), and MUST
  accept at most one `provide_vtxo_signatures` per cosign pubkey.
* For an interactive request, MUST verify each requested leaf carries exactly
  `nb_round_nonces` public-nonce pairs (one per internal level its branch may
  traverse).
* MUST NOT let the round's total leaf count exceed `4 ^ nb_round_nonces`: a
  participant only holds nonces for internal levels `0 .. nb_round_nonces`,
  so a radix-4 tree deeper than that cannot be cosigned. A request that would
  overflow the current round is deferred to a later one.
* MUST include each accepted request as a leaf with a fresh `unlock_hash`
  (one per participation; the server knows the preimage).

Response: `unlock_hash` (32 bytes) identifying this participation.

(The reference transport also carries a deprecated `offboard_requests` field
on this message; it MUST be empty — offboards are a separate flow, ARK #7.)

The server then emits `vtxo_proposal`. Requirements (participant):

* MUST verify its requests appear as leaves of the spec with the correct
  policy, amount, and cosign pubkey, and that `server_pubkey`, `exit_delta`,
  and `expiry_height` are acceptable (including the bounds of ARK #1).
* MUST verify output 0 of `unsigned_round_tx` equals the tree funding output
  implied by the spec. The cosign sighashes use the spec-implied funding
  output as prevout, so signatures over a mismatched round transaction are
  merely useless; the check MUST happen no later than validating the output
  VTXOs against the signed round transaction (the chain-anchor check of
  ARK #2 enforces it then).
* MUST cosign the branch of each of its leaves and respond with one
  `provide_vtxo_signatures` per cosign key:
  (`cosign_pubkey`, partial signatures leaf-to-root).
* SHOULD discard the cosign keys after the round.

On `round_finished`, the participant:

* MUST verify every cosign signature against the corresponding node sighash
  and taproot output key.
* MUST verify the signed round transaction matches the proposal: same txid,
  i.e. identical except for witness data.
* MUST build and structurally validate its output VTXOs.
* SHOULD wait for the round transaction to confirm to its required depth
  before forfeiting (see below); the server MAY require forfeits earlier.

### `submit_round_participation` (delegated participation)

| Field | Type |
|---|---|
| `input_vtxos` | list of (`vtxo_id`, `attestation`) |
| `vtxo_requests` | list of (`policy`, `amount`) |
| `unblinded_mailbox_id` | optional bytes |

The attestation message differs from the interactive one (no challenge, no
cosign keys):

```
sha256(
  "hArk round join ownership proof "   (32 bytes, ASCII)
‖ vtxo_id                              (36 bytes)
‖ nb_requests                          (u64, big-endian)
‖ for each request:
    amount                             (u64, big-endian)
  ‖ policy                             (protocol encoding)
)
```

Response: `unlock_hash`. The server queues the participation for a following
round and signs the tree on the participant's behalf: the resulting leaves
carry no `cosign_pubkey` (absent in the leaf spec), so only the global
cosign key signs their branches.

Requirements (server): the policy-type, dust, and `max_vtxo_amount` admission
checks listed for `submit_payment` above apply here too — they are not specific
to the interactive path. In particular, this delegated path is the **only** one
in which a `channel-funding` output policy is admitted (ARK #8): the server MUST
accept a `channel-funding` request here and MUST reject one on the interactive
`submit_payment`. For a `channel-funding` request the server MUST additionally
verify that `input_vtxos` include the current backing VTXO of one of its
channels, rejecting **at participation time** (not deferring to the later leaf
cosign) otherwise. Accepting such a participation also establishes the
channel's exact `(unlock_hash, channel_id, old_backing_vtxo_id)` refresh gate —
one admission decision, never one without the other; ARK #8 "Refresh admission
and the server gate" defines the binding, single-use, and recovery
requirements. HTLC policies are not valid round outputs,
and each requested amount MUST be at least `P2TR_DUST` (and at most
`max_vtxo_amount` if set).

### `round_participation_status`

Request: `unlock_hash`. Response:

| Field | Type | Presence |
|---|---|---|
| `status` | enum: `pending`, `issued`, `released` | always |
| `input_vtxo_ids` | list of vtxo_id | always |
| `round_funding_tx` | transaction | unless `pending` |
| `output_vtxos` | list of VTXO (full encoding, incomplete) | unless `pending` |
| `unlock_preimage` | 32 bytes | only when `released` |

Requirements (participant): on `issued`, it MUST validate the returned VTXOs
against its requests and check the funding transaction's confirmation before
proceeding to forfeit. Because a delegated participant did not cosign the tree
itself, this validation MUST verify every internal `cosigned` signature: full
ARK #2 validation must succeed for every transition and fail only at the final
`hash-locked-cosigned` item (which is still missing its leaf cosignature and
preimage). The weaker structural "validate unsigned" check is not sufficient
here, as it would let the participant forfeit against a tree with invalid
signatures. On `released` it re-runs the completion steps below idempotently.

### `leaf_vtxo_cosign` (completion step 1)

For each new VTXO, the participant computes the **leaf sighash**: the BIP-341
script-spend sighash of the leaf transaction for the unlock-clause leaf
(`hash-sign(unlock_hash, musig(A, S))`), with the parent's leaf-cosign output
as prevout.

Request: `vtxo_id`, user `pub_nonce` (bound to the leaf sighash).
Response: server `pub_nonce`, server `partial_sig` (first-signer one-shot,
ARK #0).

The participant combines both partial signatures into the final aggregate
signature, MUST verify it against the *untweaked* aggregate key
`musig(A, S)` (the script-path key), and stores it in the VTXO's final
`hash-locked-cosigned` transition.

### `request_forfeit_nonces` (completion step 2)

Request: `unlock_hash`, list of input `vtxo_id`s to forfeit.
Response: one server `pub_nonce` per input.

### `forfeit_vtxos` (completion step 3)

The **forfeit transaction** for an input VTXO `v` and unlock hash `H`:

* `nVersion = 3`, `nLockTime = 0`
* input: `v.point`, `nSequence = 0xFFFFFFFF`, key-spend witness for `v`'s
  output taproot (i.e. cooperatively signed by `musig(user, S)` tweaked with
  `v`'s policy taproot tweak)
* output 0: value `v.amount`, paying the **hark-forfeit** policy taproot
  (ARK #2): internal key `musig(user, S)`; leaves
  hash-sign `(H, S)` and delayed-sign `(v.exit_delta, user)`
* output 1: zero-value P2A fee anchor

The participant sends one **forfeit bundle** per input:

```
u8:                version (= 1)
vtxo_id:           input vtxo id        (36 bytes)
sha256:            unlock_hash
musig_pub_nonce:   user public nonce
musig_partial_sig: user partial signature
```

The partial signature is over the forfeit transaction's key-spend sighash,
with the aggregate key `musig(user, S)` tweaked by the input VTXO's output
taproot tweak.

Requirements (server):

* MUST verify each bundle's partial signature before responding.
* MUST verify all bundles cover all inputs of the participation and use the
  participation's `unlock_hash`.
* MUST respond with the 32-byte `unlock_preimage`, and from that moment MUST
  consider the inputs forfeited and the new VTXOs released.
* MUST answer repeated completion requests for an already-released
  participation idempotently (returning the preimage). A failed
  `forfeit_vtxos` attempt consumes the server's nonces; the participant
  restarts from `request_forfeit_nonces`.

Requirements (participant):

* MUST verify `sha256(unlock_preimage) = unlock_hash`, then store the
  preimage in each new VTXO's final transition and fully validate the VTXOs
  (ARK #2 validation, including signatures).
* MUST NOT send forfeit bundles for a round it has not verified (signed tree
  and round transaction as above).
* If the server accepts the forfeits but never releases the preimage, the
  participant MUST start an emergency exit (ARK #6) of its new VTXOs before
  expiry. Publishing the exit chain forces the resolution described in the
  Overview.

## Failure handling

* On `round_failed`, or on a new `round_attempt` with a higher
  `attempt_seq`, participants discard attempt state (nonces, partial
  signatures) and MAY re-submit. Secret nonces MUST never be reused across
  attempts. The server restricts later attempts of the same round to the
  inputs registered in the previous attempt whose cosigner went on to deliver
  signatures: a participant that dropped out (withheld its tree signatures)
  has its inputs excluded from, and banned for, the remaining attempts, and a
  fresh participant cannot join a later attempt of an in-progress round.
* The funding transactions of successive attempts of a round share a common
  input, so at most one attempt's round transaction can ever confirm. hArk
  forfeit transactions are exchanged only after a specific attempt's
  `round_finished` (completion step 3) and are released to the server only
  against that attempt's `unlock_preimage`; an attempt that never finishes
  therefore never reaches the forfeit stage, and a stale attempt's tree can
  never materialize on-chain. (Unlike the connector-bound forfeits of ARK #7,
  hArk forfeits are not tied to any output of the round transaction.)
* A participant that has *not* yet sent forfeit bundles can always abandon a
  round unilaterally; its inputs remain its own.
* A participant that *has* sent forfeit bundles MUST persist the round state
  until either its new VTXOs are complete and the round transaction is
  confirmed, or it has exited unilaterally.
* If the server reports an unknown participation after forfeits were sent,
  the participant MUST treat this as adversarial and exit.

## Sweeping (informative)

After `expiry_height`, the server can reclaim any unspent part of the tree
on-chain via the timelock-sign expiry leaves present in the funding output
and every node output, and can claim published forfeit outputs by revealing
the corresponding unlock preimage. Connector-based forfeits, which bind
forfeits to a specific on-chain transaction rather than to a preimage, are
used by the offboarding protocol (ARK #7).
