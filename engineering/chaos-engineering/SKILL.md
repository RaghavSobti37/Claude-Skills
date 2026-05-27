---
name: chaos-engineering
description: >
  Chaos engineering practice: hypothesis-driven fault injection to surface
  weakness before users do. Use when designing a chaos experiment, planning a
  gameday, picking what to inject (network, host, dependency, resource, state),
  computing the blast radius of an experiment, building a chaos maturity model
  for a team, or running a post-incident game to verify the same failure
  doesn't recur. Covers the four chaos maturity levels, the experiment design
  loop (steady-state → hypothesis → variables → blast-radius → run → learn),
  and the catalog of fault types per layer.
license: MIT + Commons Clause
metadata:
  version: 1.0.0
  author: borghei
  category: engineering
  domain: engineering
  updated: 2026-05-27
  tags: [chaos-engineering, resilience, fault-injection, gameday, sre, reliability, blast-radius]
---

# Chaos Engineering

End-to-end chaos engineering: experiment design, fault injection catalog, gameday execution, and the maturity model that turns one-off "let's break stuff" exercises into a reliable discipline. Provider-agnostic — works whether you use Litmus, Chaos Mesh, AWS FIS, Gremlin, ChaosToolkit, or hand-rolled scripts.

This skill answers four questions: **what to inject, where to inject it, how to size the blast, and how to extract durable learning** from each run.

---

## When to use this skill

| Situation | Skill applies |
|-----------|---------------|
| Spinning up a chaos program from scratch | Yes — start with **maturity model** + **first 5 experiments** |
| Designing a single experiment for a known concern | Yes — use the **experiment design loop** |
| Planning a gameday for a team or service | Yes — use `scripts/gameday_planner.py` |
| Validating a kill switch or fallback path actually works | Yes — chaos is the way to test these in prod-like conditions |
| Post-incident verification: "did the fix really fix it?" | Yes — re-inject the original fault, confirm the new behavior |
| Compliance evidence (SOC 2 A1 / DORA Art. 25) | Yes — chaos runs produce auditable resilience-testing evidence |
| Improving SLOs / error budgets | Pair with `engineering/observability-designer` — chaos surfaces SLO violations |

---

## The chaos engineering principles (Principles of Chaos, applied)

Five principles drive every experiment. Skip one and you're not doing chaos engineering — you're either generating noise or producing false confidence.

1. **Define steady state.** What's "normal" for the system? Measured quantitatively (RPS, error rate, latency, conversion). If you can't define it, you can't detect deviation.
2. **Hypothesize that steady state continues under the perturbation.** "If we kill one of the three recommendation pods, P99 latency stays within +50ms of baseline." Specific, falsifiable.
3. **Inject real-world events.** Things that actually happen: a node dies, a network partition, a DNS lookup fails, a downstream API returns 503, a disk fills up.
4. **Run in production (eventually).** Staging chaos finds staging bugs. Many critical failures only show up at production scale, with production traffic. Start in staging; graduate carefully.
5. **Minimize blast radius.** Constrain the experiment so a wrong hypothesis costs you minimum users / requests / dollars. Use `scripts/blast_radius_calculator.py`.

A chaos engineer's job is to find the gap between the hypothesis and reality. The interesting result is "hypothesis disproven" — that's where you learn.

---

## Chaos maturity model

Four levels. Most teams should target Level 2-3; only mature organizations need Level 4.

| Level | What it looks like | Frequency | Risk tolerance |
|-------|---------------------|-----------|----------------|
| **L0 — None** | Ad-hoc "let's see what happens" exercises, no method | Random | n/a |
| **L1 — Manual, staging** | Hand-run scripts in staging, before-after observation, post-it learnings | Quarterly | Low — staging only |
| **L2 — Automated, staging** | Tooling (Chaos Mesh / Litmus / homegrown) running scheduled experiments in staging, results captured | Weekly to daily | Low |
| **L3 — Production, scheduled** | Targeted production experiments during low-traffic windows, with explicit blast-radius limits and auto-rollback | Weekly | Medium — controlled |
| **L4 — Production, continuous** | Always-on chaos (e.g., Chaos Monkey) in prod; system designed so loss of components is non-events | Continuous | High — by design |

