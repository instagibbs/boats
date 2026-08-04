# MR-3 design note (G1): captaind as the channel counterparty — the embedded-LDK channels subsystem

**Status**: G1 draft (pre-codex). 2026-08-04.
**Series position**: part 3 of 6 upstream (internal "MR-2 captaind"), stacking
on the protocol-surface MR (!2336) which stacks on the release-contract
opener (!2321). **Baseline**: bark-stage1 `ark8-channels-stage1-protocol`
(`ff69f54c`-era; will rebase to the tip at code-start).
**Theme (plan:370)**: *captaind can be the channel counterparty.*
**Spec targets**: IB-1..7 (upgrade path), PV-6/PV-8/PV-10 (server halves),
BR-3/4/8/12/13/14/15/16/17/18, OP-1..7/14..16/18..28 (server view),
RG-1..8/14..16, WD-2..5/15/16, EX-1/7, DA-1..10 (server + config guard),
I-4/5/6c/9/10 — full discharge table in §12.

## 1. Scope: what this MR is, and the stage-1 cut it is drawn against

captaind gains an **optional** embedded Lightning node so it can be the
counterparty of a channel whose funding output lives, unbroadcast, in a
VTXO's exit chain. Concretely: accept an inbound Ark channel on stock
released LDK, admit an **open-by-upgrade** riding the arkoor cosign, hold
`ChannelReady` until the Ark transfer is durably registered, cosign the
presigned bridge, stand the parent-exit watch that defends the balance
after the input is upgraded away, and sweep the channel VTXO at expiry.
Intra-ark payments (client ↔ captaind ↔ client) then flow as ordinary
Lightning HTLCs.

**In scope**: the `[channels]` `OptionalService` subsystem; LDK node
assembly + persistence + event loop; the open-by-upgrade admission path
(the server half of the two-sided checks); the registration gate; the
bridge cosign integration; the parent-exit watch; the expiry sweep arm;
the `max_vtxo_exit_depth` config-decrease guard; forwarding on a capped,
floor-respecting profile; `supports_channels` advertised true when enabled.

**Out of scope (later MRs / later stages)**, and the note keeps clear of
each:
- **The client open flow, unilateral exit, and intra-ark payment e2e** —
  part 4 (bark). This MR builds the server half and proves it against a
  driver, not the production client.
- **Downgrade/close** — part 5. The parent-exit watch table lands here with
  the *upgrade* kind; part 5 extends it with the downgrade kind and owns the
  split-response watch and its crash tests.
- **Refresh / teleport** — excluded from stage 1 entirely. The reference
  branch's whole teleport apparatus (the C1 promotion gate, the I-1 refresh
  binding, the M3 declared-removal derivation check, `pending_teleport*`
  tables) is **not** ported. A channel-VTXO never refreshes in stage 1;
  expiry is handled by downgrade → round-refresh the plain funds → re-upgrade
  (the part-5 loop). This is why depth admission must reserve split headroom
  at open (§6): the in-place refresh remedy does not exist.
- **The Ark channel type / HTLC-success-CSV asymmetry** — stage 2 (the LDK
  fork). Stage-1 captaind negotiates a **stock** channel type
  (`anchors_zero_fee_commitments` + `static_remote_key`) and refuses any
  other; it does NOT gate on an `ark_channel` feature bit (there is none).
  The HTLC expiry-race safety the fork's CSV asymmetry would give is instead
  a deferred operator policy: capped forwarding and per-HTLC CLTV floors
  (§8). Because all stage-1 forwarding is intra-ark by construction, this is
  an operating-envelope restriction, not a correctness gap in what ships.

