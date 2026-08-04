# MR-3 design note (G1): captaind as the channel counterparty — the embedded-LDK channels subsystem

**Status**: G1 rework (codex REWORK → this revision, 2026-08-04). The prior
draft turned forwarding ON in this MR; the review showed that was both
unimplementable with the named LDK knobs and premature (payments are part 4).
This revision pulls forwarding out, restoring the plan's staging, and applies
the review's other findings. Codex record:
`docs/plans/2026-08-04-codex-mr3-g1-review.md`.
**Series position**: part 3 of 6 upstream (internal "MR-2 captaind"), stacking
on the protocol-surface MR (!2336) → the release-contract opener (!2321).
**Baseline**: bark-stage1 `ark8-channels-stage1-protocol` (rebase to tip at
code-start). **Theme (plan:370)**: *captaind can be the channel counterparty.*
**Spec targets**: IB-1..7 (upgrade path), PV-6/PV-8/PV-10 (server halves),
BR-3/4/8/12/13/14/15/16/17/18, OP-1..7/14..16/18..28 (server view),
RG-1..8/14..16, WD-2..5/15/16, EX-1/7, DA-1..10 (server + config guard),
I-4/5/9/10 — full discharge table in §12. (I-6 forwarding floor: part 4.)

## 1. Scope: what this MR is, and the stage-1 cut it is drawn against

captaind gains an **optional** embedded Lightning node so it can be the
counterparty of a channel whose funding output lives, unbroadcast, in a
VTXO's exit chain. Concretely: accept an inbound Ark channel on stock
released LDK, admit an **open-by-upgrade** riding the arkoor cosign, hold
`ChannelReady` until the Ark transfer is durably registered, cosign the
presigned bridge, stand the parent-exit watch that defends the balance after
the input is upgraded away, and sweep the channel VTXO at expiry. The channel
is then a live, usable Lightning channel between the client and captaind.

**captaind is an HTLC *endpoint*, not a *forwarder*, in this MR.** A client
and captaind can pay each other over the channel — an ordinary Lightning
HTLC where captaind is the terminal hop — and MR-3 tests exactly that
(cooperative, off-chain settlement; §6a). What MR-3 does **not** do is
*forward* (route A→captaind→B). Forwarding is where the deferred HTLC-race
safety actually bites (§3), and the plan already places intra-ark payments —
which are, by construction, captaind forwarding — in **part 4** alongside the
per-HTLC CLTV floor. So forwarding ships **configured off** here; its safety
layer (a static forwarding-CLTV ceiling, mandatory bridge retention, and a
server-side HTLC-expiry force-close scheduler) is designed and built with
part 4. Pulling it out of MR-3 is not a compromise — it restores the plan's
staging.

**In scope**: the `[channels]` `OptionalService` subsystem; LDK node
assembly + persistence + event loop; the open-by-upgrade admission path (the
server half of the two-sided checks); the registration gate; the bridge
cosign integration; the parent-exit watch; the expiry sweep arm; the
`max_vtxo_exit_depth` config-decrease guard; `supports_channels` advertised
true when enabled; cooperative client↔captaind payments as a usability
proof.

**Out of scope**, and the note keeps clear of each:
- **Forwarding + intra-ark payments (A→captaind→B)** — part 4, with the
  forwarding-safety layer above. captaind's LDK `UserConfig` sets
  `accept_forwards_to_priv_channels = false` and no channel is configured to
  route; the subsystem accepts and terminates HTLCs but does not relay them.
- **The bark client open flow and unilateral exit e2e** — part 4. This MR
  builds the server half and proves it against a test driver, not the
  production client. (Unilateral *exit* recourse the server owns — expiry
  sweep, parent-exit watch — is in scope; the client's exit *action* is
  part 4.)
- **Downgrade/close** — part 5. The parent-exit watch table lands here with
  the *upgrade* kind; part 5 adds the downgrade kind and the split-response
  watch + its crash tests.
- **Refresh / teleport** — excluded from stage 1 entirely. The reference
  branch's teleport apparatus (the C1 promotion gate, the I-1 refresh
  binding, the M3 declared-removal check, `pending_teleport*` tables) is
  **not** ported. A channel-VTXO never refreshes in stage 1; expiry is
  handled by downgrade → round-refresh the plain funds → re-upgrade (the
  part-5 loop). This is why depth admission reserves split headroom at open
  (§6): the in-place refresh remedy does not exist.
- **The Ark channel type / HTLC-success-CSV asymmetry** — stage 2 (the LDK
  fork). Stage-1 captaind negotiates a **stock** channel type
  (`anchors_zero_fee_commitments`) and refuses any other; it does NOT gate on
  an `ark_channel` feature bit (there is none). The HTLC expiry-race safety
  the fork's CSV asymmetry would give is what makes safe *forwarding* hard —
  which is the direct reason forwarding waits for part 4, where the safety
  layer is designed against this constraint (and may require the fork).

**The reference implementation is mined, not ported.** The old branch
(`ark-channels-bridge-2026-06-18`, server side in `server/src/ldk/*` +
`bark-lightning`) is the design source for the admission order, the watch
machinery, and the hardening fixes (§11). Three deliberate divergences are
called out where they arise: the gate mechanism (§5, message interception →
confirmation injection), the floor formula (§6, the old `2·exit_delta`
conflation → the spec's unified `F`), and the subsystem boundary (§2, no
`bark-lightning` shared crate — the production node lives in `bark-channels`,
promoted from its test harness).

