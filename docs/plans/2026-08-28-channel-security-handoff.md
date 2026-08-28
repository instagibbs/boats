# ARK #8 channels stage-1 — security-remediation HANDOFF

Status at handoff: the MR-7 test/surface stack (c1–c4) is implemented and
codex-PASSED per commit; a Bitcoin-transaction-level security audit
(five parallel code audits + two adversarial codex passes, converged)
found money-safety gaps that PAUSE the whole-MR-7 pass and the push.
This document is the durable handoff so the remediation can resume with
no re-derivation. Authoritative findings:
`2026-08-28-channel-security-audit.md` (§"Combined findings — FINAL").
Justice detail: `2026-08-28-channel-justice-gap.md`.

**PLAN STATUS 2026-08-28: RATIFIED by Greg — the disposition table below
is final; ready for EXECUTION.** Fix stage-1: GAP 1/2/3/4/9 (client
relay + server sweep + server force-close) and R3 (exit-fee reserve).
Accept + document: R1 (bounded per-channel HTLC race, CSV fix is
stage-2) and R2 (whole-VTXO expiry race, mitigated by the `F`
exit-lead + R3). Fix the spec: C1 (`static_remote_key`). Punt (debt):
GAP 5, dust, P2A/Esplora.

## Where the code is

- bark branch `ark8-channels-stage1-payments`, tip `28c85fff2`.
  Pushed tip = `origin/ark8-channels-stage1-payments` = **`a491cea7f`**
  (MR-6 payments). UNPUSHED (MR-7): `92b297cd2`, `cdfe88223` (c1),
  `cf423cb8a` (c2), `a1efa93fe` (c3), `28c85fff2` (c4).
- boats records (jj bookmark `stage1_upstreaming`): the per-commit codex
  records incl. c1–c4 and this audit set. UNPUSHED.
- Reference impl (has what stage-1 regressed): bark branch
  `ark-channels-bridge-2026-06-18` (+ `bark-lightning-channels` for the
  `channel_justice.rs` test and the 4-quadrant `barkd_*_punishes_*` matrix).

## The converged gap list (see the audit doc for file:line + scenarios)

Money-safety (all REGRESSIONS from the reference; 3 root causes):
- **GAP 1 [HIGH, user funds]** client JUSTICE unwired (+ HTLC-output
  justice, ex-GAP-8) — root cause A.
- **GAP 4 [HIGH, user funds]** client can't claim an HTLC off the
  server's commitment — root cause A. (Gated on an adversarial server
  racing a retained commitment onto the client's bridge during a client
  exit — the only way a server commitment reaches chain.)
- **GAP 9 [MED/HIGH-combined, user funds]** cooperative-close exit state
  never adopts a foreign commitment winner (no `obtain_commitment`
  reselect) — root cause A area; restart-compounding.
- **GAP 2 [HIGH, server funds]** server never sweeps its own
  to_local/to_remote (no `SpendableOutputs` handler) — deterministic on
  any client exit — root cause B.
- **GAP 3 [HIGH, server funds]** no production post-actualization
  force-close (admin RPC → mainnet-refused dev seam) — root cause C.

Bounded residual protocol risks (decisions, not just code):
- **R1 [HIGH residual]** no-CSV HTLC fee-race (bounded by exposure caps,
  per-channel; multiplies across channels).
- **R2 [HIGH residual]** expiry-boundary fee-race — server expiry sweep
  can RBF-replace an unconfirmed bridge → whole VTXO lost.
- **R3 [HIGH residual]** exit-fee reserve is advisory/unreserved/
  underpriced — makes R2 reachable (drained/underfunded/manual-sync).

Conformance / low:
- **C1 [MED]** `static_remote_key` profile mismatch (spec vs impl).
- **GAP 5 [LOW]** reorg test coverage debt; dust bounded-loss [LOW];
  P2A/Esplora liveness [LOW].

Confirmed NOT gaps: downgrade/no-forfeit safety, exit ordering,
coop-close/downgrade-PONR/pre-bridge expiry (wired+tested), multi-channel
forwarding, channel-vs-ARKOOR preimage domains, negotiation, SCID/gossip,
the non-round `store_vtxo_transactions` ingress. Server submitting the
bridge = stage-2.

## Dispositions — RATIFIED by Greg 2026-08-28 (⚠ = awaiting confirm)

