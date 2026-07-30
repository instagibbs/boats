# ARK #8 channels: user stories

Lightning channels whose funding output lives inside an Ark — working name
"Lark". This document states what the channel work delivers as user stories,
from the perspective of the people who run it. It is the product framing that
accompanies the protocol spec (`08-channels.md`): the spec says *how*; this
says *for whom and why*. Section references are to `08-channels.md` unless
noted.

## Personas

* **Mobile user** — a wallet on a phone: online intermittently, never wants to
  think about timelocks, holds one balance and expects payments to just work.
* **Self-hosted user** — an always-on node (merchant, power user): runs the
  same client, can act on deadlines promptly, may hold many channels.
* **Server** — the Ark server operator, who is also every channel's only
  Lightning peer and the users' gateway to the wider network.

Stories apply to both user personas unless marked otherwise.

## Status

* ✅ **Delivered** — normative in the spec and implemented (client, server,
  and LDK fork), exercised by the integration suite.
* 🧭 **Goal** — a committed product goal whose protocol design is not yet
  specified. Stated here so the deliverable is judged against it; the open
  questions are gathered in "Open design work" at the end.
* 📐 **Specified** — the protocol design is normative in the spec;
  implementation and tests pending.

---

## Open

* ✅ As a user, I want to deposit on-chain bitcoin into a Lightning channel
  with **one on-chain transaction**: board a `pubkey` VTXO, and the moment
  its confirmation registers, the upgrade opens the channel instantly —
  off-chain, with no second wait. ("Channel open")
* ✅ As a user, I want an aborted open to cost nothing: until I register the
  signed upgrade, every signed object spends an output the server cannot
  actualize. The safety gate — full exit chain, cosigned bridge, initial
  commitment — must hold before the point of no return. ("Open by upgrade")
* ✅ As a user, I want to hold **many channels**, opened and managed
  independently — and since the server is every channel's peer, MPP makes
  their combined capacity fungible for payments.
* ✅ As a server, I want to accept any well-formed open without a policy
  decision, because an open fronts **none of my liquidity** — the entire
  capacity is the user's own VTXO; my side starts at zero (unless the user
  pushes part of its own balance to me at open).