**The reference implementation is mined, not ported.** The old branch
(`ark-channels-bridge-2026-06-18`, server side in `server/src/ldk/*` +
`bark-lightning`) is the design source for the admission order, the watch
machinery, and the hardening fixes (§11). Three deliberate divergences from
it are called out where they arise: the gate mechanism (§5, message
interception → confirmation injection), the floor formula (§6/§8, the old
`2·exit_delta` conflation → the spec's unified `F`), and the subsystem
boundary (§2, no `bark-lightning` shared crate — the production node lives
in `bark-channels`, promoted from its test harness).

## 2. Architecture and module layout

### 2a. The subsystem gate

`Config.channels: OptionalService<crate::channels::Config>` (config.rs),
following the sole existing production use of `OptionalService` —
`watchman` (config.rs:349). Disabled by default; a disabled section omits
every other key. `supports_channels` in ark-info (already wired inert in
!2336 as `false`, lib.rs:284-286) becomes
`cfg.channels.enabled().is_some()`.

### 2b. Where the code lives

Upstream has no `bark-lightning`; the production node lives in the
**`bark-channels`** crate (today the release-contract test harness, MR-0),
whose `tests/common/mod.rs` is the direct precursor of the production
assembly. `src/lib.rs` (a stub today) grows the real node; the server
depends on it as a workspace path dep. Server-native glue lives in
`server/src/channels/` (mirroring `server/src/watchman/`):

| Module | Purpose | Harness precursor |
|---|---|---|
| `channels/mod.rs` | `ChannelsSubsystem`, handle, `Ctrl` | — |
| `channels/config.rs` | the `[channels]` `Config` | — |
| `channels/node.rs` | node assembly (mgr/monitor/peer/signer), reload-vs-fresh | `common::build_node`, `restart_from_persisted` |
| `channels/persist.rs` | LDK `Persist` over postgres via the dedicated executor | `common::InMemoryPersist` |
| `channels/db_executor.rs` | separate tokio runtime + pool for sync LDK persist | (old `ldk/db_executor.rs`) |
| `channels/broadcaster.rs` | capture-only broadcaster over the tx nursery | `common::CaptureBroadcaster` |
| `channels/event.rs` | event loop; the outbound-message gate (§5); close recording | `common::spawn_event_pump` |
| `channels/admission.rs` | open-by-upgrade admission (§6) — the arkoor-cosign server half | old `cosign_oor_upgrade` |
| `channels/watch.rs` | parent-exit watch arm (§7) | old `ForfeitWatcher` parent-watch |
| `server/src/database/channels.rs` | `impl Tx` blocks for the new tables (§10) | old `database/ldk.rs` (minus teleport) |
| `lib/src/channel.rs` | (exists, MR-1) bridge construct/sighash/cosign | — |

`bark-channels`'s test doubles (`CaptureBroadcaster`, `InMemoryPersist`,
`TestWalletSource`, `SyntheticChain`, the hand-rolled pump) become
production types backed by the tx nursery, postgres, the rounds BDK wallet,
and `ChainEventListener` respectively.

### 2c. Node assembly

Per MR-0's proven shape on stock `lightning` 0.2.4:
- **Signer**: the server's long-term key as the BOLT-3 counterparty funding
  key (as the old branch did, `ldk/mod.rs:319-331`) — so the channel VTXO's
  `musig(A,S)` keyspend and the funding pair stay coherent. (BR-14: these
  BOLT-3 keys are ordinary LN keys, distinct from A/S, in no policy or
  ark-info field; the server holds both, keyed by `channel_id`.)
- **Persistence**: a direct `chain::chainmonitor::Persist` impl over
  postgres (NOT a KVStore shim), calls routed through a **dedicated runtime
  + bb8 pool** (`db_executor.rs`) — this is load-bearing, not tidiness: MR-0
  and the old branch both hit a hang when LDK's synchronous
  persist-before-sign runs through the shared runtime under two concurrent
  opens (§11.1).
- **ChainMonitor**: synchronous persist (`deferred = false`).
- **Reload vs fresh**: any monitor/manager deserialize failure **panics**
  (fail-fast) — a fresh manager would silently drop justice monitors.
  Restart lifecycle (load-bearing, from the MR-0 review): quiesce and await
  pumps/transports **before** snapshotting; restore dormant; re-`watch`
  every monitor and re-establish chain + funding state; only then start
  processing and reconnect (plan:385-388).
- **Router/peers**: no gossip, no onion routing — channels are direct; peers
  are the server's own clients (intra-ark topology). Inbound TCP listener
  gated on chain catch-up.
- **Event loop**: **decision D1** — evaluate `lightning-background-processor`
  (its async variant already ships in the 0.2.x line and implements the
  event-driven manager-persist pattern) vs. the hand-rolled watchman-style
  loop the old branch and the MR-0 harness both used. Recommendation: adopt
  BP if it composes with `rtmgr` shutdown and the postgres `Persist`; fall
  back to the hand-rolled `rtmgr.spawn_critical` + `tokio::select!` loop
  (the watchman idiom) otherwise. Either way the loop MUST call
  `process_pending_htlc_forwards()` each tick (an undocumented LDK
  requirement pinned by MR-0) and persist on
  `get_and_clear_needs_persistence()`. Resolve in the commit-1 spike.

### 2d. Chain plumbing

