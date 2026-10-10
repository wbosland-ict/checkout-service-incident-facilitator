# Catch-up Cards (FACILITATOR: hand out at the start of the next exercise)

The exercises build on each other. Hand out a card at the start of the next
exercise, so groups that didn't finish (or reached a different
conclusion) can still continue. Print them, or paste them in the Teams
chat. Each card holds only what the next exercise needs, not the full model
answer.

| Card | Hand out at the start of |
|---|---|
| A: Incident outcome | Exercise 3: Problem analysis |
| B: Problem outcome | Exercise 3, Part D: Request for Change |
| C: Change outcome | Exercise 4: Fixing the source code |

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
  EF Core navigation properties (**N+1 queries**) **and** widened the DB
  transaction to include the remote inventory and payment calls.
  Connections were held ~10x longer, which exhausted the 50-connection
  pool.
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

---

## Card C: Change outcome (W 2607 012)

- **Approved RFC `W 2607 012`** for `checkout-service v2.14.1`: this is
  what Exercise 4 implements in the source code. Scope is the same as
  Card B's "In the RFC" bullet; nothing else.
- **Where the code is:** `checkout-service-incident-sourcecode/` — the
  v2.14.0 snapshot, post-PR #4821. `dotnet build && dotnet test` should
  give you **1 passed** before you change anything. The code carries no
  comments or hints about what's wrong; that's deliberate.
- **What to build:** a single-query fetch for cart items + products
  (eager loading via `.Include()`/`.ThenInclude()`), a
  transaction scope narrowed to exclude the inventory/payment remote
  calls, a
  payment client with backoff + jitter + `Retry-After` handling +
  circuit breaker + `Idempotency-Key`, and a query-count regression test
  with realistic cart sizes (4–11 items).
- **Start with the test.** Write the query-count regression test first
  and see it fail against the unchanged code, then fix. The fixtures you
  need (`TestDbContextFactory`, `QueryCountingInterceptor`,
  `CartFixtures`, and `FakeHttpMessageHandler`/`DownstreamFakes` for the
  inventory and payment-gateway calls) are already in the test project.
- **Out of scope here:** pool resizing, CI load-test stage, front-end
  retry alignment, and the rate-limit increase — these stay separate
  follow-up actions, not code changes in this exercise.
