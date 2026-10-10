# Facilitator Guide

This guide is for whoever runs the workshop. Don't share this file,
`timeline.md`, `example-problem-record.md` or
`example-rfc-checkout-service.docx` with participants before the debrief
of the relevant exercise. They contain the model answers.

## The process the workshop follows

| # | Step | Who | TopDesk | Exercise |
|---|---|---|---|---|
| 1 | Register the incident and assign it | Service desk | Incident | Incident brief |
| 2 | Triage: impact and priority | Service engineer | Incident | 1 |
| 3 | Restore service (a workaround is allowed) and close the incident | Service engineer | Incident | 2 |
| 4 | If a long-term fix is needed: register a problem | Service engineer | Problem | 2 |
| 5 | Analyse the problem thoroughly, choose a solution | Service engineer | Problem | 3 (Parts A–C) |
| 6 | Submit a Request for Change (`rfc-template.docx`) | Service engineer | Change | 3 (Part D) |
| 7 | Implement the approved change in the source code | Service engineer | Change | 4 |

Key rules to emphasise:

- A **rollback is part of a high priority incident**. It needs no RFC,
  but it must be announced in the Teams *Incidents* channel and logged in
  the TopDesk incident.
- **Solved ≠ fixed.** Closing an incident with a workaround is fine, as
  long as a problem is registered for the underlying cause.
- The RFC is for the **long-term solution** that comes out of the problem
  analysis.

## Overall timing (≈3¼ hours for Exercises 1–3; Exercise 4 has a runnable sample repo but is not yet timed)

| Segment | Time |
|---|---|
| Intro: objectives + incident → problem → change flow | 10 min |
| Hand out incident brief, architecture, evidence | 5 min |
| Exercise 1: Incident intake & triage | 25 min + 10 min debrief |
| Exercise 2: Incident resolution | 40 min + 10 min debrief |
| Break | 10 min |
| Exercise 3: Problem analysis & Change request | 80 min + 10 min debrief |
| Exercise 4: Fixing the source code with AI (sample repo + model answer ready; timing still TBD) | TBD |
| Closing discussion: where AI helps vs. doesn't | 10 min |
| **Total (Exercises 1–3)** | **~195 min** |

**Catch-up cards:** hand out `catch-up-cards.md` (Card A before
Exercise 3, Card B before Part D of Exercise 3, Card C before
Exercise 4) so groups that fell behind can continue.

**Pacing within exercises** (facilitator only; the exercise files show
just the overall time box):

- Exercise 2: Part A ~10 min, Part B ~15 min, Part C ~15 min
- Exercise 3: Part A ~15 min, Part B ~10 min, Part C ~15 min, Part D
  (Request for Change) ~35 min

## Ground truth (summary; full detail in `timeline.md`)

- **Trigger:** `checkout-service v2.14.0`, deployed 09:58 UTC.
- **Root cause:** N+1 query pattern (lazy-loaded EF Core navigation
  properties replacing eager loading via `.Include()`) **plus** a widened
  transaction scope that holds the DB connection during remote
  inventory/payment calls. Avg connection hold
  time went from ~34ms to ~376ms, exhausting the 50-connection pool by
  ~10:04. Timeouts (504) → stacked service + front-end retries →
  payment-gateway traffic ~3.3–3.4x baseline → 429s above 300 req/min.
  Checkout success rate fell from ~99.5% to ~62%.
- **Incident resolution:** roll back to `v2.13.4` (10:26–10:28); baseline
  restored by 10:35. Closure code *Solved – workaround*; problem
  **P 2607 007** registered.
- **Red herring:** `order-notification-service` queue depth alert
  (ALT-10023860). It's real but not causally connected; it tests whether
  participants and the AI deprioritise unrelated noise.
- **Long-term solution (model answer):** RFC **W 2607 012** for
  `checkout-service v2.14.1`: re-introduce the refactor with a single
  fetch (eager loading), narrow the transaction scope, harden the
  payment client (exponential backoff + jitter, honour `Retry-After`,
  circuit breaker, `Idempotency-Key`), and add a query-count regression
  test with realistic cart sizes. Pool capacity review, CI load-test
  stage and front-end retry alignment are **separate follow-up actions**.

## Exercise-by-exercise notes

### Exercise 1: Incident intake & triage