| Item | Disposition | Status |
|---|---|---|
| GAP 1, 4, 9 (client relay) | **FIX stage-1** (root cause A) — user-fund custody + penalty deterrence. GAP 4 = get the client's HTLC claim ON the network; the residual timing race is R1 (below). | RATIFIED |
| GAP 2 (server sweeps its own balance off a mined commitment) | **FIX stage-1** (root cause B) | RATIFIED — "servers do carry balance, they need to recover if a user griefs by getting the commit tx mined" |
| GAP 3 (server force-close when the client actualizes the bridge then vanishes) | **FIX stage-1** (root cause C) | RATIFIED — "…or even just [the] bridge tx" |
| R1 (no-CSV HTLC fee-race, both directions) | **ACCEPT** — stage-1 does not handle it; exposure is the bounded per-channel value per spec. The CSV Ark channel type that closes it is **STAGE 2**. Document it. | RATIFIED — "we will fix in stage 2" |
| C1 (`static_remote_key` bit) | **Impl is authoritative** — we use zero-fee-commitment channels; take whatever the zero-fee type dictates (LDK clears the bit). **FIX THE SPEC** to match. | RATIFIED — "spec seems wrong" |
| R3 (exit-fee reserve durability) | **FIX stage-1** — durable earmark, `fast` pricing, refuse/alert on underfunding, don't let manual-sync disable deadline exits. | RATIFIED |
| R2 (expiry fee-race → **whole VTXO**, distinct from R1's bounded HTLC value) | **ACCEPT + document.** The mitigation is the `F` exit-lead: the client is expected to start its exit early enough that the bridge confirms before expiry — "if they're doing that, it's fine." R3 ensures the client can fund the bridge; the residual (a miner not consensus-forced to prefer the bridge) is accepted. | RATIFIED |
| GAP 5 (reorg tests), dust, P2A/Esplora | Coverage/liveness DEBT — punt, non-blocking, schedule later. | RATIFIED (punt) |

## Remediation → root cause → fix → fold home (all homes PUSHED → force-push)

- **Root cause A (client relays LDK's self-funded claims):** mirror the
  c3 server `CaptureBroadcaster` (relay non-close-candidates) onto the
  client `QueueBroadcaster`; construct the `Justice` sweep; add the
  coop-close `obtain_commitment` reselect (GAP 9). Stock-LDK-safe, no
  fork. Fixes GAP 1/4/ex-8/9. Home: the client broadcaster/claim/exit
  machinery — introduced at `91da6a233` (channel node) / `8efd74533`
  (HTLC claim funding) / the exit state machine. Tests: port the
  reference's `barkd_client_punishes_dishonest_server_close[_with_htlc]`
  + a client-claims-HTLC-off-server-commitment vector + a coop-close-
  foreign-winner vector.
- **Root cause B (server `SpendableOutputs` sweep):** add the handler +
  sweeper. Home: server channels (`aac2675f6` scaffold / the MR-6 server
  claim commit). Test: server recovers its `to_remote` on a client exit.
- **Root cause C (server production force-close):** a real
  post-actualization force-close (not the dev seam) triggered when the
  server carries balance/HTLCs and the client's bridge has confirmed.
  Home: server channels. Tests: port `barkd_server_punishes_dishonest_
  client_close[_with_htlc]` + a WD-16 recovery vector.

Pick the EXACT introducing commit at fix-time (earliest commit whose
behavior is corrected, so the bug never appears in shipped history).

## Commit mechanics (agreed with Greg)

- Each fix = its own commit, FOLDED into its logical introducing commit
  (not a follow-up) — clean history, no "fixed-later" seam.
- Force-push is safe: nothing upstream is under review yet.
- Folding into a pushed commit cascade-rebases the stack above it → RUN
  THE PER-COMMIT BATTERY down the rewritten chain, not just at the tip.
- The boats per-commit records + MEMORY.md cite bark hashes that a
  rewrite changes → refresh those references in the same pass.
- No plan jargon (MR numbers / gate names) in code, filenames, or
  commit messages.
- Battery: `$SCRATCH/battery.sh` (env in it). CLN not installed (2 barkd
  recovery vectors fail environmentally — known).

## Resume checklist for whoever picks this up

1. Read `2026-08-28-channel-security-audit.md` (Combined findings) — the
   authoritative gap list + file:line.
2. Dispositions are RATIFIED (table above) — no decision needed; the
   fixes to build are GAP 1/2/3/4/9 + R3. R1/R2 are accept+document
   (add the operator/user notes to `doc/channel-payments.md`), C1 is a
   spec edit to `08-channels.md`, the rest is debt.
3. For each ratified fix: find the introducing commit, fold the fix +
   its tests, cascade-rebase, re-verify each commit's battery, refresh
   the boats-record hashes, then one clean force-push.
4. Re-run the whole-MR-7 pass (was paused for this).
5. Update the memory `ark8-bark-upstreaming.md`.
