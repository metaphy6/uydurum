# End-to-End Project Review — Uydurum

**Reviewed:** entire repository at `b0069bc` (3 commits, 6 markdown files, no code) · **Date:** 2026-08-04
**Relation to prior reports:** [blueprint-assessment.md](blueprint-assessment.md) reviewed the *pre-pivot* draft (P2P mesh, FFI, no scoring). This review covers what exists today and supersedes that verdict where they differ.

> **Status update (2026-08-04, same day):** every hard defect in §3 and every hygiene item in §5 was resolved in BLUEPRINT.md the same day this review landed. §4.1 resolved (hosting/inviting free for everyone; membership re-scoped to the Letter Forge room option). §4.2 consciously accepted with gating re-scoped to opt-in premium rooms. §4.3 addressed via the uydurum terminology decision and a coverage program (dispute loop + TDK fact-check policy). §4.4 rewritten. §6 decided and specced as the blueprint's new Product Baseline section. See the [Addendum](#addendum-2026-08-04--the-trajectory-reconsidered) for the reconsidered trajectory verdict.

## Verdict

The turnaround since the first assessment is real: the P2P/FFI fantasy is gone, the architecture is right-sized, licensing is genuinely resolved, and the draft-resolution design is now better than most shipped party games. But this review found **two defects in the core rules that break the game as written** — one of them introduced by a recent "fix" — plus a growing pattern that is its own risk: every commit since day one has added specification, and none has added code.

Score: **Concept 8/10 · Game design 6/10 · Technical architecture 8/10 · Spec consistency 5/10 · Progress toward a playable build 1/10**

---

## 1. What the repository actually is

Three commits over two days: the blueprint plus five reports, then two rounds of rule edits and monetization additions. There is no `client/`, no `server/`, no `tools/`, no `deploy/` — the monorepo tree in the blueprint is aspiration, not description. That is fine on day three. It stops being fine the moment further spec-polishing displaces Phase 1 Task 1. The last two commits are exactly that displacement: rule rewording and two new monetization systems.

## 2. What genuinely improved since the first assessment

Credit where due — the first report's major complaints were all addressed:

1. **Architecture pivot executed.** Server-authoritative WebSocket, exactly as recommended in [p2p-network-analysis.md](p2p-network-analysis.md). The broken cross-peer crypto is gone entirely.
2. **Scoring exists.** A concrete table, round-scoped transfers, tie-breaking. (Holes remain — §3.2 — but the skeleton is there.)
3. **Dictionary licensing is actually solved**, not hand-waved: TDD hunspell-tr (MPL-2.0) + Zemberek lexicon (Apache-2.0), vendored verbatim, curated via overlays, compiled by a build-time pipeline. TDK is explicitly never touched. This was the project-blocking gap and it is closed on paper.
4. **The FFI question is settled correctly** (pure Dart, measured triggers for reconsidering) per [offline-training-mode-assessment.md](offline-training-mode-assessment.md).
5. **Wave-based draft resolution is genuinely good design.** Blind top-3 ranking, one deterministic rule at every depth, catch-up built in, priority rotation, zero randomness in contests. This is better specified than the draft in most shipped digital board games.
6. **The dictpack workbench concept (branching factor, bluff surface, Monte Carlo) is the right answer** to "v1 values, tune via playtesting" — it turns balance questions into computable ones.
7. **Infrastructure is right-sized:** Compose now, k8s scaffolded-not-required, one config model, health probes. The over-engineering criticism no longer applies to the infra sections.

## 3. Hard defects (things that are broken as written)

### 3.1 The root pool arithmetic does not work — the game cannot deal roots past round 2

The current rule: the match starts with `players + 6` roots, **never replenished**; every player drafts one root **every round**; rounds = players. Consumption is therefore `players²`, but supply is `players + 6`. Walk the blueprint's own 5-player example (11 roots, 5 rounds):

| Round | Pool at start | Roots consumed | Pool at end |
|---|---|---|---|
| 1 | 11 | 5 | 6 |
| 2 | 6 | 5 | 1 |
| 3 | 1 | 5 | **impossible — four players get no root** |
| 4–5 | 0 | — | — |

The formula balances only at the 3-player minimum (9 supplied, 9 needed — and even then, zero roots remain, so the "final round: distribute remaining unclaimed roots" rule is dead code in every player count). At 6 players the pool dies in round 2.

