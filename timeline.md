# Ground-Truth Timeline (FACILITATOR ONLY. Don't share with participants before the debrief.)

All times UTC.

## Incident phase: I 2607 041 (priority High)

| Time | Event |
|---|---|
| 2026-07-07 09:41 | PR #4821 merged (1 approval; unit/integration tests green; no performance stage in CI). |
| 09:58 | CI/CD deploys `checkout-service v2.14.0`. Changelog: "Refactored cart items retrieval to use lazy-loaded navigation properties (EF Core), removed manual eager-loading (`.Include()`) query." |
| 10:00–10:02 | New code path live. Lazy loading causes an **N+1 query problem**: per cart with *n* items, 1 query for the cart + *n* queries for products instead of 1 joined query. In addition, the transaction scope was widened to cover the whole `StartAsync()` method, so the DB connection is now **held during the inventory and payment-gateway calls**. |
| 10:02 | Average connection hold time rises from ~34ms to ~140ms (→ ~376ms by 10:20). First customer calls reach the service desk. |
| 10:03–10:04 | DB connection pool (`max_pool_size=50`) climbs from baseline ~20 to 50 (100%). |
| 10:04–10:05 | Requests queue for a free connection; waits exceed the 5000ms timeout → `NpgsqlTimeoutException`, HTTP 504. |
| 10:05 onwards | Service retries (3 attempts, fixed 200ms, also on 429) plus front-end retries (3 attempts on 502/504) multiply payment-gateway traffic to ~3.3–3.4x baseline. Rate limit (300 req/min) breached → `429 Too Many Requests`. |
| 10:06 | Grafana alert ALT-10023841 (latency/error rate, critical) posted to Teams *SRE Alerts*. |
| 10:07 | ALT-10023845 (DB pool saturation, critical). |
| 10:08 | Service desk reports call spike in Teams. |
| 10:09 | ALT-10023852 (429s, warning): downstream effect. |
| 10:10 | Payments engineer flags 429 wave in Teams *Engineering > Payments*. |
| 10:11 | ALT-10023860 (notification queue, info): **red herring**. |
| 10:12 | Service desk notes duplicate-charge questions from customers. |
| 10:16 | Service desk registers **I 2607 041**, priority High, assigns it to *SRE – Checkout Platform*. |
| 10:20 | Engineer (participant) picks up the incident. Checkout success rate 62.4%, p95 4180ms. **Exercise 1 starts here.** |
| ~10:24 | Trigger identified (v2.14.0); rollback announced in Teams *Incidents*; action logged in TopDesk. |
| 10:26 | Rollback `v2.14.0 → v2.13.4` started (`kubectl rollout undo`). Part of the incident, no RFC. |
| 10:27 | Rollout 6/12 pods; success rate 78.5%. |
| 10:28 | Rollout complete; pool 33/50, queue draining, 429s nearly gone. |
| 10:30 | p95 210ms, success rate 98.9%. |
| 10:35 | Back to baseline: p95 155ms, pool 21/50, 0 × 429, success rate 99.4%. |
| ~10:45 | I 2607 041 resolved: "Solved – workaround". Problem **P 2607 007** registered and linked. Service desk informed; status page set to resolved. |

## Problem phase: P 2607 007

| Time | Event |
|---|---|
| 2026-07-07 14:02 | Product owner: "saved carts" (v2.15.0, planned 07-21) depends on the reverted refactor. |
| 14:20 | Release manager: deploy freeze on checkout-service `main` until an approved RFC exists. |
| 15:05 | DB/infra: Postgres `max_connections=200`; raising the pool without fixing queries moves the bottleneck to DB CPU. |
| 15:30 | Payments: rate limit can be raised to 600 req/min (2 weeks, extra cost); recommends idempotency keys + backoff. |
| 2026-07-08 09:30 | Finance reconciliation: **7 duplicate charges** (€ 612.40), refunded. Caused by retrying a charge after a client-side timeout without an `Idempotency-Key`. |
| 07-08 | Engineer picks up the problem (**Exercise 3, Parts A–C**): root cause, contributing factors, known error, solution options. Status → *Known error* → *Solution proposed*. |

## Change phase: W 2607 012

| Time | Event |
|---|---|
| 2026-07-09 | Engineer submits RFC `W 2607 012` (**Exercise 3, Part D**) for `checkout-service v2.14.1`, linked to P 2607 007 and I 2607 041. |
| target 07-16 | Desired completion: before "saved carts" (07-21) and the back-to-school campaign (August). |
| TBD | Engineer implements the approved change in the source code
  (**Exercise 4**, against `checkout-service-incident-sourcecode/`). |

## Root cause (single sentence)

`checkout-service v2.14.0` replaced a single joined query with lazy-loaded
EF Core navigation properties (an N+1 query pattern) and widened the DB transaction to
include remote inventory and payment calls. This made connections be held
~10x longer, which exhausted the 50-connection pool under normal traffic.
The resulting timeouts triggered stacked retries that breached
payment-gateway's rate limit and collapsed the checkout success rate.

## Contributing factors

1. **No performance/query-count safeguard** in CI; integration test carts
   have 1–2 items vs. a production average of 4.6 (p95 11), so the N+1
   pattern was invisible in tests.
2. **Transaction scope**: remote calls inside a DB transaction (introduced
   to make lazy loading work with `lazy_loading_proxies_enabled: true`).
3. **Pool sizing**: `max_pool_size=50` sized for ~350 req/min at the
   2026-04 capacity review; traffic now ~460 req/min (+40%) and forecast
   ~700 req/min for Black Friday (41/50 connections even on v2.13.4).
4. **Retry storm**: fixed 200ms backoff, no jitter, retries on 429, no
   circuit breaker, and front-end retries stacking on top of service
   retries.
5. **No idempotency key** on charge requests → duplicate charges when a
   timed-out charge is retried.
6. **Runbook gap**: no section for DB pool saturation or retry storms.

## Why the rollback is a workaround (not a solution)

- It removes the refactor that the next release ("saved carts") depends
  on → deploy freeze, roadmap blocked.
- The weaknesses that turned a slowdown into an outage (retry policy, no
  circuit breaker, no idempotency, tight pool headroom) are all still
  present on v2.13.4. Any future slowdown (DB, inventory, or traffic peak)
  can trigger the same retry storm and duplicate charges.
