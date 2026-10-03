# Prompt Library

Reusable prompt patterns for AI-assisted incident, problem and change
work, organised by TopDesk phase. Replace the bracketed placeholders with
your own (sanitised) data. These are starting points, not scripts.

## Incident: intake & triage

- "Here is a TopDesk incident and N monitoring alerts that fired within
  [time window] for [service]. Rank the alerts by how well they explain
  the customer impact in the ticket and flag any that look like
  downstream symptoms. Explain your reasoning. [paste ticket + alerts]"
- "Review the impact, urgency and priority of this TopDesk incident against
  the evidence. Is anything misclassified or missing?"
- "Given these alerts and this log excerpt, what's the smallest set of
  extra data I should look at next to confirm or rule out a cause?"
- "Summarise this log in 3 sentences for the service desk so they can
  update callers. No jargon. [paste log excerpt]"
- "Draft a TopDesk action entry (max 5 lines) for taking this incident into
  progress: current impact, what I checked, next step."

## Incident: resolution

- "Here's a deploy history and a metrics time series showing degradation
  starting at [time]. Which deploy, if any, most likely triggered this?
  How confident are you, and what would change that?"
- "Given cause X and this runbook, list my options to restore service,
  ranked by speed and risk, for a [priority] incident with [quantified
  impact]."
- "Draft the exact rollback commands for [platform, e.g.
  Kubernetes/ECS/systemd] based on this runbook's procedure."
- "Draft a Teams post for the Incidents channel announcing [action] for
  TopDesk incident [number] before I take it. Factual, under 4 sentences."
- "What should I monitor in the first 10 minutes after this fix to confirm
  it worked, and what would tell me early that it didn't?"
- "Draft TopDesk action entries and a resolution text for this incident.
  Is this resolution a workaround or a permanent fix? Explain why."
- "Draft a 2–3 sentence status-page update for [investigating / resolved].
  No internal jargon, no blame, no overpromising."

## Problem: analysis

- "Explain [concept, e.g. 'N+1 query problem' / 'connection pool
  exhaustion' / 'retry storm'] in plain terms and why it would cause
  [symptom]."
- "I think the root cause is X. List the evidence in these files that
  supports X, and separately, any evidence that contradicts it."
- "What other explanations are there for these symptoms besides [my
  hypothesis]? What data would tell them apart?"
- "Lay out the full causal chain from [trigger] to [customer impact].
  For each link, cite the log line, metric, code or config that supports
  it."
- "What contributing factors made the impact worse, and why wasn't this
  caught before production?"
- "Why is [workaround] not a permanent solution? What risks remain?"
- "Draft a known-error description for TopDesk that a service desk agent
  can recognise from customer symptoms."
- "List long-term solution options with pros, cons, risk and effort.
  Which belong in one RFC, and which should be separate actions?"
- "Draft a runbook section for [failure mode]: detection, immediate
  mitigation, escalation, matching the style of this runbook. [paste
  runbook]"

## Problem: postmortem summary

- "Turn this problem record and timeline into a one-page blameless
  postmortem: summary, impact, timeline, root cause, contributing factors,
  what went well/poorly, action items. Describe system and process gaps,
  not people. [paste problem record + timeline]"
- "Rewrite this action item to be specific, measurable and assignable:
  '[vague action item]'"
- "Rephrase this sentence to remove implied individual blame while keeping
  it factual: '[sentence]'"
- "Summarise this postmortem for [management / service desk / engineering]
  in [N] sentences, focusing on [business impact / customer symptoms /
  technical detail]."

## Change: Request for Change

- "Here is my TopDesk problem record and the sections of our RFC template.
  Draft the text for each section. Keep section 2 about WHAT changes and
  section 3 about HOW. [paste problem record + section list]"
- "Fill in this copy of rfc-template.docx with the drafted content. Keep
  the layout; put text in the empty cells under each heading."
- "Rewrite this risk section so each risk has a likelihood, impact,
  mitigation and owner. Add a concrete rollback plan and post-deploy
  verification criteria."
- "Create a work breakdown structure with hour estimates using these
  phases: design requirements, infrastructure, software development,
  testing, delivery & acceptance, rework."
- "Review this RFC as a sceptical Change Advisory Board member. List the
  top 5 questions you would ask before approving it."

## Habits worth modelling in every phase

- **Ask for confidence and caveats:** "How confident are you in this
  conclusion, and what evidence would change your mind?"
- **Ask for alternatives before committing:** "What's another plausible
  explanation, and how would I rule it out?"
- **Ask it to cite its sources:** "Which log lines, metrics or files did
  you base this on?" This makes made-up details easier to catch.
- **Keep it blameless:** "Rephrase this to describe the system gap,
  not the person."
- **Iterate:** treat AI output as a fast first draft to edit, not a
  final answer.
- **Protect data:** remove customer names, contact details, secrets and
  tokens before pasting tickets or logs into an external AI tool.
