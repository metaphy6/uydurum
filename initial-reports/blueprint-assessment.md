# Honest Assessment: Uydurum Blueprint

**Reviewed:** BLUEPRINT.md · **Date:** 2026-08-03

> **Historical document (note added 2026-08-04):** this assessment reviews the *original pre-pivot* draft — P2P mesh, FFI engine, undefined scoring, broken markdown. That document no longer exists; nearly every criticism below was addressed in the current BLUEPRINT.md. Kept for the decision record. Current review: [end-to-end-project-review.md](end-to-end-project-review.md).

## Verdict

The core game idea is genuinely good — a Turkish word-building bluffing game is a fresh concept with a real hook (vowel harmony as a mechanic is clever). But the technical blueprint is over-engineered for a project at this stage, and several of its most confident-sounding sections don't survive scrutiny. It reads like an AI-generated architecture fantasy (it literally still contains the chat preamble "Got it, good to know for next time..." at the top) rather than a plan grounded in what one developer or a small team can ship.

Score: **Concept 8/10 · Game design 6/10 · Technical architecture 3/10 · Shippability as written 2/10**

---

## What's genuinely good

1. **The core loop is strong.** Draft roots → draft suffixes → build words → bluff/challenge is a tight, social, replayable loop. The deception mechanic (steal points by flagging fakes, lose points on false accusations) creates real tension.
2. **Turkish agglutination is a great fit.** Vowel harmony morphing (`kafa + -lIk → kafalık`, `göz + -lIk → gözlük`) is mechanically interesting and hard to clone in English.
3. **Neo-brutalist UI is a sensible choice.** High contrast, flat fills, zero blur — cheap to render and distinctive. This part is realistic.
4. **Phase 5's language abstraction goal** is the right instinct, even if premature.

---

## Major problems

### 1. The P2P architecture contradicts the game's own trust requirements

This is the biggest flaw. The game is *about lying*, so integrity of validation is the product. Yet the blueprint chooses a serverless WebRTC mesh where every client validates locally, then tries to patch the resulting trust hole with a "cryptographic cross-peer verification" scheme that doesn't actually work:

- Hashing `PlayerID + Word + ValidityStatus + LobbySalt` with SHA-256 proves nothing if the client computing the hash is compromised. A hacked client just computes the "correct" hash for a false `ValidityStatus`. Hashes aren't signatures, and even signatures don't help when the signer is the attacker.
- "Two distinct peers must sign off" fails immediately: in a 3-player lobby, two colluding friends control the majority. There is no Sybil resistance and no honest-majority guarantee.
- The `LobbySalt` is "secret" but must be known to every peer to verify hashes — so it's not secret from the only adversary that matters (the players).

**Honest take:** you already have a server (signaling + auth + score DB, per your own diagram). Make it authoritative for word validation and scoring. A dictionary lookup is microseconds; a $5 VPS handles thousands of concurrent lobbies. The entire cross-peer crypto section can be deleted and replaced with "server validates."

### 2. WebRTC mesh is enormous complexity for tiny payloads

The actual game traffic is a handful of small JSON events per 10-second phase for ≤6 players. That is a trivial WebSocket workload. WebRTC buys you nothing here except:

- ICE/NAT traversal failures (you already admit needing TURN — which means you're paying for relay servers anyway, erasing the "cost-efficient" claim),
- host-migration and mesh-healing logic ("simulate an irregular network drop… ensure remaining mesh peers handle recoveries smoothly" — this is weeks of brutal edge-case work),
- no authoritative clock for the 10-second timers, which matters a lot when "fastest tap wins" is a core mechanic. Who arbitrates a tie in a mesh with variable peer latency? The blueprint never says.

**Honest take:** plain WebSockets through the server you already have. Latency for turn-based drafting is irrelevant; fairness and simplicity are not.

### 3. The FFI/Rust engine is premature optimization

- "Sub-millisecond dictionary lookups" — a 100k-word set in a plain Dart `HashSet` or Trie already gives you that. Dart is not the bottleneck for a hash lookup.
- Turkish vowel harmony is ~50 lines of Dart. It does not need Rust, C++, or an FFI bridge.
- Encrypted in-binary dictionary with "ephemeral key" decryption is security theater: the key ships in the binary, and anyone motivated can dump memory. Meanwhile you pay the cost of maintaining native builds for Android/iOS, FFI memory management, and cross-compilation — the single most common place hobby Flutter projects die.

**Honest take:** pure Dart for v1. If profiling later shows a real bottleneck (it won't), FFI is a Phase-6 refactor, not a Phase-1 foundation.

### 4. Game design gaps the blueprint glosses over

- **Root pool math is off.** "Players × rounds" roots for a 5-player game = 25 roots, but they're all drafted in round 1's 10-second window? Or per round? The draft/round relationship is ambiguous.
- **"Fastest tap wins" over a P2P mesh is inherently unfair** — the player with the lowest latency to the host wins drafts. This needs server-side timestamping or a different draft mechanic (simultaneous blind picks).
- **Scoring is undefined.** "Longest valid word wins the round" and "steal all round points" are stated, but the actual point values, ties, and what "all points" means are never specified. This is the heart of the game and it's missing.
- **The suffix hand rule is confusing:** "must hold exactly three suffixes by the final three rounds" — why? What happens in early rounds? Undefined.
- **Bluff economy is unbalanced on paper:** a successful unchallenged bluff steals from *every* player, which makes bluffing strictly dominant unless challenge success rates are high. Needs simulation/playtesting math, not assertion.
- **What happens with 3 players and one disconnects?** Match rules for dropouts are absent despite the P2P architecture making disconnects *more* likely.

### 5. Monetization is designed before retention exists

Rewarded ads, cosmetic bluff animations, and "Premium Host Tickets" are all fine ideas — for a game that has players. Putting monetization in Phase 4 of 5 while core scoring rules are still undefined is backwards. Also, "Premium Host Tickets" gates the thing that creates lobbies; taxing your supply side is risky when you need liquidity in a 3–6 player synchronous game (the hardest kind of game to keep populated).

### 6. Document hygiene

- The file begins with leftover LLM conversation text ("Got it, good to know for next time. I have updated…"). Remove it.
- Headers are merged on single lines (`## ⚙️ Native FFI Engine Specification## 1. Memory-Mapped…`) — broken markdown throughout.
- The title is inconsistent: the folder is `uydurumcu`, the doc says `Uydurum`, mechanics say `uydurma`. Pick one.
- No mention of: the actual Turkish dictionary source and its license (this is a real legal/practical blocker — TDK data is not freely redistributable), testing strategy beyond one-line criteria, analytics, or app-store compliance for the ad SDKs.

---

## What I'd actually build (recommended re-scope)

**v0 (prototype, weeks not months):**
- Flutter client, pure Dart. One authoritative Node/Go WebSocket server that owns lobby state, timers, validation, and scoring.
- Wordlist from an openly licensed Turkish corpus in a Dart `Set`/Trie. Vowel harmony in ~50 lines of Dart.
- Hardcode the scoring table. Playtest with real humans over hotspot Wi-Fi. Iterate on the *rules*, because that's where the game lives or dies.

**v1 (if the prototype is fun):**
- Auth, persistent profiles, matchmaking, reconnect handling — all server-side.
- Polish the brutalist UI, add Rive bluff animations.

**v2+ (only if retention proves out):**
- Monetization, language abstraction, and — only if profiling demands it — native optimization.

**Delete entirely:** WebRTC mesh, cross-peer hash consensus, encrypted in-binary dictionary, FFI-first architecture.

---

## Bottom line

You have a promising game buried under an architecture designed to sound impressive. The blueprint optimizes for the parts that don't matter yet (native performance, serverless cost savings, anti-cheat crypto) while leaving the parts that decide success (scoring rules, draft fairness, dictionary licensing, playtesting) undefined. Cut the infrastructure ambition by ~80%, define the rules completely, and get a hot-seat or single-server prototype in front of players. The idea deserves that path.
