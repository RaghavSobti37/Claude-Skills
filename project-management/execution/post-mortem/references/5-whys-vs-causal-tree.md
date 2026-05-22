# 5 Whys vs Causal Tree

Two root-cause analysis methods commonly used in blameless post-mortems. They produce different artifacts and are suited to different incident shapes. This guide explains when to pick each, with worked examples.

## 5 Whys

### Origin

The 5 Whys technique originated at Toyota in the 1950s as part of the Toyota Production System, codified by Taiichi Ohno. It is the simplest root-cause method in widespread use and the most often misused.

### How it works

Start with the proximate effect — what the customer experienced or what the monitoring detected. Ask "why?" The answer becomes the next subject. Ask "why?" of that. Continue until you reach a cause the team can act on.

The number "five" is heuristic, not literal. Stop when you reach a cause that:
- The team can address, and
- Is not itself merely a symptom of a deeper cause.

### Worked example: payment API outage

**Effect: The payment API returned HTTP 500 for 22 minutes between 14:18 and 14:40 UTC.**

1. **Why?** The Stripe-facing service ran out of database connections.
2. **Why?** A nightly batch job opened 200 connections and did not release them after an error.
3. **Why?** The connection lifecycle in the batch job is managed by a Python context manager (`with` block), but a refactor introduced an early return before the `finally` block was reached.
4. **Why?** The refactor's pull request had no test that exercised the error path; reviewers focused on the happy path because the test fixture only seeded a successful database.
5. **Why?** The test fixture standard does not require both success and error paths to be covered; the linter enforces structural review, not coverage of execution paths.

**Action item:** Update the test fixture standard to require error-path coverage; add a CI check that any new code path acquiring a database connection is exercised by both success and error tests.

### Strengths

- Fast. A 5 Whys exercise fits in 15 minutes.
- Anyone can run it. No facilitator training needed.
- Produces a clear narrative that flows well in writing.
- Naturally points to one or two action items.

### Limits

- **Linear.** It traces one chain. Real incidents are usually multi-causal.
- **Stops too early.** It is tempting to stop at the first "satisfying" answer.
- **Risks naming a person as the 4th or 5th "why".** If the chain ends at "the engineer made a mistake", you have done it wrong — keep asking why the system allowed the mistake.
- **Confirmation bias.** The questioner often anticipates an answer and unconsciously leads the chain.

### Anti-patterns

- "Why did Joe push the bad code?" — names a person. Reframe as "Why did the bad code reach production?"
- Stopping at "human error". Human error is a symptom; ask why the system permitted it.
- Stopping at "we need more training". Training is rarely the durable fix; ask why the system depends on memory rather than tooling.
- Stretching to exactly five steps when the chain is naturally three or seven.

## Causal Tree

### Origin

The causal tree comes from accident-investigation methodology in safety-critical industries (aviation, healthcare, nuclear). It is the implicit model in John Allspaw's writing on Etsy's debriefing practice and in Charles Perrow's Normal Accident Theory.

### How it works

The root of the tree is the incident outcome. Each branch is a contributing factor — a condition that, if absent, would have made the incident less likely or less severe. Each contributing factor can itself have sub-factors. Branches can run in parallel and combine with AND or OR logic.

The tree captures the structural truth that complex incidents rarely have one cause; they have a set of conditions that aligned.

### Worked example: payment API outage (same incident)

```
Outcome: Payment API 500s for 22 minutes
│
├── BRANCH 1: Database connection pool exhausted
│   ├── Batch job leaked connections (the proximate trigger)
│   │   ├── Early-return in refactored code bypassed context manager
│   │   │   └── Code review did not catch the early return (no error-path test)
│   │   └── Batch job ran during peak hours
│   │       └── Cron schedule predates current peak-hour shift
│   └── Pool size was 50 (set conservatively for cost)
│       └── Capacity planning predates current traffic shape (2024 baseline still in config)
│
├── BRANCH 2: Detection was late (alert fired at minute 8)
│   ├── Alert threshold set at 95% pool utilization
│   │   └── No warning threshold at 80%
│   ├── PagerDuty escalation to on-call DBA was stale
│   │   └── On-call rotation rotated owner in March, escalation policy not updated
│   └── Dashboard for pool depth not on the on-call's default view
│
└── BRANCH 3: Mitigation took 12 minutes after detection
    ├── Runbook for pool exhaustion missing the connection-kill command
    │   └── Runbook last reviewed 18 months ago
    ├── On-call had not done a database incident before
    │   └── New on-call onboarded last month; database incident not in the drill list
    └── Mitigation required SSH bastion access
        └── Bastion access requires a fresh 2FA token; on-call had to re-auth
```

