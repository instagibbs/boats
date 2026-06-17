# ARK #8 redesign: VTXO channels via a bridge transaction

**Date:** 2026-06-17
**Status:** Design draft — for review/argument before editing ARK #8 / ARK #2.
**Scope:** Replace the "channel VTXO *is* the funding outpoint" model in ARK #8
with a presigned **bridge transaction** between the channel VTXO and the
Lightning commitment. Goal: keep LDK almost stock (no taproot funding, no custom
commitment shape) and shrink the spec's Ark-channel deviation surface.

This document is the design we agree on *before* touching the spec. It is not
itself the spec.

---

## 1. Background and motivation

### 1.1 The current (taproot-direct-funding) design

ARK #8 today makes the **channel VTXO's own output** the channel's BOLT-3 funding
outpoint. One taproot output is simultaneously:

* an Ark cosign-taproot VTXO output (internal key `musig(A, S)`), and
* the Lightning funding output the commitment transaction spends.

Forcing those to be the same output is what drives every Ark-specific LDK change:

* the funding output must be **taproot** with a `musig(A, S)` key-path spend →
  LDK needs taproot funding + MuSig2 key-path signing, and the BOLT-3 funding
  keys must be **pinned** to `A`/`S` (`InMemorySigner::with_overridden_funding_key`);
* the unilateral-exit delay (`exit_delta`) rides on the **commitment input's**
  `nSequence`, which displaces the obscured commitment number into a dedicated
  **`OP_RETURN`** output and requires a custom `ChannelMonitor` recognizer;
* because the funding spend is MuSig2, **every commitment update** needs a
  partial-sig + public-nonce exchange — and the refresh teleport inherits a
  **two-stage MuSig2 nonce dance**.

This is the `bark-bridge-removal-2026-05-09` / `ark-ldk-integration-2026-06-09`
line in the LDK fork.

### 1.2 The bridge model is the incumbent `bark-lightning` implementation

The `bark-lightning` tree (on its `master`) implements channels with a **bridge
transaction**, not the taproot-direct model — see `bark/docs/channel-lifecycle.md`
and `bark-lightning/src/bridge_tx.rs`. This is a research implementation, not a
deployed system, and the LDK-side Ark changes are **not** in `rust-lightning`
master (they live on `ark-ldk-integration-…`). Structure:

```
Board TX (on-chain)
  └─► Leaf TX (off-chain, pre-signed)        ← actualizes the channel VTXO output
        └─► Bridge TX (off-chain, pre-signed)
              └─► Commitment TX (LDK, virtual funding)
                    ├─► to_local / to_remote / HTLCs
                    └─► P2A anchor
```

The bridge's output 0 is a **standard ECDSA 2-of-2 multisig** (segwit v0 P2WSH),
"separate from the MuSig2 hierarchy." Both repos also carry a
`bark-bridge-removal-…` branch: the bridge is the **incumbent** implementation,
and bridge-removal → taproot-direct is the in-progress migration that the current
ARK #8 describes. This redesign keeps the incumbent bridge and abandons that
migration, with one improvement (see §1.3).

### 1.3 One improvement over the old bark bridge

