---
name: post-mortem
description: >
  Blameless post-mortem expert for incidents, outages, regressions, customer
  escalations, missed launches, and failed experiments. Pairs Google SRE
  practice with Allspaw / Dekker / Perrow systems thinking.
license: MIT + Commons Clause
metadata:
  version: 1.0.0
  author: borghei
  category: project-management
  domain: pm-execution
  updated: 2026-05-22
  tech-stack: [post-mortem, incident-management, rca, blameless, 5-whys]
  tags: [post-mortem, rca, blameless, incident, sre, five-whys, causal-tree]
---
# Post-Mortem (Blameless Incident Review)

## Overview

A post-mortem is a structured, blameless review held after an incident, outage, regression, missed launch, or failed experiment. The goal is not to assign fault but to learn how the system (people, process, code, and organization) produced the outcome, and to commit to durable changes that reduce the chance of recurrence.

This skill operationalizes the Google SRE blameless post-mortem template, the Etsy "morgue" tradition, John Allspaw's "How Complex Systems Fail" reading, Charles Perrow's Normal Accident Theory, and Sidney Dekker's *Field Guide to Understanding "Human Error"*. Where the companion `discovery/pre-mortem/` skill imagines failure before it happens, post-mortem learns from failure that already did.

### When to Use

- **Severity 1 / Severity 2 incident** — customer-facing outage, data loss, security event, payments failure, or significant regression.
- **Sev 3 with novelty** — even a small incident is worth a post-mortem if it surfaced a class of failure the team has not seen before.
- **Missed launch or rolled-back release** — the launch itself is the incident.
- **Failed experiment with a negative business outcome** — a pricing test that depressed revenue, an onboarding change that hurt activation.
- **Customer escalation** — an executive customer call where the product was the proximate cause.
- **Near miss** — the deploy that "almost" took the site down. Post-mortems on near misses are some of the highest-leverage learning a team gets.

If the incident is below the team's severity threshold and the team has seen the same class of failure recently, document the recurrence in the existing post-mortem rather than producing a new one.

## Severity Thresholds

| Severity | Customer impact | Post-mortem required? | Distribution |
|---|---|---|---|
| **Sev 0 / Sev 1** | Full outage, data loss, security incident | Mandatory; within 5 business days | All engineering + execs |
| **Sev 2** | Partial outage, major degradation, payments impact | Mandatory; within 7 business days | All engineering |
| **Sev 3** | Single-feature degradation, recoverable | Optional unless novel pattern | Owning team |
| **Sev 4** | Minor issue, manual workaround available | Skip unless near-miss | None |
| **Near miss** | Almost caused Sev 0/1/2 | Recommended | Owning team + SRE |

Severity is set at incident open and confirmed at incident close; it can be revised upward but rarely downward.

## Blameless Principles

The single most-cited reason post-mortems fail to produce learning is that participants feel unsafe. Blameless does not mean "consequence-free" — accountability still exists for following process. It means: assume that everyone involved acted reasonably given what they knew at the time, and focus on the system that surrounded their decision.

### Five blameless ground rules

1. **No names in the narrative.** Refer to roles, not people. "The on-call engineer", not "Sarah". Names appear only in the contacts table and the action-item owner column.
2. **No "should have".** Replace with "the system did not surface". "The on-call should have noticed the queue depth" → "The dashboard did not alert on queue depth above 10,000".
3. **No counterfactuals in the root cause.** "If only X had not happened" is not a cause; it is a wish. Stick to mechanisms.
4. **Human error is a symptom, not a cause** (Dekker). When a human acted "incorrectly", the goal is to understand why that action was the locally rational thing to do.
5. **Hindsight bias is the enemy.** What is obvious now was not obvious then. Reconstruct what was knowable in the moment, not what is knowable now.

### The Allspaw test

Adapted from John Allspaw's writing: after writing the post-mortem, ask: *"Would I send this document to the engineer who pushed the button, and would they feel that it represented their experience fairly?"* If no, rewrite until yes.

## Post-Mortem Template Sections

A standard post-mortem has 10 sections, in this order:

