
# 📖 Uydurum — Project Architecture & Implementation Roadmap

## 🎲 The Game in Plain Words

The complete player-facing rules — the plain-language mirror of the Game Rules & Mechanics blueprint. The technical spec below is normative; if the two ever disagree, the blueprint wins.

### The basics

- **3 to 6 players** per game. A game is **3 matches** by default (the host can set up to 6).
- Every match you build one Turkish word from a **root** plus **suffixes** — then everyone tries to spot whose word is real and whose is invented.
- Everyone starts with a stack of **100 chips**; the biggest stack at the end wins. The game is named after the in-between thing you make: an **uydurum** — a word that isn't in the dictionary but sounds like it could be.

### How a game flows

A game is a series of matches, and **every match runs the same three phases below** — only the last match differs: its roots are dealt out instead of picked.

**1. Pick a root (15 seconds)**
- A pool of real dictionary roots is on screen. Everyone secretly ranks their 3 favorites — nobody sees anyone else's choices.
- When time's up, picks are revealed. If two players wanted the same root, it goes to **whoever has the fewest chips** (a built-in comeback mechanic); if still tied, a rotating priority marker decides.
- From then on, **everyone can see who got which root** — roots stay public all game.
- Picked roots are gone for good. The last match skips picking — leftover roots are just dealt out.

**2. Collect affixes (each match: 15 s block window, then one board at a time — 15 s each)**
- Every match, every player gets their **own private set** of **player count + 10 affixes**, dealt fresh. Sets are generated to be about **90% different** from one another, so the same affix appears in two sets only occasionally.
- **Everyone blocks at the same time — once per match:** when the session opens, each player privately studies their set for the same **15 seconds** and gets the match's one block action: select **1, 2, or 3 affixes** and confirm. Confirmation is final — no blocks are added, removed, or changed for the rest of the match. Nobody ever sits idle watching someone else think: it's one shared window, everyone deciding at once.
- Then the boards go up **one at a time**, in priority order — each alone on screen for **15 seconds** while everyone, its owner included, makes their move. When a board's time is up, the next comes up, until every player's board has had its turn. Blocked affixes show grayed out the whole time — on display, never takeable.
- While a board is up, everyone secretly does one thing: take an affix from it or from the public discard row, drop one of their own, or pass. Two players can take the **same** affix — nobody sees who took what.
- At the turn's close, dropped affixes go into the public row beneath the board, where everyone can see them and take them from the **next** board onward. Taken affixes join the **picked board** — a no-names list of everything taken this game, so you can guess what people might be holding, but never who holds what.
- Every dealt affix gets one chance while its board is up: if nobody takes it, it leaves circulation. Blocked affixes never get even that chance — they leave with their board, gone for good. A block is pure denial: roots are public, so you can tell which pieces your rivals are hoping for, and blocking keeps those pieces from ever reaching them — at the price of never taking them yourself.
- You'll hold **at least 3 affixes** when word-building starts; if you're short, the game tops you up automatically over the session's last three boards. Hands don't carry over: after each match settles, used pieces are consumed and leftovers are discarded — the next match deals everyone fresh.
- Some "suffixes" aren't official ones — they're pieces cut from real words (like *-kaha* out of *kahkaha*). These are bluff fuel.
- A few pieces are **prefixes** — rare but real in Turkish (*na-*, *gayri-*): they snap to the **front** of your root (*mağlup* → *namağlup*, *meşru* → *gayrimeşru*) and are drafted, blocked, and held exactly like suffixes.

**3. Build, reveal, accuse (15 s + 15 s)**
- Build one word from **your root + your pieces** and lock it in — suffixes chain after the root, a prefix (if you hold one) snaps to the front. You can't type freely — only combine what you drafted.
- You always know which you're submitting: if the word you've built isn't in the game's dictionary, a **private warning** tells you it will count as an uydurum before you lock it — with a tappable info note explaining that being grammatically correct doesn't automatically put a word in the dictionary. Only you see the warning; the table learns nothing.
- Convinced your word is real? A **one-tap report** sends it to the dictionary curators. The match still scores by the current dictionary, but accepted words join a future update.
- All words are revealed at once. Then everyone gets 15 seconds to secretly **flag one word** they think is invented. You can't flag your own — and you can only flag if you have at least **20 chips** to back the accusation.

### Keeping the pace

- A match tops out around **2½ minutes** at 6 players; the last one (roots dealt, not picked) runs a touch shorter — a default 3-match game stays around **7 minutes** of play. Those are ceilings, not the norm:
- **Ready:** confirming your blocks is your ready in the block window — the moment everyone has confirmed, the boards begin. During picking and building, tap **Ready** when you're done; the moment everyone is ready, the wait skips and the next phase starts.
- **Poke:** someone dragging their feet? **Poke** them — their screen buzzes lightly. One poke per player per wait.
- Root picking and flagging always run their full 15 seconds — those stay blind to the very end.

### Scoring (chips)

- Everyone starts the game with **100 chips** in a single stack. Words **mint** new chips, gambles **move** chips between players, and your stack never drops below 0 — at 0 you're **broke**.
- **Real word:** +1 per letter, root included (*gözlükçü* = 9 chips).
- **Clean sweep:** +15 for using every piece in your hand — prefixes included — in your word. There's no penalty for leftovers — the bonus is the whole incentive, and chaining your entire hand validly is genuinely hard.
- **Longest valid word:** +30, shared equally if tied (two-way: 15 each — the only place half-chips appear).
- **Invented word (uydurum):** no word chips — its payoff is the bluff:
  - **Nobody flags you:** +60, paid equally by your opponents (20 each at 4 players).
  - **Someone flags you:** you pay the stake on that bluff, split among everyone who caught you. Your first bluff of the game is worth **20**, your second **40**, and every later bluff **60**.
  - **You flag a real word by mistake:** you hand 20 chips to the player you accused.
- **Bluffing and flagging need skin in the game:** to submit an invented word, you must hold at least the stake for your next bluff — **20**, then **40**, then **60** for the rest of the game. This is only an eligibility threshold: the chips are not reserved or spent when you bluff, and you pay them only if another player catches you. Flagging still requires at least **20 chips** to cover a wrong accusation. Your bluff count advances whenever you submit an uydurum, whether or not it is caught.
- **Broke players (0 chips)** can't do either, obviously — but the gates kick in well before that, so reckless players near the bottom get reined in early.
- Everything settles at once at match end: word chips first, then all flags and pots from one snapshot — you pay what you have, never below 0.
- The scoreboard shows **word chips** and **gamble chips** separately, so you can always see who's building and who's gambling.

### The strategy triangle

Play it safe with a real word, gamble on a bluff for the pot, or hunt bluffers with your one flag — each option punishes the others. The quiet fourth skill is the clean sweep: draft each match's hand so every piece you take chains into one word.

### If someone disconnects

Their seat stays and the game plays it neutrally (they get dealt roots and suffixes, submit nothing). They can rejoin anytime and continue. The game keeps going as long as **at least 2 players** are connected; below that, the current match finishes and the game ends as it stands. Everyone's total is still recorded at the end — but **quitting, or still being gone when the game ends, costs you half your final score**.

### When you open the app

Every day starts with a **Word of the Day**: a real word, its meaning, and an example sentence — in whatever language your app is set to. A small dose of the dictionary before you go off inventing your own.

---

## 🧱 Tech Stack

| Layer | Technology | Role |
|---|---|---|
| Client | Flutter (pure Dart, no FFI) | UI, WebSocket client, Dart `WordEngine` (online morphing preview + offline Training Mode) |
| Backend | Go | Authoritative game server: lobbies, timers, drafts, word validation, scoring |
| Database | PostgreSQL | Durable data: profiles, game results, cosmetics, leaderboards |
| Cache / Pub-Sub | Redis | Lobby→node routing, session presence, cross-node pub-sub |
| Transport | WebSocket (JSON messages) | Single realtime channel between client and server |
| Infrastructure | Docker Compose (now) → Kubernetes + Terraform (later) | Everything containerized from day one |

Architecture decision record: server-authoritative model chosen over P2P mesh — a trusted authority wins on latency fairness, anti-cheat, and operating cost.

## 🕹️ Game Rules & Mechanics Blueprint

### 1. Game Parameters

