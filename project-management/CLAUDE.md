# Project Management Skills - Claude Code Guidance

This guide covers the **42 production-ready project management skills** organized across role-based, discovery, execution, career, and integration domains. PM is the most-used domain in this library.

## PM Skills Overview

**Available Skills (42 total):**

### Role-Based Skills (11)
1. **senior-pm/** - Portfolio management, stakeholder mapping, EMV risk analysis, executive reporting
2. **scrum-master/** - Sprint analytics, team health, velocity forecasting, capacity planning
3. **delivery-manager/** - Release management, deployment coordination, incident response
4. **program-manager/** - Multi-project coordination, dependency management
5. **agile-coach/** - Agile transformation, team coaching, maturity assessment
6. **jira-expert/** - Jira administration, workflows, JQL, automation
7. **confluence-expert/** - Documentation, knowledge management, collaboration
8. **atlassian-admin/** - Atlassian suite administration and configuration
9. **atlassian-templates/** - Ready-to-use templates for Jira and Confluence
10. **sprint-retrospective/** - Data-driven sprint retrospectives
11. **packaged-skills/** - Bundled skill packages for quick deployment

### Discovery Skills (5) — `discovery/`
12. **brainstorm-ideas/** - Product Trio ideation, Opportunity Solution Trees
13. **brainstorm-experiments/** - Lean experiment design, XYZ hypotheses, pretotyping
14. **identify-assumptions/** - Assumption mapping across 4-8 risk categories
15. **pre-mortem/** - Tiger/Paper Tiger/Elephant risk classification
16. **interview-synthesis/** ★ NEW - Customer interview → opportunity solution tree

### Execution Skills (21) — `execution/`
17. **create-prd/** - 8-section PRD scaffolding with problem framing canvas and working backwards PR
18. **prfaq/** ★ NEW - Amazon Working Backwards PR/FAQ as standalone skill
19. **brainstorm-okrs/** - OKR brainstorming and validation (Wodtke framework)
20. **north-star-metric/** ★ NEW - NSM + input metric tree definition
21. **outcome-roadmap/** - Output→outcome roadmap transformation
22. **roadmap-communication/** ★ NEW - Exec/customer/internal roadmap variants
23. **prioritization-frameworks/** - 9-framework reference with multi-scorer
24. **backlog-refinement/** ★ NEW - INVEST + DoR/DoD + splitting playbook
25. **story-splitting/** ★ NEW - 9 vertical slicing patterns (Lawrence)
26. **story-mapping/** - Jeff Patton user story mapping for release planning
27. **job-stories/** - JTBD discovery canvas and When/Want/So backlog format
28. **wwas/** - Why-What-Acceptance structured backlog items
29. **cycle-time-analyzer/** ★ NEW - Flow metrics (lead time, cycle time, CFD)
30. **dependency-map/** ★ NEW - Cross-team dependency tracking + critical path
31. **status-update-generator/** ★ NEW - Weekly exec update from Jira/Linear data
32. **summarize-meeting/** - Structured meeting summaries with action items
33. **daci-framework/** - DACI decision facilitation and governance
34. **beta-program/** ★ NEW - Closed beta playbook
35. **launch-playbook/** ★ NEW - Internal + external launch coordination
36. **release-notes/** - Release notes generation from tickets/changelogs
37. **eol-communication/** - End-of-life product messaging and sunset communication

### Career Skills (4) ★ NEW — `career/`
38. **pm-interview-prep/** - APM/PM/Sr PM/Group PM interview rubrics + frameworks
39. **pm-career-ladder/** - Rubrics, growth plans, promo packets
40. **pm-onboarding/** - 30-60-90 day plan for new PMs
41. **pm-1on1s/** - 1:1 templates by partner type (EM, designer, IC, manager)

### Integration Skills (2) ★ NEW
42. **linear-expert/** - Linear administration, GraphQL queries, Jira → Linear migration
42. **notion-pm/** - Notion DBs for PRDs/OKRs/Roadmap/Decisions, API patterns

**Key Features:**
- **Atlassian MCP Server** integration for direct Jira/Confluence operations
- **Linear + Notion** integration patterns (GraphQL + REST)
- **15+ Python CLI tools** across role-based, discovery, execution, career
- **Shared output schema** (`--format json|markdown|mermaid|confluence|notion|linear`) — see `SHARED_OUTPUT_SCHEMA.md`
- **Worked examples** in `examples/` showing end-to-end flows
- **Role-based Quick Start** in `README.md` from APM → CPO

## Shared Output Format Schema

All PM Python tools follow `SHARED_OUTPUT_SCHEMA.md`:

```bash
python <tool>.py --format <json|markdown|mermaid|confluence|notion|linear> [--output file]
```

When generating outputs for a PM artifact, always consider which format the consumer expects:
- **json** for machine pipelines (CI, MCP servers, agents)
- **markdown** for GitHub PRs/READMEs (default)
- **mermaid** for visual diagrams (story maps, roadmaps, dependency graphs, opportunity trees)
- **confluence** for Confluence pages via Atlassian MCP
- **notion** for Notion pages via API
- **linear** for Linear issues via API

## Atlassian MCP Integration

**Purpose:** Direct integration with Jira and Confluence via Model Context Protocol (MCP)

**Capabilities:**
- Create, read, update Jira issues
- Manage Confluence pages and spaces
- Automate workflows and transitions
- Generate reports and dashboards
- Bulk operations on issues

**Setup:** Atlassian MCP server configured in Claude Code settings

**Usage Pattern:**
```bash
mcp__atlassian__create_issue project="PROJ" summary="New feature" type="Story"
mcp__atlassian__create_page space="TEAM" title="Sprint Retrospective"
```

## Linear Integration (linear-expert/)

Linear's API is GraphQL-first. Common patterns:

```graphql
# List issues
query { issues(first: 50, filter: { state: { type: { eq: "started" } } }) {
  nodes { id identifier title priority assignee { name } }
}}

# Create issue
mutation { issueCreate(input: {
  teamId: "...", title: "...", description: "...", priorityLabel: "High"
}) { issue { id identifier url } } }
```

See `linear-expert/references/linear-graphql-patterns.md` for full library.

## Notion Integration (notion-pm/)

Notion API is REST. PM workflows center on DB-driven artifacts:

```bash
# Create a PRD page in the PRDs database
curl -X POST https://api.notion.com/v1/pages \
  -H "Authorization: Bearer $NOTION_TOKEN" \
  -d '{
    "parent": { "database_id": "<prd-db-id>" },
    "properties": { ... },
    "children": [ <blocks from prd_scaffolder.py --format notion> ]
  }'
```

See `notion-pm/references/notion-api-patterns.md` for full library.

## Skill-Specific Guidance

### Senior PM (`senior-pm/`)

**Focus:** Portfolio management, stakeholder mapping, risk analysis, executive reporting

**Python Tools:** `project_health_dashboard.py`, `risk_matrix_analyzer.py`, `resource_capacity_planner.py`, `stakeholder_mapper.py`

**Key Workflows:**
- Portfolio health assessment (three-tier analysis)
- Stakeholder mapping with Mendelow's Matrix
- Risk quantification with EMV analysis
- Executive report generation

### Scrum Master (`scrum-master/`)

**Focus:** Data-driven sprint analytics, team health, velocity forecasting, capacity planning

**Python Tools:** `velocity_analyzer.py`, `sprint_health_scorer.py`, `retrospective_analyzer.py`, `sprint_capacity_calculator.py`

**Key Workflows:**
- Sprint capacity calculation with ceremony overhead
- Velocity analysis with Monte Carlo forecasting
- Sprint health scoring across 6 dimensions
- Retrospective pattern analysis

### Discovery Skills (`discovery/`)

**Focus:** Product discovery workflows — ideation, experimentation, assumption mapping, risk analysis, interview synthesis

**Python Tools:** `experiment_designer.py`, `assumption_tracker.py`, `risk_categorizer.py`, `interview_synthesizer.py`

**Key Workflows:**
- Product Trio ideation (PM + Designer + Engineer perspectives)
- Lean experiment design with XYZ hypothesis format
- Assumption mapping with Impact × Risk prioritization
- Pre-mortem with Tiger/Paper Tiger/Elephant classification
- Customer interview → opportunity solution tree

### Execution Skills (`execution/`)

**Focus:** PM execution artifacts — PRDs, OKRs, roadmaps, prioritization, refinement, flow metrics, status, launch, release

**Python Tools:** `prd_scaffolder.py`, `okr_validator.py`, `roadmap_transformer.py`, `prioritization_scorer.py`, `release_notes_generator.py`, `refinement_scorer.py`, `status_generator.py`, `metric_tree_builder.py`, `flow_metrics.py`, `dependency_graph.py`

**Key Workflows:**
- PRD generation with 8-section structure
- PR/FAQ Working Backwards
- OKR validation against SMART criteria
- Output→outcome roadmap transformation
- Multi-framework prioritization (RICE, ICE, Opportunity Score, MoSCoW, Weighted)
- Backlog refinement (INVEST, DoR/DoD, splitting)
- Flow metrics (Little's Law, CFD, aging WIP)
- Cross-team dependency mapping with critical path
- Weekly exec status update from issue data
- Beta + launch playbooks
- Release notes from tickets/changelogs

### Career Skills (`career/`) ★ NEW

**Focus:** PM career growth — interview prep, ladder rubrics, onboarding, 1:1s

**Key Workflows:**
- Interview prep across product sense, execution, strategy, behavioral (CIRCLES, AARM, STAR, Decode and Conquer)
- Career ladder rubrics from APM → CPO with gap analysis
- 30-60-90 day onboarding plan (Watkins, STARS)
- 1:1 templates per partner type (manager, EM, designer, IC)

### Integration Skills

**linear-expert/** - GraphQL-first, cycle/project/initiative hierarchy, Jira → Linear migration patterns
**notion-pm/** - DB-driven PM artifacts (PRDs DB, OKRs DB, Roadmap DB, Decisions DB), Notion API REST patterns

## Integration Patterns

### Pattern 1: Discovery → Execution → Delivery

```
brainstorm-ideas → identify-assumptions → brainstorm-experiments → pre-mortem
      ↓                                                                    ↓
create-prd → brainstorm-okrs → outcome-roadmap → prioritization-frameworks
      ↓
backlog-refinement → story-splitting → (build) → release-notes
      ↓
status-update-generator → (weekly exec comms)
```

### Pattern 2: Sprint Planning (Data-Driven)

```bash
python scrum-master/scripts/sprint_capacity_calculator.py team.json
python scrum-master/scripts/velocity_analyzer.py sprints.json
python execution/prioritization-frameworks/scripts/prioritization_scorer.py backlog.json --framework rice
python execution/cycle-time-analyzer/scripts/flow_metrics.py issues.json
mcp__atlassian__create_sprint board="TEAM-board" name="Sprint 23"
```

### Pattern 3: Cross-Team Coordination

```bash
python execution/dependency-map/scripts/dependency_graph.py deps.json --format mermaid
python execution/daci-framework/...  # Document decision rights
python execution/status-update-generator/scripts/status_generator.py --format confluence
mcp__atlassian__create_page space="PROG" title="Weekly status" body="$(...)"
```

### Pattern 4: Customer Discovery → Roadmap

```bash
python discovery/interview-synthesis/scripts/interview_synthesizer.py interviews.json --format mermaid
python discovery/identify-assumptions/scripts/assumption_tracker.py opportunities.json
python execution/north-star-metric/scripts/metric_tree_builder.py --nsm "QBR engagement"
python execution/outcome-roadmap/scripts/roadmap_transformer.py outputs.json
```

### Pattern 5: Push to Linear (instead of Jira)

```bash
python execution/prioritization-frameworks/scripts/prioritization_scorer.py backlog.json \
  --framework rice --format linear | linear-cli issues create-batch --team PROD
python execution/status-update-generator/scripts/status_generator.py issues.json \
  --format linear --output status.md
```

### Pattern 6: Push to Notion DB

```bash
python execution/create-prd/scripts/prd_scaffolder.py \
  --product-name "Shared Dashboards" --format notion --output prd.md
# Then post to Notion API via notion-pm/ patterns
```

## Career Track Usage Patterns

### Pattern: Interview Prep
```
career/pm-interview-prep/ → frame question type → use CIRCLES/AARM/STAR → practice with mocks
```

### Pattern: Promo Cycle
```
career/pm-career-ladder/ → assess against rubric → identify gaps → 90-day growth plan → promo packet
```

### Pattern: New PM Onboarding
```
career/pm-onboarding/ → 30-60-90 day plan → stakeholder mapping (senior-pm/) → first PRD (create-prd/)
```

## Additional Resources

- **Shared Output Schema:** `SHARED_OUTPUT_SCHEMA.md` (critical for consistent tool integration)
- **Worked Examples:** `examples/` (end-to-end scenarios)
- **Installation Guide:** `INSTALLATION_GUIDE.txt`
- **Implementation Summary:** `IMPLEMENTATION_SUMMARY.md`
- **Real-World Scenario:** `REAL_WORLD_SCENARIO.md`
- **PM Overview:** `README.md`
- **Main Documentation:** `../CLAUDE.md`

---

**Last Updated:** 2026-05-21
**Skills Deployed:** 42/42 PM skills production-ready
**Python Tools:** 15+ CLI tools
**Integrations:** Jira, Linear, Notion, Confluence, GitHub Projects, Atlassian MCP
**Career Track:** 4 skills (unique to this library — no competitor covers PM career growth)
