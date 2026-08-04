# Server Cost Plan — Option A (Server-Authoritative WebSocket)

**Scope:** Cost projection for running Uydurum on the recommended architecture from [p2p-network-analysis.md](p2p-network-analysis.md): one authoritative WebSocket service owning lobby state, timers, draft arbitration, word validation, and scoring — plus auth and a score database. No TURN, no WebRTC infrastructure.

> Prices are approximate list prices as of early 2026 (Hetzner/DigitalOcean/AWS/Azure public pricing). Verify before committing; treat this as an order-of-magnitude plan.

---

## 1. Workload math (why this is cheap)

The whole plan rests on the fact that a turn-based JSON game is a *tiny* workload:

**Per lobby (6 players, ~6 rounds, ~8 min match):**

| Item | Estimate |
|---|---|
| Messages in (taps, picks, submissions) | ~20 msgs/player/round → ~720/match |
| Messages out (broadcasts ×6) | ~4,300/match |
| Avg message size | ~300 bytes |
| **Total bandwidth per match** | **~1.5 MB** (both directions) |
| Server CPU per event | µs-range (JSON parse + dict lookup + state update) |

**Concurrency model:**
- 1 lobby = up to 6 open sockets. Idle sockets cost only memory (~50–100 KB each in Node, less in Go).
- A 2 vCPU / 4 GB VPS comfortably holds **10,000+ concurrent sockets ≈ 1,600+ simultaneous lobbies ≈ 100,000+ matches/day**.
- Rule of thumb for planning: peak CCU ≈ 5–10% of DAU. So even **10,000 DAU → ~1,000 CCU → ~170 lobbies** — still a fraction of one small VPS.

**Conclusion up front:** you will not outgrow a single modest VPS until the game is a demonstrable success. Plan for stages, not for scale.

---

## 2. Cost stages

### Stage 0 — Development & playtesting (0–50 DAU)

| Item | Choice | Monthly |
|---|---|---|
| VPS (server + DB + monitoring, all-in-one) | Hetzner CX22 (2 vCPU, 4 GB, 20 TB traffic) | ~€3.8 / ~$4.5 |
| Database | PostgreSQL + Redis in the same Compose stack on the box | $0 |
| TLS | Let's Encrypt | $0 |
| Domain | .com/.dev (annualized) | ~$1 |
| **Total** | | **≈ $6/mo** |

Alternative: Fly.io or Railway hobby tiers can be ~$0–5/mo, but a plain VPS teaches you your real resource profile and avoids platform lock-in.

### Stage 1 — Soft launch (50–5,000 DAU)

| Item | Choice | Monthly |
|---|---|---|
| App VPS | Hetzner CX32 (4 vCPU, 8 GB) | ~€6.8 / ~$8 |
| Database | Postgres, self-hosted or managed (see §3) | $1–15 |
| Backups | Provider snapshot add-on (~20% of VPS) | ~$2 |
| Monitoring | Self-hosted Uptime Kuma + Grafana Cloud free tier | $0 |
| Error tracking | Sentry free tier | $0 |
| Push notifications | FCM | $0 |
| Auth | Roll your own JWT or Firebase Auth free tier | $0 |
| **Total** | | **≈ $10–25/mo** |

Notes:
- Hetzner's Germany/Finland regions give ~40–60 ms RTT to Türkiye — well within tolerance for this game. If your audience concentrates in Türkiye, DO Frankfurt or Hetzner Nuremberg are the sweet spots.
- 20 TB included traffic ≈ **13 million matches/month** at 1.5 MB/match. Bandwidth will never be your bill.

### Stage 2 — Traction (5,000–50,000 DAU)

This is the first point where architecture (not money) needs a decision: one bigger box vs. two smaller ones.

| Item | Choice | Monthly |
|---|---|---|
| App servers ×2 (redundancy, zero-downtime deploys) | 2× Hetzner CX32 | ~$16 |
| Load balancer | Hetzner LB11 (sticky sessions for WebSockets) | ~$6 |
| Managed Postgres | DO Managed 1 GB / Neon Scale | ~$15–30 |
| Redis (lobby routing / pub-sub between nodes) | Self-hosted on app node or managed starter | $0–15 |
| Backups + monitoring | | ~$5 |
| **Total** | | **≈ $45–75/mo** |