The subsystem registers an `Arc<RwLock<…>>` as a `ChainEventListener`
(sync/mod.rs:53-81) in `Server::start`'s listener vec before
`SyncManager::start` (lib.rs:401-432) — the watchman pattern. `on_block_added`
drives (a) real chain sync for the node's own wallet/monitors and (b) the
parent-exit watch's detect step (§7); `on_reorg` unwinds virtual
confirmations and watch state above the fork. The capture broadcaster and
the parent-exit response share the existing tx-nursery / P2A-CPFP machinery.

## 3. The two-sided invariant this MR half-owns

Every open-time check in the spec is enforced **from each party's own view**
("the client MUST refuse to build, and the server MUST refuse to cosign",
spec:438-440). This MR is the **server** half; the client half is part 4.
Stating this explicitly matters because the note's admission section (§6)
reads as if it were the whole check — it is not; it is the half that runs
inside `request_arkoor_cosign`. Neither half trusts the other.

captaind is also, in the stage-1 topology, **never itself an HTLC
destination** — it only forwards between two of its own clients (plan:15).
So the per-HTLC floor obligations split: the server owns only the
*forwarding* delta (I-6c); invoice-advertise, received-HTLC, and
force-close-scheduling floors (I-6 a/b/d) are the client's, part 4. This is
worth a one-line assertion in the code (a captaind node that finds itself
the terminal hop of an HTLC on an Ark channel is a topology violation) so
the scoping is not merely assumed.

## 4. Open by upgrade — the flow captaind participates in

The client self-spends a `pubkey` VTXO (ARK #5) into a `channel-funding`
destination; the transfer carries two channel fields (`channel_id`, bridge
nonce) so the server reconstructs and cosigns the bridge in the same
exchange. The ordering (spec:388-420) puts Lightning establishment
*before* the cosign, with **registration as the point of no return**:

```
 1 BOLT   open/accept → both BOLT-3 funding pubkeys known
 2 local  client builds transfer + bridge (out0 = P2WSH 2-of-2 funding)
 3 BOLT   funding outpoint = bridge_txid:0; exchange initial commitment
 4 gate   initial commitment held  ── last fully-free abort ──
 5 Ark    arkoor_cosign_request (channel variant) → SERVER admits (§6),
          marks input spent, returns ARK #5 partials + the bridge's
 6 local  client verifies every partial incl. the bridge's
 7 Ark    client registers the signed transactions  *** PONR ***
 8 BOLT   ChannelReady — SERVER MUST NOT send before registration (§5)
```

captaind's surface across this flow: it **accepts** the inbound channel at
step 1 (funder is the client — manual funding is funder-only, MR-0), holds
its own `ChannelReady` from step 1, admits + cosigns at step 5, and releases
`ChannelReady` when step 7's registration lands. It never calls manual
funding itself; it learns the funding outpoint from the peer's
`funding_created` and cross-checks it against the bridge it reconstructs
(§6, check 11).

## 5. The registration gate (THE core mechanism) — confirmation injection

**This is the load-bearing divergence from the reference branch.** The old
branch accepted zero-conf and intercepted the outbound `SendChannelReady`
message, diverting it until an Ark-side release flag flipped
(`FundingKeyInterceptor`, `funding_interceptor.rs:1503-1535`). That needs a
message-handler shim wrapping the `ChannelManager` — exactly the kind of
deep LDK coupling stage 1 avoids.

Stage-1 mechanism (MR-0-proven on stock 0.2.4): **ordinary inbound accept at
`minimum_depth ≥ 1`, and withhold the virtual-funding confirmation feed
until Ark registration completes.** LDK itself will not emit `ChannelReady`
until it sees the funding confirmed to the configured depth; the server
simply does not feed its node the synthetic confirmation for a channel whose
transfer is not yet registered. Feeding the confirmation *is* the release.
No message interception, no `ChannelManager` wrapper.

Consequences the note must nail down:
- **The server feeds its own node a synthetic funding confirmation.** The
  old branch's server never did (client-always-funder; its server reached
  readiness via the interceptor, and its `feed_tx_confirmation` was dead
  code hardcoding `tx_index = 0`, §11.7). Stage-1's gate *requires* the
  server to feed it — so this path is live, and the SCID synthesis below
  applies to it.
