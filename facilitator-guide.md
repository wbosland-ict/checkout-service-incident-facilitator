# Facilitator Guide

This guide is for whoever runs the workshop. Do not share this file or
`scenario/02-timeline.md` with participants before the debrief at the end
of each exercise — they contain the ground truth.

## Overall timing (≈2.5 hours)

| Segment | Time |
|---|---|
| Intro & objectives | 10 min |
| Hand out incident brief + architecture | 5 min |
| Exercise 1: Triage | 25 min + 10 min debrief |
| Exercise 2: Root cause analysis | 35 min + 10 min debrief |
| Break | 10 min |
| Exercise 3: Remediation | 30 min + 10 min debrief |
| Exercise 4: Postmortem & comms | 30 min + 10 min debrief |
| Closing discussion: where AI helps vs. doesn't | 15 min |

Adjust down (~90 min) by trimming debriefs to 5 min and dropping the
closing discussion to 5 min, if needed.

## Ground truth (summary — full detail in `scenario/02-timeline.md`)

- **Root cause:** `checkout-service v2.14.0` (deployed 09:58 UTC) replaced
  a manual `JOIN FETCH` query with lazy-loaded ORM associations, creating
  an N+1 query pattern. This increased average DB connection hold time,
  exhausting the 50-connection pool by ~10:04. Resulting timeouts (504s)
  triggered retries that pushed payment-gateway call volume to ~3.3-3.4x
  baseline, breaching its 300 req/min rate limit and causing 429s. Net
  effect: checkout success rate fell from ~99.5% to ~62%.
- **Correct fix:** Roll back `checkout-service` to `v2.13.4`.
- **Distractor alert:** `order-notification-service` queue depth alert
  (PD-10023860) is a downstream symptom (fewer completed orders → fewer
  notifications queued... actually inverted: queue depth rising suggests
  a *different* minor issue — treat it explicitly as a **red herring**
  during discussion: it's real but not causally connected to the primary
  incident, included to test whether participants/AI correctly deprioritize
  unrelated noise during triage).
- **Contributing factors to surface:** no perf/query-plan review gate for
  ORM changes in CI; DB pool sized before ~40% traffic growth; no
  backoff/jitter or circuit breaker on retries; runbook has no section for
  DB pool saturation.

## Exercise-by-exercise facilitator notes

### Exercise 1 — Triage

**Expected good outcome:** Participants (with AI help) correctly identify
alerts PD-10023841 (latency/errors) and PD-10023845 (DB pool saturation)
as the primary signals, note PD-10023852 (429s) as a very likely
downstream effect, and flag PD-10023860 (notification queue) as probably
unrelated/lower priority.

**Discussion prompts:**
- Did the AI need to be told the notification-queue alert was a red
  herring, or did it figure that out from the "note" field alone?
- How did people decide *how much* raw log/metric data to paste into the
  AI vs. summarizing first? (There's a real tradeoff between context and
  noise/token limits.)

### Exercise 2 — Root cause analysis

**Expected good outcome:** Deploy `v2.14.0` at 09:58 identified as trigger;
N+1 query explained; causal chain deploy → DB pool exhaustion → timeouts
→ retries → payment-gateway rate limiting → checkout failures, fully
articulated with supporting log lines/metrics at each link.

**Discussion prompts:**
- Ask a group to share their root-cause sentence verbatim — does it match
  the ground truth in spirit? Did the AI ever assert a cause not supported
  by evidence (e.g., blaming payment-gateway itself, which the data
  explicitly rules out via the "no incidents" status note)?
- This is the best exercise to demonstrate **prompting for alternative
  hypotheses** — have a group try asking "could this instead be a
  payment-gateway-side issue?" and see how the AI responds using the
  evidence (payment-gateway logs explicitly show it's healthy and
  client-driven).

### Exercise 3 — Remediation

**Expected good outcome:** Rollback to v2.13.4 chosen as fastest-safe
option over a forward hotfix (higher risk, slower) or just bumping pool
size (masks the symptom, doesn't fix root cause, and query load would
keep growing). Good verification checklist references p95 latency,
DB active connections, and checkout success rate returning to baseline
within minutes of rollback completing.

**Discussion prompts:**
- Did any group's AI suggest platform-specific commands that don't match
  a real environment people use at work? Good moment to discuss verifying
  AI output against your actual tooling before running anything.
- Compare the AI-drafted new runbook section across groups — what did
  people keep vs. cut? Any two answers being different is fine and a good
  discussion point about mentorship/judgment role of the human.

### Exercise 4 — Postmortem & comms

**Expected good outcome:** Blameless framing (e.g., "the deploy pipeline
lacked an automated check for query-plan regressions" rather than "a
developer wrote a bad query"), specific action items (e.g., "Add
slow-query/APM threshold check to CI pipeline for checkout-service — owner:
Platform team, target: next sprint" rather than "improve testing"),
appropriately different tone/detail across internal vs. customer vs.
leadership audiences.

**Discussion prompts:**
- Read a couple of AI-drafted action items aloud — are they specific and
  assigned, or generic? Practice tightening one together as a group.
- Compare the customer-facing update to the internal postmortem — did
  the AI ever leak internal jargon or overly technical detail into the
  customer version? Did it ever overpromise ("this will never happen
  again")?

## Closing discussion: where AI helps vs. doesn't

Suggested talking points:

- **Helps:** rapid summarization of noisy data, drafting under time
  pressure (Slack updates, postmortems), explaining unfamiliar technical
  concepts on the fly, generating alternative hypotheses you might not
  think of, tightening vague writing.
- **Doesn't replace:** verifying facts/timestamps against real evidence,
  final judgment calls on risk (e.g., which remediation to run in prod),
  ownership and accountability for the incident and its writeup,
  understanding your org's actual tooling/environment/approval process,
  data-handling/privacy judgment about what's safe to paste into an
  external AI tool.
- **Common pitfalls to name explicitly:** confirmation bias (leading
  questions get agreeable answers), hallucinated specifics (fabricated
  timestamps/numbers — always verify), over-trusting a single AI-generated
  root cause without checking the full causal chain, generic/vague action
  items unless explicitly pushed for specificity.

## Adapting this scenario

- Swap in your own team's real (sanitized) architecture, alert format, or
  runbook style to make it more relevant — the underlying failure pattern
  (bad deploy → resource exhaustion → retry storm → downstream rate
  limiting) is a common, transferable one worth keeping.
- For a shorter session, drop Exercise 4 or merge it into a 15-minute
  wrap-up.
- For a more advanced group, remove the ground-truth timeline entirely and
  have them work purely from `data/`, or introduce a second, unrelated red
  herring to sharpen prioritization skills further.
