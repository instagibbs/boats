# MR-3 design note (G1): captaind as the channel counterparty — the embedded-LDK channels subsystem

**Status**: G1 rework r3 (codex rounds 1–3, all REWORK → this revision,
2026-08-04; r3 findings were precision/completeness, no scope change). r1 pulled forwarding out; r2/r3 established the deeper invariant
(WD-16 → §3): **captaind never initiates a bridge unroll** (a no-retention
profile choice, and it holds only a bridge partial), so it cannot defend a
preimage race and therefore **originates and claims no production payments** in
stage 1 — all payment handling (endpoint *and* forwarding) is part 4 / stage 2. `F` is the
exit-*possibility* floor, not an HTLC-race defense, and is *smaller* than the
Ark-channel variant (no success-CSV term). All other r1/r2 findings (gate
linearization, funding keys, SCID, watch, state machine, hardening, async
processor) applied. Codex records:
`docs/plans/2026-08-04-codex-mr3-g1-review.md` (r1),
`…-mr3-g1-rereview.md` (r2, to be saved).
**Series position**: part 3 of 6 upstream (internal "MR-2 captaind"), stacking
on the protocol-surface MR (!2336) → the release-contract opener (!2321).
**Baseline**: bark-stage1 `ark8-channels-stage1-protocol` (rebase to tip at
code-start). **Theme (plan:370)**: *captaind can be the channel counterparty.*
**Spec targets**: IB-1..7 (upgrade path), PV-6/PV-8/PV-10 (server halves),
BR-3/4/8/12/13/14/15/16/17/18, OP-1..7/14..16/18..28 (server view),
RG-1..8/14..16, WD-2..5/15/16, EX-1/7, DA-1..10 (server + config guard),
I-4/5/9/10 — full discharge table in §12. (I-6 HTLC floors + all payment
handling: part 4 / stage 2, §3.)

## 1. Scope: what this MR is, and the stage-1 cut it is drawn against

captaind gains an **optional** embedded Lightning node so it can be the
counterparty of a channel whose funding output lives, unbroadcast, in a
VTXO's exit chain. Concretely: accept an inbound Ark channel on stock
released LDK, admit an **open-by-upgrade** riding the arkoor cosign, hold
`ChannelReady` until the Ark transfer is durably registered, cosign the
presigned bridge, stand the parent-exit watch that defends the balance after
the input is upgraded away, and sweep the channel VTXO at expiry. The channel
reaches operational state between the client and captaind.

**MR-3 originates and claims no production payments (§3).** captaind never
initiates a bridge unroll (WD-16 profile + it holds only a bridge partial), so
it can never force-close early to defend a preimage-vs-timeout race — meaning
it must never hold a preimage under deadline. So it **never claims an inbound
HTLC and never originates one**: no send/invoice API, never `claim_funds`,
`fail_htlc_backwards` on every `PaymentClaimable`, `push_msat == 0` + no
dual-funded opens, `accept_forwards_to_priv_channels = false`. (Stock LDK can
still momentarily *commit* an inbound HTLC before exposing it — absolute
pre-commit rejection needs a fork hook — but failing every claimable back is
sufficient, since exposure only arises from claiming.) Safe server HTLC
handling — the stage-2 success-CSV and/or directional policy + caps, **not**
server unroll machinery — is the **part-4 / stage-2** layer. So the channel
opens, is defended, expires, and exits, carrying no money in flight. A
cooperative HTLC round-trip may run in the *test harness* as a mechanical
operability check (§6a), never as a shipped capability. This is not a narrowing
of the plan — payments were always part 4.

**In scope**: the `[channels]` `OptionalService` subsystem; LDK node
assembly + persistence + event loop; the open-by-upgrade admission path (the
server half of the two-sided checks); the registration gate; the bridge
cosign integration; the parent-exit watch; the expiry sweep arm; the
`max_vtxo_exit_depth` config-decrease guard; `supports_channels` advertised
true when enabled.

**Out of scope**, and the note keeps clear of each:
- **All HTLCs / payments (endpoint receive, endpoint send, forwarding)** —
  part 4 / stage 2, with the HTLC-safety layer (§3). captaind's LDK
  `UserConfig` sets `accept_forwards_to_priv_channels = false` and payments are
  not enabled; the node opens and defends channels but moves no HTLCs in
  production.
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
  an `ark_channel` feature bit (there is none). The exact role of the success
  CSV in the on-chain HTLC race is stage-2 material and not restated here (the
  note deliberately does not assert its mechanism); the load-bearing fact for
  MR-3 is only that **safe server HTLC handling is not available on stock LDK**
  without either that fork-provided binding or directional operator policy +
  caps — which is why **all** HTLC handling is deferred to part 4 / stage 2
  (§3), and may genuinely require the fork rather than policy alone.

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
| `channels/confirm.rs` | virtual-confirmation feed + SCID allocator (§5), the level-triggered release reconciler | old `feed_tx_confirmation` (with the collision bug fixed) |
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
  was dead at branch tip — its signer discarded it.) **The canonical funding
  `TxOut` is assembled from two LDK events, since neither alone carries it**
  (review F5): `OpenChannelRequest` supplies `funding_satoshis` (+ the
  temporary channel id, counterparty, and proposed type — persisted at
  acceptance, before the accept call), and `ChannelPending` supplies the
  funding *outpoint* and the final channel type/`funding_redeem_script`. The
  server derives `funding_txout = (funding_satoshis, P2WSH(funding_redeem_
  script))` and admission compares the bridge's out0 against *that* (§6, check
  12) — it never reconstructs the funding keys itself.
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

