# ARK #8: Channels

This document specifies Lightning channels built on the Ark protocol: how a
channel is funded by a VTXO, operated, refreshed, closed cooperatively, and
exited unilaterally. It builds on VTXOs and policies (ARK #2), boarding
(ARK #3), rounds (ARK #4), emergency exit (ARK #6), and offboarding (ARK #7).

The channel's payment machinery — the commitment transaction, its outputs and
HTLCs, channel quiescence, and the channel state machine — is that of the
Lightning Network (BOLT-3 and related), with the deviations collected in "The
Ark channel type" and a custom refresh transport, "The teleport protocol."
This document specifies the *Ark side* — the VTXO and the presigned bridge
transaction that fund the channel, and how the channel lifecycle maps onto Ark's
existing flows — together with those commitment deviations and the teleport
messages, since both are specific to Ark and not drawn from any BOLT. The Lightning send/receive payment flows remain
out of scope (ARK #0).

## Overview

A **channel VTXO** is an ordinary `musig(A, S)` 2-of-2 VTXO (ARK #2) whose
output is spent by a presigned **bridge transaction** whose own output is the
funding output of a Lightning commitment transaction (see "The channel VTXO and
the bridge transaction"). The two channel parties are the user `A` and the
server `S`; there is no third party.

Because a channel VTXO's cooperative key is the same `musig(A, S)` as a `pubkey`
VTXO, every *cooperative* Ark operation on it — issuing it (open), giving it up
(forfeit), refreshing it (round), and closing it (offboard) — is the
corresponding standard Ark flow, unchanged. A channel VTXO differs from a
`pubkey` VTXO in only two ways:

* its output carries the `channel-funding` policy (ARK #2) rather than
  `pubkey`: the taproot script tree holds the server's expiry-sweep leaf
  instead of a user delayed-exit leaf; and
* its unilateral exit proceeds *through* the bridge transaction and then the
  Lightning commitment transaction (a force-close), not through a VTXO leaf,
  because the funds are committed to a channel.

The lifecycle:

* **Open** ("Channel open"): the user boards (ARK #3) into a depth-1 channel
  VTXO and presigns the bridge, then sets up the Lightning channel over the
  bridge's funding output.
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
Lightning transport, not the Ark service. The new Ark protocol objects are the
`channel-funding` policy (ARK #2) and the presigned **bridge transaction**,
which is cosigned *within* the existing board/leaf cosign rather than via a new
RPC. Existing requests carry optional channel fields, described where they are
used: the board cosign request a `channel_id` and a bridge nonce; the leaf cosign
request a `channel_id` and a bridge nonce; and the round participation response an
optional server liquidity bound (see "Refresh" and "Compatibility").

### Actors and keys

At the Ark layer the channel reuses the keys of ARK #0 and introduces none.

* `A` — the user's pubkey, used in the channel VTXO's cooperative key and to
  cosign every Ark transition — including the bridge — exactly as for a `pubkey`
  VTXO.
* `S` — the server pubkey.

The channel VTXO's cooperative 2-of-2 is `musig(A, S)`; it signs the bridge
transaction and every forfeit, refresh, and offboard, exactly as for any VTXO.
The `channel-funding` policy stores only `user_pubkey` (= `A`); the server side
is `S`, the VTXO's `server_pubkey`, exactly as `pubkey` derives it (ARK #2).

Separately, the channel has two **BOLT-3 funding pubkeys** — ordinary Lightning
funding keys exchanged in `open_channel` / `accept_channel`. They are *not* `A`
and `S`: the bridge transaction's output is a stock P2WSH 2-of-2 of these two
keys (what the commitment spends), while the VTXO the bridge spends remains
`musig(A, S)`. No key pinning is therefore required. The funding keys appear in
no policy, ark-info field, or cosign request: the server already holds both (from
`open_channel` / `accept_channel`) keyed by `channel_id`, so a cosign need only
name the channel (see "Messages").

### Trust assumptions

Beyond the VTXO trust model (ARK #0), a channel adds:

* **Virtual funding.** A channel's funding output is the output of the presigned
  **bridge transaction**, which spends the `channel-funding` VTXO; both live in
  the VTXO's off-chain exit chain and are never broadcast unless the channel
  force-closes. The Lightning stack nonetheless needs the funding confirmed to
  operate the channel. *Virtual funding* means treating the funding output as
  confirmed at the point the VTXO's on-chain chain anchor (ARK #2) confirms — the
  board transaction for a freshly opened channel, or the round transaction (the
  tree root) for a channel established by a refresh. Once that anchor has
  confirmed, the user holds a fully exitable VTXO whose bridge actualizes the
  funding output on-chain; treating it as confirmed is therefore sound. (How the
  implementation feeds this confirmation to the channel state machine is out of
  scope; it MAY present the funding at the anchor's true height. HTLC-timeout
  safety does not rely on backdating the funding confirmation — it follows from
  the bridge-input exit-delay CSV and the HTLC success-path CSV of "The Ark
  channel type" together with the absolute CLTV of every HTLC.)
* **Expiry is a deadline.** Like every VTXO, a channel VTXO has an
  `expiry_height` after which the server may sweep its backing funds. The user
  MUST refresh or close the channel before expiry — with enough margin to confirm
  the bridge first, and stopping new HTLCs as the deadline nears, since an
  unsettleable in-flight HTLC blocks a cooperative refresh (see "The refresh /
  force-close deadline"). A channel left un-refreshed past expiry can lose its
  funds to the server's sweep.
* **Liveness for cooperation.** Refresh and cooperative close require a
  responsive server; unilateral exit is always available without one.

## The channel VTXO and the bridge transaction

A channel is backed by two presigned, off-chain objects: the **channel VTXO**
and the **bridge transaction** that spends it. The bridge's output is the
Lightning funding output.

```
   channel VTXO output: cosign taproot (musig(A,S), S, expiry)
        │  (bridge transaction, presigned, held off-chain)
        ▼
   bridge tx ── out0: funding output = P2WSH 2-of-2 (BOLT-3 funding keys)
            └─ out1: P2A anchor (0 value)
        │  (Lightning commitment transaction, held off-chain)
        ▼
   commitment outputs: to_local / to_remote / HTLCs            (BOLT-3)
```

### The channel VTXO output

A channel VTXO's output carries the `channel-funding` policy (ARK #2, type byte
`0x08`):

* internal key `musig(A, S)` — the cooperative 2-of-2 (`A` = the policy's
  `user_pubkey`, `S` = the VTXO's `server_pubkey`; ARK #2). This is the path the
  bridge transaction spends, and the path used to forfeit, refresh, or offboard
  the VTXO;
* a single leaf, timelock-sign `(expiry_height, S)` — the server's post-expiry
  sweep.

This is the **cosign taproot** `(musig(A, S), S, expiry_height)` of ARK #2 — a
VTXO like any other. Two properties matter for channels:

* **No user exit leaf.** A `pubkey` output carries a `delayed-sign(exit_delta,
  A)` leaf for the user's unilateral claim; a `channel-funding` output does not.
  The funds are committed to a channel, so they are actualized only through the
  bridge and then the commitment — never by a VTXO-level claim. The unilateral
  exit delay a `pubkey` exit gets from its leaf is instead carried on the
  **bridge transaction's input** (below).
* **Server expiry sweep.** The `timelock-sign(expiry_height, S)` leaf lets the
  server reclaim the VTXO output after `expiry_height` — without the bridge or
  commitment — the same recourse it has over every cosign-taproot output (ARK #4,
  ARK #6 sweeping). This is its recourse when the VTXO's tree is left to time out.

Everything else about a channel VTXO — its genesis chain, encoding, and
validation — is exactly as in ARK #2.

### The bridge transaction

The bridge is a single presigned transaction, structurally an exit-chain
transaction (ARK #6): `nVersion = 3` (TRUC), `nLockTime = 0`, 0-fee.

* **Input** (single): the channel VTXO output, spent through the key path
  `musig(A, S)`, with a BIP-68 height-based relative timelock `nSequence =
  exit_delta` (the server's `vtxo_exit_delta`, ARK #0).
* **Output 0 (funding):** a stock segwit-v0 **P2WSH 2-of-2** of the channel's two
  BOLT-3 funding pubkeys (Actors and keys), value = the channel VTXO's value
  (0-fee). This is an ordinary BOLT-3 funding output — it carries *no* Ark sweep
  script: the server's expiry recourse is one hop up, on the VTXO output, and LDK
  sweeps this output with its own transactions, which have no custom-input support
  for an Ark clause. The commitment transaction spends it.
* **Output 1 (anchor):** a P2A `OP_1 <0x4e73>` of 0 value, CPFP-bumped at
  broadcast, exactly as the Ark exit transactions (ARK #6).

The bridge carries the channel's **unilateral-exit delay**. Because it is
presigned with `nSequence = exit_delta` and is the only unilateral on-chain spend
of the VTXO's key path (forfeit / refresh / offboard are cooperative and
off-chain), the delay is enforced by the bridge alone — no script on the VTXO
output, and no CSV on the commitment input. After a unilateral exit the bridge
confirms only `exit_delta` blocks after the channel VTXO output confirms — the
*same* delay a `pubkey` exit waits (ARK #6) — and the commitment then confirms
immediately. The delay MUST equal the channel's pinned `exit_delta` — fixed at
open from `vtxo_exit_delta` (ARK #0) and reused unchanged for the life of the
channel, including by every refresh bridge (see "The Ark channel type") — so it
agrees by construction.

Both parties MUST construct the identical bridge — same funding output, P2A
anchor, and `nSequence` — or the funding outpoints differ and the channel cannot
operate; the bridge cosign reconstructs it canonically (see "Messages"). The
bridge is the **last transaction of the channel VTXO's exit chain**: exiting a
channel VTXO means broadcasting its genesis chain root-first (ARK #6) until the
VTXO output confirms, then the bridge, then the commitment (see "Unilateral
exit"). Both parties hold the fully signed bridge, so either can force-close by
broadcasting bridge + commitment.

## The Ark channel type

A channel on Ark is not a vanilla BOLT channel: the parties MUST negotiate a
dedicated **Ark channel type**. It is not a registered BOLT feature — Ark-aware
peers agree on it out of band — but at channel open it is carried in the
`channel_type` as an experimental feature pair, bits `400/401` (the reference's
`ark_channel` feature; the required even bit `400` is the one set in a
`channel_type`, alongside the `static_remote_key` and `zero_fee_commitments`
bits it implies). The pair is experimental — chosen clear of allocated BOLT bits
and not advertised in `init` / `node_announcement` — so the exact number is
subject to change if the type is ever standardized. The server MUST refuse to
open a channel of any other type (channel support is advertised by the
`supports_channels` flag, see "Compatibility").

The Ark channel type is a BOLT channel using the `zero_fee_commitments` option
(BOLT #9 feature 40/41, specified in BOLT #3), with the Ark-specific deviations
below. From `zero_fee_commitments` it takes — as plain BOLT, not Ark changes —
version-3 (TRUC) commitment *and* HTLC transactions and a single `shared_anchor`
pay-to-anchor output `OP_1 <0x4e73>` in place of any commitment fee, fee-bumped
by CPFP at broadcast (anchor value per BOLT: `min(240 sat, the commitment's
residual)`, any excess becoming on-chain fee). That is the same P2A and CPFP
discipline the Ark exit transactions use (ARK #6). Everything else not listed
below — the `to_local` / `to_remote` outputs, revocation and per-commitment key
derivation, the base HTLC scripts, and HTLC resolution — is likewise unchanged
BOLT and is referenced, not restated.

An Ark channel is interoperable only if both peers reproduce the deviations
below identically: a mismatch makes the exchanged signatures fail to verify and
aborts the channel. The funding output is a stock BOLT-3 P2WSH 2-of-2 produced by
the bridge transaction (see "The channel VTXO and the bridge transaction"), so it
is **not** a deviation — the channel's two BOLT-3 funding pubkeys are ordinary
Lightning keys (Actors and keys), and the commitment is an unmodified
`zero_fee_commitments` commitment (its obscured commitment number rides in
`nLockTime`/`nSequence` as in BOLT-3 — the bridge, not the commitment, carries
the exit-delay CSV). The Ark-specific deviations are:

* **HTLC success-path CSV.** Each HTLC output's preimage-claim (success) branch
  gains an extra `<exit_delta> OP_CSV OP_DROP`, and the HTLC-claim transaction's
  `nSequence` is at least `exit_delta`. This is what guarantees that, after a
  force-close, the server's HTLC-timeout path cannot lose the race to a
  counterparty's preimage-claim, so the server never has to unwind the VTXO tree
  to resolve an HTLC. This success CSV MUST equal the channel's pinned
  `exit_delta` (see "`exit_delta` is pinned at open" below); both peers use that
  fixed value rather than re-deriving it per update or negotiating it. BOLT-3 has
  no success-branch CSV.

* **HTLC CLTV budget.** A receiver MUST reject an incoming HTLC whose CLTV
  budget is too small, and the channel's minimum `cltv_expiry_delta` MUST be at
  least `2 * vtxo_exit_delta + max_vtxo_exit_depth + cltv_safety_margin`. Resolving
  an HTLC on-chain crosses **two** `vtxo_exit_delta` delays in series: first to get
  the **commitment** confirmed — force-closing *through* the VTXO exit takes up to
  `max_vtxo_exit_depth` genesis levels (ARK #0) plus the bridge, whose input is
  time-locked `vtxo_exit_delta` — and then, on the confirmed commitment, the
  **HTLC success-path CSV** (above) delays the preimage claim a further
  `vtxo_exit_delta`. Both must complete before the HTLC's absolute CLTV (after
  which the offerer's timeout path opens), which is why `vtxo_exit_delta` appears
  twice. `cltv_safety_margin` is the ordinary Lightning confirmation buffer, but
  for an Ark channel it MUST be sized larger: it absorbs confirmation variance
  across the **whole serial chain** a force-close climbs — up to
  `max_vtxo_exit_depth` genesis levels, then the bridge, the commitment, and the
  success-path claim — each budgeted at a single block in the floor but any of
  which can run longer under fee pressure (the non-Ark race is only
  commitment-then-claim; refreshing often to keep exit chains shallow keeps the
  needed margin realistic). The exit-derived terms come from
  the channel's pinned `exit_delta` (below) and from `max_vtxo_exit_depth` (ark
  info), so both peers compute the same floor given the same buffer; each
  enforces it from its own configuration, not on the wire. `max_vtxo_exit_depth`
  is also the upper bound the budget assumes on the **channel VTXO's own** exit
  depth — the number of genesis levels a force-close must climb: a channel VTXO is
  only ever a board leaf (depth 1) or a refresh round leaf, and the client MUST
  refuse a refresh whose issued leaf would exceed it (see "Refresh"), since a
  deeper exit chain would overrun the budget.

* **Virtual funding.** The channel uses manual funding at zero confirmation
  depth — it becomes usable without its funding (bridge) transaction appearing
  on-chain (see "Trust assumptions").

**`exit_delta` is pinned at open.** A channel's `exit_delta` — the value on the
bridge input's `nSequence`, in the HTLC success-path CSV, and in the CLTV budget
above — is read from `vtxo_exit_delta` (ARK #0) **once, at open**, and is
thereafter a fixed parameter of the channel. Both parties MUST store it (with the
channel's other parameters, keyed by `channel_id`) and use the stored value for
every later commitment and for the bridge of every refresh; neither re-reads
`vtxo_exit_delta` from ark info for an existing channel. No wire field carries
it — the value is pinned by construction, since the board cosign fixes it on the
bridge input and the initial commitment exchange fixes it in the HTLC scripts, and
a disagreement makes those signatures fail to verify and aborts the open. A
refresh MUST build the new bridge at the pinned value, and the server,
reconstructing that bridge from `channel_id`, MUST use the channel's stored
`exit_delta` rather than its current ark-info value; if the server's
`vtxo_exit_delta` has since changed and it will not cosign at the pinned value,
the refresh fails and the channel must be closed and reopened.

## Channel open

Opening a channel is a board (ARK #3) whose resulting VTXO carries the
`channel-funding` policy, followed by presigning the bridge and Lightning channel
setup over the bridge's funding output.

```
   board tx (user's on-chain wallet)                           [chain anchor]
        │
        ▼
   board output: cosign taproot (musig(A,S), S, expiry)
        │  (cosigned exit transaction, held off-chain)
        ▼
   channel VTXO output: cosign taproot (musig(A,S), S, expiry)
        │  (bridge transaction, presigned, held off-chain; nSequence = exit_delta)
        ▼
   funding output: P2WSH 2-of-2 (BOLT-3 funding keys)   ← channel's funding outpoint
        │  (Lightning commitment transaction, held off-chain — virtual funding)
        ▼
   commitment outputs: to_local / to_remote / HTLCs            (BOLT-3)
```

### Board

The user boards exactly as in ARK #3 — funding an on-chain **cosign taproot**
`(musig(A, S), S, expiry_height)` (the chain anchor) and obtaining the server's
cosignature on a single exit transaction — with one difference: the exit
transaction's output 0 carries the `channel-funding` policy (ARK #2) instead of
`pubkey`. That output is the channel VTXO; it is spent by the bridge, whose
output is the channel's funding outpoint. The board fee is the server's `board`
fee, computed and applied as in ARK #3.

This uses the ARK #3 board cosign and registration ceremony, with two additions
to the board cosign request: a `channel_id` — which both marks this as a channel
board (so the server builds the leaf under the `channel-funding` policy
`musig(A, S)` rather than `pubkey`) and identifies the channel whose funding keys
the server uses — and a bridge nonce, so the server reconstructs and cosigns the
**bridge** in the same exchange (see "Messages"). The two `musig(A, S)`
cosignatures — leaf exit tx and bridge — are returned together; no separate bridge
RPC is added.

### Channel setup

The parties MUST negotiate the Ark channel type (see "The Ark channel type"),
which exchanges the two ordinary BOLT-3 funding pubkeys. With the bridge built
and cosigned, its output (`bridge_txid:0`) is given to the Lightning stack as the
funding outpoint, and the parties run Lightning channel establishment
(referenced) to exchange signatures on the initial commitment transaction, which
spends the funding output through the stock BOLT-3 2-of-2 (ECDSA) — *not* MuSig2.
The commitment is held off-chain and never broadcast unless the channel is
force-closed; the Lightning stack treats the funding output as confirmed (virtual
funding, informative).

### Safety gate

The user MUST NOT broadcast the on-chain board transaction until it holds (a)
the fully cosigned genesis exit chain of the channel VTXO (ARK #6), (b) the
cosigned bridge, and (c) a valid initial commitment transaction. In practice (c)
is observed as the Lightning stack holding a confirmed channel for the funding
outpoint — the initial commitment has been exchanged (`FundingSigned` received)
and a channel monitor exists for it. Broadcasting the board transaction is the
point of no return: before it, an aborted open costs nothing — the cosigned exit
transaction and bridge spend an output that never exists, as in ARK #3 — while
after it, the user must be able to recover the funds either cooperatively or by
force-close.

### Sequence

The Ark and LDK/BOLT steps interleave as follows (the user drives; the server
is both ASP and Lightning peer):

```
 1. [BOLT]  request_channel; open_channel / accept_channel → both BOLT-3 funding
            pubkeys known (ordinary Lightning keys, not A/S)
 2. [local] build the board tx (exit-tx output 0 = channel-funding policy) and
            the bridge over the channel VTXO output (out0 = P2WSH 2-of-2 funding,
            out1 = P2A, nSequence = exit_delta)
 3. [Ark]   RequestBoardCosign (channel_id, + leaf nonce, + bridge nonce) → ASP
            reconstructs and cosigns the musig(A,S) leaf exit tx AND the bridge
            (using the channel's funding keys it already holds); finalize both
 4. [BOLT]  provide bridge_txid:0 as the funding outpoint; exchange the initial
            commitment (FundingCreated / FundingSigned) — stock ECDSA 2-of-2
 5. [gate]  SAFETY GATE: genesis exit chain + bridge + initial commitment exist
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
* the channel VTXO output and the bridge transaction — exact construction and
  value (see "The channel VTXO and the bridge transaction");
* that the **bridge** spends the channel VTXO output through the key path, signed
  by `musig(A, S)`: a BIP-327 MuSig2 partial signature over the BIP-341 key-path
  (taproot-tweaked) sighash, aggregated from `A` and `S` sorted per ARK #0's
  KeySort. This is a one-shot cosignature collected in the board/leaf cosign (see
  "Messages"), not per-commitment. The commitment in turn spends the bridge's
  funding output through the stock BOLT-3 2-of-2 (ECDSA), with no Ark involvement
  per update.

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
  completion are exactly as in ARK #4. The client MUST verify the issued leaf's
  exit depth does not exceed `max_vtxo_exit_depth` — the bound the channel's
  `cltv_expiry_delta` budget assumes ("The Ark channel type") — and abort the
  refresh otherwise (a safe abort: the old channel is still intact), since a
  deeper leaf would break on-chain HTLC timing.
* **Fee.** The server's `refresh` fee (ARK #4) applies.

Refresh uses the ARK #4 round messages — round participation, the completion
steps (including leaf cosign), and `forfeit_vtxos`. The channel-specific
additions are: on the leaf cosign request, a `channel_id` (so the server maps the
new leaf to the channel being refreshed and looks up its funding keys) plus a
bridge nonce — so the server reconstructs and cosigns the new **bridge** in the
same exchange (the new bridge reuses the channel's funding keys; see "The teleport
protocol" and "Messages"); and an optional server liquidity bound on the round
participation response:

### Server liquidity adjustment

A channel refresh shrinks the backing VTXO just as a non-channel refresh does:
the new `channel-funding` output is smaller than the old channel capacity, and
the new commitment must reflect where the difference comes from. Write the old
capacity as `V_old` and the issued new output as `V_new`; the reduction
`V_old − V_new` has two parts, falling on opposite sides of the channel:

* the **`refresh` fee** (ARK #4), which the *client* pays — it comes off the
  **client's** side of the new commitment, exactly as a non-channel refresh
  leaves the user with a smaller VTXO. The fee value leaves the channel
  entirely: the server collects it as the round's input/output shortfall
  (ARK #4), so the channel capacity shrinks by it.
* an optional **liquidity withdrawal `X`** (≥ 0), inbound capacity the server is
  reclaiming — it comes off the **server's** side.

So the new commitment carries the client's balance reduced by the fee and the
server's balance reduced by `X`, with `V_new = V_old − fee − X`. The fee never
touches the server's side and `X` never touches the client's; with `fee = 0` and
`X = 0` both balances carry over unchanged (the pre-launch, zero-fee case).

When the client requests a channel-funding output, the server MAY decline the
requested amount and return, on the round participation response, a cap — the
maximum output it will issue (`counter_max_issuable_sat`), which folds in both
the fee it requires and the `X` it is withdrawing. The client MUST then either
request a `channel-funding` output no larger than the cap or abort the refresh;
it MUST NOT proceed at the original amount. Absent the cap, the participation is
accepted as usual.

The client derives the split locally rather than receiving it again on a
Lightning message: `fee` is the `refresh` fee it computes from the published
schedule (ARK #4), and `X = (V_old − V_new) − fee` is the rest of the reduction.
It carries `X` into the teleport as `responder_value_removal_sat` — the amount by
which the **server's** side of the new commitment is reduced (see "The teleport
protocol") — reduces its **own** side by `fee`, and bounds `X` by a local
maximum, aborting if the server's cap implies a larger withdrawal than it will
accept. A refresh requires the fee to fit on the client's side (`fee ≤` the
client's channel balance) and `X ≤` the server's; a channel whose client side
cannot cover the fee cannot be refreshed and must be closed instead (offboard or
force-close).

### Added by the channel layer: the teleport

Refreshing the backing VTXO moves the channel onto a new funding outpoint. The
Lightning layer quiesces the channel, settles in-flight HTLCs, and *teleports*
the channel onto the new funding output — re-pointing the existing channel at
the new funding outpoint rather than closing and reopening it — carrying the
client's balance forward, less the `refresh` fee the client pays out of its own
side (the server may additionally withdraw from its side; see "Server liquidity
adjustment"). Teleport is an Ark-specific message flow, not a BOLT
procedure; its messages and promotion rule are specified in "The teleport
protocol." The ordering is safety-critical and follows the
exit-before-forfeit discipline of ARK #4: the user MUST verify the new channel
VTXO is issued and its exit story is complete before forfeiting the old one.
The forfeit is the point of no return.

### Sequence

```
 1. [Ark]   SubmitRoundParticipation: forfeit the old channel VTXO, request a
            fresh channel-funding output  (→ unlock_hash)
            · if the server returns a cap (declining the requested amount),
              resubmit ≤ cap or abort (see "Server liquidity adjustment")
 2. [Ark]   poll RoundParticipationStatus until ISSUED (round tx broadcast)
 3. [gate]  MONEY GATE: the round tx is confirmed to the required depth → the
            new funding is now virtually confirmed
            ─────────── LAST SAFE ABORT (old channel still intact) ───────────
 4. [local] build the new bridge over the new VTXO output, reusing the channel's
            existing funding keys → new funding outpoint = new_bridge_txid:0
 5. [Ark]   RequestLeafVtxoCosign (channel id, + leaf nonce, + bridge nonce) →
            ASP cosigns the new leaf AND the new bridge (reusing the channel's
            funding keys); finalize + verify both musig(A,S) signatures
 6. [tele]  teleport: Stfu (quiesce) → settle in-flight HTLCs → TeleportInit
            (new_funding_txo = bridge:0, no nonces) / TeleportAck → exchange
            commitment_signed on the new funding outpoint (stock ECDSA); wait
            until the peer has accepted it (see "The teleport protocol")
 7. [Ark]   durably record "forfeit sent", then RequestForfeitNonces +
            ForfeitVtxos for the OLD VTXO
                                                  *** POINT OF NO RETURN ***
            → unlock_preimage (the old VTXO is now forfeited to the server)
 8. [tele]  TeleportComplete/CompleteAck → both sides promote the new funding;
            the channel stays open throughout (no ChannelReady is emitted)
 9. [local] use unlock_preimage to finalize the new VTXO's exit path; remove
            the old exit path
```

## The teleport protocol

Refresh re-points an *open* channel from its old funding outpoint to the new
`channel-funding` output without closing it. The mechanism is **teleport**, an
Ark-specific message flow carried over the Lightning transport (not the Ark
service, and not a BOLT procedure). It is presented here as referenced
machinery the refresh flow drives; an interoperable implementation MUST
reproduce the message exchange and the promotion rule below.

**Prerequisites.** Before teleport begins, the channel MUST be quiesced (BOLT-2
`stfu`) with all in-flight HTLCs settled, the new VTXO MUST be issued and
virtually confirmed, and its **bridge** MUST be built and cosigned (within the
leaf cosign; see "Refresh"). The teleport itself does no `musig(A, S)` signing:
the new funding output is the bridge's stock P2WSH 2-of-2, so the new-scope
commitments are signed with the ordinary BOLT-3 funding keys (ECDSA), exactly as
in "Operation."

**Messages.** Teleport is a five-message flow between the initiator (the
client) and the responder (the server):

| Message | Direction | Fields |
|---|---|---|
| `teleport_init` | initiator → responder | `channel_id`; `new_funding_txo` (the new bridge's output 0); `responder_value_removal_sat` (the server-side liquidity withdrawal `X` of "Server liquidity adjustment" — the client's refresh-fee reduction falls on its own side and is not carried here) |
| `teleport_ack` | responder → initiator | `channel_id` |
| `teleport_complete` | initiator → responder | `channel_id` |
| `teleport_complete_ack` | responder → initiator | `channel_id` |
| `teleport_abort` | either | `channel_id` |

After `teleport_init` / `teleport_ack`, the parties exchange BOLT
`commitment_signed` for the **new** funding scope (identified by the new funding
txid), producing a valid commitment on the new outpoint while the old one is
still live. On receiving `teleport_init`, the responder MUST check that
`new_funding_txo` is output 0 of the bridge it cosigned for this channel — the new
funding outpoint it reconstructed at the leaf cosign (see "Refresh") — and abort
with `teleport_abort` on any mismatch, since that would mean the Lightning
transport and the Ark-service bridge cosign disagree on the new scope.

**Funding keys for the new scope.** The new funding output is the bridge's P2WSH
2-of-2 of the channel's funding keys. The teleport **reuses** those keys — it does
not rotate them per scope. This is a deliberate divergence from LDK splicing,
which derives a fresh per-splice funding key for on-chain unlinkability; here the
bridge and commitment normally never hit the chain, so reuse costs no real
privacy and lets both sides build the new bridge with no key exchange. Because the
funding spend is the stock BOLT-3 2-of-2, there is **no** MuSig2 nonce exchange in
the teleport — the new bridge's single `musig(A, S)` cosignature is collected
once, in the leaf cosign (see "Refresh"), not per commitment.

**Commitment numbering continues across scopes.** The teleport re-points the same
channel, so both scopes share the channel's per-commitment-secret seed and
per-commitment basepoints (the funding keys are reused, not rotated, above). The
new scope MUST therefore **continue** the channel's existing commitment-number
sequence and per-commitment-point derivation — it MUST NOT reset them. Resetting
would reuse per-commitment secrets already revealed on the old scope (for every
commitment number the old scope advanced past), pre-revoking the new scope's
commitments at those numbers and letting the counterparty sweep them through the
revocation path — a catastrophic loss. LDK splicing sidesteps this by rotating the
funding key per scope; the teleport reuses it, so commitment-number continuity is
what keeps each per-commitment secret bound to a single scope.

**Promotion.** Promotion — making the new funding scope the channel's live
funding — is asymmetric: the responder promotes on receiving `teleport_complete`,
the initiator on receiving `teleport_complete_ack`. The channel never leaves its
normal operating state and emits no `channel_ready`. A `teleport_abort` (or a
disconnect before promotion) leaves the channel on its old funding outpoint; the
flow survives reconnection via the standard `channel_reestablish`, which carries
both funding scopes until promotion resolves.

**Ordering against the forfeit.** Teleport's commitment exchange — `teleport_init`
through the new-scope `commitment_signed` — completes *before* the old channel
VTXO is forfeited. Promotion (`teleport_complete` / `teleport_complete_ack`)
happens *after* the forfeit returns the unlock preimage. Teleport completion
does not consume the unlock preimage: the preimage finalizes the new VTXO's
*exit* path (ARK #4), not the channel state. The forfeit is the point of no
return (see "Refresh").

**Dual-scope safety.** Between the new-scope `commitment_signed` and promotion,
both funding scopes carry a valid commitment; each party MUST run a chain monitor
for **both** funding outpoints until promotion resolves. The forfeit, not a
teleport message, is the pivot: once the old VTXO is forfeited the new scope is
authoritative, and the client MUST NOT broadcast the old-scope commitment — an
old-scope force-close would lose the channel VTXO output to the server's forfeit
(which spends it with no timelock, beating the old bridge's `exit_delta`), so the
client's balance is recoverable only on the new scope. Symmetrically, the server
MUST confirm the forfeit has completed before promoting on `teleport_complete`:
promoting — and discarding the old scope — while the client still holds an
unforfeited old VTXO would let it claim on both scopes.

## Offboard

A cooperative channel close is an offboard (ARK #7): the server pays the user's
channel balance to an on-chain destination, and the user forfeits the channel
VTXO to the server through the connector swap. The bridge and commitment, both
held off-chain, are simply discarded.

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
 6. [local] broadcast the offboard tx → client paid on-chain; tear the channel
            down once the offboard tx confirms (not before)
```

## Unilateral exit / force-close

When the server is unavailable or uncooperative, the user recovers its funds
with no help, by exiting the channel VTXO on-chain, broadcasting the bridge, and
force-closing the channel. The halves compose: the VTXO exit (ARK #6) brings the
channel VTXO output on-chain; the bridge actualizes the funding output; the
force-close (BOLT) then spends it.

1. **Exit the VTXO (ARK #6).** Broadcast the channel VTXO's genesis chain, root
   first, until the `channel-funding` output is confirmed on-chain. This is an
   ordinary emergency exit: TRUC (v3) transactions, P2A/CPFP fee-bumping, each
   level after its parent confirms.
2. **Broadcast the bridge.** The presigned bridge spends the channel VTXO output
   and produces the funding output. Its input carries a relative timelock of
   `exit_delta` (`nSequence`; see "The channel VTXO and the bridge transaction"),
   so the bridge becomes valid only `exit_delta` blocks after the VTXO output
   confirms — the channel's unilateral-exit delay, the *same* a `pubkey` exit
   waits (ARK #6). It is a TRUC v3, P2A/CPFP transaction like the genesis chain.
3. **Force-close.** Broadcast the latest commitment transaction, which spends the
   now-on-chain funding output through the stock BOLT-3 2-of-2 (ECDSA). The
   commitment carries no extra delay of its own — the exit delay was already
   served on the bridge input — so it confirms once the bridge does.
4. **Claim.** Once the commitment confirms, claim its outputs (`to_local`,
   `to_remote`, and resolved HTLCs) per the Ark channel type's BOLT-3 rules
   (referenced).

Unlike a `pubkey` exit (ARK #6), there is no VTXO-level `delayed-sign` claim; the
channel VTXO's value is claimed only through the bridge and then the commitment.

### Server recourse after expiry

The channel VTXO output's `timelock-sign(expiry_height, S)` leaf lets the server,
after `expiry_height`, sweep the channel VTXO output directly — without the bridge
or commitment — just as it may sweep any unspent cosign-taproot output past expiry
(ARK #4, ARK #6). This is the server's recourse when a user has actualized the
VTXO output on-chain but left the channel otherwise unresolved (for example,
exited the VTXO but never broadcast the bridge). It is bounded by `expiry_height`,
which is why the user MUST refresh or close before expiry.

### The refresh / force-close deadline

The expiry sweep above sets a hard deadline: the user MUST get the **bridge
confirmed before `expiry_height`**. Once the bridge confirms, the funding output
is an ordinary on-chain UTXO and the expiry-sweep leaf — which sat on the
now-spent VTXO output — can no longer reach it; the commitment and any HTLCs then
resolve on their own BOLT-3 timelocks, no longer bounded by `expiry_height`.
Before the bridge confirms, the whole channel VTXO — both balances and any
in-flight HTLC value — is exposed to the sweep.

Reaching that point unilaterally takes, in series: the genesis exit chain (the
VTXO's exit depth, at most `max_vtxo_exit_depth`, each level confirming), the
bridge's `exit_delta` relative timelock, and the bridge's own confirmation, plus
fee-bumping slack. The user MUST therefore begin a force-close at least
`max_vtxo_exit_depth + exit_delta` blocks — plus confirmation and fee-bumping
slack for the genesis levels, the bridge, and the commitment — before
`expiry_height`, and SHOULD refresh, which resets `expiry_height`, well before
that: the same exit-before-expiry discipline as any VTXO (ARK #6), with the
bridge's `exit_delta` added to the margin.

This interacts with HTLCs. A refresh first quiesces the channel and settles all
in-flight HTLCs (see "Refresh"), so an HTLC that cannot be settled — an
unresponsive peer, a withheld preimage — **blocks the refresh** and forces a
unilateral force-close instead. A node MUST therefore stop offering and forwarding
HTLCs on the channel, and refresh or force-close, once `expiry_height − height`
falls to the force-close margin above plus headroom for any on-chain HTLC
resolution it must still drive (the `cltv_expiry_delta` budget of "The Ark channel
type"). A channel carried too close to expiry with an unsettleable HTLC can lose
the entire VTXO to the sweep.

## Messages

Channels reuse the Ark messages of ARK #3 (board), ARK #4 (round), and ARK #7
(offboard); the sections below give the channel variant of each affected
message in full — base fields as defined in its home document, with the
channel-specific fields called out. Every channel-specific field is absent for
a non-channel operation, and a peer that does not implement channels ignores it
(see "Compatibility"); no other Ark message changes for channels, and channels
add no new Ark RPC. (The refresh teleport adds messages on the Lightning
transport, not the Ark service; see "The teleport protocol.")

### `board_cosign_request` (ARK #3)

The channel variant is the ARK #3 `board_cosign_request` with two added fields, so
the server cosigns both the leaf exit tx and the bridge in one exchange. Sent by
the user to the server:

| Field | Type | Meaning |
|---|---|---|
| `amount` | u64 sats | value of the channel VTXO output (gross, before fee) |
| `utxo` | outpoint | the on-chain board outpoint |
| `expiry_height` | u32 | chosen expiry height |
| `user_pubkey` | pubkey | `A` |
| `pub_nonce` | musig_pub_nonce | the user's MuSig2 public nonce for the leaf exit tx, bound to its sighash and the tweaked `musig(A, S)` (ARK #3) |
| `channel_id` | 32 bytes | the LDK channel id of the channel being opened; its presence marks this as a channel board (absent for a plain `pubkey` board) |
| `bridge_pub_nonce` | musig_pub_nonce | the user's MuSig2 public nonce for the **bridge** key-path sighash (present only for a channel board) |

The first five fields are exactly ARK #3. `A` (`user_pubkey`) is the user's
cooperative key, *not* a funding key; the channel's BOLT-3 funding keys are
ordinary Lightning keys. They are **not** sent here — the server already holds
both (its own from `accept_channel`, the client's from `open_channel`) keyed by
`channel_id` — so the request need only identify the channel and supply the bridge
nonce.

From these the server reconstructs (a) the leaf exit output under the
`channel-funding` policy `musig(user_pubkey, S)`, and (b) the bridge — input = the
leaf output (keyspend `musig(A, S)`, `nSequence = exit_delta`), output 0 = P2WSH
2-of-2 of the channel's two funding keys (looked up by `channel_id`), output 1 =
P2A, 0-fee (see "The channel VTXO and the bridge transaction"). The response is
the ARK #3 `board_cosign_response` (server `pub_nonce` + `partial_sig` for the
leaf), extended with the server's bridge `pub_nonce` + `partial_sig`.

Requirements (server): the ARK #3 board requirements apply, and additionally —

* When `channel_id` is present, the server MUST build the exit output with the
  `channel-funding` policy `musig(user_pubkey, server_pubkey)` (ARK #2) and the
  bridge above — using the funding keys it holds for that channel — and cosign
  both. If `channel_id` names no channel it knows (with a funding key it
  controls), it MUST NOT cosign; absent `channel_id` it is a plain `pubkey` board
  (ARK #3). The named channel MUST be one the server is opening — funding keys
  exchanged, no funding outpoint yet assigned — so a board cosign cannot be
  redirected at an already-funded channel. No pinning check is otherwise needed: a
  client that built a different leaf or bridge would simply see the cosignature
  fail to verify (as with the board fee, ARK #3).

Nonce freshness (both the board and leaf cosigns): the user commits its
`bridge_pub_nonce` (and the leaf `pub_nonce`) in the request and produces its own
partial only after the server's first-signer one-shot response. On any retry the
user MUST generate **fresh** secret nonces for both — the server returns a
different nonce each time, so reusing a user secret nonce against two different
aggregate nonces would leak the user's key (the nonce-reuse discipline of ARK #4
failure handling).

### `leaf_vtxo_cosign` (ARK #4)

Request (channel refresh): the ARK #4 fields — `vtxo_id`, user `pub_nonce` (bound
to the leaf sighash) — plus, for a channel leaf: `channel_id` (32 bytes, the LDK
channel id of the channel being refreshed) and `bridge_pub_nonce` (the user's
nonce for the new bridge's key-path sighash). Both are absent for a non-channel
leaf. Response: as ARK #4 (server `pub_nonce`, `partial_sig`, first-signer
one-shot), extended with the server's bridge `pub_nonce` + `partial_sig`.

As in the board cosign, the server reconstructs and cosigns both the new leaf exit
tx and the new **bridge** in this one exchange, using the funding keys it already
holds for `channel_id` — the new bridge reuses the channel's existing funding keys
(it does not rotate them; see "The teleport protocol"). If `channel_id` names no
channel it knows (with a funding key it controls), it MUST NOT cosign. The server
also independently learns the channel from the forfeited input on the round
participation and from `teleport_init`, and MUST verify these agree: the channel
named by `channel_id`, its forfeited old VTXO, and the new leaf `vtxo_id` MUST be
consistent. The server MUST refuse to cosign a bridge that would move any VTXO
other than this channel's expected refresh leaf into the channel's funding 2-of-2.

### `submit_round_participation` response (ARK #4)

The ARK #4 response carries `unlock_hash` (32 bytes). For a channel-refresh
participation the server MAY instead return a cap in its place:

| Field | Type | Presence |
|---|---|---|
| `unlock_hash` | 32 bytes | accepted — identifies the participation (ARK #4) |
| `counter_max_issuable_sat` | u64 sats | capped — the maximum `channel-funding` output the server will issue when it declines the requested amount (at most `V_old − fee − X`, folding in the required fee and its liquidity withdrawal `X`; see "Server liquidity adjustment") |

Exactly one of the two is present. Requirements (participant): on receiving
`counter_max_issuable_sat` (and no `unlock_hash`), the participant MUST resubmit
requesting a `channel-funding` output no larger than the cap, or abandon the
refresh; it MUST NOT proceed at the original amount.

## Compatibility

The channel work extends the protocol without disturbing existing flows.

* **Additive messages.** The channel additions on the board cosign request
  (`channel_id` + `bridge_pub_nonce`), the leaf cosign request (`channel_id` +
  `bridge_pub_nonce`), and the round participation response are optional. A peer
  that does not implement channels ignores them, and the board, refresh, and
  offboard flows are unchanged for non-channel VTXOs.
* **Channel support is a capability, not a default.** A channel open or refresh
  requires a channel-aware server. An older server ignores `channel_id` (and the
  bridge nonce) and would cosign an ordinary `pubkey` leaf; the client detects
  this — the cosignature does not validate against the expected `channel-funding`
  output, and no bridge cosignature is returned — and the open fails safely, since
  the board transaction is never broadcast without a valid exit and bridge. It
  does not silently produce a non-channel VTXO. A server
  advertises channel support with the `supports_channels` flag in ark info
  (ARK #0); a client MUST NOT attempt a channel open against a server that does
  not set it.
* **New policy type.** A `channel-funding` VTXO (policy type `0x08`) is rejected
  by decoders that predate it (ARK #2 decoders reject unknown policy type
  bytes), so channel VTXOs are meaningful only between channel-aware parties.

## Security and trust notes

* **Exit guarantee preserved.** A channel VTXO carries a complete, cosigned
  genesis exit chain like any VTXO (ARK #2, ARK #6); together with the bridge and
  commitment transaction it always lets the user recover its channel balance
  on-chain without the server. The server cannot steal from a user who exits
  and force-closes in time.
* **Atomic give-up.** Because a channel VTXO is forfeited by the standard
  `musig(A, S)` hArk swap (ARK #4) or connector swap (ARK #7), refreshing or
  closing a channel hands the old VTXO to the server only in a history where
  the user has received its replacement or its payout. There is no window in
  which the user has surrendered the old channel VTXO without compensation.
* **Expiry is a deadline.** The server may sweep the channel VTXO output after
  `expiry_height`. A channel left un-refreshed and un-closed past expiry can
  lose its funds. Users MUST refresh well before expiry — early enough to confirm
  the bridge first, and before an unsettleable HTLC can block the refresh (see
  "The refresh / force-close deadline") — exactly as for any VTXO (ARK #0), with
  the bridge's `exit_delta` added to the margin.
* **Virtual funding.** The channel operates as though its funding output were
  confirmed while that output — the bridge's output 0 — lives only in the
  off-chain exit chain. The safety of this rests on the exit guarantee above: the
  funding output can always be actualized on-chain by broadcasting the genesis
  chain and then the bridge. The mechanism by which the Lightning stack is told
  the funding is confirmed is out of scope (informative).