1. **Header** — title, date, severity, duration, status (draft / final), authors.
2. **Summary** — 3-5 sentences. What happened, who was affected, how it was resolved, what the durable fix is. A reader who reads only this section should still know the headline.
3. **Impact** — customers affected (count and segments), revenue impact, SLO error budget burned, internal teams blocked.
4. **Timeline** — minute-by-minute log of events from detection to resolution. Include the trigger, detection, escalation, mitigation, and resolution timestamps. Mark "moment of decision" rows.
5. **What went well** — actions, tooling, or decisions that reduced impact. This section is critical and is the one most often skipped. It anchors the document in learning, not just blame avoidance, and surfaces practices worth replicating.
6. **What went wrong** — the system conditions that allowed the incident to happen and to persist. Phrased as system properties, not human failures.
7. **Contributing factors** — the list of conditions that, in combination, produced the outcome. Distinguish from root cause (below).
8. **Root cause analysis** — 5 Whys or Causal Tree (see next section). Document the method used.
9. **Action items** — durable changes. Each item has an owner, a due date, a tracking issue ID, and a category (prevent / detect / mitigate / respond / process).
10. **Lessons learned** — broader patterns the team takes away. These often inform engineering standards, runbooks, or hiring.

## Root Cause Methods

### 5 Whys

A linear technique popularized by Toyota. Start with the proximate effect and ask "why?" five times. Each answer becomes the next question.

```
1. Why did the API return 500 errors for 22 minutes?
   → Because the database connection pool was exhausted.
2. Why was the connection pool exhausted?
   → Because a batch job opened 200 connections without releasing them.
3. Why did the batch job not release them?
   → Because the connection lifecycle is managed by a context manager that was bypassed on the error path.
4. Why was the error path bypassing the context manager?
   → Because a refactor in March introduced an early return before the `finally` block.
5. Why did the refactor land without catching this?
   → Because the test suite does not exercise the connection-pool exhaustion path.
```

**Strengths:** fast, anyone can run it, fits in a 15-minute slot.

**Limits:** linear — only finds one chain. Real incidents usually have multiple contributing factors that combine non-linearly. Use 5 Whys when the failure is a clear chain. Use Causal Tree when it is not.

### Causal Tree (Allspaw / SRE style)

A tree where the root is the incident outcome and the branches are the contributing factors. Each factor can have its own sub-factors. Unlike 5 Whys, the tree allows for parallel chains and AND/OR relationships.

```
Outcome: API returned 500s for 22 minutes
├── Connection pool exhausted
│   ├── Batch job leaked connections (code defect)
│   │   ├── Early-return bypassed context manager
│   │   └── Test suite did not cover error path
│   └── Pool size set conservatively for cost reasons
│       └── Capacity planning predates current traffic shape
├── Alert fired late
│   ├── Threshold set at pool 95% (no warn at 80%)
│   └── PagerDuty escalation path stale (rotated owner)
└── Mitigation took 12 minutes
    ├── Runbook missing pool-restart command
    └── New on-call had not shadowed a database incident
```

**Strengths:** captures the multi-factor nature of complex systems failures. Maps cleanly to action items (one mitigation per leaf or sub-tree).

**Limits:** longer to produce. Requires a facilitator who can keep the team from arguing the tree structure rather than the substance.

### Choosing between them

| Use 5 Whys when | Use Causal Tree when |
|---|---|
| Single clear chain of cause | Multiple contributing factors |
| Sev 3 / Sev 4 incidents | Sev 0 / Sev 1 / Sev 2 incidents |
| Pressed for time | Recurring class of failure |
| Audience is a small team | Audience includes execs, multiple teams |

See `references/5-whys-vs-causal-tree.md` for worked examples of both.

## Root Cause vs Contributing Factors

A common post-mortem failure mode is to declare a single "root cause" and stop. This is Perrow's Normal Accident Theory in action: complex systems fail because multiple contributing factors align, not because one thing broke. A post-mortem that names one root cause and prescribes one fix is usually incomplete.

| | Root cause | Contributing factor |
|---|---|---|
| **Definition** | The conditions that, if absent, would have prevented the incident | Conditions that increased the probability or severity but were not individually sufficient |
| **Example** | "Connection-pool exhaustion in service X" | "Stale on-call rotation", "missing runbook entry", "no canary deploy" |
| **Action implication** | Always addressed | Each addressed unless explicitly accepted with rationale |

Allspaw's reading of Perrow argues that in tightly-coupled complex systems, there is rarely a single cause; the appropriate framing is "the set of contributing factors that aligned". Write the post-mortem accordingly.

## Workflow

### Day 0 (incident close)

1. Incident commander declares the incident resolved and assigns an author.
2. Author opens the post-mortem document from `assets/post_mortem_template.md`.
3. Fill in Header, Summary (placeholder ok), Impact, and the Timeline scaffold from the incident chat transcript.
4. Schedule the post-mortem meeting within the SLA window (5 business days for Sev 1, 7 for Sev 2).

### Day 1-3 (drafting)

