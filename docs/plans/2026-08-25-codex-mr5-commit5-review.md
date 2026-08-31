# Codex review: MR-5 commit 5 — "channels: floor the open at the downgrade's settlement bound" (1 round to PASS)

Commit `e2d255a7e` on `ark8-channels-stage1-close` (bark-stage1). Battery
at PASS: the 659-sat open-refusal vector + settlement e2e green, units
412/412 (every existing open sits far above the floor).

## Origin

Review question from Greg on commit 2's dust-isolation shape: the split
requires `V ≥ 2·P2TR_DUST` (660) when a side is sub-dust — is the
mirror-image bound enforced at OPEN, or could someone open a channel
whose close has no cooperative settlement (jagged UX)? Finding: nothing
of ours enforced it — only LDK's incidental 1000-sat reserve floor
(`MIN_THEIR_CHAN_RESERVE_SATOSHIS`) prevented sub-660 channels, and
LDK zero-reserve support is expected in an upcoming release, at which
point that incidental cover disappears entirely.

## The commit

- `ark::channel::MIN_CHANNEL_FUNDING = 2 × P2TR_DUST`, single-sourced
  (shares `P2TR_DUST_SAT` — no drift), documented as the downgrade's
  bound, not LDK's.
- Client `open_channel` pre-check (before anything irreversible, with
  the real reason); server upgrade admission enforces it beside the
  depth headroom — the same one-way-door principle, the value
  dimension.
- Spec (08-channels.md, boats `1050581e`): the split's normative 660
  floor now extends to channel-open admission as a MUST on both sides —
  capacity IS the split's total, fixed at open.

## Round 1: PASS — no findings

Verified: `>=`/`<` admit 660 and reject 659; every bridge-bearing
request reaches the authoritative admission before mutation (generic
arkoor and round paths reject channel-funding outputs); capacity stays
the correct bound under a future `push_msat` (close balances sum to the
recorded capacity); message divergence between the two refusal sites
cannot alter recovery (client refusal precedes action persistence;
server refusal is code-classified and tears down the pre-cosign
remnant).
