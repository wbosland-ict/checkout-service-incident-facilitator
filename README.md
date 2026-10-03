# AI Workshop for Service Engineers: Incident → Problem → Change

A hands-on workshop that shows service engineers / SREs where and how AI
tools (e.g. GitHub Copilot Chat/CLI, or any LLM you have access to) fit
into their daily work. The exercises follow the same process we use with
TopDesk:

1. The **service desk** registers an **incident** in TopDesk and assigns
   it to a service engineer.
2. The engineer **restores service** and closes the incident, possibly
   with a **workaround**.
3. If a long-term solution is needed, a **problem** is registered in
   TopDesk. The engineer picks it up and **analyses it thoroughly**.
4. When a solution is found, the engineer submits a **Request for Change**
   using `rfc-template.docx`.

The scenario is fictional but realistic: **ShopFast**, an e-commerce
platform, has a checkout outage caused by a bad deploy. All logs, metrics,
alerts, TopDesk records, Teams messages and runbooks are synthetic data.

## Learning objectives

By the end of this workshop, participants will be able to:

1. Use AI to quickly triage a TopDesk incident and correlate noisy alerts,
   logs and metrics.
2. Use AI to restore service safely and write clear TopDesk action and
   resolution entries, and to recognise when a resolution is only a
   workaround.
3. Use AI to analyse a problem thoroughly: root-cause hypotheses checked
   against evidence, contributing factors, known error, solution
   options, and a blameless postmortem summary.
4. Use AI to draft a complete, reviewable Request for Change.
5. Recognise where AI speeds things up, and where human judgement,
   verification and ownership must stay with the engineer.

## Prerequisites

- Access to an AI assistant (GitHub Copilot Chat/CLI, ChatGPT, Claude, etc.)
- A terminal / editor to open the files, and Microsoft Word (for the RFC)
- No special infrastructure needed: everything is static sample data

## Format

- **Duration:** ~3¾ hours (4 exercises of 25–55 min each, plus
  debriefs)
- **Group size:** solo, in pairs, or in small groups (3–4 people)
- **Style:** each exercise gives participants a task, sample AI prompts
  to try, and space to compare AI output with the model answers in the
  facilitator guide

## Folder structure

- **`rfc-template.docx`**: the organisation's Request for Change form
  (used in Exercise 4)
- **`checkout-service-incident-exercises/`**: hand this to participants
  - `scenario/00-incident-brief.md`: the process and the incident that
    kicks things off
  - `scenario/01-architecture.md`: system architecture and baseline
  - `exercises/exercise-1-incident-triage.md`
  - `exercises/exercise-2-incident-resolution.md`
  - `exercises/exercise-3-problem-analysis.md`
  - `exercises/exercise-4-change-request.md`
- **`checkout-service-incident-files/`**: synthetic evidence for participants
  - `checkout-service-topdesk/`: TopDesk incident `I 2607 041` and a
    problem template
  - `checkout-service-payload-and-log-excerpts/`: Grafana alerts (posted to
    Teams), Teams channel excerpts, service logs
  - `checkout-service-metrics-and-deploy-history/`: CSV metrics, deploy
    history, post-rollback recovery data
  - `checkout-service-runbook/`: the existing runbook (deliberately incomplete; participants find the gaps in Exercise 3)
  - `checkout-service-problem-evidence/`: extra evidence for the problem
    analysis (code diff, config, traffic trend, payment reconciliation,
    stakeholder notes). **Hand these out at the start of Exercise 3**
    for the most realistic flow, or share everything up front for
    simplicity.
- **`checkout-service-incident-facilitator/`**: facilitator only, don't
  share
  - `README.md`: this file
  - `facilitator-guide.md`: timing, model answers, discussion points
  - `catch-up-cards.md`: short summaries to hand out between exercises
    so groups that fell behind can continue
  - `timeline.md`: ground-truth timeline across incident, problem and
    change
  - `example-problem-record.md`: model answer for Exercise 3
  - `example-rfc-checkout-service.docx`: model answer for Exercise 4
  - `prompt-library.md`: reusable prompts (can be shared after the
    workshop)
  - `workshop-slides.pptx`: slides with speaker notes: Part 1 ways of
    working with AI (before the exercises), Part 2 workshop intro, Part 3
    "What if Copilot could talk to TopDesk?" (our own MCP server in front
    of TopDesk, for engineers and managers, after the exercises)

## Suggested flow

1. Present Parts 1 and 2 of `workshop-slides.pptx` (~20 min): ways of
   working with AI, and the incident → problem → change flow (see also
   `scenario/00-incident-brief.md`) and how it maps to TopDesk.
2. Hand out `checkout-service-incident-exercises/` and
   `checkout-service-incident-files/` (optionally without
   `checkout-service-problem-evidence/` until Exercise 3).
3. Run Exercises 1 → 4 in order; each builds on the previous one.
4. After each exercise, spend 5–10 minutes comparing findings and
   discussing where AI output was accurate, incomplete or made up.
5. Close with the facilitator-led discussion in `facilitator-guide.md`
   ("Where AI helps vs. where it doesn't").
6. Optionally finish with Part 3 of `workshop-slides.pptx` (~20 min plus
   discussion): a concept for our own TopDesk MCP server. TopDesk has no
   MCP server; nothing in Part 3 has been built.