* Player Count: Minimum 3, maximum 6 players per lobby to start; a started game continues as long as **at least 2 players remain connected** (see §6).
* Match Clock: every match runs draft 15 s (skipped in the final match, whose roots are dealt) + the affix session (a 15 s simultaneous block window + `players × 15 s` board turns) + construction 15 s + flag window 15 s. Worst case at 6 players: **2 min 30 s** per match (**2 min 15 s** for the final one) — about **7 min 15 s** of match time in a default 3-match game. Block-confirmation unanimity and Ready unanimity (§7) can only shorten a match, never extend it.
* Match Count: a game runs **3 matches by default**; the host can raise this at lobby creation up to a **maximum of 6**. The count is fixed once the game starts — dropouts never shrink it (see §6, Disconnects & Dropouts).
* Victory Conditions:
  * Longest-word bonus: each match's longest valid word earns +30 chips, shared equally on ties (§5) — there is no other per-match title.
  * Game Winner: the player with the biggest chip stack at game end (everyone seeds at 100 — §5).

### 2. Phase 1: The Root Draft (15 Seconds, Blind Pick)

* Pool size: the game begins with exactly `players × matches` dictionary-verified roots (e.g., 5 players over 3 matches = 15 roots). Claimed roots are removed and are **not** replaced: each match consumes exactly `players` roots, so the pool empties precisely at game end — never a root short, never one over. Dropout seats are system-played rather than removed (see §6), so consumption never varies.
* Final match: the draft is skipped — the last `players` remaining roots are dealt randomly, one per player (server RNG, seed logged for auditability). This is a deal, not a contest: randomness never resolves a contested pick.
* Every root is verified against the dictionary to ensure real words can branch from it. The draft screen groups the pool by difficulty tier so the first match's larger pool stays readable in the window.
* Draft priority (the "button"): each match every player holds a unique priority rank (1..N). The button marks priority 1 and rotates by one seat every match, so no player camps the advantage. Priority numbers are displayed in the draft UI before the window opens, making every resolution verifiable at a glance.
* Interaction — blind simultaneous pick: during the 15-second window each player secretly ranks their **top-3** roots. Nobody sees others' choices while drafting; all picks are revealed simultaneously when the timer ends. Once resolved, **every player's claimed root stays publicly visible for the rest of the game** — affix takes, drops, and current holdings are the hidden information (§3).
* Conflict resolution — wave-based, fully deterministic (one rule at every depth: *contested root goes to the player furthest behind; exact ties go to the better draft priority*):
  * Wave 1 (first picks): an uncontested first pick is claimed outright. If 2+ players ranked the same root first, the player with the **smallest chip stack** wins it (built-in catch-up mechanic); if stacks tie, the better draft priority wins.
  * Wave 2 (second picks): losers fall back to their second pick. Collisions resolve by the same rule; a second pick already claimed in wave 1 falls through to wave 3.
  * Wave 3 (third picks): same rule, recursively.
  * Exhausted all ranks: remaining players receive unclaimed roots in draft-priority order, by pool display order — no randomness anywhere in resolution.
  * Stacks are frozen at draft start: winning an earlier wave never changes a player's standing within the same draft.
* No selection made: the server assigns the player an unclaimed root after all ranked players are resolved, in draft-priority order by pool display order — the only path that bypasses ranking, and it is self-inflicted. This includes disconnected players: they rank nothing but are still dealt a root at window close, so a mid-match reconnect rejoins a playable hand.
* Rationale: blind picks make the draft latency-immune — no "fastest tap" network races — and deterministic resolution means chance may shape the pool, but never decides a contest between two players (the final-match deal assigns leftovers randomly, but contests nothing).

### 3. Phase 2: The Affix Session (Every Match: 15 s Block Window + 15 s per Board)

* The session runs **once per match**, between that match's root draft and its construction window — every match in the game carries one. It opens with the **simultaneous block window** — every player, at the same moment, privately preparing their own fresh set — then displays the boards **one at a time: one 15-second turn per player** (4 players → 4 turns; 6 → 6), starting from the seat holding the match's draft priority 1 (the button, §2) and proceeding in draft-priority order until every board has been shown. Exactly one board is ever on display; during its 15 seconds everyone — the board's owner included — makes their one hidden choice, and when its time is up the next board comes up. No seat ever waits on another player's private thinking: the only private window is one everyone plays at once.
* Private sets — one deal per match: before each session, every seat receives its own hidden set of **players + 10** affix instances (4 players → 14; 16 at the 6-player maximum). Fresh sets are sampled to be at least **90% different by affix label** from every other player's set. At supported table sizes, the pairwise overlap cap rounds down to one matching label; intentional matches are separate dealt instances.
* Affixes, not only suffixes: sets draw from the bundle's whole **affix inventory** — overwhelmingly suffixes, plus the occasional **prefix token** (Turkish has a handful of lexicalized ones: *na-*, *gayri-*). Prefixes are blocked, picked, dropped, topped up, and held under exactly the same rules; the only difference is where they attach (§4). They appear at natural inventory frequency — no quota.
* Block window (15 seconds, once per match, all players simultaneously): as the match's affix session opens, every player sees their own set and may send **one block action** containing **1, 2, or 3 affix instance IDs** — that match's single block decision. Once the server accepts it, the selection is final and repeat or replacement actions are rejected; confirming doubles as that seat's Ready (§7), and unanimity starts the first board early. A seat whose window expires without an action reveals with no blocks — the neutral behavior used for an absent owner. No player can see, or block, another player's set, and after this window closes no block is added or lifted for the rest of the match — the next block chance is the next match's window. Public roots (§2) are what make the block choice informed: an owner can see exactly which of their pieces a rival's root is hoping for.
* Board window (15 seconds, everyone — owner included): the turn's board is displayed to all; blocked affixes remain visible but are non-selectable. The public discard row is visible beneath it. Every player may act once, in secret: take one selectable affix from either source, drop one affix from their own hand, or pass. Takes are hidden and **non-exclusive**, so several players may receive the same selected affix without a latency contest.
* Turn close and discards: after blind takes resolve, a source instance selected by anyone leaves its board or the discard row and joins the picked board; every player who selected it receives that affix. Drops then enter the public row without owner names and become selectable from the **next** board onward, never midway through the window in which they were dropped. An unpicked discard stays public across boards and matches until taken or until the game ends.
* One chance each, blocks deny for good: every board instance gets exactly one public chance — its board's display turn. An unblocked instance that nobody takes expires at that turn's close. A blocked instance never enters circulation at all: it stays grayed out on the displayed board as the public record of its owner's denial play, then leaves the game at the same close. Nothing carries to a later board, nothing returns, and nothing is ever blocked twice — once the block window closes, the session is pure information and picks. A repeated affix label in another player's fresh set is an independent instance, not the same token. Player hands clear after every match's settlement: affixes used in the submitted word are consumed, unused ones are discarded — each match's session builds its hand from scratch.
* Public information: every player's drafted root is visible all game (§2); the board on display, its owner's blocks, the no-names discard row, and the no-names picked board are public. **Who took or dropped each affix and who currently holds it remain hidden.** The picked board lists each affix label once without ownership or duplicate counts.
* The minimum: every hand must hold **at least 3 affixes** when the match's construction window opens. Over the session's last three board turns, the server tops up anyone behind pace — 1 by the third-last, 2 by the second-last, 3 by the last. A top-up uses a selectable instance from the board on display and resolves like a hidden non-exclusive take, so the block rule is never bypassed and a legal source is guaranteed by the board size. A player who never acts is dealt exactly one per turn across those last three.
* No cap: one take per board turn means a hand can never exceed `players` affixes. Every affix that cannot fit the submitted word forfeits the clean-sweep bonus (§5), so hoarding still has an opportunity cost.
* Rationale: mostly distinct private sets prevent one globally bad deal from constraining everyone. Every owner gets one denial decision per match — made simultaneously, so nobody idles through another player's deliberation — while each reveal gives the whole table the same hidden, non-exclusive action. Blocking never reserves: takes are non-exclusive, so a piece an owner wants needs no protection — a block is worth spending only on pieces that serve a rival's public root better than the owner's own, and it costs the owner that piece too. Session length remains deterministic: `15 s + players × 15 s` per match (**1 min 45 s** at six players); block unanimity and Ready can shorten it.

### 4. Phase 3: The Showdown & Bluffing Mechanic