1. Author interviews everyone who took an action during the incident. Use the prompts in `assets/incident_timeline_worksheet.md`.
2. Refine the Timeline with every action, decision, and observation. Mark decision points.
3. Draft "What went well" using `assets/what_went_well_prompts.md`. Aim for 3-7 items minimum.
4. Draft "What went wrong" and "Contributing factors".
5. Run the chosen root-cause method (5 Whys or Causal Tree).

### Day 3-5 (review)

1. Hold the post-mortem meeting. 60-90 minutes. Agenda:
   - 5 min: blameless ground rules read aloud
   - 15 min: Timeline review
   - 15 min: What went well + what went wrong
   - 20 min: Root cause discussion
   - 20 min: Action items (owner + due date for each)
   - 5 min: Lessons learned
2. Apply the Allspaw test to the document.
3. Author finalizes and the incident commander signs off.

### Day 5-30 (follow-through)

1. Each action item is filed in `assets/action_item_tracker.md` and the team's tracker (Jira, Linear, GitHub Issues).
2. Action items are reviewed at sprint planning and at the weekly engineering review.
3. After 30 days, audit the action-item completion rate. Industry data shows ~50% of post-mortem action items are never completed; the team should track and target a higher rate.

## Action-Item Follow-Through

The single largest failure mode of post-mortems is that action items go unbuilt. Mitigations:

- **Each action item has a single named owner** (a person, not a team) and a due date no more than one sprint out for high-priority items.
- **Each action item is a real ticket** in the team's tracker, not a bullet in a doc. Link the ticket from the post-mortem.
- **Action-item review is on a recurring agenda** — weekly engineering review or monthly post-mortem audit. Completion rate is a tracked metric.
- **The owner has authority**, not just responsibility. If the action requires another team, escalate to a program manager or use `daci-framework/` to assign decision rights.
- **Acceptance criteria are testable.** "Improve monitoring" is not an action item. "Add an alert on queue depth > 5,000 with PagerDuty page to #payments-oncall" is.

## Post-Mortem Distribution & Archive

| Audience | Format | Channel |
|---|---|---|
| Owning team | Full document | Team wiki, Jira/Linear linked |
| All engineering | Full document | Engineering-wide channel, weekly digest |
| Executives (Sev 0/1) | Summary + Impact + Action items | Exec email or weekly review |
| Public (if customer-impacting) | Public-facing status-page postmortem | status.<company>.com |
| Future engineers | Searchable archive | Wiki tag `post-mortem`, indexed by service |

Post-mortems should be discoverable. A new engineer joining the team should be able to find every past incident on the services they own within 5 minutes. Tag every post-mortem with the affected services, the date, the severity, and the failure class.

## Tools

This skill is template-driven. No Python automation. The artifacts are:

| Artifact | Purpose |
|---|---|
| `assets/post_mortem_template.md` | Full Google SRE-style template, ready to copy and fill |
| `assets/incident_timeline_worksheet.md` | Interview prompts and timestamp scaffolding |
| `assets/action_item_tracker.md` | Spreadsheet-style tracker for action items across post-mortems |
| `assets/what_went_well_prompts.md` | 20+ prompts to draw out positive observations |
| `references/blameless-culture-guide.md` | Deep dive on blameless principles, with examples of blameful vs blameless phrasing |
| `references/5-whys-vs-causal-tree.md` | When to use each method, with worked examples |

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| Post-mortem feels blameful despite no names | Counterfactuals ("if only", "should have") embedded in narrative | Search the document for "should", "could have", "if only"; rewrite each as a system property the team can change |
| Action items never get completed | No single owner, no tracker ticket, or no recurring review | Each item must have one named owner and a real Jira/Linear ticket; review completion in weekly engineering ops |
| Same incident class recurs | First post-mortem identified a symptom, not contributing factors; or contributing-factor action items were dropped | Re-run with Causal Tree instead of 5 Whys; audit what action items from the prior post-mortem were filed but not completed |
| Team avoids difficult observations | Psychological safety low; managers in the room creating power dynamics | Run an anonymous pre-meeting survey; consider a facilitator outside the team; separate post-mortems from performance reviews explicitly |
| Post-mortem takes more than 2 weeks to publish | Author has competing priorities or scope crept into a sweeping retrospective | Set a hard "first draft within 3 business days" rule; cut anything not directly tied to this incident; spin off broader themes into a separate retrospective |
| Executives push for a single root cause | Cultural expectation of accountability framed as blame | Educate using Normal Accident Theory framing; provide the contributing-factor list as the answer to "what caused this"; offer one-line summary that lists top 3 contributing factors |
| "What went well" section is empty | Author skipped the section under time pressure, or team culture undervalues positive observations | Use `assets/what_went_well_prompts.md`; require minimum 3 items before publishing; surface positives in distribution channels |

