
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

Architecture decision record: server-authoritative model chosen over P2P mesh — rationale in [initial-reports/p2p-network-analysis.md](initial-reports/p2p-network-analysis.md), cost projections in [initial-reports/server-cost-plan.md](initial-reports/server-cost-plan.md).

## 🕹️ Game Rules & Mechanics Blueprint

### 1. Match Parameters

* Player Count: Minimum 3, maximum 6 players per lobby.
* Round Calculation: Total number of rounds in a match matches the number of active players (e.g., 5 players = 5 rounds).
* Victory Conditions:
  * Round Winner: The player who forms the longest valid word.
  * Match Winner: The player with the highest cumulative points across all rounds.

### 2. Phase 1: The Root Draft (15 Seconds, Blind Pick)

* Pool size: the match begins with exactly `players × rounds` dictionary-verified roots — and since rounds = players, that is `players²` (e.g., 25 roots for a 5-player match). Claimed roots are removed and are **not** replaced: each round consumes exactly `players` roots, so the pool empties precisely at match end — never a root short, never one over.
* Final round: the draft is skipped — the last `players` remaining roots are dealt randomly, one per player (server RNG, seed logged for auditability). This is a deal, not a contest: randomness never resolves a contested pick.
* Every root is verified against the dictionary to ensure real words can branch from it. The draft screen groups the pool by difficulty tier so round 1's larger pool stays readable in the window.
* Draft priority (the "button"): each round every player holds a unique priority rank (1..N). The button marks priority 1 and rotates by one seat every round, so no player camps the advantage. Priority numbers are displayed in the draft UI before the window opens, making every resolution verifiable at a glance.
* Interaction — blind simultaneous pick: during the 15-second window each player secretly ranks their **top-3** roots. Nobody sees others' choices while drafting; all picks are revealed simultaneously when the timer ends.
* Conflict resolution — wave-based, fully deterministic (one rule at every depth: *contested root goes to the player furthest behind; exact ties go to the better draft priority*):
  * Wave 1 (first picks): an uncontested first pick is claimed outright. If 2+ players ranked the same root first, the player with the **lowest cumulative match score** wins it (built-in catch-up mechanic); if scores tie, the better draft priority wins.
  * Wave 2 (second picks): losers fall back to their second pick. Collisions resolve by the same rule; a second pick already claimed in wave 1 falls through to wave 3.
  * Wave 3 (third picks): same rule, recursively.
  * Exhausted all ranks: remaining players receive unclaimed roots in draft-priority order, by pool display order — no randomness anywhere in resolution.
  * Scores are frozen at draft start: winning an earlier wave never changes a player's standing within the same draft.
* No selection made: the server assigns the player an unclaimed root after all ranked players are resolved, in draft-priority order by pool display order — the only path that bypasses ranking, and it is self-inflicted. This includes disconnected players: they rank nothing but are still dealt a root at window close, so a mid-round reconnect rejoins a playable hand.
* Rationale: blind picks make the draft latency-immune — no "fastest tap" network races (see [initial-reports/design-gaps-recommendations.md](initial-reports/design-gaps-recommendations.md), Gap 3) — and deterministic resolution means chance may shape the pool, but never decides a contest between two players (the final-round deal assigns leftovers randomly, but contests nothing).

### 3. Phase 2: The Suffix Draft (15 Seconds)

* Pool composition: the 20-token constructive-suffix inventory (e.g., -lık, -mak, -siz) enters the match with exactly **`players` copies of each token** (e.g., 100 tiles for 5 players), and a hand may hold at most one copy of a given token. Supply per token therefore always covers every player — pick contention is structurally impossible, so first-come picks are latency-fair because nothing is scarce.
* Hand size is a constant **3 suffixes** from round 1 onward — one rule, no phase-dependent special cases.
* Dropping a suffix returns it to the public pool immediately, where **any player except the dropper** may pick it up — the dropper cannot re-take a token they dropped within the same draft window (no drop/undo flicker).
* If a player holds fewer than 3 suffixes when the timer ends, the system fills their hand from the pool at random.

### 4. Phase 3: The Showdown & Bluffing Mechanic