## 3. The two-sided invariant, WD-16, and why MR-3 originates/claims no payments

Every open-time check in the spec is enforced **from each party's own view**
("the client MUST refuse to build, and the server MUST refuse to cosign",
spec:438-440). This MR is the **server** half; the client half is part 4.
Neither half trusts the other. Both sides pass `ChannelAuthorization::
UpgradeOutput` into the MR-1 builder — the client when it *builds* the
transfer, captaind when it re-derives from the cosign request.

**The load-bearing profile choice: captaind never initiates a bridge unroll.**
The spec permits a server to retain the completed bridge and force-close
through it (BR-12/13 — a **MAY**, spec ~283/close region); the matrix's WD-16
records that a *conforming server need not*, and MR-3 adopts the
**no-retention profile**: captaind never broadcasts a bridge or otherwise goes
*below* the channel VTXO output on its own initiative. This is a profile
decision, not an absolute law — but under the chosen open cosign exchange it is
also structurally enforced for the bridge specifically: the bridge is a
one-shot MuSig2 keyspend and captaind holds only a *partial*; the client
aggregates and keeps the completed bridge (spec ordering step 6, spec:410), so
captaind *cannot* broadcast it. captaind's on-chain acts are therefore all **at
or above the channel VTXO output**, never a bridge unroll:
1. **ChannelMonitor claims from the counterparty's own force-close** — if the
   client goes on-chain, the *client* unrolls (it has the bridge) and captaind's
   monitor claims its output from the landed commitment;
2. the **whole-output expiry sweep** via the `timelock-sign(expiry, S)` leaf on
   the channel VTXO output — the backstop;
3. the **parent-exit response** (§7) — reactively actualizing the retained
   *transfer* transactions up to the channel VTXO output. (This is above the
   channel VTXO output, so it is not a bridge unroll — it is why the invariant
   is stated as "never *initiates a bridge* unroll," not "never broadcasts.")

