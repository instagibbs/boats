# Codex G1 record — MR-7 surface + lifecycle e2e design note

**Subject:** `2026-08-27-mr7-surface-design.md` (renamed from the
parked rev1 `2026-08-25-mr6-surface-design.md`), reviewed against
bark-stage1 @ `a491cea7f` (the pushed payments tip).
**Verdict: PASS in 6 rounds** (rev1 parked → rev2 self-revision →
rev3/rev4 + two amendment rounds under review). Every finding was
adopted, none refuted; codex supplied the winning alternative on the
three hardest (the action resource, the stranded seam, the pre-release
convergence fix).

## What each round changed

- **r1 (12 findings):** the note's rev2 surface exceeded the shipped
  wallet API on four fronts — open is action-shaped (no node address,
  no synchronous ids), keysend is unserviceable without route
  discovery (EXCLUDED from the surface, decision 5), invoice
  pay/create capabilities were under-specified (amountless override +
  description added), and stranded is NOT re-payable (decision 7:
  terminal-until-authoritative; fresh invoice otherwise). Plus: the
  expiry matrix claim was WRONG (funding is deliberately
  non-refreshable — the lifecycle is deadline-close→reopen, docs
  language fixed); BlockedHtlcClaim had no surface (now in the status
  DTO + a top-up vector); the c3 staging lacked deterministic seams
  (six enumerated, test-gated); the reorg vector mismatched shipped
  lease semantics (payer-retry, no re-admission); the c4 crash-cell
  list was stale (pruned; survivors labeled proxy-layer regressions);
  restart coverage misses at the earliest phases (Establishing,
  Negotiating) and failed→same-invoice retry.
- **r2 (6):** the production binaries are NOT channel-enabled
  (feature propagation + `OpenWalletArgs.channels` wiring + one-shot
  driver lifecycle became explicit c1/c2 scope; executable CLI smoke);
  the open action needed a REAL pollable resource
  (`202 + /channels/opens/{action_id}`, terminal failure observable);
  seam (b) became a composite barrier over manager persistence + the
  terminal payment events (hard crash, pre-restart blob asserted);
  pre-release close/exit arcs joined the matrix; seam (f) exports the
  commitment only (captaind's claim relays itself); the HTTP error
  table pinned (400/404/422/503/500, downcastable wallet refusal
  categories).
- **r3 (2):** the Cosigned forced-exit vector was unreachable as
  written (row SPLIT — peer closes at Cosigned, HTLC-deadline exits
  only at Registered; no seeded pre-release HTLCs); the client
  claim-binding crash cell was missing (CAS→claim_funds hard crash,
  same-claim-id re-admission, exactly-once `received`).
- **r4–r5 (1 each, the real find of the review):** a pre-release
  cooperative close STRANDS THE OPEN ACTION in shipped code —
  `outcome_ready`/`downgraded` are neither accepted nor terminal for
  the Registering/Feeding phases, and the deeper interleave (r5) has
  the hard-line exit winning before registration, the server then
  permanently refusing release while the open consults local terminal
  states only AFTER the doomed RPC. The note now carries the
  CONVERGENCE FIX as explicit c3 scope: the open action consults
  close/exit-owned terminal states BEFORE attempting registration and
  reconciles terminally without the RPC; the vector proves both
  interleavings.
- **r6: PASS.**

## Shape of the MR as ratifiable

Five commits: c1 DTOs + REST `/channels` + binary enablement + the
error contract; c2 CLI noun (executable smoke); c3 lifecycle e2e
completion (six test-gated seams, the force-close/claims/Theirs/
backwards-scrape staging, the crash matrices, the convergence fix);
c4 the ArkRpcProxy adversarial sweep; c5 operator docs. Nine
decisions await Greg's ratification (note §Design decisions),
notably: action-oriented open (2), keysend not surfaced in stage 1
(5), stranded never re-payable (7), collect stays admin-only (8),
test-gated seams (9).