- **Virtual confirmation obeys the real chain (spec:120-144).** The
  confirmation is presented at the input anchor's *actual* best-chain height;
  the node MUST NOT advance its best-chain view to manufacture depth, MUST
  observe reorgs/unconfirmations in order, and on anchor disconnection MUST
  withdraw the confirmation and suspend operation. A restart MUST NOT weaken
  this: no HTLC accept/forward until a coherent chain+funding view is
  re-held, else fail closed (OP-18..22, WD-15).
- **SCID synthesis (I-10, spec:146-168).** A virtual funding has no block
  position, so the node synthesizes one: the anchor's real height, an index
  derived from the **bridge txid** within BOLT #7's 3-byte space avoiding
  the coinbase slot (`≥ 1`), and the funding vout. MUST be node-locally
  unique (two channels on one anchor share the height, so the index
  distinguishes them), MUST be stable across restarts, MUST persist a
  collision bump, and the channel MUST NOT be announced —
  `option_scid_alias` carries every wire reference. **Do not port the old
  server's hardcoded `tx_index = 0`** (§11.7); apply the bridge-txid
  derivation server-side, and assert collision-fatality as a fast synchronous
  check (MR-0 pinned this shape).
- **Durability (RG-3/5/6, spec:484-494).** "Registration" for this PONR flow
  means the **complete** set of new levels durably held, not merely the
  checkpoint. The release marker (a `backing_registered_at` timestamp, set
  once, `COALESCE`-idempotent — old V38) is restored at startup so a restart
  does not re-hold an already-operating channel. A registration arriving
  *after* the input's exit-chain final tx has confirmed MUST be **refused**,
  not completed (the parent-exit watch resolved with no response) — else
  `ChannelReady` would release into a channel whose backing is being clawed
  back.

## 6. Open-by-upgrade admission (the server half)

Reached from `request_arkoor_cosign`. Today that handler rejects every
`ChannelFunding` destination/input (the MR-1 gate) and passes
`ChannelAuthorization::None` into the builder (arkoor.rs:150-158, 58-60).
This MR adds the channel branch: a package part carrying **both**
`channel_id` and `bridge_pub_nonce` is an upgrade request; captaind runs the
checks below and, on success, passes `ChannelAuthorization::UpgradeOutput`
into MR-1's builder gate (captaind is the *only* call site that ever will).
Every check runs **before** the transfer cosign marks the input spent; all
refusals are structured errors with no state mutation.

Ordered admission (mined from old `cosign_oor_upgrade`, arkoor.rs:342-587,
re-mapped to spec IDs and the stage-1 floor):

1. **At most one channel part** per package (OP-24); the part names exactly
   one `channel-funding` destination (OP-2 — the MR-1 builder already refuses
   isolated / multiple / split channel destinations structurally, §2b of the
   MR-1 note; captaind re-checks the request-level count for a clean error).
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
   is rejected; a `channel-funding` destination without its bridge in the
   same exchange is rejected.
7. **Not already funded / gate not already released** (idempotent re-cosign
   of the *same* backing VTXO excepted) — rides the spent-state atomic
   reservation (RG-14), not an independent lock.
8. **Negotiated-amount equality** (OP-5, BR-4): the reconstructed
   `channel-funding` VTXO's amount == the channel's negotiated funding amount;
   "no such open in progress" is a hard refusal.
9. **Pinned parameters** (OP-16, BR-3/8): the reconstructed VTXO's decoded
   `exit_delta` == the server's `pinned_exit_delta` for this scope — fixed
   once at open from the input's decoded `exit_delta`, **stored, never
   re-read from a live ark-info value**. A server unwilling to operate at
   that value refuses (remedy: client refreshes first).
   `channel_max_vtxo_exit_depth` pinned from the published `max_vtxo_exit_depth`
   at open.
10. **Depth + split headroom** (DA-6/7, OP-14): resulting exit depth
    (input + 1, or + 2 through a checkpoint) ≤ `channel_max_vtxo_exit_depth`,
    and — a **first-release-profile MUST** (spec:1970-1975), not merely a
    SHOULD — resulting depth ≤ `channel_max_vtxo_exit_depth − 2`, reserving a
    worst-case checkpointed downgrade split cosignable for the channel's whole
    life (there is no refresh remedy in stage 1).
