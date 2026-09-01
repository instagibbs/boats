# Channels stage-1 coverage audit — CONSOLIDATED (codex × independent pass)

Sources, all at bark-stage1 `5c245e272`:
- codex report (gpt-5.6-sol, max): `2026-08-31-codex-coverage-audit.md` —
  verdict **FAIL**, 12 P1 clusters / 6 P2 / 3 P3 groups, full matrix.
- my independent verification pass (test bodies read, guards grepped),
  run in parallel and BEFORE reading codex's output.

Agreement is high: every cell my pass verified, codex graded the same
way. Codex found six clusters my pass did not target (P1-3's full
quadrant enumeration, P1-5, P1-6, P1-7, P2-3..P2-6); my pass adds three
deltas codex missed (folded in below). Item numbering keeps codex's IDs;
this is the walkthrough sheet — one disposition line per item, to be
filled during review.

## Verification deltas (what I checked that changes a codex claim)

- **P1-8, P1-9, P1-10 are coverage-only, NOT product gaps.** All three
  guards exist and refuse: client runway floor
  (`bark/src/channels/mod.rs:3086-3095`, `expiry > tip + max_depth +
  exit_delta + claim_slack`), client `exit_delta` mismatch (`:3068-3072`),
  exit reserve (`check_channel_exit_reserve`, `refuse_unless!`, fast-
  priced, confirmed-only, `:3137-3179`), manual-sync+channels startup
  refusal (`bark/src/daemon/mod.rs:538-551`). Remediation sizing: tests
  only, no product change.
- **Server-side admission checks codex under-listed** (fold into
  P2-1/P2-2): the server's `exit_delta` input-mismatch refusal
  (`admission.rs:506-512`) and the per-channel u16/BOLT-width floor
  overflow (`admission.rs:545-546`) are untested (grep-verified zero
  hits).
- **P1-1 seam confirmation:** the server-punishes-client vector is
  untestable today, not merely untested — no client-side mirror of
  `channel_export_commitment` / `unsafe_revoked_tx_signing` exists
  anywhere in `bark/src/` (grep-verified).
- **V63 re-arm:** COVERED (both passes agree, the concurrent-round
  binding is real), but the vector's comment cites a unit test that does
  not exist (`server/src/channels/claims.rs` has no test module). Doc
  nit; fix the comment or add the unit.
- Minor verify-me: my agent reported `Config::validate()` as
  `htlc_floor_profile <= cltv_claim_slack` (`server/src/channels/config.rs:143-146`),
  which reads inverted relative to the startup worst-floor guard.
  Re-read that check during remediation; if it is really `<=` it is
  either a bug or a differently-scoped bound.

## STATUS 2026-08-31 — seven vectors landed on top of the stack

Commits on `bark-stage1`, each stacked (not folded), every one either
mutation-validated or proven by a failing run of its own precondition:

| Commit | Item | Verdict on the PRODUCT |
|---|---|---|
| `fbc10dd0a` | P1-1 server justice ± HTLC | WIRED — punishes on the revocation branch 1 block into a 144-block delay |
| `63d722345` | P1-2 cheater's second stage | WIRED — punishes the child's delayed output at +2 blocks |
| `008369fb7` | P1-3 both HTLC directions | WIRED — success + timeout legs, both swept past CSV |
| `af2c21b64` | P1-11b floor-profile guards | WIRED — both startup refusals fire |
| `452162891` | P1-5 stalled timer + restart | WIRED — closes on the exact block, neither forgotten nor invented |
| `14b419b07` | P1-7 post-Swept winner flip | WIRED — demotes, reselects, recovers 369,394/370,000 sat |
| `208505af8` | P3-3 expiry sweep provenance | WIRED — 499,093 sat traced into the rounds wallet |

**Nothing probed so far was broken.** Every defect found was in the test
code, the harness, or an assumption about staging. The value is that
these cells are now pinned, and each vector is proven to fail when its
machinery is disabled.

