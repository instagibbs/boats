# Finding: the channel HTLC scripts protect one direction — the collect leg is open, and the type extension inverts it

**Date**: 2026-08-06. **Severity**: design defect in the Ark-channel type
extension (`08-channels.md`); no shipped code is affected (stage 1 carries
no channel HTLCs). **Status**: written finding; the spec amendment follows
review of this document.

## The finding

A server that never unrolls cannot claim a channel HTLC it has the
preimage for until *someone else* puts the commitment on-chain — and the
party who does that is the HTLC's offerer, who profits from timing it at
the HTLC's expiry. Every forward the server routes contains one such
**collect leg** (a client-offered HTLC the server must claim with the
preimage), in both routing directions. Under stock BOLT-3 scripts this
degrades the server's claim from "deterministic if timely" to an
adversary-timed fee race. Under the type extension **as currently
written** — a path-keyed CSV on every success branch — the server's claim
on the collect leg is *delayed* behind the client's timeout, converting
the race into a deterministic loss and widening the griefing window from
~18 blocks to `exit_delta` (144). The extension fixed the pay leg and
silently assumed the fix generalized; it does not.

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

The rule these encode: **whenever the contested output appears on-chain,
the server's branch is spendable one `exit_delta` before the user's —
whichever branch is the server's.** The watchman's decide logic banks on
it ("deadline = min(htlc_expiry, confirmed_at + 2*exit_delta)"). The
channel type's HTLC deviation is the only HTLC surface in the system
keyed by *path* ("success") instead of *party*, and it happened because
the BOLT-3 vocabulary displaced the house vocabulary.

## The four scenarios

| # | Flow | Server's HTLC role | Outcome |
|---|------|--------------------|---------|
| 1 | A pays S (endpoint) | receiver, holds own preimage | Safe by policy: S controls preimage release; worst case the payment fails. |
| 2 | S pays A (endpoint) | offerer | Safe: A claiming late = payment happened late; S is the origin, nothing cascades. |
| 3 | A → S → B | **receiver on the A-leg (collect)** | **The hole.** B's claim reveals `p`; if A stalls, S must claim on-chain; A times the unroll to the CLTV. |
| 4 | B → S → A | offerer on the A-leg (pay) — fixed by the CSV; **receiver on the B-leg (collect)** | Pay leg: timeout deterministically leads a late preimage claim ✓. Collect leg: same hole as (3) with B as griefer. |

Every forward contains exactly one collect leg. The tree expiry never
rescues it: the CLTV budget and the force-close deadline *by design*
place every HTLC's expiry comfortably before `expiry_height` (that is
what the floor is for), and once the bridge is broadcast the expiry leaf
is spent anyway.

## Quantified

Let the offerer client land the commitment at height `t`; the HTLC's
absolute CLTV is `T`; claim margin ≈ 18 blocks (`MAX_BLOCKS_FOR_CONF`).

- **Stock scripts (stage-1 core profile)**: S's preimage spend is valid
  at `t` (anchors `1 OP_CSV`); the client's timeout at `T`. If
  `t + margin < T`, S wins deterministically. The griefer must land `t`
  inside the final ~18 blocks before `T`, and even then it is a fee race
  (v3 + P2A on both sides). Bounded by per-HTLC caps × in-flight limit;
  the profile's "F removes the deterministic losses" holds.
- **Type extension as written**: S's claim is valid at `t + exit_delta`;
  the client's timeout at `max(T, t)`. Landing `t` anywhere in
  `[T − exit_delta, T]` — a 144-block window at defaults — gives the
  client a lead of up to `exit_delta`. **Deterministic loss, wider
  window: strictly worse than stock.** The CLTV budget
  (`2δ + depth + margin`) does not help: its receiver-protection prices
  the serial chain *a receiver who can unroll* climbs — vacuous for the
  server, which cannot start the chain.

## The fix: party-keyed timelocks (exact scripts)