* ✅ As a server, I want every backing output's expiry bounded at admission,
  so an absent user can never park my future channel earnings behind an
  unbounded client-held exit. (ARK #3 expiry admission bound; the upgrade
  inherits the input's already-bounded expiry.)
* 📐 As a user, I want to **upgrade** any VTXO I hold into a channel — an
  arkoor self-spend into the `channel-funding` policy — so an open from
  off-chain balance is instant: no round to wait for, no on-chain footprint,
  no confirmation wait (the anchor confirmed long ago). The next channel
  refresh then resets the inherited expiry and depth as it would anyway.
  ("Open by upgrade")
* 🧭 As a user, I want an incoming **Lightning payment to become a channel**:
  the server is both the arkoor sender and the channel peer, so it can fund a
  new channel VTXO on the spot and land the payment in it — settle-then-fund,
  or LSPS2-style forwarding of the HTLC over the just-created channel. ("Open
  design work", receive-into-channel)
* 🧭 As a user, I want to receive an **Ark-native (arkoor) payment** into a
  channel; the likely shape is receive-as-`pubkey` with an instant
  self-upgrade behind it, since a third-party sender won't interleave with
  channel establishment.

## Operate

* ✅ As a user, I want my channel to be a **standard Lightning channel**: pay
  and receive ordinary invoices, with everything my Lightning stack supports,
  and no Ark involvement per payment — the Ark boundary stops at the funding
  output. ("Operation")
* ✅ As a server, I want to forward my channel users' payments to and from the
  wider network as stock BOLT forwarding; the Ark channel type is a bilateral
  detail between me and the user, invisible to my routing peers.
* ✅ As a user, I want HTLC time budgets that already account for my longer
  unilateral path (exit chain + bridge + second stage), so no in-flight HTLC
  can outlive my ability to enforce it on-chain. ("The Ark channel type",
  `cltv_expiry_delta` budget; "The force-close deadline")
* 🧭 As a mobile user with no inbound capacity, I want the server to provision
  it **just in time** when a payment is already on the way — the server funds
  a new channel VTXO on the spot and the payment lands in it — so receiving
  just works. This is the flagship UX goal the open-path 🧭 items above exist
  to serve. ("Open design work", receive-into-channel)
* 🧭 As a user, I want incoming payments to land with **full security
  whenever my round/board-backed inbound covers them** — the trust-window
  (JIT) path is only the fallback, gated by my wallet's policy and chased by
  an immediate refresh. The server, as counterparty on every one of my
  channels, knows my aggregate inbound exactly, so the routing decision has
  no information problem. ("Open design work", receive routing)

## Refresh: the channel outlives its VTXO

* ✅ As a user, I want my channel to outlive VTXO expiry **without closing**:
  a refresh re-points the live channel at the fresh scope (the teleport, like
  a splice), carrying my balance and any committed HTLCs across, resetting
  expiry and exit depth. ("Refresh", "The teleport protocol")
* ✅ As a user, I must never be exposed mid-refresh: I verify the new scope's
  complete exit story before forfeiting the old one — the forfeit is the point
  of no return, and everything before it is a safe abort back to the intact
  old channel. (exit-before-forfeit, ARK #4 discipline)
* ✅ As a server, I want to **end my channel commitment of fronted liquidity
  at the next refresh** — the withdrawal leg comes off my side of the new
  commitment, entirely off-chain — so refresh cadence bounds my channel
  exposure; the withdrawn value joins the same maturity pipeline as any
  swept output. ("Server liquidity adjustment")
* ✅ As a server, I want to charge a refresh fee off the client's side,
  verified against my published schedule as a floor. ("Server liquidity
  adjustment")
* ✅ As a user, I want my wallet to stop offering new HTLCs and force the
  refresh-or-close decision while there is still enough runway to resolve
  everything on-chain — the deadline discipline is computed, not left to
  judgment. ("The force-close deadline")
* 🧭 As a mobile user, I never want to *think* about expiry: my wallet
  refreshes opportunistically whenever it is online with margin to spare, and
  warns me out-of-band (push) if the deadline nears without a chance to act.
  The mechanics (refresh, deadline math) are delivered; the mobile scheduling
  and notification story is the open part.
* 🧭 As a user, I want to **consolidate many small channels into one** at a
  round — close-N + open-1 atomically, not a teleport — so topping up by
  opening new channels doesn't fragment me forever. Not critical path.
  ("Open design work", consolidation)

## Cooperative close

* ✅ As a user, I want to close a channel **without leaving the Ark**: the
  standard BOLT close fixes the balances, and a **downgrade** — an arkoor
  split of the channel VTXO into plain `pubkey` VTXOs matching them —
  settles it off-chain, instantly. A balance too small to pay out on-chain
  settles here at full value. ("Downgrade: close into Ark balance")
* ✅ As a user, when I want the funds on-chain, the ordinary offboard
  (ARK #7) of my post-downgrade `pubkey` VTXOs pays them out — nothing
  channel-aware left in the path. ("The close")
* ✅ As a user, between close and settlement I keep a no-server fallback —
  the signed closing transaction over my own exit chain — so a server that
  stops cooperating after the close cannot strand my balance. ("The close",
  fallback)

## Unilateral exit / force-close

* ✅ As a user, I can always recover **everything — balance and in-flight
  HTLCs — with zero server cooperation**: exit the VTXO, broadcast the bridge,
  force-close, claim per BOLT-3. The costs are stated up front: act before the
  deadline, and hold an on-chain fee reserve to bump the chain. ("Unilateral
  exit / force-close", "Trust assumptions")
* ✅ As a server, **I never unroll a tree on my own.** Every unilateral
  broadcast in the system is the user's; my only unilateral on-chain act is
  the expiry sweep of an output already on-chain. ("Server recourse after
  expiry")
* ✅ As a server, a user who vanishes forever costs me exactly: capital locked
  until `expiry_height`, then one sweep transaction. Bounded, priced, and
  known at admission.
* ✅ As a server, forfeited state cannot beat me to the chain: a forfeit
  spends ahead of the bridge's `exit_delta` with no timelock, and my
  forfeit-watch duty — surviving restarts — enforces it. ("Refresh", the
  close and watch composition; ARK #7)
* ✅ As a user, a server that broadcasts revoked channel state is punished
  exactly as in stock Lightning — BOLT-3 penalties, unchanged, including
  second-stage claims along the Ark exit path.

## Money-safety assurances

Each of these is a *cannot happen, because* claim, not a policy.

* ✅ The server cannot move my funds without me: every Ark transition — bridge,
  forfeit, refresh, split — requires my half of `musig(A, S)`.
* ✅ A client cannot lie about a refresh's value split: the declared removals
  must sum to exactly `V_old − V_new` against the bridge the server cosigned,
  and the client-side removal must meet the fee floor — over-declaring only
  hurts the declarer. ("Server liquidity adjustment")
* ✅ A closed-and-paid channel gives the user no second claim: after the
  forfeit, neither commitment nor closing transaction can reach the chain.
* ✅ An expired HTLC resolves deterministically to the timeout side: only the
  success path carries the pinned `exit_delta` CSV — the timeout claim is
  baseline BOLT-3 and strictly leads any late preimage claim — and the
  success-path delay is priced into the CLTV budget. The
  success-CSV-equals-pinned-`exit_delta` derivation is guarded by an
  implementation self-check at the open cosign; the protocol itself
  surfaces a divergence only at the first HTLC's commitment exchange.
  ("The Ark channel type", HTLC success-path CSV)
* ✅ Neither side's crash creates a window for the other: everything
  safety-critical is stated as an observable property that must survive a
  crash — close outcomes, teleport promotion, forfeit watching — and a party
  that cannot recover its state fails **closed**, never open. ("Trust
  assumptions", crash-safety requirements throughout)

## Server economics

The epic: **bounded, priced, fixed-maturity server capital — while the
client UX stays instant.**

Server capital comes back on-chain exactly one way: the expiry sweep of a
scope's backing output. There is no early exit — not a refresh, not a
cooperative close — so every sat the server commits to a scope is a loan
with a known maturity (`expiry_height`), sized and dated at admission. The
cooperative machinery controls how much gets re-committed each cycle, and
prices it:

* ✅ A user-funded open commits nothing of the server's.
* ✅ At a refresh, the withdrawal leg and the refresh fee shrink what rolls
  into the new scope.
* ✅ Every cooperative channel operation is priced on the published
  schedule where it fronts server capital: board, refresh (floor-verified
  at the teleport). The downgrade fronts nothing and is not separately
  priced; the post-downgrade offboard is the vanilla ARK #7 flow with its
  own fee.
* ✅ The worst case is the base case: an uncooperative or vanished user
  doesn't degrade recovery — same sweep, same maturity, one transaction.
  Users choose how much the server fronted, never how or when it comes
  back.

Provisioning — how capital gets onto the server's side at all, and its
lease terms — is the open design. (🧭 "Open design work", JIT liquidity)

## Operations and robustness

* ✅ As a server, I restart at any moment without force-closing a single
  channel: channel state, chain view, and every in-flight ceremony are
  persistent and resume or fail closed.
* ✅ As a user, a crash at any point in any flow — open, refresh, close — leaves
  me in a recoverable state; no flow depends on my having personally witnessed
  an event, only on the recorded outcome.
* ✅ As either party, one consistent real-chain view governs everything:
  virtual funding never manufactures depth, and reorgs are observed in order
  everywhere. ("Trust assumptions")

## Open design work

The 🧭 items gathered, as the questions to answer next:

1. **The arkoor channel-creation primitive.** The board already shows the
   pattern — channel fields (`channel_id`, bridge nonce) added to an existing
   cosign request. Extending the same two fields to the arkoor cosign
   (ARK #5) yields both missing open paths at once:

   * **Upgrade** (self-spend into `channel-funding`): an instant open from
     off-chain balance. The ordering is clean — the arkoor tx's txid is
     independent of its signatures, so the bridge txid is known up front and
     the initial commitment can be exchanged *before* the arkoor + bridge
     cosign, giving the full safety gate before anything is signed (the
     board's gate, with "broadcast" replaced by "register" — ARK #5
     registration is the point of no return, since the intact input exit
     remains the fallback until then). A self-spend adds no arkoor
     double-sign trust:
     only the holder can request spends of its own input. The upgraded
     channel inherits the parent's expiry and depth (+1, or +2 through a
     checkpoint) — admission needs a remaining-runway floor, and the standard
     channel refresh resets both at the next round. Round-issued opens are
     subsumed: open = upgrade, maintain = refresh. **Now specified**: "Open
     by upgrade" and the `arkoor_cosign_request` variant in "Messages".
   * **Board unification** (intended endpoint): once upgrade exists, the
     ARK #3 channel-board variant is deleted — an on-chain open becomes a
     *vanilla* board plus an instant upgrade once the board is spendable.
     The user-visible stories are unchanged (still one on-chain
     transaction, still no extra wait: the upgrade is an off-chain
     round-trip behind the same confirmation), and every open converges on
     one flow, one point of no return, one failure mode — a failed
     establishment leaves a plain `pubkey` VTXO, never limbo. Costs: the
     channel VTXO opens at one extra exit-chain depth until its first
     refresh, and the intermediate `pubkey` VTXO's delayed-exit leaf
     creates a parent-exit race the server must watch — its defense is
     broadcasting the upgrade transaction it holds, so `ChannelReady` MUST
     gate on ARK #5 registration completing. Same class as forfeit
     watching, but load-bearing for channel balance — now specified as the
     registration gate and the parent-exit watch ("Open by upgrade").
     **DONE**: the board variant is deleted — every open is a vanilla board
     (when starting from the chain) plus the upgrade.
   * **Receive-into-channel.** Split by who the arkoor sender is. For a
     **Lightning receive** — the JIT case — the sender is the *server*,
     which is also the channel peer, so establishment interleaves freely.
     Two candidate shapes: *settle-then-fund* (the server settles the
     inbound HTLC by arkooring the value into a fresh channel-funding
     VTXO), or *LSPS2-style* (the server funds a channel VTXO from its own
     balance, then forwards the HTLC over the just-created channel and the
     user claims it in-channel per stock BOLT — which is also where fronted
     inbound capacity above the payment amount comes from). Either way the
     invariant stands: a channel-funding VTXO with no cosigned bridge has
     **no unilateral exit at all**, so receipt must not complete before
     bridge + commitment exist. For an **Ark-native receive** (third-party
     sender, who won't interleave with establishment), the honest shape is
     receive as `pubkey` with an instant self-upgrade behind it (one extra
     depth level). Any arkoor-created scope carries double-sign trust until
     its first refresh — no worse than today's baseline arkoor receive
     (ARK #5's deliberate trade-off) — so policy should force a refresh at
     the next round.

2. **Channel consolidation (N-to-1 at a round).** Top-up needs no new
   machinery: open another channel via the upgrade primitive, and MPP
   across the user's channels (the server is every channel's peer) makes
   capacity fungible. What that leaves is fragmentation, fixed by
   consolidation: **close-N + open-1 atomically in a round** — not a
   teleport (a teleport re-points one live channel; a merge is inherently a
   fresh channel). The N channels close at the BOLT layer — balances
   terminally fixed, closing transactions never broadcast, HTLCs fully
   drained (a real close, unlike a teleport's quiescence) — then the round
   forfeits the N channel VTXOs and issues one new channel-funding output,
   with a fresh channel established over the new bridge whose initial
   commitment carries the summed balances. Exit-before-forfeit throughout:
   the per-channel closing transactions are the fallbacks between close and
   forfeit, and the new channel's complete exit story must exist before any
   forfeit. N = 1 plus extra `pubkey` inputs is top-up by close-and-reopen
   in one round. The teleport value rules stay shrink-only. Not critical
   path. (A non-atomic alternative that needs no channel-aware round
   machinery at all is the downgrade composition, item 5.)

3. **JIT liquidity: sizing, pricing, and the trust window.** Up-front
   inbound fronting at open is not a separate feature: settle-then-fund
   needs no net fronting (the server is made whole by the HTLC it settles),
   and inbound develops organically once the user spends. Fronting is the
   **sizing knob inside JIT** — how far above the payment the server sizes
   the channel (LSPS2-style headroom, so a stream of small payments doesn't
   spawn a channel each) — plus the lease terms that make it worth it. Open
   questions:

   * **The trust window.** An arkoor-created channel leaves the user in the
     double-sign (statechain-like) model until a refresh replaces it with
     the round guarantee. Candidate mitigation: the client initiates a
     refresh immediately, shrinking the window to the next round — the
     server was fronting this channel anyway, though the liquidity
     implications aren't thought through. The trust-clean alternative — the
     server funding an on-chain board — buys full security at the cost of
     the confirmation wait JIT exists to avoid. Whether an instant *and*
     trustless JIT receive is achievable is open.
   * **Receive routing.** Policy: forward in-channel whenever existing
     round/board-backed inbound covers the payment (MPP across the user's
     channels included); back off to JIT otherwise. Incentives align — the
     in-channel path is cheaper for the server too (no ceremony, no new
     scope to maintain). The open points: the backoff changes the user's
     security model, so it must be opted into — wallet policy signaled at
     invoice time or negotiated LSPS2-style, with a strict
     fail-rather-than-trust tier (which needs "inbound" to mean
     round/board-backed scopes, not merely existing ones); **no mixing in
     v1** — one payment rides one rail, never shards split across an
     in-channel BOLT settle and an arkoor ceremony under a single preimage;
     and the JIT path holds the inbound HTLC across an interactive
     ceremony, so CLTV budgets must include ceremony time and a
     mid-ceremony failure must fail the HTLC back cleanly. The fallback's
     trust window is the off-chain analogue of a zero-conf JIT channel —
     and strictly better: it closes deterministically at the next round, on
     the client's initiative, at zero on-chain cost.
   * **Pricing.** How leased capacity is priced, for how long, at what fee.

4. **Mobile expiry UX.** Opportunistic auto-refresh scheduling and
   out-of-band deadline warnings for a wallet that is mostly asleep. The
   protocol deadline math is delivered; the product surface is not.

5. **Channel downgrade (close into Ark balance).** The upgrade run in
   reverse: a standard BOLT close terminally fixes the balances, then an
   arkoor split spends the channel VTXO into plain outputs — `pubkey(A)`
   for the user's balance, `pubkey(S)` for the server's — leaving the
   user's funds off-chain. It reuses
   the close's whole discipline: close first (a real `shutdown` drain,
   not quiescence — the close outcome is the signed object the server
   verifies the split against, and nothing times out mid-ceremony), the
   closing transaction as the user's fallback until registration, ARK #5
   registration as the point of no return. The split spends the VTXO
   output by key path at `nSequence = 0`, ahead of the old bridge's
   `exit_delta` — the forfeit's role — so both sides retain the split
   transactions and watch the old chain (the parent-exit-response duty,
   made symmetric). Admission: the server MUST NOT cosign any arkoor spend
   of a `channel-funding` VTXO except this sanctioned split, verified
   against the recorded close outcome. **Now specified**: "Downgrade:
   close into Ark balance" and the downgrade note under
   `arkoor_cosign_request` in "Messages" — with no new wire fields at all.
   **DONE**: the channel offboard and its amended amount rule are deleted —
   an on-chain close is downgrade + the vanilla ARK #7 offboard.

   The empty-channel precondition costs a wallet nothing: the client is an
   endpoint, not a forwarder — inbound HTLCs settle at its own discretion
   (no long-running inbound short of its own hold invoices), and outbound
   HTLCs are its own payments — so it starts the downgrade only with
   nothing in flight and the drain completes in seconds.

   Composition then pays for itself three ways, with zero new round
   machinery: **consolidation** — downgrade, let the plain VTXOs batch
   through ordinary rounds with the rest of the wallet, upgrade back —
   done **piecemeal**, one channel at a time: staggered cycles are never
   channel-less (no receive gap), the liquidity reset touches only the
   channel being consolidated, and each step aborts independently, so
   item 2's atomic N-to-1 round shape is subsumed; a slow maintenance
   path a client without teleport could live on; and **offboard
   unification** (**DONE**, mirroring item 1) — a channel close that wants
   the chain is downgrade + a *vanilla* ARK #7 offboard of the resulting
   `pubkey` VTXOs: the exact-balance rule holds unchanged, the payout
   batches with the user's other VTXOs, and the server's share arrives as
   an explicit `pubkey(S)` output rather than implicitly via a forfeit.
   Open = upgrade; close = downgrade: channels touch the generic flows only
   at upgrade, refresh, and downgrade. Teleport remains
   *the* refresh
   mechanism: its round pre-commitment is unconditional (the leaf request
   needs only capacity, and committed HTLCs carry across), where any
   close-based path is conditional on an empty channel at freeze time. A
   quiescence-based variant that splits dangling HTLCs out into
   hash/timeout VTXOs is conceivable as an escape hatch; nothing above
   needs it. Server-side balance returns to the server at the split, so
   inbound liquidity at re-upgrade is re-provisioned per policy — natural
   for a deliberate, user-chosen operation. Not critical path.

---

Every ✅ story is intended to map to at least one integration test; making
that mapping explicit is deferred until the story set settles.
