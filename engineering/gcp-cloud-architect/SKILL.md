---
name: gcp-cloud-architect
description: >
  Design, review, and validate Google Cloud (GCP) architectures. Use when picking
  the right GCP compute (GKE / Cloud Run / Cloud Functions / GCE / Cloud Run Jobs),
  data store (Cloud SQL / Spanner / Firestore / BigQuery / Bigtable / Cloud Storage),
  networking (VPC / Private Service Connect / Cloud Load Balancing / Cloud Armor),
  identity (IAM / Workload Identity Federation / Service Accounts), or applying the
  Google Cloud Architecture Framework (Operational Excellence, Security, Reliability,
  Cost Optimization, Performance Optimization) to a workload. Pairs with our existing
  senior-cloud-architect (multi-cloud, abstract patterns) by going deep on GCP-specific
  services, pricing, and operational defaults.
license: MIT + Commons Clause
metadata:
  version: 1.0.0
  author: borghei
  category: engineering
  domain: engineering
  updated: 2026-05-27
  tags: [gcp, google-cloud, architecture, cloud-architecture-framework, gke, cloud-run, bigquery, iam, networking, cost-optimization]
---

# GCP Cloud Architect

End-to-end GCP-specific architecture: service selection, Google Cloud Architecture Framework assessment, identity and networking patterns, cost optimization, operational defaults. Provider-specific complement to our generic `senior-cloud-architect` skill — that one covers cross-cloud patterns; this one knows when to pick Spanner over Cloud SQL, how Workload Identity Federation differs from Service Account keys, and the right Cloud Run vs GKE call.

---

## When to use this skill

| Situation | Skill applies |
|-----------|---------------|
| Designing a GCP architecture from scratch | Yes — start with **compute decision tree** |
| Reviewing an existing GCP architecture | Yes — run **CAF assessment** via `scripts/gcp_caf_scorer.py` |
| Validating a Terraform / Deployment Manager plan | Yes — `scripts/gcp_architecture_validator.py` |
| Estimating GCP cost for a workload | Yes — `scripts/gcp_cost_estimator.py` |
| Picking between GKE / Cloud Run / Functions / Cloud Run Jobs | Yes — see **compute decision tree** |
| Setting up IAM / Workload Identity correctly | Yes — see **identity reference** |
| Designing multi-region / multi-zone resilience | Yes — see **reliability reference** |
| Picking Cloud SQL vs Spanner vs Firestore vs BigQuery | Yes — see **data store decision tree** |
| Going to production without CAF review | Don't — run the CAF scorer first |

---

## Compute decision tree

GCP gives you many compute paths; picking the wrong one wastes money and operational burden.

```
Stateless HTTP service?
├── Need full control over OS / sidecar / custom runtime?
│   └── → GKE (Autopilot for managed; Standard for full control)
├── Container-packaged service, want zero infra?
│   ├── Auto-scale to zero acceptable? Per-request billing?
│   │   └── → Cloud Run (Service)
│   └── Long-running container (always-warm)?
│       └── → Cloud Run with min-instances OR GKE Autopilot
├── Function-style, event-driven?
│   └── → Cloud Functions (2nd gen, runs on Cloud Run under the hood)
├── Batch / job processing?
│   ├── Containers, finite duration?
│   │   └── → Cloud Run Jobs
│   ├── Large-scale batch (HPC)?
│   │   └── → Batch (compute engine pool) OR Dataflow (for data)
└── Long-running stateful processes / legacy?
    └── → Compute Engine VMs (MIGs for groups)

Stateful service (DBs you self-manage)?
├── → Generally prefer managed: Cloud SQL, Spanner, Firestore, BigQuery
└── Or VM + your own DB (rarely the right call)

ML inference?
├── Realtime, GPU?
│   └── → GKE (GPU node pools) OR Vertex AI online endpoints
└── Batch?
    └── → Vertex AI batch prediction OR Dataflow pipelines

Static frontend?
└── → Firebase Hosting OR Cloud Storage + Cloud CDN

API gateway?
├── In-VPC, internal-only?
│   └── → Internal HTTP(S) Load Balancer
├── Global edge, custom routing, WAF?
│   └── → External HTTP(S) Load Balancer + Cloud Armor
├── API management (rate limit, dev portal, monetization)?
│   └── → Apigee
```

