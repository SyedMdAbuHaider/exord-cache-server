# ISP Central Cache Platform — Architecture v2 + Step-by-Step Build Plan

Supersedes v1's architecture doc for implementation purposes. Same component boundaries, updated for the fast/reliable/noticeable-savings priorities: hit-ratio maximization, zero-copy serving, gateway HA, fail-open reliability.

---

## 1. Locked Tech Stack

| Layer | Choice | Why |
|---|---|---|
| Cache engine / gateway | **Go 1.23+** | goroutine-per-request model, `sendfile`/`splice` via `io.Copy` on `*os.File`, single static binary for deployment |
| Reverse proxy / L4-L7 in front of gateway | **NGINX or Envoy** (HA pair) | mature keepalive/VRRP support, proven at ISP scale |
| DNS resolver | **CoreDNS with a custom Go plugin** | plugin architecture fits the "consult local domain-registry cache" model without writing a resolver from scratch |
| Metadata DB | **PostgreSQL 16** | ACID metadata, JSONB for policy blobs, mature replication |
| Hot state / locks / DNS decision cache | **Redis 7 (or KeyDB for multi-threading)** | singleflight locks, sub-ms lookups |
| Event bus | **NATS JetStream** | lightweight, persistent streams for config propagation, no Kafka/ZooKeeper overhead you don't need yet |
| Object storage | **Local filesystem, XFS**, content-addressed layout | XFS handles large files and high metadata churn better than ext4 at this scale |
| Storage tiering | **NVMe (hot) → SSD (warm) → HDD (cold)**, manual tier hints + LRU/LFU hybrid promotion | matches real access patterns of game/OS updates |
| Admin UI | **React + TypeScript + Vite** | matches your existing stack (HRM, ticketing) — reuse component patterns |
| Observability | **Prometheus + Grafana + Loki + Tempo (OpenTelemetry)** | standard, no vendor lock-in |
| Secrets | **Vault** (or SOPS+age if Vault is overkill at your scale) | never plaintext creds in config |
| CI/CD | **GitHub Actions** | matches your existing GitHub-based workflow |
| Node orchestration | **systemd + containerd**, NOT Kubernetes | bare-metal predictability for I/O-heavy cache nodes; Kubernetes only for control plane if you already run one |
| Config delivery | **Signed, checksummed, versioned bundles pushed via NATS**, validated + auto-rollback on the node | no SSH-and-edit-YAML |

---

## 2. Updated Architecture (incorporating HA + performance decisions)

```
                              INTERNET
                                 │
                          IIG / Upstream
                                 │
                          ┌──────▼──────┐
                          │ Core Router │   MikroTik CCR
                          └──────┬──────┘
                                 │
                        ISP customer network
                                 │
                                 ▼
                 ┌───────────────────────────────┐
                 │      CACHE GATEWAY — HA PAIR   │
                 │   NGINX/Envoy, VRRP/keepalived │
                 │   NIC A: client | NIC B: origin│
                 │   circuit breaker to origin    │
                 │   fail-open on any ambiguity   │
                 └───────────────┬───────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
          Cache-01            Cache-02            Cache-03
        NVMe→SSD→HDD        NVMe→SSD→HDD        NVMe→SSD→HDD
        sendfile/splice     sendfile/splice     sendfile/splice
        singleflight lock   singleflight lock   singleflight lock
        segment map          segment map          segment map
              │                  │                  │
              └──────────────────┼──────────────────┘
                                 │
                     Async replication (hot objects only)
                                 │
                     ┌───────────────────────┐
                     │   CONTROL PLANE       │  (off data path entirely)
                     │  API/UI → Controller  │
                     │  Postgres + Redis     │
                     │  NATS JetStream       │
                     └───────────┬───────────┘
                                 │
                        OBSERVABILITY
              Prometheus → Grafana | Loki | Tempo | Alertmanager
              Live panel: Gbps served-from-cache vs Gbps to origin
```

---

## 3. Step-by-Step Build Plan

### Step 0 — Baseline measurement (before writing any code)
1. Pull top-talker domains by byte volume from MikroTik/NetFlow/sFlow for the last 30 days.
2. Rank by volume, not request count — pick the top 15–20 domains that will actually move the bandwidth bill (expect Steam, Windows Update, a couple of game publishers, maybe Linux mirrors).
3. Record current upstream Mbps baseline at peak hour — this is the number you'll show improving.

