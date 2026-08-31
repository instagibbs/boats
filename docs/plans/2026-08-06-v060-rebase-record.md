# Rebase record — the stage-1 stack onto v0.6.0 (2026-08-06)

**Subject**: the whole 9-commit stack (opener 2 + protocol 3 + captaind 4)
rebased from upstream master `6125d689` onto `900cd3ca` (v0.6.0 plus
follow-ups; 49 commits of divergence). Done between MR-3 reaching SHIP and
MR-3 being posted, per the standing rebase-before-posting rule: MR-3 would
not merge on a v0.5 base, and MR-4 builds on policy APIs that migrated in
this very range.

**Outcome**: all 9 commits rebased, conflicts resolved per-commit,
`cargo check --all --tests` green at every commit, full battery green at
the tip. Codex resolution review: see the verdict section at the bottom.

Final rebased series (change-id → commit, after the codex round's fixes):

| change-id | commit | subject |
|---|---|---|
| zznmluto | `5130da08` | bark-channels: scaffold the harness crate |
| tuxnppqp | `a7c154b8` | bark-channels: pin the LDK 0.2.4 behaviors |
| tvoqttuo | `d4b05c03` | lib, server: the channel-funding VTXO policy |
| lxvlvqnk | `30ee616b` | lib: the presigned bridge transaction |
| ttzpwnmk | `f64ca92b` | server-rpc, bark: optional channel fields |
| nrslwtrv | `88995911` | bark-channels: promote the harness into the production node |
| mynnnooz | `15c99517` | captaind: the channels subsystem scaffold |
| syyqlnln | `a9eb84c4` | captaind: admit the open-by-upgrade |
| qzxwpnzr | `8994717b` | captaind: release the gate |

## The collision groups and their resolutions

### 1. The VtxoPolicy `_v0` migration (the protocol branch's collision)

Upstream renamed `ServerHtlcSend/Recv`, `HarkLeaf`, `HarkForfeit` to `_v0`
and added new v1 variants alongside, taking wire tags `0x08–0x0b` — and
`0x08` was exactly what `ChannelFunding` had claimed on the old base. The
constants block auto-merged into a **silent tag collision**
(`SERVER_HTLC_RECV = 0x08` and `CHANNEL_FUNDING = 0x08` coexisting, a
wire-corruption bug rustc would only have warned about as an unreachable
pattern).

- **`ChannelFunding` moved to `0x0c`.** The pinned wire-bytes test
  re-pinned (`0c` + 33-byte key); the taproot and bridge vectors are
  tag-independent and unchanged. The spec followed: `02-vtxo.md`,
  `08-channels.md` ×2, matrix PV-1/PV-7 now say `0x0c`.
- Every conflicted match site union-merged: upstream's `_v0` arms + our
  `ChannelFunding` arm (policy enum, Kind enum, Display/FromStr,
  policy_type, is_arkoor_compatible, arkoor_pubkey, user_pubkey,
  taproot, clauses, encode/decode, the `ServerVtxoPolicy::User`
  passthrough set).
- The `_v0` dispositions in OUR gates preserve upstream's baseline
  exactly: round **inputs** allow both `_v0`s (upstream's baseline had no
  input-policy restriction; our extracted gate must not refuse anything
  upstream admits), round **outputs** refuse them (upstream's inline
  match refused them), `store_vtxos` and offboard inputs allow them,
  m0021 counts them as HTLCs, telemetry counts them with their v1
  counterparts. `ChannelFunding` refusals ride on top, unchanged.
- Two compiler-surfaced sites our commits own gained `_v0` arms
  (`bark/src/vtxo/mod.rs` store gate, `server/src/offboards.rs` input
  gate); one new upstream test call site
  (`testing/tests/server/lightning.rs`, the expired-HTLC-claim attack
  repro) gained the `ChannelAuthorization::None` argument our builder API
  requires.

### 2. split-watchman (the captaind branch's collision)

Upstream split the watchman into a **reaction loop** (`Action::Progress`
/ `Action::Claim`, mandatory deadlines, `reaction_interval`, default 10m)
and a **sweep loop** (`Action::Sweep`, no deadline, `sweep_interval`,
default 2h); `decide_action_expiry` now returns `Sweep`.

- Our `ChannelFunding` watchman arm initially kept delegating to
  `decide_action_expiry` → `Action::Sweep`. **The codex pass refuted that
  classification** (its one finding, Important): the channel-funding
  output is *contested* from expiry — the holder's presigned bridge
  keyspend stays valid once its relative timelock (`exit_delta` from the
  output's confirmation) matures, and never stops being valid — so by the
  split's own taxonomy it is the `HarkForfeit` class (contested ⇒
  deadline-bearing reaction), not the `Expiry` class (server-only ⇒
  sweep). Left as a sweep, the reclamation would idle on the 2-hour
  cadence while the holder could take the output. **Fixed**: a dedicated
  `decide_action_channel_funding` — Wait before expiry, then
  `Action::Claim { deadline: max(expiry_height, confirmed_at +
  exit_delta) }` (the first height at which both spend paths are live),
  riding the reaction loop; three unit tests pin the decision (pre-expiry
  wait, claim-at-expiry with a mature bridge, late-exit deadline at
  bridge maturity). Both loops share the same claim executor, so the
  spend construction is unchanged and stays e2e-proven. Note: no cadence
  can make the server *always* win this race (a bridge broadcast at
  expiry−1 beats any loop); in stage 1 the output holds zero server
  balance so the race is economically empty — the reaction class restores
  the old combined-loop promptness and the honest deadline alerting, and
  part 4 (payments → real server balance) inherits the correct
  classification.
- The parent-exit response rides the generic `decide_action_pubkey`
  Progress path = the reaction loop with its exit-delta deadline —
  exactly the semantics commit 4 was built on.
- `WatchmanHandle::trigger_sweep()` (`Ctrl::TriggerSweep`) still runs
  **both** loops, so commit 4's post-activation nudge keeps its meaning
  (an offline parent-exit response is driven promptly on restart).
- The channels e2e config sets both `reaction_interval` and
  `sweep_interval` to the old fast tick (the suite drives both loops:
  response = reaction, expiry foreclosure = sweep).

### 3. The SyncManager reorg fix: ours dropped, upstream's kept

Upstream landed `fix-captaind-sync-on-reorg-error` (`10c35912` + the
regression test `c5c9002d`) in the range — the same bug we had found and
fixed inside commit 2 (`update_sync_height` committed before the listener
loop, so a listener error during a reorg was never retried). The two
fixes are line-for-line equivalent (the same move of `update_sync_height`
to after the DB delete) and the two regression tests are the same
fail-once-listener design. Commit 2 no longer touches
`server/src/sync/block_index.rs` or `testing/tests/server/block_index.rs`;
its message dropped the fix paragraph. Upstream's test passes against the
rebased tree.

### 4. max_vtxo_exit_depth changes

Upstream made the arkoor validation's `max_input_exit_depth` an
`Option<u16>` to exempt lightning claim/revocation from the bound, and
capped vtxopool change depth. Channel-open admission keeps enforcing the
bound: `Some(config.max_vtxo_exit_depth)`, matching upstream's own
`cosign_oor`. Our captaind-side depth guard (backing depth +
`SPLIT_HEADROOM` ≤ max, ≥ 2) is untouched by either change.

### 5. Housekeeping the release bump forced

- `bark-channels` was created at the then-workspace version `0.5.0`;
  upstream's release commit bumped every workspace crate to `0.6.0` and
  could not know about ours. Bumped to `0.6.0` (Cargo.toml + lock) in the
  scaffold commit.

### Non-collisions verified

- Proto field numbers survive: ArkInfo `supports_channels = 21` (upstream
  last index still 20), cosign request `7/8`, response `3/4` (upstream
  still ends at 6 and 2).
- No new server DB migrations upstream — `V55__channels.sql` keeps its
  number; `server/schema.sql` regenerates with zero drift.
- `bark-rest/openapi.json` regenerates with zero drift.
- Commit 4's SyncManager additions (`chain_tip_watcher` /
  `sync_height_watcher`, the `sync_height == chain_tip` activation
  barrier) rebased intact and compose with upstream's reorg fix (which is
  the fix the barrier was designed against).

