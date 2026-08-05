# Uydurum — Honest Gameplay Review

Scope: rules and mechanics as specified in BASICS.md and BLUEPRINT.md (§1–§6), reviewed end-to-end for learnability, pacing, balance, and fun. No code exists yet, so this is a design review — the cheapest possible time to fix what's below.

## Verdict up front

- **The core loop is genuinely good.** Build-reveal-accuse with morphology-constrained bluffs is original, culturally resonant, and solves Balderdash's biggest flaw (obvious-garbage fakes) by construction. This is a real game worth building.
- **Fun to play: yes, for the right audience, once past the hump.** Mid-core Turkish word-game players will find real depth. Casual party players will bounce off the rule stack.
- **Fun to learn: currently no.** A first-timer must absorb ~18 interacting rules, then make blind ranked picks under a 15-second timer before ever touching the fun part. This is the single biggest threat to the game.
- **One real exploit, one structural failure.** The scoring pipeline makes flagging free for bluffers (breaks your own strategy triangle), and the 3-player configuration — your minimum player count — is degenerate on three independent axes.

---

## 1. What already works — keep all of this

| Mechanic | Why it's good |
|---|---|
| Composition constraint (no free typing) | Every bluff is plausible by construction. This is the design's best idea — it makes the deduction layer *about morphology intuition*, not spotting typos. |
| Fragment tokens (-kaha) | Engineered bluff fuel. Inventive, and the branching-factor precompute gives you a tuning dial. |
| False flag rewards the accused | Creates the "play a real word that sounds fake" bait strategy — the best emergent play in the design. Protect this dynamic. |
| Forced bluffs from bad boards | A player whose draft can't produce a valid word *must* bluff. This keeps the flag layer alive even when voluntary bluffing is -EV — the systems feed each other. Likely unintentional; definitely valuable. |
| Picked board (public *what*, hidden *who*) | Clean information design. Gives deduction material without enabling targeting. |
| Blind simultaneous everything | Latency-fair, no tap races. Correct call for mobile netplay. |
| Round-scoped transfers, steal floor at 0 | Bounds tilt per round, prevents pile-on misery for destitute players. |
| Disconnect/dropout handling | Production-grade: seat persistence, pool arithmetic invariance, rage-quit forfeiture. Nothing to fix. |
| Deterministic resolution + audit log | Trust and replayability. Also your onboarding tool — see §6. |

---

## 2. Learnability — the biggest risk

**Concept count.** Before their first informed decision, a new player must hold: blind top-3 ranking, wave resolution, catch-up-wins-contests, the priority button, block windows, blocked-carry-forward, one-chance suffix removal, non-exclusive hidden picks, the picked board, drop actions, the 3-suffix minimum and its top-up schedule, fragments, vowel harmony morphing, uydurum vs valid, the one-flag rule, the bluff pot, the false-flag penalty, the unused-suffix penalty, and its unchallenged-bluff waiver. Comparable social bluffing games: Skull ≈ 4 rules, Coup ≈ 9, Balderdash ≈ 3. You are shipping a euro-game manual on a party-game body.

**Timer × complexity.** 15 seconds to *rank three* roots by suffix-branching potential is an expert skill. Round 1 shows the whole match pool: at 6 players × 6 rounds that is **36 roots on a phone screen in 15 seconds**. Tier grouping helps; it doesn't save this. New players will pick semi-randomly, which quietly teaches them that the draft doesn't matter.

**"Block" is a misnomer and will be systematically misplayed.** Since picks are non-exclusive and hidden, blocking denies nothing meaningful *this turn* — its real function is **preservation**: keep a suffix alive (it's the only kind that returns) until a turn where your one action is free to take it. Every new player will read "block" as denial, use it wrong, and conclude it's pointless. Rename it (e.g., *sakla* / save) and the mechanic teaches itself.

**Fragments are a newcomer trap.** A novice picks *-kaha* believing it's productive, builds an accidental uydurum, gets flagged, pays the pot. Nothing in the rules protects them. Recommendation: never mark fragments in matches (marking would leak intent via the picked board), but *do* surface branching hints in Training Mode so the knowledge is learnable offline. Your Antrenman mode is the right vehicle — currently it teaches construction but nothing about fragment recognition.

---

## 3. Pacing — the block window is dead air

During another player's block window there is literally nothing to do or watch: the table is hidden until the owner confirms.

| Players | Suffix session | Your active time in it | Staring at nothing | Idle share |
|---|---|---|---|---|
| 4 | 100 s | ~55 s | 45 s | ~45% |
| 6 | 150 s | ~75 s | 75 s | 50% |

Half of the longest phase of every round is spent waiting, per player, on the platform (mobile) with the shortest attention tolerance. Downtime is the #1 party-game killer, and this is the worst kind — no information to study, no anticipation object on screen.

**Zero-rule-change fix:** during others' block windows, open the player's own board in a **word-preview sandbox** driven by the Dart `WordEngine` you already ship for instant preview. Players tinker with combinations while they wait. Dead air becomes the game's homework time, and it makes the 15-second construction window feel fair instead of frantic.

Match length itself is fine: ~3 min/round at 4p (≈10-minute default match), ~3.5–4 min at 6p. The 6-round option at 6 players (~22+ min of sustained attention) is the only configuration I'd worry about.

---

## 4. Balance findings (with math)

### 4.1 The free-flag exploit — P0, breaks the strategy triangle

The resolution pipeline is: 1) word points → 2) unused penalties (waived for unchallenged bluffers) → 3) caught-bluff pots → 4) false-flag transfers, *"50% of the flagger's current round score if positive"* → 5) steals.

