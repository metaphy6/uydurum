
# 📖 Uydurum — Project Architecture & Implementation Roadmap

## 🧱 Tech Stack

| Layer | Technology | Role |
|---|---|---|
| Client | Flutter (pure Dart, no FFI) | UI, WebSocket client, Dart `WordEngine` (online morphing preview + offline Training Mode) |
| Backend | Go | Authoritative game server: lobbies, timers, drafts, word validation, scoring |
| Database | PostgreSQL | Durable data: profiles, match results, cosmetics, leaderboards |
| Cache / Pub-Sub | Redis | Lobby→node routing, session presence, cross-node pub-sub |
| Transport | WebSocket (JSON messages) | Single realtime channel between client and server |
| Infrastructure | Docker Compose (now) → Kubernetes + Terraform (later) | Everything containerized from day one |

Architecture decision record: server-authoritative model chosen over P2P mesh — a trusted authority wins on latency fairness, anti-cheat, and operating cost.

## 🕹️ Game Rules & Mechanics Blueprint

### 1. Match Parameters

* Player Count: Minimum 3, maximum 6 players per lobby to start; a started match continues as long as **at least 2 players remain connected** (see §6).
* Round Clock: draft 15 s (skipped in the final round) + suffix block window 15 s + `players` pick turns × 15 s + construction 15 s + flag window 15 s. Worst case at 6 players: **2 min 30 s per round** (2 min 15 s in the final round — 15 s block + 1 min 30 s picks + 15 s construction + 15 s flags). Ready unanimity (§7) can only shorten a round, never extend it.
* Round Calculation: a match runs **3 rounds by default**; the host can raise this at lobby creation up to a **maximum of 6**. The count is fixed once the match starts — dropouts never shrink it (see §6, Disconnects & Dropouts).
* Victory Conditions:
  * Longest-word bonus: the round's longest valid word earns +30 chips, shared equally on ties (§5) — there is no other per-round title.
  * Match Winner: the player with the biggest chip stack at match end (everyone seeds at 100 — §5).

### 2. Phase 1: The Root Draft (15 Seconds, Blind Pick)

* Pool size: the match begins with exactly `players × rounds` dictionary-verified roots (e.g., 5 players over 3 rounds = 15 roots). Claimed roots are removed and are **not** replaced: each round consumes exactly `players` roots, so the pool empties precisely at match end — never a root short, never one over. Dropout seats are system-played rather than removed (see §6), so consumption never varies.
* Final round: the draft is skipped — the last `players` remaining roots are dealt randomly, one per player (server RNG, seed logged for auditability). This is a deal, not a contest: randomness never resolves a contested pick.
* Every root is verified against the dictionary to ensure real words can branch from it. The draft screen groups the pool by difficulty tier so round 1's larger pool stays readable in the window.
* Draft priority (the "button"): each round every player holds a unique priority rank (1..N). The button marks priority 1 and rotates by one seat every round, so no player camps the advantage. Priority numbers are displayed in the draft UI before the window opens, making every resolution verifiable at a glance.
* Interaction — blind simultaneous pick: during the 15-second window each player secretly ranks their **top-3** roots. Nobody sees others' choices while drafting; all picks are revealed simultaneously when the timer ends. Once resolved, **every player's claimed root stays publicly visible for the rest of the match** — suffix ownership and block authorship are the hidden information (§3).
* Conflict resolution — wave-based, fully deterministic (one rule at every depth: *contested root goes to the player furthest behind; exact ties go to the better draft priority*):
  * Wave 1 (first picks): an uncontested first pick is claimed outright. If 2+ players ranked the same root first, the player with the **smallest chip stack** wins it (built-in catch-up mechanic); if stacks tie, the better draft priority wins.
  * Wave 2 (second picks): losers fall back to their second pick. Collisions resolve by the same rule; a second pick already claimed in wave 1 falls through to wave 3.
  * Wave 3 (third picks): same rule, recursively.
  * Exhausted all ranks: remaining players receive unclaimed roots in draft-priority order, by pool display order — no randomness anywhere in resolution.
  * Stacks are frozen at draft start: winning an earlier wave never changes a player's standing within the same draft.
* No selection made: the server assigns the player an unclaimed root after all ranked players are resolved, in draft-priority order by pool display order — the only path that bypasses ranking, and it is self-inflicted. This includes disconnected players: they rank nothing but are still dealt a root at window close, so a mid-round reconnect rejoins a playable hand.
* Rationale: blind picks make the draft latency-immune — no "fastest tap" network races — and deterministic resolution means chance may shape the pool, but never decides a contest between two players (the final-round deal assigns leftovers randomly, but contests nothing).

### 3. Phase 2: The Suffix Session (15 s Shared Block + 15 s Pick Turns)