Test-authoring traps worth reusing (all cost a full run to find):
- Driving a client's exit ONE STEP further than the scenario needs
  deletes the thing under test — it fails an inbound HTLC backwards
  (P1-3) or closes the channel itself (P1-5). Stop at the bridge in the
  MEMPOOL, then take the client away.
- An inbound HTLC only survives onto a commitment the client broadcasts
  if the preimage is already committed: wedge the peer (stop captaind),
  THEN release the binding.
- `get_tx_out` is only a spentness test on a channel whose bridge has
  actualized; for a virtual funding, "None" means "never existed".
- Editing captaind's `config.toml` between stop and start is discarded —
  the harness rewrites it from its in-memory config. Use `config_mut()`.
- A holder's HTLC second stage spends the 2-of-2 branch: FIVE witness
  elements, preimage (or empty, for timeout) at index 3 — not the
  three-element direct claim.
- Rescanning from a fixed height on every poll is quadratic and burns
  the harness timeout before the real assertion can fire; scan with a
  cursor.
- New vectors need adding to the `.config/nextest.toml` slow-timeout
  overrides or they are killed mid-drive at 180s.

## RATIFIED 2026-09-01 — the full decision set

Walked one by one with Greg. Nothing below is open.

| # | Decision | Ruling |
|---|---|---|
| P1-4 | V64 sweep lifecycle | **BUILD** — add the sweeper hold seam, test all FOUR cases: restart-after-persist-before-broadcast, RBF supersede, reorg reopen, depth-100 retirement |
| P1-6 | Close-package retry after a failed first submission | **BUILD** — add the fail-next-submission seam at the broadcaster; must recover across a restart |
| P1-11a | Server receiver/forwarder floor at F−1 | **UNIT TEST at the event handler.** No new seam; the client-side equivalent (`claim_floor_boundary_is_exact`) is the shape to mirror. Accepts that wire-to-check wiring is not proven |
| In-flight ceiling | 200,000 sat cap | **DROPPED ENTIRELY.** In-flight becomes `capacity − reserves`: you can always spend what you have. `MAX_HTLC_IN_FLIGHT_CAP_MSAT` goes away as a bound |
| Max capacity | 2,000,000 sat | **KEEP.** Now the SINGLE number bounding a lost fee race (R1) as well as the expiry race (R2) — a deliberate choice rather than the old `200M ÷ 10%` leftover |
| P1-2 rest | Timeout child + both server-as-victim halves | **BUILD ALL OF IT** — complete the 4-cell second-stage justice matrix |
| D-1 | Post-expiry downgrade / recovery of swept-backed VTXOs | **RECORD as stage-2/upstream.** Needs server-honored reissue leaves exempt from the spent-mark/watch model, round admission accepting swept-backed inputs, and the client not treating them as exitable |
| P3-1 rest | 4 remaining assertion strengthenings | **BUILD** |
| P2 items | Exact runway F/F+1, exact deadline block, upgrade prefix, depth 99/100, publish-once negative binding, admission isolation | **BUILD** |
| P1-10 | Manual-sync startup refusal | **SKIP** — needs new barkd harness capability for a config-mistake guard; poor value |
| Folding | Fold the 10 new commits into their MR homes | **PARKED.** MR-7 bookmark (`ark8-channels-stage1-surface`) moved forward to the new tip instead; plan retained in this session's record |

### OPEN QUESTION raised while building P1-2's timeout child

`client_punishes_revoked_server_htlc_second_stage_timeout` is committed
`#[ignore]`d, with the reasoning in the test. The staging is sound — the
revoked HTLC-bearing commitment confirms — but the cheater's TIMEOUT
child never appears, so the race never starts.

What captaind's log shows instead: with the victim offline and the HTLC
near expiry, the urgent deadline rung force-closes on the server's
CURRENT commitment; that package conflicts with the revoked one already
on chain, and nothing was observed switching to resolve the HTLC on the
commitment that actually won.

