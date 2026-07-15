# AI Workshop for SREs: Incident Response Scenario

A hands-on workshop that shows Site Reliability Engineers where and how AI
tools (e.g. GitHub Copilot Chat/CLI, or any LLM you have access to) fit into
their daily incident-response workflow — triage, root cause analysis,
remediation, and postmortem writing.

The scenario is fictional but realistic: **ShopFast**, an e-commerce
platform, suffers a checkout outage caused by a bad deploy. All logs,
metrics, alerts, and runbooks are synthetic data included in this repo.

## Learning objectives

By the end of this workshop, participants will be able to:

1. Use AI to rapidly triage and correlate noisy alerts, logs, and metrics.
2. Use AI to generate root-cause hypotheses and validate them against evidence.
3. Use AI to draft safe remediation steps, rollback commands, and runbook updates.
4. Use AI to write a first-draft postmortem and stakeholder communications.
5. Recognize where AI helps speed things up, and where human judgment,
   verification, and ownership must remain in the loop.

## Prerequisites

- Access to an AI assistant (GitHub Copilot Chat/CLI, ChatGPT, Claude, etc.)
- A terminal / editor to open the files in this repo
- No special infrastructure needed — everything is static sample data

## Format

- **Duration:** ~2.5 hours (4 exercises, ~30 min each + wrap-up)
- **Group size:** Works solo or in pairs/small groups (3-4 people)
- **Style:** Each exercise gives participants a task, sample AI prompts to
  try, and space to compare AI output against the "ground truth" in the
  facilitator guide

## Repo structure

- **`scenario/`** — Narrative material — read this first
  - `00-incident-brief.md` — The page/alert that kicks off the incident
  - `01-architecture.md` — System architecture (Mermaid diagram + notes)
  - `02-timeline.md` — Ground-truth timeline (facilitator reference)
- **`data/`** — Synthetic evidence participants investigate
  - `logs/` — Raw service logs
  - `metrics/` — CSV time series (latency, DB pool, etc.)
  - `alerts/` — PagerDuty-style alert payloads
  - `runbooks/` — Existing (partially outdated) runbook
  - `deploy-history.md` — Recent deploys / change log
- **`exercises/`** — The 4 workshop exercises (hand these to participants)
  - `exercise-1-triage.md`
  - `exercise-2-root-cause-analysis.md`
  - `exercise-3-remediation.md`
  - `exercise-4-postmortem-and-comms.md`
- **`prompts/`**
  - `prompt-library.md` — Example prompts participants can adapt/reuse
- **`facilitator-guide.md`** — Answers, timing, discussion points, pitfalls

## Suggested flow

1. Hand out `scenario/00-incident-brief.md` and `scenario/01-architecture.md` only.
   Do **not** share `02-timeline.md` (facilitator-only ground truth) or the
   facilitator guide.
2. Run exercises 1 → 4 in order; each builds on the previous one's findings.
3. After each exercise, spend 5-10 minutes comparing group findings and
   discussing where AI output was accurate, incomplete, or hallucinated.
4. Close with the facilitator-led discussion in `facilitator-guide.md`
   ("Where AI helps vs. where it doesn't").

## Facilitator notes

See `facilitator-guide.md` for model answers, timing per exercise, common
pitfalls to highlight (over-trusting AI root-cause guesses, leaking
sensitive data into prompts, skipping verification), and discussion prompts.
