# Ground-Truth Timeline (FACILITATOR ONLY — do not share with participants before the debrief)

All times UTC.

| Time | Event |
|---|---|
| 09:58 | CI/CD deploys `checkout-service v2.14.0`. Changelog: "Refactored cart items retrieval to use lazy-loaded associations (ORM), removed manual `JOIN FETCH` query." |
| 10:00–10:02 | New code path in production. Lazy loading causes an **N+1 query problem**: for each cart with *n* items, the service now issues 1 query for the cart + *n* additional queries for each item's product details, instead of 1 combined query. |
| 10:02 | Average query time for cart retrieval rises from ~8ms to ~140ms under load. Individual DB connections are held longer per request. |
| 10:03–10:05 | DB connection pool (`max_pool_size=50`) usage climbs from baseline ~20 to 50 (100%) as more concurrent requests hold connections longer than they're released. |
| 10:05 | New requests start queueing for a free DB connection; wait times exceed the configured 5000ms acquisition timeout. checkout-service starts throwing `ConnectionPoolTimeoutException`. |
| 10:05–10:07 | p95 latency on checkout-service spikes from ~150ms to 2000ms+, then continues climbing. 5xx rate crosses 5%. **Alert 1 fires (high latency & error rate)**. |
| 10:07 | DB connection pool alert fires (**Alert 2**, saturation). |
| 10:06–10:10 | Client-side and checkout-service retry logic (3 retries, 200ms backoff) on the resulting 5xx/timeouts multiplies the effective request volume to payment-gateway roughly 3-4x. |
| 10:08 | payment-gateway request rate from checkout-service exceeds its published rate limit (300 req/min); payment-gateway starts returning `429 Too Many Requests`. |
| 10:08–10:20 | Checkout success rate collapses from ~99.5% to ~62%. Customer-visible impact: stuck "Processing..." states, generic error pages, some duplicate-charge risk flagged by #eng-payments (retries against payment-gateway after a timeout, before confirmation is received — **latent secondary risk**, no confirmed duplicate charges yet at time of paging). |
| 10:20 | (For the exercise) This is roughly "now" — the point at which the on-call SRE has been paged and starts investigating. |

## Root cause (single sentence)

A deploy (`checkout-service v2.14.0`) introduced an N+1 query pattern that
increased average DB connection hold time, exhausting the DB connection
pool under normal traffic; the resulting timeouts triggered aggressive
client/service retries that in turn breached payment-gateway's rate limit,
collapsing checkout success rate.

## Contributing factors worth surfacing in discussion

1. **No query-plan/perf review gate** in the deploy pipeline for ORM changes.
2. **Connection pool sizing** (`max_pool_size=50`) was last reviewed before
   a ~40% traffic growth — it was already running close to the edge.
2. **Retry storm**: retries without backoff/jitter or a circuit breaker
   amplified load onto a healthy downstream (payment-gateway) once
   checkout-service degraded.
3. **Runbook gap**: the existing runbook (`data/runbooks/checkout-service-runbook.md`)
   has no section for DB connection pool saturation or recent-deploy
   correlation — it's a good exercise-3/4 discussion point.

## Correct remediation (fastest safe fix)

**Rollback `checkout-service` to `v2.13.x`** (removes the N+1 query
pattern). This is faster and lower-risk than a hotfix forward-deploy or a
live connection-pool resize, and directly reverses the root cause.
Secondary/longer-term follow-ups: add a slow-query/APM check to CI,
increase pool size with headroom for traffic growth, add retry
backoff+jitter and a circuit breaker in front of payment-gateway calls.