Key design cost-saver: lobbies are **fully independent** — no cross-lobby state. Shard lobbies by node with sticky routing and you scale horizontally forever without distributed-state complexity. Bake this assumption in from Stage 0 (lobby ID → node affinity).

### Stage 3 — Success problem (50,000+ DAU / ~5,000+ CCU)

| Item | Monthly |
|---|---|
| 3–4 app nodes + LB | ~$40 |
| Managed Postgres (HA) | ~$50–100 |
| Managed Redis | ~$15–30 |
| Observability (paid tier) | ~$20–50 |
| CDN for static/Rive assets (Cloudflare free → Pro) | $0–20 |
| **Total** | **≈ $125–250/mo** |

At this stage, rewarded-ad revenue math: even a conservative $8–15 eCPM on rewarded video with 10% of 50k DAU watching one ad/day ≈ **$1,200–2,250/mo** — infrastructure is ~10% of ad revenue. The architecture pays for itself long before this point.

---

## 3. Database cost plan

**What the DB actually stores** — and, just as important, what it doesn't. Live match state (timers, drafts, hands) lives in server RAM and never touches the database; the DB only sees durable records:

| Data | Row size | Growth |
|---|---|---|
| Player profiles (auth, XP, cosmetics, tickets) | ~1 KB | 1 row per player, ever |
| Match results (final scores, word list, bluff outcomes) | ~2–5 KB | 1 row per match |
| Leaderboards / aggregates | — | Computed, or small materialized tables |

**Sizing reality check:** 50,000 DAU playing 5 matches/day ≈ 40k match rows/day ≈ **~4 GB/year** raw, less with pruning (keep full match detail 90 days, aggregates forever). The DB stays in single-digit gigabytes for years — you are never buying a big database, only deciding who operates it.

**The database ladder:**

| Stage | Option | Monthly | When to step up |
|---|---|---|---|
| 0 | Postgres (+ Redis) inside the app box's Compose stack | $0 | Move when you add a 2nd app node or when backups/uptime become your problem |
| 1 | Postgres on the app box + nightly `pg_dump` to object storage | ~$1 | Move when backups/uptime become your problem, not your hobby |
| 1–2 | Managed starter: DO Managed 1 GB ($15), Neon Launch (~$19), Supabase Pro ($25) | $15–25 | Move at sustained load or when you need PITR |
| 2–3 | Managed 2–4 GB + standby: DO ($30–60), AWS RDS `db.t4g.small` multi-AZ (~$50–60 + ~$5 storage), Azure PostgreSQL Flexible `B2s` + HA (~$60–120) | $30–120 | — |

**Rules of thumb:**
- Managed Postgres pricing is dominated by compute + HA replicas, not storage — at your data sizes, storage line items are pennies.
- Do not use serverless DBs with per-request pricing (DynamoDB on-demand, Aurora Serverless) for the score DB — a fixed tiny instance is cheaper and simpler at every stage of this plan.
- Redis (Stage 2+) is for lobby→node routing and pub-sub only; it holds no durable data, so the free/self-hosted option is safe.

---

## 4. Azure / AWS path (if Hetzner-class hosting won't scale)

First, the honest calibration: **Hetzner-class hosting scales further than this game will need** — their dedicated line serves workloads far beyond Stage 3, and the lobby-sharded architecture scales horizontally on any provider. The realistic reasons to move to AWS/Azure are not raw scale but: multi-region expansion, compliance requirements, managed-HA guarantees, or startup credits.

**Stage-equivalent pricing (like-for-like architecture):**

| Stage | AWS | Azure |
|---|---|---|
| 0 — Dev | Lightsail 2 GB ($12) + SQLite → **~$12** | B1s/B2ts VM (~$8–15) → **~$10–15** |
| 1 — Soft launch | `t4g.medium` (~$25) + RDS `db.t4g.micro` (~$15) + ~150 GB egress (~$13) → **~$50–65** | B2s VM (~$30) + PostgreSQL Flexible `B1ms` (~$15–30) + egress → **~$50–75** |
| 2 — Traction | 2× `t4g.medium` + ALB (~$20) + RDS `t4g.small` (~$30) + Redis on node → **~$120–180** | 2× B2s + LB/App Gateway + Flexible `B2s` → **~$150–250** |
| 3 — Success | 3–4 nodes + ALB + RDS multi-AZ + ElastiCache + egress → **~$350–600** | Equivalent + optional Web PubSub → **~$400–700** |