* The session: each round, before word construction, suffixes are drafted from **one shared table** of **players + 10** suffixes (4 players → 14; 16 at the 6-player maximum), dealt once per round and visible to **all players from the session's first second**. No part of the session happens on a private screen others must wait behind.
* Block window (15 seconds, everyone simultaneously): the session opens with a single shared window in which every player — with the full table and all public roots (§2) in view — may secretly **block up to 3** suffixes. At window close all blocks merge (a suffix blocked by several players is simply blocked) and are revealed **without attribution**: grayed out, no names. A blocked suffix is non-selectable for the entire round — including by its own blocker.
* Pick turns (15 seconds each, `players` turns, everyone acts every turn): the table then runs for exactly `players` synchronized turns. In each turn every player may act **once, in secret**: pick one selectable suffix onto their board, drop one from their board, or pass. Picks are hidden and **non-exclusive** — several players may pick the same suffix in the same turn; there is no scarcity contest and nothing to race for. At each turn's close, picked suffixes leave the table and join the picked board.
* One chance each, blocks carry forward: a suffix **picked** by anyone appears on no table again this match; an unpicked, unblocked suffix at round close does not return either. **Blocked-but-unpicked suffixes are the only ones that return** — they take slots on the next round's table (which always totals players + 10) and arrive unblocked, until someone blocks them again; leftovers at match end expire. Boards still clear at every round's scoring (used → consumed, unused → discarded).
* Public information: every player's drafted root is visible all match (§2); **who picked which suffix — and who blocked which — is hidden**. A small always-on **picked board** lists every suffix picked so far this match — one entry each, no duplicate counts, no owner names — so players reason about what is in circulation without knowing who holds what.
* The minimum: your board must hold **at least 3 suffixes** when word construction begins. The system enforces this over the session's **last three turns**: at each of those closes it deals one table suffix to anyone behind the pace — 1 by the third-last, 2 by the second-last, 3 by the last — and the final close always completes the board to three, so no sequence of drops can dodge the minimum. Top-up dealing order: selectable suffixes first, then blocked ones, then — if the table has run bare — a duplicate of a suffix already picked this round (duplicate holdings are legal by design, so the minimum is always satisfiable). A player who never acts (disconnected seats included) is dealt exactly one per turn across those last three. In a 3-player lobby every turn is one of the last three, so top-ups can begin at turn 1.
* No cap: one pick per turn means a board can never exceed `players` suffixes — and every suffix you can't fit into your word forfeits the clean-sweep bonus (§5), so hoarding still pays a price.
* Rationale: a single simultaneous block window plus everyone-acts pick turns eliminates the previous design's per-turn dead air — every second of the session is an active window for every player, and session length is a fixed function of player count: 15 s + `players` × 15 s (1 min 45 s at the 6-player maximum).

### 4. Phase 3: The Showdown & Bluffing Mechanic

* Structure — three server-owned steps: a **15-second construction & submission window** (compose from your root + board suffixes and lock in one final word; no submission = no word chips), a **simultaneous reveal** of all words, then a **15-second blind flag window**. Flags stay hidden until the window closes — no bandwagoning, no fastest-tap race — and all resolve together at close. Each player may flag **at most one** word per round, **never their own**, and **only while solvent (stack above 0)**: self-flags and broke-player flags are rejected server-side, so a bluffer cannot hedge their own pot and a broke player cannot police the table.
* Players submit real dictionary words or invented ones and bluff them through. Terminology: a word outside the dictionary bundle is an **uydurum** — not a lie, not a dictionary word, the in-between state the game is named for. In-game "valid" always means *attested in the active dict-pack bundle*, and the UI presents it exactly that way.
* Bluff & flag eligibility — skin in the game: a player whose **stack is 0 (broke)** at construction-window open may neither submit an uydurum nor flag a word that round — word chips are the only way back. The check is frozen at window open (same freeze rule as the draft) and covers both windows. The client refuses to lock in a non-valid word for a broke player and hides their flag controls; if either intent arrives anyway, the server rejects it — the uydurum is scored as **no submission**, the flag is simply dropped. The 100-chip seed (§5) means everyone is solvent in round 1 — the old round-1 lockout is gone. Priced-in ripples: stacks are public, so a broke player's submission is known-real (flagging it is a guaranteed false flag; the 20-chip fee applies as usual), and the whole table knows who is out of the detective pool.
* Composition constraint (plausibility pressure): every submission — real or bluffed — must be built from the player's own drafted root and board suffixes, with vowel harmony applied by the engine. The bluff is that the *combination* is not a dictionary word; free-typed gibberish is impossible by construction.
* Word chips go to dictionary-valid words only: an uydurum mints no letter chips and no bonuses, and only valid words compete for the longest-word bonus. A bluff's only upside is the pot. There is no leftover-suffix penalty — the clean-sweep bonus (§5) is the anti-hoarding incentive, so an unused suffix costs exactly one thing: the sweep.
* The Deception Loop — one flat **60-chip pot** per bluffed word, identical at every table size:
  * Unchallenged bluff: the bluffer collects 60 chips, paid equally by the opponents — `60 / (players − 1)` each: 30/20/15/12 at 3/4/5/6 players, always whole numbers.
  * Caught bluff: the bluffer pays the 60-chip pot, split equally among all correct flaggers — 2 catchers → 30 each, 3 → 20, 4 → 15, 5 → 12, always whole numbers, no remainder rules.
  * False flag: a player who flags a genuine dictionary word pays a flat **20-chip fee** to the word's owner. The fee binds at every stack size — no score-scaled percentage — and it keeps the bait play (a real word that smells fake) deliberately profitable.
* Settlement — one snapshot, no order dependence (also the audit-log event order): 1) word chips and bonuses are minted → 2) every flag and pot debt is computed from that single post-mint snapshot and applied at once. Every payment is capped by the payer's stack (pay what you have; stacks floor at 0, never negative); if a stack cannot cover everything, pots settle before fees. With flat amounts everywhere, no transfer depends on another transfer's outcome — draft-priority ordering and per-victim floor bookkeeping are gone from the pipeline. The seed, pot, fee, and sweep values are the `dictpack simulate` tuning constants (protocol in §5).

### 5. Scoring: The Chip Economy

One number per player: a **stack**, seeded at **100 chips** at match start and floored at **0** — nobody goes negative, and 0 = broke = no bluffing, no flagging (§4). Words **mint** new chips from the bank; gambles **move** chips between players; the biggest stack at match end wins. The scoreboard shows two lines per player — **word chips** and **gamble chips** — so building and gambling read as two visibly different games.

Scoring table (v3 values — tuned via the protocol below):

| Event | Chips |
|---|---|
| Valid word | +1 per letter, root included (*gözlükçü* = 9) |
| Clean sweep — every suffix in hand used in the word | +15 (replaces any leftover penalty; hard to earn — the whole board must chain validly) |
| Longest valid word of the round | +30, shared equally on ties (2-way: 15 each; 3-way: 10; 4-way: 7.5 — the only halves in the game) |
| Uydurum (non-dictionary word) | no word chips — its upside is the pot (§4) |
| Unchallenged bluff | +60, paid equally by all opponents (30/20/15/12 each at 3/4/5/6 players) |
| Caught bluff | −60, split equally among all correct flaggers |
| False flag (max one flag per player per round) | −20, paid to the accused |
| No word submitted | no word chips, no bonuses |
| Uydurum or flag attempted while broke | rejected server-side (§4): the word is scored as no submission, the flag is dropped |

