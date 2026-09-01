# Channel size and HTLC exposure — the UX invariants, and how to get them

Greg, 2026-09-01: "let's pick the invariants we want from UX perspective,
you figure out how to do it." This proposes the invariants, then the
mechanism.

**STATUS: RATIFIED AND BUILT.** I4 signed off 2026-09-01 ("sounds good as
well, include the bugfix"). Shipped as two commits on `bark-stage1`:
`cab54a71c` (I4 + the estimate bug) and `4d49a78b0` (the absolute
in-flight bound). 29/29 green across the channels and payments suites
plus both conformance vectors; the in-flight change is mutation-verified
— restoring the 10% rule makes an 8,000 sat payment over a 12,000 sat
channel unroutable.

Not built: nothing. The withdrawn minimum-channel-size (see I1) was the
only other proposal.

Context: `2026-08-31-coverage-audit-consolidated.md` §D-2, and the
spec-vs-implementation split established in review — the only amount
floor the spec imposes is `V ≥ 2·P2TR_DUST` (660), and it is there for
transaction shape ("normative, not an economic assumption",
`08-channels.md:781-798`), not economics. Every constraint that makes
small channels awkward today is ours.

## The invariants (what a user can rely on)

**I1 — A channel you can open is a channel you can pay over.**
Openable implies it can carry a non-dust HTLC in both directions after
reserves. Today this is FALSE, and the cause is the in-flight cap alone:
at 10% of capacity, a channel under ~3,300 sat has a cap below the
330-sat dust limit, so no routable HTLC is expressible. Openable and
useless is the bug.

NOTE (Greg, 2026-09-01): this needs NO new minimum channel size. The
spec's 660 is the DOWNGRADE SPLIT's bound — every balance split the
close could fix must have a standard conflict-winning transaction — and
has nothing to do with the unilateral exit. An earlier draft of this
note proposed a 10,000 sat minimum justified partly on exit economics;
that conflated the two and is WITHDRAWN. Fixing the cap makes small
channels payable on their own, and a policy minimum would exclude
precisely the users this is for.

**I2 — Payment size is bounded by your balance, not by a fraction of
it.** Any amount up to (capacity − reserves) can be in flight on a small
channel. The absolute exposure ceiling still applies, so it only binds
on channels large enough for it to matter.

**I3 — The one size bound we impose is a MAXIMUM, chosen on its merits.**
It bounds a single channel's worst case — which is also the R2 exposure,
since losing the expiry race costs the WHOLE VTXO and that scales with
capacity. There is no policy minimum (I1); the small end is governed by
the spec's 660 and by LDK's own reserve floors.

**I4 — Affordability is advice, not a veto.** ⚠ NEEDS SIGN-OFF: this
reverses R3's ratified "refuse/alert on underfunding". A user who cannot
currently fund an exit still gets the channel, and is told. The estimate
must also be correct, which today it is not (below).

**I5 — Nothing here is per-deployment.** The caps stay uniform policy
that both sides can verify, because the acceptance gate depends on
verifying them. Uniform must mean "same policy", not "same number" —
that distinction is what makes the rest of this safe.

## The numbers

| Knob | Now | Proposed | Why |
|---|---|---|---|
| Spec floor | 660 | 660 (unchanged) | spec, transaction shape |
| Min channel | — | **none added** | the spec's 660 stays. Below that, LDK's own floors bind on their own: `create_channel` refuses under 1,000 sat outright, and the 1,000-sat `MIN_THEIR_CHAN_RESERVE_SATOSHIS` on each side leaves nothing spendable until ~2,000-3,000. That emergent floor is LDK's, not a policy we impose — and it needs no enforcement of ours |
| Max channel | 2,000,000 sat (artifact of `200M msat ÷ 10%`) | 2,000,000 sat (unchanged, now deliberate) | no UX complaint; bounds R2. Reconsider separately |
| In-flight | 10% of capacity | **`min(200,000 sat, capacity − reserves)`** | absolute, as the profile always wanted (`min(V_cap, p%×capacity)`); the percentage half only ever bit at the small end |
| `MAX_ACCEPTED_HTLCS` | 6 | 6 (unchanged) | bounds on-chain resolution cost and dust exposure; no size bound replaces it |

Resulting in-flight by size: 3k → ~1k (everything left after the two
reserves); 10k → ~8k; 100k → ~98k; 400k → 200k; 2M → 200k. The cap binds
only above ~200k capacity, which is where it should — and the smallest
channel LDK will build is payable.

Directionality: keep it BILATERAL and uniform. The spec only requires
caps "on channels the server forwards over" (`08-channels.md:2242-2252`),
so decoupling the client side is permitted — but it buys nothing here.
The cliff came from the percentage being small on small channels, not
from the cap being bilateral. Decoupling stays available later if the
server wants to be stricter than the client.

## The mechanism — no LDK fork

The BOLT field is already absolute (`max_htlc_value_in_flight_msat`).
Only LDK's *config* is percent-only. Since capacity is known at open,
derive the percentage per channel:

```
target_msat = min(MAX_HTLC_IN_FLIGHT_CAP_MSAT, (capacity - reserves) * 1000)
percent     = clamp(floor(target_msat * 100 / capacity_msat), 1, 100)
```

Round DOWN, so the advertised cap never exceeds policy. Granularity is
1% (a `u8`), i.e. 20k steps on a 2M channel and finer as channels
shrink — acceptable, and it errs conservative.

Both call sites already accept a per-channel override in stock LDK
0.2.4:

* client (funder): `create_channel(…, override_config: Option<UserConfig>)`
  — `channelmanager.rs:4116`;
* server (acceptor): `accept_inbound_channel(…, config_overrides:
  Option<ChannelConfigOverrides>)` — `:9852`, with
  `ChannelHandshakeConfigUpdate::max_inbound_htlc_value_in_flight_percent_of_channel`
  (`config.rs:1063`). The server already runs
  `manually_accept_inbound_channels: true`, so it is on that path
  anyway.

### Why this does not break the acceptance gate

`bark-channels/src/config.rs` warns that "a deployment-varied cap would
break the uniform capability check that gates acceptance". A
capacity-derived cap is NOT deployment-varied: every channel of a given
size gets the same number, so the check remains reproducible. It changes
from comparing against a constant to recomputing policy from the
channel's capacity:

    advertised_percent × capacity  ==  policy_target(capacity)

The persisted cap evidence must therefore carry the CAPACITY alongside
the negotiated caps, so the startup gate can re-derive after a restart
without trusting the stored percentage alone.

## What this touches

Product:
1. `bark-channels/src/config.rs` — replace the `MAX_HTLC_IN_FLIGHT_PERCENT`
   constant with `in_flight_percent_for(capacity)`; keep the absolute cap
   as the policy input. `max_channel_capacity_sat()` stops being derived
   from the percentage and becomes an explicit bound.
2. Open paths — pass the derived override on `create_channel` (client)
   and `accept_inbound_channel` (server).
3. `server/src/channels/admission.rs` — verify the derived absolute
   rather than a fixed percentage; persist capacity with the evidence.
4. Client `open_channel` — no change to the amount floors (see I1). The
   existing 660 refusal keeps its wording, which already names the
   cooperative-settlement reason rather than an economic one.
5. `check_channel_exit_reserve` — price on the INPUT'S ACTUAL DEPTH
   (`input.exit_depth() + depth_cost`), which `open_channel` already
   computes, instead of `ark_info.max_vtxo_exit_depth`. This is a bug fix
   independent of I4: at the mainnet default of 100 it demands ~612k sat
   of confirmed coin for a depth-2 input. Then, per I4, downgrade the
   refusal to the `exit_funding` warning that already exists.

Tests:
6. `quarantine_disables_forwarding_and_offender_claims` and
   `server_refuses_a_weakened_htlc_floor_profile` — both assert on the
   uniform-profile gate; they must move to the derived form.
7. `open_channel_refusals` — unchanged (no new refusal).
8. The parked `open_refuses_an_unaffordable_exit_reserve`
   (scratchpad `parked-reserve-vector.patch`) — becomes a WARNS vector
   under I4, or lands as written if I4 is rejected.
9. New: a small-channel payment vector — open at the smallest channel
   LDK will build, pay a dust-adjacent amount in both directions. That
   is I1, and nothing covers it today.

Spec: no amendment needed. 660 stays; the caps remain "conservative
per-HTLC value caps and in-flight limits" as §2242-2252 requires. If the
numbers are worth recording, they belong in `doc/channel-payments.md`
with R1/R2, not in the spec.

## Sign-off needed

* **I4** — reverses a ratified R3 disposition (refuse → warn). The
  reserve *estimate* fix is a bug fix and should land either way.

(The proposed minimum channel size is withdrawn — see I1. The only
product number left is the 200,000 sat in-flight ceiling, which is
today's `MAX_HTLC_IN_FLIGHT_CAP_MSAT` unchanged.)

## Note on stage 2

The in-flight cap exists to bound a fee race with no consensus-ordered
head start. Once the CSV channel type gives the server a guaranteed
response window, the bound stops carrying that weight and can relax
much further — so this should be sized for stage 1 and revisited, not
treated as permanent.