**Expected good outcome:** ALT-10023841 (latency/errors) and ALT-10023845
(DB pool saturation) identified as primary signals; ALT-10023852 (429s) as
a very likely downstream effect; ALT-10023860 (notification queue) as
unrelated / low priority. Customer impact quantified (success rate 62.4%,
p95 4180ms, ~38% error rate, ~460 req/min affected). Priority *High*
confirmed (organisation-wide impact, revenue loss ongoing). A good
participant also notices the **duplicate-charge concern** in the ticket and
flags it in the action entry, because it adds financial/customer risk.

**Discussion prompts:**
- *Alert ranking (task 1):* Did the AI rank the notification-queue alert
  as noise by itself, or only after you pointed it at the "note" field?
  Did it treat the 429s as a cause or as a downstream effect?
- *Classification (task 2):* Did anyone's AI suggest a *different*
  priority? What reasoning did it use, and does it match how our service
  desk classifies incidents? Did it notice that the duplicate-charge
  concern is missing from the registration?
- *Staying in triage (task 3):* Did the AI jump to a root cause ("it's
  the database") instead of suggesting next evidence to look at? How did
  you keep it focused on impact and next steps?
- *Drafts (task 4):* Read one AI-drafted TopDesk action entry and Teams
  post aloud. What did you change before you would post it (numbers,
  length, tone)?
- What would you never paste from a real TopDesk ticket into an external
  AI tool (caller names, phone numbers, email addresses)?

### Exercise 2: Incident resolution

**Expected good outcome:**
- v2.14.0 at 09:58 identified as the trigger with high confidence
  (onset 10:00, no other changes, payment-gateway healthy per its logs
  and status page).
- Rollback to v2.13.4 chosen over a forward hotfix (slower, riskier
  under pressure) or a pool increase (masks the symptom; the DB/infra
  note in Exercise 3 confirms it would move the bottleneck).
- Teams announcement **before** acting; TopDesk action entries with
  timestamps.
- Recovery checked with `post-rollback-recovery.csv`: p95, pool usage,
  queued requests, 429 count and success rate all back to baseline
  within ~7–9 minutes of starting the rollback, matching the rollout
  progress (6/12 pods → partial recovery). That timing is what separates
  causation from a coincidental recovery.
- Incident closed as **solved by workaround**, with a problem registered
  and linked. Reasons: refactor reverted (next release blocked),
  retry/idempotency/pool weaknesses still present, 7 customers possibly
  double-charged (confirmed next day).

**Model TopDesk resolution text:**
> Service restored at 10:35 UTC by rolling back checkout-service from
> v2.14.0 to v2.13.4 (10:26–10:28 UTC). Cause: v2.14.0 increased DB
> connection usage, exhausting the connection pool; resulting retries
> triggered rate limiting at payment-gateway. Checkout success rate back
> to 99.4%. This is a workaround: the change is reverted but the
> underlying weaknesses remain. Problem P 2607 007 registered for the
> long-term solution, including possible duplicate charges.

**Discussion prompts:**
- *Find the trigger (Part A):* Did you let the AI find the files itself,
  or did you point it at them? How confident was it about v2.14.0, and
  what evidence did it use to rule out a payment-gateway problem?
- *Restore options (Part B):* Did any AI propose filing an (emergency)
  RFC before rolling back, or recommend raising the pool size as the
  quickest fix? In our process a rollback is part of a high priority
  incident. Good moment to discuss adapting AI advice to *your* process.
- *Commands (Part B):* Did the rollback commands match the runbook, or
  did the AI invent commands or tools that don't match your real
  environment?
- *Announce before you act (Part B):* Did your Teams post go out
  *before* the rollback? What did the AI put in it, and what did you
  remove?
- *Verification (Part B):* How did you show that the rollback *caused*
  the recovery rather than just coinciding with it? Did anyone link the
  partial recovery to 6/12 pods being rolled out?
- *Workaround or fix (Part C):* Who decided "workaround" vs. "permanent
  fix", the AI or the participant? What tipped it? Compare two groups'
  problem descriptions: would the engineer who picks up the problem
  tomorrow understand them?
- *Messages (Part C):* Read a service desk message and a status-page
  update aloud. Any jargon, blame or overpromising left in?

### Exercise 3: Problem analysis & Change request

**Expected good outcome (Parts A–C):** See `example-problem-record.md`.
Look for:
- The N+1 explanation **and** the transaction-scope finding in the diff
  (the less obvious second cause; it explains why hold time rose so
  much). Groups that only find N+1 have an incomplete analysis.
- The logs don't name the N+1 pattern; participants (or their AI) must
  spot it by comparing requests. Before the deploy (09:40–09:57, pods
  `checkout-7f9c-*` on v2.13.4), every checkout runs **one** joined query,
  whatever the cart size: reqs `8c05` (3 items), `8ca4` (6), `8e40` (5),
  `8c31` (9) and even `8c77` (11 items, 8 ms). Pool stats are stable at
  20–21 active connections with ~33 ms hold time. After the deploy,
  `8e73`, `8f02` and `8f47` run one
  `SELECT p.* FROM product p WHERE p.id = <id>` per cart item (4, 6 and 9
  times). Ask in the debrief whether the AI spotted this by itself.
- A causal chain where each link has evidence (log line, metric, code,
  config).
- Alternative hypotheses explicitly ruled out (payment-gateway outage:
  their logs say healthy and client-driven; traffic spike: req/min
  roughly flat 418→460; DB infra: no config/schema changes, DB has
  headroom).
- Contributing factors that include test-data realism, retry config,
  missing idempotency key, pool headroom vs. forecast, runbook gap.
- **Runbook review (Part B, step 4):** the runbook gives no hint about
  its gaps; participants (or their AI) must find them by comparing it
  with the incident. Expected findings:
  - No section for DB connection pool saturation (how to check pool
    usage, whether and how to adjust `max_pool_size` safely), even though
    the escalation table already lists DB pool saturation as a reason to
    call DB/Infra.
  - No guidance on retry storms and their effect on downstream
    dependencies such as payment-gateway (429s).
  - The latency alert procedure suggests restarting pods, which wouldn't
    help here, and doesn't mention checking the DB pool.
  - No guidance on checking for duplicate charges after payment errors.
  - Overdue for review (next review was due 2026-05-14), and it predates
    the traffic growth since the last capacity review.
  Ask in the debrief whether the AI found these without being told
  where to look.
- A reasoned solution choice with clear RFC scope vs. follow-up actions.

**Expected good outcome (Part D, RFC):** See
`example-rfc-checkout-service.docx`. Look for:
- General info filled in with correct references (P 2607 007, related
  incident I 2607 041) and **no invented data** presented as fact.
- Section 1 traceable to evidence (impact numbers, duplicate charges,
  deploy freeze).
- Clear split between section 2 (WHAT) and section 3 (HOW).
- Concrete risks with mitigations, a specific rollback plan (back to
  v2.13.4 via `kubectl rollout undo`, < 5 min, decision criteria), post-
  deploy verification thresholds, and a deploy window outside peak hours.
- A realistic WBS that includes a load test with production-like carts
  and a DB/infra review.
- Deliberate scope: pool resize, CI load-test stage, front-end retries and
  the rate-limit increase are **not** in this RFC's WBS but are referenced
  as separate actions.

**Discussion prompts (Parts A–C):**
- *Root cause (Part A):* Did the AI spot the N+1 pattern in the logged
  queries by itself, or only after you pointed it there? Did any group
  find the widened transaction-scope change, the second cause, without
  help?
- *Root-cause statement (Part A):* Ask a group to read out their
  root-cause sentence. Is every claim backed by a log line, metric or
  code line? Does it blame a person or describe a system gap?
- *Alternative hypotheses (Part A):* This is the best exercise for
  **prompting for alternatives**. Have a group ask "could this be a
  payment-gateway issue instead?" live and see how the AI handles the
  evidence.
- *Workaround risk (Part B):* How did the AI handle the Black Friday
  forecast? Did it work out that 41/50 connections leaves little headroom
  even on v2.13.4?
- *Known error and runbook (Part B):* Would a service desk colleague
  recognise the known-error description from what customers report?
  Which runbook gaps did the AI find by itself, and which did it miss?
  What did you change in the AI's runbook section before accepting it?
- *Solution and scope (Part C):* Did anyone's AI recommend "just increase
  the pool" or "ask for a higher rate limit" as *the* solution? Why is
  that a symptom fix? Compare two groups: what did they put in the RFC
  and what became a follow-up action?

**Discussion prompts (Part D, RFC):**
- *Drafting (step 1):* Did anyone have the AI fill the `.docx` directly?
  How well did it keep the layout, and what did you have to fix by hand?
  What did the AI invent (RFC numbers, dates, names, hours, test
  results)? How did you spot it?
- *Scope (step 2):* Which items did you deliberately leave **out** of the
  RFC, and where are they tracked? Did the AI try to put the pool resize,
  the CI load-test stage or the front-end retries in the WBS?
- *Risks and rollback (step 3):* Show two groups' risk sections side by
  side. Which would a CAB approve? Could a colleague on call at night
  carry out the rollback plan as written?
- *WBS (step 4):* Which of the AI's hour estimates did you change, and
  why? Did it include a load test with production-like carts and time
  for rework?
- *CAB review (step 5):* Run the "sceptical CAB member" prompt live on
  one RFC and discuss the questions it raises. Would you approve it?

### Exercise 4: Fixing the source code with AI

The sample source repo is at `checkout-service-incident-sourcecode/`
(EF Core, C#; reproduces the PR #4821 bug — buggy `FindAsync` +
lazy-loaded `CartItem.Product`, a transaction scope wrapping the
inventory/payment calls in `CheckoutService.StartAsync`, and a
`PaymentClient` implementing exactly the broken retry policy from
`checkout-service-config.yaml`). The inventory and payment clients are
real typed `HttpClient`s; in the tests they talk to fake downstream
services (`FakeHttpMessageHandler`, `DownstreamFakes`), and `OrderWriter`
persists the order through EF Core. It builds and tests clean with
`dotnet build` / `dotnet test` (**1 passed** — no comments or notes in
the code point out what's wrong).

A model-answer diff is available at `example-fix-diff.md`, covering RFC
options A (eager loading + narrowed transaction), B (hardened payment
client), and C (the query-count regression test), with notes on where a
participant's AI-assisted diff can reasonably differ. After the fix the
suite shows **4 passed**.

**Expected good outcome:**

- A regression test written *first* and demonstrably **failing** against
  the unchanged code (the buggy code issues ~6 queries for a 4-item cart
  and ~10 for 8 items), then passing after the fix. Groups that write the
  test after the fix have no evidence it catches anything.
- `CartRepository` fetching cart items and their products in one query
  (`.Include()`/`.ThenInclude()`, a split query, or a projection — all
  defensible).
- A transaction in `CheckoutService.StartAsync` that no longer spans the
  inventory and payment-gateway calls.
- A payment client that backs off exponentially with jitter, only retries
  a 429 when `Retry-After` is present, trips a circuit breaker, and
  threads an idempotency key generated **per checkout**, not per attempt.
- A diff scoped to the RFC: no pool resizing, no CI changes, no
  opportunistic refactors.
- A participant who can explain every line without re-reading the AI's
  summary.

**Discussion prompts:**

- Who wrote the test first, and who let the AI write test and fix in one
  go? Did the second group ever see their test fail? What would it have
  taken to notice it was asserting nothing?
- The payment retry policy was in the config file all along, and the
  incident timeline shows what it caused. Did your AI connect those two
  on its own, or only after you pointed at them?
- Where did the AI exceed the RFC scope? Who caught it, and how — by
  reading the diff, or because something broke?
- The payment client talks HTTP to a fake payment-gateway in the tests
  (`FakeHttpMessageHandler`), so the new retry logic *can* be exercised.
  Did anyone write tests for a 429 with and without `Retry-After`, or a
  timeout followed by a retry with the same `Idempotency-Key`? If not, the
  hardening is untested. What would you want before shipping it?
- Compare two groups' `CartRepository` fixes. Joined query, split query,
  or projection — can each group defend their choice on round trips vs.
  payload size vs. change tracking?
- How much of the final diff would you have written faster by hand? Where
  did the AI genuinely save time?

**Still to do before running this exercise:**
- Pilot end-to-end with an AI coding assistant and fill in the timing row
  in "Overall timing" above.

## Closing discussion: where AI helps vs. doesn't

- **Helps:** quickly summarising noisy data, drafting under time pressure
  (Teams posts, TopDesk entries, RFC sections), explaining unfamiliar
  technical concepts, generating alternative hypotheses, turning analysis
  into structured documents, critical review ("act as a CAB member").
- **Doesn't replace:** checking facts and numbers against real evidence;
  judgement on risk (which fix to run in production, what goes in an RFC);
  ownership of the incident, problem and change; knowing your organisation's
  process (e.g. rollback without RFC during a high priority incident);
  judgement on what data is safe to paste into an external AI tool.
- **Common pitfalls:** confirmation bias (leading questions get agreeable
  answers), invented specifics (made-up timestamps, numbers, ticket
  numbers), stopping at the first plausible cause, vague risks and
  actions, scope creep in RFCs.

## Adapting this scenario

- Swap in your own (sanitised) TopDesk exports, Teams conventions, runbook
  style and RFC template to make it even more recognisable.
- For a shorter session, run Exercises 1+2 as one combined "incident"
  exercise.
- For a more advanced group, hold back `problem-evidence/`
  and make participants request specific evidence ("I'd need the PR diff
  and the retry config"), handing out files only when asked.
