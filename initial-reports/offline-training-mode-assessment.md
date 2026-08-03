# Offline Training Mode — Honest Assessment (and the FFI Question)

**Feature under review:** single-player, offline word-construction drills ("Antrenman") — no bluffing, no uydurumcu detection, time-constrained, tiny XP rewards. Spec now in [BLUEPRINT.md](../BLUEPRINT.md) § Game Modes & Live Ops.

---

## 1. The mode itself: good idea, well-scoped

- **It trains the exact skill the core game tests** (fast root+suffix composition under time pressure), so it doubles as onboarding — new players can learn vowel harmony mechanics without being fed to bluffers.
- **Dropping bluff detection offline is the right call**, not a compromise: bluffing only works against humans; a solo "detect the fake" mode would need an AI opponent that is either trivially exploitable or expensive to build.
- **"Tiny points" is correct design, not stinginess:** offline progress runs on the player's device and is forgeable by definition. Keeping rewards small, daily-capped, server-validated on sync, and excluded from competitive leaderboards means cheating it is possible but pointless. Don't be tempted to raise these rewards later — that's the whole security model.

## 2. The FFI question — honest answer: you almost certainly don't need it

Your instinct ("offline needs a local dictionary, so we need FFI") is half right: the mode does require an on-device dictionary + morphology engine. But the jump to FFI doesn't follow. The numbers:

| Requirement | What the mode needs | What pure Dart delivers |
|---|---|---|
| Lookup rate | ~10–30 validations/min (human drill speed) | HashSet/trie lookup in µs — ~6 orders of magnitude headroom |
| Dictionary memory | 100k words | DAWG/trie ≈ 1–5 MB; even a naive `Set<String>` ≈ 15–25 MB — fine on low-end devices |
| Cold load | Before first drill starts | Compact binary format parsed in an isolate ≈ 100–300 ms — hidden behind one screen transition |
| Morphing | Vowel harmony token expansion | ~50–100 lines of Dart; **already planned** as the client `linguistics/` preview module |

There is no performance requirement in this mode that pure Dart fails. FFI would optimize a bottleneck that does not exist.

**The argument *against* FFI is stronger than "unnecessary" — it's architectural:**

- Your authoritative engine is **Go** (server). Your preview engine is **Dart** (client). A Rust/C++ FFI engine would be a **third implementation** of the same rules — three codebases to keep in golden-file lockstep instead of two. The offline mode should *extend the existing Dart `linguistics/` module into a full `WordEngine`*, which means the offline engine and the online preview are literally the same code, already parity-tested against Go in Phase 2.
- FFI reintroduces exactly the complexity class this project deliberately deleted: NDK/Xcode toolchains, per-ABI builds, CI matrix expansion, memory safety at the boundary, cross-language debugging. That cost was rejected when it promised "sub-millisecond lookups" for the online game; the offline mode's performance bar is far lower.

**When FFI *would* become justified (the honest escape hatch):**

1. Profiling on a real low-end target device shows dictionary load > 1–2 s or memory pressure causing kills — *measured, not assumed*.
2. You later adopt a full morphological analyzer (e.g., Zemberek-class analysis rather than token morphing) whose only good implementation is JVM/Rust — then binding beats porting.

Because the blueprint specs the engine behind a `WordEngine` interface, adopting FFI later is a swap, not a rewrite. Decide with profiler data, not upfront.

**Verdict: pure Dart. FFI is a documented contingency with clear trigger conditions.**

## 3. The real risks of this feature (none of them are FFI)

| Risk | Severity | Mitigation |
|---|---|---|
| **Licensing (highest):** shipping the dictionary in the APK is *redistribution*. Server-side use of a restrictively licensed list is grey; embedding it in a public binary is not. Gap 7 now constrains the client dataset too. | Blocking | Resolve dictionary licensing (Zemberek/Apache-2.0, hunspell-tr) **before** building this mode; ship only the license-clean subset on-device. |
| **Extraction:** on-device dictionary can be pulled from the APK. | Low — accept it | It's a word list, not a secret. Online anti-cheat is server-side and unaffected. Do not spend effort on client-side encryption (see original blueprint review — it was theater then, it's theater now). |
| **Parity drift:** offline Dart verdicts disagreeing with server Go verdicts teaches players wrong instincts. | Medium | The Phase 2 golden-file suite must run against *both* engines in CI; ship the same dictionary version tag on both sides. |
| **Forged XP sync** | Low by design | Tiny rewards + server-side daily caps + no leaderboard impact (already specced). |
| **APK size** | Trivial | +2–5 MB compressed. |

## 4. Bottom line

Build it — it's a low-risk, high-retention feature that reuses code you're already writing. But build it in pure Dart: the FFI instinct solves a performance problem this mode doesn't have, at the cost of a third rule-engine implementation and the exact toolchain complexity this project already chose to avoid. The one genuinely blocking dependency is dictionary licensing, which you must settle anyway (Gap 7) — settle it once, for server and client together.
