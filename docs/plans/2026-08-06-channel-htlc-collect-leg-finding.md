# Finding: the channel HTLC scripts protect one direction — the collect leg is open, and the type extension inverts it

**Date**: 2026-08-06. **Severity**: design defect in the Ark-channel type
extension (`08-channels.md`); no shipped code is affected (stage 1 carries
no channel HTLCs). **Status**: written finding; the spec amendment follows
review of this document.

## The finding

A server that never unrolls cannot claim a channel HTLC it has the
preimage for until the client actualizes the funding. The client can then
publish its own commitment, or publish only the bridge and force the server
to exercise its specified recourse by publishing the server commitment.
The client therefore controls when either contest can begin, even though it
does not necessarily control which commitment is ultimately published.

Every forward whose incoming leg is one of these client channels contains
one such **collect leg** (a client-offered HTLC the server must claim with
the preimage). In particular, every client-to-client forward contains one;
an incoming leg on an ordinary, already-funded Lightning channel does not
have this Ark-specific defect. Under stock BOLT-3 scripts the affected leg
degrades the server's claim from "deterministic if timely" to an
adversary-timed fee race. Under the type extension **as currently written**
— a path-keyed CSV on every success branch — the server's claim on the
collect leg is *delayed* behind the client's timeout, converting the race
into a deterministic loss and widening the griefing window from ~18 blocks
to `exit_delta` (144). The extension fixed the pay leg and silently assumed
the fix generalized; it does not.

## The security model, from the system's own primitives

**Standard Lightning states the receiver's obligation explicitly.**
BOLT #2 (`cltv_expiry_delta` selection): a node "MUST estimate a
fulfillment deadline" and, "if an HTLC it has fulfilled is in either
node's current commitment transaction, AND is past this fulfillment
deadline: … MUST fail the channel" — *fail the channel* meaning
force-close and claim on-chain. LDK 0.2.4 implements it as
`channelmonitor::CLTV_CLAIM_BUFFER = MAX_BLOCKS_FOR_CONF * 2 = 36`, doc
comment: *"If an HTLC expires within this many blocks, force-close the
channel to broadcast the HTLC-Success transaction"* — triggered in
`should_broadcast_holder_commitment_txn` for any inbound HTLC whose
preimage is known (`!htlc_outbound && htlc.cltv_expiry <= height +
CLTV_CLAIM_BUFFER && payment_preimages.contains_key(..)`). Receiver
safety in the standard model **is** the unilateral ability to reach the
chain with ~2×conf-margin to spare. The Ark server does not have it: its
commitment spends a funding output that exists only after a tree unroll
it never initiates (WD-16), and the offerer client controls when — if
ever — that unroll happens.

**bark's own HTLC VTXOs face the identical problem and already encode
the correct fix.** Both ark-native HTLC policies (`02-vtxo.md`) are
per-party asymmetric, with the rationale written into the leaf
definitions:

> `server-htlc-send` — leaf 1: hash-delay-sign `(payment_hash,
> exit_delta, S)` — server claims with the preimage; leaf 2:
> delay-timelock-sign `(2 * exit_delta, htlc_expiry, A)` — user reclaims
> after expiry; **"the doubled delay gives the server time to respond
> with the preimage path."**
>
> `server-htlc-recv` — leaf 1: delay-timelock-sign `(exit_delta,
> htlc_expiry, S)` — server reclaims after expiry; leaf 2:
> hash-delay-sign `(payment_hash, htlc_expiry_delta + exit_delta, A)` —
> **"the longer delay gives the server time to use its expiry path if
> the user exits too late."**

