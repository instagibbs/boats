# Stage-1 stack rebase onto upstream master (2026-08-24)

Decision (after a measured trial): rebase the whole 13-commit stack now
rather than cutting the next MR on the stale base. The clincher was that
upstream landed the multi-piece arkoor change-builder (their VTXO-chain-
depth work) — exactly the substrate the close-by-downgrade MR's split
path builds on; designing against the superseded single-output builder
would have baked in immediate rework. Secondary: drift compounds at
~90 upstream commits/week (222 since our 2026-08-06 base), none of the
posted MRs had merged, and the stack sat at its cleanest checkpoint.

## What the drift cost (resolution log)

All thirteen commits conflicted; all resolutions were mechanical
re-seats — no design changed:

- **Workspace**: member-list union (upstream extracted `bark-common` /
  `bark-runtime` utility crates — small, no structural displacement).
- **Policy commit**: re-seated the channel-funding refusals into
  upstream's multi-piece change builder (`new_with_checkpoints(inputs,
  outputs, channel_authorization)` — the merged lib carried our
  authorization parameter through their new builder cleanly); respected
  upstream's deletion of the bitcoind-backed exit checks ("checked at
  the database" — the DB layer refuses `has exited onchain` at every
  state transition), which also obsoleted our admission's
  `check_vtxos_not_exited` pre-check; patched upstream's new
  `new_claim_all_with_checkpoints` test call site for the added
  parameter.
- **Wire fields**: proto renumbering — upstream took field numbers
  21/22 in ark info, ours moved to 23 (`supports_channels`) and 24–26
  (channel node id / addresses / claim slack); threaded the new fields
  through upstream's new custom `ArkInfo` deserializer stub; openapi
  regenerated.
- **Gate release**: `bcd::get_tx_out` → `try_get_tx_out` (the
  `BitcoinAsyncRpcExt` trait) for the registration prevout probe.
- **Client node / open / exit**: upstream moved `daemon.rs` →
  `daemon/mod.rs` (TipWatcher rework, 56% rewritten) — ported the
  channel-node driver supervision, the fee push, the channels sync
  loop (peer maintenance / pending opens / feed reconcile / deadlines /
  exit reconcile), and the teardown transport sweep into the new
  module, adapted to its `self.wallet()` accessor; Cargo.lock reset to
  upstream's + `lightning --precise 0.2.4` (upstream's lock pins 0.2.2,
  which does not even build their rapid-gossip-sync pin; our verified
  0.2.4 behavior pin stands, so the ledger/floor analysis needs no
  re-verification); added a channel-refusal arm to upstream's new
  `exit/estimate.rs` exhaustive state match (unreachable in practice —
  the wallet store refuses channel VTXOs before the match — kept as a
  refusal rather than a silent mis-estimate).

**The migration hazard materialized — server side.** Upstream added
`V55__lightning_payment_attempt_subscription_link` (+V56) to the
refinery migrations; our `V55__channels.sql` collided (duplicate key on
`refinery_schema_history`), killing every captaind at startup. Renamed
to `V57__channels.sql` in its owning commit. The client sqlite numbering
(m0043–46) remained free.

## Verification at the rebased tip

- Workspace + all test binaries build; units: bark-wallet 137/0
  (upstream's new units merged in), bark-server 116/0, bark-channels
  17/0.
- Server suite: **149 passed, 40 failed-environmental, 1 skipped** —
  every failure is `lightningd: gmp version mismatch` (CLN cannot start
  on this machine; documented pre-existing limitation, now wider only
  because upstream added many lightning tests). Zero rebase regressions.
- Client channel battery: **13/13 green** — the four fast opens plus
  the cancel lifecycle and the expiry race (6, nextest), the four fast
  ones again under the double-drive reentrancy harness, and the heavy
  trio (unilateral, reorg recovery, mixed portfolio) in parallel under
  the nextest slow-timeout override at 372–379s each.
- bark/schema.sql matches its dump at tip; server schema.sql dump is
  DDL-only (no migration-name rows), unaffected by the rename.
- Per-commit `cargo check --workspace` sweep across all thirteen
  commits: green (each commit independently compiles on the new base).

## Codex sanity pass (2 rounds)

Round 1 verified all thirteen own-diff comparisons account cleanly for
the documented resolutions (admission's exited-VTXO refusal preserved
through the DB layer, RPC semantics, proto/JSON/OpenAPI threading,
migration uniqueness, no lost hunks) and found ONE blocking regression:
the ported `run_channel_node` retained an upgraded `Wallet` across the
lifetime-long `driver.await`, defeating upstream's new drop-stops-daemon
weak-reference invariant (the daemon holds only a `Weak`; a strong
wallet across an always-on await keeps `WalletInner::drop` from ever
cancelling it).

Fixed per the suggested shape — the upgrade is a bounded scope cloning
only the `Arc<ChannelNode>` — folded into the client-node commit. A
regression test (`daemon_stops_when_the_last_wallet_drops`, folded into
the exit commit's e2e) drops the only wallet without `stop_daemon_wait`
and requires a reopened wallet to reach usable and complete an exit
initiate/cancel round-trip; the negative control (fix reverted) fails
in 37.5s via duplicate-peer rejection while the fixed build passes in
7.8s. Round 2: "Fix holds only Arc<ChannelNode> across the lifetime
await; sync/fee work is bounded and teardown has no await… VERDICT:
PASS."

## Final rebased series

`ark8-channels-stage1` `6cfbb2c4` → `-protocol` `425de13e` →
`-captaind` `136354ac` → `-client` `81e0a08d`
(client chain: node `91da6a23`, discovery `19c27803`, open `b3697e3c`,
exit `81e0a08d`). All four bookmarks require force-pushes. Units
re-verified at the final tip (137/0).
