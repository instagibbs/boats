# ARK #8: Channels

This document specifies Lightning channels built on the Ark protocol: how a
channel is funded by a VTXO, operated, refreshed, closed cooperatively, and
exited unilaterally. It builds on VTXOs and policies (ARK #2), boarding
(ARK #3), rounds (ARK #4), out-of-round payments (ARK #5), emergency exit
(ARK #6), and offboarding (ARK #7).

The channel's payment machinery — the commitment transaction, its outputs and
HTLCs, channel quiescence, the shutdown and closing flow, and the channel
state machine — is that of the Lightning Network (BOLT-3 and related), with
the deviations collected in "The
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

* **Open** ("Channel open"): the user funds a channel VTXO and presigns the
  bridge, then sets up the Lightning channel over the bridge's funding
  output. The channel VTXO comes from one of two sources: a board (ARK #3)
  into a depth-1 channel VTXO, or an **upgrade** — an out-of-round self-spend
  (ARK #5) of a VTXO the user already holds ("Open by upgrade").
* **Operate** ("Operation"): payments flow over the channel per BOLT; the
  funding VTXO sits unchanged.
* **Refresh** ("Refresh"): before expiry, a round (ARK #4) forfeits the
  channel VTXO and issues a fresh one, resetting expiry and exit depth while
  preserving the channel balance.
* **Offboard** ("Offboard"): a cooperative close — the BOLT close fixes the
  final balances, then an offboard (ARK #7) forfeits the channel VTXO and
  pays the user's final balance on-chain.
* **Unilateral exit / force-close** ("Unilateral exit"): the user recovers its
  funds on-chain with no server cooperation.

Open, refresh, and offboard reuse the Ark RPCs and ceremonies of ARK #3 or
ARK #5 (open), ARK #4 (refresh), and ARK #7 (offboard): there are no
channel-specific Ark RPCs, the forfeit object
is the standard one (ARK #4), and channel establishment itself runs over the
Lightning transport, not the Ark service. The new Ark protocol objects are the
`channel-funding` policy (ARK #2) and the presigned **bridge transaction**,
which is cosigned *within* the existing board/arkoor/leaf cosign rather than via
a new RPC. Existing requests carry optional channel fields, described where they
are used: the board cosign request a `channel_id` and a bridge nonce; the arkoor
cosign request the same pair; the leaf cosign
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
  board transaction for a board-opened channel, the round transaction (the
  tree root) for a channel established by a refresh, or the input VTXO's
  existing anchor — already confirmed before the open began — for a channel
  opened by upgrade ("Open by upgrade"). Once that anchor has
  confirmed, the user holds a fully exitable VTXO whose bridge actualizes the
  funding output on-chain; treating it as confirmed is therefore sound. Virtual
  funding is a logical confirmation tied to that anchor, and it changes nothing
  else about the node's view of the chain: the funding is presented as confirmed
  at the anchor's actual best-chain height, and an implementation MUST NOT
  advance its best-chain view beyond the real best chain to manufacture virtual
  depth. Every decision that depends on chain height — channel operation,
  HTLC/invoice CLTV validation, expiry and exit scheduling — MUST observe one
  consistent, real-chain history, including reorganizations and unconfirmations
  in order; virtual funding is not a license for one part of the node to run on
  a different chain view than another. An anchor disconnection withdraws the
  virtual confirmation, suspending normal channel operation until a valid
  anchor is re-established. A restart MUST NOT weaken any of this: a node MUST
  NOT accept or forward HTLCs on a channel until it again holds a coherent
  chain view and funding status for it, and if it cannot recover them it MUST
  leave the channel fail-closed rather than rebuild a fresh state and continue.
* **Expiry is a deadline.** Like every VTXO, a channel VTXO has an
  `expiry_height` after which the server may sweep its backing funds. The user
  MUST refresh or close the channel before expiry — with enough margin to confirm
  the bridge first, and stopping new HTLCs as the deadline nears, since a forced
  force-close must still resolve any in-flight HTLC on-chain in time (see "The
  refresh / force-close deadline"). A channel left un-refreshed past expiry can lose its
  funds to the server's sweep.
* **Liveness for cooperation.** Refresh and cooperative close require a
  responsive server; unilateral exit is always available without one.
* **On-chain fee reserve for exit.** Unilateral exit is a longer 0-fee TRUC chain
  than a plain VTXO exit — the genesis levels, then the bridge, the commitment, and
  any HTLC second-stage transactions — each CPFP-bumped at broadcast (ARK #6). The
  user therefore needs spendable on-chain funds to fee-bump that chain within the
  force-close deadline (see "The refresh / force-close deadline"); a user with no
  fee reserve can miss the deadline and lose the channel to the expiry sweep. This
  is the ordinary emergency-exit fee assumption (ARK #0), enlarged by the extra hops
  a channel exit adds.

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
  pinned_exit_delta`. `pinned_exit_delta` is fixed at open — the
  `vtxo_exit_delta` accepted at a board open (ARK #0), or the input VTXO's
  decoded `exit_delta` at an upgrade open ("Open by upgrade") — stored with
  the channel, and is not a live ark-info value.
* **Output 0 (funding):** a stock segwit-v0 **P2WSH 2-of-2** of the channel's two
  BOLT-3 funding pubkeys (Actors and keys), value = the channel VTXO's value
  (0-fee), which the open cosigns verify equals the channel's negotiated
  funding amount (see "Messages"). This is an ordinary BOLT-3 funding output — it carries *no* Ark sweep
  script: the server's expiry recourse is one hop up, on the VTXO output, and LDK
  sweeps this output with its own transactions, which have no custom-input support
  for an Ark clause. The commitment transaction spends it.
* **Output 1 (anchor):** a P2A `OP_1 <0x4e73>` of 0 value, CPFP-bumped at
  broadcast, exactly as the Ark exit transactions (ARK #6).

The bridge carries the channel's **unilateral-exit delay**. Because it is
presigned with `nSequence = pinned_exit_delta` and is the only unilateral on-chain spend
of the VTXO's key path (forfeit / refresh / offboard are cooperative and
off-chain), the delay is enforced by the bridge alone — no script on the VTXO
output, and no CSV on the commitment input. After a unilateral exit the bridge
confirms only `pinned_exit_delta` blocks after the channel VTXO output confirms — the
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
exit"). The client holds the fully signed bridge — its unilateral exit (the genesis chain,
then the bridge, then the commitment) depends on it. The server MAY retain it
too, but need not (the reference does not): its recourse to a channel left
stranded by an absent client is the expiry sweep of the channel VTXO output (see
"Server recourse after expiry"), which needs neither the bridge nor the
commitment. A server that does retain the bridge can force-close before expiry
by broadcasting bridge + commitment (e.g. to reclaim liquidity from an idle
channel early) — an optimization, not a requirement.

## The Ark channel type

A channel on Ark is not a vanilla BOLT channel: the parties MUST negotiate a
dedicated **Ark channel type**. It is not a registered BOLT feature — Ark-aware
peers agree on it out of band — but it is negotiated as an experimental feature
pair, bits `400/401` (the reference's `ark_channel` feature): peers advertise
the *optional* bit `401` in `init` (and `node_announcement`) — alongside the
optional `zero_fee_commitments` bit, so the dependency bundle is recognizable —
and at channel open the *required* bit `400` is the one set in the
`channel_type`, alongside the `static_remote_key` and `zero_fee_commitments`
bits it implies. The pair is experimental — chosen clear of allocated BOLT
bits — so the exact number is subject to change if the type is ever
standardized. The server MUST refuse to open a channel of any other type
(channel support on the Ark service is advertised by the `supports_channels`
flag, see "Compatibility").

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
  gains an extra `<exit_delta> OP_CSV OP_DROP`; the timeout branch carries no
  extra CSV. Every spend of the success branch — the presigned second-stage
  HTLC-success transaction and a direct preimage claim of the counterparty's
  commitment output alike — MUST set `nSequence` to at least `exit_delta` or it
  is consensus-invalid; the second-stage HTLC-success transaction is presigned
  with `nSequence = exit_delta`. The second-stage HTLC-timeout transaction is
  presigned with the baseline `nSequence` (0) — **not** `exit_delta`. The
  asymmetry is deliberate and normative: these are v3 transactions, so any
  `nSequence` below `0x80000000` is a real BIP-68 relative timelock, and
  presigning the timeout template at `exit_delta` would delay the offerer's
  own timeout claim by the very delta that exists to protect it, turning its
  head start into a fee-race tie (and re-opening the replacement-cycling
  window the head start closes). The HTLC signatures commit to `nSequence`,
  so peers that disagree on either template simply fail to verify each
  other's signatures. With the asymmetry, a timeout claim strictly leads a
  preimage claim by `exit_delta` on whichever commitment confirms — the
  direct timeout claim of a counterparty commitment carries no relative
  delay either. This is what guarantees that an HTLC whose absolute CLTV
  expired while the slow exit chain was confirming resolves
  deterministically to the timeout side rather than racing, so the offerer
  never depends on out-racing (or extracting a preimage from) a competing
  claim. This success CSV MUST equal the channel's pinned `exit_delta` (see
  "`exit_delta` is pinned at open" below); both peers use that fixed value
  rather than re-deriving it per update or negotiating it. BOLT-3 has no
  success-branch CSV. This CSV is a required construction parameter of the Ark
  channel type, not an optional feature or a defaultable setting: a node MUST
  NOT advertise, accept, or construct the Ark channel type unless a nonzero
  success CSV equal to the pinned `exit_delta` is in force on every HTLC
  success branch, and it MUST validate that binding before it advertises the
  type or commits the board. Both the bridge `nSequence` and the success CSV
  MUST derive from the single pinned value (see "`exit_delta` is pinned at
  open" below).

  The same invariant binds a channel restored from persistence: a stored scope
  whose success CSV is absent, zero, or too large for its CLTV floor MUST NOT
  be resumed or offered new HTLCs — and the CSV cannot be repaired from current
  configuration, since the commitment scripts it appears in are already signed.
  Force-closing the scope (or keeping it fail-closed) is the only safe
  disposition.

* **HTLC CLTV budget.** A receiver MUST reject an incoming HTLC whose CLTV
  budget is too small, and the channel's minimum `cltv_expiry_delta` MUST be at
  least `2 * pinned_exit_delta + channel_max_vtxo_exit_depth +
  cltv_safety_margin`. Resolving an HTLC on-chain crosses **two**
  `pinned_exit_delta` delays in series: first to get
  the **commitment** confirmed — force-closing *through* the VTXO exit takes up to
  `channel_max_vtxo_exit_depth` genesis levels (ARK #0) plus the bridge, whose
  input is time-locked `pinned_exit_delta` — and then, on the confirmed
  commitment, the **HTLC success-path CSV** (above) delays the preimage claim a
  further `pinned_exit_delta`. Both must complete before the HTLC's absolute CLTV (after
  which the offerer's timeout path opens), which is why `pinned_exit_delta` appears
  twice. The serial chain a force-close climbs before the preimage claim is valid:

  ```
     force-close begins
          │  ≤ channel_max_vtxo_exit_depth genesis levels (each ~1 block)
          ▼
     channel VTXO output on-chain
          │  pinned_exit_delta    (bridge input nSequence CSV)
          ▼
     bridge confirmed → commitment confirms (no extra delay of its own)
          │  pinned_exit_delta    (HTLC success-path CSV)
          ▼
     HTLC-success spend valid  ── must land before the HTLC's absolute CLTV ──
  ```

  Worked example (defaults `pinned_exit_delta = 144`,
  `channel_max_vtxo_exit_depth = 16`): the
  floor is `2·144 + 16 + cltv_safety_margin = 304 + cltv_safety_margin` blocks
  (~2.1 days before the buffer; the reference sizes the margin at 18 blocks,
  for a 322-block floor). The common error is to budget only the success-path
  CSV — `1·pinned_exit_delta + cltv_safety_margin` — and fold the rest into the margin;
  that under-counts by a whole `pinned_exit_delta + channel_max_vtxo_exit_depth` and lets the
  offerer's timeout win the on-chain race. `cltv_safety_margin` is the ordinary
  Lightning confirmation buffer, but
  for an Ark channel it MUST be sized larger: it absorbs confirmation variance
  across the **whole serial chain** a force-close climbs — up to
  `channel_max_vtxo_exit_depth` genesis levels, then the bridge, the commitment, and the
  success-path claim — each budgeted at a single block in the floor but any of
  which can run longer under fee pressure (the non-Ark race is only
  commitment-then-claim; refreshing often to keep exit chains shallow keeps the
  needed margin realistic). The exit-derived terms come from
  the channel's pinned `exit_delta` (below) and from its pinned
  `channel_max_vtxo_exit_depth`, so both peers compute the same floor given the
  same buffer; each enforces it from its own configuration, not on the wire.
  Because these values are pinned per channel ("`exit_delta` is pinned at
  open"), the floor is a **per-channel** quantity, and a refresh — which reuses
  the pinned timing profile — does not move it. A receiver that commits to a
  single CLTV floor not bound to
  one channel — for example the `min_final_cltv_expiry_delta` of a BOLT-11 invoice
  payable over any of its channels — MUST therefore advertise the **maximum**
  floor across the channels it could receive on, so the budget holds whichever
  channel the payment lands on (the value is fixed when the invoice is issued). A
  payment that carries *no* receiver-committed CLTV floor — a **spontaneous
  (keysend) payment**, which has no invoice `min_final_cltv_expiry_delta` — cannot
  be budgeted this way at all, so a receiver MUST reject a keysend HTLC on an Ark
  channel (the reference fails it back). Keysend is therefore **unsupported** on
  Ark channels until a per-channel floor can be enforced for it.
  `channel_max_vtxo_exit_depth`
  is also the upper bound the budget assumes on the **channel VTXO's own** exit
  depth — the number of genesis levels a force-close must climb: a channel VTXO
  is only ever a board leaf (depth 1), an upgrade output ("Open by upgrade"),
  or a refresh round leaf, and the client MUST refuse a refresh whose issued
  leaf would exceed it (see "Refresh") — or an upgrade whose resulting depth
  would ("Open by upgrade") — since a deeper exit chain would overrun the
  budget.

  Every timing-profile calculation MUST use checked arithmetic in a type wider
  than its encoded fields. BOLT's channel `cltv_expiry_delta` is a `u16`, so the
  complete floor
  `2 * pinned_exit_delta + channel_max_vtxo_exit_depth + cltv_safety_margin`
  MUST be no greater than `65535`, and a node MUST NOT advertise, open, or
  accept an Ark channel whose complete floor is unrepresentable or whose
  configured CLTV floor is smaller. An implementation that enforces the floor's
  terms in separate layers MUST still check the complete floor before the
  channel is enabled. Saturating an overflowing floor is non-conforming: it
  silently admits HTLCs with less time than the on-chain path requires.

* **Virtual funding.** The channel uses manual funding at zero confirmation
  depth — it becomes usable without its funding (bridge) transaction appearing
  on-chain (see "Trust assumptions").

**`exit_delta` is pinned at open.** A channel's `exit_delta` — the value on the
bridge input's `nSequence`, in the HTLC success-path CSV, and in the CLTV budget
above — is fixed **once, at open** — read from `vtxo_exit_delta` (ARK #0) at a
board open, or from the input VTXO's decoded `exit_delta` at an upgrade open
("Open by upgrade") — and is
thereafter a fixed parameter of the channel. Both parties MUST store it (with the
channel's other parameters, keyed by `channel_id`) and use the stored value for
every later commitment and for the bridge of every refresh; neither re-reads
`vtxo_exit_delta` from ark info for an existing channel. No wire field carries
it. Cross-peer agreement is enforced at open by the **bridge cosign**: the
`musig(A, S)` key-path sighash commits the bridge input's `nSequence`, so peers
holding different values fail to verify each other's partial signatures and the
open aborts with nothing on-chain. The **HTLC success-path CSV is not what
catches a disagreement**: the initial commitment carries no HTLC outputs, so
nothing signed at establishment commits to the success CSV, and a divergence
confined to it — one peer's success CSV drifting from its own bridge
`nSequence` — stays hidden through a successful open and surfaces only at the
first HTLC's commitment exchange, whose signatures fail to validate and
force-close an established channel *after* the board transaction confirmed
(the point of no return). An implementation MUST therefore derive the success
CSV and the bridge `nSequence` from the single pinned value rather than
configuring them independently: the protocol re-verifies the bridge copy at
every cosign, but a split between a peer's own two copies is caught by nothing
until that first-HTLC force-close. A
refresh MUST build the new bridge at the pinned value, and the server,
reconstructing that bridge from `channel_id`, MUST use the channel's stored
`exit_delta` rather than its current ark-info value; if the server's
`vtxo_exit_delta` has since changed and it will not cosign at the pinned value,
the refresh fails and the channel must be closed and reopened.

**The channel timing profile is pinned at open.** At open both parties MUST
record the pair `(pinned_exit_delta, channel_max_vtxo_exit_depth)`, where
`channel_max_vtxo_exit_depth` is the maximum VTXO exit depth on which this
channel's CLTV floor was based. The channel's CLTV base is
`2 * pinned_exit_delta + channel_max_vtxo_exit_depth`; a receiver MUST retain
the floor committed in each outstanding invoice and MUST NOT later apply a
smaller floor. Ark info is an admission policy for new channels, not a source of
timing parameters for an existing channel.

Every timing decision for a backing scope binds to values fixed when that scope
was issued: the absolute `expiry_height` and actual exit depth decoded from the
VTXO itself, and the channel's pinned timing profile. A refresh is valid only if
the decoded new VTXO has `exit_delta == pinned_exit_delta` and actual exit depth
`<= channel_max_vtxo_exit_depth`; a configuration change cannot raise either
bound for an existing channel. HTLC admission, invoice validation, the
stop-forwarding threshold, refresh scheduling, and force-close scheduling MUST
use those scope-fixed values together with the real chain height, across
restarts. They MUST NOT recompute expiry as anchor/root height plus a current
`vtxo_expiry_delta`, or substitute a current ark-info exit parameter or depth
bound. A peer unable to issue or cosign a scope meeting the stored profile MUST
refuse refresh, leaving the client to refresh earlier or force-close.

## Channel open

A channel opens by bringing a `channel-funding` VTXO into existence, presigning
the bridge, and running Lightning channel setup over the bridge's funding
output. The VTXO comes to exist one of two ways: a **board** (ARK #3) whose
resulting VTXO carries the `channel-funding` policy — an open from on-chain
funds — or an **upgrade** — an out-of-round self-spend (ARK #5) of a VTXO the
user already holds: an open from off-chain balance, with no round, no on-chain
footprint, and no confirmation wait. The subsections through "Sequence" specify
the board path; "Open by upgrade" specifies the upgrade path, sharing "Channel
setup".

```
   board tx (user's on-chain wallet)                           [chain anchor]
        │
        ▼
   board output: cosign taproot (musig(A,S), S, expiry)
        │  (cosigned exit transaction, held off-chain)
        ▼
   channel VTXO output: cosign taproot (musig(A,S), S, expiry)
        │  (bridge transaction, presigned, held off-chain; nSequence = pinned_exit_delta)
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

The ARK #3 expiry range applies unchanged to a channel board. In particular, a
channel request does **not** authorize a client-selected long-lived backing
output: before it produces either partial signature, the server MUST reject an
`expiry_height` above its current tip plus `vtxo_expiry_delta` and its bounded
tip-skew allowance. Otherwise an absent client could choose a far-future expiry
(for example `u32::MAX`) and leave the channel scope unresolved while any
balance the server later earns remains locked behind the client-held exit. The
expiry sweep is the server's only **protocol-guaranteed** unilateral recovery.
As noted above, a server that separately retained the complete bridge may be able
to force-close earlier, but that optional optimization does not replace the
mandatory admission bound.

### Channel setup

The parties MUST negotiate the Ark channel type (see "The Ark channel type"),
which exchanges the two ordinary BOLT-3 funding pubkeys. With the bridge built
and cosigned, its output (`bridge_txid:0`) is given to the Lightning stack as the
funding outpoint, and the parties run Lightning channel establishment
(referenced) to exchange signatures on the initial commitment transaction, which
spends the funding output through the stock BOLT-3 2-of-2 (ECDSA) — *not* MuSig2.
The commitment is held off-chain and never broadcast unless the channel is
force-closed; the Lightning stack treats the funding output as confirmed (virtual
funding; see the normative chain-feed requirements in "Trust assumptions").

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
            out1 = P2A, nSequence = pinned_exit_delta)
 3. [Ark]   RequestBoardCosign (channel_id, + leaf nonce, + bridge nonce) →
            server reconstructs and cosigns the musig(A,S) leaf exit tx AND the
            bridge (using the channel's funding keys it already holds);
            finalize both
 4. [BOLT]  provide bridge_txid:0 as the funding outpoint; exchange the initial
            commitment (FundingCreated / FundingSigned) — stock ECDSA 2-of-2
 5. [gate]  SAFETY GATE: genesis exit chain + bridge + initial commitment exist
            ──────────────── LAST SAFE ABORT (nothing on-chain) ──────────────
 6. [local] sign + broadcast the board tx          *** POINT OF NO RETURN ***
 7. [local] board tx confirms → virtually confirm the funding (board-tx height)
 8. [Ark]   RegisterBoardVtxo
 9. [BOLT]  ChannelReady — the channel is usable
```

### Open by upgrade

An **upgrade** opens a channel from off-chain balance: an out-of-round transfer
(ARK #5) in which the user self-spends an existing `pubkey` VTXO into a
destination carrying the `channel-funding` policy. The transfer is a standard
ARK #5 package — checkpoint machinery, balance and dust rules, signing,
registration, all unchanged — with the board cosign's two channel fields added
to the arkoor cosign request (`channel_id` and a bridge nonce), so the server
reconstructs and cosigns the **bridge** in the same exchange (see "Messages").
Because the input VTXO's chain anchor confirmed before the open began, there is
nothing to wait for: the funding is virtually confirmed at that anchor's height
("Trust assumptions"), and the channel is usable the moment the ceremony
completes.

```
   input `pubkey` VTXO output (held, off-chain)             [chain anchor: the
        │  (checkpoint tx, when used — see below)            input's, already
        ▼                                                    confirmed]
   channel VTXO output: cosign taproot (musig(A,S), S, expiry)
        │  (bridge transaction, presigned, held off-chain;
        │   nSequence = pinned_exit_delta)
        ▼
   funding output: P2WSH 2-of-2 (BOLT-3 funding keys)   ← channel's funding outpoint
        │  (Lightning commitment transaction, held off-chain — virtual funding)
        ▼
   commitment outputs: to_local / to_remote / HTLCs            (BOLT-3)
```

**The transfer.** A package MUST carry exactly one `channel-funding`
destination, and it MUST be a normal destination (never isolated, ARK #5). The
rest of the destination set is unrestricted: change back to the user's own
`pubkey` policy — sizing the channel below the input's value — or any other
destinations, all under the unchanged ARK #5 balance, dust, and checkpoint
rules. One shape gets a carve-out: for a single-part package whose destination
set is exactly the one `channel-funding` output, there is no co-recipient for
the checkpoint to isolate — it degenerates to a pass-through hop — so the
server SHOULD accept `use_checkpoint = false` there, making the scope one exit
level shallower and every later server broadcast one transaction instead of
two; every other shape follows the server's ordinary checkpoint policy
(ARK #5). The `channel-funding` destination's `user_pubkey` MUST equal the
input VTXO's user key: the requester is at once the arkoor sender and the
channel user, which is what lets a single request carry both the transfer
nonces and the bridge nonce. Its amount MUST equal the channel's negotiated
funding amount — the bridge's funding output carries exactly this value, and
the cosign verifies the equality ("Messages"). The new VTXO inherits the input's
`expiry_height`, `server_pubkey`, `exit_delta`, and `anchor_point` (ARK #5);
the upgrade resets nothing — the next channel refresh does, as it would anyway.
There is no fee: an upgrade commits no new server capital and inherits the
scope's existing maturity, so like any arkoor transfer it is not separately
priced.

**Ordering and the safety gate.** The arkoor transactions' txids are
independent of their witnesses, so the client computes the channel VTXO's
`point` — and from it the bridge txid — before anything is signed. The parties
therefore run Lightning establishment *before* the cosign, and the board's
safety gate holds with its point of no return moved from "broadcast" to
"register":

```
 1. [BOLT]  request_channel; open_channel / accept_channel → both BOLT-3
            funding pubkeys known
 2. [local] build the transfer's transactions (one destination =
            channel-funding policy) and the bridge over the channel VTXO
            output (out0 = P2WSH 2-of-2 funding, out1 = P2A,
            nSequence = pinned_exit_delta)
 3. [BOLT]  provide bridge_txid:0 as the funding outpoint; exchange the
            initial commitment (FundingCreated / FundingSigned) — stock
            ECDSA 2-of-2
 4. [gate]  initial commitment held
            ──────────────── LAST FULLY-FREE ABORT ────────────────
 5. [Ark]   arkoor_cosign_request (channel variant: channel_id + bridge
            nonce) → server checks admission, persists the input as spent,
            and returns the ARK #5 partial signatures plus the bridge's
 6. [local] verify every partial signature — the bridge's included — and
            complete; the full channel exit story is now held
 7. [Ark]   register the signed transactions (ARK #5)
                                                *** POINT OF NO RETURN ***
 8. [BOLT]  ChannelReady — the server MUST NOT send it before registration
            (see "Registration and the parent-exit response"); the channel
            is usable
```

At every step the user holds at least one of two complete exit stories:
(i) the input VTXO's own — the server's spent-marking does not touch the
signed exit chain or the input's delayed-exit leaf, and before the user
completes the signatures the server holds only partials (first-signer
one-shot, ARK #0), so it cannot actualize the transfer — or (ii) the
channel's: the genesis chain including the new level(s), the bridge, and the
initial commitment. (ii) is verified complete at step 6, before registration
surrenders (i) — registration is what arms the server's response, below. The
client MUST NOT complete or register unless **every** partial signature
verifies, the bridge's included: completing without a valid bridge would
produce a `channel-funding` VTXO with no unilateral exit at all, stranded
behind server cooperation until the expiry sweep. A post-cosign abort leaves
the input marked spent on the server and unilateral exit of the input as the
user's recovery — the standing posture of any failed ARK #5 exchange; an
abort at or before step 4 costs nothing.

**Admission.** Each party enforces from its own view — the client MUST refuse
to build, and the server MUST refuse to cosign, an upgrade that violates any
of these:

* **Depth.** The resulting channel VTXO's exit depth — the input's depth plus
  one, or plus two through a checkpoint — MUST be at most the
  `channel_max_vtxo_exit_depth` being pinned at open. This tightens ARK #5,
  which bounds only the *input* (at `max_vtxo_exit_depth − 1`) and so admits
  checkpointed outputs at depth `max_vtxo_exit_depth + 1` — an overshoot that
  would overrun the pinned CLTV budget ("The Ark channel type").
* **Runway.** The scope inherits the input's `expiry_height`, and the
  remaining runway (`expiry_height − real chain tip`) MUST exceed the party's
  computed complete CLTV floor
  (`2 * pinned_exit_delta + channel_max_vtxo_exit_depth + cltv_safety_margin`):
  below it the channel opens already inside its refresh-or-close discipline
  ("The refresh / force-close deadline"), and below the force-close margin,
  unable to beat the expiry sweep at all. The remedy is to refresh the input
  first — resetting expiry and depth — then upgrade. No far-future admission
  bound is needed: the inherited expiry was admitted when the input was
  issued, and its runway only shrinks.
* **Pinned parameters.** `pinned_exit_delta` is the input VTXO's decoded
  `exit_delta` — the new VTXO inherits it, so the scope invariant
  `decoded exit_delta == pinned_exit_delta` holds from birth — and the bridge
  `nSequence` and HTLC success-path CSV derive from it exactly as at a board
  open. A server unwilling to operate a channel at that value MUST refuse to
  cosign; the client's remedy is refresh first.
  `channel_max_vtxo_exit_depth` is pinned from the published
  `max_vtxo_exit_depth`, as at a board open.

**Registration and the parent-exit response.** The upgrade leaves the input
VTXO's own unilateral claim alive in the old chain: its
`delayed-sign(exit_delta, A)` leaf survives, so after the open the user could
broadcast the input's exit chain and, `exit_delta` blocks later, sweep the
input output directly — bypassing the transfer and taking back capacity that
now backs the channel, including any balance the server has since earned in
it. The server's defense is the transfer itself: the registered
checkpoint/arkoor transactions spend the input output by key path with
`nSequence = 0`, so once the input output confirms they are mineable the
next block — `exit_delta − 1` blocks ahead of the leaf's claim (the response
window of ARK #0; a degenerate `exit_delta` shrinks the lead toward a pure
fee race, which bounds `vtxo_exit_delta` itself, not this flow).
Concretely — each an observable property that MUST survive a crash:

* The server MUST NOT send `ChannelReady` before it durably holds the fully
  signed transactions of the new level(s); ARK #5 registration completing is
  the trigger. Until then the channel MUST NOT operate — which is exactly
  what keeps the pre-registration race harmless: the server's balance is
  still zero, so an aborting user can only reclaim its own funds.
* The server MUST retain those transactions, and MUST watch for any prefix
  of the input's exit chain confirming, for as long as the input's
  delayed-exit leaf is live: not merely for the scope's life, but until the
  input VTXO's output is conclusively spent on-chain (by the retained
  transfer or a forfeit-driven actualization), or a confirmed expiry sweep
  of one of its ancestors has made its creation impossible — a `pubkey`
  output carries no expiry leaf of its own (ARK #2); the sweep recourse
  lives on its cosign-taproot ancestors. A
  refresh or offboard does **not** end the duty — after the forfeit, the
  forfeited upgrade output can be actualized for the forfeit claim only
  through these same transactions, so discarding them at forfeit would let
  the input's leaf outrace the forfeit and take the value backing the
  replacement scope or payout. On seeing the input's chain confirm, the
  server MUST broadcast the retained transaction(s), fee-bumped via the P2A
  anchor, ahead of the input's `exit_delta` window. This duty is the
  forfeit-watch pattern (ARK #4, ARK #7) applied to an open — and it is
  load-bearing for the channel balance.
* The response merely actualizes part of the channel's exit chain: the
  channel VTXO output (or its checkpoint parent) lands on-chain, the bridge
  and commitment are unaffected, and the server's expiry-sweep recourse is
  preserved.

An upgrade adds no arkoor double-sign trust of its own — only the holder can
request spends of its own input, so the self-spend introduces no new
co-signing surface. The scope inherits exactly the trust the input already
carried: an arkoor-received input keeps its refresh recommendation (ARK #5),
which the next channel refresh satisfies.

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
anchors, HTLCs), HTLC resolution, channel quiescence, and the channel state
machine. The virtual-funding confirmation and chain-feed requirements are
normative and are specified in "Trust assumptions" above.

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
  completion are exactly as in ARK #4. Before requesting a leaf cosign and again
  before sending any forfeit, the client MUST select the VTXO that will back the
  refreshed channel and fully validate it as an exact channel-refresh receipt.
  From the decoded VTXO and its committed transactions — not RPC metadata — it
  MUST verify: policy is exactly `channel-funding(A)`; amount is exactly the
  accepted request or cap; server key equals the channel server key; `exit_delta`
  equals the channel's pinned value; decoded absolute expiry is acceptable for
  the client's exit margin; actual depth is at most
  `channel_max_vtxo_exit_depth`; every required ARK #2 transition, signature,
  and round anchor validates; and the VTXO point, scriptPubKey, and value are
  exactly the output spent by the new bridge. The client MUST verify the
  bridge's input against that exact decoded output before accepting its
  cosignature. Any mismatch is a safe pre-forfeit abort. Owner, amount, and
  hash-lock validation alone is not validation against a `channel-funding`
  request.
* **Fee.** The server's `refresh` fee (ARK #4) applies.

Refresh uses the ARK #4 round messages — round participation, the completion
steps (including leaf cosign), and `forfeit_vtxos` — in the **delegated**
participation mode (`submit_round_participation`): a `channel-funding` output
is not a valid request in an interactive `submit_payment`. The server
correlates the participation to a channel through its forfeited input — the
channel's current backing VTXO. The channel-specific additions are: on the leaf
cosign request, a `channel_id` (so the server maps the new leaf to the channel
being refreshed and looks up its funding keys) plus a bridge nonce — so the
server reconstructs and cosigns the new **bridge** in the same exchange (the new
bridge reuses the channel's funding keys; see "The teleport protocol" and
"Messages"); and an optional server liquidity bound on the round participation
response.

### Refresh admission and the server gate

For a delegated participation that requests `channel-funding`, the server MUST
derive the affected channel or channels from the current channel-backing VTXO(s)
in `input_vtxos`. A channel-funding request with no such current backing is
invalid. For each affected channel, the server MUST identify exactly one current
old backing; a stale backing, an unknown mapping, or two different current
backings for the same channel is an admission failure, not something deferred to
leaf cosigning or teleport.

Accepting the participation and establishing its refresh gates are one
admission decision: a participation is either accepted with a gate for every
affected channel, or not accepted at all — in every execution, including one
interrupted by a crash. Each gate is bound to the exact tuple
`(unlock_hash, channel_id, old_backing_vtxo_id)`, where `unlock_hash` is the
participation identifier returned by `submit_round_participation` and used by
the later status and forfeit requests. It is not sufficient to
remember only a channel id, because a later refresh changes that channel's
current backing; nor is it sufficient to look up the current backing only when
the leaf is cosigned. The acceptance MUST fail as a whole if any gate cannot be
established, conflicts with an outstanding gate, or cannot be tied to the exact
input; the server MUST NOT return an `unlock_hash` or any success response for
such a participation.

The gate is single-use and non-overwritable. Before acknowledging
`teleport_init`, the server MUST bind the gate to the exact new funding
outpoint reconstructed from the leaf/bridge; it MUST reject a different outpoint
or a second conflicting binding. Before releasing the round preimage, promoting,
or resuming, it MUST require the gate for that same participation, old backing,
and new funding scope. This binding deliberately fixes the **old** backing and
the eventual funding scope, while the ordinary round rules retain their
value-conservation and leaf-validation rules; it does not weaken those checks or
turn response metadata into an authorization.

A restart MUST NOT weaken a gate. After one, the server MUST NOT progress a
refresh — release a preimage, acknowledge a teleport, cosign the new leaf or
bridge, promote, or resume — for an affected channel until the gate and its
exact participation and channel bindings are available again; if it cannot
recover them, it MUST fail closed for the affected channel(s) rather than
proceed as though no gate existed.

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

The client derives the split locally — `fee` is the `refresh` fee it computes
from the published schedule (ARK #4), and `X = (V_old − V_new) − fee` is the
rest of the reduction — and declares **both halves** on `teleport_init` (see
"The teleport protocol"): `initiator_value_removal_sat = fee`, the amount by
which its **own** side of the new commitment is reduced, and
`responder_value_removal_sat = X`, the amount by which the **server's** side is
reduced. It bounds `X` by a local maximum, aborting if the server's cap implies
a larger withdrawal than it will accept. The server MUST verify the declared
split against the bridge it cosigned rather than trust it:

* the two declared removals MUST sum to exactly `V_old − V_new`, so the new
  commitment spends the new bridge's funding value `V_new` with the client's
  side shrinking by exactly `initiator_value_removal_sat` and the server's by
  exactly `responder_value_removal_sat` — the sum rule is what keeps either
  declaration from shifting any of the reduction onto the other side; and
* the declared `initiator_value_removal_sat` MUST be at least the `refresh` fee
  the server computes from its own schedule. The fee bound is a floor, not
  equality — like the round's `inputs − outputs ≥ fee` rule (ARK #4), so a
  client computing against a slightly stale schedule overpays rather than
  being rejected. Over-declaring is the client's own loss to take: with the
  sum fixed, a larger `initiator_value_removal_sat` only moves reduction from
  the server's side to the client's. The floor applies only when there is a
  reduction to allocate (`V_old − V_new > 0`): a re-initiated teleport of a
  scope the responder has already promoted sees no remaining reduction and
  declares `(0, 0)` — its fee was allocated by the original exchange, and the
  round's shortfall rule has already collected it.

A `teleport_init` violating either MUST be rejected (aborting the teleport, a
safe abort — the old channel is still intact). A refresh requires the fee to
fit on the client's side and `X` on the server's — each side's post-removal
balance MUST remain at or above its channel reserve — so a channel whose
client side cannot cover the fee cannot be refreshed and must be closed
instead (offboard or force-close).

### Added by the channel layer: the teleport

Refreshing the backing VTXO moves the channel onto a new funding outpoint. The
Lightning layer quiesces the channel (no *pending* HTLC updates) and *teleports*
the channel onto the new funding output — re-pointing the existing channel at
the new funding outpoint rather than closing and reopening it, carrying the
client's balance (and any committed HTLCs) forward, less the `refresh` fee the
client pays out of its own
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
            fresh channel-funding output  (→ unlock_hash; acceptance also
            establishes the channel's exact old-backing refresh gate)
            · if the server returns a cap (declining the requested amount),
              resubmit ≤ cap or abort (see "Server liquidity adjustment")
 2. [Ark]   poll RoundParticipationStatus until ISSUED (round tx broadcast)
 3. [gate]  MONEY GATE: the round tx is confirmed to the required depth → the
            new funding is now virtually confirmed
            ─────────── LAST SAFE ABORT (old channel still intact) ───────────
 4. [local] build the new bridge over the new VTXO output, reusing the channel's
            existing funding keys → new funding outpoint = new_bridge_txid:0
 5. [Ark]   RequestLeafVtxoCosign (channel id, + leaf nonce, + bridge nonce) →
            server cosigns the new leaf AND the new bridge (reusing the
            channel's funding keys); finalize + verify both musig(A,S)
            signatures
 6. [tele]  teleport: Stfu (quiesce; committed HTLCs carried) → TeleportInit
            (new_funding_txo = bridge:0, no nonces) / TeleportAck → exchange
            commitment_signed on the new funding outpoint (stock ECDSA); the
            server binds that exact outpoint to the refresh gate before
            acknowledging; wait until the peer has accepted it (see "The
            teleport protocol")
 7. [local] enter `ForfeitSent` (see "Teleport recovery"), then request
            forfeit nonces and send ForfeitVtxos for the OLD VTXO
                                                  *** POINT OF NO RETURN ***
            → unlock_preimage (the old VTXO is now forfeited to the server)
 8. [local] validate the preimage; finalize and validate the new VTXO exit
            path; enter `ForfeitCompleted` before sending or honoring
            TeleportComplete (the server releases the preimage only against
            the matching gate and funding binding)
 9. [tele]  TeleportComplete/CompleteAck → both sides promote and resume,
            gated on step 8; discard the old exit route only after promotion
            (see "Teleport recovery"). The channel stays open throughout (no
            ChannelReady is emitted).
```

## The teleport protocol

Refresh re-points an *open* channel from its old funding outpoint to the new
`channel-funding` output without closing it. The mechanism is **teleport**, an
Ark-specific message flow carried over the Lightning transport (not the Ark
service, and not a BOLT procedure). It is presented here as referenced
machinery the refresh flow drives; an interoperable implementation MUST
reproduce the message exchange and the promotion rule below.

**Prerequisites.** Before teleport begins, the channel MUST be quiesced (BOLT-2
`stfu`), which requires no *pending* (uncommitted) HTLC updates in flight. It does
**not** require the channel to be HTLC-free: any already-*committed* HTLCs are
carried across to the new funding scope by the teleport, exactly as a splice
carries them (the new-scope commitment reproduces them under the continued
commitment number; see "Commitment numbering continues across scopes"). The new
VTXO MUST be issued and virtually confirmed, and its **bridge** MUST be built and
cosigned (within the leaf cosign; see "Refresh"). The teleport itself does no `musig(A, S)` signing:
the new funding output is the bridge's stock P2WSH 2-of-2, so the new-scope
commitments are signed with the ordinary BOLT-3 funding keys (ECDSA), exactly as
in "Operation."

**Messages.** Teleport is a five-message flow between the initiator (the
client) and the responder (the server):

| Type | Message | Direction | Fields |
|---|---|---|---|
| 32850 | `teleport_init` | initiator → responder | `channel_id`; `new_funding_txo` (the new bridge's output 0); `responder_value_removal_sat` (the server-side liquidity withdrawal `X` of "Server liquidity adjustment"); `initiator_value_removal_sat` (the client-side `refresh`-fee reduction, same section) |
| 32851 | `teleport_ack` | responder → initiator | `channel_id` |
| 32852 | `teleport_abort` | either | `channel_id` |
| 32853 | `teleport_complete` | initiator → responder | `channel_id` |
| 32854 | `teleport_complete_ack` | responder → initiator | `channel_id` |

The **Type** column gives the reference's Lightning message-type number, and
the rows are ordered by it — note that `teleport_abort` sits between the setup
handshake and completion because it is the flow's early exit. The numbers live
in BOLT #1's custom-message range (≥ 32768, where protocol extensions belong)
rather than low unassigned space, which the splicing standardization is
actively (re)assigning; BOLT #1's odd/even rule still applies there, and the
parity is deliberate — an even `teleport_init`, `teleport_abort`, or
`teleport_complete_ack` reaching a peer that does not know the protocol fails
the connection outright rather than being silently dropped.
`responder_value_removal_sat` and `initiator_value_removal_sat` ride on
`teleport_init` as optional TLVs (types 4 and 6) defaulting to 0; the responder
MUST verify both against the bridge it cosigned per "Server liquidity
adjustment". Like the `ark_channel` feature pair, the numbers are subject to
change if the protocol is ever standardized.

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

**Promotion and resumption.** Completing a teleport involves two distinct
transitions that MUST NOT be conflated — and neither is the new-scope
`commitment_signed` exchange that precedes them:

* **Promote** — make the new funding scope the channel's *sole* live funding and
  **abandon the old scope**: cease monitoring and defending the old funding
  outpoint. (The commitment exchange only *adds* the new scope alongside the old;
  both stay live and monitored until promotion. A party that has promoted no
  longer watches the old funding outpoint, so it will neither react to nor
  penalize an old-scope commitment — including a revoked one.)
* **Resume** — exit quiescence (the BOLT-2 `stfu` the channel entered to begin the
  teleport) so HTLCs may again be offered, forwarded, and settled on the channel.

Promotion is asymmetric: the responder promotes on receiving `teleport_complete`,
the initiator on receiving `teleport_complete_ack`. The channel never leaves its
normal operating state and emits no `channel_ready`. A `teleport_abort`, or a
disconnect **before the forfeit** whose recorded state and round status prove
that the preimage was not released, leaves the channel on its old funding
outpoint. A
disconnect after the forfeit is a new-scope recovery case, not an old-scope
fallback. The flow survives reconnection through an Ark extension to
`channel_reestablish`.
Standard `channel_reestablish` does **not** carry two funding scopes. Ark adds
optional-TLV type 7, `teleport_funding_txid`, a 32-byte transaction id in BOLT
wire encoding. It MUST be present on every reestablish sent after a party has a
new-scope commitment and before promotion, and it MUST equal the new scope's
bridge transaction id (the funding output is fixed at vout 0); it MUST be absent
otherwise. The existing BOLT fields retain their ordinary meanings and do not
encode both scopes. The Ark channel type requires support for this TLV. Before a
forfeit, matching TLVs permit re-drive in dual-scope quiescence; absent or
mismatched TLVs permit old-scope abort only after recorded state and round
status prove the forfeit was not released. In `ForfeitSent` state a party MUST query
round status before deciding. In `ForfeitCompleted`, absence or mismatch MUST
NOT cause fallback or abort: the party must remain on, recover, or force-close
the new scope. No normal HTLC traffic may resume until this reconciliation
finishes.

`teleport_abort` is valid from either side at any point **before the forfeit** —
the point of no return — and a party MUST NOT send or honor one once it has
forfeited the old VTXO (the server after releasing the unlock preimage, the
client after receiving it), since from that moment the old scope is unclaimable
and only the new scope is safe. An implementation MAY narrow the window further
and accept an explicit `teleport_abort` only before the new-scope
`commitment_signed` exchange (the initiator while awaiting `teleport_ack`, the
responder while deciding whether to ack), relying on a disconnect/reconnect to
unwind the dual-scope quiesced state that follows rather than an explicit abort;
the reference does exactly this, and treats a `teleport_abort` arriving in any
later pre-forfeit state as a protocol violation.

**Ordering against the forfeit.** Teleport's commitment exchange — `teleport_init`
through the new-scope `commitment_signed` — completes *before* the old channel
VTXO is forfeited, leaving the channel **dual-scope and quiesced**: both funding
scopes carry a valid commitment, both outpoints are monitored, and no HTLC moves.
Both **promotion** and **resumption** happen only *after* the forfeit — for the
server, after it has released the unlock preimage; for the client, after it has
received it. The dual-scope quiesced state is the safe place to wait for the
forfeit, and a party MUST remain there until the forfeit resolves. The old-scope
commitment stays valid (non-revoked) throughout this wait — quiescence holds the
commitment number fixed (it continues across the teleport, never advancing; see
"Commitment numbering continues across scopes"), so the old scope is a sound
fallback on abort. Resumption is the first advance past that number, and thus the
moment the old commitment's revocation secret is revealed — a further reason
resumption, like promotion, MUST follow the forfeit. Teleport completion does not
consume the unlock preimage: the preimage finalizes the new VTXO's *exit* path
(ARK #4), not the channel state. The forfeit is the point of no return (see
"Refresh").

**Dual-scope safety.** Between the new-scope `commitment_signed` and promotion,
both funding scopes carry a valid commitment; each party MUST watch **both**
funding outpoints until promotion resolves — and not just the funding
outpoints: every output a scope's commitment makes claimable or contestable
joins the watch set as it appears, and later spends of those outputs (with
their confirmations and reorgs) MUST be observed too. A remote HTLC-success
second-stage spend, for example, reveals the preimage needed to settle an
upstream HTLC, so a party that stops watching at the funding outpoints can
strand an otherwise claimable payment.

The forfeit, not a teleport message, is the pivot. Once the old VTXO is forfeited
the new scope is authoritative, and the client MUST NOT broadcast the old-scope
commitment — an old-scope force-close would lose the channel VTXO output to the
server's forfeit (which spends it with no timelock, beating the old bridge's
`exit_delta`), so the client's balance is recoverable only on the new scope.
Symmetrically, the server MUST gate **both** promotion and resumption on the
forfeit having completed — on its own release of the unlock preimage — even though
both are triggered by the same `teleport_complete` message. Each is unsafe alone
if done early: promoting **abandons the old scope's defense**, letting a
counterparty that still holds the unforfeited old VTXO force-close the old scope
— even at a revoked state — unpunished; resuming lets that counterparty move its
balance out on the new scope while the old scope is still its own to claim. The
forfeit is what makes the old scope unclaimable (it spends the old VTXO output
ahead of the old bridge), so it MUST precede both.

**Teleport recovery.** A teleport progresses through the logical states
`OldOnly → DualScopePrepared → ForfeitSent → ForfeitCompleted → Promoted`. A
party's position in this progression — and everything acting from the current
state needs: both scopes' commitments and watch sets, the participation's
`unlock_hash`, the new bridge — MUST survive a crash at any point; entering a
state means being able to resume from it after a restart. `ForfeitSent` means
the forfeit may have been transmitted; `ForfeitCompleted` means the unlock
preimage has been validated and the exact new-scope exit path is recoverable.

States order external actions. The client MUST have entered `ForfeitSent`
before it first transmits a forfeit, and `ForfeitCompleted` — the preimage
validated, the exact new VTXO exit path finalized and validated — before it
sends or honors `teleport_complete`, clears refresh state, discards the old
exit route, or resumes normal operation. The server MUST hold the exact
participation/old-backing gate and expected new-funding binding before it
releases the unlock preimage, and MUST NOT promote or resume without that
completed forfeit. A party that cannot establish a state MUST NOT take the
external actions that state authorizes.

After a restart, a party in `ForfeitSent` MUST resolve round status before
aborting or falling back; it MUST NOT infer from a missing, unreadable, or
malformed local record that the preimage was not released. Once
`ForfeitCompleted` is known, it MUST never fall back to or broadcast the old
scope: it recovers or retransmits completion, or force-closes the new scope.
In particular, an automatic expiry or recovery handler MUST NOT force-close
the old scope after the forfeit merely because it cannot load or validate the
new path. Recovery state may be discarded only once promotion has made the new
scope the sole live scope. A party that cannot recover a channel's state,
watch set, gate, or scope record after a restart MUST leave that channel
fail-closed rather than recreate an empty state or resume without
reconciliation.

## Offboard

A cooperative channel close has two halves, in strict order. First the
channel is closed at the Lightning layer by the standard BOLT-2 close —
`shutdown`, HTLC drain, closing negotiation — which terminally fixes the
final balances. Then the close is settled at the Ark layer by an offboard
(ARK #7): the server pays the user's final balance to an on-chain
destination, and the user forfeits the channel VTXO to the server through the
connector swap. The commitment, the bridge, and the closing transaction the
close produced are all held off-chain throughout, and discarded once the
offboard confirms.

The forfeit is the standard ARK #7 connector-bound forfeit of the channel
VTXO — a `musig(A, S)` cosignature over a forfeit transaction that also spends
a connector output, valid only in a chain where the offboard transaction (and
thus the user's payout) exists. The offboard request, connector fanout, and
atomicity are all as in ARK #7, using the `prepare_offboard` /
`finish_offboard` messages unchanged. The **balance rule**, however, is
amended (below): ARK #7's exact-balance check cannot hold for a channel input.

### The close

The close is the ordinary BOLT-2 closing flow (referenced, not restated):
either peer MAY initiate with `shutdown` — the server included, e.g. to wind
down an idle channel, though the Ark leg that follows is always the user's to
drive. From `shutdown` on, no new HTLC is offered while in-flight HTLCs
resolve normally; once the channel is empty of HTLCs, closing negotiation
(`closing_signed`, or `closing_complete` / `closing_sig` under
`option_simple_close`; the reference uses `closing_signed`) produces a fully
signed **closing transaction** spending the funding output. Channel
quiescence (BOLT-2 `stfu`, the teleport's tool) plays no part: `stfu` cannot
be sent while updates are pending, forbids the very settles a drain needs,
does not survive disconnection, and force-disconnects after 60 seconds with
HTLCs still committed — it freezes an *operating* channel for a moment,
whereas `shutdown` drains a *terminating* channel monotonically, for as long
as that takes, surviving reconnection via `channel_reestablish`
retransmission. The freeze the offboard needs is not an extra protocol at
all; it is the close itself.

The closing transaction is negotiated exactly per BOLT-2, but its job here is
not to be broadcast: the funding output it spends is the bridge's output,
which exists only in the off-chain exit chain (virtual funding), so on the
cooperative path it can never confirm, and neither side need relay it (a
stack that broadcasts it unconditionally is harmless — the transaction spends
an outpoint the chain has never seen). Its jobs:

* **The balance agreement.** Completing the negotiation commits both peers,
  by signature, to the same pair of final balances — a peer that disagreed
  would produce a closing signature that fails to verify, aborting the close
  before the Ark flow rather than surfacing inside it. The amount rule below
  is checked against this close-fixed figure, not a live channel-state read,
  and the figure cannot go stale: the close is terminal, so no later update
  can move it.
* **The user's fallback between close and payout.** Until the forfeit, the
  user can settle the closed channel without the server: exit the VTXO
  (ARK #6), broadcast the bridge, and spend the funding output with the
  closing transaction in place of the commitment — the better tail, since its
  outputs carry no `to_self_delay` and the channel is empty of HTLCs by
  construction (the final commitment, never revoked, remains valid too). The
  closing transaction is an ordinary v2 transaction broadcast only after the
  bridge confirms, so TRUC's unconfirmed-ancestry rules never see it; a stale
  negotiated fee is repaired by CPFP of the user's own output.
* **Nothing, after the forfeit.** Once the VTXO is forfeited, the closing
  transaction — exactly like the commitment — can no longer reach the chain:
  the forfeit spends the channel VTXO output with no timelock, ahead of the
  bridge's `exit_delta`, so the funding output is never actualized (the
  argument that retires the old scope after a teleport; see "The teleport
  protocol"). Holding a signed closing transaction therefore gives the user
  no post-payout claim; enforcing that is the server's ordinary
  forfeit-watch duty (ARK #7, server sweeping).

Requirements:

* The user MUST complete the close — hold a fully signed closing transaction
  paying its final balance (less any closing fee it pays) — before
  `prepare_offboard`. This is the exit-before-forfeit discipline of ARK #4:
  the closing transaction is the exit story the user falls back on if the
  server stops cooperating after the close.
* The server MUST NOT cosign an offboard spending a channel VTXO whose
  channel has not completed the close — before that point there is no fixed
  balance to verify the payout against. It MUST retain the close outcome
  (the final balances, keyed to the channel's backing VTXO) until the VTXO
  is resolved — offboarded or expiry-swept — since the offboard may arrive
  arbitrarily later, after its Lightning stack has forgotten the channel.
* The close outcome MUST survive anything that happens after the close
  completes — a crash, a restart, or however the implementation routes its
  close notifications — until the backing VTXO is resolved. For the server the
  outcome is the close-fixed balances, bound to the exact channel and backing
  VTXO; for the user it additionally includes the fully signed closing
  transaction, which MUST be retained from the moment its final signed form
  exists. Whether the offboard is driven interactively, by a background task,
  or by recovery, all of them MUST see the same recorded outcome — beginning
  the Ark leg MUST NOT depend on having personally observed the close
  complete. A party that cannot record the outcome MUST NOT report the close
  complete or begin the Ark leg, and MUST keep the channel fail-closed rather
  than lose the outcome.
* A completed cooperative close is not a force-close: automatic recovery MUST
  NOT react to one by starting the unilateral exit or by discarding the close
  outcome. The user retains the unilateral path as a fallback and may take it
  on explicit policy (for example server failure or the expiry deadline), but
  a failed or delayed offboard remains retryable from the recorded close
  outcome first.
* A completed close disappears from the live Lightning channel set, but it does
  not disappear from expiry scheduling. Until `finish_offboard` may have made
  the connector-bound forfeit live, the client MUST hold the closed channel and
  backing scope to the same force-close deadline as an operating channel; if no
  offboard is potentially live by that deadline, it MUST begin the unilateral
  fallback in time to confirm the bridge before expiry. Sending
  `finish_offboard` is preceded by the transition from **Prepared** (finish
  definitely not sent, so the unilateral fallback remains safe) to
  **FinishOutcomeUnknown** (the request is eligible to be sent and may have
  reached the server); from that point, and across any restart, the client MUST
  be able to replay the exact finish request and recognize its payout (ARK #7).
  In `FinishOutcomeUnknown` it MUST replay only that byte-identical finish
  request and monitor the known payout txid; it MUST NOT prepare another
  offboard or race the potentially live forfeit with the old-scope fallback.
  The server's idempotent completed outcome (ARK #7) makes this recovery live
  across a lost response or restart. Only ARK #7's authenticated, definitive
  *not accepted* result — proof the finish never completed — permits a
  transition back to `Prepared`; a timeout or transport error does not. Once
  the signed payout is returned or observed, the client enters
  **PaidPendingFinality** and rebroadcasts/monitors it to the required depth.
* The closing fee (BOLT-2) is deducted from closing outputs only — under
  `closing_signed` from the funder's (the user's: it opened the channel),
  under `option_simple_close` from each closer's own. It never enters the
  offboard payout, which is computed from the final balance (below); the
  user SHOULD negotiate it low, since the closing transaction is broadcast
  only in the fallback, where it can be fee-bumped at use.
* The unilateral fallback MUST be able to confirm at current feerates: either
  the closing transaction is submitted with a fee-paying child spending the
  user's shutdown output, or the latest non-revoked commitment is used with
  its Lightning anchor/package path instead. A retained low-fee closing
  transaction with neither fee path is not a complete fallback, and
  re-selecting it unchanged makes no progress.

The close is a one-way door: `shutdown` commits the channel to closing
(BOLT-2), and there is no abort-and-resume — unlike a refresh, whose teleport
can abort back to normal operation. Before the finish intent becomes eligible
to send, an offboard failure leaves the channel closed but loses nothing —
balances stay fixed, `prepare_offboard` can be retried, and the unilateral
fallback remains available. What the close does not extend is expiry: if the client is still in
that pre-finish state and cooperation is not completing, it MUST begin fallback
in time to confirm the bridge before expiry ("The refresh / force-close
deadline"). After the finish intent becomes eligible to send, the VTXO is spent
and recovery instead replays and tracks the connector-bound payout as specified
above; it MUST NOT switch back to the old scope. A user SHOULD check its balance
clears the gate below before initiating the close: a closed channel below the
gate is recoverable only through the fallback if finish has not been attempted,
at on-chain cost.

### Amount rule

A channel VTXO's `amount` is the whole channel capacity `V` — *both* sides'
final balances — while the offboard pays out only the user's side. ARK #7's
requirement that the input amounts equal `net_amount` plus the fee *exactly*
therefore cannot apply to a channel input; in its place:

* An offboard that spends a channel VTXO MUST spend it as the request's only
  input (the amended rule is per-channel; mixing inputs would make the
  balance and fee attribution ambiguous).
* `net_amount + fee ≤ V`. The difference `V − net_amount − fee` is the
  server's own final channel balance: no output pays it — the server keeps
  it implicitly by collecting the forfeit of the whole VTXO.
* The server MUST verify `net_amount` against the user's final balance as
  fixed by the completed close — the figure its own closing signatures
  commit to, before any closing-fee deduction:
  `net_amount = user_balance − fee`. It MUST NOT cosign an offboard paying
  more (the excess would come out of the server's own side), and SHOULD
  reject one paying less, since the user has no way to recover the shortfall
  once the VTXO is forfeited. The close subsumes the settled-HTLC
  precondition the balance needs: closing negotiation only begins on a
  channel empty of HTLCs (BOLT-2), so a completed close leaves no unsettled
  HTLC to attribute.
* `fee` is the ARK #7 offboard fee (`offboard` schedule entry) with the
  user's final balance as the chargeable amount for the ppm-expiry
  component: the user pays fees on its own funds, not on the server's share.

The exact-balance rule of ARK #7 is unchanged for non-channel inputs.

### Sequence

```
 1. [BOLT]  shutdown (either peer) → no new HTLCs; in-flight HTLCs resolve →
            closing negotiation → fully signed closing tx in hand
            → record the close outcome (exact backing, final balances,
              closing tx)
            ──────── the channel is now closed; final balances fixed ────────
 2. [gate]  user_balance > 0 and (user_balance − fee) ≥ dust
 3. [Ark]   PrepareOffboard: destination + amount, channel VTXO id, ownership
            proof  → unsigned offboard tx + forfeit-cosign nonce
 4. [local] validate the offboard tx; sign the connector-bound forfeit of the
            channel VTXO (musig(A,S))
 5. [local] enter FinishOutcomeUnknown, retaining the exact FinishOffboard
            request for replay
    [Ark]   FinishOffboard: send (or byte-identically replay) the forfeit
            signature
                                                  *** POINT OF NO RETURN ***
            → idempotently recover the fully-signed offboard tx
 6. [local] broadcast the offboard tx → client paid on-chain; retain the
            genesis chain, bridge, closing tx, and exact finish record until it
            confirms to the required depth (recovery evidence, not permission
            to spend the old scope)
```

A failure at steps 3–4 leaves a closed channel and an unresolved VTXO: retry the
offboard, or fall back to the unilateral exit ("The close", above) before the
force-close deadline. Step 5 is the point of no return exactly as in ARK #7:
once the finish intent is eligible to be sent, the user MUST consider the VTXO
spent. A failure or lost response at step 5 is recovered by replaying the
exact request and tracking its txid; it is not permission to fall back. The
connector binding protects the payout, and the server's retained forfeits
protect it from a later double spend of the old scope. The only way back is the explicit
ARK #7 *not accepted* result, which proves no completed outcome or actionable
forfeit exists.

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
   `pinned_exit_delta` (`nSequence`; see "The channel VTXO and the bridge transaction"),
   so the bridge becomes valid only `pinned_exit_delta` blocks after the VTXO output
   confirms — the channel's unilateral-exit delay, the *same* a `pubkey` exit
   waits (ARK #6). It is a TRUC v3, P2A/CPFP transaction like the genesis chain.
3. **Force-close.** Broadcast the latest commitment transaction, which spends the
   now-on-chain funding output through the stock BOLT-3 2-of-2 (ECDSA). The
   commitment carries no extra delay of its own — the exit delay was already
   served on the bridge input — so it confirms once the bridge does. (After a
   completed cooperative close whose offboard never resolved, spend the
   funding output with the closing transaction instead — no `to_self_delay`,
   no HTLCs; see "Offboard".)
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
VTXO's exit depth, at most the scope's pinned
`channel_max_vtxo_exit_depth`, each level confirming), the bridge's
`pinned_exit_delta` relative timelock, and the bridge's own confirmation, plus
fee-bumping slack. The user MUST therefore begin a force-close at least
`channel_max_vtxo_exit_depth + pinned_exit_delta` blocks — plus confirmation and
fee-bumping slack for the genesis levels and the bridge — before that scope's
fixed `expiry_height`. The deadline is the **bridge's** confirmation: the
commitment confirms only after the bridge and, once the bridge has spent the VTXO
output, is no longer bounded by `expiry_height` (above), so it does not enter this
margin. The user SHOULD refresh, which resets `expiry_height`, well before that:
the same exit-before-expiry discipline as any VTXO (ARK #6), with the bridge's
`pinned_exit_delta` added to the margin.

This interacts with HTLCs. A refresh quiesces the channel — which needs no
*pending* (uncommitted) HTLC updates in flight — and **carries** any committed
HTLCs across to the new scope, like a splice (see "The teleport protocol"), so a
committed HTLC does not by itself block the refresh. What blocks it is an
**unresponsive peer**: both quiescence and the teleport's `commitment_signed`
exchange need the peer, so if it stops responding near the deadline the refresh
cannot complete and the node must force-close unilaterally instead. The user
MUST therefore stop offering new HTLCs on the channel, and refresh or
force-close, once that scope's fixed `expiry_height − real_chain_height`
falls to the **larger** of (a) the force-close margin above
(`channel_max_vtxo_exit_depth + pinned_exit_delta` + slack, which gets the bridge
confirmed) and (b) the full on-chain HTLC-resolution budget for any HTLC it must
still drive (the `cltv_expiry_delta` budget of "The Ark channel type",
`2 · pinned_exit_delta + channel_max_vtxo_exit_depth` + margin). This is a
maximum, not a sum: budget (b) already contains the genesis-plus-bridge climb of
(a) as its first `pinned_exit_delta + channel_max_vtxo_exit_depth`, then adds the
success-path CSV's second `pinned_exit_delta` on the confirmed commitment — so
(b) dominates whenever an in-flight HTLC must be resolved on-chain, while (a)
governs only an idle channel. Adding the two would double-count the shared
force-close prefix.

A channel carried too close to expiry — for example one an unresponsive peer
prevents refreshing — can lose the entire VTXO to the sweep. This deadline is
the user's own discipline, not a forwarding restriction on the server: the
server's expiry-sweep leaf *gains* the whole channel on a missed deadline (see
"Security and trust notes"), so it has no aligned incentive to stop forwarding
HTLCs onto a channel nearing expiry. The user cannot rely on it and MUST protect
itself by refreshing in time.

## Messages

Channels reuse the Ark messages of ARK #3 (board), ARK #5 (arkoor), ARK #4
(round), and ARK #7
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
| `amount` | u64 sats | the gross board amount (ARK #3); the channel VTXO output's value — and hence the funding output's — is this **net of the board fee** |
| `utxo` | outpoint | the on-chain board outpoint |
| `expiry_height` | u32 | chosen expiry height |
| `user_pubkey` | pubkey | `A` |
| `pub_nonce` | musig_pub_nonce | the user's MuSig2 public nonce for the leaf exit tx, bound to its sighash and the tweaked `musig(A, S)` (ARK #3) |
| `channel_id` | 32 bytes | the identifier under which both peers address the channel being opened on the Lightning transport (for a v1 open, the BOLT-2 *temporary channel id* — the final channel id is assigned only once the funding outpoint exists); its presence marks this as a channel board (absent for a plain `pubkey` board), and it MUST match the identifier under which the server stored the channel's funding keys |
| `bridge_pub_nonce` | musig_pub_nonce | the user's MuSig2 public nonce for the **bridge** key-path sighash (present only for a channel board) |

The first five fields are exactly ARK #3. `A` (`user_pubkey`) is the user's
cooperative key, *not* a funding key; the channel's BOLT-3 funding keys are
ordinary Lightning keys. They are **not** sent here — the server already holds
both (its own from `accept_channel`, the client's from `open_channel`) keyed by
`channel_id` — so the request need only identify the channel and supply the bridge
nonce.

From these the server reconstructs (a) the leaf exit output under the
`channel-funding` policy `musig(user_pubkey, S)`, and (b) the bridge — input = the
leaf output (keyspend `musig(A, S)`, `nSequence = pinned_exit_delta`), output 0 = P2WSH
2-of-2 of the channel's two funding keys (looked up by `channel_id`), output 1 =
P2A, 0-fee (see "The channel VTXO and the bridge transaction"). The response is
the ARK #3 `board_cosign_response` (server `pub_nonce` + `partial_sig` for the
leaf), extended with the server's bridge `pub_nonce` + `partial_sig`.

Requirements (server): the ARK #3 board requirements apply, and additionally —

* Before producing either channel-board partial signature, the server MUST
  enforce ARK #3's lower and upper expiry range. More precisely, it MUST reject
  an `expiry_height` below its current tip or above
  `current_tip + vtxo_expiry_delta + allowed_tip_skew`; this is mandatory for
  channel boards as well as ordinary boards and is not a client-selected policy.

* When `channel_id` is present, the server MUST build the exit output with the
  `channel-funding` policy `musig(user_pubkey, server_pubkey)` (ARK #2) and the
  bridge above — using the funding keys it holds for that channel — and cosign
  both. If `channel_id` names no channel it knows (with a funding key it
  controls), it MUST NOT cosign; absent `channel_id` it is a plain `pubkey` board
  (ARK #3). The named channel MUST be one the server is opening — funding keys
  exchanged, no funding outpoint yet assigned — so a board cosign cannot be
  redirected at an already-funded channel. The reconstructed bridge's output 0
  MUST equal the channel's negotiated funding output: value equal to the
  negotiated funding amount (`open_channel.funding_satoshis` — equivalently,
  the channel VTXO's value MUST equal it), script the P2WSH 2-of-2 of the
  stored funding keys. The cosignature check cannot catch a violation — both
  parties' bridges agree — while the Lightning stacks sign every commitment
  over the negotiated amount (BIP-143 commits the input value), so a mismatch
  yields commitments unusable against the real funding output: an unexitable
  channel and, once the bridge confirms, funds stranded in a 2-of-2 no expiry
  leaf reaches. Otherwise no pinning check is needed: a
  client that built a different leaf or bridge would simply see the cosignature
  fail to verify (as with the board fee, ARK #3).
* The server MUST NOT cosign a `channel-funding` output except together with
  its bridge in the same exchange (equivalently: never without `channel_id`).
  A channel-funding output carries no user spending clause at all (ARK #2), so
  a channel-funding VTXO without a cosigned bridge would strand the funds
  behind server cooperation until the expiry sweep.

Nonce freshness (both the board and leaf cosigns): the user commits its
`bridge_pub_nonce` (and the leaf `pub_nonce`) in the request and produces its own
partial only after the server's first-signer one-shot response. On any retry the
user MUST generate **fresh** secret nonces for both — the server returns a
different nonce each time, so reusing a user secret nonce against two different
aggregate nonces would leak the user's key (the nonce-reuse discipline of ARK #4
failure handling).

### `leaf_vtxo_cosign` (ARK #4)

Request (channel refresh): the ARK #4 fields — `vtxo_id`, user `pub_nonce` (bound
to the leaf sighash) — plus, for a channel leaf: `channel_id` (32 bytes, the
BOLT-2 channel id of the channel being refreshed) and `bridge_pub_nonce` (the
user's nonce for the new bridge's key-path sighash). Both are absent for a non-channel
leaf. Response: as ARK #4 (server `pub_nonce`, `partial_sig`, first-signer
one-shot), extended with the server's bridge `pub_nonce` + `partial_sig`.

As in the board cosign, the server reconstructs and cosigns both the new leaf exit
tx and the new **bridge** in this one exchange, using the funding keys it already
holds for `channel_id` — the new bridge reuses the channel's existing funding keys
(it does not rotate them; see "The teleport protocol"). If `channel_id` names no
channel it knows (with a funding key it controls), it MUST NOT cosign. For a leaf
whose policy is `channel-funding`, `channel_id` and `bridge_pub_nonce` MUST be
present, and the server MUST NOT cosign such a leaf without also cosigning its
bridge — the no-bridgeless-VTXO rule of the board cosign, above. The server
also independently learns the channel from the forfeited input on the round
participation and from `teleport_init`. It MUST ensure the refresh is a genuine,
value-conserving re-pointing of the named channel — the channel's current backing
VTXO is forfeited in the round (round value-conservation already guarantees the
new funding leaf is covered by the forfeited value) — and MUST refuse to cosign a
bridge that would move *another* channel's backing VTXO into this channel's
funding 2-of-2. It need NOT atomically pin the specific new leaf `vtxo_id` to the
specific forfeited outpoint: value conservation plus the channel↔forfeit
correlation already prevent a free or cross-channel funding, and a client
re-pointing its own channel onto a different leaf it owns (value-conserved, not
another channel's) is not an attack. This looser binding is deliberate — it leaves
room for future refresh extensions that combine channels (several forfeited VTXOs
re-pointed into one channel, or splices), which a strict 1:1 forfeit↔leaf pin
would foreclose. The participation gate of "Refresh admission and the server
gate" is different: it pins the exact **old** backing and participation for every
affected channel, not a particular new leaf at admission. For the selected
`vtxo_id`, however, the server MUST decode the
actual VTXO and reconstruct the bridge only from that VTXO's actual
`channel-funding` output, including its point, scriptPubKey, value, expiry, and
pinned `exit_delta`; it MUST NOT reconstruct from response metadata.

### `arkoor_cosign_request` (ARK #5)

The channel variant is the ARK #5 `arkoor_cosign_request` with the board
variant's two fields added, carried on the part whose destination set includes
the `channel-funding` output; the server cosigns the transfer and the bridge in
one exchange. A package MUST carry at most one part with these fields (exactly
one `channel-funding` destination; "Open by upgrade").

| Field | Type | Meaning |
|---|---|---|
| *ARK #5 part fields* | | `input`, `outputs`, `isolated_outputs`, `use_checkpoint`, `user_pub_nonces`, `attestation` — unchanged (ARK #5) |
| `channel_id` | 32 bytes | as in the board variant: the BOLT-2 temporary channel id under which the server stored the channel's funding keys; its presence marks the part as an upgrade and MUST match the identifier under which the funding keys were stored |
| `bridge_pub_nonce` | musig_pub_nonce | the user's MuSig2 public nonce for the **bridge** key-path sighash |

From these the server reconstructs the ARK #5 transactions exactly as the
generic endpoint does, then the bridge — input = the `channel-funding` output
of the reconstructed transfer (keyspend `musig(A, S)`,
`nSequence = pinned_exit_delta`, the input VTXO's decoded `exit_delta`),
output 0 = P2WSH 2-of-2 of the channel's funding keys (looked up by
`channel_id`), output 1 = P2A, 0-fee (see "The channel VTXO and the bridge
transaction"). The response is the ARK #5 `arkoor_cosign_response`, extended
with the server's bridge `pub_nonce` + `partial_sig`. The attestation is the
unchanged ARK #5 attestation: the destination's policy bytes already commit
the `channel-funding` `user_pubkey` and amount, and a tampered `channel_id`
yields a bridge over the wrong funding keys, which the client rejects at
partial-signature verification, before completing anything.

Requirements (server): the ARK #5 requirements apply, and additionally —

* When `channel_id` is present, the named channel MUST be one the server is
  opening: funding keys exchanged, not yet operating (no prior funding
  registered, `ChannelReady` never sent), and its Lightning establishment run
  against exactly the funding output this cosign produces. Two equalities
  bind that: the channel's assigned funding outpoint MUST equal the
  reconstructed transfer's `bridge_txid:0`, and the reconstructed bridge's
  output 0 MUST equal the channel's negotiated funding output — value equal
  to the negotiated funding amount (equivalently: the `channel-funding`
  destination's amount MUST equal `open_channel.funding_satoshis`), script
  the P2WSH 2-of-2 of the stored funding keys. The outpoint alone does not
  bind the amount: the txid commits the bridge's value, but nothing ties the
  *negotiated* amount to it, and the Lightning stacks sign every commitment
  over the negotiated amount (BIP-143 commits the input value) — a mismatch
  yields commitments unusable against the real output, an unexitable
  channel, and, once the bridge confirms, funds stranded in a 2-of-2 no
  expiry leaf reaches. (The upgrade exchanges the initial commitment
  *before* the cosign, so unlike the board variant an assigned funding
  outpoint is **required** here, not forbidden; these equalities are what
  prevent a cosign from being redirected at an already-funded channel.) If
  `channel_id` names no such channel, or either equality fails, the server
  MUST NOT cosign any part of the package.
* The server MUST NOT cosign a `channel-funding` destination except together
  with its bridge in the same exchange — the no-bridgeless-VTXO rule of the
  board cosign. Equivalently: a request with a `channel-funding` destination
  and no `channel_id` MUST be rejected (restated for the generic endpoint in
  ARK #5).
* The admission rules of "Open by upgrade" apply: the self-spend binding (the
  `channel-funding` destination's `user_pubkey` equals the input's user key),
  the resulting-depth bound, the runway floor, and the pinned `exit_delta`.
* For a single-part package whose destination set is exactly the one
  `channel-funding` output, the server SHOULD accept `use_checkpoint = false`
  ("Open by upgrade"); every other shape follows its ordinary checkpoint
  policy (ARK #5).

Retries follow ARK #5's operation-identity rule, with the channel fields
inside it: `channel_id` joins the part's operation identity, and in a
re-signed session the bridge slot gets fresh nonces and a fresh partial
exactly like the transfer slots. As everywhere (ARK #4 failure handling, the
board cosign), the user MUST discard and regenerate its secret nonces for
every slot — the bridge's included — on any retry; a secret nonce never
participates in more than one signing session.

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
  (`channel_id` + `bridge_pub_nonce`), the arkoor cosign request (the same
  pair), the leaf cosign request (`channel_id` +
  `bridge_pub_nonce`), and the round participation response are optional. A peer
  that does not implement channels ignores them, and the board, arkoor, refresh,
  and offboard flows are unchanged for non-channel VTXOs.
* **Channel support is a capability, not a default.** A channel open or refresh
  requires a channel-aware server. An older server ignores `channel_id` (and the
  bridge nonce) and would cosign an ordinary `pubkey` leaf; the client detects
  this — the cosignature does not validate against the expected `channel-funding`
  output, and no bridge cosignature is returned — and the open fails safely, since
  the board transaction is never broadcast without a valid exit and bridge. It
  does not silently produce a non-channel VTXO. An upgrade fails even earlier
  against a channel-unaware server: the `channel-funding` destination policy
  itself is rejected as an unknown policy type (ARK #2), before anything is
  marked spent. (Compatibility is defined against channel-unaware peers
  only; intermediate revisions of this document are not a supported
  deployment target — the stack is pre-release and deploys one spec revision
  at a time.)
  A server
  advertises channel support with the `supports_channels` flag in ark info
  (ARK #0); a client MUST NOT attempt a channel open against a server that does
  not set it.
* **New policy type.** A `channel-funding` VTXO (policy type `0x08`) is rejected
  by decoders that predate it (ARK #2 decoders reject unknown policy type
  bytes), so channel VTXOs are meaningful only between channel-aware parties.

## Security and trust notes

* **Exit guarantee preserved.** A channel VTXO carries a complete, cosigned
  genesis exit chain like any VTXO (ARK #2, ARK #6); together with the bridge and
  the commitment transaction (or the closing transaction, after a cooperative
  close) it always lets the user recover its channel balance
  on-chain without the server. The server cannot steal from a user who exits
  and force-closes in time.
* **Atomic give-up.** Because a channel VTXO is forfeited by the standard
  `musig(A, S)` hArk swap (ARK #4) or connector swap (ARK #7), refreshing or
  closing a channel hands the old VTXO to the server only in a history where
  the user has received its replacement or its payout. There is no window in
  which the user has surrendered the old channel VTXO without compensation.
* **The upgrade's parent-exit race.** An upgrade leaves the input VTXO's
  delayed-exit leaf alive in the old chain. The server's defense — broadcasting
  the registered transfer, which spends the input by key path with no delay —
  is why `ChannelReady` gates on ARK #5 registration, and why the retained
  chain and the watch duty must survive restarts *and outlive the scope* —
  the duty ends only when the input's output is spent or an ancestor's
  confirmed expiry sweep forecloses its creation, not at forfeit ("Open by
  upgrade", registration and the parent-exit
  response). It is the forfeit-watch pattern
  applied to an open, and load-bearing for the channel balance.
* **An upgrade adds no arkoor trust.** Only the holder can request spends of
  its own input, so a self-spend introduces no double-sign surface of its own;
  the scope inherits exactly the trust its input already carried, and an
  arkoor-received input keeps its refresh recommendation (ARK #5), which the
  next channel refresh satisfies.
* **Expiry is a deadline, and the sweep takes the whole channel.** The server
  may sweep the channel VTXO output after `expiry_height`. That output is a
  single `musig(A, S)` output holding the *entire* channel capacity, and its
  expiry leaf is the server's alone — so the sweep claims the whole value (both
  balances and any in-flight HTLC value), not merely the server's `to_remote`
  share. This is the deliberate flip side of "Exit guarantee preserved": there
  is no balance-aware split at the sweep, so a channel carried past expiry loses
  the user's *entire* balance, not just the server's recovery. The user's only
  protection is the deadline — it MUST force-close (actualizing its balance into
  the commitment's `to_local`, claimable on its own timelock) or refresh before
  expiry. Users MUST therefore refresh well before expiry — early enough to
  confirm the bridge first, and while any in-flight HTLC still has the on-chain
  resolution budget a forced force-close needs (see "The refresh / force-close
  deadline") — exactly as for any VTXO (ARK #0),
  with the bridge's `pinned_exit_delta` added to the margin.
* **Virtual funding.** The channel operates as though its funding output were
  confirmed while that output — the bridge's output 0 — lives only in the
  off-chain exit chain. The safety of this rests on the exit guarantee above: the
  funding output can always be actualized on-chain by broadcasting the genesis
  chain and then the bridge. The logical confirmation and real-chain feed are
  subject to the normative requirements in "Trust assumptions"; they are not an
  implementation-defined shortcut that may leave the channel state machine or
  monitor set behind the best chain.
