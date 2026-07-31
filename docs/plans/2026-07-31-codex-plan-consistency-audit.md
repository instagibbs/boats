## Findings

1. **Severity: Important — the series has two incompatible counts and numbering schemes.** The plan still says “five upstream MRs” while now describing the draft opener plus the old MR-1..5 stack—six artifacts ([plan §5](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-bark-upstreaming-plan.md:267)). The draft instead lists five and compresses close-by-downgrade and surface/hardening into item 5 ([draft](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-30-draft-mr-description.md:59)). The G1 note and discharge table retain pre-pivot numbers ([G1 note](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-31-mr1-protocol-surface-design.md:1), [table](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-bark-upstreaming-plan.md:461)).

   Fix: use one mapping everywhere: old MR-0→MR-1 opener, old MR-1→MR-2 protocol, MR-2→MR-3 captaind, MR-3→MR-4 bark, MR-4→MR-5 close, MR-5→MR-6 surface. Apply that mapping mechanically to every §7 owner and rename the G1 note. Make the draft a six-item list, separating close from REST/CLI hardening.

2. **Severity: Important — pre-pivot engagement and staging statements survive.** The revision says the branch “is posted” as the opener ([plan](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-bark-upstreaming-plan.md:279)), while the draft still says Greg will post it and has no MR number. Elsewhere the plan still says “design issue first,” G3 before the first submission, and the spike remains local ([process](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-bark-upstreaming-plan.md:508), [decisions](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-bark-upstreaming-plan.md:525)). The captaind stage also says the already-upstreamed crate/tests will “land” again ([plan](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-bark-upstreaming-plan.md:353)).

   Fix: mark the spec track complete; replace design-issue-first/local-spike language with the draft opener; say “planned for posting” until an MR URL exists; change G3 to before marking the series ready; and make the captaind MR extend the existing crate rather than reland it. Rename “MR-0 CLOSED” to “release-contract review closed” to avoid implying the upstream draft itself was closed.

3. **Severity: Important — the matrix is advertised as the current acceptance checklist but remains tied to the pre-restructure snapshot.** Its source is explicitly `9f1c7fd` ([matrix](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-conformance-matrix.md:10)), while the plan calls it the acceptance checklist. Consequently bare citations now point at unrelated HEAD text—for example PV-1 cites current lines 164–166, now SCID collision rules, while the policy is at [current spec line 204](/home/greg/bitcoin-dev/cleanroom/boats/08-channels.md:204). Part 2 still presents E-1..E-17 as pending restructure work ([matrix](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-conformance-matrix.md:255)), and I-10 still calls synthesized position an unspecified “spec gap” despite its landing ([matrix](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-conformance-matrix.md:310)).

   Fix: regenerate Part 1 citations against HEAD and mark E-1..E-17 resolved. Alternatively, clearly label Parts 2–3 archival and qualify every citation as `9f1c7fd:08-channels.md`. Also make OP-21’s core-only override explicit: the core says suspend, while the first-release profile accepts stock force-close.

4. **Severity: Important — the post-review CLTV quantifiers never reached the plan or matrix.** They still say merely “invoice ≥ F,” forwarding delta ≥ F, and force-close as budget “approaches F” ([plan](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-bark-upstreaming-plan.md:91), [matrix](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-conformance-matrix.md:296)). The normative spec now requires:

   - invoice floor = maximum `F` across eligible receiving channels;
   - received HTLC uses the actual receiving channel’s `F`;
   - `incoming_cltv − outgoing_cltv ≥ F_in`, or an advertised delta dominating all possible incoming scopes;
   - force-close no later than exactly `F` remaining.

   These are normative at [spec lines 1063–1079](/home/greg/bitcoin-dev/cleanroom/boats/08-channels.md:1063) and originated in the completion review ([review](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-30-codex-refactor2-review.md:3)). Fix §3.2, I-6, the MR implementation text, and the §7 floor row.

5. **Severity: Important — MR-0 coverage is overstated.** The plan claims `lightning-background-processor` fit was proven and lists it as item (g) ([plan](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-bark-upstreaming-plan.md:162), [scope list](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-bark-upstreaming-plan.md:320)). The branch declares that dependency but the release-contract suite uses a hand-rolled pump; no test references the background processor. This also makes “ALL SIX ASSERTIONS” inconsistent with seven lettered items and eleven tests. The draft’s “every LDK behavior the feature depends on” is therefore too broad ([draft](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-30-draft-mr-description.md:24)).

   Fix: narrow the claim to the eleven enumerated virtual-funding/fee-bump contracts, mark background-processor integration for the captaind MR, and reconcile six/seven/eleven terminology. Also remove “`ZeroConf` feature accessors on our path” from the draft ([line 27](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-30-draft-mr-description.md:27)); this profile uses ordinary inbound acceptance with `minimum_depth ≥ 1`, not the `ZeroConf` feature.

6. **Severity: Important — a load-bearing restart ordering decision exists only in review/commit evidence.** The plan carries monitor re-registration but not the final lifecycle fence ([plan](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-bark-upstreaming-plan.md:306)). The MR-0 review established that restart must quiesce and await pumps/transports before snapshotting, restore dormant, re-register monitors and chain/funding state, then start processing and reconnect ([review](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-30-codex-mr0-review.md:19)).

   Fix: add this ordering to the harness/captaind stage and cross-cutting engineering notes.

7. **Severity: Minor — upstream-draft preflight remains incomplete.** Replace the spec-link and sign-off placeholders ([draft](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-30-draft-mr-description.md:19)). Its sequencing feedback omits `lightning-checkpoint-arkoor-builder`, which the plan lists as adjacent work ([plan](/home/greg/bitcoin-dev/cleanroom/boats/docs/plans/2026-07-29-stage1-bark-upstreaming-plan.md:67)); add it or explain the omission. Attribute the claimed `11/11 (~3s)` run to Greg, since the Codex review environments explicitly did not rerun the socket suite.

## Verified consistent

- `F = channel_max_vtxo_exit_depth + pinned_exit_delta + cltv_claim_slack`.
- Core-only type, split headroom/live-bound guard, expiry treadmill, deep-reorg force-close, and both recorded deviations.
- Core-only keysend is allowed through the per-HTLC floor; the deferred type extension rejects it.
- SCID uniqueness, persistence, bounds, peer disagreement, non-announcement, and alias posture.
- `supports_channels` naming and advertisement posture.
- Branch source/target, commit hashes `a79035a9`/`ea33bbf4`, lockfile `lightning` 0.2.4, two-commit shape, and eleven test functions.
- Draft-quoted crate/test paths and existing repository names all exist.
- The I-1..I-10 resolutions did land in the normative spec.

## Verdict

**NEEDS-EDITS**: series renumbering/count, pivot cleanup, current matrix/profile overlay, CLTV quantifiers, release-contract claim narrowing, restart ordering, and draft preflight.