## 2. Architecture and module layout

### 2a. The subsystem gate

`Config.channels: OptionalService<crate::channels::Config>` (config.rs),
following the sole existing production use of `OptionalService` — `watchman`
(config.rs:349). Disabled by default; a disabled section omits every other
key. `supports_channels` in ark-info (wired inert in !2336 as `false`,
lib.rs:284-286) becomes `cfg.channels.enabled().is_some()`.

**The channels subsystem depends on an always-available watcher (§7).** A
money-safety watch (the parent-exit response) cannot be optional while
channels are enabled. MR-3 runs the watch on an **always-on embedded**
watcher inside the channels subsystem; it does not rely on the separately
toggled `watchman` service being enabled. (If a deployment prefers the
standalone `watchmand` host, channel startup MUST verify it is present and
healthy before accepting opens — but the default is embedded-and-always-on.)

### 2b. Where the code lives

Upstream has no `bark-lightning`; the production node lives in the
**`bark-channels`** crate (today the release-contract harness, MR-0), whose
`tests/common/mod.rs` is the direct precursor of the production assembly.
`src/lib.rs` grows the real node; the server depends on it as a workspace
path dep. Server-native glue lives in `server/src/channels/` (mirroring
`server/src/watchman/`):

| Module | Purpose | Harness precursor |
|---|---|---|
| `channels/mod.rs` | `ChannelsSubsystem`, handle, `Ctrl` | — |
| `channels/config.rs` | the `[channels]` `Config` | — |
| `channels/node.rs` | node assembly, reload-vs-fresh, the durable open state machine (§6b) | `common::build_node`, `restart_from_persisted` |
| `channels/persist.rs` | LDK monitor `Persist` over postgres via the dedicated executor; a small `KVStore` for the singleton manager (§2c) | `common::InMemoryPersist` |
| `channels/db_executor.rs` | separate tokio runtime + pool for sync LDK persist | (old `ldk/db_executor.rs`) |
| `channels/broadcaster.rs` | capture-only broadcaster over the tx nursery | `common::CaptureBroadcaster` |
| `channels/event.rs` | `process_events_async` driver; close recording | `common::spawn_event_pump` |
| `channels/admission.rs` | open-by-upgrade admission (§6), the arkoor-cosign server half | old `cosign_oor_upgrade` |
| `channels/confirm.rs` | virtual-confirmation feed + SCID allocator (§5), the idempotent release outbox | old `feed_tx_confirmation` (with the collision bug fixed) |
| `channels/watch.rs` | parent-exit watch arm (§7) | old `ForfeitWatcher` parent-watch |
| `server/src/database/channels.rs` | `impl Tx` blocks for the new tables (§10) | old `database/ldk.rs` (minus teleport) |
| `lib/src/channel.rs` | (exists, MR-1) bridge construct/sighash/cosign | — |

`bark-channels`'s test doubles become production types backed by the tx
nursery, postgres, and `ChainEventListener`.

### 2c. Node assembly

Per MR-0's proven shape on stock `lightning` 0.2.4:
- **Funding keys (corrected per review F5)**: use the **stock
  `KeysManager`-derived per-channel funding key** — NOT the server's
  long-term `S`. The spec is explicit (PV-4, BR-14): the BOLT-3 funding keys
  are ordinary Lightning keys, distinct from the cooperative `A`/`S`. (The
  old branch appeared to override the funding key to `S`, but that override
  was dead at branch tip — its signer discarded it.) The server persists the
  canonical `funding_redeem_script` and funding output that LDK supplies on
  the `ChannelPending` event, and admission compares the bridge's out0
  against *that* (§6, check 12) rather than reconstructing keys.
- **Persistence**: a direct `chain::chainmonitor::Persist` impl over postgres
  for **monitors** (NOT a KVStore shim), calls routed through a **dedicated
  runtime + bb8 pool** (`db_executor.rs`) — load-bearing: LDK's synchronous
  persist-before-sign hangs under two concurrent opens if run on the shared
  runtime (§11.1). The **singleton `ChannelManager`** blob is persisted
  through a small postgres-backed async `KVStore` (see the event driver
  below).
- **ChainMonitor**: synchronous persist (`deferred = false`).
- **Reload vs fresh**: any monitor/manager deserialize failure **panics**
  (fail-fast) — a fresh manager would silently drop justice monitors.
  Restart lifecycle (from the MR-0 review): quiesce and await
  pumps/transports **before** snapshotting; restore dormant; re-`watch` every
  monitor; re-establish the real chain view and *then* replay committed
  registrations (§5); only then start processing and reconnect.
- **Router/peers**: no gossip, no pathfinding — peers are the server's own
  clients; the node terminates HTLCs but does not route. (It still processes
  onion payloads for HTLCs addressed to it; "no routing" ≠ "no onion".)
- **Event driver (D1, resolved per review F9)**: adopt the async
  `lightning-background-processor` 0.2.3 `process_events_async` — it composes
  with a custom synchronous monitor `Persist` and an external shutdown-capable
  sleeper (`rtmgr.shutdown_signal()`), and already drives
  `process_pending_htlc_forwards` and manager persistence. Manager
  persistence goes through the small postgres `KVStore` above; the monitor
  `Persist` stays custom. (If the `KVStore` adapter proves unpalatable in
  review, the fallback is the hand-rolled `rtmgr.spawn_critical` +
  `tokio::select!` loop — but the async processor is the plan of record.)

