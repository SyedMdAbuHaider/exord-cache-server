# HELIOS — Adaptive ISP Cache Fabric
### The Complete A–Z Technical Reference

**What this is not:** a Go rewrite of LanCache. **What this is:** a self-learning, traffic-aware caching fabric that predicts what to cache before it's requested, classifies traffic in the kernel instead of userspace, and proves its own ROI in real dollars and watts — not just a hit-ratio percentage.

This document is the single source of truth: every subsystem, decision, and rationale, A to Z.

---

## Positioning — why this isn't "just another LanCache clone"

| Dimension | Traditional caches (LanCache, basic transparent proxies) | Commercial appliances (Qwilt, PeerApp) | **HELIOS** |
|---|---|---|---|
| Traffic classification | DNS redirection only | Proprietary deep packet inspection, closed appliance | **eBPF/XDP in-kernel classification** — line-rate, no userspace hop, fully inspectable |
| What gets cached | Whatever gets requested | Popularity-based, reactive | **Predictive pre-warming** — cache objects *before* the first customer request, from release calendars and patch telemetry |
| Config management | SSH + YAML | Vendor GUI, closed | **GitOps** — every policy change is a pull request, reviewed, versioned, revertible |
| Storage tiering | Static or none | Vendor black box | **ML-assisted tier prediction** — learns access patterns per object class, not just LRU |
| Reporting | Hit ratio % | Vendor dashboard | **Yield Analytics** — bandwidth saved in ৳/$ *and* kWh, tied to your actual upstream contract cost |
| Ownership | N/A (OSS, unsupported) | Vendor lock-in, appliance lease | **You own the fabric** — runs on your hardware, your data, your roadmap |

That's the pitch. Now the substance, A to Z.

---

## A — Architecture Overview

Three planes, strictly separated:

```
DATA PLANE (survives everything else dying)
   Customer → Core Router → HA Gateway → Cache Node(s) → Origin (on miss)

CONTROL PLANE (never in the request hot path)
   Admin/API → Controller → Postgres + Redis → NATS → Cache Nodes (config push)

INTELLIGENCE PLANE (new — this is what makes HELIOS not a clone)
   Traffic telemetry → Prediction Engine → Pre-warm Scheduler → Cache Nodes
```

The Intelligence Plane is additive: if it's down, HELIOS behaves exactly like a conventional reactive cache. It never gates the data path — prediction only ever *adds* pre-warmed objects, it never blocks a request waiting for a decision.

---

## B — Bandwidth Economics & Savings Engine

Every served byte is tagged with its **origin cost basis**: your actual upstream ৳/GB rate (configurable, since IIG contracts vary). The engine computes, per hour:

```
bandwidth_avoided_bytes = cache_bytes_served - origin_bytes_fetched
cost_avoided = bandwidth_avoided_bytes × contracted_rate_per_GB
```

This number — not hit ratio — is the headline metric on the dashboard, because it's the number that means something to whoever signs off on infrastructure spend.

---

## C — Cache Decision Engine

```
Request → Normalize → Domain Registry lookup → Policy Evaluation
                                                     │
                          ┌──────────────────────────┼──────────────────────────┐
                          ▼                          ▼                          ▼
                       CACHE                     BYPASS                   PASSTHROUGH
                          │
                     Cache Lookup → HIT (serve) / MISS (coalesce → origin → validate → store)
```

Unchanged in principle from a conventional cache — the difference is *what feeds the policy*: static rules plus live signals from the Intelligence Plane (§P, §M).

---

## D — Data Plane & Distributed Storage

Each cache node owns its storage independently — no synchronous distributed filesystem, no Ceph, no consensus protocol on the hot path. Content-addressed layout:

```
/cache/objects/<sha256[0:2]>/<sha256>
```

with metadata (ETag, TTL, hit count, segment map) kept in a local embedded index (BoltDB/BadgerDB) for sub-millisecond lookups, asynchronously mirrored to Postgres for the control plane's view.

---

## E — eBPF/XDP Traffic Classification *(key differentiator)*

Instead of classifying traffic only via DNS response, HELIOS attaches an **XDP program** at the NIC driver level on the gateway. This does two things traditional DNS-only steering can't:

1. **Catches traffic that bypasses your DNS** (customers using 8.8.8.8/1.1.1.1 directly) by matching on SNI/IP ranges known to belong to cacheable CDNs — without decrypting anything.
2. **Line-rate pass/cache/drop decisions in the kernel**, before packets ever reach userspace — this is the same technique used by Cilium/Meta's Katran, applied to cache steering instead of load balancing.

This is the single biggest architectural leap over "DNS redirect and hope," and it's genuinely advanced — most ISP-grade caches, including commercial ones, still lean heavily on DNS alone.

---

## F — Failure Matrix & Fail-Open Design

| Component down | Effect | Behavior |
|---|---|---|
| Postgres | None immediate | Nodes run on last-pulled policy snapshot |
| Redis | Coalescing degrades | Falls back to per-request origin fetch (logged, alerted) |
| NATS | Config frozen | Nodes keep serving on last-known-good config |
| Intelligence Plane | No pre-warming | Falls back to pure reactive caching — never blocks |
| Gateway (non-HA) | Total outage | This is why Gateway HA is mandatory, not optional (§H) |

