# MR-4 series-level codex review (2026-08-08)

After all four commits individually converged (c4 in 7 rounds), a
series-wide adversarial pass — read the MR the way an upstream reviewer
will — following the MR-3 precedent. Dispatched read-only at max effort
against `d18d33a1..2155b539`.

## Round 1: HOLD — one ship blocker + coherence findings

**P1 (ship-blocking) — commit 4 mutated the already-applied `m0044`
migration.** Commit 3 creates `bark_channel` without
`pinned_claim_slack`; commit 4 edited that same CREATE TABLE to add it.
The migration runner executes only at exact version matches, so a
database at version 44 skips the edited migration and later queries
select a missing column. Exactly the class of bug the series pass
exists for — per-commit review cannot see it (each commit is internally
consistent; only the upgrade path between them breaks), and the battery
never exercises c3→c4 upgrades.

*Fixed:* m0044 restored to c3's shape; the column arrives by
`ALTER TABLE` in m0045 (commit 4's own migration, matching its existing
ALTERs) with `DEFAULT 65535` — the fail-safe direction: slack feeds the
deadline floor (`max exit depth + exit delta + slack`), so an
understated default would compute a too-late deadline, while the u16
maximum makes any pre-existing row's floor unsatisfiable and forces the
conservative path (immediate unilateral exit). `MigrationContext` grew
a test-only `up_to(version)` cap, and a regression test builds a
database exactly as commit 3 leaves it, inserts a channel row, upgrades,
and asserts the back-fill.

**P2 — the broadcast-queue contract was false in the final design.**
Docs and the c1/c4 commit messages still claimed LDK-originated
transactions are drained from the queue into the exit's transaction
manager; the shipped design never reads the queue back (the commitment
comes from the close-event capture; sweeps are built from persisted
descriptors). The drain machinery itself was production-dead: its only
consumer was pump's own hygiene loop.

*Fixed (option a, capture-only):* the queue is now explicitly a
capture-only audit — `drained_at`/`swept_at` columns and the six dead
store methods (`undrained_broadcasts`, `mark_broadcast_drained`,
`unswept_spendable_outputs`, `mark_spendable_output_swept`,
`undrained_channel_close_events`, `mark_channel_close_event_drained`)
removed **in the commits that introduced them** (all four commits are
unpushed, so debris folds where it arose: c1 never creates them); pump's
hygiene loop deleted; `ChannelBroadcast.sweep_kind` became mandatory and
is **derived from the descriptor's shape** (DelayedPayment → ToLocal,
StaticPayment/Static → StaticPayment) instead of hardcoded ToLocal; the
exit's dead commitment/funding-spend filters on pumped items dropped.

**P2 — close-event identity not enforced on upsert.** LDK re-emissions
for a claim id could overwrite channel, counterparty, and commitment
under the same key, violating the design note's seam-audit obligation.

*Fixed:* the upsert now verifies the declared txid hashes from the
carried bytes, and a re-emission under an existing claim id must match
the stored `(channel_id, counterparty, commitment_txid)` tuple —
mismatch refuses, never overwrites. Regression test covers renamed
channel, swapped commitment, and inconsistent bytes.

**P2 — abandoned outputs remain terminal obligations (docs said
excluded).** Shipped behavior is the intentional fail-closed design
from the c4 arc (abandonment suppresses sweep attempts, never terminal
accounting; revisit-on-rates-fall is the recovery policy). The design
note §8 and the store docs said the opposite (a dust write-off design
that was never built). *Fixed in the docs* — the note's §8 now
describes the layered terminal barrier as shipped, including "no
batching, no dust write-off."

