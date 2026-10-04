# Example Problem Record — P 2607 007 (FACILITATOR MODEL ANSWER)

| Field | Value |
|---|---|
| Number | P 2607 007 |
| Status | Solution proposed |
| Brief description | checkout-service v2.14.0 exhausts DB connection pool; retries cause payment-gateway rate limiting and duplicate charges |
| Category / Subcategory | Webshop / Checkout |
| Object / Asset | checkout-service (production) |
| Impact | Organisation-wide (all checkout traffic) |
| Urgency | Medium (workaround in place; release blocked; Black Friday risk) |
| Priority | High |
| Operator group | SRE – Checkout Platform |
| Operator | (participant) |
| Linked incident(s) | I 2607 041 |
| Registered | 2026-07-07 10:45 UTC |

## Problem description

On 2026-07-07 checkout-service v2.14.0 caused checkout success rate to drop
from ~99.5% to ~62% between 10:00 and 10:35 UTC (incident I 2607 041).
About 5,760 checkouts failed or were abandoned, and 7 customers were
charged twice (€ 612.40, refunded). Service was restored by rolling back to
v2.13.4. The underlying causes have not been removed.

## Workaround (currently in place)

checkout-service rolled back to v2.13.4. Limitations:
- The cart-retrieval refactor needed for "saved carts" (v2.15.0, planned
  2026-07-21) is reverted; `main` is under a deploy freeze.
- Retry policy, missing circuit breaker, missing idempotency key and tight
  pool headroom are unchanged. Any slowdown (DB, inventory, traffic peak)
  can trigger the same retry storm and duplicate charges.
- At the Black Friday forecast (~700 req/min) v2.13.4 is expected to use
  ~41/50 pool connections, leaving little headroom.

## Analysis

### Root cause

v2.14.0 replaced a single joined query with lazy-loaded ORM associations
(N+1 queries) and widened the DB transaction to cover remote inventory
and payment calls. Average connection hold time rose from ~34ms to
~376ms, which exhausted the 50-connection pool under normal traffic.

### Causal chain

| # | Link | Evidence |
|---|---|---|
| 1 | v2.14.0 deployed 09:58 | deploy-history.md; checkout-service.log 09:58:02–09:58:47 |
| 2 | N+1 queries: 1 + n queries per cart | checkout-service.log: before deploy (v2.13.4, 09:40–09:57), every checkout runs 1 joined query regardless of cart size (reqs 8c05, 8c31, 8c77 with 11 items, 8ca4, 8e40); after deploy, req 8e73 runs 1 cart + 1 cart_item + 4 product queries, req 8f02 runs 6 product queries, req 8f47 runs 9 product queries (`SELECT p.* FROM product p WHERE p.id = <id>`, a different product ID each time); pr-4821-diff.md (eager-loading `.Include()` removed, `virtual` lazy-loaded navigation property added) |
| 3 | Connection held during remote calls | pr-4821-diff.md: transaction scope widened around `StartAsync()`, wrapping `inventoryClient.Reserve` and `paymentClient.Charge` |
| 4 | Hold time ↑ → pool exhausted | db_connection_pool.csv: hold 34→376ms, active 21→50 by 10:04, queue up to 41 |
| 5 | Pool timeouts → 504s | log 10:04:10 `NpgsqlTimeoutException`, `status=504 duration_ms=5008` |
| 6 | Stacked retries multiply traffic | config: service retries 3× fixed 200ms incl. 429, front-end retries 3× on 502/504; log `retry.scheduled`; payment_gateway_calls.csv 181→~600 req/min (3.3–3.4x) with flat inbound traffic (~460 req/min) |
| 7 | Rate limit breached → 429s | payment-gateway.log "rate limit exceeded: 300 req/min"; 85–91 429s/min |
| 8 | Checkout failures | success rate 99.5% → 62.4% (payment_gateway_calls.csv, log 10:20:00) |
| 9 | Duplicate charges | timed-out charge retried without `Idempotency-Key` (config `idempotency_key.enabled: false`; payment-reconciliation.md: 7 duplicates) |

### Contributing factors

1. No performance / query-count check in CI; test carts have 1–2 items vs.
   production average 4.6 (p95 11).
2. Transaction scope widened to make lazy loading work while keeping
   `lazy_loading_proxies_enabled: true`; not spotted in single-reviewer code review.
3. Pool sized at 50 for ~350 req/min (April); traffic now ~460 req/min.
4. Retry policy: fixed backoff, no jitter, retries on 429, no circuit
   breaker; front-end retries on top of service retries.
5. No idempotency key on payment charges.
6. Runbook has no section for DB pool saturation / retry storms.

### Alternative hypotheses considered and ruled out

| Hypothesis | Evidence |
|---|---|
| payment-gateway outage | Their status page is green; payment-gateway.log: "elevated 429s are client-driven"; 429 response times 6–9ms (healthy); successful charges take 71–88ms both before and after the deploy |
| Traffic spike | Inbound req/min flat (418 → ~460); only outbound to payment-gateway rose |
| DB infrastructure issue | No schema/config changes in 30 days; DB `max_connections=200` with headroom; issue began exactly at deploy and stopped on rollback |
| inventory-service | No inventory errors in logs; no alerts |
| Notification queue | Alert started 10:11, after impact; unrelated / downstream |

## Known error

Checkout stuck on "Processing..." or generic error, with DB pool saturation
and payment-gateway 429s, after a checkout-service deploy that changes
how cart data is loaded. Workaround: roll back checkout-service to the
previous version (runbook "Rollback procedure"). Check for duplicate
charges with Finance.

## Solution options

| Option | Description | Pros | Cons / risks | Effort |
|---|---|---|---|---|
| A | Re-release refactor with eager loading (`.Include()`/`.ThenInclude()`) + narrow transaction (read-only for cart, remote calls outside) | Removes root cause; unblocks saved carts | Needs careful testing of lazy-loading edge cases | M |
| B | Payment client resilience: exp. backoff + jitter, honour `Retry-After`, max 1 retry on 429, circuit breaker, `Idempotency-Key` | Prevents retry storms and duplicate charges for *any* slowdown | Behaviour change on payment path; needs payments review | M |
| C | Query-count regression test with realistic carts (4–11 items) | Catches the same class of defect before production | Test maintenance | S |
| D | Increase pool 50 → 80 | Headroom for growth | Masks inefficiency; moves bottleneck to DB CPU; needs DB/infra capacity review | S |
| E | Raise payment-gateway limit to 600 req/min | Headroom | Extra cost; 2-week lead time; amplifies retry storms instead of preventing them | S |
| F | CI load-test stage with production-like data | Broad protection | Platform-team work; larger effort | L |

## Proposed solution

**RFC W 2607 012** for checkout-service v2.14.1: options **A + B + C**,
one deployable unit owned by the Checkout Platform Team.

Separate follow-up actions (not in this RFC):

| Action | Owner | Target date | TopDesk reference |
|---|---|---|---|
| Capacity review of DB pool size vs. traffic forecast (option D) | DB/Infra team | 2026-07-31 (before back-to-school) | separate change |
| CI load-test stage with production-like data (option F) | Platform team | 2026-09-30 | separate change |
| Align front-end retry policy with service retries | Web team | 2026-08-15 | separate change |
| Add "DB pool saturation / retry storm" section to runbook | SRE – Checkout Platform | 2026-07-17 | action on P 2607 007 |
| Rate-limit increase (option E): **not pursued** after B is implemented | Payments | n/a | decision logged |