* Structure — three server-owned steps: a **20-second construction & submission window** (compose from your root + held suffixes and lock in one final word; no submission = 0 round points), a **simultaneous reveal** of all words, then a **15-second blind flag window**. Flags stay hidden until the window closes — no bandwagoning, no fastest-tap race — and all resolve together at close. Each player may flag **at most one** word per round.
* Players submit real dictionary words or invented ones and bluff them through. Terminology: a word outside the dictionary bundle is an **uydurum** — not a lie, not a dictionary word, the in-between state the game is named for. In-game "valid" always means *attested in the active dict-pack bundle*, and the UI presents it exactly that way.
* Composition constraint (plausibility pressure): every submission — real or bluffed — must be built from the player's own drafted root and held suffixes, with vowel harmony applied by the engine. The bluff is that the *combination* is not a dictionary word; free-typed gibberish is impossible by construction.
* Word points go to dictionary-valid words only: an uydurum earns no letter points and no bonuses, and only valid words can win the round. A bluff's entire upside is the steal.
* The Deception Loop (all transfers operate on **round** points, never cumulative match points). Every bluffed word carries a fixed **bluff pot** of `25 × (players − 1)`:
  * Unchallenged bluff: the bluffer collects 25 points from each opponent's round score, floored at 0 per victim — the full pot when everyone can pay.
  * Caught bluff: the bluffer pays the full pot, split equally among all correct flaggers (integer split; the remainder goes to the correct flagger furthest behind in match score, ties broken by draft priority). A caught bluffer's round score may go negative — the risk is self-inflicted and exactly symmetric to the reward.
  * False flag: a player who flags a genuine dictionary word transfers 50% of their own round score to the word's owner.
* Deterministic resolution pipeline (also the Phase 4 audit-log event order): 1) word points and bonuses for valid words → 2) caught-bluff pots paid out → 3) false-flag transfers, in flagger draft-priority order, 50% of the flagger's current round score if positive → 4) unchallenged-bluff steals, in bluffer draft-priority order, floored at 0 per victim. The pot size and the 50% false-flag rate are the first candidates for `dictpack simulate` tuning.

### 5. Scoring & Tie-Breaking

Baseline scoring table (v1 values — to be tuned via playtesting):

| Event | Points |
|---|---|
| Valid word, per letter | +10 / letter |
| Longest valid word of the round (bonus) | +30 |
| Valid chain of 2+ suffixes (bonus) | +15 per suffix beyond the first |
| Uydurum (non-dictionary) word | 0 word points — its only upside is the bluff pot |
| Unchallenged bluff | collects the pot: +25 from each opponent's round score (floored at 0 per victim) |
| Correct challenge | bluffer pays the full pot `25 × (players − 1)`, split equally among all correct flaggers |
| False challenge (max one flag per player per round) | 50% of own round score to the accused |
| No word submitted | 0 |

Round-winner tie-breaking, in order:

1. Longest word → 2. Earliest submission (server arrival order) → 3. Fewer suffixes used → 4. Shared round win (all tied players receive the bonus).

### 6. Disconnects & Dropouts

* Grace window: a disconnected player has 20 seconds to reconnect; while absent, phases auto-pass for them (no picks, no submission, 0 round points) — but roots and suffix auto-fills are still dealt to them, so a reconnect mid-round rejoins a playable hand.
* Rejoin: a reconnecting client receives a full server snapshot of the current round state (session-token based, per the network architecture).
* Below minimum: if active players drop under 3, the current round is finished and the match ends, scored as-is.
* Rage-quit disincentive: leaving mid-match forfeits any pending score transfers in the leaver's favor.
* No bot takeover: absent players are never replaced by AI stand-ins — bots are farmable in a bluffing game.

---

## 🎮 Game Modes & Live Ops

### 1. Offline Training Mode ("Antrenman")

A single-player, fully offline practice mode with **no bluffing and no uydurumcu detection** — pure word construction under time pressure:

* Loop: the player picks a root from a rotating pool and constructs as many valid words as possible with a set of constructive suffixes before the timer expires (e.g., 60-second drills).
* Rewards: a tiny, daily-capped XP trickle for completed sessions — deliberately low-stakes because offline progress is client-side and unverifiable by design; offline XP never feeds competitive leaderboards.
* Sync: XP earned offline is submitted on reconnect and server-capped (max credited sessions/day) before being applied to the profile.
* Engine — **decided: pure Dart, no FFI.** The client's `linguistics/` `WordEngine` (shared with the online preview) carries the on-device dictionary and morphology. FFI remains only a contingency with measured triggers (dictionary load >1–2 s or memory kills on real low-end devices) — rationale in [initial-reports/offline-training-mode-assessment.md](initial-reports/offline-training-mode-assessment.md).
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