### 2d. Chain plumbing

The subsystem registers an `Arc<RwLock<…>>` as a `ChainEventListener`
(sync/mod.rs:53-81) in `Server::start`'s listener vec before
`SyncManager::start` (lib.rs:401-432) — the watchman pattern. `on_block_added`
drives (a) real chain sync for the node's own monitors and (b) the
parent-exit watch's detect step (§7); `on_reorg` unwinds virtual
confirmations, SCID allocations, and watch resolutions above the fork. The
capture broadcaster and the parent-exit response share the existing
tx-nursery / P2A-CPFP machinery.

## 3. The two-sided invariant, and captaind's HTLC role

Every open-time check in the spec is enforced **from each party's own view**
("the client MUST refuse to build, and the server MUST refuse to cosign",
spec:438-440). This MR is the **server** half; the client half is part 4.
Neither half trusts the other. Both sides pass `ChannelAuthorization::
UpgradeOutput` into the MR-1 builder — the client when it *builds* the
transfer, captaind when it re-derives from the cosign request (corrected: the
prior draft wrongly called captaind the sole `UpgradeOutput` caller).

**captaind's HTLC role in MR-3.** captaind may be the **terminal hop** of an
HTLC on a channel — it can receive a payment from its client, and send one to
its client — and this is the usability MR-3 proves (§6a). It does **not
forward**. Two consequences:
- captaind enforces its own **receive** CLTV floor without needing the
  forwarding machinery: it sets `min_final_cltv_expiry` on invoices it issues,
  and rejects any inbound HTLC whose CLTV is below its floor at the channel
  level (the HTLC's CLTV is visible before the HTLC is irrevocably committed,
  independent of the spontaneous-payload exposure timing that defeats a
  post-hoc keysend check). Whether captaind accepts keysend/spontaneous
  receipt at all is a config choice; if it does, the same pre-commit CLTV
  floor applies.
- captaind never holds the forwarder's "paid the outbound leg, inbound leg
  stuck" obligation, because it never forwards. That obligation (I-6d, and the
  force-close-to-recover it demands) is the part-4 forwarding layer's, and is
  the substantive reason D2/D3 below are safe *for this MR*.

## 4. Open by upgrade — the flow captaind participates in

The client self-spends a `pubkey` VTXO (ARK #5) into a `channel-funding`
destination; the transfer carries two channel fields (`channel_id`, bridge
nonce) so the server reconstructs and cosigns the bridge in the same
exchange. Establishment runs *before* the cosign, with **registration as the
point of no return** (spec:388-420):

```
 1 BOLT   open/accept → both BOLT-3 funding pubkeys known; LDK ChannelPending
 2 local  client builds transfer + bridge (out0 = P2WSH 2-of-2 funding)
 3 BOLT   funding outpoint = bridge_txid:0; exchange initial commitment
 4 gate   initial commitment held  ── last fully-free abort ──
 5 Ark    arkoor_cosign_request (channel variant) → SERVER admits (§6),
          marks input spent, returns ARK #5 partials + the bridge's
 6 local  client verifies every partial incl. the bridge's
 7 Ark    client registers the signed transactions  *** PONR ***
 8 BOLT   ChannelReady — released by feeding the confirmation, only after
          registration commits (§5)
```

captaind **accepts** the inbound channel at step 1 (funder is the client —
manual funding is funder-only, MR-0-confirmed; captaind never calls it, it
learns the outpoint from `funding_created`). It reaches step-1 LDK
`ChannelPending` *before* the cosign exists — which the durable open state
machine (§6b) must expect. It admits + cosigns at step 5, and releases
`ChannelReady` when step 7's registration commits, by feeding the synthetic
confirmation (§5). captaind does **not** hold a `ChannelReady` message from
step 1 — LDK has not generated one yet; the gate is that LDK never generates
it until fed (corrected from the prior draft).

## 5. The registration gate (THE core mechanism) — confirmation injection

**The load-bearing divergence from the reference branch, and the review
confirmed the primitive is real on stock 0.2.4.** The old branch accepted
zero-conf and intercepted the outbound `SendChannelReady` message. Stage-1
mechanism instead: **ordinary inbound accept at `minimum_depth ≥ 1`, and
withhold the virtual-funding confirmation feed until Ark registration
commits.** Verified against the crate
(`ln/channel.rs::check_get_channel_ready` + the release-contract test): with
`accept_inbound_channel` at `minimum_depth ≥ 1` and no `Confirm` feed,
neither timer ticks nor peer messages produce `ChannelReady`; feeding the
confirmation does. No message interception, no `ChannelManager` wrapper.

**Linearization (review C3 — the critical correctness fix).** Registration
and chain-event resolution race: registration could check "input's final exit
not confirmed," a block listener could then confirm-and-resolve the watch, and
registration could still commit and release `ChannelReady` against backing
already being clawed back. The gate MUST be linearized:

- Registration and the `ChainEventListener`'s watch-resolution **contend on
  one lock** — the channel's `channel_state` row (`SELECT … FOR UPDATE`), so
  the "is the input's exit terminal yet?" check and the commit cannot
  interleave with a resolving block.
- In **one DB transaction**: validate the complete signed level set (RG-3 —
  the whole chain, not merely the checkpoint), verify no terminal
  chain-derived resolution has been recorded for the input, persist the
  signatures, set the release marker (`backing_registered_at`, set-once,
  `COALESCE`-idempotent), and **arm the parent-exit watch** (§7). All-or-
  nothing (RG-4); a byte-identical re-upload is idempotent (RG-4); the set
  survives a crash (RG-5).
- The synthetic confirmation is fed **only after that transaction commits**,
  through an **idempotent outbox/reconciler**: the commit enqueues "release
  channel_id"; a reconciler feeds the node and is safe to replay. A crash
  between commit and feed is recovered at startup — the outbox still has the
  entry — so the channel is never stranded registered-but-unfed. Startup
  first catches the node up to the *real* chain, then drains the outbox.
- **Late-registration refusal (spec:489-494)**: a registration arriving after
  the input's exit-chain final tx has confirmed — or after a confirmed
  ancestor expiry sweep forecloses the input — MUST be refused, not completed.
  Both are recorded as the watch's terminal resolution (§7); the linearized
  check above reads that record under the same lock.

**The server feeds its own node the synthetic confirmation** (the release).
The old branch's server never did (its interceptor released readiness, and
its `feed_tx_confirmation` was dead code hardcoding `tx_index = 0`, §11.7);
stage-1's gate makes this path live, so the SCID synthesis below applies to
it.

