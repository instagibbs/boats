# PROMPT — channels stage-1 test-coverage audit: states × Bitcoin transactions

You are auditing the TEST COVERAGE of the Ark Lightning-channel feature
(ARK #8 stage 1), not re-auditing the code. A transaction-level security
audit already re-derived the wiring story and its remediation shipped
(`2026-08-28-channel-security-audit.md`, `2026-08-31-channel-security-remediation.md`
— read both first; the "Combined findings — FINAL" section and the
ratified dispositions are context you may rely on for SCOPE, not for
coverage claims). Your question is the complement:

> For every reachable state of a channel's transaction DAG, is each
> party's REQUIRED ACTION proven by a test that would FAIL if that
> action were wrong or missing?

Report only. Do not edit code. Deliver one markdown report per the
output contract at the bottom.

## Inputs

- Implementation + tests: `/home/greg/bitcoin-dev/cleanroom/bark-stage1`
  @ `5c245e272` (19-commit stack, `b3697e3c5..5c245e272`). Client:
  `bark/src/channels/`, `bark/src/exit/`; LDK node internals:
  `bark-channels/`; server: `server/src/channels/`,
  `server/src/watchman/`; suites listed below.
- Spec: `/home/greg/bitcoin-dev/cleanroom/boats/08-channels.md`.
- Context docs (same repo, `docs/plans/`): the security audit,
  remediation record, security handoff (ratified dispositions), justice
  gap memo, and `2026-08-31-channel-test-flakes.md`.
- Reference impl (for the quadrant matrix only): bark branches
  `ark-channels-bridge-2026-06-18` and `bark-lightning-channels`
  (`bark-lightning/tests/channel_justice.rs`,
  `barkd_{server,client}_punishes_dishonest_{client,server}_close[_with_pending_htlc]`).

## The unit of audit: a matrix cell

A cell is **(chain state × payload × party × posture)**. For each cell
you must produce exactly one of:

- **COVERED** — `file :: test_name`, plus WHICH assertion binds the
  obliged action (quote or line-cite it);
- **PARTIAL** — the nearest test and precisely what it does not prove;
- **DARK** — no test reaches it.

Rules of evidence (all three required for COVERED):

1. The test reaches the state through a sanctioned flow or an existing
   seam — not by asserting on intermediate scaffolding.
2. The assertion is BOUND to the obliged action: recovered value derived
   from the actual transactions, tied to an outpoint only the intended
   party can spend. "Balance grew," a hardcoded sat floor, or "the
   output was spent" (by anyone) are not proof — the remediation caught
   five illicit passes of exactly these shapes (see "What the process
   caught" in the remediation record; use them as calibration).
3. The test would fail if the machinery were disabled (the mutation
   standard). You are not asked to run mutations; you are asked to judge
   each load-bearing vector against it and flag the ones that would
   survive their machinery being turned off.

Derive the cell↔test mapping from TEST BODIES, not names. Name-level
matching is not evidence.

## Axis 1 — chain states of the DAG

The presigned/held transaction DAG (spec §"transaction DAG"; audit
families T1–T12): genesis exit chain → bridge (nSequence =
`pinned_exit_delta`, out0 = 2-of-2 funding, out1 = P2A) → commitment
(zero-fee, shared P2A) → second-stage HTLC txs / direct claims /
revocation spends / sweeps; alternative spends of the channel VTXO
output: server expiry-sweep leaf (`timelock-sign(expiry_height, S)`),
downgrade split (nSequence=0), cooperative closing tx (fallback-only,
spends the funding).

Walk at minimum these nodes:

- N0 operating: anchor confirmed, bridge held, channel VTXO virtual.
- N1 input/old-scope exit chain PREFIX confirmed (upgrade in flight;
  downgrade watch armed) — cue, not foreclosure.
- N2 exit chain FINAL tx confirmed (fallback foreclosure; late
  registration must be refused).
- N3 bridge in mempool only (client vanished here = WD-16 entry).
- N4 bridge confirmed — funding is a real UTXO.
- N5 a commitment confirmed: OURS / THEIRS-latest / THEIRS-revoked,
  each with and without HTLC outputs.
- N6 second stage: own presigned HTLC-success/timeout; AND the
  cheater's second-stage tx spending an HTLC output of a REVOKED
  commitment.
- N7 cooperative closing tx pending vs. a foreign commitment winner
  (reselect).
- N8 expiry crossed pre-bridge: before vs. after downgrade-split
  registration (watchman `Wait`/`Claim{max(bridge_mature, expiry)}` vs.
  `Progress{bridge_mature−1}`, `server/src/watchman/policy.rs:162-181`).
- N9 sweeps/justice confirmed → `DEEPLY_CONFIRMED` terminal
  (ChannelClaimed / tombstone / swept accounting).
- N10 reorg edges of the above — ONLY where the winner or a money
  decision flips (broader reorg breadth is punted debt, see fences).

## Axis 2 — payload

HTLC in flight (none / offered / received / both); a revoked prior
state exists or not; balance split (all-client / both sides /
sub-dust side); position relative to each PONR (open registration;
downgrade-split registration `past_downgrade_ponr()`; close outcome
recorded).

## Axis 3 — party × posture

For EACH cell judge BOTH parties' obligations separately:

- Client duties (bark daemon): `sync_channel_watch`,
  `relay_channel_broadcasts` (ungated), `maintain_channel_claims`,
  `maintain_downgrade_watches`, `maintain_channel_deadlines` (rungs:
  HTLC-deadline → peer-close → close-lead → hard-line),
  `reconcile_channel_exits`, the exit state machine
  (`ChannelBridge → CooperativeClosing/Commitment → Swept → Claimed`,
  with demotions).
- Server duties (captaind): block feed + withheld-funding release,
  parent watches (upgrade + downgrade) and the monotonic tombstone,
  close capture/outcome, HTLC/close claim funding + relay
  (`CaptureBroadcaster` relays non-candidates), `channel_spendable_output`
  sweeps (build/RBF/batch/retire-at-`DEEPLY_CONFIRMED`), stalled-close
  policy, force-close publish-once-funding-canonical, watchman expiry
  actions, round-admission rejection of ChannelFunding VTXOs.

Postures: counterparty HONEST / ADVERSARIAL (races a retained or
revoked commitment onto the client's bridge; stale-DB revival) /
ABSENT (vanished, never returns); and SELF crashed-and-restarted at a
durable boundary. Remember the audit's refined threat model: a server
commitment reaches the chain only via an adversarial race during a
client exit — use it to prune unreachable cells, and say when you do.

## Seeded cells — check these first

Several are SUSPECTED DARK from the inventory taken while writing this
prompt; verify against HEAD rather than trusting it.

1. **The dishonest-close quadrant matrix ± HTLC.** Client-punishes-
   revoked-server exists (`lifecycle.rs :: client_punishes_revoked_server_close`
   and `_with_htlc`). The inventory found NO
   server-punishes-revoked-CLIENT vector (the reference had
   `barkd_server_punishes_dishonest_client_close[_with_pending_htlc]`;
   the security audit graded server-side justice "plausibly wired,
   UNTESTED"). If truly absent, that is a P1 candidate — note it needs a
   client-side revoked-commitment export seam (the server-side seam
   `channel_export_commitment` has no client mirror), so cost it
   honestly.
2. **Cheater's second stage on a revoked commitment.** The `_with_htlc`
   justice vector revokes a commitment carrying an HTLC and mines it.
   Does any vector cover the cheater ALSO getting its HTLC-success/
   timeout second-stage tx confirmed off that revoked commitment (the
   reference's `fed_output_spends` case), with the victim's justice
   then claiming the second-stage output?
3. **HTLC claims per commitment class per party.**
   `force_close_server_claims_and_payer_scrapes` proves the server's
   second-stage claim on its OWN commitment;
   `client_claims_htlc_off_server_commitment` proves the client's claim
   on the SERVER's commitment. What proves the SERVER claims an HTLC
   (preimage or timeout) off the CLIENT's commitment — the ordinary
   client force-close with an HTLC in flight? (`theirs_commitment_sweeps`
   is explicitly HTLC-free; `channel_unilateral_exit_end_to_end` has no
   HTLC.) And the client's own-commitment second stage: which vector
   binds the client's presigned HTLC-timeout/success confirming and its
   value landing in the exit accounting?
4. **Expiry sweep, server side, asserted as the SERVER's recovery.**
   `channel_expiry_race_server_claims` asserts the CLIENT never
   fabricates a terminal after losing the race. Which test asserts the
   server's expiry-leaf sweep itself — confirmed, value accounted, by
   provenance — pre-bridge at expiry with the client dark? Check the
   watchman suite too; if that machinery is generic-VTXO-tested and the
   channel-funding case differs only by `ChannelFundingExtra`, say so
   and mark the cell covered-by-layer instead of demanding a duplicate.
5. **Bridge confirmed, then each party alone.** Client alone: exit e2e
   + crash ladder. Server alone: WD-16
   (`server_recovers_from_a_vanished_client`), urgent CLTV relay
   (`urgent_htlc_deadline_close_is_relayed`). Verify each of the four
   force-close entry reasons in the remediation's rule (HTLC deadline,
   stalled policy, operator, protocol fault) has its publish-once-
   funding-canonical path proven, or is consciously covered by one
   representative.
6. **Balances by provenance on every close class**: cooperative,
   downgrade split (incl. sub-dust side and zero-output closing
   artifact), ours-commitment, theirs-latest, theirs-revoked,
   zero-local-output counterparty commitment (LDK-level in
   `release_contract.rs :: test_counterparty_commitment_with_zero_local_output`
   — is there a wallet-level cell that needs it, or is the LDK layer
   the right proving layer?). Both parties' `to_local`/`to_remote`
   per class.
7. **Justice liveness vs `to_self_delay`.** The cheater's `to_local`
   CSV is LDK's default 144. No vector binds "the penalty lands within
   the delay window given the daemon's tick cadence and
   `OBSERVED_TIP_STALE_SECS`." Decide whether that is a real coverage
   gap or a consciously-skipped cell (bounded by watch-liveness
   assumptions already documented) — and justify.
8. **Server crash-restart symmetric cells.** The client crash matrix is
   extensive. Server side: restart mid-sweep-attempt (V64 rows with
   `superseded_txids`), restart between close-package relay attempts,
   V63 claim-lock re-arm (one vector exists — is it bound?), downgrade
   tombstone monotonic across restart (the reorg half is tested
   server-side; the restart half?), `stale_manager_cannot_resurrect_a_closed_channel`
   (exists — check binding).
9. **Timelock boundary refusals** — the checklist below: for each
   enforced invariant, is the REFUSAL side tested at the boundary
   (±1 only where money flips), or only the happy side?
   `keysend_at_floor_claims` tests exactly-at-floor acceptance; what
   tests floor−1 refusal?
10. **Winner-flip reorg cells that move money** (the one reorg family
    NOT punted): closing-tx vs foreign-commitment flip after adoption
    (`coop_close_adopts_foreign_commitment` covers adopt-before-settle;
    what about a flip AFTER `ChannelSwept` demotes — client covered via
    `exit_recovers_from_commitment_reorg`? bind it), and the server
    tombstone's reorg monotonicity (covered —
    `downgrade_watch_response_and_fallback_tombstone`; verify binding).

## Timelock coherence checklist

For each row: enforcement site exists (given), so the audit question is
WHERE IS THE TEST — and is the failing side tested, not just the happy
side. Do not propose renegotiating the constants.

| Invariant | Enforcement site |
|---|---|
| Open admission: runway > F (spec 08-channels.md:456-461) | server admission |
| Client hard line: `floor = pinned_max_vtxo_exit_depth + pinned_exit_delta + pinned_claim_slack`; `hard = floor + refresh_lead`; close starts at `hard + channel_close_lead` | `bark/src/channels/mod.rs:3922-4067` |
| Deadline rung ordering: HTLC-deadline → peer-close → close-lead(cooperative) → hard-line(exit), and cooperative preferred inside the window | `maintain_channel_deadlines` |
| Receiver per-HTLC floor `cltv_expiry − height ≥ F`; forwarder `incoming − outgoing ≥ F_in`; u16 fit refusal | client payments / server `htlc_floor_profile` coherence + immutability guards (`server/src/channels/mod.rs:208-242`) |
| Watchman channel-funding deadlines: `Claim{max(bridge_mature, expiry)}`, registered response `Progress{bridge_mature − 1}` | `server/src/watchman/policy.rs:162-181` |
| `DOWNGRADE_EXPIRY_MARGIN = 6` refusal band | `server/src/channels/downgrade.rs:50-62` |
| Split headroom (`≤ max_vtxo_exit_depth − 2`) at open AND at split | `admission.rs`, `downgrade.rs`, reopen ladder |
| `MIN_CHANNEL_FUNDING = 2×P2TR_DUST` at open and at close relayability | `lib/src/channel.rs:72-73`, `admission.rs:483-490` |
| `DEEPLY_CONFIRMED = 100` gates BOTH server sweep retirement and client terminal | `sweeps.rs:122-140`, `exit/progress/states.rs:957-981` |
| `to_self_delay` (LDK 144) independent of `pinned_exit_delta`; bridge CSV served before commitment (commitment carries no extra delay) | LDK default + bridge nSequence |

Also flag any timelock whose SAFETY story depends on mempool
observation — the standing rule is durable local state + chain only,
never mempool contents.

## Fences — what is NOT a finding (gold-plating control)

- **Stage-2 / extension material**: the CSV Ark channel type (the
  per-branch `exit_delta` CSV HTLC scripts, response windows, spec
  ~§1288-1622 extension parts), the retained-bridge EARLY server
  force-close (spec :288), and the in-place refresh/teleport quiescence
  FSM (~§1703-2223). Stage-1 is the core profile; these have no
  stage-1 cells.
- **Ratified ACCEPTS**: R1 (no-CSV HTLC fee-race) and R2
  (expiry-boundary fee-race) are accepted+documented — do not demand
  tests for losing races the design accepts. Their MITIGATIONS'
  machinery (deadline rungs, close lead, R3 `exit_funding` reserve/
  reporting) IS in scope.
- **Punted debt**: reorg breadth beyond winner-flip cells (GAP 5), dust
  trimming, P2A pinning / Esplora capability. Confirm the punt boundary
  if a cell sits on it; do not re-litigate.
- **Environment**: the 5 CLN/bip321 failures and the 3 load flakes
  (`2026-08-31-channel-test-flakes.md`) are not coverage findings.
- **Layering**: one proving layer suffices. `bark-channels/tests/*`
  (synthetic chain) proves LDK-contract invariants but CANNOT prove
  daemon/duty wiring — a duty cell needs at least one integration/e2e
  proof; conversely do not demand an e2e re-proof of a pure invariant
  already pinned at unit level. Never file "add a unit test duplicating
  an e2e" or vice versa.
- **Reachability**: to raise a cell you must state its reachability
  path (sanctioned flow or existing seam). Structurally unreachable
  states go in the consciously-skipped list, not findings.
- **Cost discipline**: prefer existing seams (surface below). A finding
  whose vector needs a NEW seam must say so explicitly and justify the
  seam. These vectors run 200-900s each; every proposal carries a
  runtime class and must earn it.

## Existing suites (verify counts at HEAD; read bodies)

- `testing/tests/bark_sdk/lifecycle.rs` — 32 vectors (force-close arc,
  justice pair, HTLC-off-server-commitment, WD-16, server balance
  sweep, exit crash ladder, coop-close foreign-winner reselect, blocked
  claims, deadline rungs, quarantine, chain-overrules-tombstone…).
- `testing/tests/bark_sdk/channels.rs` — 19 (open/close/exit
  lifecycles, PONR, FALLBACK_WON, downgrade watch respond-by-exit,
  reopen ladder, expiry race, reorg recovery, cancel semantics).
- `testing/tests/bark_sdk/channel_proxy.rs` — 4 (gRPC adversarial:
  duplication, lost responses, tampered attestation).
- `testing/tests/bark_sdk/payments.rs` — 5 (composition, forward,
  floor-exact keysend, single-claim, fabricated strand).
- `testing/tests/server/channels.rs` — 10 + `server/postgres/channels.rs`
  — 8 + 4 in `server/mod.rs` (admission, real-LDK establishment, watch
  lifecycles, close outcomes, split admission/registration atomicity,
  tombstone monotonicity, DB constraints, caps, reorg).
- `bark-channels/tests/release_contract.rs` — 14 (LDK pin: virtual
  funding, SCID, TRUC claims, reload, zero-local-output) +
  `feed_barrier.rs` — 5 + config units — 4.
- Surface suites: `testing/tests/barkd/channels.rs` (2, REST),
  `testing/tests/bark/channel.rs` (1, CLI).
- Unit tests: `bark/src/channels/` 33 (claim funding 14, watch engine
  9…), `bark/src/exit/` 5, `server/src/channels/` 7.

Seam surface (client `test-util`): `test_hold_open_{establish,registration,feed}`,
`test_hold_close_{registration,negotiating}`, `test_hold_send_issue`,
`test_hold_claim_{binding,pass}`, `test_hold_event_handling`,
`test_payment_cltv_budget`, `test_pending_htlc_expiries`,
`test_set_channel_feerate`, `test_claim_lease_fresh`,
`test_exit_owned_txs`, restart helpers, stale-DB-copy fixtures.
Server (`dev_seams` + `channel-test-seams`, mainnet-refused):
`channel_cooperative_close`, `channel_export_commitment`
(non-destructive revoked staging), `test_crash_after_collect_binding`;
`channel_force_close` is a real operator RPC tests reuse. Harness: the
`ArkRpcProxy` trait, real-reorg via `invalidateblock`, attacker-chosen
`generateblock` direct mining, compressed `stalled_close_*` fixtures.

## Second pass — quality of EXISTING money vectors

Independent of dark cells: list every money-recovery vector whose
assertions could pass with its machinery broken (the illicit-pass
audit). Judge against the standing standard: value derived from actual
txs; outpoint-owner binding; terminal-state assertion (ChannelClaimed /
Downgraded / tombstone / swept row) rather than a passable
intermediate; and no assertion satisfied by the OPPOSING party's
success. Where you find one, propose the strengthened assertion
concretely — quote the line to change and what to change it to.

## Output contract

Produce, in order:

1. **The matrix** — compact table(s), one row per audited cell:
   state/payload | party+posture | obligation (spec/disposition cite) |
   verdict COVERED/PARTIAL/DARK | proving test + binding assertion.
   Covered cells stay terse; this table is the standing coverage map.
2. **Findings, ranked**: P1 = DARK money cell (loss or indefinite lock
   if the action is wrong); P2 = normative MUST or state transition
   unproven but bounded; P3 = quality (illicit-pass risk, unbound
   assertion). Each finding: the cell, reachability path, concrete
   consequence scenario (inputs/state → wrong outcome), nearest
   existing vector and precisely why it does not prove the cell, and a
   PROPOSED vector — name, home suite, seams used (flag NEW seams),
   assertion sketch meeting the provenance standard, runtime class.
3. **Adequately covered** — one line per verified-strong cell cluster.
4. **Consciously not worth testing** — cells you examined and decline
   to demand, each with a one-line reason (unreachable / accepted risk
   / covered-by-layer / consequence-free). This list is mandatory; it
   is where gold-plating pressure goes to die, and it makes a PASS
   auditable.
5. **Verdict**: PASS (coverage ship-adequate for stage 1) or FAIL
   (any P1). If you disagree with a fence or a ratified disposition's
   coverage implication, do not just object — give the concrete
   scenario and the concrete alternative vector, and mark it clearly as
   a scope challenge rather than a finding.
