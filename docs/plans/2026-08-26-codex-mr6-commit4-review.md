# MR-6 commit 4 — payments state and policy: codex review record

Commit: `844e31938` "payments state and policy: the journal, the floor
plumbing, the exposure profile" (bark-stage1, ark8-channels-stage1-close).
Reviewer: codex (read-only, pinned LDK 0.2.4 citations required).
Verdict: **PASS at round 13** (r13: "No findings"). Battery green on the
final tree and on every intermediate amend: 632 workspace units /
bark-channels 23 / bark-sdk channels 19 / server channel 22.

This was the longest convergence of the MR (13 rounds vs c1's 3, c2's 10,
c3's 5+8-durable). Two structural events shaped it: the **floor got bound
into the cosign protocol** (rounds 3, 9–12 kept finding config-change
windows until the binding covered declare → admit-snapshot → fresh-only
verification), and the **startup-recovery subsystem was deleted outright**
(rounds 4–9 patched it four times before Greg ratified the simplification;
the round-9 attack then became unrepresentable).

## Round history

| round | findings | disposition |
|---|---|---|
| 1 | 9 | replayed-event corruption, unadvertised absolute in-flight cap, ExactReplay ignoring evidence, command-time movement strands, PK collision across directions, keysend floor source, floor candidate set, +3 pad (ratified), stale schema — all fixed |
| 2 | 7 | cursor CAS on mirrored state; sweep-owned movements via action id; sent-replay preserving mirrors; barrier→startup-recovery re-issue; capacity ceiling in the conformance predicate; composite-PK schema snapshot; intended balance |
| 3 | 4 P1 | state-free action id + figures-at-finish (ONE movement per payment); clean-pass recovery latch; **floor tuple bound into the bridge cosign** (proto fields 9/10 + admit snapshot + cosign verify + ExactReplay); MAX_PAYMENT_MSAT + checked conversions |
| 4 | 2 | sweep single-writer mutex; recovery-vs-disconnected-peer (RouteNotFound classification) + **failed send rows replaceable in place** |
| 5 | 4 | send commands share the sweep mutex (ABA); replacement defers to LDK's `list_recent_payments`; recovery snapshot (stops absorbing live rows); accept-side evidence in ExactReplay |
| 6 | 3 P1 | construction-time worklist capture (ids LDK never saw); receive same-state ABA → **first-claim freeze** (`state='pending'` guard on mark_received) |
| 7 | 1 P1 | per-id consumption of the worklist |
| 8 | 2 P1 | consume-on-replacement; server forwarding config → **F\*-derived per-channel delta pinned now**, acceptance flip deferred to the enablement gate |
| 9 | 2 P1 | **open_floor_profile joins the snapshot trio** (LDK clones F\* into the channel config at accept — a profile change mid-open must refuse at cosign); worklist-vs-replacement corruption → **THE SIMPLIFICATION** (below) |
| 10 | 1 P1 | committed-cosign replay wedged by current-config checks → **fresh-only screens** (declared pair, snapshot trio, depth/floor/profile/runway) |
| 11 | 1 P1 | `max_arkoor_fanout` validated before replay classification → two-pass validation (structural → classify → policy for fresh only) |
| 12 | 1 P2 | replay exemption covered the package, identity covered the part → **single-part upgrade packages, unconditionally** |
| 13 | none | **PASS** |

## The simplification (round 9, Greg's call)

The startup-recovery re-issue subsystem had accumulated four patches
(clean-pass latch → construction capture → per-id consumption →
consume-on-replacement) and codex kept finding lifecycle gaps — the same
smell c3 had before the trust-LDK pivot. Greg ratified deleting it:

- **No automatic re-issue.** At node construction — the one provably
  quiescent moment (manager reloaded, driver not started, no commands
  possible) — every pending send row whose id LDK does not track is
  marked FAILED: it was journaled but never issued; the crash cut the
  send. The wallet reports it honestly and the user retries explicitly
  through the failed-row replacement seam.
- Rationale: a wallet must not silently pay an invoice minutes after a
  restart with no user in the loop, and every re-issue design shadows
  LDK's attempt lifecycle (LDK drops an exhausted id BEFORE queuing its
  terminal event — any later re-issue is a potential resurrection).
  Everything LDK tracks is LDK's: its events settle the journal.
