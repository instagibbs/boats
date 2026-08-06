# Codex review record — MR-3 series-level review (the final gate before posting)

**Subject**: one pass over the whole captaind-channels MR as a shippable
unit — four commits on `ark8-channels-stage1-captaind`, base = the MR-2
protocol tip `1ddde4ad`. Each commit was already reviewed to PASS
individually (3 / 5 / 5 / 9 rounds); this pass judged cross-commit
coherence, the seams, spec-vs-shipped drift, and upstream readiness —
what the per-commit passes structurally could not see.

**Verdict**: HOLD → resolved (a confirmation pass then caught two doc/message stragglers of the Blocker fixes — the code was already correct — now also fixed). 2 Blockers, 2 Should-fix, 1 Nit; the Nit's
"embedded tabs" half was a codex misjudgment (verified below). All
addressed; the series is now clean.

## Findings and resolutions

1. **Blocker — the fee-bump reserve was dead across the series.**
   `FeeBumpReserve` (commit 1, `bark-channels/src/bump.rs`) and its three
   config knobs (commit 2) had no production caller: the watchman selects
   wallet funds on demand for the channel sweeps, like every other sweep.
   Commits 1/2 promised an invariant commit 4 never composed with.
   **Resolved (Greg's call): removed.** The whole `bump` module, the
   config knobs + validation, the default-TOML lines, and the spec claims
   are gone; channel fee-bumping rides the watchman's existing wallet
   CPFP. Justification: in stage 1 a channel holds ZERO server balance
   (`push_msat == 0`, no payments), so the parent-exit response and expiry
   sweep protect no server funds — a dedicated always-funded reserve
   guards nothing yet. Part 4 (payments → server balance) is where the
   reserve earns its keep and is built; nothing safety-relevant is
   deferred.
2. **Blocker — the design note over-claimed the registration handoff.**
   The note (and commit 4's message) described a "cursor handoff" that
   required the listener to have indexed the exact height+hash the screens
   saw. The CODE uses a `scan_epoch` guard instead, and its safety comes
   from the bitcoind-DIRECT screens (final-exit `tx_status` + the
   exit-chain viability walk — both query bitcoind, so a lagging listener
   cannot hide a foreclosure) plus the epoch recheck under `chain_lock`.
   The code is SAFE; the doc was wrong. **Resolved**: §5 and the commit
   message rewritten to the scan-epoch mechanism.
3. **Should-fix — the capture broadcaster was an unbounded dead sink.**
   Every LDK broadcast was appended to a `Vec` forever, with a dead
   `captured()` and a stale field. **Resolved**: log-and-drop; the
   retention, the API, and the subsystem's dead field removed (the node
   keeps its own broadcaster via the seams).
4. **Should-fix — the note carried superseded shapes.** Config (a single
   reserve knob vs the removed pair), schema (a plural `ldk_channel_monitors`
   and `pinned_* NOT NULL from birth`, both contradicting the shipped
   singular table + nullable-until-cosign CHECK). **Resolved** alongside
   finding 1.
5. **Nit — stale/odd comments.** `event.rs` said payments are refused
   "permanently" (conflating the permanent no-unroll invariant with the
   stage-1 refusal — corrected to name part 4). `watch.rs` computed and
   discarded `closed_any` (a vestige of the removed persistence barrier —
   removed). The claimed "literal tabs in a log message" was a
   **misjudgment**: the message uses a `\`-continuation, which strips
   leading whitespace per Rust's string-continuation rule (verified
   empirically with a standalone compile) — no tabs leak, no change.

## Judged clean at the series level

The crate → dormant scaffold → admission → release staging is coherent
and independently buildable (`cargo check --all --tests` at each). The
lifecycle abstractions (dormant start → activate, `chain_lock` /
`scan_epoch`, `ManagerStore`'s sequential-writer contract, the open/reaping
states, authenticated withholding, the release latch, the parent watch,
reorg reopen, terminal reconciliation, channels⇒watchman) compose across
the commits with no vestigial shapes left. The migration + regenerated
schema follow the house conventions (singular tables, synthetic-id PKs,
hex TEXT, timestamp markers). The stage-1 posture is airtight: no
send/invoice API, no reachable `claim_funds`, every `PaymentClaimable`
failed back, no relayed LDK broadcast; the one accepted residual (a
foreclosed channel is closable one block later, reactive detection,
benign under no-payments) is part 4's. No internal MR/round/codename
leakage in code, comments, or the commit messages.

## Confirmation pass (round 2)

A narrow re-review confirmed S3/S4/N5 closed but found B1 and B2 had
DOC/MESSAGE stragglers the code fixes missed: commit 2's message still
claimed "per-tranche fee-bump reserves" and the `channels/mod.rs` module
doc still named a "fee-bump reserve ledger" (B1); commit 4's message
still described the "exact-tip cursor handoff" its own body contradicted
(B2). All three were text-only (the code was correct) and are now fixed
in the owning commits via change-ids. (A jj divergence occurred applying
them — foreground `jj describe` of hashes a background per-commit-check
task had concurrently rewritten — recovered with `jj op restore`; lesson:
never run foreground jj mutations while a background jj task runs, and
address commits by stable change-id, not commit hash.)

## Post-resolution verification

`cargo check --all --tests` green at the tip and at all four (rebased)
commits standalone; units 511/511 (down from 524 — the 13 removed tests
were `bump.rs`'s reserve suite, gone with the deleted module); clippy
ark-lib clean, bark-server at the 236 baseline, bark-channels clean; the
full channels e2e battery green (admission matrix, real establishment,
parent-exit watch lifecycle); the lightningd-gap slice failure is
pre-existing and unrelated.
