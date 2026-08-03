# P2P Network Analysis — Is WebRTC Mesh Right for Uydurum?

**Question under discussion:** *"I think I'll stay on a P2P network for fast and stable peer-to-peer connections. Am I wrong in this assumption?"*

**Short answer:** For *this specific game*, yes — both halves of the assumption ("fast" and "stable") are weaker than they sound for a turn-based mobile game. P2P is a legitimate architecture, but its real advantages don't apply to Uydurum's traffic profile, while its real costs hit Uydurum's weakest points (fairness, trust, mobile reliability). Details and a middle-ground option below.

---

## 1. Testing the "fast" assumption

**Where P2P genuinely is faster:** high-frequency, latency-critical streams — voice chat, real-time action games sending 20–60 updates/sec — where shaving 30–60 ms per packet matters and server relay cost is real.

**Uydurum's actual traffic profile:**

| Phase | Messages per player | Latency sensitivity |
|---|---|---|
| Root draft (10 s) | 1–3 taps | Only *relative* fairness matters |
| Suffix draft (10 s) | 1–5 picks/drops | Same |
| Showdown | 1 submission + maybe 1 flag | Same |

That's roughly **10–20 small JSON messages per player per round**. At this rate, the difference between a 25 ms peer hop and an 80 ms server round-trip is imperceptible to humans.

**The fairness inversion:** here's the counterintuitive part — for "fastest tap wins" mechanics, raw speed isn't what you need; you need *equal and arbitrated* speed. In a full mesh there is no single referee:

- Each peer sees tap events in a different order (5 peers = 5 different orderings of the same draft).
- Peer A with fiber beats Peer B on 4G *every single time* — and there's no neutral clock to correct for it.
- Resolving "who tapped first" requires the peers to run a consensus/ordering protocol among themselves, which adds round trips — often ending up *slower* than one authoritative server decision.

So for Uydurum's core mechanic, P2P is not faster where it counts. A single authority answering "player 3 got it" in one round trip is both faster to resolve and provably fair.

## 2. Testing the "stable" assumption

This is the assumption that fails hardest on mobile:

- **NAT traversal:** industry experience with WebRTC consistently shows a meaningful share of peer pairs (commonly cited around 8–20%, worse on carrier-grade NAT, which dominates mobile networks in many countries, including Türkiye) cannot connect directly and need a TURN relay. A 6-player mesh needs 15 pairwise connections — the probability that *at least one pair* needs TURN or fails outright grows fast. One bad pair degrades the whole lobby.
- **TURN is a server:** when fallback kicks in, you're paying for relay bandwidth — the "serverless cost savings" quietly evaporate, and you now operate signaling + TURN + auth/score DB. That's more infrastructure than the alternative.
- **Mobile network transitions:** Wi-Fi ↔ cellular handoff, elevator dead zones, OS backgrounding — each kills ICE connections. Mesh recovery means re-negotiating up to 5 peer connections mid-round; a client-server socket reconnects with one handshake and a state snapshot.
- **Host/state recovery:** with no server holding match state, a disconnect during the showdown raises "who has the true state?" A mesh needs state-reconciliation logic that is genuinely hard to get right (this is the class of bugs that ships broken in many indie multiplayer games).

**Verdict:** for 3–6 phones on heterogeneous mobile networks, a client-server WebSocket is empirically *more* stable, not less.

## 3. The trust problem (unique to this game)

Uydurum is a game **about lying**. The one thing the architecture must guarantee is that word validity and score transfers can't be cheated — and pure P2P structurally cannot guarantee it:

- The blueprint's SHA-256 cross-peer scheme doesn't hold: a modified client computes a "correct" hash for a false `ValidityStatus`; the `LobbySalt` must be known to all peers (i.e., to the attackers); and "2 peers must agree" is defeated by two friends in the same lobby. There is no honest-majority guarantee in a 3-player game.
- Any real fix (threshold signatures, commit-reveal rounds, external notary) is dramatically more complex than just having a server say "this word is valid, transfer the points."

## 4. Where you're *not* wrong

To be fair to the P2P instinct:

- P2P **does** eliminate per-message server compute at scale — if Uydurum hit hundreds of thousands of concurrent lobbies, offloading traffic would matter. (At that point you'd have revenue for servers anyway.)
- WebRTC data channels **are** excellent technology; the skills transfer to voice chat, which would be a great addition to a bluffing game.
- For a **LAN/offline party mode** (same room, same Wi-Fi), P2P is genuinely the right tool — no internet needed, zero server cost.

## 5. Recommended options, ranked

### Option A — Server-authoritative WebSocket (recommended)
Your diagrammed signaling server, promoted to full referee: owns lobby state, phase timers, draft arbitration, word validation, scoring.
- **Pros:** solves fairness, trust, timers, reconnects, and disconnect handling in one move; ~70% less networking code; a $5–10 VPS handles thousands of 6-player lobbies (turn-based JSON is a trivial workload).
- **Cons:** server latency in the loop (irrelevant at this message rate); you run a stateful service (you already planned to run signaling + auth + scores).

### Option B — Hybrid: P2P star + thin server referee
If you want to keep P2P: use a **star topology** (host relays, not full mesh — 5 connections instead of 15), but route *only* draft claims, word validation, and score transfers through the server; cosmetic/chatty events (emotes, typing indicators, bluff animations) go peer-to-peer.
- **Pros:** keeps P2P where it's harmless, puts authority where it's required.
- **Cons:** two transport layers to build and debug; host migration still needed.

### Option C — Pure P2P mesh (as blueprinted) — not recommended
Only defensible for a LAN-only party mode with friends who trust each other (in which case, drop the crypto verification entirely — it doesn't work and friends don't need it).

### Pragmatic path
Build **Option A** for v1 — it's the fastest route to a playable, fair, cheat-resistant game. Keep the network layer behind an abstract `GameTransport` interface (your Clean Architecture tree already supports this in `core/network/`) so a P2P LAN mode can be added later as a second transport without touching game logic. That preserves the P2P option instead of betting the project on it.

---

## Bottom line

"Fast and stable" is the right goal, but for a 10-messages-per-round mobile game, P2P mesh delivers neither: fairness needs an arbiter P2P doesn't have, and mobile NATs make mesh the *less* stable option while still requiring servers (signaling + TURN). Since the discussion is still open — prototype Option A first; it is strictly less work, and nothing about it forecloses adding P2P later.