Universal rule: **ambiguity always resolves to passthrough**, never to blocking the customer.

---

## G — GitOps Policy Management *(key differentiator)*

Domain policies, cache rules, and eviction weights live as YAML in a Git repo, not a database edited via GUI:

```
policies/
  steam.cdn.yaml
  windows-update.yaml
  epic-games.yaml
```

A merge to `main` triggers CI validation (schema + simulation against recent traffic logs, see §Z) then publishes a signed `configuration_version` via NATS. Every policy change has a PR history, a reviewer, and a one-click `git revert`. This turns "who changed the cache policy and why" from a mystery into `git blame`.

---

## H — HA Gateway & Health State Machine

Deployed as an active/active pair behind VRRP/keepalived — the one component in the whole system without a graceful degraded mode, so it gets the most redundancy. Node health isn't binary:

```
HEALTHY → DEGRADED → DRAINING → UNHEALTHY → OFFLINE
```

`DEGRADED` (e.g., disk at 90%) stops new cache writes but keeps serving reads — a much better failure mode than either "pretend everything's fine" or "take the node down."

---

## I — Integrity & Anti-Poisoning Validation

Before any object is marked `READY`:
- Content-Length match, checksum computation, ETag/Last-Modified consistency across segments
- Unexpected redirect or status-code change mid-fetch → reject
- Failures route to `CORRUPTED` → `QUARANTINED`, retained as metadata-only for a cooldown to prevent re-fetch storms, never served

---

## J — JSON Structured Logging & Distributed Tracing

Every request gets a `request_id` carried through logs (Loki) and traces (OpenTelemetry → Tempo). Structured JSON, not `access.log` text — so "show me every request that touched this object in the last hour" is a query, not a grep.

---

## K — Kubernetes vs. Bare-Metal — the deliberate choice

Control plane: Kubernetes optional, fine if you already run one. Cache nodes: **bare-metal + systemd/containerd, deliberately not Kubernetes.** I/O-heavy cache workloads want predictable NIC and disk affinity that a scheduler fighting for the same resources works against. This is a considered trade-off, not a skill gap — Kubernetes is the wrong tool for the data plane here.

---

## L — Locking / Singleflight Request Coalescing

500 customers requesting the same 50GB update simultaneously must trigger exactly **one** origin fetch:

```
500 clients → cache key → Redis distributed lock → ONE origin fetch → fan-out to all 500
```

Implemented with Go's `singleflight` package backed by a Redis lock for cross-node coordination.

---

## M — Multi-Tier Storage with ML-Assisted Placement *(key differentiator)*