**Why this means MR-3 originates and claims no payments.** Because captaind
never initiates a bridge unroll, it can never force-close *early* to win a
preimage-vs-timeout race — the standard-Lightning defense for a party that must
enforce a preimage claim on-chain against a counterparty's timeout. The danger
is not *knowing* a preimage (a keysend `PaymentClaimable` even hands captaind
the sender's preimage) but **claiming/fulfilling** with it — that is what puts
captaind in the race it can't win. So captaind **never claims/fulfills an
inbound HTLC and never originates one.** Making the CLTV floor bigger does not
help — it only moves the deadline a staller waits for. Safe server HTLC
participation is deferred wholesale to **part 4 / stage 2**, whose tools are the
stage-2 success-CSV and/or directional operator policy + bounded caps — **not**
server unroll machinery (which the no-retention profile declines). MR-3
therefore enforces, on stock LDK:
- **no origination**: expose no send or invoice-generation API; never call
  `send_payment`/`create_inbound_payment`;
- **no claim**: never call `claim_funds`; on **every** `PaymentClaimable`
  (including replay), immediately `fail_htlc_backwards` — captaind never reveals
  a preimage, so it is never in the race;
- **no server balance at open**: require `push_msat == 0` and reject
  dual-funded (v2) opens, so the "server balance is zero" premise (spec:484)
  holds and nothing moves at open;
- forwarding stays off (`accept_forwards_to_priv_channels = false`).

The honest guarantee is **"captaind originates and claims no production
payments"** — *not* "no HTLC is ever momentarily committed": stock LDK commits
an inbound final-hop HTLC before exposing it, and always advertises keysend, so
an inbound HTLC can exist for the interval before captaind fails it back.
Absolute pre-commit rejection would need an LDK hook (a fork), which stage 1
does not have; failing every claimable back is the enforceable stock-LDK
equivalent and is sufficient, because captaind's exposure only ever arises from
*claiming*. The channel opens, is defended, expires, and exits — economically
inert. This is not a narrowing of the plan; payments were always part 4.

The MR-3 operability proof is that a released channel reflects usable
capacity (`ChannelReady` exchanged, `next_outbound_htlc_limit_msat` non-zero,
the peer sees it operational). A cooperative HTLC round-trip **may** also run
in the test harness — but stated precisely: a cooperative settle *does* update
and sign new commitments and HTLC second-stage transactions; "clean" means
none is **broadcast** and no CSV path executes, and the test must wait through
the final `revoke_and_ack` (the harness warns the final revoke can lag
`PaymentClaimed`). It is a mechanical operability check on a controlled
cooperative counterparty — explicitly **not** a shipped capability and **not**
a safety claim (and it necessarily drives the very `claim_funds`/`send` paths
production disables, so it is a test-only seam).

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
  and delivery is **level-triggered, not a one-shot outbox**: the release is a
  durable *state* on the `channel_state` row (registered + a
  `confirmation_fed` latch), and a reconciler re-derives the unfed rows and
  feeds them idempotently. The original round-2 formulation gated the latch
  on a manager-persistence barrier ("set only after the fed state is on
  disk"); implementation found every such barrier unsound against the
  driver's encode timing (`process_events_async` encodes before it calls the
  store, so no write-side acknowledgement can prove the snapshot contains
  the feed). The shipped design needs no ordering proof at all:
  `confirmation_fed` is a **per-process latch**, and boot clears every latch
  and re-feeds all registered channels against whatever manager snapshot
  reloaded — safe because a re-feed of the same confirmation under the same
  header is idempotent in LDK, and a moved anchor is withdrawn by the
  catch-up's stale-confirmation pass before the reconciler runs. A crash at
  any point leaves the DB state authoritative and the reconciler converges —
  the channel is never stranded registered-but-operating-with-an-unfed-
  manager. Registration itself commits only after a cursor handoff: the
  transaction requires the block listener to have indexed the exact block
  (height AND hash) the chain-side screens observed as tip, so every block
  the screens could have missed has its facts recorded before the release
  can commit.
- **Reorg / chain-generation gate.** The release reconciler and the
  `on_reorg` handler are serialized on a single chain-generation guard, and
  the reconciler **revalidates the anchor against the current best chain
  immediately before feeding**: LDK forbids `transactions_confirmed` for a
  header not in the best chain as of `best_block_updated`
  (`chain/mod.rs:161`), so a registration committed just before its anchor is
  reorged out must **not** feed the now-stale anchor — the reorg invalidates
  the pending feed instead, and the release waits for a re-established anchor
  (or fails closed if none returns, OP-22). Startup catches the node up to the
  *real* chain first, then runs the reconciler.
- **Late-registration refusal (spec:489-494)**: a registration arriving after
  the input's exit-chain final tx has confirmed — or after a confirmed
  ancestor expiry sweep forecloses the input — MUST be refused, not completed.
  Both are recorded as the watch's terminal resolution (§7); the linearized
  check reads that record under the same lock, and it is THE authority. The
  chain-side screens (the final-exit look and the exit-chain viability walk)
  run BEFORE the transaction — RPC walks must never hold database locks —
  and anything they miss that confirms afterwards lands as a recorded fact
  the in-transaction re-read refuses on; a final exit confirming inside the
  residual cursor-lag window degrades to the ordinary armed-watch race the
  response answers. This chain→fact→refusal ladder is sound only while the
  listener sees every block a live channel could care about, hence the
  CONVERSE of §2a's invariant: **captaind refuses to start with `[channels]`
  disabled while any cosigned/registered/terminal row exists** — a disabled
  subsystem would blind the monitors, the watch and the confirmation filter,
  and the shared cursor would skip blocks the watch can never revisit. The admission at open
  also **rejects `push_msat ≠ 0` and dual-funded (v2) opens** (§3), so no value
  can move server-ward at open.

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
`tx_index` derived from the **bridge txid**, and the funding vout (fixed at 0).
The `tx_index` MUST be allocated from the band **`2500..2²⁴`** — LDK's own SCID
*aliases* occupy `0..2499` (`util/scid_utils.rs`), so keeping synthetic indices
above that band is what prevents a *future* LDK-generated alias from colliding
with an earlier synthetic SCID (the round-2/round-3 hole). Requirements:
- **Persist `{anchor_hash, height, tx_index, collision_bump}` transactionally
  *before* feeding**, with a **`UNIQUE(height, tx_index)`** constraint in the
  schema (§10; vout is fixed 0) and checked wrapping arithmetic on the bump
  within the `2500..2²⁴` band.
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
11. **Runway vs the exit-possibility floor F** (OP-15, I-4/5): remaining runway
    (`expiry_height − real tip`) MUST exceed the server's computed
    **`F = channel_max_vtxo_exit_depth + pinned_exit_delta + cltv_claim_slack`**.
    F is the **exit-possibility floor**, not an HTLC-race defense (§3, §8): it
    guarantees the channel can be unrolled/exited before expiry at all — the
    exit-chain climb (up to `channel_max_vtxo_exit_depth` genesis levels, ~1
    block each) plus the bridge's `pinned_exit_delta` CSV must fit inside the
    runway. The **`1× pinned_exit_delta`** term is correct: the reference
    branch's second `exit_delta` was the HTLC-success CSV, excluded in stage 1,
    so **stage-1 F is smaller than the Ark-channel-variant floor by exactly one
    `pinned_exit_delta`** (the variant is `depth + 2·pinned_exit_delta + slack`,
    spec:1291). Checked arithmetic + u16-representability guard (F feeds BOLT
    `cltv_expiry_delta` fields). The numeric `cltv_claim_slack` is a small fixed
    confirmation/fee-bump margin (D5, §8), NOT a depth-scaled value.
12. **Bridge reconstruction + funding-outpoint cross-check** (BR-18, OP-25):
    reconstruct the bridge with `lib::channel::bridge_tx`; assert the two
    equalities — the LDK-assigned funding outpoint (from `ChannelPending`) ==
    the reconstructed `bridge_txid:0`, and the reconstructed bridge out0
    (value + script) == the persisted canonical funding output/redeem script
    (§2c). Either failure ⇒ MUST NOT cosign any part.

Then everything that persists commits in **one DB transaction** (flux-lock
and not-exited check first, as on the generic endpoint), with the channel
row locked (`SELECT … FOR UPDATE`) and the input rows re-read fresh and
locked under it: the in-transaction state decision (awaiting → fresh;
cosigned with the identical backing → idempotent retry; anything else
refuses and rolls the whole exchange back untouched), the fresh-view
spendability re-check (a ban or spend that landed during admission refuses
here, before anything is marked), the spent-marking/unsigned-virtual-tx
persist, the `cosigned` transition and the (unarmed) watch row (§6b). The
row lock is what serializes admission against a concurrent close or reap of
the same channel: whichever commits first wins, the other refuses with no
Ark mutation — so the spent-marking can never outlive the channel row it
was admitted against. Only after that commit: sign the transfer
(`server_cosign`) and the bridge (`server_cosign_bridge` — MR-1's one-shot
MuSig2 partial over the bridge keyspend sighash, against the
**reconstructed** channel VTXO's user key, like every other admitted
value). BR-17: fresh nonce per session; the byte-identical duplicate is
re-signed with a fresh nonce (documented stage-1 conformance deviation,
spec:2007-2010 — no store-and-replay).

**Operability check — an HTLC probe (test harness only).** Once a channel is
registered and released, the harness drives a one-hop HTLC probe from the
test client: the HTLC crosses, commits on both sides, and — because
production claims nothing, ever (`PaymentClaimable` is failed back on
sight, and a random-hash probe fails even earlier inside stock LDK) — is
failed back after the complete commitment dance, leaving the channel
usable. This deliberately implements the check WITHOUT any test seam into
the never-claim policy: the earlier "cooperative settle by preimage"
formulation would have required driving `claim_funds`, which production
disables; a full add→commit→fail→commit round-trip is equal evidence that
the gate yields an *operational* channel (HTLC machinery moves), and it is
explicitly a **mechanical operability check, not a shipped capability and
not a safety claim**: production HTLC send/receive is not enabled in MR-3
(§3 — the server cannot defend a preimage-claim on-chain without unrolling,
which WD-16 forbids; that safety is the part-4/stage-2 layer). The
adversarial force-close-with-pending-HTLC behavior belongs to part 4.

### 6b. The durable open state machine (review F4)

The prior draft said "incomplete Ark state at `ChannelPending` → force-close";
implemented literally that force-closes **every** valid open, because every
valid upgrade reaches LDK `ChannelPending` *before* the cosign — so at that
moment there legitimately is **no** Ark state yet (the reference branch's
commit `0595bb594` fixed exactly this). The subsystem tracks a durable state
machine keyed by channel:

```
opening(temp_channel_id, counterparty, funding_satoshis, proposed_type)
                                             -- OpenChannelRequest: persisted
                                             --   BEFORE accept; push_msat==0 and
                                             --   non-dual-funded REQUIRED (§3)
  → awaiting_upgrade(channel_id, funding_txo, funding_redeem_script, final_type)
                                             -- ChannelPending: outpoint+script; no
                                             --   Ark backing yet — VALID
  → cosigned(backing_vtxo, watch_unarmed)    -- admission passed, bridge cosigned
  → registered(released)                     -- gate released (§5)
  → terminal                                 -- expiry/ancestor-sweep (§8)

opening | awaiting_upgrade → reaping         -- outstayed the pending bound: durably
  → (row deleted on ChannelClosed)           --   marked FIRST, LDK side closed, row
                                             --   deleted only when the close lands
```

- **`push_msat == 0` and non-dual-funded are checked at `OpenChannelRequest`,
  before `accept_inbound_channel`** (§3) — else value could move server-ward at
  open and the zero-server-balance premise (spec:484) breaks. The
  `funding_satoshis` and counterparty are persisted here (F5: `ChannelPending`
  does not carry them).
- **No Ark state at `awaiting_upgrade` is valid**, not a violation. Only
  *partial or contradictory* state (e.g. a `ChannelPending` whose declared
  type/outpoint disagrees with a persisted record) is fatal → force-close.
- The client-controlled failure mode (a `ChannelPending` reaching funding with
  inconsistent Ark state) maps to **`ForceClose`, never `Replay`** (review /
  §11.3) — a `Replay` would let a peer poison the shared LDK event queue.
  Server-local `Persist` failures map to `Replay`.
- **Crash-replay idempotence** (lightning 0.2.4 documents that a handled
  event whose removal did not persist re-delivers): an exact replay of
  `OpenChannelRequest` (an identical `opening` row already on record) and of
  `ChannelPending` (the identical promotion already applied — in
  `awaiting_upgrade` or ANY later state) is a no-op, not a contradiction;
  only **mismatched ingredients** force-close.
- **Pending-open resource control**: an `opening`/`awaiting_upgrade` channel
  consumes LDK + DB resources before any cosign, so the subsystem bounds
  concurrent pending opens per peer and times out/cleans up stale ones. (This
  is why admission refusals are "no *Ark* state mutation" — the LDK pending
  channel is real and is reaped by this machine, not by admission.)
- **Reaping is mark-then-close**: the reaper durably marks stale pre-cosign
  rows `reaping`, force-closes the LDK side, and the row is deleted only when
  the close lands (`ChannelClosed`) — no accepted LDK channel ever exists
  without its record. A close that does not land retries every block; a
  channel LDK does not know (`ChannelUnavailable`) has its orphaned row
  deleted directly. `reaping` rows still count against the peer's pending
  bound, and admission's row lock serializes against the mark.
- The gate (§5) splits into a **blocking half and a release half**, and the
  blocking half MUST land in the same commit that accepts opens: the chain
  feed withholds every COSIGNED channel's bridge txid from the node, so a
  real bridge confirmation cannot walk an unregistered channel to
  operational — and an unexpected `ChannelReady` fails closed (force-close).
  **Only authenticated txids enter the filter**: admission inserts the
  bridge txid its own reconstruction produced (before releasing the
  partial), and boot seeds from `cosigned` rows only. The funding txid a
  peer declares at `ChannelPending` is UNTRUSTED and must never be
  filtered — a peer could name an arbitrary transaction (another channel's
  commitment!) and blind the node's monitors to it. Pre-cosign, a decoy
  confirmation at the declared outpoint is answered by the fail-closed
  `ChannelReady`, not by filtering. The release half — feeding the
  confirmation on committed registration — is stage 4; decoding an open
  without the blocking half is the same unsafe intermediate the MR-1 arc
  rejected.

## 7. The parent-exit watch (WD-2..5)

An upgrade leaves the input VTXO's `delayed-sign(exit_delta, A)` leaf alive in
the old chain, so a malicious user could **unroll the input's own exit chain**
— actualizing the input VTXO output on-chain — and, `exit_delta` blocks later,
sweep that output directly, clawing back capacity now backing the channel
(including balance the server has since earned). The defense is a **forfeit**
(spec:470-506, the forfeit-watch pattern of ARK #4/#7 applied to an open): the
registered checkpoint/arkoor *transfer* transaction spends the input output by
key path at `nSequence = 0`, mineable the next block — `exit_delta − 1` ahead
of the input leaf's claim. So the trigger is precisely "the client has unrolled
the input to its VTXO output": captaind **retains** the transfer transaction(s)
and **watches** for any prefix of the input's exit chain confirming, and on
seeing it **broadcasts the retained forfeit/transfer tx**, P2A-CPFP-bumped,
landing the channel VTXO output (or its checkpoint parent) ahead of the input's
`exit_delta` window. This response is entirely **at or above the channel VTXO
output** — it never touches the bridge, which is why it is not a bridge unroll
(§3, WD-16).

Mechanism, mined from the old `ForfeitWatcher` parent-watch, pared to the
upgrade kind and sharpened per review F7:

- **Arming.** The watch row (`channel_parent_watch`, §10) is written at
  cosign but **armed only at registration**, in the same linearized
  transaction as the gate release (§5) — a cosigned-but-unregistered upgrade
  never operated, so an unarmed watch needs no response.
- **Reconcile at insertion and arming (review F7).** A prefix of the input's
  exit chain — or the final exit, or an ancestor sweep — may have **already
  confirmed** before the row is written or armed; the listener only sees
  *future* blocks. So both writing and arming the row MUST, under the same
  channel-state lock, reconcile against the authoritative TxIndex / current
  chain and immediately resolve-or-respond to anything already seen (an
  already-confirmed final exit at arming time is a late-registration refusal,
  §5; an already-confirmed prefix on an arming row fires the response at once).
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

The RESPONSE rides the generic watchman: a spent input whose exit confirms
has its registered spending transaction progressed on-chain, CPFP-bumped
from the watchman wallet — nothing channel-specific, which is exactly why
channels-enabled REQUIRES the embedded watchman (§2a, enforced at config
validation) and why the watchman wallet must be funded (the response CPFP
spends it). What is channel-specific is the BOOKKEEPING in the channels
`ChainEventListener`: the durable resolution record with its evidence, the
armed/unarmed predicates, the reorg reopen, and the foreclosure detection —
a confirmed FOREIGN spend of a live channel VTXO's own outpoint (anything
but the bridge, e.g. the expiry sweep) walks the row to `terminal`, as does
a transitive `ancestor_swept` resolution. The response actualizes only the
channel-VTXO (or checkpoint parent) level; the bridge and commitment are
untouched (WD-4).

## 8. Expiry, the config guard, and the permanent no-unroll invariants

- **Expiry sweep (EX-1/7)**: the `timelock-sign(expiry_height, S)` leaf is the
  server's recourse when the user actualized the channel VTXO output but left
  the channel unresolved; the sweep takes the **whole** channel (single
  `musig(A,S)` output, expiry leaf server's alone). A watchman **policy arm** —
  the MR-1 forced-match already routes a `channel-funding` VTXO to
  `decide_action_expiry`; this MR makes that arm live for tracked VTXOs. It is
  the server's *only* self-initiated on-chain act (WD-16).
- **The LDK terminal transition (review, new).** When an expiry sweep — or an
  ancestor sweep that forecloses the funding — makes the virtual funding
  permanently impossible, the LDK node still believes the channel is live and
  0.2.4 has **no public "abandon without broadcast" API**. So the subsystem
  must drive a durable **Ark-terminal transition**: mark the `channel_state`
  row terminal, force-close the channel in LDK (a *logical* force-close of
  LDK's own view — **not** an Ark bridge unroll), and **capture-and-suppress**
  the resulting commitment/HTLC broadcasts in the capture broadcaster (they
  spend an off-chain-only funding output and can never relay — the same
  suppression the cooperative-close path already uses, §11.4; and the
  `BumpTransactionEvent`s the logical force-close emits are likewise suppressed),
  persist the manager, and release the channel's watches and fee-bump reserve.
  Because `terminal_at` and the manager snapshot are separate durable writes,
  the terminal transition is **level-triggered like the release (§5)**: a
  reconciler re-drives the logical close until the restored manager no longer
  contains the channel. Covered by expiry-then-restart tests with crash
  injection before/after the force-close and before/after manager persistence.
- **Config-decrease guard (I-9)**: the server MUST refuse a `max_vtxo_exit_depth`
  config decrease that would strand a live channel past downgrade eligibility
  (or require closure first), and MUST enforce `max_vtxo_exit_depth ≥ 2`.
  Checked at config-load and across restart. This is the one DA-8..10 piece
  this MR owns; the downgrade eligibility check is part 5.
- **D2 (no server-side *bridge*-unroll force-close scheduler) — a PERMANENT
  profile invariant.** captaind never initiates a bridge unroll (§3): under the
  chosen cosign exchange it holds only a bridge *partial* and cannot broadcast
  the bridge, and MR-3 adopts the no-retention profile (declining the spec's
  BR-12/13 retention MAY). Its recourses are the monitor-claim from the
  counterparty's force-close and the expiry sweep; the *LDK-internal* force-close
  of the terminal transition (above) is a logical close that broadcasts nothing.
  No server-side force-close-*through-the-bridge* scheduler is built in MR-3,
  part 4, or ever — the earlier "part-4 I-6d scheduler" framing was a
  mis-derivation from a review finding that measured against standard Lightning;
  the Ark design
  rejects server unroll outright.
- **D3 (the server never retains/actualizes the bridge) — also PERMANENT.**
  Only the *client* holds the completed bridge and unrolls; the server's expiry
  sweep uses the expiry leaf on the channel VTXO output (above the bridge),
  never the bridge itself. captaind stores the close *outcome*, not the bridge.
  (The part-4 HTLC-safety layer does **not** reverse this — safe server HTLC
  participation comes from the stage-2 success-CSV and/or directional policy +
  caps, §3, not from teaching the server to unroll.)
- **D5 — `cltv_claim_slack`, and the size of `F`.** `F` is the exit-possibility
  floor (§6, check 11), computed from captaind's own view. The exit chain does
  not blow up the slack: each genesis level is a TRUC (v3) tx confirmed by its
  own P2A CPFP and relayed level-by-level as its parent confirms — **~1 block
  per level**, not one giant package (codex is right that the whole chain can't
  be one `submitpackage` — parents can't depend on each other — but that means
  it confirms *serially at one level per block*, which the `depth` term of `F`
  already captures; it does **not** mean each level must be buried to `C`
  confirmations). So `cltv_claim_slack` is a **small fixed margin** for
  confirmation variance + fee-bump rounds + processing grace — on the order of
  the conf/claim margin the offboard/forfeit paths already use (~10–20 blocks),
  **not** codex's depth-scaled ≈1757. Concretely at the defaults
  (`depth = 100`, `pinned_exit_delta = 144`): `F ≈ 100 + 144 + ~18 ≈ 262`, and —
  as expected — **less than the Ark-channel variant's `100 + 288 + ~18 ≈ 406`**
  by one `pinned_exit_delta`. There is **no** pending-HTLC second-stage term in
  stage-1 `F` (no success CSV, and no server HTLC-claim on-chain — WD-16);
  that term is exactly what the stage-2 variant adds. **Resolved:
  `cltv_claim_slack = 18`** (codex round-3 grounded this in BIP-431 / Core v29
  TRUC policy / `submitpackage` — a level needs one confirmation, not deeper
  burial — and LDK's `MAX_BLOCKS_FOR_CONF`; it is an operational next-block-CPFP
  margin, not a consensus guarantee under unbounded congestion). Checked
  arithmetic + u16 guard; confirm the constant against the offboard/forfeit
  margin at code-start.

## 9. Config surface

`[channels]` (`OptionalService`):

```toml
[channels]
enabled = false
listen_address = "0.0.0.0:9735"
# exit-possibility floor slack (§8): small next-block-CPFP margin
cltv_claim_slack = 18
# fee-bump reserve for the parent-exit response + expiry sweep (§11.9)
fee_bump_reserve_sat = <sized at code-start, §11.9>
fee_bump_max_feerate_sat_vb = <sized at code-start, §11.9>
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
| `channel_state(channel_id PK, temporary_channel_id, counterparty, funding_satoshis, funding_redeem_script, funding_txo, final_channel_type, pinned_exit_delta, pinned_max_vtxo_exit_depth NOT NULL, backing_vtxo, open_state, backing_registered_at, confirmation_fed, terminal_at)` | per-channel Ark state, the open state machine (§6b), the gate marker + the `confirmation_fed` per-process latch (§5), the terminal transition (§8); `funding_satoshis` from `OpenChannelRequest` + `funding_redeem_script` from `ChannelPending` derive the canonical funding `TxOut` (§2c, F5) | V30+V34+V36+V38, merged; `funding_satoshis`/`funding_redeem_script`/`open_state`/`confirmation_fed`/`terminal_at` new |
| `channel_scid(channel_id PK, anchor_hash, height, tx_index, collision_bump, UNIQUE(height, tx_index))` | synthetic SCID allocation from the `2500..2²⁴` band, persisted before feeding; vout is fixed 0 so `(height, tx_index)` is the full-SCID key (§5) | new (fixes the reference's hardcoded index) |
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
6. **SweepWalletSource trusts own unconfirmed change** — so P2A-CPFP
   fee-funding is not starved by a lagging BDK view.
7. **SCID non-collision derivation applied server-side** (§5) — the old
   server's `feed_tx_confirmation` hardcoded `tx_index = 0`; derive from the
   bridge txid, persist, assert collision-fatality.
8. **Channel-type hard gate** — accept only the designated stock
   `anchors_zero_fee_commitments` type; refuse static-remote-key-only, legacy
   anchors, empty feature sets. (The `ark_channel` bit + its CSV are stage 2.)

Added by the review (F8):
9. **Fee-bump reserve — quantified, and scoped to what MR-3 actually
   broadcasts.** MR-3 has **no HTLC-resolution** bumps (no HTLCs); the only
   server liabilities that need CPFP fee-funding are the **parent-exit
   response package(s)** (§7) and the **expiry sweep** (§8). The reserve policy
   admits an open only if the confirmed-wallet-UTXO reserve still covers, at a
   defined **maximum admitted feerate** and package weight, one worst-case
   parent-exit response plus one expiry sweep per live channel: reserve
   confirmed UTXOs per live channel at `accept_inbound_channel`, key them by the
   broadcast's `claim_id` so **rebumps of the same package reuse the same
   reserved UTXOs** (never double-spend the reserve). The parent-exit and expiry
   tranches release independently: the **expiry tranche is retained even after a
   parent-exit response confirms** (a channel whose input was forfeited can
   still reach expiry and need its sweep bumped), and each tranche releases only
   after its own package confirms or the channel reaches its terminal transition
   (§8). The **`BumpTransactionEvent`s emitted by the logical terminal
   force-close are suppressed**, not funded from the reserve (§8 — those
   broadcasts can never relay). The **Core v29+/TRUC** relay requirement is
   stated. Concrete reserve-sat and max-feerate numbers are sized at code-start
   from the package weights.
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
| BR-12/13 | keep close outcome, not bridge — the server never retains/actualizes the bridge, permanent (D3, WD-16) |
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
| I-6 | all per-HTLC floors (forwarding delta, received, invoice, force-close scheduling) — **part 4 / stage 2**; MR-3 originates/claims no payments (§3) |
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
   reconstruct + cosign + the two equalities, channel-state persist, and the
   gate's blocking half (the withheld-funding confirmation filter +
   fail-closed `ChannelReady`). Channel opens but is not usable until the
   gate's release half lands (stage 4).
4. **The linearized gate + watch + expiry + guard** — confirmation-injection
   release via the level-triggered reconciler on committed registration, the SCID
   allocator, the parent-exit watch (arm/detect/respond/resolve incl. the
   ancestor-sweep terminal condition + reorg reopen), the watchman expiry arm,
   the config-decrease guard.

Server-side e2e (real captaind + a test driver for the part-4 client) proves:
open-by-upgrade happy path; each admission refusal (runway, depth/headroom,
amount, pinned-delta, exiting-input, unknown channel_id, missing bridge);
not-ready-before-registration; the linearized late-registration refusal;
a **cooperative HTLC round-trip** as a mechanical operability check (§6a — not
a shipped capability); parent-exit watch arm + response + reorg reopen; expiry
sweep; config-decrease guard across restart; concurrent opens (the deadlock
regression); crash-before-first-feed SCID recovery. Mine the old branch's
`barkd_*` scenario names for coverage parity, minus forwarding/teleport/
downgrade and minus anything asserting adversarial HTLC safety (part 4).

## 14. Decisions (resolved) and residuals

- **D1 (event driver)** — RESOLVED: async `lightning-background-processor`
  0.2.3 `process_events_async` + a small postgres `KVStore` for the singleton
  manager; custom monitor `Persist` retained (§2c).
- **D2 (no bridge-unroll force-close scheduler)** — PERMANENT profile invariant:
  captaind never initiates a bridge unroll (WD-16 no-retention profile + holds
  only a bridge partial); recourses are monitor-claim + expiry sweep + the
  reactive parent-exit response (all at/above the channel VTXO output), plus the
  broadcast-nothing LDK-internal terminal close (§3, §8). No such scheduler is
  ever built.
- **D3 (server keeps the close outcome, not the bridge)** — PERMANENT profile
  invariant (§8): MR-3 declines the spec's BR-12/13 retention MAY; only the
  *client* holds the completed bridge and unrolls.
- **HTLCs / payments** — NOT in MR-3. captaind originates and claims none (no
  send/invoice API, never `claim_funds`, `fail_htlc_backwards` on every
  claimable, `push_msat==0`, no dual-fund, no forwarding); the HTLC-safety layer
  (stage-2 success-CSV and/or directional policy + caps — **not** server unroll
  machinery) is part 4 / stage 2 (§3).
- **Funding keys** — RESOLVED: LDK-derived per-channel, not `S` (§2c, BR-14).
- **Watcher availability** — RESOLVED: always-on embedded watcher; not gated on
  the optional `watchman` service (§2a/§7).
- **SCID privacy** — RESOLVED: `negotiate_scid_privacy = true` (the client/funder
  must set it), synthetic index in the `2500..2²⁴` band (LDK aliases use
  `0..2499`), persisted allocation, collision-fatal vs real + alias SCIDs (§5).
- **D5 (`cltv_claim_slack` / size of F)** — RESOLVED: **`cltv_claim_slack = 18`**
  (codex r3 grounded it in BIP-431 / Core v29 TRUC / `submitpackage` — one
  confirmation per level, not deeper burial — and LDK `MAX_BLOCKS_FOR_CONF`),
  giving `F ≈ 262` at the defaults — *less* than the Ark-channel variant's
  `≈406` by one `pinned_exit_delta`, since stage-1 carries no success-CSV term.
  codex's earlier serial `≈1757` is rejected. Confirm the constant against the
  offboard/forfeit margin at code-start.
- **Boundary** — the plan's §7 owner table predates the MR-1 shape-bounding
  fold and this HTLC-scoping; this note treats OP-2's shape as done in MR-1 and
  all HTLC/forwarding (I-6) as part 4. Recorded so scope isn't re-litigated.
