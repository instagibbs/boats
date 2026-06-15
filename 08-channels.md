# ARK #8: Channels

This document specifies Lightning channels built on the Ark protocol: how a
channel is funded by a VTXO, operated, refreshed, closed cooperatively, and
exited unilaterally. It builds on VTXOs and policies (ARK #2), boarding
(ARK #3), rounds (ARK #4), emergency exit (ARK #6), and offboarding (ARK #7).

The channel's payment machinery — the commitment transaction, its outputs and
HTLCs, channel quiescence, and the channel state machine — is that of the
Lightning Network (BOLT-3 and related) and is used essentially unmodified.
This document specifies only the *Ark side*: the VTXO that funds the channel,
and how the channel lifecycle maps onto Ark's existing flows. The Lightning
send/receive payment flows remain out of scope (ARK #0).

## Overview

A **channel VTXO** is an ordinary `musig(A, S)` 2-of-2 VTXO (ARK #2) whose
output is the funding output of a Lightning commitment transaction. The two
channel parties are the user `A` and the server `S`; there is no third party.

Because the funding output's cooperative key is the same `musig(A, S)` as a
`pubkey` VTXO, every *cooperative* Ark operation on a channel VTXO — issuing it
(open), giving it up (forfeit), refreshing it (round), and closing it
(offboard) — is the corresponding standard Ark flow, unchanged. A channel VTXO
differs from a `pubkey` VTXO in only two ways:

* its output carries the `channel-funding` policy (ARK #2) rather than
  `pubkey`: the taproot script tree holds the server's expiry-sweep leaf
  instead of a user delayed-exit leaf; and
* its unilateral exit proceeds *through* the Lightning commitment transaction
  (a force-close), not through a VTXO leaf, because the funds are committed to
  a channel.

The lifecycle:

* **Open** ("Channel open"): the user boards (ARK #3) into a depth-1 channel
  VTXO, then sets up the Lightning channel over its funding output.
* **Operate** ("Operation"): payments flow over the channel per BOLT; the
  funding VTXO sits unchanged.
* **Refresh** ("Refresh"): before expiry, a round (ARK #4) forfeits the
  channel VTXO and issues a fresh one, resetting expiry and exit depth while
  preserving the channel balance.
* **Offboard** ("Offboard"): a cooperative close — an offboard (ARK #7)
  forfeits the channel VTXO and pays the user's balance on-chain.
* **Unilateral exit / force-close** ("Unilateral exit"): the user recovers its
  funds on-chain with no server cooperation.

Open, refresh, and offboard reuse the Ark RPCs and ceremonies of ARK #3,
ARK #4, and ARK #7: there are no channel-specific Ark RPCs, the forfeit object
is the standard one (ARK #4), and channel establishment itself runs over the
Lightning transport, not the Ark service. The only new Ark protocol object is
the `channel-funding` policy (ARK #2). Three existing requests carry optional
channel fields, described where they are used: the board cosign request conveys
the channel's funding pubkeys and a temporary channel identifier; the leaf
cosign request a channel identifier; and the round participation response an
optional server liquidity bound (see "Refresh" and "Compatibility").

### Actors and keys

The channel reuses the Ark keys of ARK #0; it introduces no new keys at the
Ark layer.

* `A` — the user's pubkey, used in the funding VTXO's cooperative key and to
  cosign every Ark transition, exactly as for a `pubkey` VTXO.
* `S` — the server pubkey.

The funding output's cooperative 2-of-2 is `musig(A, S)`. A channel's two
BOLT-3 funding pubkeys MUST be pinned to the user key `A` and the server key
`S` — the channel uses the parties' ark keys as its funding keys — so the
channel's BOLT-3 funding output and the VTXO's `channel-funding` output are one
and the same taproot output (see "The channel-funding output"). This pinning is
what makes every cooperative spend of a channel VTXO — the commitment, forfeit,
refresh, and offboard — an ordinary `musig(A, S)` operation; without it the
cooperative paths could not sign against the funding output.

### Trust assumptions

Beyond the VTXO trust model (ARK #0), a channel adds:

* **Virtual funding.** A channel's funding output is the `channel-funding`
  VTXO's own output; it lives in the VTXO's off-chain exit chain and is never
  broadcast unless the channel force-closes. The Lightning stack nonetheless
  needs the funding confirmed to operate the channel. *Virtual funding* means
  treating the funding output as confirmed at the point the VTXO's on-chain
  chain anchor (ARK #2) confirms — the board transaction for a freshly opened
  channel, or the round transaction (the tree root) for a channel established
  by a refresh. Once that anchor has confirmed, the user holds a fully exitable
  VTXO, so the funding output can always be actualized on-chain; treating it as
  confirmed is therefore sound. (How the implementation feeds this confirmation
  to the channel state machine — including any backdating it applies for
  HTLC-timeout safety — is out of scope.)
* **Expiry is a deadline.** Like every VTXO, a channel VTXO has an
  `expiry_height` after which the server may sweep its backing funds. The user
  MUST refresh or close the channel before expiry; a channel left
  un-refreshed past expiry can lose its funds to the server's sweep.
* **Liveness for cooperation.** Refresh and cooperative close require a
  responsive server; unilateral exit is always available without one.

## The channel-funding output

A channel VTXO's output carries the `channel-funding` policy (ARK #2, type
byte `0x08`):

* internal key `musig(A, S)` — the cooperative 2-of-2;
* a single leaf, timelock-sign `(expiry_height, S)` — the server's
  post-expiry sweep.

This is the **cosign taproot** `(musig(A, S), S, expiry_height)` of ARK #2.
The VTXO's `point` output *is* the channel's BOLT-3 funding outpoint: both
parties MUST derive the identical taproot output, and the Lightning commitment
transaction spends it through the key path, signed by `musig(A, S)`.

Two properties distinguish it from a `pubkey` output:

* **No user exit leaf.** A `pubkey` output carries a `delayed-sign(exit_delta,
  A)` leaf for the user's unilateral claim. A `channel-funding` output does
  not: after a unilateral exit the funds are claimed through the commitment
  transaction, which is encumbered by the *same* `exit_delta` relative delay a
  `pubkey` exit waits (ARK #6) — carried on the commitment input rather than a
  VTXO leaf (see "The Ark channel type" and "Unilateral exit"). A VTXO-level
  delay would therefore be redundant and is omitted.
* **Server expiry sweep.** The `timelock-sign(expiry_height, S)` leaf lets the
  server reclaim the funding output after expiry without broadcasting the
  commitment transaction — the same recourse it has over every other
  cosign-taproot output in the protocol (ARK #4, ARK #6 sweeping).

Everything else about a channel VTXO — its genesis chain, encoding, and
validation — is exactly as in ARK #2. It is a VTXO like any other.

## The Ark channel type

A channel on Ark is not a vanilla BOLT channel: the parties MUST negotiate a
dedicated **Ark channel type**. There is no BOLT feature bit for it — Ark-aware
peers agree on it, and on its parameters, out of band — and the server MUST
refuse to open a channel of any other type (channel support is advertised by
the `supports_channels` flag, see "Compatibility").

The Ark channel type is standard BOLT-3 anchor-channel commitment machinery
with the deviations below. Everything not listed — the `to_local` / `to_remote`
outputs, revocation and per-commitment key derivation, the base HTLC scripts,
and HTLC resolution — is unchanged BOLT-3 and is referenced, not restated. An
Ark channel is interoperable only if both peers reproduce these deviations
identically: a mismatch makes the exchanged commitment signatures fail to
verify and aborts the channel. The parties pin the channel's two BOLT-3 funding
pubkeys to `A` and `S` (Actors and keys); the remaining deviations follow.

* **Funding output.** The funding output is the `channel-funding` taproot (The
  channel-funding output) — internal key `musig(A, S)`, with the server's
  `timelock-sign(expiry_height, S)` sweep leaf — spent by the commitment
  transaction through the key path. BOLT-3 uses a P2WSH 2-of-2 of the two
  funding pubkeys instead. (As noted in "The channel-funding output," the
  reference currently builds this output keyspend-only on *both* the Ark and the
  Lightning sides; adding the expiry leaf is the pending `asp-leaf-sweep-gap`
  fix and changes the funding `scriptPubkey` on both sides together.)

* **Transaction version.** The commitment transaction is version 3 (TRUC), like
  every off-chain Ark transaction (ARK #0), not BOLT-3's version 2.

* **Unilateral-exit delay on the commitment input.** The commitment
  transaction's input — the one spending the funding output — carries a BIP-68
  height-based relative timelock of `exit_delta` blocks (the same encoding as
  ARK #2's CSV clauses). Once the funding output is actualized on-chain by a
  VTXO exit, the commitment becomes spendable only `exit_delta` blocks later —
  the *same* delay a `pubkey` VTXO's unilateral exit waits (ARK #6). This is the
  channel's unilateral-exit delay; it lives on the commitment input rather than
  a VTXO leaf (which is why `channel-funding` carries no exit clause), and it
  MUST equal the VTXO's `exit_delta`. BOLT-3 instead puts the high 24 bits of
  the obscured commitment number in this `nSequence`.

* **Commitment number in an `OP_RETURN`.** Because the input `nSequence` now
  carries the exit-delay CSV, the obscured commitment number cannot ride there.
  The commitment transaction instead carries a dedicated zero-value output
  `OP_RETURN <obscured_commitment_number>` — the 8-byte big-endian BOLT-3
  obscured commitment number — and its `nLockTime` is `0`. BOLT-3 carries no
  such output and splits the obscured number across `nLockTime` (low 24 bits)
  and the input `nSequence` (high 24 bits).

* **HTLC success-path CSV.** Each HTLC output's preimage-claim (success) branch
  gains an extra `<exit_delta> OP_CSV OP_DROP`, and the HTLC-claim transaction's
  `nSequence` is at least `exit_delta`; the channel's minimum `cltv_expiry_delta`
  is additionally raised to cover `exit_delta` plus a safety margin. Together
  these guarantee that, after a force-close, the server's HTLC-timeout path
  cannot lose the race to a counterparty's preimage-claim — so the server never
  has to unwind the VTXO tree to resolve an HTLC — and a receiver MUST reject an
  incoming HTLC whose CLTV budget is too small. (This CSV is a separate
  parameter the reference pins to `exit_delta`; both peers MUST use the same
  value.) BOLT-3 has no success-branch CSV.

* **Zero-fee commitment with a P2A anchor.** The commitment pays no fee of its
  own; it carries the pay-to-anchor output `OP_1 <0x4e73>` (the P2A anchor of
  ARK #0) and is fee-bumped by CPFP at broadcast, exactly as the exit
  transactions are (ARK #6). This subsumes BOLT's zero-fee-commitment /
  zero-fee-HTLC anchor variants.

* **Virtual funding.** The channel uses manual funding at zero confirmation
  depth — it becomes usable without its funding transaction appearing on-chain
  (see "Trust assumptions").

## Channel open

Opening a channel is a board (ARK #3) whose resulting VTXO carries the
`channel-funding` policy, followed by Lightning channel setup over the funding
output.

```
   funding tx (user's on-chain wallet)
        │
        ▼
   funding output: cosign taproot (musig(A,S), S, expiry)      [chain anchor]
        │  (cosigned exit transaction, held off-chain)
        ▼
   channel-funding output: cosign taproot (musig(A,S), S, expiry)
        │     ↑ this output IS the channel's funding outpoint
        │  (Lightning commitment transaction, held off-chain — virtual funding)
        ▼
   commitment outputs: to_local / to_remote / HTLCs            (BOLT-3)
```

### Board

The user boards exactly as in ARK #3 — funding an on-chain **cosign taproot**
`(musig(A, S), S, expiry_height)` (the chain anchor) and obtaining the
server's cosignature on a single exit transaction — with one difference: the
exit transaction's output 0 carries the `channel-funding` policy (ARK #2)
instead of `pubkey`. The resulting depth-1 VTXO's output is the channel's
funding outpoint. The board fee is the server's `board` fee, computed and
applied as in ARK #3.

This uses the ARK #3 board cosign and registration ceremony, except the board
cosign request additionally carries the channel's funding pubkeys (pinned to
`A` and `S`) and a temporary channel identifier, so the server builds the leaf
sighash against the `channel-funding` policy rather than `pubkey`.

### Channel setup

The parties MUST negotiate the Ark channel type (see "The Ark channel type").
With the funding VTXO's output as the funding outpoint, they run Lightning
channel establishment (referenced) to produce and exchange signatures on the
initial commitment transaction, which spends the funding output through its
key path (a `musig(A, S)` cosignature). It is held off-chain and never
broadcast unless the channel is force-closed; the Lightning stack treats the
funding output as confirmed (virtual funding, informative).

### Safety gate

The user MUST NOT broadcast the on-chain funding transaction until it holds
both (a) the fully cosigned genesis exit chain of the channel VTXO (ARK #6)
and (b) a valid initial commitment transaction. Broadcasting the funding
transaction is the point of no return: before it, an aborted open costs
nothing — the cosigned exit transaction spends an output that never exists, as
in ARK #3 — while after it, the user must be able to recover the funds either
cooperatively or by force-close.

### Sequence

The Ark and LDK/BOLT steps interleave as follows (the user drives; the server
is both ASP and Lightning peer):

```
 1. [BOLT]  negotiate the Ark channel → learn the funding pubkeys and a
            temporary channel id (the funding pubkeys are forced to A and S)
 2. [local] build the board tx whose leaf output is the channel-funding policy
 3. [Ark]   RequestBoardCosign (carries the funding pubkeys + temp channel id)
            → ASP cosigns the leaf/exit tx; finalize the musig(A,S) signature
 4. [BOLT]  provide the leaf output as the funding outpoint; exchange the
            initial commitment (FundingCreated / FundingSigned)
 5. [gate]  SAFETY GATE: the genesis exit chain AND a valid commitment exist
            ──────────────── LAST SAFE ABORT (nothing on-chain) ──────────────
 6. [local] sign + broadcast the board tx          *** POINT OF NO RETURN ***
 7. [local] board tx confirms → virtually confirm the funding (board-tx height)
 8. [Ark]   RegisterBoardVtxo
 9. [BOLT]  ChannelReady — the channel is usable
```

## Operation

Once open, payments flow over the channel using the Lightning protocol (BOLT),
unmodified, and the funding VTXO sits unchanged until the channel is refreshed,
closed, or force-closed.

The Ark/Lightning boundary is drawn as follows.

**Normative (Ark side):**

* the channel is of the Ark channel type (see "The Ark channel type");
* the funding output's exact taproot and value (see "The channel-funding
  output");
* that the commitment transaction spends the funding output through the key
  path, signed by `musig(A, S)`.

**Referenced (BOLT and the channel implementation), not specified here:** the
commitment transaction's structure and outputs (`to_local`, `to_remote`,
anchors, HTLCs), HTLC resolution, channel quiescence, the channel state
machine, and the mechanism by which the funding output is presented to that
state machine as confirmed (virtual funding).

## Refresh

A channel VTXO expires like any other, and its exit chain grows no shallower
over time. **Refresh** replaces it with a fresh channel VTXO before expiry,
resetting both, while preserving the channel and its balance. It is a round
(ARK #4): the old channel VTXO is forfeited and a new `channel-funding` VTXO
is issued as a round leaf.

### Reused from ARK #4

* **Forfeit.** The old channel VTXO is forfeited by the standard hArk forfeit
  (ARK #4): a `musig(A, S)` cosignature over a forfeit transaction spending
  the VTXO's output, released to the server only against the round's unlock
  preimage. Because the channel-funding cooperative key is `musig(A, S)`, this
  is the *same* forfeit as for any VTXO — which is what lets the user give up
  the old channel VTXO atomically against issuance of the new one.
* **Issuance.** The new channel VTXO is a round leaf whose requested policy is
  `channel-funding` (admitted for the channel-refresh flow, ARK #4). Its
  ownership attestation, tree cosigning, leaf cosignature, and unlock-preimage
  completion are exactly as in ARK #4.
* **Fee.** The server's `refresh` fee (ARK #4) applies.

Refresh uses the ARK #4 round messages — round participation, the completion
steps (including leaf cosign), and `forfeit_vtxos`. The only channel-specific
additions to them are a channel identifier on the leaf cosign request (so the
server maps the new leaf to the channel being refreshed) and an optional server
liquidity bound on the round participation response:

### Server liquidity adjustment

When the client requests a channel-funding output of value `V`, the server MAY
decline that amount and return, on the round participation response, a cap
`V − X`, where `X` is liquidity the server is withdrawing from its own side of
the channel. The client MUST then either request a channel-funding output no
larger than the cap (subject to its own balance) or abort the refresh. The
client's balance is unaffected — `X` comes from the server's side — so the
adjustment can only shrink the server-provided (inbound) capacity, never the
client's funds. Absent the cap, the participation is accepted as usual.

### Added by the channel layer (referenced)

Refreshing the backing VTXO moves the channel onto a new funding outpoint. The
Lightning layer (referenced) quiesces the channel, settles in-flight HTLCs,
and *teleports* the channel onto the new funding output — re-pointing the
existing channel at the new funding outpoint rather than closing and reopening
it — with the client's balance preserved (the server may adjust its own side;
see "Server liquidity adjustment"). The ordering is safety-critical and follows the
exit-before-forfeit discipline of ARK #4: the user MUST verify the new channel
VTXO is issued and its exit story is complete before forfeiting the old one.
The forfeit is the point of no return.

### Sequence

```
 1. [Ark]   SubmitRoundParticipation: forfeit the old channel VTXO, request a
            fresh channel-funding output  (→ unlock_hash)
            · if the server caps issuance at V−X, resubmit ≤ cap or abort
              (see "Server liquidity adjustment")
 2. [Ark]   poll RoundParticipationStatus until ISSUED (round tx broadcast)
 3. [gate]  MONEY GATE: the round tx is confirmed to the required depth → the
            new funding is now virtually confirmed
            ─────────── LAST SAFE ABORT (old channel still intact) ───────────
 4. [Ark]   RequestLeafVtxoCosign (carries the channel id) → ASP cosigns the
            new leaf; finalize + verify the musig(A,S) signature
 5. [BOLT]  teleport: Stfu (quiesce) → settle in-flight HTLCs → re-point the
            channel at the new funding outpoint (exchange commitments); wait
            until the peer has accepted it
 6. [Ark]   RequestForfeitNonces + ForfeitVtxos for the OLD VTXO
                                                  *** POINT OF NO RETURN ***
            → unlock_preimage (the old VTXO is now forfeited to the server)
 7. [BOLT]  complete the teleport with the preimage-unlocked state → ChannelReady
 8. [local] persist the new exit path; remove the old
```

## Offboard

A cooperative channel close is an offboard (ARK #7): the server pays the user's
channel balance to an on-chain destination, and the user forfeits the channel
VTXO to the server through the connector swap.

The forfeit is the standard ARK #7 connector-bound forfeit of the channel
VTXO — a `musig(A, S)` cosignature over a forfeit transaction that also spends
a connector output, valid only in a chain where the offboard transaction (and
thus the user's payout) exists. The offboard request, fee (`offboard`,
ARK #7), connector fanout, and atomicity are all as in ARK #7, using the
`prepare_offboard` / `finish_offboard` messages unchanged.

The channel layer adds (referenced) the close negotiation: the channel is
quiesced and its settled balance read before the offboard amount is fixed.

### Sequence

```
 1. [BOLT]  Stfu (quiesce) → settle in-flight HTLCs → read the settled balance
 2. [gate]  balance > 0 and (balance − fee) ≥ dust
 3. [Ark]   PrepareOffboard: destination + amount, channel VTXO id, ownership
            proof  → unsigned offboard tx + forfeit-cosign nonce
 4. [local] validate the offboard tx; sign the connector-bound forfeit of the
            channel VTXO (musig(A,S))
 5. [Ark]   FinishOffboard: send the forfeit signature
                                                  *** POINT OF NO RETURN ***
            → fully-signed offboard tx
 6. [local] broadcast the offboard tx → client paid on-chain; close the channel
```

## Unilateral exit / force-close

When the server is unavailable or uncooperative, the user recovers its funds
with no help, by exiting the channel VTXO on-chain and force-closing the
channel. The two halves compose: the VTXO exit (ARK #6) brings the funding
output on-chain; the force-close (BOLT) then spends it.

1. **Exit the VTXO (ARK #6).** Broadcast the channel VTXO's genesis chain,
   root first, until the `channel-funding` output is confirmed on-chain. This
   is an ordinary emergency exit: TRUC (v3) transactions, P2A/CPFP
   fee-bumping, each level after its parent confirms.
2. **Force-close.** Broadcast the latest commitment transaction, which spends
   the now-on-chain funding output through the key path (`musig(A, S)`). By the
   Ark channel type, that input carries a relative timelock of `exit_delta`
   (see "The Ark channel type"), so the commitment becomes valid only
   `exit_delta` blocks after the funding output confirms. This delay — on the
   commitment input, not a VTXO leaf — is the channel's unilateral-exit delay,
   and is why the `channel-funding` output carries no delayed-exit clause.
3. **Claim.** Once the commitment confirms, claim its outputs (`to_local`,
   `to_remote`, and resolved HTLCs) per the Ark channel type's BOLT-3 rules
   (referenced).

Unlike a `pubkey` exit (ARK #6), there is no VTXO-level `delayed-sign` claim;
the channel VTXO's value is claimed only through the commitment transaction.

### Server recourse after expiry

The `channel-funding` output's `timelock-sign(expiry_height, S)` leaf lets the
server, after `expiry_height`, sweep the funding output directly — without
broadcasting the commitment transaction — just as it may sweep any unspent
cosign-taproot output past expiry (ARK #4, ARK #6). This is the server's
recourse when a user has actualized the funding output on-chain but left the
channel otherwise unresolved. It is bounded by `expiry_height`, which is why
the user MUST refresh or close before expiry.

## Messages

Channels reuse the Ark messages of ARK #3 (board), ARK #4 (round), and ARK #7
(offboard); see those documents for the base definitions. Channel operations
add the fields below. Every added field is absent for a non-channel operation,
and a peer that does not implement channels ignores it (see "Compatibility");
no other Ark message changes for channels, and channels add no new RPC.

### `board_cosign_request` (ARK #3)

Added fields:

| Field | Type | Meaning |
|---|---|---|
| `temporary_channel_id` | 32 bytes | BOLT temporary channel id of the pending LDK channel this board funds |
| `holder_funding_pubkey` | pubkey | the channel's holder (client) BOLT-3 funding pubkey |
| `counterparty_funding_pubkey` | pubkey | the channel's counterparty (server) BOLT-3 funding pubkey |

Requirements (server):

* The two funding pubkeys are present together or not at all; exactly one
  present is invalid. When both are present, the server MUST build the exit
  transaction's output with the `channel-funding` policy — computing the cosign
  sighash against it — instead of `pubkey`; when both are absent it is a plain
  board (ARK #3).
* The server MUST verify the funding pubkeys are pinned to the channel parties'
  ark keys — `holder_funding_pubkey` equal to the request's `user_pubkey` (`A`)
  and `counterparty_funding_pubkey` equal to its own key `S` — and reject
  otherwise, since the funding 2-of-2 must be `musig(A, S)` for the VTXO to be
  cooperatively spendable and forfeitable (The Ark channel type).

### `leaf_vtxo_cosign` (ARK #4)

Added field:

| Field | Type | Meaning |
|---|---|---|
| `channel_id` | 32 bytes | LDK channel id of the channel being refreshed, letting the server map it to its internal channel id; absent for a non-channel leaf |

### `submit_round_participation` response (ARK #4)

For a channel-refresh participation the server MAY return, in place of the
participation's `unlock_hash`:

| Field | Type | Meaning |
|---|---|---|
| `counter_max_issuable_sat` | u64 sats | the maximum `channel-funding` output it will issue (`V − X`) when it declines the requested amount (see "Server liquidity adjustment") |

Requirements (participant): on receiving `counter_max_issuable_sat` (and no
`unlock_hash`), the participant MUST resubmit requesting a `channel-funding`
output no larger than the cap, or abandon the refresh; it MUST NOT proceed at
the original amount.

## Compatibility

The channel work extends the protocol without disturbing existing flows.

* **Additive messages.** The channel fields on the board cosign request, the
  leaf cosign request, and the round participation response are optional
  additions. A peer that does not implement channels ignores them, and the
  board, refresh, and offboard flows are unchanged for non-channel VTXOs.
* **Channel support is a capability, not a default.** A channel open or refresh
  requires a channel-aware server. An older server ignores the funding pubkeys
  and would cosign an ordinary `pubkey` leaf; the client detects this — the
  cosignature does not validate against the expected `channel-funding` output —
  and the open fails safely, since the funding transaction is never broadcast
  without a valid exit. It does not silently produce a non-channel VTXO. A
  server advertises channel support with the `supports_channels` flag in ark
  info (ARK #0); a client MUST NOT attempt a channel open against a server that
  does not set it.
* **New policy type.** A `channel-funding` VTXO (policy type `0x08`) is rejected
  by decoders that predate it (ARK #2 decoders reject unknown policy type
  bytes), so channel VTXOs are meaningful only between channel-aware parties.

## Security and trust notes

* **Exit guarantee preserved.** A channel VTXO carries a complete, cosigned
  genesis exit chain like any VTXO (ARK #2, ARK #6); together with the
  commitment transaction it always lets the user recover its channel balance
  on-chain without the server. The server cannot steal from a user who exits
  and force-closes in time.
* **Atomic give-up.** Because a channel VTXO is forfeited by the standard
  `musig(A, S)` hArk swap (ARK #4) or connector swap (ARK #7), refreshing or
  closing a channel hands the old VTXO to the server only in a history where
  the user has received its replacement or its payout. There is no window in
  which the user has surrendered the old channel VTXO without compensation.
* **Expiry is a deadline.** The server may sweep the funding output after
  `expiry_height`. A channel left un-refreshed and un-closed past expiry can
  lose its funds. Users MUST refresh well before expiry, exactly as for any
  VTXO (ARK #0).
* **Virtual funding.** The channel operates as though its funding output were
  confirmed while that output lives only in the off-chain exit chain. The
  safety of this rests on the exit guarantee above: the funding output can
  always be actualized on-chain. The mechanism by which the Lightning stack is
  told the funding is confirmed is out of scope (informative).