**P3 — stale column/state debris.** `swept_at` (no runtime authority
since the state-derived rework) and the close-event/broadcast
`drained_at` columns: removed at their source (c1's m0043), per the
pre-release no-legacy rule. schema.sql regenerated per commit.

**Commit messages.** c1's queue paragraph rewritten to the capture-only
contract; c3 "Five phases" → "five states" and the milestone sentence
replaced with "while payments remain disabled in this client"; c4's
pump/terminal paragraph rewritten around the shipped barrier (ledger +
floor + health gate + state-derived subtraction) per codex's suggested
replacement.

**Jargon sweep.** All plan-phasing language ("milestone", "stage-1",
"bridge/commitment stage", one pre-existing "Pre-MR" in a touched file)
replaced with invariants or state vocabulary across code, migrations,
bark-json (OpenAPI regenerated), tests, and messages.

**Placement audit (no change):** codex confirmed keeping the node
safety primitives (`driver_running`/`is_healthy`) and the
ledger/pump in commit 4 is appropriate — they are only meaningful once
terminal accounting exists.

**Design-note reconciliation:** §8 amended (follow-up commit — the
note's original commit is pushed and immutable) to the shipped design:
close-event capture table, capture-only queue, self-fee-paying direct
sweeps, abandoned-included accounting, and the five-layer terminal
barrier. §9 was confirmed consistent as written.

**Found during this pass's own battery — the usability gate consulted
only LDK.** `open_channel_restart_matrix`'s gate assert ("no channel may
be usable before readiness releases it") failed under machine load (and
passed quiet — the gap predates this round; load merely widened the
window until it showed): the
peer's `channel_ready` is pre-queued server-side, so the instant the
client's sanctioned confirmation feed lands, LDK completes the readiness
exchange (~26ms in the captured trace) while the record's
`registered → ready` flip waits for the next drive pass —
`channel_is_usable` returned LDK's `is_usable` alone, so the wallet
reported a channel usable that its own bookkeeping had not released.
The stated gate (test comment and design note) was always
record-released usability; the implementation missed the conjunct.
*Fixed in commit 3 (which owns the surface):* `channel_is_usable` now
requires `record.state == Ready` **and** LDK usability — deterministic
under any load, and `Ready` being trailing bookkeeping merely delays
reported usability by one drive pass.

## Round 2: HOLD — behavioral fixes all verified; wording residue

Every behavioral fix verified clean: "m0044 is unchanged after commit 3;
m0045 safely backfills … schemas match each tree"; capture-only removal
complete with no surviving consumer; "pump filtering removal is safe for
the shipped path"; close-event identity and same-ClaimId re-emission
correct; "record-released usability gating is correct; readiness
recovery re-feed does not weaken it"; `up_to` cannot skip a production
migration. The HOLD was residual comment wording my sweep missed: the
`queue_broadcast` impl still described a drainer lifecycle,
`SpendableOutputRow`'s doc said "unswept", and two comments used
plan-flavored "payments disabled" instead of the deployment invariant.

*Fixed with codex's suggested wording, folded to the owning commits*
("captured and never consumed here"; "One persisted spendable output";
"this deployment has no supported HTLC-resolution path"; m0046 grounds
the single-output identity in the enforced facts — client funds the
whole channel, push refused, no HTLC output can exist — plus one more
"payments disabled" in fees.rs swept for consistency). The delta is 38
comment-only lines across 5 files; 124 units re-confirmed at the new
tip; the e2e battery stands from the round-2 tip (trees differ only in
comments).

## Round 3: SHIP

"Verified all five sites against the shipped contract. The round-3 diff
is comment-only: 5 files, 15 additions/13 deletions, all changed lines
comments; `git diff --check` is clean. Vocabulary sweep found no
series-added `drainer`, `unswept`, `milestone`, `stage`, or `payments
disabled`; remaining `drain`/`swept` uses are operational or state
terminology. VERDICT: SHIP"

## Final state

Series tip `952bd9ba` on `ark8-channels-stage1-client`
(c1 `cb89d366`, c2 `7a7e3f71`, c3 `3440348a`, c4 `952bd9ba`).
Battery: 124 bark units at tip (two regression tests added by this
pass); 6/6 channel SDK e2e green at the behavior-identical round-2 tip
(restart matrix 3x consecutively after the usability-gate fix; the two
heavy exit tests run to completion — their >180s nextest overrun is
pre-existing bdk per-block sync latency, noted for a future
slow-timeout override); per-commit bark-wallet units 120/120/122 at
c1/c2/c3's own trees.
