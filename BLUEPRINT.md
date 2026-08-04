
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

Architecture decision record: server-authoritative model chosen over P2P mesh — rationale in [reports/p2p-network-analysis.md](reports/p2p-network-analysis.md), cost projections in [reports/server-cost-plan.md](reports/server-cost-plan.md).

## 🕹️ Game Rules & Mechanics Blueprint

### 1. Match Parameters

* Player Count: Minimum 3, maximum 6 players per lobby.
* Round Calculation: Total number of rounds in a match matches the number of active players (e.g., 5 players = 5 rounds).
* Victory Conditions:
  * Round Winner: The player who forms the longest valid word.
  * Match Winner: The player with the highest cumulative points across all rounds.

### 2. Phase 1: The Root Draft (10 Seconds, Blind Pick)

* Pool size: the match begins with exactly `players + 6` dictionary-verified roots (e.g., 11 roots for a 5-player game). Claimed roots are removed and are **not** replaced, so the available pool shrinks each round. In the final round, any remaining unclaimed roots are distributed to the players in draft-priority order, by pool display order.
* Every root is verified against the dictionary to ensure real words can branch from it.
* Draft priority (the "button"): each round every player holds a unique priority rank (1..N). The button marks priority 1 and rotates by one seat every round, so no player camps the advantage. Priority numbers are displayed in the draft UI before the window opens, making every resolution verifiable at a glance.
* Interaction — blind simultaneous pick: during the 10-second window each player secretly ranks their **top-3** roots. Nobody sees others' choices while drafting; all picks are revealed simultaneously when the timer ends.
* Conflict resolution — wave-based, fully deterministic (one rule at every depth: *contested root goes to the player furthest behind; exact ties go to the better draft priority*):
  * Wave 1 (first picks): an uncontested first pick is claimed outright. If 2+ players ranked the same root first, the player with the **lowest cumulative match score** wins it (built-in catch-up mechanic); if scores tie, the better draft priority wins.
  * Wave 2 (second picks): losers fall back to their second pick. Collisions resolve by the same rule; a second pick already claimed in wave 1 falls through to wave 3.
  * Wave 3 (third picks): same rule, recursively.
  * Exhausted all ranks: remaining players receive unclaimed roots in draft-priority order, by pool display order — no randomness anywhere in resolution.
  * Scores are frozen at draft start: winning an earlier wave never changes a player's standing within the same draft.
* No selection made: the server assigns the player an unclaimed root after all ranked players are resolved, in draft-priority order by pool display order — the only path that bypasses ranking, and it is self-inflicted.
* Rationale: blind picks make the draft latency-immune — no "fastest tap" network races (see [reports/design-gaps-recommendations.md](reports/design-gaps-recommendations.md), Gap 3) — and deterministic resolution means chance may shape the pool, but never decides a contest between two players.

### 3. Phase 2: The Suffix Draft (10 Seconds)

* Players select from a pool of 20 constructive suffixes (e.g., -lık, -mak, -siz), duplicated symmetrically for the number of players; multiple players can hold identical suffixes.
* Hand size is a constant **3 suffixes** from round 1 onward — one rule, no phase-dependent special cases.
* Dropping a suffix returns it to the public pool, where other players can pick it up in later rounds.
* If a player holds fewer than 3 suffixes when the timer ends, the system fills their hand from the pool at random.

### 4. Phase 3: The Showdown & Bluffing Mechanic

* Players submit their final constructed word. Long dictionary words win natively, but players may submit invented fake words ("uydurma") and bluff them through.
* Composition constraint (plausibility pressure): every submission — real or bluffed — must be built from the player's own drafted root and held suffixes, with vowel harmony applied by the engine. The bluff is that the *combination* is not a dictionary word; free-typed gibberish is impossible by construction.
* The Deception Loop (all transfers operate on **round** points, never cumulative match points):
  * Correct flag: the first player (by server arrival order) to correctly flag a made-up word takes the bluffer's full round score.
  * False flag: a player who flags a genuine dictionary word transfers 50% of their own round score to the word's creator.
  * Unchallenged bluff: the bluffing player takes a fixed 25 points from each opponent's round score, floored at 0 — no player goes negative from a single bluff.

### 5. Scoring & Tie-Breaking

Baseline scoring table (v1 values — to be tuned via playtesting):

| Event | Points |
|---|---|
| Valid word, per letter | +10 / letter |
| Longest valid word of the round (bonus) | +30 |
| Valid chain of 2+ suffixes (bonus) | +15 per suffix beyond the first |
| Unchallenged successful bluff | +25 from each opponent's round score (floored at 0) |
| Correct challenge | takes the bluffer's full round score |
| False challenge | 50% of own round score to the accused |
| No word submitted | 0 |

Round-winner tie-breaking, in order:

1. Longest word → 2. Earliest submission (server arrival order) → 3. Fewer suffixes used → 4. Shared round win (all tied players receive the bonus).

### 6. Disconnects & Dropouts