**Virtual confirmation obeys the real chain (spec:120-144, OP-18..22).**
Fed at the input anchor's *actual* best-chain height; feed **both**
`ChannelManager` and `ChainMonitor`; historical (already-buried)
confirmations MUST NOT regress `best_block_updated`; the node MUST NOT advance
its best-chain view to manufacture depth. On a genuine deep reorg the anchor's
confirmation is withdrawn — and the profile's honest behavior here is
**not** "suspend operation" (there is no such stock primitive) but the stock
LDK response: funding unconfirmation triggers a force-close (confirmed by the
release-contract reorg test). The note records this as the accepted disposition
(profile OP-21 relaxation).

**SCID synthesis (I-10, spec:146-168; review F6).** A virtual funding has no
block position; the node synthesizes one: the anchor's real height, a
24-bit `tx_index` derived from the **bridge txid** (avoiding the coinbase
slot, `≥ 1`), and the funding vout. Requirements the design must satisfy:
- **Persist `{anchor_hash, height, tx_index, collision_bump}` transactionally
  *before* feeding**, with a **unique full-SCID constraint** in the schema
  (§10) and checked wrapping arithmetic on the bump.
- The synthesized SCID MUST be checked for collision against **every** local
  real SCID *and* alias — LDK inserts both into one map and panics on either
  collision (`ln/channelmanager.rs` SCID insert). Assert collision-fatality
  as a fast synchronous check (MR-0 pinned this shape).
- The channel MUST NOT be announced; **`negotiate_scid_privacy = true`** so
  `option_scid_alias` carries every wire reference (the harness did not
  negotiate it — this must be set, and the final `ChannelPending` type must
  support it).
- Tests: crash-before-first-feed (allocator recovers from the persisted row,
  not from an already-serialized manager), and concurrent same-anchor
  allocation.

## 6. Open-by-upgrade admission (the server half)

Reached from `request_arkoor_cosign`. Today that handler rejects every
`ChannelFunding` destination/input (the MR-1 gate) and passes
`ChannelAuthorization::None` into the builder (arkoor.rs:150-158, 58-60). This
MR adds the channel branch: a package part carrying **both** `channel_id` and
`bridge_pub_nonce` is an upgrade request; captaind runs the checks below and,
on success, passes `ChannelAuthorization::UpgradeOutput` into MR-1's builder
gate. Every check runs **before** the transfer cosign marks the input spent;
all admission refusals leave no *Ark* state mutation (the LDK pending channel
from step 1 already exists — see §6b — so "no state mutation" is scoped to Ark
side).

### 6a. Ordered admission

Mined from old `cosign_oor_upgrade`, re-mapped to spec IDs and the stage-1
floor. Checks 1–12 were validated against the spec by the review:

1. **At most one channel part** per package (OP-24); the part names exactly
   one `channel-funding` destination (OP-2 — the MR-1 builder already refuses
   isolated/multiple/split channel destinations structurally; captaind
   re-checks the request-level count for a clean error).
2. **Inputs are `pubkey`** VTXOs (self-spend); IB-1/2 attestation +
   spendable/registered/not-banned/not-reserved via the existing generic
   validators (route-through, not reimplemented).
3. **IB-3 exiting-input refusal** — an input already exiting is refused at
   cosign (distinct from the late-registration refusal of §5); its own test.
4. **Self-spend binding** (OP-4): destination `user_pubkey` == input's user
   key.
5. **Checkpoint carve-out** (OP-3): single-part / single-`channel-funding`
   destination / no other outputs ⇒ `use_checkpoint = false` accepted; every
   other shape follows ordinary checkpoint policy.
6. **`channel_id` lookup** (OP-23, OP-26): the id names a channel the server
   is currently opening; a `channel-funding` destination with no `channel_id`
   is rejected; a `channel-funding` destination without its bridge in the same
   exchange is rejected.
7. **Not already funded / gate not already released** (idempotent re-cosign of
   the *same* backing VTXO excepted) — rides the spent-state atomic
   reservation (RG-14), not an independent lock.
8. **Negotiated-amount equality** (OP-5, BR-4): the reconstructed
   `channel-funding` VTXO's amount == the channel's negotiated funding amount;
   "no such open in progress" is a hard refusal.
