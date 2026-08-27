# Codex review record — MR-7 opener: the open-action convergence fix

**Commit:** `92b297cd2` "bark: a taken channel record never wedges the
open action" — one standalone commit on the pushed payments tip
`a491cea7f` (`ark8-channels-stage1-payments`). The wedge was found by
the MR-7 design-note G1 (rounds 4–5); the introducing commits are in
the PUSHED payments stack, so per the immutability rule the fix lands
standalone rather than folded back (Greg asked; answered).

**Verdict: PASS in 2 rounds.** Full battery green on the amended
commit.

## The bug

A peer can cooperatively close a channel the moment it is cosigned —
before the client ever registers the chain — and the hard-line exit
can take a record whose registration keeps failing. Both leave the
record in states the open action's Registering/Feeding phases neither
accepted nor terminated on; a terminal input exit makes the server
refuse the WHOLE upload permanently (the not-exited re-check), and
every post-cosign rejection parks the action forever.

## The shipped shape (round 1 sharpened it substantially)

- The open consults the record BEFORE the registration RPC.
- **The demotion is EXIT-SCOPED** (r1 f1): only `exiting`/`closed`
  tolerate a refused upload. Every CLOSE-owned state (negotiating,
  outcome_ready, registration_pending, downgraded, fallback_only)
  keeps REQUIRING registration — the close's own split cosign needs
  the backing registered server-side (`downgrade.rs` loads it from the
  registered rows), the server still accepts it there (its row is
  cosigned; the registration also arms the watch the close needs), and
  transient failures re-drive. An exit taking the record meanwhile is
  read on the next drive.
- **Change is never exposed as spendable without a successful
  registration covering it** (r1 f2): in the exit-scoped path the
  change is registered FIRST via a change-only payload — carrying no
  channel-funding VTXO, it bypasses the channel leg's whole-upload
  refusal in `store_vtxo_transactions` and registers ordinarily. A
  change-only failure is a hard error that re-drives (exit states are
  stable, so the demotion is reproducible). No change → nothing to
  register.
- The post-RPC reconcile accepts every taken state (only a
  still-cosigned record flips to registered); the feed-phase carve-out
  widens from exiting/closed to every taken state, finishing the
  movement and the action; a cosigned record at the feed phase remains
  a loud contradiction (deliberately NOT carved out).

## Deferred by design (the G1-passed note owns them)

The e2e proof — peer close at `Cosigned`; a hard-line exit winning
with registration held until its chain confirms — rides the MR-7
lifecycle-e2e commit, which adds the deterministic holds the
choreography needs.