* Structure — three server-owned steps: a **15-second construction & submission window** (compose from your root + board affixes and lock in one final word; no submission = no word chips), a **simultaneous reveal** of all words, then a **15-second blind flag window**. Flags stay hidden until the window closes — no bandwagoning, no fastest-tap race — and all resolve together at close. Each player may flag **at most one** word per match, **never their own**, and **only with at least 20 chips**: self-flags and underfunded flags are rejected server-side, so a bluffer cannot hedge their own submission and a nearly-broke player cannot recklessly police the table.
* Players submit real dictionary words or invented ones and bluff them through. Terminology: a word outside the dictionary bundle is an **uydurum** — not a lie, not a dictionary word, the in-between state the game is named for. In-game "valid" always means *attested in the active dict-pack bundle*, and the UI presents it exactly that way.
* Progressive bluff stake: each player's accepted uydurums are counted across the game. Their **first** carries a 20-chip stake, their **second** 40, and every later one 60: `next_stake = min(20 × (accepted_uydurums + 1), 60)`. The count advances whenever an uydurum is accepted, whether it survives or is caught; real words and rejected submissions do not advance it.
* Bluff & flag eligibility — skin in the game, not an ante: the construction-window-open stack snapshot sets both gates for the match. A player must hold at least their **next bluff stake** to submit an uydurum and at least **20 chips** to flag. Meeting the bluff gate does not reserve, escrow, or deduct chips: an unchallenged bluffer pays none of that stake, and only a caught bluffer owes it. The client rejects an underfunded uydurum and hides the flag control from an underfunded detective; a forged intent is scored as no submission or a dropped flag. Public stacks plus resolved bluff counts make each player's current gates knowable.
* Composition constraint (plausibility pressure): every submission — real or bluffed — must be built from the player's own drafted root and board affixes: suffixes chain after the root, a held prefix may attach in front (stacking limits are per-language; Turkish allows at most one prefix), with the language module's attachment rules applied by the engine. The bluff is that the *combination* is not a dictionary word; free-typed gibberish is impossible by construction.
* Submission clarity — the uydurum warning: the composer's live preview marks the assembled word as dictionary-valid or not at every composition change, and locking a non-attested word raises a private confirmation: *this will be submitted as an uydurum*. An info icon on that sheet opens a one-paragraph explainer of the game's central nuance — a grammatically well-formed Turkish word is not necessarily an attested dictionary entry, and in-game *valid* means only *attested in the active bundle* (Word Engine §3). The same sheet is where an underfunded uydurum is rejected against the player's next 20/40/60 stake, and it carries the one-tap **validity dispute** ("this is a real word", Word Engine §3) for players who believe the dictionary is missing their word — logged for curation review, zero effect on the live verdict. The warning renders only on the submitter's device; nothing about it is observable at the table.
* Word chips go to dictionary-valid words only: an uydurum mints no letter chips and no bonuses, and only valid words compete for the longest-word bonus. A bluff's only upside is its survival reward. There is no leftover-affix penalty — the clean-sweep bonus (§5) is the anti-hoarding incentive, so an unused affix costs exactly one thing: the sweep.
* The Deception Loop:
  * Unchallenged bluff: the bluffer collects 60 chips, paid equally by the opponents — `60 / (players − 1)` each: 30/20/15/12 at 3/4/5/6 players, always whole numbers.
  * Caught bluff: the bluffer pays only that bluff's **20/40/60 stake**, split among all correct flaggers. Each receives `floor(stake / catchers)`; any remaining whole chips go one each to catchers in the match's rotating draft-priority order. The stake is a total liability, not an amount paid to every catcher.
  * False flag: a player who flags a genuine dictionary word pays a flat **20-chip fee** to the word's owner. The fee binds at every stack size — no score-scaled percentage — and it keeps the bait play (a real word that smells fake) deliberately profitable.
* Settlement — one snapshot, no verdict dependence (also the audit-log event order): 1) word chips and bonuses are minted → 2) all bluff transfers are computed from that post-mint snapshot → 3) false-flag fees are computed → 4) all transfers apply together. A payer never falls below 0. If their bluff debts exceed their available stack, the stack is distributed proportionally across those debts, with whole-chip remainders assigned by recipient draft priority; false-flag fees use only what remains. No incoming transfer funds an outgoing one in the same settlement, and no transfer changes another verdict, gate, or stake.

### 5. Scoring: The Chip Economy

One number per player: a **stack**, seeded at **100 chips** at game start and floored at **0** — nobody goes negative. Bluffing requires enough to cover the player's next **20/40/60** stake and flagging requires **20+ chips** at construction-window open; a player below the relevant gate is locked out of that action for the match (§4). Words **mint** new chips from the bank; gambles **move** chips between players; the biggest stack at game end wins. The scoreboard shows two lines per player — **word chips** and **gamble chips** — so building and gambling read as two visibly different games.

Scoring table (v4 values — tuned via the protocol below):

| Event | Chips |
|---|---|
| Valid word | +1 per letter, root included (*gözlükçü* = 9) |
| Clean sweep — every affix in hand (prefixes included) used in the word | +15 (replaces any leftover penalty; hard to earn — the whole board must chain validly) |
| Longest valid word of the match | +30, shared equally on ties (2-way: 15 each; 3-way: 10; 4-way: 7.5 — the only halves in the game) |
| Uydurum (non-dictionary word) | no word chips — its upside is the survival reward (§4) |
| Unchallenged bluff | +60, paid equally by all opponents (30/20/15/12 each at 3/4/5/6 players) |
| Caught bluff | −20 on your first bluff, −40 on your second, −60 thereafter; split among all correct flaggers |
| False flag (max one flag per player per match) | −20, paid to the accused |
| No word submitted | no word chips, no bonuses |
| Uydurum attempted below its next 20/40/60 stake | rejected server-side (§4): scored as no submission; bluff count unchanged |
| Flag attempted below 20 chips | rejected server-side (§4): the flag is dropped |

No tie-breaking cascade: the longest-word bonus is shared on ties, so nothing downstream needs an order — the old "fewer affixes" and "earliest submission" rules are gone (no server-arrival races anywhere). A match with no valid word simply mints no word chips and awards no longest-word bonus.

Playtest tuning protocol — `dictpack simulate` acceptance bands, one lever per failure mode:

| Symptom | Healthy band | The one lever |
|---|---|---|
| Bluff rate runs hot | 20–35% of eligible matches | False-flag fee 20 → 15 (cheaper policing squeezes survival rates) |
| Flag spam | ~40–60% flag participation | Fee 20 → 30 |
| Bluffing dies (<~15%) | — | Survived uydurum also scores its letters (+1/letter) — last resort, adds a rule line |

Bluff submission, catch, and survival rates must also be reported separately for the **first**, **second**, and **later** bluff tiers; one aggregate rate would hide first-bluff dominance created by the progressive stake.

The seed (100), letter value (+1), stake ladder (20/40/60), survival reward (60), false-flag fee (20), sweep (+15), and longest-word bonus (+30) are the tunable constants — each one a key under `scoring:` in `configs/gameplay/tuning.yaml` (Word Engine, Designer Workbench); the structure is fixed.

### 6. Disconnects & Dropouts

* Seats persist: a dropped-out player's seat is never removed — the system auto-plays it exactly like a fully passive player (no ranks → a root is still dealt; no block action in a match's block window → its board displays blockless on its turn; every board window is a pass; last-three-turn top-ups still fill the hand to three; no submission → no word chips minted). Root-pool consumption and the match count are both fixed at game start, so departures never shrink either — the `players × matches` pool arithmetic holds regardless of who is present.
* Grace window: a disconnected player has 20 seconds to reconnect before counting as dropped for the below-minimum check; the seat is auto-played from the moment of disconnect until its owner returns, however long that takes.
* Rejoin: a reconnecting client receives a full server snapshot of the current match state (session-token based, per the network architecture) and picks its seat back up mid-phase — scores from auto-played matches stand.
* Below minimum: the game continues as long as **at least 2 players are connected** (auto-play covers the rest). If connected players drop below 2, the current match is finished — auto-played to completion — and the game ends, scored as-is.
* Finalization & the 50% penalty: at game end every seat's final stack is recorded — but a player who **quit mid-game**, or who is **still disconnected at finalization**, is recorded at **50% of their final stack**. Rejoining and finishing the game connected avoids the penalty entirely; the auto-played stretch (no words, no chips minted) is already its own cost, so walking away is never score-neutral.
* No bot takeover: absent players are never replaced by AI stand-ins — bots are farmable in a bluffing game. Development bots exist (§ Development & Test Bots) but connect only in dev/staging environments and never hold a production seat.

### 7. Pace Controls: Ready & Poke

