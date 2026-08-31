# Adversarial review

1. **Important — the “pure move” splits the `Messages` section.**  
   Citation: `1323ff01:08-channels.md:1584,1786,1824-1839`; previously `1323ff01^:08-channels.md:1692-1703,1824`.  
   `leaf_vtxo_cosign` and `submit_round_participation` remain H3 headings but now fall under `## The teleport protocol`, not `## Messages` or `## Refresh`. This mis-scopes Ark refresh messages as teleport subsections.  
   Suggested fix: make “The teleport protocol” an H3 beneath `## Refresh`, or add an explicit refresh-extension wrapper.

2. **Important — the first extension does not build only on preceding material.**  
   Citation: `0957f4c9:08-channels.md:76-87,1328-1331,1353-1356,1372-1398`.  
   “The Ark channel type” normatively specifies refresh admission, refresh bridges, and refresh scheduling before `## Refresh` exists. Its unqualified “every Ark channel MUST negotiate this type” language also conflicts with the preceding core-only note (`:1184-1202`).  
   Suggested fix: move every refresh-specific clause into `Refresh`, and scope the type obligations explicitly to implementations negotiating that extension.

3. **Important — two core rewrites are not compositional with the extensions.**  
   Citation: `681b1102:08-channels.md:208-209,501-503`; conflicting extension behavior at `:1400-1413`.  
   The core says the funding VTXO remains unchanged until close/force-close, while Refresh replaces it without closing. The parenthetical also reads as though the downgrade split is the cooperative spend, although Refresh adds the forfeit spend.  
   Suggested fix: qualify these statements with “In a core-only profile,” or phrase them so later extensions add cases without contradicting the core.

4. **Important — “the core changes exactly one message” is literally false.**  
   Citation: `0957f4c9:08-channels.md:1088-1093,1119-1120`; `681b1102:05-arkoor.md:224-230`.  
   The core extends both `arkoor_cosign_request` and the distinct `arkoor_cosign_response`.  
   Suggested fix: say “one RPC exchange, comprising two extended messages,” enumerate both, and include response additions in Compatibility.

5. **Important — the core purity sweep missed two extension-only concepts.**  
   Citation: `681b1102:08-channels.md:515-516,597-603`.  
   The normative Operation list still says “open/leaf cosign,” although the core has no leaf cosign variant, and close-outcome retention can terminate when the channel VTXO is “forfeited,” an extension-only event.  
   Suggested fix: use “open cosign” and “split or expiry-swept” in core; add leaf/forfeit behavior under Refresh.

6. **Important — “stock BOLT implementation” overpromises without qualification.**  
   Citation: `0957f4c9:08-channels.md:76-86`; `681b1102:08-channels.md:117-165`.  
   Core virtual funding keeps the bridge off-chain and injects logical confirmation. BOLT #2 ordinarily requires broadcasting the funding transaction and waiting for confirmation; even LDK’s manual-funding API expects caller-managed broadcast. A stock LDK library can host this through an Ark-specific funding/chain adapter, but the resulting behavior is not literally stock BOLT operation. [BOLT #2](https://github.com/lightning/bolts/blob/master/02-peer-protocol.md), [LDK manual funding API](https://docs.rs/lightning/latest/lightning/ln/channelmanager/struct.ChannelManager.html).  
   Suggested fix: say “an unmodified Lightning implementation exposing manual-funding and application-supplied chain hooks.”

7. **Minor — three quoted references are only prefixes, not existing titles.**  
   Citation: `681b1102:08-channels.md:58,266-267,599`; actual headings at `:668,983`.  
   `"Unilateral exit"` and `"Downgrade"` do not exactly name `Unilateral exit / force-close` and `Downgrade: close into Ark balance`.  
   Suggested fix: use the complete titles.

## Verified clean

- **Losslessness:** both `1323ff01` versions contain 1,929 lines; sorted contents are identical with SHA-256 `df92b440…82d60`. Moved paragraphs are byte-identical and unsplit. Finding 1 is the sole hierarchy defect.
- **Semantic fidelity:** every `681b1102` hunk was reviewed. No unintended modal or actor change was found beyond the findings and excluded pending work. The new pre-upgrade `SHOULD` correctly specializes ARK #5’s recipient-refresh recommendation and does not contradict `05-arkoor.md`.
- **References:** the old deadline title is absent from all six requested files; both user-story references were updated. Other exact heading/run-in references resolve, except Finding 7.
- **Layer notes:** the Refresh extension note accurately describes the core alternatives. Exceptions are Findings 2, 4, and 6.
- **Core purity:** ordinary remaining refresh mentions concern pubkey-VTXO round refreshes or deliberate overview pointers. Exceptions are Findings 3 and 5.

**Verdict: PASS-WITH-CHANGES**

The requested file could not be saved because the workspace is mounted read-only; the repository remains untouched.