No tie-breaking cascade: the longest-word bonus is shared on ties, so nothing downstream needs an order — the old "fewer suffixes" and "earliest submission" rules are gone (no server-arrival races anywhere). A round with no valid word simply mints no word chips and awards no longest-word bonus.

Playtest tuning protocol — `dictpack simulate` acceptance bands, one lever per failure mode:

| Symptom | Healthy band | The one lever |
|---|---|---|
| Bluff rate runs hot | 20–35% of eligible rounds | False-flag fee 20 → 15 (cheaper policing squeezes survival rates) |
| Flag spam | ~40–60% flag participation | Fee 20 → 30 |
| Bluffing dies (<~15%) | — | Survived uydurum also scores its letters (+1/letter) — last resort, adds a rule line |

The seed (100), pot (60), fee (20), and sweep (+15) are the tunable constants; the structure is fixed.

### 6. Disconnects & Dropouts

* Seats persist: a dropped-out player's seat is never removed — the system auto-plays it exactly like a fully passive player (no ranks → a root is still dealt at window close; no suffix actions → the last-three-turn top-ups still fill the board to three; no submission → no word chips minted). Root-pool consumption and the round count are both fixed at match start, so departures never shrink either — the `players × rounds` pool arithmetic holds regardless of who is present.
* Grace window: a disconnected player has 20 seconds to reconnect before counting as dropped for the below-minimum check; the seat is auto-played from the moment of disconnect until its owner returns, however long that takes.
* Rejoin: a reconnecting client receives a full server snapshot of the current round state (session-token based, per the network architecture) and picks its seat back up mid-phase — scores from auto-played rounds stand.
* Below minimum: the match continues as long as **at least 2 players are connected** (auto-play covers the rest). If connected players drop below 2, the current round is finished — auto-played to completion — and the match ends, scored as-is.
* Finalization & the 50% penalty: at match end every seat's final stack is recorded — but a player who **quit mid-match**, or who is **still disconnected at finalization**, is recorded at **50% of their final stack**. Rejoining and finishing the match connected avoids the penalty entirely; the auto-played stretch (no words, no chips minted) is already its own cost, so walking away is never score-neutral.
* No bot takeover: absent players are never replaced by AI stand-ins — bots are farmable in a bluffing game.

### 7. Pace Controls: Ready & Poke

* Ready: in the three work windows — the suffix **block window**, every **pick turn**, and the **construction window** — a player whose action is locked (or who chooses to pass) may tap **Ready**; it is final for that window. The moment every seat is ready the window closes early and the next phase starts. **Auto-played seats count as always ready**, so a match running at the 2-connected minimum (§6) still fast-forwards. The root draft and the flag window are exempt — the blind contest and the blind accusation always run their full 15 seconds.
* Poke: once per wait window, any player may poke one player who has not readied yet. The target's device gives a **light haptic buzz** and a brief screen shake — pressure, not punishment: pokes are anonymous, have no score effect, and the once-per-window cap is enforced server-side.
* Transport: both are ordinary WebSocket intents (`ready`, `poke`) validated by the server (window identity, once-per-window, target not yet ready). Ready unanimity emits the same phase-advance event as timer expiry, so clients need no special handling.

---

## 🎮 Game Modes & Live Ops

### 1. Offline Training Mode ("Antrenman")

A single-player, fully offline practice mode with **no bluffing and no uydurumcu detection** — pure word construction under time pressure:

* Loop: the player picks a root from a rotating pool and constructs as many valid words as possible with a set of constructive suffixes before the timer expires (e.g., 60-second drills).
* Rewards: a tiny, daily-capped XP trickle for completed sessions — deliberately low-stakes because offline progress is client-side and unverifiable by design; offline XP never feeds competitive leaderboards.
* Sync: XP earned offline is submitted on reconnect and server-capped (max credited sessions/day) before being applied to the profile.
* Engine — **decided: pure Dart, no FFI.** The client's `linguistics/` `WordEngine` (shared with the online preview) carries the on-device dictionary and morphology. FFI remains only a contingency with measured triggers (dictionary load >1–2 s or memory kills on real low-end devices).
* Licensing note: shipping a dictionary inside the client is redistribution — resolved by the dict-pack bundle (MPL-2.0 + Apache-2.0 sources, license-clean by construction; see Word Engine §3).

### 2. Weekly Uydurum Pool (Community Word Event)

A weekly live-ops loop where the community invents a Turkish equivalent for a foreign/loan word found in the Turkish dictionary:

* Monday: the server announces the week's challenge — a foreign-derived word — alongside last week's winner.
* Monday–Friday (proposal window): each player may submit **exactly one** proposal; submissions are immutable (no editing, no deleting). Duplicate proposals collapse into a single entry credited to the earliest submitter.
* Saturday–Sunday (voting window): one vote per player, enforced server-side; voting for your own proposal is not allowed. Vote tallies stay hidden until the window closes to prevent bandwagoning.
* Monday: winner announced (most votes; ties broken by earliest submission) and the new uydurum week begins.
* Infrastructure: a pure PostgreSQL feature (proposals, votes, weekly schedule) plus a scheduled job — no realtime channel needed; fits the existing server as-is.
* Safeguards: minimum account level to propose/vote, profanity/moderation filter at submission time, a report mechanism, and rate-limited endpoints.

---

## � How-to-Play Clip (Ship-Gated)

A ≤45-second, watch-don't-read onboarding clip: a first-timer should be able to follow their first match after one viewing.