The tree exposes 11 distinct contributing factors across 3 branches. Each leaf or sub-tree maps to a candidate action item. The team can then prioritize: which factors, if changed, would most reduce probability or severity of recurrence?

**Action items from this tree (prioritized):**

1. Fix the connection leak in the batch job (P0, prevent recurrence of trigger).
2. Add a warning alert at 80% pool utilization (P0, improve detection).
3. Update PagerDuty escalation to the current on-call DBA (P0, improve detection).
4. Add the connection-kill command to the runbook; review all DBA runbooks for staleness (P1, improve mitigation).
5. Add a database incident to the on-call drill rotation (P1, improve mitigation).
6. Move the bastion 2FA token to a longer session (P2, improve mitigation).
7. Re-baseline pool size against current peak-hour traffic (P2, prevent recurrence).
8. Move batch job to off-peak hours (P2, reduce impact).

### Strengths

- Captures the multi-factor structure of complex failures.
- Each leaf maps cleanly to an action item.
- Visible structure makes it easy to spot single-point-of-failure assumptions and missing coverage.
- Encourages thinking about prevent / detect / mitigate / respond as separate dimensions.

### Limits

- Time-consuming. A causal tree for a Sev 1 can take 90 minutes of facilitated discussion.
- Requires a facilitator who can keep the discussion focused on system properties.
- Risk of analysis-paralysis: trees can grow indefinitely. Cap at 3 levels deep per branch.
- Less narrative-friendly. The post-mortem document needs prose to wrap the tree.

## How to choose

| Decision factor | 5 Whys | Causal Tree |
|---|---|---|
| Severity | Sev 3 / 4 / near miss | Sev 0 / 1 / 2 |
| Failure shape | Single clear chain | Multiple contributing factors |
| Time available | 15-30 min | 60-90 min |
| Audience | Owning team only | Owning team + cross-team + execs |
| Recurrence | First instance of this class | Recurring failure mode |
| Output format | Inline in post-mortem | Tree diagram + prose |

When in doubt for a Sev 1 or higher, default to Causal Tree. The extra time pays off in better action items and fewer recurrences.

## Combining both

For complex incidents, a hybrid pattern is effective:

1. Start with 5 Whys on the proximate trigger to get a clean chain quickly.
2. Then ask: "What other conditions, if absent, would have prevented or reduced this?"
3. The answers become parallel branches of a causal tree, with the 5 Whys chain as one branch.

This gives the team the speed and narrative clarity of 5 Whys plus the structural completeness of a causal tree.

## Common mistakes (both methods)

- Naming people in any branch or "why". Names belong only in the action-item owner column.
- Stopping at "human error" or "we need training". Always one more why.
- Counterfactuals as causes. "If only X had not happened" is not a cause; "the system permitted X" is.
- Confirmation bias. The investigator goes in with a hypothesis and the chain or tree confirms it. Counter by inviting someone outside the team to ask questions.
- Single-root-cause framing. The conclusion should say "the set of conditions that aligned", not "the root cause was".

## References

- Taiichi Ohno, *Toyota Production System: Beyond Large-Scale Production* (1988)
- Richard Cook, "How Complex Systems Fail" (1998)
- John Allspaw, "Etsy's Debriefing Facilitation Guide" — https://github.com/etsy/DebriefingFacilitationGuide
- Charles Perrow, *Normal Accidents* (1984) — for the theoretical grounding on why single-cause framing fails in complex systems
- Sidney Dekker, *The Field Guide to Understanding "Human Error"* (3rd ed., 2014)