11. **Runway vs floor F** (OP-15, I-4/5): remaining runway
    (`expiry_height − real tip`) MUST exceed the server's computed
    **`F = channel_max_vtxo_exit_depth + pinned_exit_delta + cltv_claim_slack`**.
    **Do not port the old branch's `ark_min_cltv_expiry_delta = 2·exit_delta
    + max_depth + margin`** — that conflates the per-HTLC HTLC-recv CSV
    (doubled delta) with the channel VTXO's own unroll distance and is the
    "1×-not-2×" defect class. Use the spec's `F` with checked arithmetic and
    a u16-representability guard (F feeds BOLT `cltv_expiry_delta` fields).
12. **Bridge reconstruction + funding-outpoint cross-check** (BR-18, OP-25):
    reconstruct the bridge with the server's *own held* funding keys (never
    client-supplied) via `lib::channel::bridge_tx`; assert the two equalities
    — the LDK-assigned funding outpoint == the reconstructed `bridge_txid:0`,
    and the reconstructed bridge out0 (value + script) == the negotiated
    funding output. Either failure ⇒ MUST NOT cosign any part.

Then: the generic `cosign_oor_with_builder` (flux-lock, not-exited check,
persist unsigned virtual txs, sign the transfer) → `server_cosign_bridge`
(MR-1's one-shot MuSig2 partial over the bridge keyspend sighash) → persist
the channel state (§10). BR-17: fresh nonce per session; the byte-identical
duplicate is re-signed with a fresh nonce (documented stage-1 conformance
deviation, spec:2007-2010 — no store-and-replay built).

## 7. The parent-exit watch (WD-2..5)

An upgrade leaves the input VTXO's own `delayed-sign(exit_delta, A)` leaf
alive in the old chain, so a user could exit the input and claw back the
capacity now backing the channel — including any balance the server has
since earned. The defense (spec:470-506): the registered checkpoint/arkoor
transactions spend the input at `nSequence = 0`, mineable the next block —
`exit_delta − 1` ahead of the leaf's claim. captaind must:

- **Retain** the new level(s)' signed transactions and **watch** for any
  prefix of the input's exit chain confirming, for as long as the input's
  delayed-exit leaf is live: until the input's output is conclusively spent
  on-chain (by the retained transfer), or a confirmed expiry sweep of an
  ancestor forecloses its creation (WD-2).
- On seeing the input's chain confirm, **broadcast** the retained
  transaction(s), P2A-CPFP-bumped, ahead of the input's `exit_delta` window
  (WD-3). Response actualizes only the channel-VTXO (or checkpoint parent)
  level; bridge and commitment are untouched (WD-4).

Mechanism, mined from the old `ForfeitWatcher` parent-watch but pared to the
upgrade kind: a `channel_parent_watch` row (input_vtxo_id, channel_id,
kind=`upgrade`, response package txids, `registered_at` as the **armed**
flag). The row is written at cosign; it is **armed** only when ARK #5
registration completes (the same trigger that releases the gate) — a
cosigned-but-unregistered upgrade never operated, so its watch resolves with
no response. Detect / respond / progress-to-confirmation run in the
`ChainEventListener`, sharing the offboard-forfeit watcher's TxIndex /
nursery / wallet. Deployment: embedded in captaind or hosted in the
standalone `watchmand` — a topology choice, same machinery (as the old
branch).

**Two carried-over correctness points, one carried-over gap:**
- The row is written **before** the response can fire and survives a crash
  (WD-15) — it is the durable record that arms the defense.
- Arming is set-once; a row may sit cosigned-but-unarmed harmlessly until
  registration.
- **Gap to close, not inherit** (old `forfeits.rs:384-391` `TODO(ark8)`): the
  "resolve when a confirmed ancestor expiry sweep forecloses the input"
  terminal condition was unimplemented, leaking a dead row per settled
  channel. It is a liveness/cleanup gap, not fund-safety, but this MR should
  implement it rather than reproduce the leak (§10 gives the row an explicit
  resolution on ancestor-sweep-observed).

## 8. Expiry, the config guard, and forwarding

- **Expiry sweep (EX-1/7)**: the `timelock-sign(expiry_height, S)` leaf is the
  server's recourse when the user actualized the channel VTXO output but left
  the channel unresolved. The sweep takes the **whole** channel (single
  `musig(A,S)` output; the expiry leaf is the server's alone). This is a
  watchman **policy arm**: the MR-1 forced-match already routes a
  `channel-funding` VTXO to `decide_action_expiry`; this MR makes that arm
  live for VTXOs the subsystem actually tracks. It is the server's *only*
  self-initiated on-chain act (WD-16) — captaind never unrolls a tree on its
  own initiative.
- **No server-side proactive force-close-ahead-of-expiry scheduler.** The
  reference branch had none server-side either (only the passive parent-exit
  watch + client-side treadmill). **Decision D2**: stage-1 captaind relies on
  (a) the client's force-close discipline, (b) the parent-exit watch, and
  (c) the expiry sweep — and adds **no** independent server deadline
  scheduler. Rationale: the sweep already recovers the whole output at
  expiry, so the server loses nothing by not force-closing earlier; a
  proactive scheduler is a liquidity optimization (BR-13), explicitly not
  built (see BR-12/13 below). Flag for review — a reviewer may want a
  server backstop.
- **Bridge retention (BR-12/13)**: a server MAY retain the bridge and MAY
  force-close early by broadcasting bridge + commitment. **Decision D3**:
  stage-1 captaind keeps the **close outcome** (recorded from the
  `ChannelClosed` event), not the bridge — so BR-13's early-force-close
  optimization is documented as *unused*, not built. State this in the MR
  description so its absence is not read as an oversight.
- **Config-decrease guard (I-9)**: the server MUST refuse a
  `max_vtxo_exit_depth` config decrease that would strand a live channel past
  downgrade eligibility (or require closure first), and MUST enforce
  `max_vtxo_exit_depth ≥ 2`. Checked at startup/config-load and across
  restart. This is the one DA-8..10 piece this MR owns; the downgrade
  eligibility check itself is part 5.
- **Forwarding (I-6c)** — **decision D4, and the note's home for it** (the
  plan locks the posture but assigns no commit stage): forwarding is ON when
  the subsystem is enabled, **capped** — the LDK `UserConfig` forwarding
  `cltv_expiry_delta` set at or above `F_in` (the incoming scope's floor), a
  small `max_htlc_value_in_flight`, per-HTLC size caps, and a config kill
  switch. Normative form (spec:1086-1090): a forwarding node MUST require
  `incoming_cltv − outgoing_cltv ≥ F_in`. All stage-1 forwarding is intra-ark
  by construction. These knobs live in `channels/config.rs` and are applied
  in `channels/node.rs`'s `UserConfig` construction; the kill switch is a
  config bool gating `accept_forwards`.
- **`cltv_claim_slack` default** — **decision D5** (the plan deferred the
  numeric default to exactly this note, plan:572-573). F is computed from
  each party's own view, so captaind's default is independent of the
  client's. Proposed: **[TBD — propose a concrete integer, e.g. the
  conf/claim margin the offboard/forfeit paths already assume, and justify
  against a worst-case unroll]**. Resolve before code-start; it sets the
  runway (check 11) and forwarding-delta thresholds the subsystem enforces.

## 9. Config surface

`[channels]` (`OptionalService`), fields (superset of the old branch's
one-field `[ldk]`, which is under-specified for a real deployment):

```toml
[channels]
enabled = false
listen_address = "0.0.0.0:9735"
# forwarding envelope (I-6c / D4)
forwarding_enabled = true          # the kill switch
max_htlc_value_in_flight_msat = <small>
max_accepted_htlc_value_msat = <small>
# timing (D5)
cltv_claim_slack = <TBD>
```

Everything else (channel-type negotiation, reserve = 0, dust exposure, accept
flags) stays hardcoded in the node's `UserConfig` builder, as the old branch
did — not every LDK knob is a config surface. `channels.validate()` runs in
`Config::validate()` (unconditionally; a disabled service validates
trivially). The `captaind.default.toml` artifact gains the section (complete
+ valid so the roundtrip config tests stay green); no test-only fault
injection knobs in production config.

## 10. Database schema

New migration(s) starting at the next free `V<n>` (V55+ at time of writing;
the exact number is assigned at code-start against the rebased tree). Tables,
mined from the old V30/V38/V40 set with the **teleport tables dropped**
(V31/V32/V33/V36's `pending_teleport*` are refresh machinery, out of scope):

| Table / column | Purpose | Old origin |
|---|---|---|
| `ldk_channel_monitors` | LDK monitor blobs | V30 |
| `ldk_channel_manager` | singleton manager blob | V30 |
| `channel_state(channel_id PK, client/server_funding_pubkey, pinned_exit_delta, pinned_max_vtxo_exit_depth NOT NULL, current_backing_vtxo, closed_capacity/balance, backing_registered_at)` | per-channel Ark state + the gate's release marker | V30+V34+V36+V38, merged |
| `channel_parent_watch(input_vtxo_id PK, channel_id, kind CHECK('upgrade'), response_txids TEXT[], registered_at, resolved_at)` | parent-exit watch (§7); `registered_at` = armed | V39/V40 (upgrade kind only; part 5 adds 'downgrade') |

Notes: `pinned_max_vtxo_exit_depth`/`pinned_exit_delta` land **NOT NULL from
birth** (no pre-release backfill migration needed — this is a fresh table on
a fresh feature); the old V36 "delete rows missing now-mandatory columns"
hardening is not reproduced because there are no such rows. The
`ldk_virtual_fundings` table (old V30) is retained if the node's KeysManager
persistence needs it; confirm during commit-1. Schema regeneration:
`just dump-server-sql-schema` → `server/schema.sql` (the committed artifact,
not hand-aggregated migrations); wired via `generate-static-files`.

## 11. Hardening not to regress (from the reference branch)

Each is a fix the old branch earned; a fresh implementation must carry the
property (the teleport-specific ones are excluded with stage-1's teleport
exclusion):

1. **Concurrent-open deadlock** — LDK sync persist through the shared
   runtime/pool starves it under two concurrent opens. → the dedicated
   `db_executor` runtime+pool (§2c). Regression test: two concurrent opens.
2. **Event-queue poison via cooperative-close of an untracked channel** —
   recording a close for a channel with no state row errors forever, stalling
   the shared LDK event queue for *every* channel. → record a close only when
   it has Ark backing.
3. **DoS via client-controllable event replay** — a `ChannelPending` reaching
   funding with incomplete Ark state is a client-controlled violation and
   MUST map to `ForceClose`, never `Replay` (which would let a peer poison the
   shared queue). Server-local `Persist` failures correctly map to `Replay`.
4. **Capture broadcaster skips `CooperativeClose`** — the cooperative closing
   tx spends an off-chain-only bridge output and can never relay; the outcome
   is recorded from the event, never from a broadcast attempt.
5. **Fail-fast on monitor/manager deser** — panic rather than continue with a
   fresh manager that silently drops justice monitors.
6. **SweepWalletSource trusts own unconfirmed change** — trust
   unconfirmed-but-not-`unavailable` outputs so HTLC bump fee-funding is not
   starved by a lagging BDK view.
7. **SCID non-collision derivation applied server-side** (§5) — the old
   server's `feed_tx_confirmation` still hardcodes `tx_index = 0` (dead there;
   live here). Derive from the bridge txid; assert collision-fatality.
8. **Channel-type hard gate** — accept **only** the designated stock type;
   refuse static-remote-key-only, legacy anchors, empty feature sets. (Stage-1
   form: the designated type is the stock `zero_fee_commitments`, NOT an
   `ark_channel` bit — the fork's bit and its CSV protection are stage 2.)

Explicitly **not** carried (teleport/refresh, out of scope): the C1
promotion gate, the I-1 refresh binding, the M3 declared-removal derivation
check, the teleport decide-then-park TOCTOU fix.

## 12. Requirement → mechanism discharge table

*(S = satisfied by reused upstream machinery; T = new test; D = design
decision recorded here. "part 4/5" = deferred to a later MR by design.)*

| ID(s) | Mechanism in this MR |
|---|---|
| IB-1..7 | Upgrade routes through the generic ARK #5 validators (S); IB-3 exiting-input refusal gets its own test (T) |
| PV-6 | captaind is the sole `UpgradeOutput` caller into MR-1's builder gate (§6) |
| PV-8 | `supports_channels` = `channels.enabled()` (§2a) |
| PV-10 | suite green subsystem-off and subsystem-on for non-channel flows (T) |
| BR-3/4/8 | pinned_exit_delta stored-once/never-reread; negotiated-amount equality (§6.8-9) |
| BR-12/13 | keep close outcome, not bridge — early force-close unused (D3) |
| BR-14 | server holds both BOLT-3 keys by `channel_id`, in no policy/ark-info field (§2c) |
| BR-15/16/18 | one-shot bridge cosign; commitment via stock 2-of-2; reconstruct + equalities (§6.12) |
| BR-17 | fresh nonce/session; duplicate re-signed (stage-1 conformance nit, §6) |
| OP-1..7 | server view of the transfer-shape checks (§6.1-5) |
| OP-14..16 | depth / runway / pinned params, server half (§6.9-11) |
| OP-18..22 | virtual-confirmation chain integrity + fail-closed restart (§5) |
| OP-23..26 | channel_id lookup, at-most-one-part, the two equalities (§6.1,6,12) |
| OP-27 | client retry nonce discard — part 4 |
| OP-28/29 | tampered channel_id caught at partial-sig verify; attestation binding parked (§6, MR-1 note) |
| RG-1..5 | complete-set registration, all-or-nothing, idempotent re-upload, crash-survival (§5) |
| RG-6..8 | the gate: no ChannelReady before durable registration; late-registration refusal (§5) |
| RG-14..16 | upgrade rides the single atomic spent-state reservation (§6.7); concurrent test (T) |
| WD-2..5 | parent-exit watch: retain, watch, respond P2A-bumped (§7) |
| WD-15/16 | gate + watch + guard crash-survival; sweep is the only self-initiated broadcast (§7,8) |
| EX-1/7 | watchman expiry arm sweeps the whole channel VTXO (§8) |
| DA-1..7 | published-bound refusal (S); resulting-depth + split-headroom (§6.10) |
| DA-8..10 | only the config-decrease guard here (§8); eligibility check is part 5 |
| I-4/5 | floor F at runway admission — spec formula, not the old 2×delta (§6.11) |
| I-6c | forwarding delta `incoming − outgoing ≥ F_in` (§8, D4) |
| I-9 | config-decrease guard, `≥ 2`, across restart (§8) |
| I-10 | virtual-confirmation fail-closed + bridge-txid SCID synthesis server-side (§5) |

## 13. Commit plan (four stages, per plan:369-395)

Each builds workspace-wide; per-commit `cargo check --all --tests`; TDD;
upstream conventions (tabs, `anyhow` in captaind, changelog keyed to the MR
number, `contrib/agents/agents.md`).

1. **Production node in `bark-channels`** — promote the harness: real
   capture broadcaster over the tx nursery, postgres `Persist` + the
   dedicated `db_executor`, the `BumpTransaction` fee-bump interface, the
   restart lifecycle. **Resolve D1** (background-processor vs hand-rolled)
   here with a spike. No captaind wiring yet; the crate's own tests prove the
   node.
2. **Subsystem scaffold** — `[channels]` `OptionalService`, `channels/`
   module skeleton, the new migrations + schema.sql, the DB executor blocks,
   the peer listener gated on chain catch-up, `supports_channels` flipped to
   track enablement. Inert: accepts no opens yet.
3. **Open-by-upgrade admission** — the §6 checks, `UpgradeOutput` into the
   builder, bridge reconstruct + cosign + equalities, channel-state persist.
   The channel opens and is usable *after* the gate releases (stage 4), or
   this stage gates readiness immediately on registration if sequenced with
   it — decide at implementation.
4. **The gate + watch + expiry + guard** — confirmation-injection release on
   registration, the parent-exit watch (arm/detect/respond/resolve incl. the
   ancestor-sweep terminal condition), the watchman expiry arm, the
   config-decrease guard, forwarding config applied.

Server-side e2e (real captaind + a test driver standing in for the part-4
client) proves: open-by-upgrade happy path; each admission refusal (runway,
depth/headroom, amount, pinned-delta, exiting-input, unknown channel_id,
missing bridge); not-ready-before-registration; late-registration refusal;
parent-exit watch arm + response; expiry sweep; config-decrease guard across
restart; concurrent opens (the deadlock regression). The full open→pay→close
lifecycle e2e is part 4/5; this MR proves the server mechanisms in isolation
against a driver, matching how the old branch's `barkd_*` suite was
structured (mine its scenario names for coverage parity — §9 of the mining
report).

## 14. Open decisions for review (surfaced for Greg + codex)

- **D1** — `lightning-background-processor` vs hand-rolled event loop
  (§2c). Recommendation: adopt if it composes with `rtmgr` + postgres
  `Persist`; else the watchman idiom. Resolved by the commit-1 spike.
- **D2** — no server-side proactive force-close scheduler (§8); rely on
  client discipline + parent-exit watch + expiry sweep. A reviewer may want a
  server backstop.
- **D3** — keep close outcome, not the bridge; BR-13 early force-close
  unused (§8).
- **D4** — forwarding envelope knobs live in `[channels]` config, applied in
  the node `UserConfig`; `incoming − outgoing ≥ F_in` enforced (§8).
- **D5** — captaind's `cltv_claim_slack` numeric default (§8) — **must be
  proposed concretely before code-start.**
- **Boundary reconciliation** — the plan's §7 owner table predates the MR-1
  shape-bounding fold; this note treats OP-2's shape half as already done in
  MR-1 and owns only the request-level amount/identity checks (§6.1). Not a
  contradiction; recorded so scope isn't re-litigated.