* Format constraints: real UI capture only; silent-autoplay friendly — big brutalist captions carry the story, audio optional; one idea per beat, one beat per phase; every beat readable at phone size.
* Storyboard (7 beats, 4–6 s each):
  1. Hook — "Invent a word. Get away with it." A real word morphs letter by letter into an uydurum.
  2. Pick a root — the draft screen, three secret favorites, simultaneous reveal.
  3. Block — the shared table appears; three suffixes gray out with a thunk.
  4. Pick — suffixes fly to hidden boards; the picked board fills, no names shown.
  5. Build — root + suffixes snap together; vowel harmony visibly morphs the seam.
  6. Reveal & flag — all words up at once; caption "one of these is invented"; a flag lands.
  7. Score — the strategy triangle in one line: safe word / bold bluff / sharp flag. Logo out.
* Ship gate: produced on the final production UI and released with prod — **explicitly deprioritized until then**; no clip work is scheduled while gameplay, engine, and netcode areas remain open.

---

## �🎨 Visual Identity: Neo-Brutalism Design Matrix

The entire game client will be rendered using a strict Neo-Brutalism framework to guarantee text scannability and optimal drawing performance on low-end mobile devices:

* Containers: High-saturation background fills (`#FFF176`, `#FF5252`, `#00FFCC`) wrapped in thick, solid borders (`border: Border.all(width: 3.5, color: Colors.black)`).
* Shadows: Rigid, hard-edged box offsets with zero blurring (`boxShadow: [BoxShadow(color: Colors.black, offset: Offset(6, 6), blurRadius: 0)]`).
* Feedback Signifiers: Pure neon mint green (`#00FFCC`) for genuine word confirmations and hot warning pink (`#FF3366`) for challenge flag triggers.

---

## 🏛️ Codebase Taxonomy & Separation of Concerns

Monorepo layout — client, server, and infrastructure live together and version together:

```text
uydurum/
├── client/                      # Flutter application
│   └── lib/
│       ├── core/
│       │   ├── config/          # Client config loader (server URL, feature flags)
│       │   └── network/         # GameTransport abstraction + WebSocket implementation
│       ├── data/
│       │   ├── models/          # GameState, Player, Word, Suffix DTOs (mirror server protocol)
│       │   └── repositories/    # Implementations of domain repository contracts
│       ├── domain/
│       │   ├── entities/        # Pure, immutable business objects
│       │   ├── repositories/    # Abstract repository contracts
│       │   └── usecases/        # Client-side flows (JoinLobby, SubmitWord, FlagBluff)
│       ├── presentation/
│       │   ├── state/           # Riverpod state for the server-driven phases
│       │   ├── screens/         # MainMenu, Lobby, Draft, Showdown, Scoreboard
│       │   └── widgets/         # Brutalist UI (BrutalistTimer, SuffixCard, PickedSuffixBoard, BluffButton, ReadyButton, PokeNudge)
│       └── linguistics/         # Dart WordEngine: morphing + on-device dictionary (online preview & offline training)
├── server/                      # Go authoritative game server
│   ├── cmd/
│   │   └── uydurumd/            # main.go — wiring, config load, graceful shutdown
│   ├── internal/
│   │   ├── config/              # Single typed config struct, loaded from YAML
│   │   ├── transport/           # WebSocket handling, message codec, connection lifecycle
│   │   ├── lobby/               # Lobby lifecycle, matchmaking, reconnect grace windows
│   │   ├── game/                # Phase state machine, timers, draft arbitration, scoring
│   │   ├── words/               # Dictionary (DAWG/trie in RAM), morphology, validation
│   │   └── store/               # Postgres repositories, Redis presence/pub-sub
│   └── migrations/              # SQL migrations (golang-migrate)
├── deploy/
│   ├── compose/                 # docker-compose.yaml + per-env overrides (local now)
│   ├── k8s/                     # Manifests/Helm chart (future — scaffolded, not required to run)
│   └── terraform/               # Provider-agnostic modules (future)
├── tools/
│   └── dictpack/                # Go CLI: vendor dictionaries → versioned dict-pack bundles; designer workbench & simulator
├── data/
│   └── vendor/                  # Verbatim upstream sources + licenses (TDD hunspell-tr MPL-2.0, Zemberek lexicon Apache-2.0)
└── configs/                     # Centralized config: base.yaml + <env>.yaml overlays (incl. gameplay/tuning.yaml)
```

Separation rule: `server/internal/game` and `server/internal/words` contain **all** game rules and validation — the client renders state and sends intents; it never decides outcomes.

---

## ⚙️ Word Engine Specification

The word engine lives **server-side in Go** and is the single source of truth for validity and scoring. The client carries a lightweight Dart mirror for instant visual feedback only.

### 1. Server-Side Dictionary (Authoritative)

* The 100,000+ word dictionary is compiled into a DAWG or trie held entirely in server RAM (<10 MB) — lookups are microsecond-range with zero I/O in the hot path.
* The client does ship its own dictionary (for instant preview and offline Training Mode), but extracting or tampering with it gains nothing: online verdicts come exclusively from the server's copy, and offline rewards are daily-capped and never feed competitive leaderboards — so no client-side encryption scheme is needed.
* Dictionary sourcing — **resolved:** TDD `hunspell-tr` (MPL-2.0; the `tr_TR` dictionary LibreOffice ships) supplies the attested-forms set; Zemberek-NLP's lexicon (Apache-2.0; `master-dictionary.dict` + `first-10K` frequency list) supplies POS-tagged roots and the CI morphology oracle. TDK content is never scraped or shipped. Compilation model in §3.

### 2. Linguistic Suffix Morphing & Validation