## 🎨 Visual Identity: Neo-Brutalism Design Matrix

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
│       │   └── widgets/         # Brutalist UI (BrutalistTimer, SuffixCard, BluffButton)
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
├── configs/                     # Centralized config: base.yaml + <env>.yaml overlays (incl. gameplay/tuning.yaml)
└── initial-reports/             # Design & architecture decision documents
```

Separation rule: `server/internal/game` and `server/internal/words` contain **all** game rules and validation — the client renders state and sends intents; it never decides outcomes.

---

## ⚙️ Word Engine Specification

The word engine lives **server-side in Go** and is the single source of truth for validity and scoring. The client carries a lightweight Dart mirror for instant visual feedback only.

### 1. Server-Side Dictionary (Authoritative)

* The 100,000+ word dictionary is compiled into a DAWG or trie held entirely in server RAM (<10 MB) — lookups are microsecond-range with zero I/O in the hot path.
* The client does ship its own dictionary (for instant preview and offline Training Mode), but extracting or tampering with it gains nothing: online verdicts come exclusively from the server's copy, and offline rewards are daily-capped and never feed competitive leaderboards — so no client-side encryption scheme is needed.
* Dictionary sourcing — **Gap 7 resolved:** TDD `hunspell-tr` (MPL-2.0; the `tr_TR` dictionary LibreOffice ships) supplies the attested-forms set; Zemberek-NLP's lexicon (Apache-2.0; `master-dictionary.dict` + `first-10K` frequency list) supplies POS-tagged roots and the CI morphology oracle. TDK content is never scraped or shipped. Compilation model in §3; original gap analysis in [initial-reports/design-gaps-recommendations.md](initial-reports/design-gaps-recommendations.md).

### 2. Linguistic Suffix Morphing & Validation

* Suffixes are stored as abstract tokens (e.g., `-{lIk}`, `-{mAk}`).
* The Go engine applies vowel harmony rules to transform tokens into concrete strings (e.g., `kafa + -{lIk}` → `kafalık`, `göz + -{lIk}` → `gözlük`), then checks the result for validity and scores it.
* The client's `linguistics/` module implements the same rules in Dart as a full `WordEngine` (morphing + on-device dictionary). Online it powers **instant preview** — cosmetic, only the server verdict counts; offline it is the sole engine for the Training Mode.
* The morphology interface is language-abstract from day one (`Morphology` interface in `server/internal/words`) so non-Turkish modules can plug in later.

### 3. Dictionary Data Pipeline (`tools/dictpack`)

Both dictionary sources stay **out of the runtime entirely** — a build-time pipeline compiles them into one versioned, game-ready bundle consumed by both engines:

* Sources: verbatim upstream files live under `data/vendor/` with their licenses — TDD `hunspell-tr` (`tr_TR.dic` + `tr_TR.aff`, MPL-2.0) and the Zemberek lexicon (Apache-2.0). Vendor files are never edited; curation lives in overlay files (`exclusions.txt`, `additions.dic`). MPL-2.0 obligations on the compiled bundle stay minimal by construction — `manifest.json` carries the license notices (surfaced in the client credits screen) and points at the verbatim MPL sources vendored under `data/vendor/`.
* Compilation: `tools/dictpack` (Go CLI, containerized, runs in CI) expands `.dic`+`.aff` into attested word forms; filters proper nouns, out-of-charset entries, and length outliers with Turkish-locale casing; intersects Zemberek roots with the attested set; and emits the **dict-pack bundle**: `words.dawg`, `roots.tsv` (POS, frequency tier, branching factor), `suffixes.json` (hand-curated 20-token inventory + attach rules), `manifest.json` (version tag, checksums, license notices).
* Branching-factor precompute: for every root, the pipeline walks all suffix chains (bounded depth) against the DAWG and records how many attested words are reachable with the current inventory. The root-pool sampler guarantees every drafted root has ≥K real derivations — no dead-end roots — and difficulty tiers fall out of the same number. The complement (morphable-but-unattested forms) is each root's **bluff surface**, the raw material of the deception loop.
* Coverage program: in-game validity means *attested in the bundle*; anything outside it is an uydurum by definition, and the client states this openly (help/credits screens — it is the game's namesake, not fine print). To keep the bundle honest, the client ships a one-tap validity dispute (“this is a real word”): disputes are logged server-side, reviewed by curation, and accepted words enter `additions.dic` in the next bundle version. Individual disputed words may be fact-checked against official references (TDK GTS) — verifying that a word exists is a fact lookup, not redistribution of the dictionary; any *systematic* import of TDK content would require TDK's written permission first.
* Runtime consumption: `server/internal/words` (Go) and `client/lib/linguistics` (Dart) load the same bundle; neither ships Hunspell or Zemberek code. Zemberek runs only as a JVM oracle in CI, verifying the twin vowel-harmony implementations through the golden-file corpus.
* Version discipline: both sides load the same bundle tag (e.g., `tr-2026.08`); the server embeds it in `phase_started`, so a stale client knows its previews may drift — the server verdict still rules. Premium slang/dialect dictionaries are additional bundles in the same format, selected per lobby.

#### Designer Workbench & Tuning (`dictpack` subcommands)

`dictpack` doubles as the balancing tool: a seeded, config-driven sampler and Monte Carlo simulator over roots, suffix inventories, and difficulty. The workbench and the game server read the **same bundle and the same tuning file** — what was explored is exactly what ships.

Every knob lives in one versioned file, `configs/gameplay/tuning.yaml` — nothing hardcoded:

```yaml
seed: 42                          # reproducible randomness — same seed, same pool
roots:
  frequency_weights: {common: 0.6, mid: 0.3, rare: 0.1}   # likelihood
  min_branching_factor: 8         # every root guarantees ≥8 real words
  min_bluff_surface: 15           # ≥15 morphable non-words (bluffability)
  length_range: [3, 6]
  pos_mix: {noun: 0.7, verb: 0.3} # specification
  exclude_tags: [proper, archaic, offensive]
