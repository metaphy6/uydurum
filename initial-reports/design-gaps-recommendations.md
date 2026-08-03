# Design Gap Recommendations — Uydurum

**Scope:** Concrete proposals for every rule/mechanic left undefined or ambiguous in [BLUEPRINT.md](../BLUEPRINT.md). These are recommendations for discussion, not decisions.

> **Status update (2026-08-03) — decisions applied to BLUEPRINT.md:**
> - Gap 1 ✅ applied as recommended, including tie-breaking.
> - Gap 2 ✅ applied, modified: pool is `players + 6` roots, replenished each round back to full size (not `players + 2`).
> - Gap 3 ✅ applied: blind simultaneous pick (option 1).
> - Gap 4 ✅ partially applied: multi-steal cap + plausibility pressure. Bluff budget and escalating challenge rewards **deferred** — unproven that they'd work as intended; revisit with playtest/simulation data.
> - Gap 5 ✅ applied: constant hand size of 3.
> - Gap 6 ✅ applied in full, except bot takeover which is explicitly rejected (as recommended).
> - Gap 7 ⏳ still open — remains project-blocking, and now also constrains the client-side dataset for the Offline Training Mode.
> - Gap 8 ✅ already resolved by the server-authoritative architecture (server owns the phase clock, client timers display-only) — verified present in the blueprint's Word Engine and Network sections.

---

## Gap 1: Scoring is undefined

The blueprint says "longest valid word wins the round" and "steals all round points," but never defines what a point is. This is the single most important missing piece — the bluff economy cannot be balanced without it.

**Recommended scoring table (v1 starting point, tune via playtesting):**

| Event | Points |
|---|---|
| Valid word, per letter | +10 / letter |
| Longest valid word of the round (bonus) | +30 |
| Correct vowel-harmony chain of 2+ suffixes (bonus) | +15 / extra suffix |
| Unchallenged successful bluff | +25 from **each** opponent's round score (capped, see Gap 4) |
| Correct challenge (flagger) | takes the bluffer's full round score |
| False challenge (flagger) | transfers 50% of own round score to the accused |
| No word submitted | 0 |

**Key principle:** score transfers should operate on *round* points, not *match* points. Transferring cumulative match points makes early leads catastrophically swingy and lets one late bluff invalidate four rounds of play.

**Tie-breaking (currently undefined):**
1. Longest word → 2. Word submitted earliest → 3. Fewer suffixes used (elegance wins) → 4. Shared round win (both get the bonus).

---

## Gap 2: Root pool math and draft structure are ambiguous

"Players × rounds" roots (25 for a 5-player game) but the draft is a single 10-second phase — is the whole pool drafted in round 1, or per round?

**Recommendation: per-round pools.**
- Each round presents `players + 2` roots (e.g., 7 roots for 5 players). The +2 surplus gives the slowest players a real choice instead of a leftover.
- Total roots generated per match ≈ `rounds × (players + 2)` — adjust the blueprint's math accordingly.
- Roots used in a previous round are removed from the pool.

**Why:** a 25-root single draft in 10 seconds is unreadable on a phone screen and rewards nothing but tap speed. Small per-round pools keep every draft legible and tense.

---

## Gap 3: "Fastest tap wins" is a fairness problem

Whoever has the lowest network latency wins every contested draft. This is true in *any* networked architecture but is worst in P2P (see [p2p-network-analysis.md](p2p-network-analysis.md)).

**Recommendations (pick one):**
1. **Blind simultaneous pick (preferred):** all players secretly rank their top-2 roots during the 10 seconds; conflicts resolve by (a) current lowest score wins — a built-in catch-up mechanic — or (b) random. No latency sensitivity at all, and it adds strategy (predict what others want).
2. **Server/host timestamping:** claims carry a client timestamp; the authority resolves within a tolerance window (e.g., 150 ms buckets) with random tie-break inside a bucket.
3. **Snake draft:** turn order by reverse standings, 3 seconds per pick. Slower, fully fair, very readable.

Option 1 keeps the 10-second energy while deleting the entire class of latency disputes.

---

## Gap 4: Bluff economy is unbalanced on paper

An unchallenged bluff steals from *every* player. In a 6-player lobby that makes bluffing pay ~5× a normal win — bluffing becomes strictly dominant, and the game collapses into "everyone bluffs every round."