* Suffixes are stored as abstract tokens (e.g., `-{lIk}`, `-{mAk}`).
* Inventory tokens need not be official morphemes: the pipeline also cuts attested words into **fragment tokens** (e.g., *kahkaha* minus the root *kah* yields the fragment *-kaha*). Fragments attach and harmonize like any other token; they mostly build uydurums — deliberate bluff material — and the branching-factor precompute (§3) tells them apart from productive suffixes. Fragments are what let the inventory scale to match-scoped exclusion.
* The Go engine applies vowel harmony rules to transform tokens into concrete strings (e.g., `kafa + -{lIk}` → `kafalık`, `göz + -{lIk}` → `gözlük`), then checks the result for validity and scores it.
* The client's `linguistics/` module implements the same rules in Dart as a full `WordEngine` (morphing + on-device dictionary). Online it powers **instant preview** — cosmetic, only the server verdict counts; offline it is the sole engine for the Training Mode.
* The morphology interface is language-abstract from day one (`Morphology` interface in `server/internal/words`) so non-Turkish modules can plug in later.

### 3. Dictionary Data Pipeline (`tools/dictpack`)

Both dictionary sources stay **out of the runtime entirely** — a build-time pipeline compiles them into one versioned, game-ready bundle consumed by both engines:

* Sources: verbatim upstream files live under `data/vendor/` with their licenses — TDD `hunspell-tr` (`tr_TR.dic` + `tr_TR.aff`, MPL-2.0) and the Zemberek lexicon (Apache-2.0). Vendor files are never edited; curation lives in overlay files (`exclusions.txt`, `additions.dic`, `catalogues/` topic lists). MPL-2.0 obligations on the compiled bundle stay minimal by construction — `manifest.json` carries the license notices (surfaced in the client credits screen) and points at the verbatim MPL sources vendored under `data/vendor/`.
* Compilation: `tools/dictpack` (Go CLI, containerized, runs in CI) expands `.dic`+`.aff` into attested word forms; filters proper nouns, out-of-charset entries, and length outliers with Turkish-locale casing; intersects Zemberek roots with the attested set; and emits the **dict-pack bundle**: `words.dawg`, `roots.tsv` (POS, frequency tier, branching factor, topic tags), `suffixes.json` (the constructive-suffix inventory + attach rules — attested suffixes plus generated fragments, sized above worst-case match consumption: `rounds × players × (players + 10)` fresh table slots, i.e., 576 tokens for a maxed 6-player, 6-round lobby), `manifest.json` (version tag, checksums, license notices).
* Branching-factor precompute: for every root, the pipeline walks all suffix chains (bounded depth) against the DAWG and records how many attested words are reachable with the current inventory. The root-pool sampler guarantees every drafted root has ≥K real derivations — no dead-end roots — and difficulty tiers fall out of the same number. The same statistics certify which root-length ranges and topic catalogues a bundle can serve (§ Monetization 6–7). The complement (morphable-but-unattested forms) is each root's **bluff surface**, the raw material of the deception loop.
* Coverage program: in-game validity means *attested in the bundle*; anything outside it is an uydurum by definition, and the client states this openly (help/credits screens — it is the game's namesake, not fine print). To keep the bundle honest, the client ships a one-tap validity dispute (“this is a real word”): disputes are logged server-side, reviewed by curation, and accepted words enter `additions.dic` in the next bundle version. Individual disputed words may be fact-checked against official references (TDK GTS) — verifying that a word exists is a fact lookup, not redistribution of the dictionary; any *systematic* import of TDK content would require TDK's written permission first.
* Runtime consumption: `server/internal/words` (Go) and `client/lib/linguistics` (Dart) load the same bundle; neither ships Hunspell or Zemberek code. Zemberek runs only as a JVM oracle in CI, verifying the twin vowel-harmony implementations through the golden-file corpus.
* Version discipline: both sides load the same bundle tag (e.g., `tr-2026.08`); the server embeds it in `phase_started`, so a stale client knows its previews may drift — the server verdict still rules. Premium slang/dialect dictionaries are additional bundles in the same format, selected per lobby; root-catalogue rotations ride the same hot-swap mechanism as ordinary bundle bumps.

#### Designer Workbench & Tuning (`dictpack` subcommands)

`dictpack` doubles as the balancing tool: a seeded, config-driven sampler and Monte Carlo simulator over roots, suffix inventories, and difficulty. The workbench and the game server read the **same bundle and the same tuning file** — what was explored is exactly what ships.

Every knob lives in one versioned file, `configs/gameplay/tuning.yaml` — nothing hardcoded:

```yaml
seed: 42                          # reproducible randomness — same seed, same pool
roots:
  frequency_weights: {common: 0.6, mid: 0.3, rare: 0.1}   # likelihood
  min_branching_factor: 8         # every root guarantees ≥8 real words
  min_bluff_surface: 15           # ≥15 morphable non-words (bluffability)
  length_range: [2, 10]           # hard feasibility clamp — premium rooms choose a sub-range
  length_weights: {short: 0.05, core: 0.85, long: 0.10}   # 2 / 3–6 / 7–10 letters — default deal is mostly core
  pos_mix: {noun: 0.7, verb: 0.3} # specification
  exclude_tags: [proper, archaic, offensive]
difficulty:
  curve: round_progressive        # later rounds draw rarer tiers
  tier_shift_per_round: 0.1
suffixes:
  inventory: v1                   # named inventory sets, swappable wholesale
```

Workbench subcommands:

| Command | What it answers |
|---|---|
| `dictpack roots --sample 11 --seed 7` | A draft pool exactly as the server would deal it, with per-root stats: tier, branching factor, bluff surface, best achievable word |
| `dictpack derive kafa --inventory v1` | Every attested derivation of a root, scored by the scoring table — with the morphable fakes alongside |
| `dictpack simulate --matches 10000 --players 5` | Monte Carlo over full drafts with simple bot policies: longest-word distributions, score spreads, dead-draft probability, how often the catch-up mechanic flips a contested root |
| `dictpack diff tuning-a.yaml tuning-b.yaml` | Side-by-side comparison of two parameter sets before committing one |

* Why simulation, not just sampling: the scoring table is "v1 — to be tuned via playtesting." Monte Carlo answers the questions humans are slow at *before* playtests: does the +30 longest-word bonus dominate outcomes? does `round_progressive` actually tighten score gaps? what is the expected bluff-success rate given each root's bluff surface? Tune the YAML until distributions look right, then spend scarce playtest hours validating feel.
* Output discipline: every command takes `--csv`/`--json` for spreadsheet analysis, and every run prints its seed + tuning-file checksum so any interesting pool is reproducible in a bug report. A local web UI (`dictpack serve`) may come later; CLI + CSV covers tuning work and ships in Phase 2, not after it.

### 4. Server-Authoritative Anti-Cheat

* All game-deciding events — draft claims, word submissions, bluff flags, score transfers — are sent as *intents* and resolved exclusively by the server. A modified client can render anything it wants; it cannot change a verdict or a score.
* The server owns the phase clock: every phase window (15 s root draft, 15 s suffix block + 10 s pick per turn, 15 s submission, 15 s flagging) opens and closes on server time, and late intents are rejected server-side. Client timers are display-only.
* This replaces the previously specced cross-peer hash verification entirely — with a trusted authority, no peer consensus scheme is required.

---

## 🌐 Network Architecture: Server-Authoritative WebSocket

```text
[ Flutter Client 1 ] ─┐                          ┌─────────────────┐
[ Flutter Client 2 ] ─┼─( WSS / JSON intents )─► │  GO GAME SERVER │◄─►[ Redis ]
[ Flutter Client N ] ─┘   ◄─( state events )──   │  - Lobby & auth │     routing/presence
                                                 │  - Phase timers │
                                                 │  - Draft judge  │◄─►[ PostgreSQL ]
                                                 │  - Word engine  │     profiles/results
                                                 │  - Scoring      │
                                                 └─────────────────┘
```

* Protocol: One persistent WebSocket per client. Clients send **intents** (`rank_roots`, `block_suffix`, `approve_blocks`, `pick_suffix`, `drop_suffix`, `submit_word`, `flag_bluff`); the server responds with **state events** (`phase_started`, `roots_resolved`, `round_scored`) broadcast to the lobby. Messages are versioned JSON with sequence numbers for ordered replay.
* Fair arbitration: the root draft, every suffix turn, and the bluff-flag window are all blind-simultaneous — collected privately, revealed and resolved at window close — so no game-deciding event is a fastest-tap race; the server is the single referee for every deadline.
* Reconnect model: A dropped client reconnects with its session token — anonymous and server-issued per connection until Phase 4 binds tokens to accounts — and receives a full state snapshot of the current round (20-second grace window; dropout rules in Game Rules §6).
* Lobby→node affinity: Every lobby lives on exactly one server instance (Redis maps `lobby_id → node`); no cross-node game state. This is the property that makes horizontal scaling — and later Kubernetes — trivial.
* Live match state is held in server memory only; PostgreSQL records durable outcomes (profiles, match results), Redis handles routing, presence, and cross-node pub-sub.

---

## 📦 Infrastructure & Deployment

### 1. Containerization Principles

* Every runnable component ships as a container from day one: `server` (distroless/scratch Go image, single static binary), `postgres`, `redis`, and dev-only tooling (migrations runner, adminer).
* Images are multi-arch (amd64/arm64) and environment-agnostic — the **same image** runs under Compose locally, Kubernetes later, and any Terraform-provisioned host. Behavior differs only by mounted configuration.
* The server is built stateless-by-design: all durable state in PostgreSQL, all coordination state in Redis, live match state in memory with lobby→node affinity. This is the exact property Kubernetes needs, designed in now rather than retrofitted.

### 2. Centralized Configuration (No Scattered Env Vars)

* All configuration lives in `configs/` as clean, layered YAML: `base.yaml` holds every key with sane defaults; `local.yaml`, `staging.yaml`, `prod.yaml` are thin overlays that override only what differs.
* The Go server loads exactly one merged config into a single typed struct (`internal/config`) at boot and fails fast with a clear error listing any missing/invalid keys — no `os.Getenv` calls sprinkled through business logic.
* Environment variables are reserved for exactly two things: selecting the config file (`UYDURUM_CONFIG=/etc/uydurum/config.yaml`) and injecting **secrets** (DB password, JWT signing key) via `${VAR}` interpolation inside the YAML. Secrets never live in YAML files or images.
* This maps 1:1 onto the future platforms: Compose mounts `configs/local.yaml` as a volume; Kubernetes mounts the same file as a ConfigMap with secrets from a Secret resource; Terraform templates the same file per environment. One config model, three delivery mechanisms.

### 3. Local Development — Docker Compose

* `deploy/compose/docker-compose.yaml` brings up the full stack with one command: server (live-reload via air in dev profile), PostgreSQL with auto-applied migrations, Redis, and adminer.
* Compose profiles separate concerns: `core` (server+db+redis), `tools` (adminer, dashboards), `test` (ephemeral db for integration tests). No hand-managed `.env` sprawl — a single `deploy/compose/.env` holds only secrets and the config-file path.
* The Flutter client targets `ws://localhost` in `local.yaml`-mirrored client config; a full 6-player match must be playable against the local stack with zero cloud dependencies.

### 4. Kubernetes & Terraform Readiness (Future, Designed-For Now)

* Server exposes `/healthz` (liveness) and `/readyz` (readiness, checks Postgres/Redis connectivity) plus Prometheus metrics on a separate port — required for k8s probes, useful under Compose immediately.
* Graceful shutdown: on SIGTERM the server stops accepting new lobbies, drains in-flight matches to completion, then exits — this makes k8s rolling deploys and node drains safe. Live-match handoff between nodes is explicitly out of scope: a draining node simply finishes its matches.
* WebSocket routing under k8s uses sticky sessions/lobby-affinity at the ingress; because lobbies never span nodes, no service mesh or distributed state layer is needed.
* `deploy/terraform/` is structured as provider-agnostic modules (network, database, compute) with thin provider roots — consistent with the Hetzner-first / Azure-or-AWS-later hosting strategy.
* Rule: no k8s/TF-blocking decisions in application code — no local file writes, no in-container state, no hardcoded hostnames, config exclusively via the mounted YAML model above.

---

## 💰 Monetization Systems

### 1. Rewarded Multipliers

* Mechanic: Integrating ad provider SDKs (such as Google Mobile Ads) cleanly within the Neo-Brutalism scoreboard UI. At the conclusion of a match, players can optionally watch a 30-second video to double their match XP or profile level progression points.
* Technical Impact: multipliers are granted exclusively via **ad-network server-side verification** (e.g., AdMob SSV): the ad network's servers call a verification endpoint on the game server with a signed payload; the server validates the signature and a one-time nonce, then applies the multiplier to the player profile in PostgreSQL. The client's completion callback is UX-only and grants nothing.

### 2. Cosmetic Deception Assets

* Mechanic: High-impact visual overlays that flash across the devices of opponents when a player successfully pulls off a bluff.
* Assets: Comic-book-style explosion animations, glitch screens, or high-contrast custom stamps that take over the screen. These are managed locally via custom Rive animation files.

### 3. Premium Host System

* Mechanic: Players join rooms completely free of charge, but hosting rooms with specialized slang, dialect, or technical dictionaries requires a "Premium Host Ticket." Tickets can be obtained via one-time purchases or rewarded ad engagements, driving revenue from highly active power users.

### 4. Premium Membership (Monthly Subscription)

* Mechanic: hosting rooms and inviting players is **free for everyone** — the social loop is never paywalled. Premium Membership is an optional monthly entitlement for power-user perks: the Letter Forge (§5), Custom Root Length (§6), and Root Catalogue (§7) room options; further perks (e.g., premium cosmetics) can attach later.
* Relation to Host Tickets: Premium Host Tickets gate rooms with specialized dictionaries (§3); Premium Membership gates the Letter Forge, Custom Root Length, and Root Catalogue room options — independent entitlements, independently purchasable.
* Technical Impact: membership is a PostgreSQL-backed entitlement with an expiry date, checked server-side when creating a Letter Forge room and at join intents for such rooms; expired memberships fail fast with a renewal prompt.

### 5. Letter Forge (Premium Room Option)

* Room setup: a host with an active Premium Membership may enable **Letter Forge** at lobby creation. An enabled room admits only players with an active membership (enforced server-side at join intent); with the option off (default), the room is open to everyone and the perk is inactive for all. The flag is immutable after creation.
* Mechanic: in Letter Forge rooms, players may **add, change, or delete a single letter** in their constructed word during the showdown's construction window.
* Hard constraint: the drafted **root word is immutable** — letter edits apply exclusively to the constructive (suffix-built) portion of the word; any intent touching the root is rejected.
* Technical Impact: letter edits are sent as intents and validated server-side by the Go word engine (edit position must fall outside the root span; result still scored/validated normally).

### 6. Custom Root Length (Premium Room Option)

* Room setup: a host with an active Premium Membership may set an explicit **root-length range** at lobby creation — minimum 2, maximum 10 letters — replacing the default sampler distribution. Unlike Letter Forge, the room stays **open to everyone**: the option shapes the root deck and grants nobody an in-match ability, so gating joiners would only hurt the social loop. The range is immutable after creation.
* Default (non-premium) rooms: root lengths follow the bundle's default distribution — minimum 2 letters, **mostly 3–6**, with longer roots appearing rarely (`length_weights` in `configs/gameplay/tuning.yaml`).
* Feasibility guardrail: the dict-pack precompute certifies, per bundle, which length ranges hold enough qualifying roots (branching factor, bluff surface, and pool size for a maxed lobby — 36 roots at 6 players × 6 rounds). The lobby UI offers only certified ranges and the server rejects uncertified ones — no dead drafts by construction.
* Balance note: at +10/letter, wide ranges raise draft stakes sharply (a 10-letter root out-scores a 2-letter root by 80 points before suffixes). `dictpack simulate` across candidate ranges is the acceptance gate before this option ships.
* Technical Impact: the range is a lobby-config field validated server-side against the bundle's certified ranges; the root sampler filters `roots.tsv` by length; the entitlement is checked at room creation exactly like Letter Forge (§5).

### 7. Root Catalogues (Premium Room Option)

* Mechanic: a host with an active Premium Membership may pick a **themed root catalogue** at lobby creation — general topics such as health, weather, food, sports — and the match's entire root pool is sampled from that theme. Like Custom Root Length (§6), the room stays **open to everyone**: the theme shapes the deck and grants nobody an in-match ability.
* Live-ops rotation: each catalogue declares a refresh cadence — **daily, weekly, or monthly** — and a scheduled job activates new versions on schedule. Rotating content is what makes the monthly subscription recurringly worth paying for. Running matches are unaffected: the root pool is sampled once at match start.
* Composability: catalogue and Custom Root Length stack — the sampler intersects both filters. The lobby UI offers only combinations the bundle certifies.
* Feasibility guardrail: the same certification as §6 — the dict-pack precompute verifies every catalogue, and every catalogue × length-range combination, holds enough qualifying roots for a maxed lobby (36 at 6 players × 6 rounds); uncertified selections are unofferable in the UI and rejected server-side.
* Technical Impact: roots carry **topic tags** in `roots.tsv`, curated via `catalogues/` overlay lists in the pipeline; the chosen catalogue is a lobby-config field validated like the length range; rotation ships as a bundle version bump the server hot-swaps without redeploying — and since catalogues never alter word validity, client preview bundles need no update when a catalogue rotates.

---

## 🧾 Product Baseline (v1 Decisions)

Small decisions that unblock implementation — each deliberately minimal, expanded only when live data demands it:

* Matchmaking & discovery: private rooms use 6-character room codes (free for everyone); public play is a single Quick Match FIFO queue per dictionary bundle and lobby size. No public room browser and no skill rating at v1 — both are post-retention features.
* Progression: one server-side XP track. XP events (match completed, valid word, round won, correct flag) carry values in `configs/gameplay/tuning.yaml`; levels are a fixed XP-threshold table. Levels gate Weekly Pool proposals/votes and cosmetic unlocks. Offline Training XP stays daily-capped (§ Game Modes).
* Compliance (KVKK + GDPR): anonymous device accounts by default (no PII to protect), optional account linking later; privacy notice at first launch; Google UMP consent flow before any personalized ads; in-app delete-my-data action backed by a server endpoint; 13+ age gate; all purchases exclusively through platform billing (Play Billing / StoreKit).
* Analytics: no third-party client SDK at v1 — the authoritative server already witnesses every gameplay event. The Phase 4 scoring/audit events persist to PostgreSQL; nightly jobs derive the KPIs (retention, match completion, bluff rate, flag accuracy). Client-side funnel analytics wait for the store launch.
* Moderation & admin: nickname profanity filter (same filter as the Weekly Pool), an in-match report action (logged, no automated punishment at v1), and an authenticated admin endpoint set: kick, ban (account + device), close lobby. Dictionary fixes ship as dict-pack version bumps — the server swaps bundles without redeploying; a stale client preview is cosmetic (server verdict rules).
* Platforms & release: Android first (primary Türkiye audience, cheaper device testing); iOS follows once retention is proven. CI builds both targets from day one, so the iOS gap stays a release decision, not a porting project.

---

## 🛠️ Step-by-Step Implementation Lifecycle

### Phase 1: Foundation — Monorepo, Containers & Config

* Task 1: Establish the monorepo tree (`client/`, `server/`, `deploy/`, `configs/`). Implement the layered YAML config loader in `server/internal/config` with fail-fast validation.
* Task 2: Stand up `deploy/compose/docker-compose.yaml`: Go server skeleton with `/healthz`–`/readyz`, PostgreSQL with migrations, Redis. Scaffold the Flutter client with the `GameTransport` abstraction and a working WebSocket echo loop.
* Testing Criteria: `docker compose up` yields a healthy full stack from a fresh clone with no manual steps; config validation errors are clear and complete.

### Phase 2: Word Engine (Go)

* Task 1: Implement `tools/dictpack` (expand TDD `hunspell-tr`, intersect Zemberek roots, emit the versioned dict-pack bundle — sources are settled), then the in-RAM DAWG/trie loader and lookup in `server/internal/words`.
* Task 2: Implement token-based suffix morphing with vowel harmony behind the language-abstract `Morphology` interface; implement the client's Dart `linguistics/` `WordEngine` (same rules + on-device dictionary loader) serving both online preview and the offline Training Mode.
* Testing Criteria: Golden-file test suite of root+suffix combinations passes identically on the Go engine (authoritative) and the Dart `WordEngine`; both ship the same dictionary version tag; lookups benchmark in microseconds.

### Phase 3: Realtime Game Loop

* Task 1: Implement the lobby lifecycle and the versioned intent/event WebSocket protocol with sequence numbers and reconnect snapshots, keyed by anonymous server-issued session tokens (bound to accounts in Phase 4).
* Task 2: Build the server-side phase state machine: server-owned phase timers (15 s root draft, 15 s suffix block + 10 s pick per turn, 15 s submission, 15 s blind flag window), blind root-pick resolution with catch-up tie-breaks, the `players × rounds` root pool with random final-round deal, turn-based suffix sessions (private owner block windows, hidden non-exclusive picks, one-chance tables with blocked-suffix carry-forward, the public picked board, last-three-turn top-ups to the three-suffix minimum), showdown submission, blind flag resolution (self-flags rejected) with pot splitting, and the deterministic round-scoring pipeline including unused-suffix penalties.
* Task 3: Build the corresponding Flutter screens and phase state management against the live protocol.
* Testing Criteria: A scripted 6-bot integration test plays full matches against the Compose stack, including forced mid-round disconnects/reconnects, with no state corruption; contested blind picks always resolve to exactly one winner per root.

### Phase 4: Accounts, Persistence & Hardening

* Task 1: Implement auth (JWT sessions) and bind Phase 3's anonymous session tokens to accounts; player profiles and match-result persistence in PostgreSQL; Redis presence and lobby→node routing.
* Task 2: Harden the intent pipeline: rate limiting, server-side deadline enforcement, input validation at the protocol boundary, and structured audit logs of scoring events — persisted to PostgreSQL as the v1 analytics event stream (see Product Baseline).
* Testing Criteria: A deliberately modified client (forged scores, late intents, replayed messages) cannot alter any outcome; load test sustains hundreds of concurrent lobbies on one node.

### Phase 5: Monetization & Polish

* Task 1: Integrate rewarded ads via ad-network server-side verification (SSV) callbacks, the UMP consent flow, and platform billing; implement cosmetic unlocks, Premium Host Tickets, and the Premium Membership entitlement (Letter Forge, Custom Root Length, and Root Catalogue rooms) in PostgreSQL, plus the catalogue-rotation scheduled job.
* Task 2: Ship the auxiliary modes: Offline Training Mode (Dart `WordEngine`, daily-capped XP sync) and the Weekly Uydurum Pool (Postgres schema, scheduled job, proposal/voting screens).
* Task 3: Run performance profiling on client (layout paints on low-end devices) and server (allocation/GC under lobby load).
* Testing Criteria: Ad-completion events are verified server-side; entitlement checks gate premium lobbies correctly; weekly pool windows open/close on schedule with one-proposal/one-vote enforcement verified.

### Phase 6: Linguistic Abstraction & Expansion

* Task 1: Verify the `Morphology` interface isolates all Turkish-specific logic; scaffold a second-language module as proof.
* Task 2: Localize client strings and dictionary-pack selection per lobby.
* Testing Criteria: A full match runs in a second language purely via configuration and a plug-in language module — zero core-engine changes.