See [references/gcp-services-reference.md](references/gcp-services-reference.md) for service-by-service depth: tiers, SLAs, limits, when to upgrade.

---

## Data store decision tree

```
Relational?
├── Standard OLTP, regional or multi-zone?
│   └── → Cloud SQL (MySQL / PostgreSQL / SQL Server)
├── Global, strong consistency, horizontal scale?
│   └── → Cloud Spanner (regional or multi-region)
├── Multi-region with high concurrency, fault-tolerant?
│   └── → Cloud Spanner (true multi-region active-active)

Document / NoSQL?
├── Mobile/web client-direct, real-time updates?
│   └── → Firestore (Native mode)
├── Schemaless, low-latency, regional or multi-region?
│   └── → Firestore OR Datastore (legacy Datastore Mode of Firestore)
├── Wide-column at massive scale, < 10ms reads?
│   └── → Bigtable

Key-value cache?
└── → Memorystore (Redis or Memcached)

Object storage?
└── → Cloud Storage (pick Standard / Nearline / Coldline / Archive)

Time-series / metrics?
├── Operational (Stackdriver-style)?
│   └── → Cloud Monitoring (built-in metric store)
├── Application time series?
│   └── → Bigtable OR BigQuery (depending on cardinality/query pattern)

Search?
├── Full-text on app data?
│   └── → Vertex AI Search OR self-managed Elasticsearch on GKE
└── Vector search for ML?
    └── → Vertex AI Vector Search OR pgvector on Cloud SQL OR Bigtable with vectors

Data warehouse?
└── → BigQuery (the answer to "should we use a warehouse?" on GCP)

Analytical OLAP?
└── → BigQuery (serverless) OR BigQuery + BigQuery BI Engine

Stream processing?
└── → Dataflow (Apache Beam) OR Pub/Sub + Dataflow
```

---

## Networking patterns

### Three core building blocks

| Component | What it does | When |
|-----------|--------------|------|
| **VPC** | L3 isolation; private IP space; global by default in GCP | Every non-trivial GCP deployment |
| **Private Service Connect (PSC)** | Brings managed services into your VPC privately | Default for production access to managed services |
| **Cloud Interconnect / VPN** | On-prem connectivity (Interconnect is dedicated; VPN is over internet) | Hybrid setups |

### Load balancers

| LB | When |
|----|------|
| **Global External HTTP(S) Load Balancer** | Global anycast; Cloud Armor; CDN; serverless backends |
| **Regional External HTTP(S) LB** | Regional only; cheaper for non-global workloads |
| **Internal HTTP(S) LB** | Internal services; supports serverless backends |
| **TCP/UDP Network LB** | L4 load balancing; lower cost; for non-HTTP workloads |
| **Internal TCP/UDP LB** | Internal L4 |

### Common networking patterns

| Pattern | What | When |
|---------|------|------|
| **Shared VPC** | Central host project owns VPC; service projects attach their resources | Enterprise / multi-team |
| **VPC peering** | Connect two VPCs (transitive routing not supported) | Multi-project organizations |
| **Private Service Connect** | Consumer endpoint in your VPC → producer service | Default for managed services |
| **Cloud Armor + global LB** | DDoS protection + WAF rules at the edge | Public-facing apps |
| **Hub-and-spoke via Network Connectivity Center** | Centralized routing for multi-VPC orgs | Large orgs |

---

## Identity patterns

### IAM, Service Accounts, Workload Identity Federation

| Concept | Use |
|---------|-----|
| **Cloud IAM** | Role-based access control for users, groups, service accounts |
| **Service Account (SA)** | Identity for an app or workload |
| **Service Account Key** | Static credential for SA — avoid in modern setups |
| **Workload Identity Federation** | Federated identity; on-prem / other-cloud workloads get GCP access without keys |
| **Workload Identity (GKE)** | K8s service accounts mapped to GCP SAs; no key mounting in pods |
| **Application Default Credentials (ADC)** | Standard library for auth; uses ambient credentials |