* Ready: in each match's block window, confirming a final block selection is that seat's Ready — unanimity starts the first board early. In every shared board window and the construction window, a locked action or pass enables Ready, and unanimity closes the window early. **Auto-played seats submit no blocks, pass, and count as always ready**, so a game running at the 2-connected minimum (§6) still fast-forwards. The root draft and flag window always run their full 15 seconds.
* Poke: once per wait window, any player may poke one player who has not yet confirmed blocks or readied in the current window. The target's device gives a **light haptic buzz** and a brief screen shake — pressure, not punishment: pokes are anonymous, have no score effect, and the once-per-window cap is enforced server-side.
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

### 3. Word of the Day ("Günün Kelimesi")

Shown on every app start: one curated word with its **meaning** and an **example sentence**, localized to the language the app is set to.

* Content: a hand-curated editorial feed per language module — `{word, meaning, example}` rows in PostgreSQL, published on a daily schedule by the same scheduled-job machinery as the Weekly Pool. Meanings are written or licensed for this feed, never scraped: dict-pack bundles attest word *forms* only, and the TDK rule (Word Engine §3) applies to definitions too.
* Delivery: fetched once at app start (plain HTTPS, no realtime channel), cached locally. Fallbacks: the last cached entry when offline; a small bundled starter set for a first-ever offline launch. One entry per calendar day; the server's schedule is authoritative.
* Localization: the feed is part of the language-module contract (Word Engine §2) — switching the app language switches the feed.
* Touchpoint, not a gate: a dismissible home-screen card; tapping it seeds a Training Mode drill with the word's root when that root exists in the active bundle. It never blocks the path to Play.

### 4. Weekly Leaderboard

* Cycle: Monday to Sunday on the server clock, reset by the same scheduled-job machinery as the Weekly Pool; Monday announces last week's podium alongside the pool winner — one shared weekly rhythm.
* Score: the sum of a player's final game stacks for the week, recorded post-penalty (Game Rules §6). Only **Quick Play** games count — private rooms are collusion-farmable and stay off the board — and offline Training XP never counts (§1). A daily cap on counted games (value in `configs/gameplay/tuning.yaml`) blunts pure grind volume.
* Display: top 100 plus the viewer's own rank and score; ties share a rank. One board per language module — leaderboards, like every feed, ride the module contract (Word Engine §2).
* History: each weekly close snapshots the standings into an immutable history table; podium finishes surface on player profiles (Profiles & Community §1).
* Infrastructure: pure PostgreSQL aggregation over the Phase 4 game results — no realtime channel, no new stores.

---

## 👤 Profiles & Community

### 1. Public Player Statistics

* Every profile is open to every player — tap a name in the lobby, scoreboard, or leaderboard: games played and won, weekly podium finishes, bluff submissions / survivals / catches, flag accuracy, clean sweeps, longest word ever built, and word chips vs gamble chips lifetime totals.
* Deliberately public: the table already shows stacks and resolved bluff counts (Game Rules §4) — lifetime stats extend the same metagame. A profile that bluffs 40% of the time is a read, and playing against your own reputation is part of the game.
* Source of truth: derived nightly from the Phase 4 audit-event stream — the same events behind the KPI jobs (Product Baseline, Analytics). No client-reported numbers anywhere, so stats are exactly as trustworthy as the server.
* Privacy: profiles are pseudonymous — nickname + avatar, no PII (Compliance baseline) — and the in-app delete-my-data action erases stats with the account.

### 2. Avatars (Free Presets, Paid Uploads)

* Default: every account picks from a curated neo-brutalist preset gallery — free forever, and moderation-proof by construction.
* Custom upload: a one-time **Custom Avatar** unlock (game currency, Monetization §7) lets a player upload their own image. Server-side processing: crop and resize to 256×256 WebP, EXIF stripped, size-capped; stored as small blobs in PostgreSQL (no new object store at v1) and served over HTTPS with caching.
* Moderation gate: an automated image screen (server-side, provider-swappable) holds every upload until cleared, and uploads stay reportable forever (§3). An admin takedown reverts the account to presets and can revoke the upload privilege — the purchase buys the feature, not immunity.
* Compliance: avatars are user content — delete-my-data removes the stored image; the 13+ age gate applies here as everywhere.

### 3. Player Reports (Avatars, Conduct, Cheating)

* Naming: the UI says **Report** — "flag" stays reserved for bluff accusations (Game Rules §4).
* One tap from any profile or the in-game scoreboard: a category — inappropriate avatar, offensive nickname or harassment, cheating or collusion — plus an optional note. Reports log reporter, target, game id, and bundle version, and join to the audit log so an admin can replay exactly what the reporter saw.
* Honest scope: the server-authoritative design already makes technical cheating impossible — forged scores and late intents die at the server (Word Engine §4). Reports exist for the human kind: seat collusion, boosting, offensive identity. Win-trading in private rooms is pre-defused — they never count toward the leaderboard (Live Ops §4).
* Handling: no automated punishment at v1 (Product Baseline) — reports feed the authenticated admin queue (kick, ban account+device, close lobby, avatar takedown). Rate-limited per reporter; repeat reports on the same target collapse into one case.

### 4. Feedback & Ideas

* An in-app **Send feedback** form: category (bug, idea, other), free text, and an auto-attached context snapshot — app version, bundle tag, last game id — shown to the user before sending.
* Dictionary disagreements stay in their own lane: the one-tap validity dispute (Word Engine §3) is purpose-built for "this word is real"; the feedback form is for everything else.
* Infrastructure: a rate-limited endpoint into a PostgreSQL table with a status column (new / seen / done), listed in the admin endpoint set. No third-party helpdesk at v1.

---

## � How-to-Play Clip (Ship-Gated)

A ≤45-second, watch-don't-read onboarding clip: a first-timer should be able to follow their first game after one viewing.

* Format constraints: real UI capture only; silent-autoplay friendly — big brutalist captions carry the story, audio optional; one idea per beat, one beat per phase; every beat readable at phone size.
* Storyboard (7 beats, 4–6 s each):
  1. Hook — "Invent a word. Get away with it." A real word morphs letter by letter into an uydurum.
  2. Pick a root — the draft screen, three secret favorites, simultaneous reveal.
  3. Block — every player's private set appears at once, each visible only to its owner; everyone locks 1–3 affixes together.
  4. Reveal & pick — the boards then go up one at a time, blocks gray out, and hidden takes resolve beside the public discard row.
  5. Build — root + suffixes snap together; vowel harmony visibly morphs the seam.
  6. Reveal & flag — all words up at once; caption "one of these is invented"; a flag lands.
  7. Score — the strategy triangle in one line: safe word / bold bluff / sharp flag. Logo out.
* Ship gate: produced on the final production UI and released with prod — **explicitly deprioritized until then**; no clip work is scheduled while gameplay, engine, and netcode areas remain open.

---

## �🎨 Visual Identity: Soft Neo-Brutalism Design Matrix

Direction locked from reference art: **pastel neo-brutalism, illustration-light**. The brutalist skeleton stays — thick ink borders, hard zero-blur shadows, chunky type, flat fills — but it wears a soft candy palette, and the personality comes from tiles, type, and color, not mascots or scene art.

* Palette tokens (Flutter constants; light theme only at v1):
  * `canvas` `#DCC8F7` — lavender field, with a faint low-contrast grid tile; `surface` `#F7F2E9` warm cream for cards and sheets; `#FFFFFF` content wells inside them.
  * `ink` `#141414` — every border and every glyph; text is never gray-on-gray.
  * `violet` `#B49AF5` — the neutral interactive: buttons, selected tiles, timers, progress fills.
  * `lime` `#D4F04C` — the truth/reward signal: valid-word confirmations, minted chips, clean sweep — and the **highlighter motif** (below).
  * `pink` `#FF9ED2` — the risk/accusation signal: flag actions, caught bluffs, the uydurum warning sheet. The palette's single permitted gradient (`#FFD9EC → #FF9ED2`) is reserved for celebratory reveal headers.