The git history shows how this happened: the original rule (`players + 6`, **replenished to full each round**) was arithmetically sound; commit `acdb1cf` removed replenishment to make the pool "shrink each round" without re-checking the consumption math, and `b0069bc` kept the broken rule while polishing everything around it. The design-gap report's original recommendation (per-round pools of `players + 2`) and the pre-`acdb1cf` rule both work; the current rule does not. **Fix before anything else — this is one sentence in the blueprint, but it is the game.**

### 3.2 The bluff economy's central rule is missing, and one reading makes challenging irrational

"Correct flag: takes the bluffer's full round score." What *is* a bluffer's round score? The blueprint never says whether an **unchallenged fake word earns word points** (letter points, chain bonus, longest-word bonus) as if it were real. Both readings are consistent with the text, and they produce opposite games:

- **Reading A — fake words earn no word points:** the bluffer's round score at flag time is ≈0, so a correct flag wins ≈nothing, while a false flag costs 50% of your own round score. Rational players never flag; unchallenged bluffs then reliably harvest 25 points per opponent; always-bluff dominates. The economy collapses — this is the exact failure Gap 4 was supposed to prevent, reintroduced through an undefined term.
- **Reading B — unchallenged fakes score as real (Fibbage rule):** the economy is coherent: bluffing risks a large pot (word points + steals) against a real reward for the flagger.

Reading B is almost certainly the intent. It must be written down, along with two adjacent decisions: can an unchallenged fake win *longest word of the round* (and the round itself)? And what is the **resolution order** when a round contains multiple bluffs and multiple flags (flags in server-arrival order first? steals after? floors applied per-step?)? Transfers are order-dependent — the "structured audit logs of scoring events" promised in Phase 4 cannot exist without a defined order.

### 3.3 The suffix draft reintroduces the latency race the root draft just eliminated

Gap 3 was closed for roots with blind simultaneous picks — the blueprint even brags the draft is "latency-immune." Phase 2 then specs a **live shared suffix pool** picked from during a 10-second window, with drops returning to the pool mid-window. If two players tap the last copy of `-lIk`, who gets it? First intent to arrive — i.e., the fiber player beats the 4G player, every time. The identical fairness problem, one phase later, unsolved. Options: blind-rank suffixes too, snake-pick by draft priority (3 s/pick), or duplicate suffixes enough that contention is impossible ("duplicated symmetrically" is currently too vague to guarantee that — 20 total or 20 × players is never stated).

### 3.4 Phase 3 has no clock and no structure

Phases 1 and 2 are 10 seconds each. Phase 3 — composing the word, submitting, revealing, flagging — has **no duration and no sub-phase order**. When does flagging open: during composition, or after simultaneous reveal? Can you flag after scores display? The showdown is the emotional core of the game and it is the least-specified phase in the document. (Note: the round-winner tie-break "earliest submission" and first-flag-wins both re-admit mild latency sensitivity. Acceptable — but say so, since the blueprint elsewhere claims latency immunity.)

### 3.5 "Server-verified" rewarded ads, as described, are client-asserted

"Ad-completion callbacks post an XP-multiplier event to the server" — a client posting an event after a client-side SDK callback *is* client assertion; any modified client posts the same event without watching anything. Actual server verification means the ad network calls **your server** (e.g., AdMob Server-Side Verification postbacks with signature checks). One sentence to fix, but the current text claims a property the described mechanism doesn't have — worth catching now because the same pattern (client-asserted vs server-verified) is the project's whole security stance.

### 3.6 Phase-order wrinkle: reconnects need session tokens one phase before auth exists

Phase 3 requires session-token reconnect snapshots; JWT/auth arrives in Phase 4. Trivially solvable (anonymous per-connection session tokens in Phase 3, upgraded in Phase 4) — but sequence it deliberately or Phase 3's reconnect test criteria can't be met.

## 4. Product risks (legal decisions, not spec bugs)

### 4.1 The Premium Membership paywalls the viral loop

Social deception games grow through friend groups — that is the entire distribution story of this genre. The new subscription gates **hosting and inviting**: no friend group plays a private match until someone subscribes, from day one, before the game has proven it's worth paying for. Public matchmade play staying free doesn't compensate — bluffing is weakest among strangers. The first assessment warned against "taxing the supply side"; the subscription doubles that tax. Recommendation: free hosting at launch, convert to premium *perks* (specialty dictionaries via Host Tickets — already well-designed for exactly this) once retention exists. A subscription can gate *extras*; it should not gate *the social graph*.