### Choosing identity

```
Workload running on GCP that calls other GCP services?
├── On GKE → GKE Workload Identity (KSA → GSA)
├── On Cloud Run / Functions → service identity (built-in)
├── On Compute Engine → instance service account
└── In a CI/CD pipeline outside GCP → Workload Identity Federation (no keys)

Workload outside GCP needing GCP access?
├── From AWS / Azure / OIDC provider → Workload Identity Federation
└── Last resort → Service Account key (rotate frequently)

User-facing auth?
└── Identity Platform (GCP's auth-as-a-service; or Firebase Auth for client-direct)
```

### Least-privilege IAM

GCP supports three forms:
- **Predefined roles** (e.g., `roles/storage.objectViewer`) — preferred
- **Custom roles** at organization or project — when predefined doesn't fit
- **Basic roles** (`owner`, `editor`, `viewer`) — too broad; avoid in production

Bind roles at the most specific scope:
- Resource → preferred
- Project → standard for project-scoped apps
- Folder → for organizational sub-tree
- Organization → only org-wide admins

---

## Google Cloud Architecture Framework (CAF)

GCP's framework has five pillars (same naming families as Azure/AWS but with Google flavor).

| Pillar | Core question |
|--------|---------------|
| **Operational Excellence** | Can the team operate, deploy, observe, and recover safely? |
| **Security, Privacy, and Compliance** | Can the workload defend, contain, recover, and meet regulatory needs? |
| **Reliability** | Will the workload remain available under expected and unexpected conditions? |
| **Cost Optimization** | Is the workload spending only what's needed for the value delivered? |
| **Performance Optimization** | Does it meet performance needs without over-provisioning? |

Use `scripts/gcp_caf_scorer.py --workload-config workload.yaml` to score against each pillar.

See [references/gcp-well-architected.md](references/gcp-well-architected.md) for the per-pillar deep dive: 10-question checklist per pillar, common findings, remediation patterns.

---

## Cost optimization

### Cost levers from biggest to smallest

| Lever | Typical savings | Effort |
|-------|----------------|--------|
| **Right-sizing** | 30-50% | Low |
| **Committed Use Discounts (CUDs) / Sustained Use Discounts (SUDs)** | 20-70% | Low (commitment) |
| **Autoscaling** | 20-40% | Medium |
| **Preemptible / Spot VMs** | up to 91% | Medium |
| **Storage class tiering** | 30-95% on storage | Low |
| **Egress reduction** (Cloud CDN; private peering) | Variable, large | Medium-High |
| **Decommission unused** | Variable | Low |
| **BigQuery slot reservations** | 30-50% on analytics | Medium |
| **Region choice** | 10-25% | High (move) |

### Cost anti-patterns

- **Premium service tiers by default.** Enterprise Spanner / large BigQuery on-demand / GKE Standard when Autopilot suffices.
- **No autoscaling.** Always provisioned at peak. Easy 30-40% savings.
- **Egress through public internet.** Multi-region without peering or Cloud CDN.
- **Logs / metrics retention at default 30+ days for all data.** Tiering needed.
- **BigQuery on-demand pricing for stable, high-query workloads.** Reserved slots beat on-demand at scale.
- **Preemptible VMs not used for batch / fault-tolerant workloads.** Up to 91% savings missed.
- **Public IPs forgotten.** Each costs a few dollars/mo; multiply by hundreds of orphans.

See [references/gcp-cost-optimization.md](references/gcp-cost-optimization.md) for the full lever catalog and detection patterns.

---

## End-to-end workflows

### Workflow: Design a new workload