See [references/chaos-principles-and-maturity.md](references/chaos-principles-and-maturity.md) for the per-level scorecard, the "first five experiments" list every L0→L1 team should run, and the org/SRE prerequisites for each level.

---

## The experiment design loop

Every chaos experiment follows this loop. Skipping a step usually means the result is unactionable.

```
1. Define steady state    → quantitative metrics defining "normal"
2. Form hypothesis        → specific, falsifiable prediction
3. Pick variables         → what to vary (one at a time, ideally)
4. Compute blast radius   → who/what is affected; cap the impact
5. Define abort criteria  → when to halt early
6. Run                    → inject; observe; record
7. Analyze                → was hypothesis confirmed or disproven?
8. Act                    → file fixes, update runbooks, retire experiment OR keep it
9. Document               → permanent artifact for future engineers
```

### Step 1: Define steady state

**Bad:** "The service is healthy."
**Good:**
```
SLOs:
  - 99.5% of /search requests return 2xx
  - P95 latency < 200ms
  - P99 latency < 500ms
KPIs:
  - Conversion rate stable within 10% of 7-day MA
  - Active user count stable within 5%
```

If your monitoring can't quantify steady state for the target service, that's your real first problem. Pause chaos; build observability first.

### Step 2: Form a hypothesis

| Bad hypothesis | Good hypothesis |
|----------------|-----------------|
| "Things will be fine if we kill a pod" | "If we kill 1 of 6 search pods, P95 latency stays under 250ms and error rate stays under 0.6% for the duration of the experiment (max 10 min)" |
| "The fallback works" | "If we block all outbound calls to the recommendations service for 5 minutes, the homepage continues to render with static recommendations, and error rate on /home stays under 0.2%" |
| "We can survive a region outage" | "If we fail over from us-east-1 to us-west-2, the failover completes in under 60 seconds, with < 0.5% requests dropped during the cutover and zero data loss" |

A good hypothesis names: the perturbation, the expected outcome, the measurable threshold, the time bound.

### Step 3: Pick variables

Vary one thing at a time when possible. If you inject network latency AND kill a pod simultaneously and the system breaks, you don't know which fault caused it.

Common variables:
- Pod / instance / VM (kill, pause, network-isolate)
- Network (latency, packet loss, partition, bandwidth limit)
- Dependency (block, slow, return error, return malformed response)
- Resource (CPU pressure, memory pressure, disk full, IO throttle)
- State (corrupt cache, expire all tokens, clock skew)
- Traffic (10x spike, replay attack pattern, slowloris)

See [references/fault-injection-catalog.md](references/fault-injection-catalog.md) for the full catalog per layer with tool mappings (Chaos Mesh / Litmus / AWS FIS / Gremlin / ChaosToolkit).

### Step 4: Compute blast radius

Use `scripts/blast_radius_calculator.py` with inputs: total users / traffic, % targeted, duration, expected fallback. Output: worst-case affected users, recommended caps, abort-trigger thresholds.

Key rule: blast radius starts tiny and grows only after each level passes. First run of any experiment: 1 pod / 1% of traffic / 1 minute / one region. Expand from there.

### Step 5: Define abort criteria

Before running, write down: at what metric value do we halt the experiment?

```yaml
abort_if:
  - error_rate_for_target > 5%
  - p99_latency_for_target > 1000ms
  - any_dependent_service_alerting
  - on-call_says_so
abort_method:
  - automatic (kill the chaos tool)
  - manual (stop button in the dashboard)
duration_cap: 10 minutes
```

If your abort can't fire in < 30 seconds, the blast radius is too big.

### Step 6: Run

Run during a window with:
- The owner team online and watching
- Active monitoring dashboards open
- Communication channel open (Slack thread for the experiment)
- The abort mechanism tested and at the ready
- An explicit "GO" call by the experiment lead

Record: start time, end time, what was actually injected, all metrics during the window.

### Step 7: Analyze

After the experiment ends, answer:

- Did the hypothesis hold? (yes / no / partial)
- What unexpected behavior did we see?
- What didn't we measure that we wish we had?
- What was the actual blast radius vs. predicted?
- Did the system recover to steady state, and how long did that take?

### Step 8: Act

Each finding generates an action:

