# Prompt Library

A reusable set of prompt patterns for AI-assisted incident response,
organized by phase. Adapt the bracketed placeholders to paste your real
data. These are starting points, not scripts — encourage participants to
iterate on them.

## Triage

- "Here are N alerts that fired within [time window] for [service]. Rank
  them by likely customer impact and flag any that look like downstream
  symptoms rather than root causes. Explain your reasoning. [paste alerts]"
- "Summarize this log file in plain language for a status update, in 3
  sentences or less. [paste log excerpt]"
- "Given these alerts and this log excerpt, what's the smallest set of
  additional data I should pull next to confirm or rule out a root cause?"
- "Extract every error type and its first-seen timestamp from this log.
  [paste log]"

## Root cause analysis

- "Here's a deploy history and a metrics time series showing degradation
  starting at [time]. Which deploy, if any, most likely triggered this?
  State your confidence and what would increase/decrease it."
- "Explain [technical concept, e.g. 'N+1 query problem' / 'connection pool
  exhaustion' / 'retry storm'] in plain terms and why it would produce
  [specific symptom, e.g. 'DB connections climbing to 100% while request
  volume is roughly flat']."
- "I have hypothesis X for the root cause of this incident. List evidence
  in the attached logs/metrics that supports X, and separately, evidence
  that would contradict X. [paste hypothesis + data]"
- "What's an alternative explanation for these symptoms besides [my
  hypothesis]? What data would distinguish between the two?"

## Remediation

- "Given root cause X and this existing runbook, list my remediation
  options ranked by speed and risk for a [severity] incident with
  [quantified impact]."
- "Draft the exact rollback/mitigation commands for [platform, e.g.
  Kubernetes/ECS/systemd] based on this runbook's existing procedure."
- "What should I monitor in the first 10 minutes after this fix to confirm
  it worked, and what would tell me early that it didn't?"
- "Draft a short Slack incident update announcing [action] before I take
  it. Factual, no jargon, under 4 sentences."
- "Draft a new runbook section for [failure mode] including detection,
  immediate mitigation, and escalation steps, matching the style of this
  existing runbook. [paste existing runbook]"

## Postmortem & communications

- "Draft a blameless postmortem using this structure: [structure]. Focus
  on systemic causes, not individual blame. [paste timeline/root
  cause/contributing factors]"
- "Rewrite this action item to be specific, measurable, and assignable:
  '[vague action item]'"
- "Rephrase this sentence to remove implied individual blame while keeping
  it factual: '[sentence]'"
- "Draft a 2-3 sentence public status-page update for [investigating /
  identified / resolved] status, no internal jargon, no overpromising."
- "Summarize this postmortem for a [VP / customer success / engineering]
  audience in [N] sentences, emphasizing [business impact / prevention /
  technical detail] as appropriate for that audience."

## Cross-cutting habits worth modeling

- **Ask for confidence and caveats**: "State your confidence in this
  conclusion and what evidence would change your mind."
- **Ask for alternatives before committing**: "What's a plausible
  alternative explanation, and how would I rule it out?"
- **Ask it to cite what it used**: "Which specific log lines/metrics did
  you base this on?" — makes hallucination easier to catch.
- **Iterate, don't accept the first draft**: treat AI output as a fast
  first draft to edit, not a final answer to ship.
