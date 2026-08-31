# Codex review record — MR-7 c1: the `/channels` REST surface

**Commit:** `cdfe88223` "bark-rest, bark-json: the /channels surface"
(amended through the rounds), on the convergence-fix opener
`92b297cd2`, branch `ark8-channels-stage1-payments`.
**Verdict: PASS in 5 rounds.** Full battery green on the amended tree;
the two new barkd REST e2e vectors green; the featureless bark build
(broken latently since the payments MR — nothing ever built that
shape) fixed and re-verified.
Full barkd suite 66/68 — the two failures are the CLN-dependent
recovery vectors (`recovered_wallet_finds_lightning_send_*`), which
die at lightningd spawn: CLN is not installed on this machine
(confirmed with Greg), environmental and unrelated to this commit.

## What shipped

- **REST `/channels`** (bark-rest, exits/boards idiom): list + get
  (the composed status: record ⋈ live LDK view ⋈ operator markers),
  the ACTION-ORIENTED open (`202` + declared `Location:
  /api/v1/channels/opens/{action_id}`), the pollable open-action
  resource, close, exit + cancel, `POST/GET /channels/payments`,
  `POST /channels/invoice`. Keysend not surfaced (G1 decision 5).
- **bark-json DTOs** incl. journal states verbatim (stranded visible
  by design) and the close-phase detail (retained split's response
  txid / registration / leaf count) + the exit's own phase.
- **Binary enablement**: the channels feature propagated to bark-rest
  and bark-cli; the shared opener passes the sqlite client as the
  channel store (a filestore wallet refuses channel verbs with the
  typed node-unavailable error).
- **The typed error contract end to end**: `ChannelRefusal`
  (malformed/not-found/refused/node-unavailable) attached at the
  wallet's refusal sites as the ROOT of the anyhow chain (messages
  unchanged for every existing consumer), downcast-mapped in bark-rest
  to 400/404/422/503 (new `ServiceUnavailable` variant); request-body
  rejections ride the same 400 (an `AppJson` extractor).
- **openapi.json + the generated bark-rest-client regenerated**
  (generator 7.22.0 — the pinned version; the repo's hand-maintained
  model shim + manifest preserved over the generated models).
- **e2e** (barkd suite): the full surface arc — typed refusals before
  any channel exists, the open polled through the action resource to
  readiness, invoice shapes (amountless + refusals), real payments to
  captaind's collect leg incl. the amountless-override rules, the
  close observed through record states, exit + live cancel.

## The rounds

- **r1 (9 findings):** the action resource read the checkpoint before
  the movement (a lingering checkpoint after terminal → lied
  in-progress); Successful lacked the ids (now both non-null, their
  absence an invariant error); cancel of a never-exited channel
  returned 200 (the store's idempotent branch — now a typed refusal,
  and the sdk's pinned repeat-cancel expectation updated to match);
  a dozen unmarked refusal sites (close depth/hard-line, cancel
  gates incl. the store origin gates, the taken open input, the
  enter-close CAS — typed surgically; storage errors deliberately
  stay 500, blanket marking rejected per codex); payments-table
  violations (amounted-override reclassified 422; hash reuse a
  PRE-check so db errors stay internal; immediate LDK send failures
  typed; the listing through channel_node); the max_receiving_floor
  blanket removed (only its no-admissible outcomes typed); the
  close-phase detail added; the Json-rejection 422→400 extractor;
  the Location header declared in the schema.
- **r2 (3):** a poll TOCTOU (terminal landing between the two reads —
  one movement re-read before NotFound); two remaining untyped cancel
  gates (the HTLC-deadline gate; the exit manager's past-the-bridge
  gates via a cfg-bridged helper so the featureless build stands);
  the open op's undeclared 400.
- **r3 (1):** the pay handler's journal lookup direction-scoped
  (pay-your-own-invoice journals a receive row under the same hash).
- **r4 (1):** the generated client lacked channels_api → regenerated
  properly (the environment initially lacked the generator; fetched
  the pinned 7.22.0 jar and preserved the hand-maintained shim).
- **r5: PASS.**

## Key decisions for review

- **Marker-at-root**: the refusal kind is the error chain's ROOT with
  the human message as context — downcast works, every existing
  message and message-matching test unchanged.
- **Surface cancel is not idempotent**: a canceled exit leaves a ready
  record, and a ready record has no exit to cancel — the earlier
  silent-success idempotence read "never exited" as success.
- **The e2e drives raw HTTP** (status codes asserted, bark_json types
  parsed); the generated client is regenerated for consumers but not
  used by the harness for these assertions.