| Finding | Action |
|---------|--------|
| Hypothesis confirmed, system resilient | Document; consider expanding blast radius next time; consider Level-up |
| Hypothesis confirmed, system degraded acceptably | Document the degraded-mode behavior; ensure runbook reflects it |
| Hypothesis disproven, real bug found | File P-level bug; chaos experiment becomes regression test |
| Hypothesis disproven, missing instrumentation | File observability task; rerun chaos after adding |
| Hypothesis disproven, fallback broken | Fix fallback; retest; until then, add kill switch on the upstream |

### Step 9: Document

Permanent artifact: experiment name, hypothesis, blast radius, results, actions taken, link to ticket(s), date last run. This is your chaos test catalog — without it, every experiment is one-off.

---

## Gameday playbook

A gameday is a scheduled, larger-scale chaos exercise — one full afternoon, one specific scenario, the whole team involved.

### Standard gameday agenda (4 hours)

```
T-2 days:  Scenario announced; team briefed; runbooks reviewed
T+0:00     Kick-off, roles assigned (lead, scribe, observers, oncall)
T+0:15     Steady-state baseline captured
T+0:30     Scenario 1 injected — observe, document
T+1:00     Recovery, debrief on scenario 1
T+1:30     Scenario 2 injected
T+2:00     Recovery, debrief
T+2:30     Scenario 3 injected
T+3:00     Recovery, debrief
T+3:30     Synthesis: what surprised us, action items, runbook updates
T+4:00     Wrap, action items assigned with owners + due dates
```

Use `scripts/gameday_planner.py` to generate a tailored agenda + roles + scenarios for a given service.

### Roles

| Role | Responsibility |
|------|----------------|
| Lead | Calls the scenarios, manages time, makes go/no-go calls |
| Scribe | Records what happened, when, with timestamps |
| Operator | Actually injects the chaos (using the tool of choice) |
| Observers | Watch dashboards for each domain (frontend, backend, infra, business KPIs) |
| Oncall | Real on-call rotation, observing but not participating — if a real incident lands during the gameday, they handle it |
| External rep | Customer support or a product manager — sees user-impact reports if any |

### Scenarios — picking good ones

A good gameday scenario:

- Tests a hypothesis the team isn't sure about
- Has been **rehearsed in staging** before being run in prod
- Has a clear abort
- Produces durable artifacts (runbook updates, dashboards, fixes)

Bad scenarios: too small (no learning), too big (real incident risk), unrelated to actual production risks (training a muscle nobody needs).

See [references/gameday-playbook.md](references/gameday-playbook.md) for 12 scenario templates by domain (API service, database, frontend, async pipeline, multi-region, etc.), agendas for half-day and full-day formats, and post-gameday writeup templates.

---

## Anti-patterns

- **Chaos without observability.** You inject a fault, the system behavior changes, and you can't tell because dashboards don't show it. Fix observability first.
- **No hypothesis.** "We turned off the database and the app broke." That's not chaos engineering, that's an outage.
- **Big-bang first experiment.** Don't start with "kill the primary database." Start with "kill one pod of a stateless service."
- **No abort criteria.** Experiment runs longer than intended; one observer goes to lunch; nobody knows when to stop.
- **Single-person chaos.** One engineer experiments without team coordination. Customer impact happens, no one knows why. Always announce, always have an observer.
- **Chaos as competition.** "Let's see if we can break X" framings encourage gotcha-style runs that miss the point. The point is learning, not winning.
- **No follow-through.** Run a gameday, surface 10 issues, fix none. The next gameday surfaces the same 10. Discipline = closing the loop on findings.
- **Skipping staging.** Production-first chaos is L4. If you're at L1-L2, do not run in prod. Earn the right by demonstrating the practice in staging.
- **Chaos and a release the same day.** Compounds variables; you can't tell which change caused what.

---

## First five experiments (for an L0/L1 team)

Run these in staging, in order, before attempting anything in production:

1. **Kill one pod of a stateless service.** Hypothesis: service continues serving traffic with under 2× latency increase for < 30 seconds. Tool: `kubectl delete pod`.
2. **Inject 200ms network latency between service A and service B.** Hypothesis: end-to-end latency increases by ≤ 200ms; no errors. Tool: Chaos Mesh / `tc qdisc`.
3. **Block all outbound network from a service** for 60 seconds. Hypothesis: service either returns degraded responses (caches, fallbacks) or fails fast with a clear error. Tool: Chaos Mesh NetworkChaos.
4. **Fill the disk** of a writeable service to 95%. Hypothesis: service rejects new writes gracefully; alerting fires; no data corruption. Tool: `dd if=/dev/zero of=/var/tmp/fill bs=1M count=...`
5. **Spike traffic 5x** for 5 minutes against the search endpoint. Hypothesis: autoscaling kicks in within 90 seconds; latency stays under SLO; no errors > 1%. Tool: load generator (k6 / Locust / Vegeta).

Each experiment generates its own artifact (hypothesis + result + action items) and starts the team's catalog.

---

## End-to-end workflows

### Workflow: Design and run a single experiment

1. Pick a concern — "I'm not sure our payment service handles upstream PSP timeouts gracefully."
2. Use `scripts/chaos_experiment_designer.py --target payments-svc --fault dependency-timeout --duration 5m` to scaffold the experiment doc.
3. Define steady state (current error rate, latency).
4. Write a falsifiable hypothesis.
5. Compute blast radius with `scripts/blast_radius_calculator.py`.
6. Define abort criteria.
7. Run in staging. Observe. Record.
8. Analyze, document, file follow-ups.
9. If staging-clean, propose a smaller prod version (1% / 60s).

### Workflow: Plan a gameday

1. Identify the scenario — "What happens if our primary region degrades?"
2. Generate agenda with `scripts/gameday_planner.py --service search-api --duration 4h --scenarios region-failover,dep-outage,cache-flush`.
3. Brief the team 2 days ahead.
4. Rehearse the scenarios in staging.
5. Run the gameday with full roster.
6. Write up findings; assign action items with owners.
7. Re-run the same scenarios in 3 months to confirm fixes.

### Workflow: Use chaos to verify a kill switch (cross-skill with feature-flags-architect)

1. The team has a kill switch on the recommendations service (`ops.recs.kill_switch`).
2. Hypothesis: when the kill switch is flipped, no calls go to the recs service, traffic falls back to static recommendations, error rate stays under 0.2%.
3. In staging: inject 100% network block to the recs service. Confirm the kill switch was actually flipped (not just relying on circuit breaker), confirm fallback works.
4. In prod: during low-traffic window, flip the kill switch for 5 minutes with observers watching.
5. Confirm hypothesis. Document. Schedule quarterly re-test.

### Workflow: Post-incident verification

1. After an incident is resolved, write a chaos experiment that recreates the failure.
2. Run in staging — should now succeed (incident shouldn't recur).
3. Keep the experiment in the catalog; re-run periodically (monthly or quarterly).
4. If it ever fails again, you've caught a regression before customers did.

---

## Tooling outputs

| Script | Input | Output |
|--------|-------|--------|
| `scripts/chaos_experiment_designer.py` | Target service, fault type, optional duration / blast targets | Markdown experiment doc with hypothesis stub, steady-state template, abort criteria, observation checklist |
| `scripts/blast_radius_calculator.py` | Total users / traffic, % targeted, fault type, fallback assumption | Worst-case affected users, recommended caps per maturity level, abort-trigger metrics |
| `scripts/gameday_planner.py` | Target service, duration, scenario list, team size | Markdown gameday agenda with roles, timeline, observation checklist, debrief template |

All scripts: stdlib only, argparse CLI, JSON or markdown output.

---

## References

- [chaos-principles-and-maturity.md](references/chaos-principles-and-maturity.md) — 5 principles, 4 maturity levels, level-up criteria, "first 5 experiments"
- [gameday-playbook.md](references/gameday-playbook.md) — 12 scenario templates, half-day / full-day agendas, debrief templates
- [fault-injection-catalog.md](references/fault-injection-catalog.md) — fault types per layer (network, host, dep, resource, state, traffic) with tool mappings

---

## Related skills

- `engineering/observability-designer` — wire metrics needed to define steady state
- `engineering/incident-commander` — chaos is rehearsal for the incident response you'll need
- `engineering/feature-flags-architect` — kill switches verified by chaos; chaos verified by flags
- `ra-qm-team/dora-compliance-expert` — DORA Article 25 requires resilience testing; chaos runs are evidence