**Recommendations:**
- **Cap the multi-steal:** unchallenged bluff takes a fixed amount (e.g., 25 pts) from each opponent's *round* score, floored at 0 — never from cumulative match totals.
- **Bluff cooldown or budget:** each player may bluff at most ⌈rounds / 2⌉ times per match. Forces bluff timing to be a strategic resource.
- **Escalating challenge reward:** the later in the match, the more a correct challenge pays. Keeps everyone reading words in round 5 instead of tuning out.
- **Plausibility pressure:** a bluffed word must still be composed of the player's actual drafted root + suffixes with correct vowel harmony applied. The bluff is that the *combination* isn't a dictionary word — not free-typing gibberish. (Worth stating explicitly in the blueprint; currently ambiguous.)

**Validation:** before building anything, run a spreadsheet/Monte-Carlo simulation of 1,000 matches with naive strategies (always-bluff, never-bluff, always-challenge) and confirm no strategy dominates. This costs an afternoon and de-risks the whole design.

---

## Gap 5: Suffix hand rule is confusing

"A player must hold exactly three suffixes by the final three rounds" — undefined for early rounds, and the reason for the rule is unclear.

**Recommendation: simplify to a constant hand size.**
- Hand size = 3 suffixes at all times, from round 1.
- Each suffix draft phase: draw/steal up to hand limit; dropping a suffix returns it to the public pool (keep this — it's a good mechanic).
- Auto-fill at timer end (keep).

One rule, no phase-dependent special cases, same strategic depth.

---

## Gap 6: Disconnects and dropouts are unhandled

A 3–6 player synchronous game *will* have mid-match disconnects, and with a 3-player minimum, one dropout can make a match unplayable.

**Recommendations:**
- **Grace window:** 20 seconds to reconnect; the player's turn auto-passes (no word submitted, 0 round points) while absent.
- **Below minimum:** if a lobby drops under 3 active players, finish the *current* round, then end the match and score it as-is.
- **Bot takeover — avoid:** an AI stand-in in a bluffing game is worse than an empty seat (players can farm it).
- **Rejoin state:** the authority (server or host) must be able to replay current round state to a rejoining client — design the state model as a serializable snapshot from day one.
- **Rage-quit disincentive:** leaving mid-match forfeits any pending score transfers in the leaver's favor.

---

## Gap 7: Dictionary sourcing and licensing (project-blocking)

The blueprint assumes "the official dictionary" (100k+ words) exists as a usable asset. **TDK's Güncel Türkçe Sözlük is not freely redistributable** — scraping and shipping it in a commercial app is a legal risk.

**Recommendations:**
- Evaluate openly licensed sources: **Zemberek-NLP** morphology data (Apache-2.0), Wiktionary Turkish extracts (CC BY-SA — check share-alike implications for a closed app), or hunspell-tr word lists.
- Because Uydurum generates words via root+suffix morphing, a **morphological analyzer approach** (Zemberek) is actually a better fit than a flat word list: it can answer "is *kafalık* a well-formed and attested word?" rather than requiring every inflected form to be pre-listed.
- Decide the validity criterion explicitly: *dictionary-attested* vs *morphologically well-formed*. These differ, and the bluffing game hinges on which one "counts."
- Resolve this **before** Phase 1 of implementation — it shapes the entire validation engine.

---

## Gap 8: Timer authority

Every phase is timer-driven, but no component owns the clock. Client-local timers **will** drift and disagree, causing "I tapped in time on my screen" disputes.

**Recommendation:** one authority (server, or host in a P2P star) emits `phase_start {phase_id, duration, server_time}` and is the sole judge of whether an action beat the deadline. Client timers are cosmetic only. State this explicitly in the blueprint regardless of which network architecture wins the discussion.

---

## Priority order for closing gaps

| # | Gap | Why first |
|---|---|---|
| 1 | Scoring table (Gap 1) | Everything else balances against it |
| 2 | Dictionary licensing (Gap 7) | Legal/feasibility blocker; shapes the engine |
| 3 | Bluff economy simulation (Gap 4) | Cheapest way to validate the core fantasy |
| 4 | Draft fairness model (Gap 3) | Decides network authority requirements |
| 5 | Disconnect rules (Gap 6) | Shapes state-model design |
| 6 | Suffix hand + root pool cleanup (Gaps 2, 5) | Quick spec edits once the above are set |
