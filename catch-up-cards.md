# Catch-up Cards (FACILITATOR: hand out at the start of the next exercise)

The exercises build on each other. Hand out a card at the start of the next
exercise, so groups that didn't finish (or reached a different
conclusion) can still continue. Print them, or paste them in the Teams
chat. Each card holds only what the next exercise needs, not the full model
answer.

| Card | Hand out at the start of |
|---|---|
| A: Incident outcome | Exercise 3: Problem analysis |
| B: Problem outcome | Exercise 4: Request for Change |

---

## Card A: Incident outcome (I 2607 041)

- **Impact:** checkout success rate fell from ~99.5% to ~62% between
  10:00 and 10:35 UTC; p95 latency up to ~4.2 s; customers reported stuck
  checkouts and possible duplicate charges.
- **Primary signals:** latency/error-rate alert and DB pool saturation
  alert. The 429 alert is a downstream effect; the notification-queue
  alert is unrelated noise.
- **Trigger:** checkout-service **v2.14.0**, deployed 09:58 UTC.
  payment-gateway was healthy (its 429s were caused by our own traffic).
- **Resolution:** rolled back to **v2.13.4** (10:26–10:28 UTC) as part of
  the high priority incident; back to baseline (99.4%) by 10:35 UTC.
- **Closure:** *Solved – workaround*. Problem **P 2607 007** registered:
  the cause of the pool exhaustion is not analysed yet, and the
  v2.14.0 refactor is reverted.

---

## Card B: Problem outcome (P 2607 007)

- **Root cause:** v2.14.0 replaced a single joined query with lazy-loaded
  ORM associations (**N+1 queries**) **and** widened the DB transaction to
  include the remote inventory and payment calls. Connections were held
  ~10x longer, which exhausted the 50-connection pool.
- **Amplifiers:** fixed-interval retries (also on 429) plus front-end
  retries tripled payment-gateway traffic → rate limit; no idempotency
  key → **7 duplicate charges** (€ 612.40, refunded).
- **Why the rollback is a workaround:** the next release ("saved carts")
  depends on the refactor (deploy freeze), and the retry/idempotency
  weaknesses are still present on v2.13.4.
- **In the RFC (checkout-service v2.14.1):** single-query fetch, short
  transaction without remote calls, payment client with backoff + jitter,
  `Retry-After`, circuit breaker and idempotency key, plus a query-count
  regression test.
- **Separate follow-ups (not in the RFC):** DB pool capacity review
  (DB/Infra, by 2026-07-31), CI load-test stage (Platform team, by
  2026-09-30), front-end retry alignment (Web team, by 2026-08-15),
  runbook section (SRE, by 2026-07-17).