### 4.2 Letter Forge quietly breaks the game's own plausibility axiom

The composition constraint is load-bearing: "free-typed gibberish is impossible by construction," and the precomputed **bluff surface** bounds what fakes can exist. Letter Forge (add/change/delete one letter) breaks both: in all-premium rooms, submissions are no longer root+suffix-composable, so the mental test every flagger runs ("could their tiles build this?") stops working, and the bluff surface no longer bounds anything. It also lets edited *fake* words exist outside the simulated economy. Either constrain it (edited result must still be dictionary-valid — turning it into a "reach words your hand can't" perk, not a bluff enabler) or accept that all-premium rooms are a different game and spec that game. Also flagged: with pool fragmentation in a 3–6 player synchronous title, the all-premium condition will trigger so rarely the feature may effectively never run.

### 4.3 Dictionary coverage *is* perceived fairness

Turkish morphology is productive far beyond what hunspell expansion attests; players **will** build legitimate words the bundle lacks and be told they're invalid — in a game whose entire premise is judging word validity. The bounded 20-suffix inventory is a strong existing mitigation (the reachable space is small and auditable via the branching precompute), but chained suffixes grow it combinatorially. Two additions needed: a validity-dispute feedback loop (one tap: "this is a real word" → logged for curation) and an explicit statement that the in-game oracle is "this bundle," not "Turkish."

### 4.4 The MPL claim is overconfident by one notch

"MPL's file-level share-alike never triggers" — compiling `tr_TR.dic` into `words.dawg` arguably produces a file containing MPL-covered material, so the honest position is: obligations are *minimal and trivially satisfiable* (ship notices, point at upstream source — the manifest already does this), not *nonexistent*. Low risk; adjust the wording, not the plan.

## 5. Documentation hygiene

- **Every cross-reference in BLUEPRINT.md is broken:** six links point to `reports/…`; the folder is `initial-reports/`. (The taxonomy tree plans a future `reports/` — either rename the folder now or fix the links.)
- **Inter-document contradiction:** the blueprint says final-round leftover roots distribute "in draft-priority order"; [design-gaps-recommendations.md](design-gaps-recommendations.md)'s status note still says "randomly." (Both describe a rule that's dead code anyway — §3.1.)
- **[blueprint-assessment.md](blueprint-assessment.md) now criticizes a document that no longer exists** (P2P, FFI, merged headers, chat preamble). Mark it as historical or future readers will distrust the current blueprint for sins it already fixed.
- **Unresolved choices presented as resolved:** "Riverpod/Bloc" (pick one); suffix pool sizing "duplicated symmetrically" (define the count).
- **Cost-plan drift:** [server-cost-plan.md](server-cost-plan.md) Stage 0 assumes SQLite and Redis-at-Stage-2; the blueprint mandates Postgres + Redis from day one. Defensible (dev/prod parity) — reconcile the docs one way or the other.
- **Casually-specced hard feature:** "drains in-flight matches (or snapshots them to Redis for handoff)" — cross-node live-match handoff is a genuinely hard distributed-systems feature hiding in a parenthetical. Cut it; drain-and-finish is enough for years.

## 6. Missing entirely (end-to-end completeness)

| Area | Status |
|---|---|
| Matchmaking & room discovery (queue? room codes? public list?) | Not specced — yet monetization already depends on "matchmade public play" |
| XP / levels / progression | Referenced by three systems (training rewards, ad multipliers, weekly-pool gates); defined nowhere |
| KVKK/GDPR, ad-consent flows (UMP/TCF), store billing rules for subscriptions/IAP, age rating | Zero mention — KVKK applies to a Türkiye-targeted app with accounts and ad SDKs; subscriptions must use platform billing |
| Analytics/telemetry | Flagged in the first assessment, still absent — you cannot tune the bluff economy without event data |
| Moderation/admin for the core game (nicknames, kick/ban, dictionary hotfix path) | Only the Weekly Pool has moderation specced |
| Platform targets (Android-first? iOS same day?) | Never stated; affects ads, billing, CI, release cadence |

## 7. The trajectory risk