* Grace window: a disconnected player has 20 seconds to reconnect; while absent, their turn auto-passes (no submission, 0 round points).
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
* Engine — **decided: pure Dart, no FFI.** The client's `linguistics/` `WordEngine` (shared with the online preview) carries the on-device dictionary and morphology. FFI remains only a contingency with measured triggers (dictionary load >1–2 s or memory kills on real low-end devices) — rationale in [reports/offline-training-mode-assessment.md](reports/offline-training-mode-assessment.md).
* Licensing note: shipping a dictionary inside the client is redistribution — the on-device word list must be license-clean (Gap 7 applies to the client dataset too).

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
│       │   ├── state/           # Riverpod/Bloc state for the 10s phases
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
├── configs/                     # Centralized config: base.yaml + <env>.yaml overlays
└── reports/                     # Design & architecture decision documents
```

Separation rule: `server/internal/game` and `server/internal/words` contain **all** game rules and validation — the client renders state and sends intents; it never decides outcomes.

---

## ⚙️ Word Engine Specification

The word engine lives **server-side in Go** and is the single source of truth for validity and scoring. The client carries a lightweight Dart mirror for instant visual feedback only.

### 1. Server-Side Dictionary (Authoritative)

* The 100,000+ word dictionary is compiled into a DAWG or trie held entirely in server RAM (<10 MB) — lookups are microsecond-range with zero I/O in the hot path.
* The client does ship its own dictionary (for instant preview and offline Training Mode), but extracting or tampering with it gains nothing: online verdicts come exclusively from the server's copy, and offline rewards are daily-capped and never feed competitive leaderboards — so no client-side encryption scheme is needed.
* Dictionary source and licensing are resolved before implementation (see [reports/design-gaps-recommendations.md](reports/design-gaps-recommendations.md), Gap 7).

### 2. Linguistic Suffix Morphing & Validation

* Suffixes are stored as abstract tokens (e.g., `-{lIk}`, `-{mAk}`).
* The Go engine applies vowel harmony rules to transform tokens into concrete strings (e.g., `kafa + -{lIk}` → `kafalık`, `göz + -{lIk}` → `gözlük`), then checks the result for validity and scores it.
* The client's `linguistics/` module implements the same rules in Dart as a full `WordEngine` (morphing + on-device dictionary). Online it powers **instant preview** — cosmetic, only the server verdict counts; offline it is the sole engine for the Training Mode.
* The morphology interface is language-abstract from day one (`Morphology` interface in `server/internal/words`) so non-Turkish modules can plug in later.

### 3. Server-Authoritative Anti-Cheat

* All game-deciding events — draft claims, word submissions, bluff flags, score transfers — are sent as *intents* and resolved exclusively by the server. A modified client can render anything it wants; it cannot change a verdict or a score.
* The server owns the phase clock: every 10-second window opens and closes on server time, and late intents are rejected server-side. Client timers are display-only.
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
* Fair arbitration: the root draft is latency-immune by design (blind simultaneous picks, resolved at timer end with catch-up tie-breaks); the remaining time-sensitive event — bluff flagging — resolves by server arrival order, with the server as single referee.
* Reconnect model: A dropped client reconnects with its session token and receives a full state snapshot of the current round (20-second grace window; see design gap report for dropout rules).
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
* Graceful shutdown: on SIGTERM the server stops accepting lobbies, drains in-flight matches (or snapshots them to Redis for handoff), then exits — this makes k8s rolling deploys and node drains safe.
* WebSocket routing under k8s uses sticky sessions/lobby-affinity at the ingress; because lobbies never span nodes, no service mesh or distributed state layer is needed.
* `deploy/terraform/` is structured as provider-agnostic modules (network, database, compute) with thin provider roots — consistent with the Hetzner-first / Azure-or-AWS-later strategy in [reports/server-cost-plan.md](reports/server-cost-plan.md).
* Rule: no k8s/TF-blocking decisions in application code — no local file writes, no in-container state, no hardcoded hostnames, config exclusively via the mounted YAML model above.

---

## 💰 Monetization Systems

### 1. Rewarded Multipliers

* Mechanic: Integrating ad provider SDKs (such as Google Mobile Ads) cleanly within the Neo-Brutalism scoreboard UI. At the conclusion of a match, players can optionally watch a 30-second video to double their match XP or profile level progression points.
* Technical Impact: Ad-completion callbacks post an XP-multiplier event to the server, which applies it to the player profile in PostgreSQL — the multiplier is server-verified, not client-asserted.

### 2. Cosmetic Deception Assets

* Mechanic: High-impact visual overlays that flash across the devices of opponents when a player successfully pulls off a bluff.
* Assets: Comic-book-style explosion animations, glitch screens, or high-contrast [REDACTED] custom stamps that take over the screen. These are managed locally via custom Rive animation files.

### 3. Premium Host System

* Mechanic: Players join rooms completely free of charge, but hosting rooms with specialized slang, dialect, or technical dictionaries requires a "Premium Host Ticket." Tickets can be obtained via one-time purchases or rewarded ad engagements, driving revenue from highly active power users.

### 4. Premium Membership (Monthly Subscription)

* Mechanic: Hosting a room and inviting friends or other players at will requires an active **Premium Membership**, renewed monthly. Non-members can still join any room they are invited to, and matchmade public play remains free.
* Relation to Host Tickets: membership gates the *ability to host and invite*; Premium Host Tickets additionally gate rooms with specialized dictionaries — a member hosting a specialty-dictionary room needs both.
* Technical Impact: membership is a PostgreSQL-backed entitlement with an expiry date checked server-side at room-creation and invite intents; expired memberships fail fast with a renewal prompt.

### 5. Letter Forge (Premium-Only, All-Premium Rooms)

* Mechanic: During the constructive-word (suffix) rounds, premium players may **add, change, or delete a single letter** in their constructed word. The perk activates only when *every* player in the room holds an active Premium Membership — one non-premium player in the lobby disables it for everyone, keeping matches fair.
* Hard constraint: the drafted **root word is immutable** — letter edits apply exclusively to the constructive (suffix-built) portion of the word; any intent touching the root is rejected.
* Technical Impact: letter edits are sent as intents and validated server-side by the Go word engine (edit position must fall outside the root span; result still scored/validated normally). The all-premium check is evaluated at lobby lock, not per-intent.

---

## 🛠️ Step-by-Step Implementation Lifecycle

### Phase 1: Foundation — Monorepo, Containers & Config

* Task 1: Establish the monorepo tree (`client/`, `server/`, `deploy/`, `configs/`). Implement the layered YAML config loader in `server/internal/config` with fail-fast validation.
* Task 2: Stand up `deploy/compose/docker-compose.yaml`: Go server skeleton with `/healthz`–`/readyz`, PostgreSQL with migrations, Redis. Scaffold the Flutter client with the `GameTransport` abstraction and a working WebSocket echo loop.
* Testing Criteria: `docker compose up` yields a healthy full stack from a fresh clone with no manual steps; config validation errors are clear and complete.

### Phase 2: Word Engine (Go)

* Task 1: Resolve dictionary sourcing/licensing, then implement the in-RAM DAWG/trie loader and lookup in `server/internal/words`.
* Task 2: Implement token-based suffix morphing with vowel harmony behind the language-abstract `Morphology` interface; implement the client's Dart `linguistics/` `WordEngine` (same rules + on-device dictionary loader) serving both online preview and the offline Training Mode.
* Testing Criteria: Golden-file test suite of root+suffix combinations passes identically on the Go engine (authoritative) and the Dart `WordEngine`; both ship the same dictionary version tag; lookups benchmark in microseconds.

### Phase 3: Realtime Game Loop

* Task 1: Implement the lobby lifecycle and the versioned intent/event WebSocket protocol with sequence numbers and reconnect snapshots.
* Task 2: Build the server-side phase state machine: server-owned 10-second timers, blind root-pick resolution with catch-up tie-breaks, a fixed-size root pool with no replenishment, suffix drafting, showdown submission, bluff flagging, and round-scoped scoring transfers.
* Task 3: Build the corresponding Flutter screens and phase state management against the live protocol.
* Testing Criteria: A scripted 6-bot integration test plays full matches against the Compose stack, including forced mid-round disconnects/reconnects, with no state corruption; contested blind picks always resolve to exactly one winner per root.

### Phase 4: Accounts, Persistence & Hardening

* Task 1: Implement auth (JWT sessions), player profiles, and match-result persistence in PostgreSQL; Redis presence and lobby→node routing.
* Task 2: Harden the intent pipeline: rate limiting, server-side deadline enforcement, input validation at the protocol boundary, and structured audit logs of scoring events.
* Testing Criteria: A deliberately modified client (forged scores, late intents, replayed messages) cannot alter any outcome; load test sustains hundreds of concurrent lobbies on one node.

### Phase 5: Monetization & Polish

* Task 1: Integrate rewarded ads with server-verified XP multipliers; implement cosmetic unlocks, Premium Host Tickets, and monthly Premium Memberships (with Letter Forge gating) as PostgreSQL-backed entitlements.
* Task 2: Ship the auxiliary modes: Offline Training Mode (Dart `WordEngine`, daily-capped XP sync) and the Weekly Uydurum Pool (Postgres schema, scheduled job, proposal/voting screens).
* Task 3: Run performance profiling on client (layout paints on low-end devices) and server (allocation/GC under lobby load).
* Testing Criteria: Ad-completion events are verified server-side; entitlement checks gate premium lobbies correctly; weekly pool windows open/close on schedule with one-proposal/one-vote enforcement verified.

### Phase 6: Linguistic Abstraction & Expansion

* Task 1: Verify the `Morphology` interface isolates all Turkish-specific logic; scaffold a second-language module as proof.
* Task 2: Localize client strings and dictionary-pack selection per lobby.
* Testing Criteria: A full match runs in a second language purely via configuration and a plug-in language module — zero core-engine changes.