- Deleted: the worklist field and capture, the recovery branch of the
  sweep, the recovery flag + RouteNotFound classification on both issue
  paths, the one-shot latch. The round-9 attack (a replayed old
  PaymentFailed corrupting a replacement through the worklist) ends at
  construction now: the crashed replacement is failed, the replay
  no-ops on the already-failed row, the user retries.
- LDK backing (codex-verified r10): tracked payments are exposed via
  `list_recent_payments` (channelmanager.rs:4289) and monitor-only sends
  are reconstructed on stale-manager reload (channelmanager.rs:17513),
  so "untracked at construction" really means "never reached the wire";
  a mislabel in the residual reload window heals by sent-dominates.

## Key decisions for Greg's review

1. **The simplification** (above) — ratified in-session.
2. **Failed send rows are replaceable in place** (the one seam in
   hash-single-use): a retry of the same invoice re-arms the row, resets
   the movement link, and the payment's single movement converges on the
   final outcome. The replacement refuses while LDK still tracks the id.
3. **The floor is bound at the cosign seam**: the bridge cosign request
   declares the client's timing pair (proto fields 9/10, group-presence
   enforced); the server refuses a fresh cosign whose pair differs from
   current config; the admit snapshots the trio (depth, slack, **and
   F\*** — LDK clones the profile into the channel's own forwarding
   config at accept); the cosign verifies current config == snapshot;
   ExactReplay compares everything.
4. **Committed-cosign replays are sacred**: every current-config screen
   (declared pair, snapshot trio, depth budget, floor fit, profile
   bound, runway, fanout/depth policy validation) binds FRESH cosigns
   only. A cosigned-same-backing retry is the recovery of an
   already-committed cosign — refusing it wedges spent value behind a
   refusal the client reads as pre-commit. Validation is two-pass
   (structural → classify → policy-if-fresh).
5. **Upgrade packages are exactly one part** — every legitimate flow
   spends one input, and the bound is what makes the replay carve-out's
   identity (the reconstructed part) cover the whole package.
6. **Server forwarding delta = F\***, pinned in the node's channel
   config now so pre-enablement channels already carry it. The
   `accept_forwards_to_priv_channels` flip stays OFF until the
   enablement gate (c5) lands with the capability checks.
7. **Receives freeze at the first claim** (mark_received settles only
   from pending): LDK can deliver a second PaymentClaimed for a
   paid-twice hash. Single-claim ENFORCEMENT (refusing the second
   payment before it is claimed) is the c5 claim seam's.
8. The sweep and the send commands share one single-writer mutex; the
   cursor CAS handles event flips within a pass, the mutex removes
   pass-vs-pass and replacement-vs-pass interleavings.
9. §3.7 exposure profile as handshake constants both sides + capacity
   ceiling (LDK's in-flight knob is percent-only); evidence persisted at
   OpenChannelRequest (not re-derivable later); conformance predicate
   fail-closed. PaymentForwarded = at-least-once telemetry, never
   summed.

## Debts carried to c5 (enablement)

- `accept_forwards_to_priv_channels` flip, gated on the capability
  checks (decision 6).
- Single-claim enforcement at the claim seam before any claim
  (decision 7), with the claim-seam floor and the fail-backwards rules.
- The HTLC deadline rung (non-cancellable, level-triggered).
- The collect leg (B→captaind) and capability-gated acceptance flip.
- Payment e2e matrix (c6): real sends/receives, floor boundary vectors
  (F vs F−1), crash cuts at every journal/movement boundary, hostile
  MPP, HTLC force-closes both directions, and the replay-recovery
  vectors this review pinned statically (committed-cosign retry under
  changed config; crash-cut send failing at construction).