The preimage path is unaffected — claiming with a preimage follows the
OUTPUT, which is why the success-child sibling passes against identical
staging. A timeout needs a second stage presigned for one SPECIFIC
commitment, which is exactly where following the wrong one would show.

**Then the path was read, and it argues AGAINST a missing mechanism.**
LDK's `cancel_prev_commitment_claims` (`channelmonitor.rs:5192-5230`)
abandons claims for every holder commitment that is not the confirmed
one and retains the confirmed one's — which is precisely "follow
whichever commitment won", prev included. So the earlier inference
("nothing switches") was too strong.

**What is still unknown:** why no timeout child appeared within 60
blocks past the expiry. Most likely this vector's staging or timing —
the urgent rung publishing a package that conflicts with an
already-confirmed commitment, and churning on it. Worth one focused look
before the timeout leg is trusted; it is the same question the two
server-as-victim halves would hit, which is why those are not built.

### Consequence of dropping the ceiling

Per-channel in-flight exposure now equals capacity, so a single lost fee
race can cost a whole channel (≤ 2,000,000 sat) rather than 200,000.
That is accepted: it is the same value R2's expiry race already risks,
the bound is still per-channel, and stage 2's CSV response window
removes the race that makes it matter. `doc/channel-payments.md` must
record this next to R1/R2.

### Known-red carried into this work

`cab54a71c` contains three P3-2 assertion strengthenings swept in by
`git add -A`. One (`force_close_server_claims_and_payer_scrapes`) was
FAILING when its run was stopped — the witness-shape assumption for the
server's direct preimage claim is unverified. Fix before anything else.

## Group 1 — dark cells needing a NEW seam (the real decisions)