Beyond static LRU/LFU, HELIOS trains a lightweight per-domain model (gradient-boosted trees, retrained nightly, not deep learning — this doesn't need a GPU) on features like:

```
time_of_day, day_of_week, object_age, size, publisher,
historical_hit_curve, is_patch_tuesday, is_release_window
```

Output: a predicted hit-probability per object, used to decide NVMe vs. SSD vs. HDD placement *before* the object cools down naturally — catching patterns like "this file will be hot again on the last Tuesday of the month" that simple recency-based eviction misses entirely.

---

## N — Node Agent & Heartbeat Protocol

Every cache node runs `cache-agent`, reporting every few seconds:

```json
{"node":"cache-01","cpu":31,"ram":42,"disk":68,
 "net_bps":8400000000,"cache_state":"HEALTHY","version":"1.4.2"}
```

Missed heartbeats beyond a threshold → gateway routes around the node automatically.

---

## O — Observability Stack

Prometheus (metrics, low-cardinality labels only — never IP/URL) → Grafana (dashboards) → Loki (structured logs) → Tempo (traces) → Alertmanager (paging). Recording rules pre-aggregate hit ratio and bandwidth-avoided at 5-minute resolution so dashboards stay fast even at high query volume.

---

## P — Predictive Pre-Warming Engine *(the headline differentiator)*

This is what makes HELIOS feel advanced rather than reactive:

1. Subscribe to publisher release calendars / patch-note feeds (Steam, Windows Update, major game publishers) where available.
2. On a known release/patch date, pre-fetch the object to cache nodes **before customers request it** — during off-peak hours, throttled so it doesn't itself spike origin traffic.
3. Result: the *first* customer to download a new game patch on release day gets a cache hit, not just the hundredth.

This directly targets the exact pain point that makes caching "noticeable" — big, predictable traffic spikes (patch days, game releases) are precisely when bandwidth cost hurts most, and precisely when a purely reactive cache is least effective (everyone's a miss until the cache warms up on its own).

---

## Q — QUIC/HTTP-3 Readiness

Increasing CDN traffic is shifting to QUIC (UDP-based, encrypted transport headers). HELIOS's classification layer (§E) is designed to fall back gracefully to SNI/IP-based classification for QUIC flows it can't parse via traditional HTTP semantics, rather than silently failing to cache them — flagged explicitly as `passthrough: quic-unclassified` in metrics so you can see the gap rather than have it hide inside a generic miss count.

---

## R — Replication & Hot-Object Propagation

Asynchronous, selective, event-driven — not a distributed filesystem:

```
Cache-01 detects object crossing hit-count threshold
        → publishes cache.object.hot event via NATS
        → Controller schedules replication to Cache-02/03
```

Cold objects are never replicated — this keeps replication traffic proportional to value, not volume.

---

## S — Security Model — Zero-Trust Internal, Never MITM External

- mTLS between every internal component (node ↔ controller, controller ↔ Postgres proxy)
- OIDC for human users, RBAC roles (`SUPER_ADMIN` → `AUDITOR`), every mutation written to an immutable audit log
- **Hard boundary:** never TLS-intercept customer HTTPS traffic. HTTPS is cached only via legitimate CDN/publisher-supported mechanisms; everything else is explicitly classified `passthrough: https-unsupported`, never MITM'd. This is a trust commitment, not just a technical limitation.

---

## T — Tenant Model (future-proofed for productization)

Every table that could ever need per-customer scoping carries a `tenant_id` from day one, even when there's only one tenant (Exord itself). This means "we're going to license this to another ISP" (as discussed) becomes a configuration change, not a schema migration — the cost of doing this now is a few extra foreign keys; the cost of doing it later is rewriting the data model under production traffic.

---

## U — Unique Differentiators, Summarized

1. eBPF/XDP kernel-level classification (§E) — catches traffic DNS-only systems miss
2. Predictive pre-warming from release calendars (§P) — cache before the spike, not during it
3. ML-assisted storage tiering (§M) — learns patterns LRU can't see
4. GitOps policy management (§G) — reviewable, revertible, auditable by design
5. Yield Analytics in currency and energy, not just hit-ratio percentage (§Y)
6. Explainability dashboard for every cache/evict decision (§X)
7. Multi-tenant from day one (§T) — built to be sellable without a rewrite

---

## V — Versioned Configuration & Rollback

Every published config is a numbered `configuration_version` row, signed and checksummed. A node that receives an invalid or unsigned bundle automatically rejects it and stays on the last good version — bad config can be pushed by mistake, but it can never be *applied* by mistake.

---

## W — Workflow: CI/CD & Chaos Testing

GitHub Actions pipeline: lint → unit tests → integration tests against a synthetic origin → **chaos tests** (kill a cache node mid-test-suite, verify gateway reroutes within N seconds; kill Redis, verify coalescing degrades gracefully instead of erroring). Chaos testing isn't optional polish here — it's the only way to actually trust the fail-open claims in §F before production traffic tests them for you.

---

## X — eXplainability Dashboard

For any object, a single API call answers "why is this cached / why was this evicted / why wasn't this pre-warmed":

```json
{
  "object": "steam/app/730/update_18452.vpk",
  "cached": true,
  "reason": "predicted_hit_probability=0.92, release_window=true",
  "tier": "nvme",
  "tier_reason": "ml_score=0.87 > promotion_threshold=0.75"
}
```

This matters for trust — both yours, debugging why the cache behaved a certain way, and a future customer's, if this is ever sold and they ask "how does it decide what to cache."

---

## Y — Yield Analytics / ROI Dashboard

The panel that creates the "I can actually see this working" moment:

```
┌─────────────────────────────────────────────┐
│              HELIOS — TODAY                  │
├─────────────────────────────────────────────┤
│ Bandwidth avoided        14.7 TB             │
│ Cost avoided             ৳XX,XXX             │
│ Energy avoided (est.)    XXX kWh             │
│ Pre-warm hit assist      +12% of total hits  │
│ Hit ratio                83.4%               │
└─────────────────────────────────────────────┘
```

Energy estimate uses a configurable ৳or-kWh-per-GB-transited figure for your upstream link — a genuinely differentiating metric almost no competitor reports, and one that resonates with sustainability-conscious enterprise customers if this is ever sold.

---

## Z — Zero-Downtime Deployment & Digital Twin Simulation

Two things, both aimed at "never let this look worse than having no cache":

1. **Zero-downtime node upgrades** via the `DRAINING` state (§H) — a node finishes in-flight requests, stops taking new ones, upgrades, rejoins.
2. **Digital Twin simulation mode** — before a policy change goes live (§G), replay the last 24 hours of real traffic logs against the *proposed* policy in a sandboxed instance and diff the projected hit ratio and bandwidth-avoided against current production numbers. This is what lets you say yes to a policy PR with actual evidence instead of hoping it works.

---

## Build Order (unchanged priority, now dated against this doc)

Steps 0–8 from the prior build plan still apply as the implementation sequence. The Intelligence Plane (§P, §M, §X, §Z simulation) is explicitly **Phase 2 work** — build the honest, reliable, observable reactive cache first (Steps 0–7), prove the bandwidth-avoided number is real and trusted, and only then layer in prediction and ML tiering. Shipping "advanced" before "correct" is how trust in the whole system gets lost on day one.