difficulty:
  curve: round_progressive        # later rounds draw rarer tiers
  tier_shift_per_round: 0.1
suffixes:
  inventory: v1                   # named 20-token sets, swappable wholesale
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
* The server owns the phase clock: every phase window (15 s drafts, 20 s submission, 15 s flagging) opens and closes on server time, and late intents are rejected server-side. Client timers are display-only.
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

* Protocol: One persistent WebSocket per client. Clients send **intents** (`rank_roots`, `pick_suffix`, `submit_word`, `flag_bluff`); the server responds with **state events** (`phase_started`, `roots_resolved`, `round_scored`) broadcast to the lobby. Messages are versioned JSON with sequence numbers for ordered replay.
* Fair arbitration: the root draft and the bluff-flag window are both blind-simultaneous — collected privately, revealed and resolved at window close — so no game-deciding event is a fastest-tap race; the server is the single referee for every deadline.
* Reconnect model: A dropped client reconnects with its session token — anonymous and server-issued per connection until Phase 4 binds tokens to accounts — and receives a full state snapshot of the current round (20-second grace window; see design gap report for dropout rules).
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
* `deploy/terraform/` is structured as provider-agnostic modules (network, database, compute) with thin provider roots — consistent with the Hetzner-first / Azure-or-AWS-later strategy in [initial-reports/server-cost-plan.md](initial-reports/server-cost-plan.md).
* Rule: no k8s/TF-blocking decisions in application code — no local file writes, no in-container state, no hardcoded hostnames, config exclusively via the mounted YAML model above.

---

## 💰 Monetization Systems

### 1. Rewarded Multipliers

* Mechanic: Integrating ad provider SDKs (such as Google Mobile Ads) cleanly within the Neo-Brutalism scoreboard UI. At the conclusion of a match, players can optionally watch a 30-second video to double their match XP or profile level progression points.
* Technical Impact: multipliers are granted exclusively via **ad-network server-side verification** (e.g., AdMob SSV): the ad network's servers call a verification endpoint on the game server with a signed payload; the server validates the signature and a one-time nonce, then applies the multiplier to the player profile in PostgreSQL. The client's completion callback is UX-only and grants nothing.

### 2. Cosmetic Deception Assets

* Mechanic: High-impact visual overlays that flash across the devices of opponents when a player successfully pulls off a bluff.
* Assets: Comic-book-style explosion animations, glitch screens, or high-contrast [REDACTED] custom stamps that take over the screen. These are managed locally via custom Rive animation files.

### 3. Premium Host System

* Mechanic: Players join rooms completely free of charge, but hosting rooms with specialized slang, dialect, or technical dictionaries requires a "Premium Host Ticket." Tickets can be obtained via one-time purchases or rewarded ad engagements, driving revenue from highly active power users.

### 4. Premium Membership (Monthly Subscription)