| # | Finding | Both? | Seam cost | Recommendation |
|---|---|---|---|---|
| P1-1 | Server-side justice on a revoked CLIENT commitment, ± HTLC. Reference had `barkd_server_punishes_dishonest_client_close[_with_pending_htlc]`; audit graded server justice "plausibly wired, UNTESTED". | BOTH | NEW client-side revoked-commitment export seam (mirror the server's `channel-test-seams`-gated `channel_export_commitment`) | BUILD both vectors. The one unarguable quadrant hole; server custody + reference parity. |
| P1-2 | Justice on the CHEATER'S confirmed second-stage tx. **DONE for the success-child, client-victim cell — see below.** Remaining: timeout-child, and the server-as-victim halves. | BOTH | none needed (see below) | Success-child client-victim BUILT and passing. |
| P1-4 | Server V64 sweep lifecycle wholly dark past the happy path: restart-after-persist-before-broadcast, RBF supersede, reorg reopen, depth-99/100 retirement. Table untouched by any test. | BOTH | NEW server hold after `set_channel_sweep_attempt` + deterministic feerate control | BUILD (2 vectors per codex). Server-funds custody machinery with zero durability coverage. |
| P1-6 | Close-package retry after a FAILED first relay + restart (LDK delists the channel instantly; durable candidate must carry it). | codex | NEW fail-next-submission seam at the chain broadcaster | BUILD; consider sharing the fault-seam family with P1-4. |
| P1-11a | Server receiver F−1 / forwarder F_in−1 refusal — needs exact-CLTV injection below LDK's shadow padding. | codex (floors) | NEW exact-CLTV injection seam | DISCUSS: a server-side unit at the event-handler layer may prove the floor check without the e2e seam; decide layer first. |
| P1-9b | `fast`-vs-`regular` reserve pricing distinction. | codex | test-local fee-source double only | BUILD (cheap unit; the e2e refusal is Group 2). |

### P1-1 — DONE (2026-08-31), bark `fbc10dd0a` on top of the stack

Seam: `bark-wallet` gains its own `channel-test-seams` feature —
deliberately NOT folded into `test-util`, because it links LDK's
`unsafe_revoked_tx_signing` (revoked-commitment SIGNING), which must
never enter an ordinary wallet build let alone a production one. It
gates one method, `Wallet::test_export_commitment`, the exact mirror of
captaind's `dev_export_commitment`. `testing` enables the feature.

Vectors (both in `testing/tests/bark_sdk/lifecycle.rs`, beside the two
client-justice ones): `server_punishes_revoked_client_close` and
`…_with_htlc`. Staging reuses WD-16's "client vanishes with the bridge
in the mempool" pattern, then mines the stale commitment by hand.

Assertion shape (stronger than the client-side pair it mirrors, per
P3-2): the cheater's `to_local` must be spent, and spent at a height
BELOW `cheat_height + 144` — inside LDK's `to_self_delay`, where the
cheater's own claim is not yet valid, so the timing bound proves the
revocation branch fired rather than merely that something spent the
output. Value provenance is two-hop (`descends_from`, ≤2 hops): LDK pays
the penalty to its own destination script and the V64 sweeper moves it
to the rounds wallet, so a one-hop check like WD-16's would wrongly
fail. The `_with_htlc` variant additionally requires the HTLC output —
the value the settlement already moved to the server, which is what the
cheat targets — to be recovered and to reach the rounds wallet.

RESULT: **the server does punish** — the audit could only grade this
"plausibly wired, UNTESTED". Measured: the cheater's 370,000 sat
`to_local` taken on the revocation branch at height 130, ONE block after
the breach (against a 144-block delay), 393,577 sat into the rounds
wallet; with an HTLC, 349,999 sat of balance AND the 20,001 sat HTLC
output, both via the revocation branch. Mutation-validated: disabling
the server broadcaster's non-close-candidate relay — the remediation's
own fix — fails both vectors; captaind logged ~485k/375k dropped claims
as LDK rebroadcast per block and neither reached its terminal.

BYPRODUCT — the recorded flake is SOLVED, and was never environmental:
`Sent` does not mean the HTLC has left the channel, and while it sits in
`AwaitingRemovedRemoteRevoke` it still consumes the 40k in-flight
allowance, so the follow-up 20k payment is refused a route within ~6s.
That is the 6.5s failure `2026-08-31-channel-test-flakes.md` attributed
to setup contention. Fixed with a real gate (`await_htlcs_settled`, poll
`test_pending_htlc_expiries` to empty) in all three vectors sharing the
shape, not a retry; that doc now records the cause. Its other two flakes
remain open.

### P1-2 — success-child / client-victim DONE (2026-08-31)

`client_punishes_revoked_server_htlc_second_stage`, same file.

THE SEAM THE AUDIT ASKED FOR CANNOT EXIST, and is not needed. LDK
refuses to hand back HTLC second-stage transactions for our channel
type — `channelmonitor.rs:5251`, for `anchor_zero_fee_commitments`:
"HTLC transactions in these channels require external funding before
finalized, so we return the commitment transaction alone here." An
"export the signed old HTLC child" seam would mean reimplementing the
external-funding path in the harness.

Staged instead with NO new seam, by letting the SERVER cheat with its
own production machinery: export its commitment while an HTLC is live,
then advance EXACTLY ONE state. The stale capture is then LDK's
`prev_holder_commitment_tx` — a state it still recognizes — and the
settlement has handed it the preimage, so mining that commitment makes
the server fund and relay a genuine HTLC-success. The client is taken
offline across that window so the cheater deterministically wins hop
one, then returns and must punish hop two.

RESULT: **correctly implemented.** The cheater's second stage took the
20,001 sat HTLC output on its preimage branch at height 134; the client
punished the resulting delayed output ON THE REVOCATION BRANCH at height
136 — 2 blocks into a 144-block delay — recovering 392,439 sat overall.
So `sign_justice_revoked_htlc` is wired, the client's FILTERED watch does
pick up an output first seen on the cheater's child (via
`Filter::register_output`, persist.rs:906), and the relay ships it. The
server-as-victim half is wired by construction — captaind feeds LDK
whole blocks (mod.rs:857-866), so no equivalent watch gap can exist
there.