## Success Criteria

- Every Sev 0 / Sev 1 / Sev 2 incident has a post-mortem published within the SLA window (5 / 5 / 7 business days)
- 100% of post-mortems pass the Allspaw test ("would I send this to the engineer who pushed the button?")
- Each post-mortem has at least one item in "What went well" (minimum 3 preferred)
- Every action item has a named owner, a tracker ticket, and a due date
- Post-mortem action-item 30-day completion rate >= 70%; 90-day completion rate >= 90%
- Recurring incident classes (same failure mode within 90 days) decrease quarter over quarter
- New engineers can find post-mortems for their service within 5 minutes of starting
- Post-mortems are explicitly disconnected from performance review processes; no engineer has ever been disciplined as a direct result of a post-mortem narrative

## Scope & Limitations

**In Scope:** Post-mortem authoring for incidents, outages, regressions, missed launches, failed experiments, customer escalations, and near misses. Severity classification, blameless facilitation, 5 Whys, Causal Tree analysis, action-item tracking, distribution and archival practice.

**Out of Scope:** Live incident command and on-call coordination (see `delivery-manager/`). Sprint retrospectives on team practice (see `sprint-retrospective/`). Risk surfacing before launch (see `discovery/pre-mortem/`). Quantitative reliability engineering and error-budget policy (see `engineering/` skills if present).

**Important Caveats:**
- Blameless culture is a prerequisite, not an output. If management uses post-mortems as a performance signal, the documents will become sanitized and useless. Establish the explicit separation in writing.
- Allspaw's "How Complex Systems Fail" (Cook, 1998) is a 4-page paper worth reading verbatim before facilitating a complex Sev 0 / Sev 1 post-mortem. Linked in `references/blameless-culture-guide.md`.
- Some regulated industries (medical devices, aviation, financial services) require named accountability and cannot use a fully blameless framing in compliance contexts. In those cases, run a parallel internal blameless post-mortem for learning, and produce the regulator-facing document separately.
- A post-mortem is a learning artifact, not a fixing artifact. The fix happens in the action items and their follow-through. A perfect post-mortem with zero completed action items is a failed post-mortem.

## Integration Points

| Integration | Direction | What flows |
|---|---|---|
| `discovery/pre-mortem/` | Bidirectional | Post-mortem findings update next launch's pre-mortem risk register; pre-mortem mitigations become post-mortem-checked controls |
| `delivery-manager/` | Receives from | Incident response context, severity classification, on-call handoffs |
| `sprint-retrospective/` | Bidirectional | Team-practice retros surface incident patterns; post-mortems feed retro themes |
| `daci-framework/` | Feeds into | Action items use DACI to assign owner (D), accountable (A), consulted, informed |
| `execution/dependency-map/` | Receives from | Cross-team contributing factors map to dependency-graph nodes |
| `execution/status-update-generator/` | Feeds into | Sev 0/1 incidents surface in weekly executive status updates |
| `senior-pm/` | Feeds into | Repeated incident classes feed portfolio risk register via `risk_matrix_analyzer.py` |
| `scrum-master/` | Feeds into | Action items become sprint backlog items with mitigation-focused stories |

## References

- `references/blameless-culture-guide.md` — blameless principles, blameful vs blameless phrasing, the Allspaw test
- `references/5-whys-vs-causal-tree.md` — comparison of methods with worked examples
- `assets/post_mortem_template.md` — Google SRE-style 10-section template
- `assets/incident_timeline_worksheet.md` — interview prompts and timestamp scaffold
- `assets/action_item_tracker.md` — cross-incident action-item tracker
- `assets/what_went_well_prompts.md` — prompts to draw out positive observations
- Google SRE Book, "Postmortem Culture: Learning from Failure" — https://sre.google/sre-book/postmortem-culture/
- Etsy "morgue" tool and post-mortem tradition — Allspaw, "Blameless PostMortems and a Just Culture" (2012)
- John Allspaw, "Etsy's Debriefing Facilitation Guide" — https://github.com/etsy/DebriefingFacilitationGuide
- Richard Cook, "How Complex Systems Fail" (1998) — 4-page paper foundational to SRE thinking
- Charles Perrow, *Normal Accidents: Living with High-Risk Technologies* (1984)
- Sidney Dekker, *The Field Guide to Understanding "Human Error"* (3rd ed., 2014)
- Sidney Dekker, *Just Culture: Restoring Trust and Accountability in Your Organization* (3rd ed., 2017)