* Mechanic: hosting rooms and inviting players is **free for everyone** — the social loop is never paywalled. Premium Membership is an optional monthly entitlement for power-user perks, starting with the Letter Forge room option (§5); further perks (e.g., premium cosmetics) can attach later.
* Relation to Host Tickets: Premium Host Tickets gate rooms with specialized dictionaries (§3); Premium Membership gates the Letter Forge room option — independent entitlements, independently purchasable.
* Technical Impact: membership is a PostgreSQL-backed entitlement with an expiry date, checked server-side when creating a Letter Forge room and at join intents for such rooms; expired memberships fail fast with a renewal prompt.

### 5. Letter Forge (Premium Room Option)

* Room setup: a host with an active Premium Membership may enable **Letter Forge** at lobby creation. An enabled room admits only players with an active membership (enforced server-side at join intent); with the option off (default), the room is open to everyone and the perk is inactive for all. The flag is immutable after creation.
* Mechanic: in Letter Forge rooms, players may **add, change, or delete a single letter** in their constructed word during the showdown's construction window.
* Hard constraint: the drafted **root word is immutable** — letter edits apply exclusively to the constructive (suffix-built) portion of the word; any intent touching the root is rejected.
* Technical Impact: letter edits are sent as intents and validated server-side by the Go word engine (edit position must fall outside the root span; result still scored/validated normally).

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

* Task 1: Implement `tools/dictpack` (expand TDD `hunspell-tr`, intersect Zemberek roots, emit the versioned dict-pack bundle — Gap 7 sources are settled), then the in-RAM DAWG/trie loader and lookup in `server/internal/words`.
* Task 2: Implement token-based suffix morphing with vowel harmony behind the language-abstract `Morphology` interface; implement the client's Dart `linguistics/` `WordEngine` (same rules + on-device dictionary loader) serving both online preview and the offline Training Mode.
* Testing Criteria: Golden-file test suite of root+suffix combinations passes identically on the Go engine (authoritative) and the Dart `WordEngine`; both ship the same dictionary version tag; lookups benchmark in microseconds.

### Phase 3: Realtime Game Loop

* Task 1: Implement the lobby lifecycle and the versioned intent/event WebSocket protocol with sequence numbers and reconnect snapshots, keyed by anonymous server-issued session tokens (bound to accounts in Phase 4).
* Task 2: Build the server-side phase state machine: server-owned phase timers (15 s drafts, 20 s submission, 15 s blind flag window), blind root-pick resolution with catch-up tie-breaks, the `players × rounds` root pool with random final-round deal, contention-free suffix drafting, showdown submission, blind flag resolution with pot splitting, and the deterministic round-scoring pipeline.
* Task 3: Build the corresponding Flutter screens and phase state management against the live protocol.
* Testing Criteria: A scripted 6-bot integration test plays full matches against the Compose stack, including forced mid-round disconnects/reconnects, with no state corruption; contested blind picks always resolve to exactly one winner per root.

### Phase 4: Accounts, Persistence & Hardening

* Task 1: Implement auth (JWT sessions) and bind Phase 3's anonymous session tokens to accounts; player profiles and match-result persistence in PostgreSQL; Redis presence and lobby→node routing.
* Task 2: Harden the intent pipeline: rate limiting, server-side deadline enforcement, input validation at the protocol boundary, and structured audit logs of scoring events — persisted to PostgreSQL as the v1 analytics event stream (see Product Baseline).
* Testing Criteria: A deliberately modified client (forged scores, late intents, replayed messages) cannot alter any outcome; load test sustains hundreds of concurrent lobbies on one node.

### Phase 5: Monetization & Polish

* Task 1: Integrate rewarded ads via ad-network server-side verification (SSV) callbacks, the UMP consent flow, and platform billing; implement cosmetic unlocks, Premium Host Tickets, and the Premium Membership entitlement (Letter Forge rooms) in PostgreSQL.
* Task 2: Ship the auxiliary modes: Offline Training Mode (Dart `WordEngine`, daily-capped XP sync) and the Weekly Uydurum Pool (Postgres schema, scheduled job, proposal/voting screens).
* Task 3: Run performance profiling on client (layout paints on low-end devices) and server (allocation/GC under lobby load).
* Testing Criteria: Ad-completion events are verified server-side; entitlement checks gate premium lobbies correctly; weekly pool windows open/close on schedule with one-proposal/one-vote enforcement verified.

### Phase 6: Linguistic Abstraction & Expansion

* Task 1: Verify the `Morphology` interface isolates all Turkish-specific logic; scaffold a second-language module as proof.
* Task 2: Localize client strings and dictionary-pack selection per lobby.
* Testing Criteria: A full match runs in a second language purely via configuration and a plug-in language module — zero core-engine changes.