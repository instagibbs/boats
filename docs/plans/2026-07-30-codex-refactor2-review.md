## Findings

1. **Important — channel-specific `F` is under-quantified for invoices and forwarding.**  
   **Citation:** `dfa97d7b:08-channels.md:1062-1072`; comparison at `f5783105:08-channels.md:1323-1328`.  
   **Why:** An invoice is not inherently bound to one receiving channel, and a forwarding delta must protect the actual incoming Ark scope. Selecting the wrong channel’s smaller `F` can underbudget forwarding; per-HTLC admission only prevents the invoice case from becoming a loss.  
   **Fix:** Require invoices to use `max(Fᵢ)` across eligible receiving channels. For forwarding, require `incoming_cltv − outgoing_cltv ≥ F_in`, or advertise a delta dominating every incoming scope that can feed that hop.

2. **Important — the per-HTLC force-close trigger is not stated.**  
   **Citation:** `dfa97d7b:08-channels.md:1071-1072`.  
   **Why:** “MUST force-close while … retains at least `F`” can literally require immediate force-close for every accepted HTLC. The intended rule is to force-close when an unresolved HTLC approaches `F`.  
   **Fix:** Say the node MUST initiate force-close no later than the point where the unresolved HTLC’s remaining budget reaches `F`—or the extension’s larger threshold.

3. **Important — refresh-then-downgrade lost its explicit modal.**  
   **Citation:** old `f806dab^:08-channels.md:799-802`; replacement `8df78ab2:08-channels.md:1436-1439`.  
   **Why:** The former text said an over-depth channel “MUST refresh first”; the counterpart now says only “refresh first” descriptively. The pre-close depth check remains a MUST, limiting the damage, but the mandatory remedy was weakened.  
   **Fix:** Restore: “A client pursuing cooperative settlement of an over-depth channel MUST refresh first, then initiate downgrade.”

4. **Minor — the type budget does not necessarily dominate `F`.**  
   **Citation:** `8df78ab2:08-channels.md:1344-1351`.  
   **Why:** Algebraically, `B = F − cltv_claim_slack + pinned_exit_delta + cltv_safety_margin`. Because the two margins are unconstrained, the explicit `max(F,B)` is correct, but “the budget whenever an HTLC…” is not always true.  
   **Fix:** State `max(F,B)` directly and say `B` is active only when it is the larger term.

5. **Minor — board-era and moved-counterpart references remain.**  
   **Citation:** `f5783105:08-channels.md:1110-1113,1380-1387,1691-1694`; `f5783105:docs/channel-user-stories.md:156-181`.  
   **Why:** The text still names a nonexistent “board cosign,” calls board confirmation the point of no return, and the user story points after-forfeit behavior at “The close” even though it moved to Refresh. The user story also contradicts the normative statement that success-CSV divergence is first detected at the first HTLC commitment exchange.  
   **Fix:** Replace board-cosign references with upgrade/arkoor cosign and upgrade registration; point the forfeit story to Refresh and describe first-HTLC detection accurately.

## Verified clean

1. **Timing:** `F = D + d + slack` is sufficient and conservative at all six intended sites once Findings 1–2 are clarified. The split-registration escape is sound: once registration may be final, broadcasting the old scope is forbidden, while a registered split wins through `nSequence=0`. The type budget shares exactly `D+d` with `F`; it is never summed. The `u16` refusal is compatible with BOLT #7’s [`u16 cltv_expiry_delta`](https://github.com/lightning/bolts/blob/master/07-routing-gossip.md), while BOLT #11’s invoice field is [variable-length](https://github.com/lightning/bolts/blob/master/11-payment-encoding.md).

2. **Counterparts:** All requested refresh counterparts exist. Timing/profile rules are at lines 1514–1528; anchor, expiry/depth reset, third seam, liveness, arkoor recommendation, quiesce/carry, close/forfeit composition, teleport retirement, and parent-exit continuation are at lines 1420–1461. Exception: Finding 3’s weakened modal.

3. **Layering:** Clean. The core has no normative extension dependency; the type section has no refresh/teleport reference; its obligations are extension-scoped. F2 is complete.

4. **Profiles:** Clean. The SHOULD/MUST split, live-bound rule, expiry treadmill, type designation, HTLC posture, and two deliberate deviations are internally consistent. Both intermediate profiles are coherent.

5. **References:** Every quoted section title resolves, including “Implementation profiles.” Exception: Finding 5’s semantically stale references. Matrix drift, noted without requesting fixes:

   - I-4’s “documented default” for `cltv_claim_slack` did not land.
   - I-6’s intended “approaches `F`” trigger became Finding 2.
   - I-9’s explicit checked `max_vtxo_exit_depth ≥ 2` guard did not land.
   - I-10’s SCID-position requirements landed, but its first-release deep-reorg/stock-force-close disposition is not stated.

6. **Prior fixes:** F1 and F3–F7 remain intact.

**Verdict: PASS-WITH-CHANGES**