> Reconsidered in the [Addendum](#addendum-2026-08-04--the-trajectory-reconsidered) after all fixes landed.

The blueprint has crossed the threshold where more specification has negative marginal value. Evidence from this review: the two most recent commits refined prose and **added two monetization systems** to a game with no code, while introducing (§3.1) and preserving (§3.2) core-rule defects that any playable prototype — even hot-seat, even hardcoded — would have exposed in the first session. Paper does not push back; players do. The plan is good enough to build against. The risk is no longer building the wrong thing; it is not building.

## 8. Recommended next steps, in order

1. **Fix the root-pool rule** (§3.1) — restore replenishment or per-round pools. One paragraph.
2. **Define the bluff-resolution rules** (§3.2) — unchallenged-fake scoring, longest-word eligibility, transfer ordering. One section.
3. **Decide suffix-draft fairness** (§3.3) and give Phase 3 a clock (§3.4). One section.
4. **Fix the six broken links** and reconcile the final-round contradiction (§5). Ten minutes.
5. **Defer the Premium Membership decision** (§4.1) out of the blueprint's committed scope; keep Host Tickets.
6. **Then freeze the blueprint and start Phase 1 Task 1.** The next commit that touches only markdown should be treated as a warning sign.
7. When the engine exists, `dictpack simulate` (§3.2's economy questions) is already specced as the tool to validate the numbers — use it before the first public playtest, not after.

---

## Bottom line

Three days in, this is one of the better-specified indie multiplayer designs you'll find — and it currently describes a game that breaks in round 3, an economy whose central term is undefined, a fairness fix applied to one draft and forgotten in the next, and a subscription aimed at its own growth engine. Every one of those is cheap to fix today and expensive to discover in a playtest. Fix the four rule defects, cut the subscription from v1, freeze the document, and write `server/internal/config` this week. The concept earned a real plan; the plan has now earned a prototype.

---

## Addendum (2026-08-04) — the trajectory, reconsidered

Every finding above was addressed in BLUEPRINT.md within hours of this review. What is now specced:

| Finding | Resolution |
|---|---|
| §3.1 root pool | `players × rounds` (= `players²`) at match start, no replenishment — empties exactly at match end; final round skips the draft and deals the last `players` roots randomly (a deal, not a contest — the one sanctioned randomness) |
| §3.2 bluff economy | Uydurum words earn no word points; every bluff carries a fixed pot of `25 × (players − 1)` — collected as steals when unchallenged, paid in full to correct flaggers (equal split, deterministic remainder) when caught; one flag per player per round; four-step deterministic resolution pipeline that doubles as the audit-log order |
| §3.3 suffix draft | Contention removed structurally: `players` copies per token, one copy per hand, drops re-enter the pool for everyone except the dropper |
| §3.4 clocks | 15 s root draft · 15 s suffix draft · 20 s construction & submission · simultaneous reveal · 15 s **blind** flag window (which also deleted the last first-tap race) |
| §3.5 ads | AdMob-style SSV postbacks with signature + nonce; client callback is UX-only |
| §3.6 tokens | Anonymous server-issued session tokens in Phase 3, bound to accounts in Phase 4 |
| §4.1 subscription | Hosting and inviting free for everyone; membership now gates only the opt-in Letter Forge room type |
| §4.2 Letter Forge | Accepted by design; re-scoped to premium-only rooms set at creation, open rooms unaffected |
| §4.3 coverage | “Uydurum = outside the bundle” is now the stated, branded validity model; dispute loop feeds `additions.dic`; TDK used for individual fact-checks only, bulk import only with written permission |
| §4.4–§5, §6 | Wording, links, tree, Riverpod, cost-plan drift, SIGTERM parenthetical all fixed; §6 items decided in a new Product Baseline section |

**The reconsidered verdict.** The responsiveness cuts both ways. Fixing these on paper first was the correct order — they were one-sentence fixes that would have been rework in code — so today's markdown was investment, not displacement. But the ledger now reads: four commits, all specification, zero software, and this addendum closes the last *known* spec defect. There is nothing left in this repository that another day of writing improves and nothing further a reviewer can falsify from an armchair — every remaining unknown (does the bluff economy feel tense? is 20 seconds enough to compose? is the draft readable at 25 roots?) is only answerable by a running game. The trajectory risk is therefore now binary: either the next commit creates `server/`, or §7's pattern has merely acquired a better-audited blueprint. The blueprint should be treated as frozen by its own review trail. Phase 1, Task 1 is the entire remaining recommendation.