## Battery at the rebased tip

- `cargo check --all --tests` green at each of the 9 commits.
- Units (nextest, workspace minus ark-testing): **515/515** (512 at the
  first pass; +3 = the new `decide_action_channel_funding` pins).
- Channels + block_index e2e vs real bitcoind v29 + postgres: **13/13**
  (establishment, open-by-upgrade admission, parent-exit watch lifecycle,
  shallow reorg, inert boot, disabled-not-advertised, cosign refusal, the
  4 postgres channel suites, and upstream's reorg-retry regression).
- Wider server e2e slice: **51/55**; the 4 failures are the pre-existing
  lightningd environment gap (`gmp version mismatch`, machine-level,
  predates the rebase; they die at daemon startup).
- Clippy: `ark-lib` 0, `bark-channels` 0, `bark-server` 235 (the old 236
  baseline shifted by upstream churn; the handful of warnings in files we
  touch existed pre-rebase inside the accepted baseline — byte-for-byte
  the same code — and the rest are upstream's own).
- `just dump-server-sql-schema` and the openapi dump: no drift.

## Codex resolution review

Round 1 (max effort, scoped to the resolutions + a wildcard-arm sweep
over the upstream range): **REWORK — exactly one finding**, the
watchman-classification error described in group 2 above. Everything else
judged clean: the 0x0c tag move and its test/vector coherence, the `_v0`
dispositions, the proto field survival, the dropped-in-favor-of-upstream
SyncManager fix, the admission depth-bound `Some(...)`, and no upstream
wildcard arm mishandling `ChannelFunding`.

Fix applied same day (see group 2), battery re-green (per-commit checks
×9, units 515/515, channels e2e 13/13, clippy baselines unchanged).
Narrow confirmation pass: **CONFIRMED-FIXED, no functional findings** —
`Claim` verified as the correct class (frontier routes it through the
reaction loop into the same unchanged `process_claims` executor); the
deadline formula verified (`confirmed_at` is the confirmation height of
the transaction carrying the VTXO outpoint, i.e. the bridge's CSV
parent; checked arithmetic matches `decide_action_hark_forfeit`);
`decide_action_expiry` retains only uncontested policies. Its one Minor
(an e2e comment still calling the expiry foreclosure a "sweep") fixed in
the same commit. It also noted the e2e's equal 2-second intervals mean
the suite proves the outcome but not the loop routing — the three unit
tests are what pin the routing.