9. **Pinned parameters** (OP-16, BR-3/8): the reconstructed VTXO's decoded
   `exit_delta` == the server's `pinned_exit_delta` for this scope — fixed once
   at open from the input's decoded `exit_delta`, **stored, never re-read from a
   live ark-info value**. `channel_max_vtxo_exit_depth` pinned from the
   published `max_vtxo_exit_depth` at open.
10. **Depth + split headroom** (DA-6/7, OP-14): resulting exit depth (input+1,
    or +2 through a checkpoint) ≤ `channel_max_vtxo_exit_depth`, and — a
    **first-release-profile MUST** (spec:1970-1975), not merely SHOULD —
    resulting depth ≤ `channel_max_vtxo_exit_depth − 2`, reserving a worst-case
    checkpointed downgrade split cosignable for the channel's whole life (no
    refresh remedy in stage 1). The review confirmed the `max−2` retention.
11. **Runway vs floor F** (OP-15, I-4/5): remaining runway
    (`expiry_height − real tip`) MUST exceed the server's computed
    **`F = channel_max_vtxo_exit_depth + pinned_exit_delta + cltv_claim_slack`**.
    The review confirmed the **`1× pinned_exit_delta`** term is correct and
    that the reference branch's second `exit_delta` belonged to the excluded
    HTLC-success CSV — it MUST NOT be carried into stage 1. Checked arithmetic
    + u16-representability guard (F feeds BOLT `cltv_expiry_delta` fields). The
    numeric `cltv_claim_slack` is D5 (§8).
12. **Bridge reconstruction + funding-outpoint cross-check** (BR-18, OP-25):
    reconstruct the bridge with `lib::channel::bridge_tx`; assert the two
    equalities — the LDK-assigned funding outpoint (from `ChannelPending`) ==
    the reconstructed `bridge_txid:0`, and the reconstructed bridge out0
    (value + script) == the persisted canonical funding output/redeem script
    (§2c). Either failure ⇒ MUST NOT cosign any part.

Then: the generic `cosign_oor_with_builder` (flux-lock, not-exited check,
persist unsigned virtual txs, sign the transfer) → `server_cosign_bridge`
(MR-1's one-shot MuSig2 partial over the bridge keyspend sighash) →
transition the channel to `cosigned` (§6b). BR-17: fresh nonce per session;
the byte-identical duplicate is re-signed with a fresh nonce (documented
stage-1 conformance deviation, spec:2007-2010 — no store-and-replay).

**Usability proof — cooperative client↔captaind payment (in-scope test).**
Once a channel is registered and released, the server-side e2e drives an
ordinary bidirectional HTLC exchange between captaind and the test client
(client→captaind and captaind→client), settled off-chain by preimage. This
proves the gate yields a *usable* channel — not merely an opened one — with
captaind as the terminal hop and **no forwarding, no on-chain action**. The
adversarial/force-close-with-pending-HTLC paths stay with the exit tests and
are framed as stock-behavior documentation, not stage-1 safety proofs
(that is the deferred CSV territory).

### 6b. The durable open state machine (review F4)

The prior draft said "incomplete Ark state at `ChannelPending` → force-close";
implemented literally that force-closes **every** valid open, because every
valid upgrade reaches LDK `ChannelPending` *before* the cosign — so at that
moment there legitimately is **no** Ark state yet (the reference branch's
commit `0595bb594` fixed exactly this). The subsystem tracks a durable state
machine keyed by channel:

```
opening(temp_channel_id)                     -- inbound accepted, pre-funding
  → awaiting_upgrade(channel_id, funding_txo, funding_redeem_script, final_type)
                                             -- ChannelPending seen; no Ark state yet — VALID
  → cosigned(backing_vtxo, watch_unarmed)    -- admission passed, bridge cosigned
  → registered(released)                     -- gate released (§5)
```

- **No Ark state at `awaiting_upgrade` is valid**, not a violation. Only
  *partial or contradictory* state (e.g. a `ChannelPending` whose declared
  type/outpoint disagrees with a persisted record) is fatal → force-close.
- The client-controlled failure mode (a `ChannelPending` reaching funding with
  inconsistent Ark state) maps to **`ForceClose`, never `Replay`** (review /
  §11.3) — a `Replay` would let a peer poison the shared LDK event queue.
  Server-local `Persist` failures map to `Replay`.
- **Pending-open resource control**: an `opening`/`awaiting_upgrade` channel
  consumes LDK + DB resources before any cosign, so the subsystem bounds
  concurrent pending opens per peer and times out/cleans up stale ones. (This
  is why admission refusals are "no *Ark* state mutation" — the LDK pending
  channel is real and is reaped by this machine, not by admission.)
- The gate (§5 release) MUST land in or before the first commit that accepts
  opens — decoding an open without the gate is the same unsafe intermediate
  the MR-1 arc rejected.

## 7. The parent-exit watch (WD-2..5)

An upgrade leaves the input VTXO's `delayed-sign(exit_delta, A)` leaf alive in
the old chain, so a user could exit the input and claw back capacity now
backing the channel — including balance the server has since earned. The
defense (spec:470-506): the registered checkpoint/arkoor transactions spend
the input at `nSequence = 0`, mineable the next block — `exit_delta − 1` ahead
of the leaf's claim. captaind must **retain** the new level(s)' signed
transactions and **watch** for any prefix of the input's exit chain
confirming, and on seeing it **broadcast** the retained transaction(s),
P2A-CPFP-bumped, ahead of the input's `exit_delta` window.

Mechanism, mined from the old `ForfeitWatcher` parent-watch, pared to the
upgrade kind and sharpened per review F7:

- **Arming.** The watch row (`channel_parent_watch`, §10) is written at
  cosign but **armed only at registration**, in the same linearized
  transaction as the gate release (§5) — a cosigned-but-unregistered upgrade
  never operated, so an unarmed watch needs no response.
- **Resolution predicates (precise).** An **unarmed** watch does **not**
  resolve on a mere prefix of the input's exit chain confirming; it resolves —
  and makes any later registration terminal (a late-registration refusal, §5)
  — only on (i) the input's **final** exit tx confirming (the input output is
  actualized, its `exit_delta` clock started), or (ii) a confirmed **ancestor
  expiry sweep** foreclosing the input's creation. An **armed** watch fires
  its retained response when any prefix confirms, then drives it to
  confirmation.