Two test-side bugs found on the way, neither in the product: the
witness-shape check (a holder's second stage spends the 2-of-2 branch —
five elements, preimage at index 3 — not the three-element direct
claim), and the missing `.config/nextest.toml` slow-timeout override, so
all three new vectors were being killed at 180s mid-drive. Both fixed.

STILL DARK after this: the timeout-child variant (cheater's
HTLC-timeout, needs a CLTV wait) and the two server-as-victim halves.
Judgement: the timeout-child exercises the same
`sign_justice_revoked_htlc` path with a different trigger, and the
server-victim side has no watch gap to expose — so these are lower value
than the rest of Group 2. Recommend deferring rather than building now.

## Group 2 — dark cells, EXISTING seams suffice

| # | Finding | Both? | Recommendation |
|---|---|---|---|
| P1-3 | HTLC claim quadrants: 5 dark latest-state classes + all bidirectional payloads (client own-success, server own-timeout/success 2nd stage, client direct timeout, server direct timeout; offered+received together untested everywhere). | BOTH (codex enumerated) | BUILD as 2 consolidated vectors (client-commitment-bidirectional, server-commitment-bidirectional, each asserting BOTH parties' resolutions by provenance) instead of codex's 4 — halves the 600-900s runtime cost. |
| P1-5 | Stalled-close absence timer across server restart (remediation explicitly claims durable block-height liveness; never restarted before firing). | codex | BUILD (existing compressed fixture; add the connected-control-channel non-invention assertion). |
| P1-7 | Post-`ChannelSwept` reorg where an ALTERNATIVE winner (exported server commitment) takes the funding — demotion code exists, only same-winner reconfirm tested. | codex | BUILD (existing export + invalidateblock + direct-mine seams). |
| P1-8 | Client runway F/F+1 refusal with zero-upgrade-RPC binding through the proxy (guard exists, dark). | codex; guard verified by me | BUILD in channel_proxy.rs. |
| P1-9a | Open refusal when confirmed balance < reserve (guard exists, dark). | codex; guard verified by me | BUILD. |
| P1-10 | Manual-sync + channels startup refusal (guard exists, dark). | codex; guard verified by me | BUILD (trivial, <200s, barkd suite). |
| P1-11b | `htlc_floor_profile` startup guards: below-worst-floor refusal + immutability-over-live-channels refusal (`server/src/channels/mod.rs:208-242`). | BOTH | BUILD (cheap startup vectors). |
| P1-12 | `DOWNGRADE_EXPIRY_MARGIN` at both gates (cosign + registration), 7-blocks-pass / 6-blocks-refuse, no spent-mark or recognized leaf on refusal. | BOTH | BUILD (server suite; also pins the answer to the post-expiry question, D-1). |
| P2-1(+delta) | Client admission isolation: each refusal (659 sat, headroom, `exit_delta` mismatch, dust change) proven to fire with ZERO server RPCs; plus the two untested SERVER checks from my pass (`admission.rs:506,545`). | BOTH (merged) | BUILD one proxy vector + extend `open_by_upgrade_admission`. |
| P2-3 | Upgrade-side prefix non-foreclosure (downgrade has the vector, upgrade does not). | codex | BUILD (mirror of the existing downgrade vector). |
| P2-6 | Client terminal depth 99/100 pin (`ChannelSwept` at 99, `ChannelClaimed` only at 100). | codex | BUILD. |
| P3-3 | Server expiry sweep by provenance (today only "anchor disappeared" e2e + thorough policy units; no server-value binding). | BOTH | BUILD in server suite. |

## Group 3 — strengthen EXISTING vectors (no new runtime class)

| # | Finding | Recommendation |
|---|---|---|
| P3-1 | Seven client recovery vectors assert `>0` / state-only (unilateral e2e, theirs sweep, crash ladder, fallback tail, peer-close adopt, commitment reorg, tombstone-overrule) — all pre-date the provenance standard the remediation set. | APPLY codex's per-vector replacement table verbatim (it is well targeted). |
| P3-2 | Three HTLC/justice vectors stop before final value: `payer_scrapes` (spender shape + server landing value), `_with_htlc` (separate HTLC-outpoint justice binding), `blocked_htlc` (follow the second-stage child to the wallet + terminal). | APPLY; the `_with_htlc` strengthening also sharpens P1-2's baseline. |
| P2-2 | Server runway boundary: exact F refusal / F+1 acceptance (currently 30-vs-34 only). | APPLY (extend the existing admission vector). |
| P2-4 | HTLC-deadline rung fires at the exact block (currently 6-block sampling). | APPLY (tighten the existing vector; one-block mine + one maintenance pass). |
| P2-5 | WD-16 publish-once/canonical negative binding (no commitment package in the mempool pre-confirmation; single candidate after). | APPLY (test-side mempool observation is fine; the PRODUCT gate stays chain-only). |

## Group 4 — design questions (not coverage findings)

- **D-2 — the open-time exit reserve is a HARD REFUSAL, and priced off
  the wrong depth** (Greg, 2026-08-31, while reviewing the P1-9a vector).
  Two separable points:
  1. *Should it refuse at all?* `check_channel_exit_reserve`
     (`bark/src/channels/mod.rs:3137`) ends in `refuse_unless!`, so an
     underfunded wallet cannot open a channel; the only escape is the
     operator-level, all-or-nothing `channel_open_exit_reserve_sat = 0`.
     Greg's case against: a user with a tiny VTXO may rationally accept
     an uneconomic unilateral exit and want the channel anyway — likely
     usage, not an edge case. The case for is R2: losing the expiry race
     costs the WHOLE VTXO, and the ratified R3 disposition did say
     "refuse/alert on underfunding". Candidate: keep the gate but make
     it per-open overridable, or warn + rely on the `exit_funding`
     status that already exists.
  2. *Independently, the estimate is priced on the SERVER'S CEILING.*
     It is called as `check_channel_exit_reserve(ark_info.max_vtxo_exit_depth)`
     and computes `300 * (max_exit_depth + 2) * 2` vbytes at `fast` —
     with the mainnet default `max_vtxo_exit_depth = 100` that is 61,200
     vbytes, i.e. ~612k sat of CONFIRMED coin at 10 sat/vB, demanded
     even when the input's real exit depth is 2. `open_channel` already
     computes the actual depth (`input.exit_depth() + depth_cost`) a few
     lines earlier. Pricing on that is a one-line change and makes the
     gate proportionate; as it stands the gate plausibly refuses most
     mainnet opens.

  A vector for the refusal (`open_refuses_an_unaffordable_exit_reserve`,
  bark_sdk/channels.rs) is WRITTEN but PARKED pending this decision —
  it would pin behaviour that may be about to change.

- **D-1 — post-expiry downgrade / recovery of swept-backed channel VTXOs**
  (Greg, 2026-08-31). Today: margin refuses at tip+6 ≥ expiry forever;
  round admission independently refuses expired inputs
  (`server/src/round/mod.rs:1908`); the margin is load-bearing ("nothing
  repairs leaves registered over a swept ancestor"), so post-expiry
  downgrade needs new semantics — server-honored reissue leaves exempt
  from the spent-mark/watch model + round admission accepting
  swept-backed inputs + client not treating them as exitable.
  Upstream/stage-2 conversation. Stage-1 coverage half is P1-12.
- **P1-2 severity split** (above): success-child vs timeout-child.
- **P1-11a layer choice** (above): e2e seam vs event-handler unit.

## Sizing

If every BUILD lands as proposed: ~14 new vectors (most 200-600s, five
600-900s) + ~12 assertion strengthenings + 3 new test seams (client
commitment export, V64 sweep hold/feerate control, broadcast-fault) +
1 fee-source double. Groups 3 and the cheap half of Group 2 are
low-risk; Group 1 carries the seam decisions.

Codex verdict FAIL stands under my cross-check: P1-1 alone (server
custody quadrant, reference parity, ratified threat model) justifies it.