The rule these encode is party-keyed: **in the late-exit danger region, the
server's competing branch gets the response window and the user's branch
waits longer — whichever path is the server's.** This does not mean the
server always wins before expiry: on `server-htlc-recv`, an early-exiting
user still has its intended pre-expiry success opportunity. The watchman's
decide logic banks on the late-exit ordering ("deadline =
min(htlc_expiry, confirmed_at + 2*exit_delta)"). The channel type's HTLC
deviation is the only HTLC surface in the system keyed by *path*
("success") instead of *party*, and it happened because the BOLT-3
vocabulary displaced the house vocabulary.

## The four scenarios

Here A and B are Ark-channel clients. If B is instead an ordinary external
Lightning peer, the B-to-S collect leg is already on-chain funded and S can
force-close it in the standard way.

| # | Flow | Server's HTLC role | Outcome |
|---|------|--------------------|---------|
| 1 | A pays S (endpoint) | receiver, holds own preimage | Safe for S by policy: S controls whether to fulfill. A's on-chain refund is deliberately delayed by the fix. |
| 2 | S pays A (endpoint) | offerer | Safe for S: A claiming late = payment happened late; S is the origin, nothing cascades. A still relies on the client-receiver CLTV budget. |
| 3 | A → S → B | **receiver on the A-leg (collect)** | **The hole.** B's claim reveals `p`; if A stalls, S must claim on-chain; A times the unroll to the CLTV. |
| 4 | B → S → A | offerer on the A-leg (pay) — fixed by the CSV; **receiver on the B-leg (collect)** | Pay leg: S's timeout gets the intended response window ahead of A's late preimage claim ✓. Collect leg: same hole as (3) with B as griefer. |

Every client-to-client forward contains exactly one affected collect leg.
The tree expiry never rescues it: the CLTV budget and the force-close
deadline *by design* place every admitted HTLC's expiry comfortably before
`expiry_height` (that is what the floor is for), and once the bridge is
broadcast the expiry leaf is spent anyway.

## Quantified

Let the offerer client land the commitment at height `t`; the HTLC's
absolute CLTV is `T`; claim margin ≈ 18 blocks (`MAX_BLOCKS_FOR_CONF`).

- **Stock scripts (stage-1 core profile)**: under
  `zero_fee_commitments`, S's spend uses the baseline `nSequence = 0` —
  there is no anchors-era outer `1 OP_CSV`. Under this series' conservative
  relay convention it is mineable in the block after the commitment is
  observed; the client's timeout opens at `T`. If `t + margin < T`, S wins
  with room to confirm. The griefer must land `t` inside the final ~18
  blocks before `T`, and even then it is a fee race. The shared P2A bumps
  the commitment; an HTLC resolution pays its own fee or adds fee inputs —
  there is not a P2A "on both sides" of this contest. Exposure is bounded
  by per-HTLC caps × in-flight limit; the profile's "F removes the
  deterministic losses" holds.
- **Type extension as written**: S's claim is valid at `t + exit_delta`,
  while the client's timeout has only the baseline sequence plus absolute
  `T`. Landing `t` anywhere in roughly `(T − exit_delta, T]` — a
  144-block window at defaults, with the boundary itself a tie — gives the
  client an exclusive lead of up to `exit_delta − 1` under the same
  next-block convention. **Deterministic loss through the interior of a
  wider window: strictly worse than stock.** The CLTV budget
  (`2δ + depth + margin`) does not help: its receiver-protection prices
  the serial chain *a receiver who can unroll* climbs — vacuous for the
  server, which cannot start the chain.

## The fix: party-keyed timelocks (exact scripts)

The channel's roles are fixed — one Ark server and one client endpoint —
so party-keyed script templates are well-defined. The rule, mirroring the
HTLC VTXOs, is:

> **On every commitment HTLC output, if a non-revocation branch resolves to
> the client, that branch carries
> `<pinned_exit_delta> OP_CHECKSEQUENCEVERIFY OP_DROP`. If it resolves to
> the server, it carries no added CSV. The revocation branch stays
> immediate.**

This is an output-contract invariant, independent of whether the spend is
direct or presigned. A direct client resolution uses transaction version 2
or later and a height-based HTLC-input `nSequence` of at least
`pinned_exit_delta`. A client second-stage transaction is presigned as version
3 with `nSequence = pinned_exit_delta`; a server second-stage transaction uses
the `zero_fee_commitments` baseline `nSequence = 0`. The existing requirement
that `pinned_exit_delta` be nonzero remains.
`pinned_exit_delta` is a `u16` block count encoded with
`Sequence::from_height`, so the version-3 second-stage transaction's signed
sequence is itself a consensus BIP-68 relative timelock. BIP-143 also
covers the signed input's `nSequence` under
`SIGHASH_SINGLE|ANYONECANPAY`, so the peer's HTLC signature pins that
template. The CSV remains in the client branch even in the presigned case:
the contract is party-keyed rather than relying on presignedness as the
source of its semantics.

There is no inherited anchors-era outer `1 OP_CHECKSEQUENCEVERIFY` in the
base scripts. `zero_fee_commitments` is a distinct BOLT channel type: its
HTLC scripts use the non-`option_anchors` form, while its commitment and
HTLC transactions are version 3, its HTLC signatures use
`SIGHASH_SINGLE|ANYONECANPAY`, and its baseline HTLC-transaction input
sequence is 0.

**Why the direct branches are required.** Delaying only the presigned
second-stage transactions is insufficient. A client may confirm the bridge
near `T`, stop cooperating in the channel protocol, and continue watching
the chain. "Server recourse after the bridge confirms" then requires S
eventually to publish its own commitment. On that commitment S's collect
claim is a second-stage HTLC-Success, but A's competing timeout is direct.
Without CSV on that direct branch, publishing the server commitment at the
only time the funding exists recreates the fee race. Never publishing it
instead strands the server's balance and unresolved HTLCs indefinitely. The
same argument applies to the direct client-success branch of a
server-offered HTLC. These direct spends have no peer HTLC signature to pin
their transaction template; the output script is therefore what must enforce
the client delay.

Concretely, against the BOLT-3 `zero_fee_commitments` base scripts, the
relevant non-revocation bodies are below. The stock revocation prefix is
unchanged and omitted only for readability.

**Server-offered HTLC (the pay leg) — as the extension already has it:**

*On S's commitment (offered-HTLC script):* the client's direct preimage
branch gains the CSV; S's second-stage timeout branch is baseline.

```
    OP_ELSE
        <remote_htlcpubkey> OP_SWAP OP_SIZE 32 OP_EQUAL
        OP_NOTIF
            # To local node (SERVER) via HTLC-timeout transaction (timelocked).
            OP_DROP 2 OP_SWAP <local_htlcpubkey> 2 OP_CHECKMULTISIG
        OP_ELSE
            # To remote node (CLIENT) with preimage.
            OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
            <pinned_exit_delta> OP_CHECKSEQUENCEVERIFY OP_DROP      # ← Ark deviation
            OP_CHECKSIG
        OP_ENDIF
    OP_ENDIF
```

*On A's commitment (received-HTLC script):* A's second-stage success
branch gains the CSV (HTLC-Success presigned `nSequence =
pinned_exit_delta`); S's direct timeout branch stays at baseline after
its absolute CLTV.

```
    OP_ELSE
        <remote_htlcpubkey> OP_SWAP OP_SIZE 32 OP_EQUAL
        OP_IF
            # To local node (CLIENT) via HTLC-Success transaction.
            OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
            <pinned_exit_delta> OP_CHECKSEQUENCEVERIFY OP_DROP      # ← Ark deviation
            2 OP_SWAP <local_htlcpubkey> 2 OP_CHECKMULTISIG
        OP_ELSE
            # To remote node (SERVER) after timeout.
            OP_DROP <cltv_expiry> OP_CHECKLOCKTIMEVERIFY OP_DROP
            OP_CHECKSIG
        OP_ENDIF
    OP_ENDIF
```

**Client-offered HTLC (the collect leg) — the CHANGED direction:**

*On A's commitment (offered-HTLC script):* A's second-stage **timeout**
branch gains the CSV (HTLC-Timeout presigned `nSequence =
pinned_exit_delta`, on top of its `nLockTime = cltv_expiry` — valid at
`max(T, t + δ)`); S's direct preimage branch carries **no** extra CSV.

```
    OP_ELSE
        <remote_htlcpubkey> OP_SWAP OP_SIZE 32 OP_EQUAL
        OP_NOTIF
            # To local node (CLIENT) via HTLC-timeout transaction (timelocked).
            OP_DROP
            <pinned_exit_delta> OP_CHECKSEQUENCEVERIFY OP_DROP      # ← Ark deviation (NEW)
            2 OP_SWAP <local_htlcpubkey> 2 OP_CHECKMULTISIG
        OP_ELSE
            # To remote node (SERVER) with preimage.
            OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
            OP_CHECKSIG
        OP_ENDIF
    OP_ENDIF
```

*On S's commitment (received-HTLC script):* S's second-stage success
branch is baseline (HTLC-Success presigned with `nSequence = 0`);
A's direct timeout branch gains the CSV on top of its CLTV:

```
    OP_ELSE
        <remote_htlcpubkey> OP_SWAP OP_SIZE 32 OP_EQUAL
        OP_IF
            # To local node (SERVER) via HTLC-Success transaction.
            OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
            2 OP_SWAP <local_htlcpubkey> 2 OP_CHECKMULTISIG
        OP_ELSE
            # To remote node (CLIENT) after timeout.
            OP_DROP <cltv_expiry> OP_CHECKLOCKTIMEVERIFY OP_DROP
            <pinned_exit_delta> OP_CHECKSEQUENCEVERIFY OP_DROP      # ← Ark deviation (NEW)
            OP_CHECKSIG
        OP_ENDIF
    OP_ENDIF
```

Summary — who waits, per template:

| HTLC | Branch | Spender | Timelock |
|------|--------|---------|----------|
| server-offered | timeout (2nd stage / direct) | server | baseline `nSequence = 0` (and CLTV) |
| server-offered | success (2nd stage / direct) | client | `pinned_exit_delta` CSV |
| client-offered | success (2nd stage / direct) | server | baseline `nSequence = 0` |
| client-offered | timeout (2nd stage / direct) | client | `pinned_exit_delta` CSV ∧ CLTV |
| any | revocation | cheated party | immediate |

Consequences to fold into the amendment:

- **The extension's ordering claim generalizes correctly, but must be stated
  as a response window rather than an unconditional winner.** On a
  client-offered HTLC, S's success is baseline while A's timeout is valid no
  earlier than `max(T, confirmed_at + pinned_exit_delta)`. On a
  server-offered HTLC, an early commitment still gives A its intended
  pre-expiry success opportunity; a commitment that lands too late makes
  S's absolute timeout baseline while A's success waits the relative delta.
  Under this series' relay convention, a baseline response observed only
  after the contested output confirms is mineable at `H + 1`, while the
  delayed client branch is mineable at `H + pinned_exit_delta`: the
  exclusive response window is `pinned_exit_delta − 1`, not a full delta.
- **CLTV budget**: the client-side budget (`2δ + depth + margin`) is
  unchanged — a client receiver still climbs the serial chain. The
  server-side acceptance rule becomes watch-and-claim rather than pretending
  S can climb that chain itself. The forwarding delta floor (`Δ ≥ F_in`) is
  conservative room for the preimage to arrive, but it is not a substitute
  for sizing `pinned_exit_delta` to cover watch latency, broadcast,
  confirmation, and reorganization margin after either commitment appears.
- **Client cost**: a delayed refund on its own offered HTLCs. Clients do
  not forward in the intended hub-and-spoke profile, so an offered HTLC is
  the client's own payment's first hop — `exit_delta` of extra capital
  time, no cascade. This endpoint-only assumption MUST be made normative in
  the profile. If clients may forward, the delayed outgoing refund becomes
  an upstream risk and their forwarding delta must be re-derived.
- **On-chain resolution semantics change with the script.** The off-chain
  CLTV remains `T`, but no client branch of a confirmed output is on-chain
  eligible before `confirmed_at + pinned_exit_delta`; a client timeout must
  additionally wait until `T`, so its threshold is
  `max(T, confirmed_at + pinned_exit_delta)`. Monitors,
  payment-failure/retry reporting, persistence, and claim pruning MUST use
  that contract: an HTLC is not terminal merely because `T` passed, and a
  known preimage is retained until the output is conclusively spent. Tests
  must cover both commitment owners, late bridge confirmation, restart, and
  reorganization.
- **Construction and accounting become party-aware.** A generic
  `success-path CSV` channel-type switch is no longer an adequate model. Both
  peers MUST recover the fixed Ark client/server roles unambiguously and
  reproduce the same four beneficiary-keyed templates when constructing or
  recovering either commitment. Exact script test vectors must cover all four
  templates. The added script bytes also change offered/received HTLC witness
  lengths and success/timeout spend weights (though the P2WSH commitment
  outputs remain fixed-size), so fee estimates, batching limits, and TRUC
  child-size calculations MUST be updated rather than reusing the stock
  constants.
- **The response must confirm inside the exclusive window.** Once the client
  branch matures, the contest is a fee race again. The shared P2A belongs to
  the commitment, not to both HTLC resolutions; HTLC transactions add fee
  inputs, and a direct spend is not consensus-pinned to version 3 once its
  parent is confirmed. The server therefore needs a persistent watcher,
  confirmed fee inputs, and rebroadcast/replacement logic. Claim scheduling
  must also respect TRUC's one-child topology and child-size limit while the
  commitment is unconfirmed; it MUST NOT assume the shared P2A permits an
  arbitrary set of parallel HTLC children or that every set fits in one
  batch. "Deterministic" throughout this document means deterministic under
  that existing respond-and-confirm assumption, not protection against
  censorship past the response window.
- **Trimmed HTLCs remain policy exposure.** If an HTLC is trimmed from a
  commitment there is no output and no party-keyed script to protect. The
  extended profile MUST retain `max_dust_htlc_exposure_msat`-style bounds;
  a profile claiming no forwarded collect-leg loss MUST reject an incoming
  forwarded HTLC that is trimmed from either commitment. The party-keyed
  amendment removes the race only for materialized HTLC outputs.

## Alternatives considered

- **Bridge retention** (BR-12/13 MAY): lets the server fire the bridge
  once the VTXO output is on-chain — but only helps against a
  pre-positioned griefer. The collect-leg griefer never pre-positions;
  it times the whole unroll to land at `T`. Not a fix.
- **Server-initiated unroll / force-close scheduler**: forbidden, and
  deliberately so (WD-16; the r2 arc). Not on the table.
- **Operator policy** (per-HTLC caps, in-flight limits, halt on observed
  backing-exit): remains the stock-script interim bound and the correct
  stage-1 statement; it bounds losses, it does not remove them. Dust caps
  remain necessary even under the party-keyed type because a trimmed HTLC
  has no script output to order.

## Why this was caught late

The extension was designed from the pay-leg incident (a server-offered
HTLC claimed late), and the fix's name — "success-path CSV" — encoded
that framing; every subsequent reader, including the reviews, believed
the name. The budget section's receiver reasoning silently presupposes a
receiver who can unroll. And the stage-1 no-payments posture kept every
shipped review's scope away from forwarding, so no adversarial pass ever
walked the collect leg. The earlier design round *did* record "the
server cannot safely hold any HTLC" — but filed it as "stage 2's CSV
solves it" without re-deriving which branches the CSV must sit on.

## Disposition

1. Shipped code: unaffected (no channel HTLCs exist in stage 1).
2. `08-channels.md` type extension: MUST be amended to the party-keyed
   `zero_fee_commitments` scripts above before any stage-2 implementation
   work. The extended-profile claim, directional CLTV-budget framing,
   on-chain resolution semantics, role-aware construction and weight
   accounting, watcher/fee obligations, client-endpoint restriction (or
   replacement forwarding analysis), and residual dust exposure are
   corrected in the same pass. This document is the amendment's input.
3. Payments-era planning (the M4/payments G1): the stock-script collect
   leg is stated honestly as the landing-window fee race, bounded by
   caps and the unroll-watch policy trigger; the per-party scripts are
   the stage-2 dependency it builds toward. Tests cover all four
   commitment-owner/HTLC-direction combinations, direct and second-stage
   resolution, late bridge confirmation on both commitments, trim on either
   commitment, restart, and reorganization.