**What changes the math on big clouds:**
- **Egress billing** (~$0.09/GB) is the item Hetzner makes invisible (20 TB included). At ~1.5 MB/match this stays modest — 1M matches/month ≈ $135 — but it's the line to watch and the reason bandwidth discipline (§6) matters more there.
- **Managed WebSockets exist** if you want them: Azure Web PubSub / SignalR Service (~$50/mo per 1,000 concurrent connections) or AWS API Gateway WebSockets (per-message + per-minute billing — gets expensive with chatty broadcasts). Fine as an ops shortcut on Azure; on AWS, plain EC2 sockets are cheaper for this traffic shape.
- **Startup credits flip the comparison**: AWS Activate ($1k–5k+) and Microsoft for Startups Founders Hub (up to $150k Azure credits, no funding requirement) can make Azure/AWS effectively free for the first year+. If you qualify, starting on Azure with credits is a perfectly rational Stage 0–2 plan — just stay portable.

**The portability insurance policy:** build as one container + Postgres + Redis, no provider-proprietary services (no Lambda-shaped logic, no DynamoDB, no SignalR-specific protocol). Then Hetzner → Azure/AWS (or the reverse, when credits expire) is a weekend migration, and you can defer the provider decision indefinitely.

**On home servers:** fine for development and LAN playtesting, not for production — residential uplinks mean CGNAT (players can't reach you reliably), no uptime guarantee, and poor IP reputation. A $6 VPS beats a home server on every axis that matters here except sentiment.

---

## 5. Costs you explicitly avoid by choosing Option A

| Avoided item | Would have cost |
|---|---|
| TURN relay bandwidth (mesh fallback) | $0.04–0.09/GB metered, or ~$5–20/mo self-hosted coturn + ops time |
| Managed WebRTC infra (if outsourced) | LiveKit/Daily-style pricing at per-participant-minute rates — easily $100s/mo at modest scale |
| Mesh debugging engineering time | The largest hidden cost — weeks of work with no player-visible feature output |

Note the irony: Option C ("serverless" P2P) still required signaling + TURN + auth/score DB, i.e., **more** infrastructure than Stage 1 above.

---

## 6. Cost-control principles

1. **One box, until it hurts.** The same Compose stack everywhere — Go server + Postgres + Redis on one VPS, dictionary in RAM (a 100k-word set is <10 MB — no reason for it to live anywhere but memory).
2. **Never pay for idle.** No Kubernetes, no autoscaling groups, no managed message queues at this scale — each adds fixed cost and ops burden for capacity you won't use.
3. **Bandwidth discipline is free:** send diffs not snapshots, and gzip/permessage-deflate on the WebSocket. Halves the (already negligible) traffic.
4. **Budget alarm:** set provider billing alerts at 2× expected spend from day one. The failure mode to guard against isn't gradual growth — it's a bug (reconnect storm, broadcast loop) burning bandwidth.
5. **Exit costs stay near zero** if you avoid provider-proprietary services: plain VPS + Postgres + Redis moves anywhere in an afternoon.

---

## 7. Summary table

| Stage | DAU | Infra | Hetzner-class | AWS | Azure |
|---|---|---|---|---|---|
| 0 — Dev | <50 | 1 small VPS + SQLite | **~$6** | ~$12 | ~$10–15 |
| 1 — Soft launch | 50–5k | 1 VPS + Postgres + backups | **~$10–25** | ~$50–65 | ~$50–75 |
| 2 — Traction | 5k–50k | 2 nodes + LB + managed DB | **~$45–75** | ~$120–180 | ~$150–250 |
| 3 — Success | 50k+ | 3–4 nodes + HA DB + observability | **~$125–250** | ~$350–600 | ~$400–700 |

**Bottom line:** Option A costs about one coffee per month until you have thousands of daily players, roughly a Netflix subscription until tens of thousands, and stays under ~10% of plausible ad revenue at every stage after launch — on any of the three providers. The big clouds run ~3–5× Hetzner for identical architecture, but startup credits can neutralize that for a year or more; staying container+Postgres portable means the provider choice is reversible either way. Cost is not a reason to prefer P2P for this game.