The channel's roles are fixed — the server is always the acceptor,
inbound-only — so party-keyed script templates are well-defined. The
rule, mirroring the HTLC VTXOs: **on every HTLC output, the
client-spendable branch carries `<pinned_exit_delta>
OP_CHECKSEQUENCEVERIFY OP_DROP`; the server-spendable branch carries
only the baseline anchors `1 OP_CHECKSEQUENCEVERIFY`; the revocation
branch stays immediate.** Client second-stage transactions are presigned
with `nSequence = pinned_exit_delta`, server second-stage transactions
at the baseline; these are v3 transactions, so any `nSequence <
0x80000000` is a consensus BIP-68 timelock, and BIP-143 covers the
signed input's `nSequence` even under `SIGHASH_SINGLE|ANYONECANPAY`, so
each peer's HTLC signatures pin the other's templates — the extension's
existing enforcement argument, unchanged.

Concretely, against the BOLT-3 `option_anchors` templates (which
`zero_fee_commitments` inherits):

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
        1 OP_CHECKSEQUENCEVERIFY OP_DROP
    OP_ENDIF
```

*On A's commitment (received-HTLC script):* A's second-stage success
branch gains the CSV (HTLC-Success presigned `nSequence =
pinned_exit_delta`); S's direct timeout branch stays
`<cltv_expiry> OP_CLTV` + baseline.

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
        1 OP_CHECKSEQUENCEVERIFY OP_DROP
    OP_ENDIF
```

*On S's commitment (received-HTLC script):* S's second-stage success
branch is baseline (HTLC-Success presigned at baseline `nSequence`);
A's direct timeout branch gains the CSV on top of its CLTV:

```
        OP_ELSE
            # To remote node (CLIENT) after timeout.
            OP_DROP <cltv_expiry> OP_CHECKLOCKTIMEVERIFY OP_DROP
            <pinned_exit_delta> OP_CHECKSEQUENCEVERIFY OP_DROP      # ← Ark deviation (NEW)
            OP_CHECKSIG
        OP_ENDIF
```

Summary — who waits, per template:

| HTLC | Branch | Spender | Timelock |
|------|--------|---------|----------|
| server-offered | timeout (2nd stage / direct) | server | baseline |
| server-offered | success (2nd stage / direct) | client | `+ exit_delta` CSV |
| client-offered | success (2nd stage / direct) | server | baseline |
| client-offered | timeout (2nd stage / direct) | client | `+ exit_delta` CSV (∧ CLTV) |
| any | revocation | cheated party | immediate |

Consequences to fold into the amendment:

- **The extension's ordering claim generalizes correctly**: "a
  *server* claim strictly leads a *client* claim by `exit_delta` on
  whichever commitment confirms, in both HTLC directions" — replacing
  the current path-keyed sentence.
- **CLTV budget**: the client-side budget (`2δ + depth + margin`) is
  unchanged — a client receiver still climbs the serial chain. The
  server-side acceptance rule becomes watch-and-claim: its claim needs
  only `conf-margin` after any commitment confirms, and the forwarding
  delta floor (`Δ ≥ F_in`) keeps guaranteeing the preimage arrives with
  room to use it.
- **Client cost**: a delayed refund on its own offered HTLCs. Clients do
  not forward in this topology (hub-and-spoke), so an offered HTLC is
  always the client's own payment's first hop — `exit_delta` of extra
  capital time, no cascade.
- **Replacement-cycling symmetry**: the extension's argument that the
  head start (not a tie) is what closes the cycling window must be
  restated with roles swapped for the collect leg; v3/TRUC + P2A carry
  the same weight, but the paragraph must exist.

## Alternatives considered

- **Bridge retention** (BR-12/13 MAY): lets the server fire the bridge
  once the VTXO output is on-chain — but only helps against a
  pre-positioned griefer. The collect-leg griefer never pre-positions;
  it times the whole unroll to land at `T`. Not a fix.
- **Server-initiated unroll / force-close scheduler**: forbidden, and
  deliberately so (WD-16; the r2 arc). Not on the table.
- **Operator policy** (per-HTLC caps, in-flight limits, halt on observed
  backing-exit): remains the stock-script interim bound and the correct
  stage-1 statement; it bounds losses, it does not remove them.

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
   scripts above before any stage-2 implementation work; the
   extended-profile claim and the CLTV-budget framing are corrected in
   the same pass. This document is the amendment's input.
3. Payments-era planning (the M4/payments G1): the stock-script collect
   leg is stated honestly as the landing-window fee race, bounded by
   caps and the unroll-watch policy trigger; the per-party scripts are
   the stage-2 dependency it builds toward.