### Step 1 — Single cache node, no HA, HTTP-only domains
1. Provision one server: NVMe for hot tier + cache index, HDD/SSD for bulk storage, dedicated NIC for origin traffic if possible.
2. `sysctl` tuning up front (don't wait): `net.core.rmem_max`, `net.core.wmem_max`, increase `fs.file-max` and ephemeral port range, enable **BBR** congestion control.
3. Build the Cache Node binary in Go:
   - HTTP reverse proxy handler
   - Content-addressed storage writer (SHA-256 based path)
   - Cache-key normalization per the domain policy (start with a hardcoded policy file, not a DB, for step 1)
   - Serve hits via `io.Copy` from an `*os.File` (confirm `sendfile` is actually engaged — verify with `strace`)
4. Point DNS for your 15–20 chosen domains at this single node (manual `/etc/hosts`-style override or a minimal DNS zone — not CoreDNS yet).
5. Measure: hit ratio, Mbps served from cache vs. origin. This is your proof-of-concept milestone.

### Step 2 — Reliability primitives
1. Add a circuit breaker around origin fetches (fail fast to direct passthrough after a short timeout — a few hundred ms — rather than hanging).
2. Add fail-open behavior everywhere: policy fetch stale → passthrough; disk full → passthrough + alert; node overloaded → passthrough.
3. Add persistent connection pooling to origin CDNs (Go `http.Transport` with `MaxIdleConnsPerHost` tuned up) to cut miss latency.
4. Add request coalescing (singleflight) so concurrent requests for the same cold object don't each hit origin — this is where Redis enters, as the distributed lock.

### Step 3 — Multi-tier storage + segment engine
1. Introduce NVMe/SSD/HDD tiering with a promotion/demotion policy based on hit count + recency.
2. Implement object segmentation (e.g., 64MB chunks) with a segment presence map, so Range requests on large objects (game/OS updates) don't require the whole object to be cached before serving partially.
3. Add checksum validation per segment — this is your cache-poisoning defense, not optional.

### Step 4 — Control plane
1. Stand up PostgreSQL: domains, services, providers, domain_policies (versioned), nodes, objects metadata.
2. Stand up NATS JetStream for config propagation; node pulls/validates/applies config bundles, rolls back on validation failure.
3. Build the minimal API (`/domains`, `/policy`, `/purge`, `/statistics/summary`) — no UI yet, curl/Postman is fine at this stage.
4. Migrate domain policy from the hardcoded file (Step 1) into Postgres-backed, versioned policy.

### Step 5 — Horizontal scale + Gateway HA
1. Add a second and third cache node; put NGINX/Envoy in front as the Cache Gateway.
2. Deploy the gateway as an HA pair with VRRP/keepalived — this is the step where "single point of failure" gets closed. Do not skip or defer this once you have real customer traffic on the cache.
3. Add async replication of hot objects (hit-count threshold based) between nodes so gateway failover doesn't mean a fully cold cache on the surviving node.

### Step 6 — Observability that makes savings visible
1. Prometheus metrics: `cache_requests_total`, `cache_hits_total`, `cache_bytes_served_total`, `cache_origin_bytes_total`, `cache_request_duration_seconds` — labeled by `service`/`pop`/`result`, never by IP or URL.
2. One Grafana panel, front and center: live Gbps served-from-cache vs. Gbps to origin, plus a running "bandwidth avoided this month" counter.
3. Loki for structured JSON access logs (per-request detail lives here, not in Prometheus labels).
4. Alertmanager: page on gateway pair both down, disk >90% on any node, hit ratio dropping sharply (signals a policy or origin problem).

### Step 7 — Admin UI + RBAC
1. React/TS admin panel: node health, domain policy editor (with versioning/rollback in the UI), purge tool, bandwidth dashboard.
2. RBAC roles (`SUPER_ADMIN`, `NETWORK_ADMIN`, `CACHE_ADMIN`, `NOC_OPERATOR`, `READ_ONLY`, `AUDITOR`) enforced at the API Gateway; every mutating action written to `audit_logs`.

### Step 8 — Expand domain coverage
1. Only after Steps 1–7 are stable in production: widen from your initial 15–20 domains using the same volume-first prioritization from Step 0, re-measured monthly.
2. Add HTTPS-cacheable domains only where a legitimate CDN/publisher mechanism supports it — never TLS interception.

---

## 4. What "done" looks like for each phase

- **After Step 1:** you can point to a dashboard and say "this domain's traffic dropped from X Mbps to Y Mbps at peak."
- **After Step 2–3:** the cache never makes things *slower* than no cache, even under origin failure or a cold segment.
- **After Step 5:** losing one cache node or one gateway instance does not interrupt customer traffic.
- **After Step 6:** the bandwidth savings are visible in real time to anyone at Exord, not just inferred from bills at month-end.

Build in this order — reliability and observability (Steps 2, 6) come before horizontal scale (Step 5) on purpose: a fast, honest single-node cache with a visible dashboard is worth more right now than a distributed system nobody trusts yet.