- **Durable resolution record.** Persist `resolution_reason` (responded /
  input-final-confirmed / ancestor-swept), the spending txid, and the block
  hash + height — `resolved_at` alone cannot identify or unwind a reorg.
  `on_reorg` **reopens** any chain-derived resolution above the fork point.
- **Watcher is not optional** (§2a): the watch runs on the always-on embedded
  watcher; channels-enabled implies watch-running.
- **Terminal cleanup (gap the reference leaked).** The old branch left a dead
  row per settled channel (`TODO(ark8)`); MR-3 implements the ancestor-sweep
  resolution so rows are collected, not leaked.

Detect/respond/progress run in the `ChainEventListener`, sharing the
offboard-forfeit watcher's TxIndex / nursery / wallet. The response
actualizes only the channel-VTXO (or checkpoint parent) level; the bridge and
commitment are untouched (WD-4).

## 8. Expiry, the config guard, and the retained decisions

- **Expiry sweep (EX-1/7)**: the `timelock-sign(expiry_height, S)` leaf is the
  server's recourse when the user actualized the channel VTXO output but left
  the channel unresolved; the sweep takes the **whole** channel (single
  `musig(A,S)` output, expiry leaf server's alone). A watchman **policy arm** —
  the MR-1 forced-match already routes a `channel-funding` VTXO to
  `decide_action_expiry`; this MR makes that arm live for tracked VTXOs. It is
  the server's *only* self-initiated on-chain act (WD-16).
- **Config-decrease guard (I-9)**: the server MUST refuse a `max_vtxo_exit_depth`
  config decrease that would strand a live channel past downgrade eligibility
  (or require closure first), and MUST enforce `max_vtxo_exit_depth ≥ 2`.
  Checked at config-load and across restart. This is the one DA-8..10 piece
  this MR owns; the downgrade eligibility check is part 5.
- **D2 (no server-side force-close scheduler) — stands, and is now safe for
  this MR.** With forwarding out, captaind never holds a "paid-outbound /
  stuck-inbound" obligation (§3), and cooperative payments never go on-chain.
  Its defenses — the parent-exit watch and the expiry sweep — plus the client's
  own exit discipline cover every in-scope case. (The forwarder's I-6d
  scheduler is part 4.)
- **D3 (keep the close outcome, not the bridge) — stands for this MR.** Nothing
  in MR-3 requires captaind to self-actualize a channel: the client actualizes
  on unilateral exit (it holds the bridge), and captaind's expiry sweep uses
  the expiry leaf, not the bridge. Mandatory bridge retention arrives with the
  part-4 forwarding layer, where the force-close-to-recover path needs it.
- **D5 — `cltv_claim_slack`, the runway-floor slack.** Still MR-3's (the runway
  admission, check 11, is at open regardless of payments). `F` is computed from
  captaind's own view, independent of the client's. The review proposed a
  depth-derived default; before fixing the number the **confirmation model**
  must be settled — specifically whether the pre-signed exit chain rides to the
  commitment as a single TRUC/CPFP package (in which case the slack does not
  scale with depth × per-tx confirmations) or serially (in which case it does).
  Proposal: pick an operational confirmation target `C` (≈ the conf/claim
  margin the offboard/forfeit paths already assume), define
  `cltv_claim_slack` from the *package* model as `C + processing_grace`
  (a small constant, since depth is already the `channel_max_vtxo_exit_depth`
  term of `F` and the chain confirms as a package), and record the alternative
  serial-model figure (codex's `(C−1)·D + 3C + 3`, ≈1757 at `D=100`) as the
  conservative upper bound if the package assumption is wrong. **Resolve with a
  concrete integer + the model justification before code-start**; F must remain
  safe for the eventual part-4 HTLC path, so a pending-HTLC second-stage term
  is included even though MR-3 itself settles cooperatively.

## 9. Config surface

`[channels]` (`OptionalService`):

```toml
[channels]
enabled = false
listen_address = "0.0.0.0:9735"
# runway-floor slack (D5); the exact default is set at code-start
cltv_claim_slack = <TBD, §8>
```

No forwarding knobs (forwarding is part 4). Channel-type negotiation, the
zero reserve (LDK clamps to ≥1000 sat regardless), dust exposure, and
`accept_forwards_to_priv_channels = false` stay hardcoded in the node's
`UserConfig` builder. `channels.validate()` runs in `Config::validate()`
unconditionally (a disabled service validates trivially). The
`captaind.default.toml` artifact gains the section (complete + valid so the
roundtrip config tests stay green); no test-only fault-injection knobs in
production config.

## 10. Database schema

New migration(s) starting at the next free `V55` (assigned at code-start
against the rebased tree). Teleport tables (old V31/V32/V33/V36
`pending_teleport*`) are **not** ported.

| Table / column | Purpose | Old origin |
|---|---|---|
| `ldk_channel_monitors` | LDK monitor blobs | V30 |
| `ldk_channel_manager` | singleton manager blob (via the `KVStore` adapter) | V30 |
| `channel_state(channel_id PK, funding_redeem_script, funding_txo, pinned_exit_delta, pinned_max_vtxo_exit_depth NOT NULL, final_channel_type, backing_vtxo, open_state, backing_registered_at)` | per-channel Ark state, the open state machine (§6b), the gate marker; funding keys are LDK-derived so the canonical redeem script/outpoint are persisted (§2c) | V30+V34+V36+V38, merged; `funding_redeem_script`/`open_state` new |
| `channel_scid(channel_id PK, anchor_hash, height, tx_index, collision_bump, UNIQUE(height, tx_index, vout))` | synthetic SCID allocation, persisted before feeding, unique full-SCID (§5) | new (fixes the reference's hardcoded index) |
| `channel_parent_watch(input_vtxo_id PK, channel_id, kind CHECK('upgrade'), response_txids TEXT[], armed_at, resolution_reason, spending_txid, block_hash, block_height, resolved_at)` | parent-exit watch (§7); `armed_at` = armed, rich resolution for reorg unwind | V39/V40 (upgrade kind only; part 5 adds 'downgrade') |

`pinned_*` land NOT NULL from birth (fresh table, no backfill migration).
`ldk_virtual_fundings` (old V30) is **not** carried unless commit-1 finds the
`KeysManager` persistence needs it (review F11 says it does not). Schema
regeneration: `just dump-server-sql-schema` → `server/schema.sql`.

## 11. Hardening not to regress (from the reference branch)

All eight carry (none is teleport-only; the teleport-specific fixes are
excluded with teleport):

1. **Concurrent-open deadlock** — dedicated `db_executor` runtime+pool (§2c);
   regression test: two concurrent opens.
2. **Event-queue poison via cooperative-close of an untracked channel** —
   record a close only when it has Ark backing.
3. **DoS via client-controllable event replay** — a `ChannelPending` reaching
   funding with inconsistent Ark state maps to `ForceClose`, never `Replay`
   (§6b); server-local `Persist` failures map to `Replay`.
4. **Capture broadcaster skips `CooperativeClose`** — that tx spends an
   off-chain-only bridge output and can never relay; the outcome is recorded
   from the event.
5. **Fail-fast on monitor/manager deser** — panic rather than continue with a
   fresh manager that drops justice monitors.
6. **SweepWalletSource trusts own unconfirmed change** — so HTLC bump
   fee-funding is not starved by a lagging BDK view.
7. **SCID non-collision derivation applied server-side** (§5) — the old
   server's `feed_tx_confirmation` hardcoded `tx_index = 0`; derive from the
   bridge txid, persist, assert collision-fatality.
8. **Channel-type hard gate** — accept only the designated stock
   `anchors_zero_fee_commitments` type; refuse static-remote-key-only, legacy
   anchors, empty feature sets. (The `ark_channel` bit + its CSV are stage 2.)

Added by the review (F8):
9. **Fee-bump reserve admission** — a documented fee-bump reserve policy
   checked *before* `accept_inbound_channel`, so the node can always fund a
   `BumpTransactionEvent` CPFP; and the **Core v29+/TRUC** relay requirement is
   stated (the P2A/zero-fee package path needs it).
10. **Foreign-input-safe bump PSBT signing** (reference `6edd047f7`) — a mixed
    bump PSBT signs only wallet-owned inputs, leaving the LDK channel input for
    the channel signer.
11. **`ChannelCloseMinimum` pinned to the relay floor** (reference
    `6c954e14f`) — so a cooperative-close fee negotiation cannot wedge (the
    same "unfailable close" property as the downgrade PR).

## 12. Requirement → mechanism discharge table

*(S = reused upstream machinery; T = new test; D = decision here. "part N" =
deferred by design.)*

| ID(s) | Mechanism in this MR |
|---|---|
| IB-1..7 | Upgrade routes through the generic ARK #5 validators (S); IB-3 exiting-input refusal test (T) |
| PV-6 | authorized **spend** is `DowngradeInput` (part 5); this MR's channel path is `UpgradeOutput` *creation* (§6) |
| PV-8 | `supports_channels` = `channels.enabled()` (§2a) |
| PV-10 | suite green subsystem-off and -on for non-channel flows (T) |
| BR-3/4/8 | pinned_exit_delta stored-once/never-reread; negotiated-amount equality (§6.8-9) |
| BR-12/13 | keep close outcome, not bridge — bridge retention is part 4 (D3) |
| BR-14 | LDK-derived per-channel funding keys, persisted redeem script; distinct from A/S (§2c) |
| BR-15/16/18 | one-shot bridge cosign; commitment via stock 2-of-2; reconstruct + equalities (§6.12) |
| BR-17 | fresh nonce/session; duplicate re-signed (stage-1 nit, §6) |
| OP-1..7 | server view of the transfer-shape checks (§6.1-5) |
| OP-14..16 | depth / runway / pinned params, server half (§6.9-11) |
| OP-18..22 | virtual-confirmation chain integrity; force-close on unconfirmation (profile); fail-closed restart (§5) |
| OP-23..26 | channel_id lookup, at-most-one-part, the two equalities (§6.1,6,12) |
| OP-27 | client retry nonce discard — part 4 |
| OP-28/29 | tampered channel_id caught at partial-sig verify; attestation binding parked |
| RG-1..5 | complete-set registration, all-or-nothing, idempotent, crash-survival (§5) |
| RG-6..8 | the linearized gate: no ChannelReady before committed registration; late-registration refusal (§5) |
| RG-14..16 | upgrade rides the single atomic spent-state reservation (§6.7); concurrent test (T) |
| WD-2..5 | parent-exit watch: retain, watch, respond, precise resolution (§7) |
| WD-15/16 | gate + watch + guard crash-survival; sweep is the only self-initiated broadcast (§7,8) |
| EX-1/7 | watchman expiry arm sweeps the whole channel VTXO (§8) |
| DA-1..7 | published-bound refusal (S); resulting-depth + `max−2` split headroom (§6.10) |
| DA-8..10 | config-decrease guard only (§8); eligibility check is part 5 |
| I-4/5 | floor F at runway admission — spec `1×` formula (§6.11) |
| I-6 | forwarding floor + received/invoice/force-close floors — **part 4** (captaind's own receive floor via invoice min_final_cltv is §3) |
| I-9 | config-decrease guard, `≥ 2`, across restart (§8) |
| I-10 | virtual-confirmation fail-closed + persisted bridge-txid SCID synthesis, scid_privacy (§5) |

## 13. Commit plan (four stages)

Each builds workspace-wide; per-commit `cargo check --all --tests`; TDD;
upstream conventions (tabs, `anyhow` in captaind, changelog keyed to the MR
number, `contrib/agents/agents.md`, `protocol-encoding`/`database-schema`
skills).

1. **Production node in `bark-channels`** — promote the harness: real capture
   broadcaster over the tx nursery, postgres monitor `Persist` + the dedicated
   `db_executor`, the singleton-manager `KVStore`, `process_events_async`, the
   `BumpTransaction` fee-bump path (with the reserve policy + foreign-input-safe
   signing), the restart lifecycle. The crate's own tests prove the node; no
   captaind wiring yet.
2. **Subsystem scaffold** — `[channels]` `OptionalService`, `channels/` module
   skeleton, the new migrations + schema.sql, the DB executor blocks, the
   always-on embedded watcher wiring, the peer listener gated on chain catch-up,
   `supports_channels` tracking enablement. Inert: accepts no opens yet.
3. **Open-by-upgrade admission + the open state machine** — the §6 checks,
   the §6b durable state machine, `UpgradeOutput` into the builder, bridge
   reconstruct + cosign + the two equalities, channel-state persist. Channel
   opens but is not usable until the gate releases (stage 4).
4. **The linearized gate + watch + expiry + guard** — confirmation-injection
   release via the idempotent outbox on committed registration, the SCID
   allocator, the parent-exit watch (arm/detect/respond/resolve incl. the
   ancestor-sweep terminal condition + reorg reopen), the watchman expiry arm,
   the config-decrease guard.

Server-side e2e (real captaind + a test driver for the part-4 client) proves:
open-by-upgrade happy path; each admission refusal (runway, depth/headroom,
amount, pinned-delta, exiting-input, unknown channel_id, missing bridge);
not-ready-before-registration; the linearized late-registration refusal;
**cooperative bidirectional client↔captaind payment** (usability, §6a);
parent-exit watch arm + response + reorg reopen; expiry sweep; config-decrease
guard across restart; concurrent opens (the deadlock regression);
crash-before-first-feed SCID recovery. Mine the old branch's `barkd_*`
scenario names for coverage parity, minus forwarding/teleport/downgrade.

## 14. Decisions (resolved) and residuals

- **D1 (event driver)** — RESOLVED: async `lightning-background-processor`
  0.2.3 `process_events_async` + a small postgres `KVStore` for the singleton
  manager; custom monitor `Persist` retained (§2c).
- **D2 (no server force-close scheduler)** — STANDS, now safe: forwarding is
  out, so captaind carries no forwarder obligation; defenses are the watch +
  expiry sweep (§8).
- **D3 (keep close outcome, not bridge)** — STANDS for this MR; bridge
  retention moves to part-4 forwarding (§8).
- **Forwarding** — REMOVED from MR-3 (was the prior D4); restored to part 4
  with its safety layer (static forwarding-CLTV ceiling, mandatory bridge
  retention, I-6d force-close scheduler). captaind ships `accept_forwards…
  = false`.
- **Funding keys** — RESOLVED: LDK-derived per-channel, not `S` (§2c, BR-14).
- **Watcher availability** — RESOLVED: always-on embedded watcher; not gated on
  the optional `watchman` service (§2a/§7).
- **SCID privacy** — RESOLVED: `negotiate_scid_privacy = true`, persisted
  allocation, collision-fatal (§5).
- **D5 (`cltv_claim_slack` default)** — RESIDUAL, must resolve before
  code-start: settle the confirmation model (package vs serial), then fix a
  concrete integer (§8). Package model → small constant; serial model →
  codex's ≈1757 upper bound.
- **Boundary** — the plan's §7 owner table predates the MR-1 shape-bounding
  fold and this forwarding move; this note treats OP-2's shape as done in MR-1
  and I-6 forwarding as part 4. Recorded so scope isn't re-litigated.