The old bark bridge spends the channel VTXO via a **taproot script path** — a
`<timelock> OP_CSV <channel_agg_key> CHECKSIG` leaf (`VtxoScriptPath` +
`control_block()` in `bark-lightning/src/bridge_tx.rs`). This redesign instead
spends the channel VTXO via its **keyspend** `musig(A, S)` and enforces the exit
delay purely via the **bridge input's `nSequence`** — so the VTXO needs no
exit-path script leaf at all (its only tapleaf is the ASP expiry sweep, §4). That
removes the old exit-timelock leaf, the control block, and the script-path sighash
entirely (agreed: design decision #1).

---

## 2. Core idea

**Insert one presigned transaction between the channel VTXO and the commitment,
relocating the Ark/Lightning boundary to a clean seam.**

* Everything from the channel VTXO output *up through the bridge* is pure Ark:
  taproot, `musig(A, S)`, TRUC (v3) + P2A, presigned, 0-fee.
* The bridge's **output 0** is a stock BOLT-3 segwit-v0 P2WSH 2-of-2 funding
  output. Everything below it is **stock LDK**: standard commitment, standard
  funding keys, standard ECDSA 2-of-2 signing.
* The bridge is the **adapter** between the two worlds, and it is just the last
  transaction of the VTXO's exit chain.

MuSig2(`A`, `S`) is thereby confined to **one-shot** Ark cosigns — board exit
tx, **the bridge (once per funding scope)**, forfeit, refresh leaf, offboard —
and never appears per-commitment. That is what lets the commitment be stock and
the teleport drop its nonce exchange.

---

## 3. What changes

| | Item | Why |
|---|---|---|
| **Removed (LDK + spec)** | Taproot funding output, MuSig2 key-path funding spend, funding-key pinning to `A`/`S`, `with_overridden_funding_key` | funding is a stock P2WSH 2-of-2 over ordinary LDK funding keys |
| **Removed** | `exit_delta` CSV on the commitment input | moves to the bridge input's `nSequence` |
| **Removed** | `OP_RETURN` obscured-commitment-number output + custom `ChannelMonitor` recognizer | commitment `nSequence`/`nLockTime` are free again → stock BOLT-3 |
| **Removed** | Teleport two-stage MuSig2 nonce exchange (`next_local_nonces`) | funding spend is ECDSA 2-of-2, not MuSig2 |
| **Simplified** | `channel-funding` policy `0x08` | stays; `A` becomes just the cooperative key (not a "pinned funding key"); exit-timelock citation moves commitment-input → bridge-input; keeps the ASP expiry-sweep leaf, now **pure-Ark** (never touches the LDK funding output) |
| **Added** | The **bridge transaction** (one spec section) | the single new on-chain object |
| **Added** | The bridge **keyspend cosign**, *folded into* the existing board/leaf cosign (§7) | no new message: the existing cosign round-trip gains a `channel_id` + a bridge nonce and returns a bridge partial |
| **Added** | One extra tx in the exit/force-close path | broadcast bridge after the genesis chain, before the commitment |
| **Kept (in scope)** | HTLC success-path CSV; the (now non-taproot) teleport state machine | unchanged by the bridge |

The "Ark channel type" deviation list shrinks from **six** bullets to roughly
**three**: HTLC success-path CSV, the HTLC CLTV budget, and virtual funding —
plus "funding is ordinary BOLT-3" (a non-deviation).

---

## 4. The channel VTXO (ARK #2 `channel-funding`, `0x08`)

Unchanged in structure; only the prose moves:

* internal key `musig(A, S)` — cooperative 2-of-2 keyspend. This is the path the
  **bridge transaction** spends (was: the commitment), and the path used to
  forfeit, refresh, or offboard the VTXO.
* **No user unilateral-exit leaf.** Essential: the user must not be able to claim
  the raw channel balance directly; the only actualization path is the bridge →
  commitment, which splits funds per the channel balance.
* expiry-sweep leaf `timelock-sign(expiry_height, S)` — **in scope**, on the VTXO
  output: the ASP's recourse when the whole tree is meant to time out. This makes
  the channel VTXO the full cosign-taproot `(musig(A,S), S, expiry_height)`. The
  **commit funding output** (bridge `out0`) carries **no** sweep script — it is
  swept by LDK's own transactions and LDK has no custom-input support for an Ark
  sweep there, so the tree-timeout recourse lives one hop up, on the VTXO. Unlike
  the taproot design's gap-C, this leaf is **pure-Ark**: it never touches the LDK
  funding `scriptPubkey`, so it needs no cross-repo coordination.
* `A` is just the user's cooperative key, exactly as in every other policy
  (drop the "holder BOLT-3 funding key" role — the funding keys are no longer
  `A`/`S`).

The only normative change to `0x08`: the exit timelock is enforced by **the
bridge transaction's input `nSequence`**, not the commitment input.

---

## 5. The bridge transaction

A single presigned transaction, structurally a TRUC exit-chain transaction:

| Field | Value |
|---|---|
| `nVersion` | 3 (TRUC) |
| `nLockTime` | 0 |
| input (single) | the channel VTXO output; spent via key path `musig(A, S)`; **`nSequence = exit_delta`** (height-based, BIP-68), where `exit_delta = vtxo_exit_delta` from ark info |
| output 0 (funding) | segwit-v0 P2WSH 2-of-2 of the channel's BOLT-3 funding pubkeys; value = channel VTXO value (0-fee) |
| output 1 (anchor) | P2A `OP_1 <0x4e73>`, 0 value |
| fee | 0 (bumped by CPFP on output 1 at broadcast, per ARK #6) |

Properties:

* **Exit delay.** Because the bridge is presigned with `nSequence = exit_delta`
  and is the *only* unilateral on-chain spend via the VTXO's keyspend (forfeit
  / refresh / offboard are cooperative and off-chain), the exit delay needs no
  script on the VTXO output — it rides on the bridge's `nSequence` alone. (The
  VTXO's one tapleaf is the ASP expiry sweep of §4, unrelated to the exit delay.)
  The bridge can confirm only `exit_delta` blocks after the channel VTXO output
  confirms on-chain. The commitment then
  confirms immediately (stock BOLT-3 input). Net unilateral-exit timeline equals
  today's: exit genesis chain → VTXO output on-chain → wait `exit_delta` →
  funds claimable. The delay now lives on a *fixed* tx instead of *every*
  commitment.
* **Last exit-chain tx.** Exiting a channel VTXO = broadcast the genesis chain
  root-first (ARK #6) until the channel VTXO output confirms, then the bridge,
  then the commitment. The bridge is TRUC + P2A + 0-fee + presigned — identical
  in kind to every other exit transaction; it reuses ARK #6 CPFP machinery and
  counts as one extra exit-chain level.
* **Funding keys.** Output 0's two pubkeys are ordinary BOLT-3 funding keys that
  LDK's stock `InMemorySigner` mints and exchanges in `open_channel` /
  `accept_channel`. They are *not* `A`/`S`. No key import; no ark-info key
  distribution.
* **Neither party can cheat the delay.** The keyspend is `musig(A, S)`; neither
  side can produce a faster spend unilaterally. Cooperative re-signing would just
  be a cooperative close/refresh.

---

## 6. Funding-key handling (resolves design decision #2)

Stock all the way down:

* `ArkChannelSigner` wraps a stock `InMemorySigner` and delegates `pubkeys()` to
  it (`bark-lightning/src/signer.rs`). The funding pubkey is whatever LDK chose.
* The wallet reads both funding pubkeys (its own from the signer, the
  counterparty's from `accept_channel`), builds the bridge, and hands
  `bridge_txid:0` to LDK as the funding outpoint via the stock
  `unsafe_manual_funding_transaction_generated()` (virtual funding). LDK then
  validates all commitments against its internal state.
* `with_overridden_funding_key` is a taproot-design artifact and is **unused**
  here.

The ordering "build the bridge after `accept_channel`" is **not** an Ark-imposed
round-trip — it is the stock LDK funding hook (`FundingGenerationReady` hands you
the funding `output_script` derived from both pubkeys; the wallet supplies the
funding tx in response). The pre-funding state is the standard mid-open channel
state; the inbound-open DoS surface is the standard one LDK bounds. Fixed keys
would not let us skip the handshake (channel open gates the bridge regardless),
so they would add a key-distribution scheme for no real gain.

From the spec's view: **the funding keys are the channel's ordinary BOLT-3
funding pubkeys, exchanged in `open_channel`/`accept_channel`.** Nothing about
them goes in ark info or `0x08`.

**Refresh reuses the funding keys — a deliberate divergence from splicing.** The
teleport does **not** rotate the funding keys for the new scope. In the fork,
`FundingScope::for_teleport` (channel.rs) clones the prior scope's
`channel_transaction_parameters` (funding pubkeys included), changes only the
`funding_outpoint`, and sets `splice_parent_funding_txid = None`. LDK's splice
path *does* rotate — `send_splice_init` → `new_funding_pubkey(prev_funding_txid)`,
a **secret-derived** tweak (`compute_funding_key_tweak` hashes the funding *secret*
key, so the new pubkey is not publicly derivable) chosen for on-chain
unlinkability. The teleport deliberately skips that rotation. This is sound here
because the bridge and commitment normally never hit the chain, so there is no
real privacy cost — and it is what keeps refresh simple: both sides already hold
the funding keys, so there is **no per-scope key exchange**, the new bridge is
buildable immediately (no circularity), and the folded leaf+bridge cosign needs
**no reorder**. Worth flagging explicitly, since a reader who knows splicing will
expect key rotation here and there is none.

---

## 7. The bridge cosignature (folded into the existing cosign)

**No new message.** The bridge is cosigned *inside* the existing cosign exchange
— the board cosign (ARK #3) at open, the leaf cosign (ARK #4) at refresh — by
adding the bridge to what that one round-trip already cosigns.

### 7.1 What is signed

A BIP-327 MuSig2 **key-path** (BIP-341 taproot-tweaked) cosignature of
`musig(A, S)` over the bridge, spending the channel VTXO output. Same primitive
as the leaf/board exit-tx cosign — a one-shot 2-party MuSig2 session yielding one
BIP-340 Schnorr signature, baked into the presigned bridge. The key-path sighash
is over the VTXO's **full** taproot output (internal key **plus** the expiry-sweep
leaf of §4), which the server reconstructs.

(Non-normative: in `bark`, `S` is a nested `MuSig2(channel_agg_key, asp)`; the
keyspend bridge therefore uses the nested ceremony — unlike the *old* bark bridge,
which signed a `channel_agg_key`-only script path. The spec models one server `S`
and one server partial.)

### 7.2 Concurrent with the leaf — one round-trip

The leaf and the bridge are cosigned **together**, not in sequence. A segwit txid
does not depend on the witness, so the client builds the unsigned leaf, computes
`leaf_txid`, builds the bridge over `leaf_txid:0`, and requests cosigns on **both**
sighashes in the same exchange. The two only have to be finalized **before the
forfeit is signed** (refresh) / before the board is broadcast (open). The one
ordering constraint is that the bridge needs the channel's funding keys — at open
from `open_channel`/`accept_channel`, at refresh the *same* keys reused (the
teleport does not rotate them; §6), so they are already in hand and nothing extra
precedes the cosign.

### 7.3 Both sides reconstruct from the message

Like the rest of Ark: the **client constructs** the leaf and bridge and sends the
cosign request; the **server reconstructs both from that message plus its stored
channel state**, validates, and cosigns. No new RPC. The request extends the
board/leaf cosign with two fields:

| Added field | Type | Meaning |
|---|---|---|
| `channel_id` | 32 bytes | identifies the channel; the server looks up both funding keys (and the VTXO) from its stored state |
| `bridge_pub_nonce` | musig pub nonce | the user's nonce for the bridge key-path sighash |

From `channel_id` the server looks up the channel's two funding keys — its own
`lsp_funding` and the client's, both stored from `open_channel`/`accept_channel` —
and reconstructs `out0 = P2WSH 2-of-2` of them, plus the rest of the bridge
(`out1` P2A, `v3`, `nLockTime 0`, `nSequence = exit_delta`, funding amount = VTXO
value). It computes both key-path sighashes, validates the client's partials, and
returns its nonces + partials for **leaf and bridge**. The client finalizes both
and MUST validate before relying on them. (The funding keys are *not* re-sent in
the request — the server holds the authoritative copies, so re-sending would be
redundant and a footgun.)

### 7.4 Safety: the server's own funding key, and the decline rule

Cosigning the bridge authorizes spending the VTXO into `out0`. At **refresh**, if
`out0` were not a 2-of-2 the server co-controls, the client could forfeit the old
VTXO *and* sweep the new VTXO whole via a bogus bridge — stealing the server's
channel balance. So the server MUST build `out0` with **its own** `lsp_funding`.

The binding is `channel_id`, the LSP-allocated handle the server keys its
per-channel funding key by. (A holder funding pubkey would *not* do — a client may
reuse one across channels, so it cannot uniquely identify the channel.) **If
`channel_id` names no channel the server knows (with a funding key it controls),
it simply does not cosign.** Correct-by-construction `out0` — built from the
server's own stored funding key — plus this decline rule is the whole safety
argument. The fully signed bridge is held off-chain by both
parties; the server can broadcast bridge + commitment to force-close, exactly as
it can broadcast the commitment today.

---

## 8. Lifecycle

### 8.1 Open

```
 1. [LN]    request_channel → channel_id, server LN context
 2. [LN]    open_channel / accept_channel → both base BOLT-3 funding pubkeys known
 3. [local] build board tx (VTXO output = cosign-taproot musig(A,S) + expiry leaf);
            build the unsigned leaf exit tx; build the bridge over leaf_txid:0
            (out0 = P2WSH 2-of-2(funding pubkeys), out1 = P2A, nSequence=exit_delta,
            v3, 0-fee).                                            (do NOT broadcast)
 4. [Ark]   RequestBoardCosign (channel_id, +leaf nonce, +bridge nonce) → server
            reconstructs the leaf AND the bridge (funding keys looked up by
            channel_id), cosigns both; finalize (§7)
 5. [LN]    provide bridge_txid:0 as the funding outpoint
            (unsafe_manual_funding_transaction_generated); FundingCreated/Signed →
            initial commitment exchanged (stock ECDSA 2-of-2)
 6. [gate]  SAFETY GATE: leaf + bridge + initial commitment all exist
            ──────── LAST SAFE ABORT (nothing on-chain) ────────
 7. [local] sign + broadcast the board tx               *** POINT OF NO RETURN ***
 8. [local] board confirms → virtually confirm funding (board height);
            RegisterBoardVtxo; [LN] ChannelReady — channel usable
```

### 8.2 Operate

Stock BOLT over the funding outpoint, unchanged. The `musig(A, S)` cosignature
signs the **bridge** (once, at open); the commitment is plain BOLT-3 ECDSA
2-of-2 and updates without any Ark involvement.

### 8.3 Refresh + the (non-taproot) teleport

```
 1. [Ark]   SubmitRoundParticipation: forfeit old channel VTXO (intent), request
            a fresh channel-funding output (→ unlock_hash; server may cap at V−X)
 2. [Ark]   poll until ISSUED (round tx broadcast); new channel VTXO is a round leaf
 3. [gate]  MONEY GATE: round tx confirmed → new VTXO virtually confirmable
            ─────────── LAST SAFE ABORT (old channel intact) ───────────
 4. [local] build the NEW bridge over the new VTXO output, reusing the channel's
            existing funding keys → new_funding_txo = new_bridge_txid:0
 5. [Ark]   RequestLeafVtxoCosign (channel id, +leaf nonce, +bridge nonce) →
            server reconstructs new leaf AND new bridge (reusing the channel's
            funding keys), cosigns both; finalize (§7)
 6. [tele]  Stfu (quiesce) → settle in-flight HTLCs → TeleportInit(new_funding_txo,
            no nonces)/TeleportAck → commitment_signed on new_funding_txo (stock
            ECDSA) → wait for peer acceptance
 7. [verify] new exit path complete: new leaf + new bridge + new commitment
            ─────────── LAST SAFE ABORT (old channel intact) ───────────
 8. [Ark]   forfeit the OLD channel VTXO              *** POINT OF NO RETURN ***
            → unlock_preimage
 9. [tele]  TeleportComplete/CompleteAck → both promote the new funding scope;
            channel stays open (no ChannelReady)
10. [local] finalize the new VTXO's exit path with the preimage; drop old exit path
```

**Non-taproot teleport variant.** Identical state machine, promotion rule,
reconnect/persistence, and `responder_value_removal_sat` handling. Two changes
from the taproot teleport: (a) the new-scope funding spend is ECDSA 2-of-2, so
`teleport_init`/`teleport_ack` carry **no** `next_local_nonces` (nothing to roll
per commitment); (b) `new_funding_txo` points at the new **bridge** output, not
the VTXO output. The bridge's own one-shot cosign rides on the leaf cosign
(step 5), not the teleport, so the teleport gains no cosign step. Because the
teleport **reuses** the funding keys (§6), the new bridge is buildable as soon as
the new VTXO is issued — so there is no per-scope key exchange and no reorder
relative to today's leaf-cosign position.

### 8.4 Offboard

Unchanged from ARK #8: quiesce, read the settled balance, the standard ARK #7
connector-bound forfeit of the channel VTXO (`musig(A, S)`), broadcast the
offboard. The bridge and commitment are simply never broadcast.

### 8.5 Unilateral exit / force-close

```
1. Exit the VTXO (ARK #6): broadcast the genesis chain root-first until the
   channel VTXO output confirms on-chain (TRUC, P2A/CPFP).
2. Broadcast the BRIDGE: presigned, nSequence=exit_delta → confirms exit_delta
   blocks after the channel VTXO output confirms (its own P2A/CPFP package).
3. Broadcast the COMMITMENT: stock BOLT-3, spends bridge output 0 immediately.
4. Claim to_local / to_remote / resolved HTLCs per BOLT-3 (referenced).
```

The unilateral-exit delay is the bridge input's `nSequence` (not a VTXO leaf, not
the commitment input). Server post-expiry recourse is the VTXO's expiry-sweep leaf
(§4): after `expiry_height` the server sweeps the channel VTXO output directly, one
hop above the bridge, without broadcasting the bridge or commitment.

---

## 9. HTLC timing

The **HTLC success-path CSV** (`<exit_delta> OP_CSV OP_DROP` on the commitment's
preimage branch) is unchanged and still in scope: it keeps the server's
HTLC-timeout path from losing a race to a counterparty preimage-claim after a
force-close, so the server never has to unwind the VTXO tree. It is a property of
the commitment's HTLC outputs and is unaffected by the bridge.

The **HTLC CLTV budget** floor (`vtxo_exit_delta + max_vtxo_exit_depth +
cltv_safety_margin`) gains the bridge as effectively **one extra exit-chain
level** (depth `+1`), absorbed by the safety margin. No structural change.

---

## 10. LDK fork impact

Dropped (taproot-design artifacts): taproot funding output + gating, the three
`InMemorySigner` MuSig2 funding sighash sites, `with_overridden_funding_key`,
the prevout-mismatch fix, the custom commit-TX shape (`nSequence=exit_delay` +
`OP_RETURN`), the `OP_RETURN` decoder, the Ark-commit `ChannelMonitor` recognizer,
and the teleport `next_local_nonces` exchange + post-teleport nonce refresh.

Kept: the `ArkChannel` channel-type bit + negotiation + `anchor_zero_fee_commitments`
inheritance + `verify_channel_type_features`; the HTLC success-path CSV (script,
HTLC-tx `nSequence`, weight predictors); the teleport state machine
(quiescence-driven, reconnect/persistence, `responder_value_removal_sat`) in its
non-taproot form; the wallet-side bridge construction, registry, virtual funding,
and `unsafe_manual_funding_transaction_generated()` wiring.

(Reconciling the fork to this is out of scope for this doc — we agreed not to
touch branches. The point here is only to confirm the spec describes the simpler
target.)

---

## 11. Spec change plan

**ARK #2** — one edit to `channel-funding` (`0x08`): reword `A` as the
cooperative key (drop "holder BOLT-3 funding key"); change the exit-timelock
sentence from "the commitment transaction's input `nSequence`" to "the bridge
transaction's input `nSequence`." Structure (keyspend `musig(A, S)` + ASP
expiry-sweep leaf `timelock-sign(expiry_height, S)`, no user-exit leaf) retained —
and the expiry-sweep leaf is now firmly in scope (drop its Target-vs-reference
"keyspend-only" note, since in the bridge design the leaf is pure-Ark and never
touches the LDK funding output).

**ARK #8**, section by section:

1. **Overview / Actors & keys** — channel VTXO backs the channel via a presigned
   bridge instead of being the funding output; drop funding-key pinning; funding
   keys are ordinary BOLT-3 keys.
2. **"The channel-funding output" → "The channel VTXO and the bridge transaction"**
   — define the bridge (§5), framed as the last exit-chain tx; the funding output
   is stock P2WSH 2-of-2 with no sweep script.
3. **"The Ark channel type"** — delete the taproot-funding, commitment-input-CSV,
   and `OP_RETURN` bullets; keep HTLC success-path CSV, the CLTV budget (note the
   bridge `+1` level), and virtual funding.
4. **Channel open / Sequence** — insert "build + cosign the bridge" (§7, §8.1);
   safety gate covers leaf + bridge + commitment.
5. **Operation** — the `musig(A, S)` cosignature signs the bridge, not the
   commitment; the commitment is stock BOLT-3.
6. **Refresh / The teleport protocol** — §8.3; switch to the non-taproot variant;
   strike `next_local_nonces` from the message table and delete the two-stage
   nonce subsection; add the new-scope bridge cosign.
7. **Unilateral exit / force-close** — add the bridge broadcast step (§8.5);
   retitle the delay "bridge input."
8. **Messages** — extend the board cosign (open) and leaf cosign (refresh) with
   `channel_id` + `bridge_pub_nonce`, and add the bridge partial to each response
   (§7); **no new message**. Strike teleport `next_local_nonces`.
9. **Offboard / Security notes** — minor wording (bridge never broadcast; exit
   guarantee now leaf + bridge + commitment).

---

## 12. Decisions

**Settled:**

1. **Exit delay** — bridge input `nSequence`; channel VTXO carries no *user-exit*
   leaf (its only tapleaf is the ASP expiry sweep, §4).
2. **Funding keys** — stock LDK keys, no import; nothing in ark info.
3. **ASP sweep leaf** — *in scope*, on the channel VTXO output (pure-Ark); the
   commit funding output has no sweep (no LDK custom-input support).
4. **Bridge cosign** — folded into the board/leaf cosign (no new message),
   concurrent with the leaf, both before the forfeit/board-broadcast.
5. **Reconstruction** — client sends minimum data (`channel_id` + bridge nonce);
   server reconstructs both leaf and bridge from its stored channel state (both
   funding keys, keyed by `channel_id`), validates, cosigns; declines if
   `channel_id` is unknown. Funding keys are not re-sent — a holder pubkey is not
   a unique channel identifier (it may be reused across channels), and the server
   holds the authoritative copies.
6. **Refresh reuses funding keys** — the teleport clones the prior funding scope's
   keys (`for_teleport`), *not* splice rotation; verified in the fork (§6). So
   there is **no** per-scope key exchange and **no** reorder: the folded leaf+bridge
   cosign stays at today's leaf-cosign position, and the teleport only drops
   `next_local_nonces` and retargets `new_funding_txo` to the bridge output (§8.3).

The full lifecycle is walked in §8.