- A bluffer whose bluff survives enters step 4 at exactly **0** (no word points, penalty waived, no pot paid). 50% of 0 = **flagging costs them nothing**, and their steal lands afterward at step 5, untouched.
- A *caught* bluffer enters step 4 **negative** (pot already paid at step 3). The "if positive" clause: they also pay nothing.

So **every bluffer flags for free, every round**, while an honest player with an 80-point word risks 40 to flag. The dominant line is "always bluff + always flag": zero marginal flag cost, chance at a pot share, chance at the steal. Your strategy triangle claims each option punishes the others — rules-as-written, bluff+flag punishes nothing and is punished only by being flagged.

Fix options, best first:
1. **A wrong flag voids your own bluff payoff** (no steal, no penalty waiver that round). Thematic ("a dirty flag dirties your bluff"), self-balancing, no new numbers.
2. Flat false-flag fee (e.g., −40 to round score, allowed to go negative) so the penalty binds at zero and below.
3. Reordering steps 4/5 does **not** fix it (caught bluffers stay negative). Don't bother.

### 4.2 Flag economics punish exactly the wrong players

The false-flag cost scales with the flagger's own round score. Break-even confidence to flag (6p, assuming a 2-way pot split):

| Flagger's round score | Wrong-flag cost | Break-even suspicion |
|---|---|---|
| 0 (bluffer) | 0 | **0%** — always flag |
| 70 (decent word) | 35 | ~29% |
| 120 (great word) | 60 | ~41% |

The better your word, the less you can afford to police bluffs — yet the flat 25/victim steal means strong players lose the same as weak ones to unchallenged bluffs. There's a charitable reading (players having a bad round get a live comeback lever as detectives — that's decent for engagement), but combined with 4.1 the flag corner belongs to bluffers and strugglers while your best-word player is structurally a spectator-victim. Fixing 4.1 mostly resolves this; consider also letting a correct flag refund the flagger's steal exposure explicitly (it already does implicitly — worth surfacing in UI).

### 4.3 The bluff pot doesn't scale with the word economy

Pot = 25 × (players − 1), while word scores (~40–120) are player-count independent:

| Players | Unchallenged upside | Typical good word | Bluff attractiveness |
|---|---|---|---|
| 3 | +50 | ~65 | Structurally bad — bluff only when your board is dead |
| 6 | +125 | ~70 | Dominant variance play |

Voluntary bluffing is near-pointless at 3p and swingy at 6p. Rough EV check at 6p: bluffing beats a 65-point honest word only above ~74% survival odds — a knife-edge mixed equilibrium that flat per-player scaling makes fragile at both ends. You already flagged pot size as a `dictpack simulate` tuning candidate; I'd make the pot a function of round economy (e.g., anchored to a fraction of the round's median word score) rather than purely of player count, and tune from there.

### 4.4 Three-player games are degenerate — and 3 is your advertised minimum

Independent failures at N=3:

1. **Board space collapses:** max picks = 3 turns = 3, forced minimum = 3 → every player ends with exactly 3 suffixes, always. No hoarding decision, no board texture.
2. **The drop action is dead:** swapping a suffix costs 2 of your 3 pick windows, and the top-up refills you with a random one anyway.
3. **Bluff pot = 50** vs ~65 for an ordinary word (see 4.3): the game's headline mechanic is -EV even when it *works*.
4. Top-ups start force-feeding at turn 1, so the "draft" is substantially the dealer.

Either redesign 3p (two picks per turn fixes 1 and 2 at a stroke; pot rescale fixes 3) or raise the minimum to 4 and be honest about it. Shipping a degenerate mode at your minimum count is how the first three-friend group churns.

### 4.5 Root-length luck, amplified in the final round

+10/letter includes the root, so a dealt 6-letter root outscores a 3-letter root by 30 before any skill applies. The draft launders this — you chose your root. But the **final round deals roots randomly**, so the highest-stakes round has the highest luck share and the least agency, and the catch-up mechanic (contested roots go to the trailer) is also gone exactly when trailing players need it. Two cheap fixes, either works:

- Keep the draft in the final round — the pool is exactly N roots, blind ranking and your wave rule still resolve it deterministically. The "deal, not a contest" rationale is arithmetic convenience, not a design need.
- Or score suffix letters ×10 with a flat root credit, removing root-length variance everywhere at once.

### 4.6 Small inconsistencies

- **Tie-break #3 "earliest submission" reintroduces the latency race** your architecture decision record explicitly rejects everywhere else. Replace with root-alphabetical or go straight to shared win (your tie-break #4).
- Round 1's catch-up rule is a no-op (all scores 0 → button decides all contests) — fine, but in a 3-round match that's a third of the game where the advertised comeback mechanic doesn't exist.
- Multiple false-flaggers resolve "in draft-priority order on *current* round score" — if a flagger is also receiving a transfer in the same step, order changes amounts. Deterministic, but arbitrary-feeling. Consider computing all step-4 amounts from the step-3 snapshot.

---

## 5. Rule ambiguities to pin down before implementation

1. **Dropped suffixes — where do they go?** "Taken ones never show up again" — does a taken-then-dropped suffix return to circulation, or vanish? Unstated in both docs; affects the drop action's entire meaning.
2. **Is a bare-root submission legal?** Roots are dictionary-verified words. If root-alone is submittable, it's a guaranteed-valid "fold" option (eat ~−12 in penalties, bank ~40–50). That's arguably good for the game as a floor — but decide and state it, because it also interacts with the ≥3-suffix minimum's purpose.
3. **Fragment visibility:** are fragments visually distinguishable from productive suffixes in the match UI? (Recommended: no in matches, yes in Training — see §2.)
4. **Collusion channel:** a false flag is a voluntary 50% gift to any player you choose. Irrelevant among friends; a real problem if ranked/leaderboard play ever ships. Note it now, solve it when ranked exists.

---

## 6. Onboarding plan — how "fun to learn" becomes yes

The showdown (build → reveal → accuse) is the fun core and it's *simple*. The draft machinery is where the complexity lives. So teach in that order:

1. **First-match mode:** skip blocks (auto-none), skip drops, deal 3 suffixes automatically, keep only pick-a-root + build + flag. That's a complete, fun game with ~6 rules. One match, players get the fantasy.
2. **Second match:** add the suffix session. **Third:** add blocking (renamed). Progressive disclosure, gated by matches played, skippable by veterans.
3. **Post-round score breakdown UI** driven by the audit-log event order you already specified (word points → penalties → pots → transfers → steals, line by line). Nothing teaches a 5-step pipeline like watching your own round replay through it. The server work is already done; this is pure client surface.
4. **Training Mode carries fragment literacy** (see §2) — it's currently construction-only, which teaches half the skill the real game tests.

---

## 7. What to measure in `dictpack simulate` / playtests

- Equilibrium bluff rate per round (healthy target: roughly 20–35%; near-0% or >50% means the pot is mistuned — check per player count).
- Flag precision and flag participation, split by flagger's round score (validates/refutes §4.2 after the §4.1 fix).
- Share of rounds with zero flags and zero bluffs ("silent rounds" — deadness indicator).
- Score contribution split: root letters vs suffix letters vs bonuses vs transfers (how much is draft luck worth?).
- Round-score variance at 3p vs 6p (quantifies §4.3/§4.4).
- Forced-bluff frequency (boards that cannot produce any valid word) — you want this > 0 but modest; it's your flag-layer heartbeat.

---

## 8. Priorities

| Priority | Item | Ref |
|---|---|---|
| P0 | Close the free-flag exploit (wrong flag voids bluff payoff) | §4.1 |
| P0 | Fill block-window dead air with the WordEngine sandbox | §3 |
| P0 | Fix or drop 3-player mode | §4.4 |
| P0 | First-match progressive onboarding | §6 |
| P1 | Final-round draft instead of random deal (or suffix-only letter scoring) | §4.5 |
| P1 | Resolve ambiguities 1–2 in the rules docs | §5 |
| P1 | Replace the earliest-submission tie-break | §4.6 |
| P1 | Post-round audit-log breakdown UI | §6 |
| P2 | Rename "block", tune pot scaling via simulate, Training-mode fragment hints | §2, §4.3 |

## Bottom line

You asked whether this is fun to learn and play. Honest answer: **play, yes — learn, not yet.** The showdown loop is original and strong, the information design is clean, and several subsystems (disconnects, determinism, forced bluffs) are better than what ships in most released games. But the current design front-loads its two weakest phases — a rushed draft and a wait-heavy suffix session — before every payoff, carries one genuine scoring exploit, and is degenerate at its own minimum player count. All of it is fixable, most of it cheaply, and you've already built the right levers (simulate, audit log, Training Mode) to fix it with evidence instead of taste.