* Structure: every container carries `Border.all(width: 3, color: ink)` and a hard shadow `BoxShadow(color: ink, offset: Offset(4, 4), blurRadius: 0)`. Corners are rounded — radius 16 for cards and sheets, 12 for buttons, full pill for stat chips — soft geometry, hard ink. Pressing a control collapses its shadow to zero offset while the control translates onto its own shadow footprint: the signature brutalist click.
* Typography: a chunky rounded display face for headings, timers, and chip numbers (Baloo 2 / Fredoka class — both OFL; lock one after a Turkish-diacritics render check), a plain geometric sans for body. **Highlighter emphasis is the house style:** the revealed word, a chip delta, the clip caption "one of these is invented" — key phrases sit on a lime marker sweep, not bold-only.
* Illustration policy — deliberately sparse: no mascot, no scene art anywhere in the match flow. One tiny single-weight doodle glyph set (sparkle, cloud, star; ~12 glyphs) is reserved for empty states, win moments, and the Word of the Day card. A board mid-match must read as pure tiles + type at a glance.
* Fixed color semantics: violet = interact, lime = real/reward, pink = accuse/risk, ink = information. No verdict ever leans on hue alone — valid/uydurum states always pair color with an icon and a label (colorblind-safe by construction).
* Performance guardrails unchanged: flat fills everywhere (the one gradient exception above), zero blur radii, no stacked translucency — the same low-end-device constraint that motivated brutalism in the first place.
* Asset strategy — code first, raster last: the entire design system above is geometry, so it ships as Flutter widgets and `CustomPainter`s (borders, hard shadows, grid tile, highlighter sweep, press animation) — no image assets in the UI chrome. The doodle glyph set ships as hand-authored SVG paths. True raster art — preset avatars, the app icon, store screenshots — is produced **offline in curated batches** (image-generation tools or a designer, consistency-passed against the palette), bundled at build time, and never generated at runtime.
* Raster production workflow — decided: when asset production begins, the project owner provides **nano banana (Gemini image) API** access, and the AI assistant (Claude Fable) drives generation — deriving prompts directly from this design matrix (palette hexes, ink stroke weight, rounded shape language, illustration policy) so every batch lands on-brand. Scope is exactly the three raster classes: **(1) preset avatar gallery** — consistent character portraits generated as candidate batches, curated for stroke and palette consistency, then bundled (needed by Phase 5's avatar system); **(2) app icon** — one high-stakes artifact, worth iterating in the image tool (needed at store launch); **(3) store screenshots & feature graphics** — marketing surfaces, not app code (store launch). The API key is a secret under the config discipline (Infrastructure §2 — never in YAML files or images), and the pipeline stays design-time only: generated candidates → human curation → consistency pass → committed to the repo like any other asset.

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
│       │   ├── screens/         # MainMenu, Lobby, Draft, Showdown, Scoreboard, Profile, Leaderboard
│       │   └── widgets/         # Brutalist UI (BrutalistTimer, AffixCard, PickedAffixBoard, PublicDiscardRow, BluffButton, ReadyButton, PokeNudge, WordOfDayCard)
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
│   ├── dictpack/                # Go CLI: vendor dictionaries → versioned dict-pack bundles; designer workbench & simulator
│   └── gamebot/                 # Go CLI: protocol-level test bots — fill dev/staging lobbies, drive integration & load tests
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

### 2. Linguistic Affix Morphing & Validation (Language-Module Framework)

* Affixes are stored as abstract tokens with an **attachment side**: suffixes (`-{lIk}`, `-{mAk}`) and prefixes (`{na}-`, `{gayri}-`). Turkish is suffix-dominant but genuinely has both — *mağlup* → *namağlup*, *meşru* → *gayrimeşru* — so position is a first-class token property from day one, not an English-only special case. Turkish prefixes are few, lexicalized, and harmony-invariant; prefix-heavy languages (English *un-*, *re-*) use the same token model with their own rule module.
* Inventory tokens need not be official morphemes: the pipeline also cuts attested words into **fragment tokens** (e.g., *kahkaha* minus the root *kah* yields the fragment *-kaha*), and fragments may be cut from either end of a word. Fragments attach and harmonize like any other token; they mostly build uydurums — deliberate bluff material — and the branching-factor precompute (§3) tells them apart from productive affixes. Fragments are what let the inventory scale to game-scoped exclusion.
* The Go engine applies the language module's attachment rules — vowel harmony, affix position, stacking limits (Turkish: at most one prefix per word) — to transform tokens into concrete strings (`kafa + -{lIk}` → `kafalık`, `{gayri}- + meşru` → `gayrimeşru`), then checks the result for validity and scores it.
* The client's `linguistics/` module implements the same rules in Dart as a full `WordEngine` (morphing + on-device dictionary). Online it powers **instant preview** — cosmetic, only the server verdict counts; offline it is the sole engine for the Training Mode.
* Language extensibility — the module contract: the `Morphology` interface in `server/internal/words` (mirrored in Dart) is language-abstract from day one, and a playable language is exactly four deliverables — **(1)** a dict-pack bundle in the standard format, **(2)** a `Morphology` implementation (attachment sides, harmony/agreement, stacking rules), **(3)** a CI oracle corpus of golden files proving the Go and Dart twins agree, **(4)** a `tuning.yaml` overlay plus a Word of the Day feed (Live Ops §3). English is second in line; the framework counts as proven the day two modules pass the same test suite unchanged. Lobbies select a language the way they select a premium bundle — per lobby, via the existing bundle mechanism (§3 version discipline).

### 3. Dictionary Data Pipeline (`tools/dictpack`)

Both dictionary sources stay **out of the runtime entirely** — a build-time pipeline compiles them into one versioned, game-ready bundle consumed by both engines:

* Sources: verbatim upstream files live under `data/vendor/` with their licenses — TDD `hunspell-tr` (`tr_TR.dic` + `tr_TR.aff`, MPL-2.0) and the Zemberek lexicon (Apache-2.0). Vendor files are never edited; curation lives in overlay files (`exclusions.txt`, `additions.dic`, `catalogues/` topic lists). MPL-2.0 obligations on the compiled bundle stay minimal by construction — `manifest.json` carries the license notices (surfaced in the client credits screen) and points at the verbatim MPL sources vendored under `data/vendor/`.
* Compilation: `tools/dictpack` (Go CLI, containerized, runs in CI) expands `.dic`+`.aff` into attested word forms; filters proper nouns, out-of-charset entries, and length outliers with Turkish-locale casing; intersects Zemberek roots with the attested set; and emits the **dict-pack bundle**: `words.dawg`, `roots.tsv` (POS, frequency tier, branching factor, topic tags), `affixes.json` (the constructive affix inventory + attach rules — attested suffixes, the language's prefixes, and generated fragments, every token tagged with its attachment side; sized to sample `matches × players × (players + 10)` fresh private-set slots without forced overlap — every match deals fresh sets, so a maxed 6-player, 6-match game needs **576** slots), `manifest.json` (version tag, checksums, license notices).
* Branching-factor precompute: for every root, the pipeline walks all suffix chains (bounded depth) against the DAWG and records how many attested words are reachable with the current inventory. The root-pool sampler guarantees every drafted root has ≥K real derivations — no dead-end roots — and difficulty tiers fall out of the same number. The same statistics certify which root-length ranges and topic catalogues a bundle can serve (§ Monetization 5–6). The complement (morphable-but-unattested forms) is each root's **bluff surface**, the raw material of the deception loop.
* Coverage program: in-game validity means *attested in the bundle*; anything outside it is an uydurum by definition, and the client states this openly (help/credits screens — it is the game's namesake, not fine print). To keep the bundle honest, the client ships a one-tap validity dispute (“this is a real word”), surfaced where the verdict stings — the pre-submission uydurum warning (Game Rules §4) and the match scoreboard: disputes are logged server-side, reviewed by curation, and accepted words enter `additions.dic` in the next bundle version. Individual disputed words may be fact-checked against official references (TDK GTS) — verifying that a word exists is a fact lookup, not redistribution of the dictionary; any *systematic* import of TDK content would require TDK's written permission first.
* Runtime consumption: `server/internal/words` (Go) and `client/lib/linguistics` (Dart) load the same bundle; neither ships Hunspell or Zemberek code. Zemberek runs only as a JVM oracle in CI, verifying the twin vowel-harmony implementations through the golden-file corpus.
* Version discipline: both sides load the same bundle tag (e.g., `tr-2026.08`); the server embeds it in `phase_started`, so a stale client knows its previews may drift — the server verdict still rules. Specialty slang/dialect dictionaries are additional bundles in the same format, selected per lobby through the Themed Rooms unlock (Monetization §6); root-catalogue rotations ride the same hot-swap mechanism as ordinary bundle bumps.

#### Designer Workbench & Tuning (`dictpack` subcommands)

`dictpack` doubles as the balancing tool: a seeded, config-driven sampler and Monte Carlo simulator over roots, suffix inventories, and difficulty. The workbench and the game server read the **same bundle and the same tuning file** — what was explored is exactly what ships.

Every knob lives in one versioned file, `configs/gameplay/tuning.yaml` — nothing hardcoded. The dividing line: *structure* (what a phase does, what may fast-forward, how settlement orders) is code; *every number a playtest could question* is a key:

```yaml
seed: 42                          # reproducible randomness — same seed, same pool

game:                             # lobby structure (Game Rules §1, §6)
  players: {min: 3, max: 6}
  matches: {default: 3, max: 6}   # host-set at lobby creation, fixed once the game starts
  min_connected: 2                # below this, the current match auto-completes and the game ends
  reconnect_grace_s: 20           # disconnect → counts as dropped after this many seconds
  dropout_penalty: 0.5            # final-stack multiplier for quitters / still-gone at finalization
  pokes_per_window: 1             # per player per wait window

timers:                           # every server-owned phase window, in seconds (Game Rules §§2–4)
  root_draft: 15                  # always runs full — blind to the end (§7)
  block_window: 15                # may end early on block unanimity
  board_turn: 15                  # one board on display per turn; may end early on Ready unanimity
  construction: 15                # may end early on Ready unanimity
  flag_window: 15                 # always runs full — blind to the end (§7)
  # which windows may fast-forward is structure (§7), not tuning — only durations live here

draft:
  ranks: 3                        # secret top-N root ranking per player (§2)

roots:
  frequency_weights: {common: 0.6, mid: 0.3, rare: 0.1}   # likelihood
  min_branching_factor: 8         # every root guarantees ≥8 real words
  min_bluff_surface: 15           # ≥15 morphable non-words (bluffability)
  length_range: [2, 10]           # hard feasibility clamp — premium rooms choose a sub-range
  length_weights: {short: 0.05, core: 0.85, long: 0.10}   # 2 / 3–6 / 7–10 letters — default deal is mostly core
  pos_mix: {noun: 0.7, verb: 0.3}
  exclude_tags: [proper, archaic, offensive]

difficulty:
  curve: match_progressive        # later matches draw rarer tiers
  tier_shift_per_match: 0.1

affixes:                          # renamed from `suffixes` — the inventory holds prefixes and fragments too (Word Engine §2)
  inventory: v1                   # named inventory sets, swappable wholesale
  set_size_offset: 10             # each private set holds players + 10 affixes
  max_pairwise_overlap: 0.10      # fresh sets are at least 90% different by label
  prefix_frequency: natural       # inventory-proportional; replace with a multiplier if playtests want more prefixes
  blocks: {min: 1, max: 3}        # the one final block action per player per match (§3)
  hand_minimum: 3                 # guaranteed by construction-window open
  topup_boards: 3                 # top-ups spread over the session's last N boards — 1/2/3 behind pace (§3)

scoring:
  seed_chips: 100
  letter_value: 1                 # per letter, root included
  longest_word_bonus: 30          # per match, shared equally on ties — the bonus `simulate` interrogates below
  clean_sweep_bonus: 15
  bluff_stakes: [20, 40, 60]      # first, second, then repeat the last value — eligibility gate and liability alike
  survived_bluff_reward: 60
  false_flag_fee: 20              # doubles as the flag-eligibility threshold: the gate covers the fee by design (§4)

xp:                               # server-side progression (Product Baseline) — v1 placeholder values
  game_completed: 20
  match_won: 10
  valid_word: 5
  correct_flag: 5
  rewarded_ad_multiplier: 2       # members receive it automatically, no ad (Monetization §§1, 3)
  training_daily_credited_sessions: 5   # offline Training Mode cap (Game Modes §1)

economy:
  free_daily_games: 10            # per free account per server day; Premium Membership = unlimited. Expected to tighten as the base grows.
  unlock_prices:                  # in game currency — v1 placeholders, tuned like everything else
    letter_forge: 2500
    custom_root_length: 1500
    themed_rooms: 1500
    custom_avatar: 1000

liveops:
  leaderboard_daily_counted_games: 10   # the daily grind cap Live Ops §4 reads from this file
  weekly_pool_min_level: 3              # account level to propose/vote (Game Modes §2) — v1 placeholder
```

Workbench subcommands:

| Command | What it answers |
|---|---|
| `dictpack roots --sample 11 --seed 7` | A draft pool exactly as the server would deal it, with per-root stats: tier, branching factor, bluff surface, best achievable word |
| `dictpack derive kafa --inventory v1` | Every attested derivation of a root, scored by the scoring table — with the morphable fakes alongside |
| `dictpack simulate --games 10000 --players 5` | Monte Carlo over full drafts with simple bot policies: longest-word distributions, score spreads, dead-draft probability, how often the catch-up mechanic flips a contested root |
| `dictpack diff tuning-a.yaml tuning-b.yaml` | Side-by-side comparison of two parameter sets before committing one |

* Why simulation, not just sampling: the scoring table is "v1 — to be tuned via playtesting." Monte Carlo answers the questions humans are slow at *before* playtests: does the +30 longest-word bonus dominate outcomes? does `match_progressive` actually tighten score gaps? what is the expected bluff-success rate given each root's bluff surface? Tune the YAML until distributions look right, then spend scarce playtest hours validating feel.
* Output discipline: every command takes `--csv`/`--json` for spreadsheet analysis, and every run prints its seed + tuning-file checksum so any interesting pool is reproducible in a bug report. A local web UI (`dictpack serve`) may come later; CLI + CSV covers tuning work and ships in Phase 2, not after it.

### 4. Server-Authoritative Anti-Cheat

* All game-deciding events — draft claims, word submissions, bluff flags, score transfers — are sent as *intents* and resolved exclusively by the server. A modified client can render anything it wants; it cannot change a verdict or a score.
* The server owns the phase clock: every phase window (15 s root draft, each match's 15 s block window, 15 s per shared board turn, 15 s submission, 15 s flagging) opens and closes on server time, and late intents are rejected server-side. Client timers are display-only.
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

* Protocol: One persistent WebSocket per client. Clients send **intents** (`rank_roots`, `lock_affix_blocks`, `take_affix`, `drop_affix`, `pass_affix_turn`, `submit_word`, `dispute_word`, `flag_bluff`); the server responds with **state events** (`phase_started`, `roots_resolved`, `affix_board_displayed`, `affix_turn_resolved`, `match_scored`) broadcast to the lobby. `lock_affix_blocks` carries 1–3 instance IDs and is accepted at most once per player per match, during that match's block window; `dispute_word` is fire-and-forget — logged for curation, never part of settlement. Messages are versioned JSON with sequence numbers for ordered replay.
* Fair arbitration: the root draft, the block window, each shared board window, and the bluff-flag window are blind-simultaneous — collected privately and resolved at window close — so no game-deciding event is a fastest-tap race. Block selections stay private from confirmation until their board's display turn; the server is the referee for every deadline and duplicate intent.
* Reconnect model: A dropped client reconnects with its session token — anonymous and server-issued per connection until Phase 4 binds tokens to accounts — and receives a full state snapshot of the current match (20-second grace window; dropout rules in Game Rules §6).
* Lobby→node affinity: Every lobby lives on exactly one server instance (Redis maps `lobby_id → node`); no cross-node game state. This is the property that makes horizontal scaling — and later Kubernetes — trivial.
* Live game state is held in server memory only; PostgreSQL records durable outcomes (profiles, game results), Redis handles routing, presence, and cross-node pub-sub.

---

## 🤖 Development & Test Bots

Bots exist to fill seats during development and testing — never to play against the public. Game Rules §6 stands: no bot ever takes over a production seat, and no bot enters Quick Play.

* Architecture — bots are ordinary clients: `tools/gamebot` (Go CLI) spawns N bot players that connect over the same WebSocket protocol, send the same intents, and obey the same server timers as humans — no server backdoors, so every bot game exercises exactly the code path a human game does. Word construction reuses `server/internal/words` as a library; bot accounts are flagged in PostgreSQL and excluded from XP, currency, statistics, and leaderboards.
* Policy — deliberately moderate, tunable, seeded: rank roots by a noisy branching-factor preference; block 1–3 pieces that fit rivals' public roots; on each board, take a piece that extends the bot's own root (greedy over engine derivations) or pass; construct a mid-length valid derivation rather than the optimum; **bluff occasionally** — with configured probability, when the stake gate allows, submit a morphable non-word from hand; flag occasionally on a noisy suspicion heuristic. Every bot runs on a seed, so a failing game replays exactly.
* Pace: bots act early and Ready immediately, so a bot-filled lobby fast-forwards through Ready unanimity — a full 6-seat dev game crosses every phase boundary in well under a minute of wall clock.
* Uses: solo development against 5 bots in a private dev room; the Phase 3 scripted integration test; Phase 4 load tests (hundreds of bot lobbies per node); feature smoke tests after every rules change. Division of labor: `dictpack simulate` answers balance questions offline; `gamebot` answers "does the live loop still work."
* Configuration: a `bots:` block lives only in environment overlays (`local.yaml`, `staging.yaml`) — `enabled`, policy rates (bluff, flag, drop, pass), action-delay range, seed. The key is absent from `prod.yaml`, and the server refuses bot connections when it is unset — production isolation by configuration shape, not by discipline.

---

## 📦 Infrastructure & Deployment

### 1. Containerization Principles

* Every runnable component ships as a container from day one: `server` (distroless/scratch Go image, single static binary), `postgres`, `redis`, and dev-only tooling (migrations runner, adminer).
* Images are multi-arch (amd64/arm64) and environment-agnostic — the **same image** runs under Compose locally, Kubernetes later, and any Terraform-provisioned host. Behavior differs only by mounted configuration.
* The server is built stateless-by-design: all durable state in PostgreSQL, all coordination state in Redis, live game state in memory with lobby→node affinity. This is the exact property Kubernetes needs, designed in now rather than retrofitted.

### 2. Centralized Configuration (No Scattered Env Vars)

* All configuration lives in `configs/` as clean, layered YAML: `base.yaml` holds every key with sane defaults; `local.yaml`, `staging.yaml`, `prod.yaml` are thin overlays that override only what differs.
* The Go server loads exactly one merged config into a single typed struct (`internal/config`) at boot and fails fast with a clear error listing any missing/invalid keys — no `os.Getenv` calls sprinkled through business logic.
* Environment variables are reserved for exactly two things: selecting the config file (`UYDURUM_CONFIG=/etc/uydurum/config.yaml`) and injecting **secrets** (DB password, JWT signing key) via `${VAR}` interpolation inside the YAML. Secrets never live in YAML files or images.
* This maps 1:1 onto the future platforms: Compose mounts `configs/local.yaml` as a volume; Kubernetes mounts the same file as a ConfigMap with secrets from a Secret resource; Terraform templates the same file per environment. One config model, three delivery mechanisms.

### 3. Local Development — Docker Compose

* `deploy/compose/docker-compose.yaml` brings up the full stack with one command: server (live-reload via air in dev profile), PostgreSQL with auto-applied migrations, Redis, and adminer.
* Compose profiles separate concerns: `core` (server+db+redis), `tools` (adminer, dashboards), `test` (ephemeral db for integration tests). No hand-managed `.env` sprawl — a single `deploy/compose/.env` holds only secrets and the config-file path.
* The Flutter client targets `ws://localhost` in `local.yaml`-mirrored client config; a full 6-player game must be playable against the local stack with zero cloud dependencies.

### 4. Kubernetes & Terraform Readiness (Future, Designed-For Now)

* Server exposes `/healthz` (liveness) and `/readyz` (readiness, checks Postgres/Redis connectivity) plus Prometheus metrics on a separate port — required for k8s probes, useful under Compose immediately.
* Graceful shutdown: on SIGTERM the server stops accepting new lobbies, drains in-flight games to completion, then exits — this makes k8s rolling deploys and node drains safe. Live-game handoff between nodes is explicitly out of scope: a draining node simply finishes its games.
* WebSocket routing under k8s uses sticky sessions/lobby-affinity at the ingress; because lobbies never span nodes, no service mesh or distributed state layer is needed.
* `deploy/terraform/` is structured as provider-agnostic modules (network, database, compute) with thin provider roots — consistent with the Hetzner-first / Azure-or-AWS-later hosting strategy.
* Rule: no k8s/TF-blocking decisions in application code — no local file writes, no in-container state, no hardcoded hostnames, config exclusively via the mounted YAML model above.

---

## 💰 Monetization Systems

### 1. Rewarded Multipliers

* Mechanic: Integrating ad provider SDKs (such as Google Mobile Ads) cleanly within the scoreboard UI. At the conclusion of a game, players can optionally watch a 30-second video to double their game XP or profile level progression points. Premium members never see the prompt — or any other ad surface — and their multiplier applies automatically (§3).
* Technical Impact: multipliers are granted exclusively via **ad-network server-side verification** (e.g., AdMob SSV): the ad network's servers call a verification endpoint on the game server with a signed payload; the server validates the signature and a one-time nonce, then applies the multiplier to the player profile in PostgreSQL. The client's completion callback is UX-only and grants nothing.

### 2. Game Currency

* One soft currency funds every standalone unlock: Letter Forge, Custom Root Length, Themed Rooms, and Custom Avatar (§§4–7) are all priced in it. It is deliberately **not** the in-match chip — chips are score, seeded fresh every game (Game Rules §5); currency is a persistent wallet with its own name and icon, and nothing converts between the two in either direction. (Final currency name pends the same Turkish-flavor pass as the rest of the brand.)
* Sources: **bulk packs via platform billing** (Play Billing / StoreKit, per the Compliance baseline) at launch; earned trickles (e.g., level-up grants) can attach later — every grant flows through the same server ledger, so adding sources never touches the spend path.
* Technical Impact: a PostgreSQL wallet with an append-only ledger (grants, spends, refunds); a debit is atomic with its entitlement write — an unlock either fully completes or fully rolls back. Balances live server-side only; the client renders them. Unlock prices live in `configs/gameplay/tuning.yaml` (`economy.unlock_prices`) and change without a client release.

### 3. Premium Membership (Monthly Subscription)

* Mechanic: hosting rooms and inviting players is **free for everyone** — the social loop is never paywalled. Premium Membership is a monthly entitlement with exactly two benefits: **no ads** — every ad surface disappears, the rewarded multiplier applying automatically (§1) — and **unlimited play**: a free account can start `economy.free_daily_games` games per server day (**10 at launch**, a pure config knob expected to tighten as the user base grows); members have no cap. Further perks (e.g., premium cosmetics) can attach later.
* Relation to the unlocks: Letter Forge, Custom Root Length, Themed Rooms, and Custom Avatar are **standalone game-currency purchases** (§2, §§4–7) — membership neither includes nor discounts them; the two lanes are fully independent.
* Technical Impact: membership is a PostgreSQL-backed entitlement with an expiry date, checked at every ad-surface render and at the daily-cap gate (queue and join intents). A game counts against the cap when the player's seat finalizes — quitting still counts. Expired memberships fail fast with a renewal prompt.

### 4. Letter Forge (Standalone Unlock — Room Option)

* Room setup: a host who owns the **Letter Forge** unlock (game currency, §2) may enable it at lobby creation. An enabled room admits only players who also own Letter Forge — it grants an in-match ability, so every seat needs it (enforced server-side at join intent); with the option off (default), the room is open to everyone. The flag is immutable after creation.
* Mechanic: in Letter Forge rooms, players may **add, change, or delete a single letter** in their constructed word during the showdown's construction window.
* Hard constraint: the drafted **root word is immutable** — letter edits apply exclusively to the constructive (suffix-built) portion of the word; any intent touching the root is rejected.
* Technical Impact: letter edits are sent as intents and validated server-side by the Go word engine (edit position must fall outside the root span; result still scored/validated normally).

### 5. Custom Root Length (Standalone Unlock — Room Option)

* Room setup: a host who owns the **Custom Root Length** unlock (game currency, §2) may set an explicit **root-length range** at lobby creation — minimum 2, maximum 10 letters — replacing the default sampler distribution. Unlike Letter Forge, the room stays **open to everyone**: the option shapes the root deck and grants nobody an in-game ability, so gating joiners would only hurt the social loop. The range is immutable after creation.
* Default rooms: root lengths follow the bundle's default distribution — minimum 2 letters, **mostly 3–6**, with longer roots appearing rarely (`length_weights` in `configs/gameplay/tuning.yaml`).
* Feasibility guardrail: the dict-pack precompute certifies, per bundle, which length ranges hold enough qualifying roots (branching factor, bluff surface, and pool size for a maxed lobby — 36 roots at 6 players × 6 matches). The lobby UI offers only certified ranges and the server rejects uncertified ones — no dead drafts by construction.
* Balance note: at +1/letter, a 10-letter root starts 8 chips ahead of a 2-letter root before affixes. `dictpack simulate` across candidate ranges is the acceptance gate before this option ships.
* Technical Impact: the range is a lobby-config field validated server-side against the bundle's certified ranges; the root sampler filters `roots.tsv` by length; the entitlement is checked at room creation exactly like Letter Forge (§4).

### 6. Themed Rooms: Root Catalogues & Specialty Dictionaries (Standalone Unlock — Room Option)

* Mechanic: one **Themed Rooms** unlock (game currency, §2) covers both ways of theming a room's word pool at lobby creation: a **root catalogue** — a topic-tagged root pool (health, weather, food, sports) sampled inside the standard bundle — or a **specialty dictionary** — an alternative dict-pack bundle entirely (slang, dialect, technical). This absorbs the former "Premium Host Ticket": specialty-dictionary hosting is no longer a separate purchasable. Like Custom Root Length (§5), themed rooms stay **open to everyone**: a theme shapes the deck and grants nobody an in-game ability.
* Live-ops rotation: each catalogue declares a refresh cadence — **daily, weekly, or monthly** — and a scheduled job activates new versions on schedule; specialty bundles update through ordinary dict-pack version bumps. Rotating content keeps a one-time unlock earning its price long after purchase. Running games are unaffected: the root pool is sampled once at game start.
* Composability: a catalogue and Custom Root Length stack — the sampler intersects both filters; a specialty bundle brings its own certified ranges. The lobby UI offers only combinations the active bundle certifies.
* Feasibility guardrail: the same certification as §5 — the dict-pack precompute verifies every catalogue, and every catalogue × length-range combination, holds enough qualifying roots for a maxed lobby (36 at 6 players × 6 matches); a specialty bundle certifies the same statistics in its own manifest; uncertified selections are unofferable in the UI and rejected server-side.
* Technical Impact: roots carry **topic tags** in `roots.tsv`, curated via `catalogues/` overlay lists in the pipeline; the chosen theme — catalogue id or bundle tag — is a lobby-config field validated server-side; rotations and specialty updates ship as bundle version bumps the server hot-swaps without redeploying. Since catalogues never alter word validity, client preview bundles need no update when a catalogue rotates; a specialty bundle, which does change validity, rides the normal version-discipline path (Word Engine §3).

### 7. Custom Avatar Unlock

* Mechanic: a one-time unlock priced in game currency (§2) that opens uploading a personal avatar image (Profiles & Community §2). Preset avatars stay free for everyone — identity is never paywalled, only the self-expression upload is.
* Technical Impact: a PostgreSQL entitlement checked at the upload endpoint; the upload itself always passes the automated moderation screen before display, and an admin takedown can revoke the privilege without refunding the spent currency.

---

## 🧾 Product Baseline (v1 Decisions)

Small decisions that unblock implementation — each deliberately minimal, expanded only when live data demands it:

* Matchmaking & discovery: private rooms use 6-character room codes (free for everyone); public play is a single Quick Play FIFO queue per dictionary bundle and lobby size. No public room browser and no skill rating at v1 — both are post-retention features.
* Progression: one server-side XP track. XP events (game completed, valid word, match won, correct flag) carry values in `configs/gameplay/tuning.yaml`; levels are a fixed XP-threshold table. Levels gate Weekly Pool proposals/votes and cosmetic unlocks. Offline Training XP stays daily-capped (§ Game Modes).
* Compliance (KVKK + GDPR): anonymous device accounts by default (no PII to protect), optional account linking later; privacy notice at first launch; Google UMP consent flow before any personalized ads; in-app delete-my-data action backed by a server endpoint; 13+ age gate; all purchases exclusively through platform billing (Play Billing / StoreKit).
* Analytics: no third-party client SDK at v1 — the authoritative server already witnesses every gameplay event. The Phase 4 scoring/audit events persist to PostgreSQL; nightly jobs derive the KPIs (retention, game completion, bluff submission/catch/survival by 20/40/60 tier, flag accuracy). Client-side funnel analytics wait for the store launch.
* Moderation & admin: nickname profanity filter (same filter as the Weekly Pool), player reports across categories (Profiles & Community §3 — inappropriate avatar, harassment, cheating/collusion; logged, no automated punishment at v1), and an authenticated admin endpoint set: kick, ban (account + device), close lobby, avatar takedown. Dictionary fixes ship as dict-pack version bumps — the server swaps bundles without redeploying; a stale client preview is cosmetic (server verdict rules).
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

* Task 1: Implement the lobby lifecycle and the versioned intent/event WebSocket protocol with sequence numbers and reconnect snapshots, keyed by anonymous server-issued session tokens (bound to accounts in Phase 4). Build the `tools/gamebot` harness alongside (§ Development & Test Bots) — this phase's own testing criteria depend on it.
* Task 2: Build the server-side phase state machine: server-owned phase timers (15 s root draft, each match's 15 s block window, 15 s shared board turns, 15 s submission, 15 s blind flag window), blind root-pick resolution with catch-up tie-breaks, the `players × matches` root pool with random final-match deal, mostly distinct per-match private sets, one irreversible 1–3-instance block per player per match, hidden non-exclusive takes, permanent block denial (a blocked instance never re-enters circulation), the public discard row and picked board, per-match hand clearing at settlement, last-three-turn top-ups, showdown submission, progressive 20/40/60 bluff stakes, blind flag resolution, and deterministic settlement.
* Task 3: Build the corresponding Flutter screens and phase state management against the live protocol.
* Testing Criteria: A scripted 6-bot integration test plays full games against the Compose stack, including forced mid-match disconnects/reconnects, with no state corruption; contested root picks resolve to exactly one winner; fresh private sets satisfy the overlap cap; a second block action in the same match is rejected; a dropped affix first becomes selectable on the next board; and caught first/second/later bluffs transfer 20/40/60 chips without charging unchallenged bluffers.

### Phase 4: Accounts, Persistence & Hardening

* Task 1: Implement auth (JWT sessions) and bind Phase 3's anonymous session tokens to accounts; player profiles, game-result persistence, and the report & feedback endpoints (Profiles & Community §3–4) in PostgreSQL; Redis presence and lobby→node routing.
* Task 2: Harden the intent pipeline: rate limiting, server-side deadline enforcement, input validation at the protocol boundary, and structured audit logs of scoring events — persisted to PostgreSQL as the v1 analytics event stream (see Product Baseline). Nightly jobs derive the public player statistics and the weekly leaderboard from the same stream (Profiles & Community §1, Live Ops §4).
* Testing Criteria: A deliberately modified client (forged scores, late intents, replayed messages) cannot alter any outcome; a `gamebot` load test sustains hundreds of concurrent bot-driven lobbies on one node.

### Phase 5: Monetization & Polish

* Task 1: Integrate rewarded ads via ad-network server-side verification (SSV) callbacks, the UMP consent flow, and platform billing; implement the game-currency wallet and append-only ledger with bulk packs (Monetization §2), the four standalone unlocks (Letter Forge, Custom Root Length, Themed Rooms, and Custom Avatar with its upload-and-moderation pipeline — Profiles & Community §2), cosmetic unlocks, and the Premium Membership entitlement (ad-free + unlimited play) with the config-driven free-tier daily game cap, plus the catalogue-rotation scheduled job.
* Task 2: Ship the auxiliary modes: Offline Training Mode (Dart `WordEngine`, daily-capped XP sync) and the Weekly Uydurum Pool (Postgres schema, scheduled job, proposal/voting screens).
* Task 3: Run performance profiling on client (layout paints on low-end devices) and server (allocation/GC under lobby load).
* Testing Criteria: Ad-completion events are verified server-side and no ad surface renders for an active member; entitlement checks gate unlock-priced rooms correctly; a free account's game past the daily cap is rejected at queue time and the counter resets on server day; currency debits are atomic with their entitlement writes; weekly pool windows open/close on schedule with one-proposal/one-vote enforcement verified; avatar uploads clear the automated screen before display, and admin takedown reverts the profile to presets.

### Phase 6: Linguistic Abstraction & Expansion

* Task 1: Verify the `Morphology` interface isolates all Turkish-specific logic; scaffold a second-language module as proof.
* Task 2: Localize client strings and dictionary-pack selection per lobby.
* Testing Criteria: A full game runs in a second language purely via configuration and a plug-in language module — zero core-engine changes.