1. **Understand requirements** — traffic, data scale, latency, region requirements, compliance.
2. **Pick compute** using the decision tree.
3. **Pick data stores** using the decision tree.
4. **Design networking** — VPC topology, PSC, LB pattern.
5. **Design identity** — Service Accounts, Workload Identity, IAM scopes.
6. **Plan observability** — Cloud Logging, Cloud Monitoring, Cloud Trace, Cloud Profiler.
7. **Estimate cost** with `scripts/gcp_cost_estimator.py`.
8. **Validate against CAF** with `scripts/gcp_caf_scorer.py`.
9. **Document** the architecture; share for review.

### Workflow: Review an existing GCP architecture

1. **Gather artifacts** — Terraform code, network diagrams, service inventory.
2. **Run the validator** — `scripts/gcp_architecture_validator.py --terraform ./infra/*.tf` flags structural issues.
3. **Run CAF scorer** with the workload's actual config.
4. **Identify high-cost components** with the cost estimator.
5. **Produce findings** by pillar with severity and recommendation.

### Workflow: Migrate from AWS / Azure to GCP

1. **Map services** — most have equivalents (SQS → Pub/Sub; SNS → Pub/Sub topics; Lambda → Cloud Functions / Cloud Run; DynamoDB → Bigtable or Firestore; S3 → Cloud Storage; RDS → Cloud SQL or Spanner).
2. **Re-evaluate the architecture** in GCP-native terms (BigQuery is often the right answer for analytics in ways no other cloud quite matches).
3. **Network parity** — VPC equivalent (global VPC is unique to GCP); IAM equivalent; private connectivity (PSC).
4. **Data migration** — Database Migration Service for many SQL scenarios; Storage Transfer Service for object data; Datastream for CDC.
5. **Cost re-estimate** — GCP pricing differs per-service; don't assume parity.

---

## Anti-patterns (GCP-specific)

- **Service Account keys committed to source control.** Use Workload Identity Federation everywhere possible.
- **Single-zone production** — use multi-zone or regional resources by default.
- **No org policies** — set up Org Policy constraints (e.g., disallowed services, allowed regions, no public IPs).
- **Compute Engine VMs with public IPs by default.** Use NAT Gateway + private IPs.
- **GCS bucket allUsers read** — almost never wanted; use IAM + signed URLs.
- **Default network in use** — delete the default VPC; create your own with explicit subnets.
- **BigQuery on-demand for known high-volume workloads.** Buy slot reservations.
- **Cloud SQL without HA** — single-zone DB is one zone outage from disaster.
- **Service account = same email as default Compute SA used everywhere.** Create distinct SAs per workload.
- **Firestore Native + Datastore mixed** — same project can't have both modes simultaneously; design once.
- **GKE Standard when Autopilot would work.** Autopilot eliminates node management; cheaper to operate.

---

## Tooling outputs

| Script | Input | Output |
|--------|-------|--------|
| `scripts/gcp_architecture_validator.py` | Terraform file or YAML workload spec | Structural issues, anti-pattern findings, missing best-practice settings |
| `scripts/gcp_cost_estimator.py` | YAML workload spec (services + tiers + scale) | Per-service monthly cost estimate, total, optimization opportunities |
| `scripts/gcp_caf_scorer.py` | YAML workload spec | Score per CAF pillar, gap analysis, recommendations |

All scripts: stdlib only, argparse CLI, JSON or markdown output.

---

## References

- [gcp-services-reference.md](references/gcp-services-reference.md) — per-service depth: tiers, SLAs, limits, when to upgrade
- [gcp-well-architected.md](references/gcp-well-architected.md) — 5-pillar CAF assessment with questions and remediations
- [gcp-cost-optimization.md](references/gcp-cost-optimization.md) — cost levers, anti-patterns, detection heuristics

---

## Related skills

- `engineering/senior-cloud-architect` — generic multi-cloud architecture patterns
- `engineering/aws-solution-architect` — AWS counterpart
- `engineering/azure-cloud-architect` — Azure counterpart
- `engineering/kubernetes-operator` — for GKE operator-pattern workloads
- `ra-qm-team/information-security-manager-iso27001` — compliance-mapped controls (GCP has Security Command Center)
- `ra-qm-team/soc2-compliance-expert` — GCP-specific SOC 2 evidence collection
