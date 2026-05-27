---
name: azure-cloud-architect
description: >
  Design, review, and validate Azure cloud architectures. Use when picking
  the right Azure compute (AKS / App Service / Container Apps / Functions /
  VMs / VMSS), data store (SQL DB / Cosmos / Postgres Flexible / Storage),
  networking (VNet / Private Endpoint / Front Door / Application Gateway),
  identity (Entra ID / Managed Identity / RBAC scopes), or applying the
  Azure Well-Architected Framework (Reliability, Security, Cost Optimization,
  Operational Excellence, Performance Efficiency) to a workload. Pairs with
  our existing senior-cloud-architect (multi-cloud, abstract patterns) by
  going deep on Azure-specific services, pricing, and operational defaults.
license: MIT + Commons Clause
metadata:
  version: 1.0.0
  author: borghei
  category: engineering
  domain: engineering
  updated: 2026-05-27
  tags: [azure, cloud-architecture, well-architected-framework, aks, app-service, entra-id, networking, cost-optimization]
---

# Azure Cloud Architect

End-to-end Azure-specific architecture: service selection, Well-Architected Framework assessment, identity and networking patterns, cost optimization, and operational defaults. Provider-specific complement to our generic `senior-cloud-architect` skill — that one covers cross-cloud patterns; this one knows AKS pricing tiers, when to pick Cosmos over SQL DB, and how Front Door differs from Application Gateway.

---

## When to use this skill

| Situation | Skill applies |
|-----------|---------------|
| Designing an Azure architecture from scratch | Yes — start with the **service selection decision tree** |
| Reviewing an existing Azure architecture | Yes — run **WAF assessment** via `scripts/azure_waf_scorer.py` |
| Validating an ARM/Bicep/Terraform plan | Yes — `scripts/azure_architecture_validator.py` |
| Estimating Azure cost for a workload | Yes — `scripts/azure_cost_estimator.py` |
| Picking between AKS / App Service / Container Apps / Functions | Yes — see **compute decision tree** |
| Setting up identity / RBAC / Managed Identity correctly | Yes — see **identity reference** |
| Designing a multi-region active-active or DR posture | Yes — see **reliability reference** |
| Picking SQL DB vs Cosmos vs Postgres Flexible vs Storage | Yes — see **data store decision tree** |
| Going to production without WAF review | Don't — run the WAF scorer first |

---

## Compute decision tree

Azure offers many compute options; picking the wrong one wastes money and adds operational burden.

```
Stateless HTTP service?
├── Need full control over OS / sidecar / custom runtime?
│   └── → AKS (Kubernetes; serious operational investment)
├── Standard web app (Node / Python / .NET / Java)?
│   ├── < 5 services, low traffic, want zero infra?
│   │   └── → App Service (Linux/Windows; managed PaaS)
│   ├── Need autoscale to zero + container runtime?
│   │   └── → Container Apps (KEDA-based; sweet spot for microservices)
│   └── Need true serverless / event-driven?
│       └── → Functions (Premium for VNet/always-warm; Consumption for cheap+bursty)
├── Batch / job processing?
│   └── → Container Apps Jobs OR Batch (HPC-scale)
└── Long-running stateful processes / legacy?
    └── → VMs / VMSS

Stateful service (DBs you self-manage)?
├── → Generally prefer managed: SQL DB, Cosmos, PG Flexible
├── Or VM + your own DB (rarely the right call)

ML inference?
├── Realtime, GPU?
│   └── → AKS (GPU node pools) OR Azure ML managed endpoints
└── Batch?
    └── → Azure ML pipelines OR Container Apps Jobs

Static frontend?
└── → Static Web Apps (auto SSL, GitHub/Bitbucket deploy)

API gateway?
├── In-VNet, internal-only?
│   └── → Application Gateway
├── Global edge, custom routing, WAF?
│   └── → Front Door
├── API management (rate limit, dev portal, monetization)?
│   └── → API Management
```

See [references/azure-services-reference.md](references/azure-services-reference.md) for service-by-service depth: pricing tiers, SLAs, limits, when to upgrade.

---

## Data store decision tree

```
Relational?
├── Standard OLTP, low-to-medium scale?
│   └── → Azure SQL Database (Single DB; serverless tier for spiky)
├── Want PostgreSQL specifically?
│   └── → Azure Database for PostgreSQL Flexible Server
├── Want MySQL specifically?
│   └── → Azure Database for MySQL Flexible Server
├── Multi-region with high concurrency, global reads?
│   └── → Cosmos DB (PostgreSQL or NoSQL API)

Document / NoSQL?
├── Global distribution, multiple consistency models, multi-region writes?
│   └── → Cosmos DB
├── Single-region key-value at scale?
│   └── → Azure Table Storage (cheap) OR Cosmos DB (Table API)

Key-value cache?
└── → Azure Cache for Redis (Standard for HA, Premium for VNet/persistence)

Blob / object storage?
└── → Storage Account (Blob); pick Hot / Cool / Cold / Archive tier per access pattern

Time-series / metrics?
├── Operational (low cost, queryable)?
│   └── → Azure Monitor Logs (Log Analytics workspace)
├── Application time series?
│   └── → Azure Data Explorer (Kusto) — purpose-built

Search?
└── → Azure AI Search (formerly Cognitive Search) — managed Lucene-based

Vector / embeddings?
├── Same workload as existing Cosmos / SQL DB?
│   └── → Cosmos DB (vector search support) OR Azure SQL DB (vector type)
└── Dedicated vector search?
    └── → Azure AI Search (vector indexes)

Data warehouse?
├── < 10TB, occasional queries?
│   └── → Azure SQL Database (DTU/vCore high tier) OR Synapse Serverless
└── > 10TB, analytical workloads?
    └── → Azure Synapse Analytics (Dedicated Pool) OR Microsoft Fabric
```

---

## Networking patterns

### Three core building blocks

| Component | What it does | When |
|-----------|--------------|------|
| **Virtual Network (VNet)** | L3 isolation; private IP space; subnets | Every non-trivial Azure deployment |
| **Private Endpoint** | Brings Azure PaaS services into your VNet via private IP | Default for production access to PaaS (Storage, SQL, etc.) |
| **Service Endpoint** | Lower-cost predecessor to Private Endpoint; restricts access to your VNet | Cost-sensitive, lower-security workloads only |

### Gateway services

| Service | Use when |
|---------|----------|
| **Application Gateway (v2)** | L7 LB inside VNet; WAF; OWASP rules; private + public modes; not global |
| **Front Door** | Global L7; multi-region routing; WAF at edge; CDN; URL-based routing |
| **Azure Firewall** | L3-L4-L7 stateful; egress filtering with FQDN allowlists |
| **NAT Gateway** | Predictable outbound IP for VNet egress; replaces SNAT exhaustion concerns |
| **VPN Gateway / ExpressRoute** | On-prem connectivity (S2S VPN cheaper; ER private + faster) |

### Common networking patterns

| Pattern | What | When |
|---------|------|------|
| **Hub-and-spoke** | Central hub VNet with shared services (firewall, ER, AD DS), peered with workload-specific spoke VNets | Multi-team / multi-workload orgs |
| **VWAN** | Microsoft-managed hub network simplifying multi-region + on-prem | When hub-spoke complexity gets painful |
| **Private Link for everything** | Every PaaS access via Private Endpoint; no public endpoints | Default for production / regulated workloads |
| **Azure Front Door + AppGw** | Global edge (AFD) → regional WAF/L7 (AGW) → backend | High-traffic global apps |

---

## Identity patterns

### Entra ID, Managed Identity, RBAC

| Concept | Use |
|---------|-----|
| **Microsoft Entra ID** (formerly Azure AD) | Identity provider for users, groups, apps |
| **Service Principal** | Identity for an app; has a secret or cert |
| **Managed Identity** | Service Principal whose lifecycle is tied to an Azure resource; no secrets |
| **System-assigned Managed Identity** | One per resource; deleted when resource is deleted |
| **User-assigned Managed Identity** | Independent lifecycle; can attach to multiple resources |
| **Workload Identity (AKS)** | Federated identity for K8s pods; no secret mounting |

### Choosing identity

```
Service running in Azure that calls other Azure services?
├── Single Azure resource → System-assigned MI
├── Multiple resources sharing identity (e.g., a pool of VMSS instances) → User-assigned MI
├── K8s pod → AKS Workload Identity
└── External app (CI/CD, on-prem) → Service Principal with cert (NOT secret)

Service calling external API (non-Azure)?
└── Service Principal OR app secret in Key Vault, accessed via MI

User-facing auth?
└── Entra ID with OIDC; B2C if customer-facing identity
```

### RBAC scopes (least privilege)

Assign RBAC at the smallest necessary scope:

| Scope | When |
|-------|------|
| Resource | Most specific; preferred default |
| Resource group | When the role applies to a logical group |
| Subscription | Only for subscription-wide admins |
| Management group | Cross-subscription enterprise governance |

Use built-in roles when they exist (Reader, Contributor, Storage Blob Data Reader, etc.). Custom roles only when truly needed.

---

## Azure Well-Architected Framework (WAF)

Microsoft's WAF has five pillars. Every architecture should be assessed against all five.

| Pillar | Core question |
|--------|---------------|
| **Reliability** | Will it stay up under expected and unexpected load? |
| **Security** | Can it defend against threats, contain breaches, recover from compromise? |
| **Cost Optimization** | Is it spending only what's needed for the value delivered? |
| **Operational Excellence** | Can the team deploy, observe, and recover changes safely? |
| **Performance Efficiency** | Does it meet performance needs without over-provisioning? |

Use `scripts/azure_waf_scorer.py --workload-config workload.yaml` to score a workload against each pillar with concrete questions and recommendations.

See [references/azure-well-architected.md](references/azure-well-architected.md) for the per-pillar deep dive: 10-question checklist per pillar, common findings, remediation patterns.

---

## Cost optimization

### Cost levers from biggest to smallest

| Lever | Typical savings | Effort |
|-------|----------------|--------|
| **Right-sizing** (matching SKU to load) | 30-50% | Low |
| **Reserved Instances / Savings Plans** | 20-72% | Low (commitment) |
| **Autoscaling** (compute + DB) | 20-40% | Medium |
| **Spot Instances** (for batch / tolerant workloads) | up to 90% | Medium |
| **Storage tiering** (Hot/Cool/Cold/Archive) | 30-70% on storage | Low |
| **Egress reduction** (Front Door caching; CDN; private peering) | Variable, can be huge | Medium-High |
| **Decommission unused** (orphan disks, idle resources) | Variable | Low |
| **Region choice** (cheaper regions for non-latency-sensitive) | 10-25% | High (move) |

### Cost anti-patterns

- **Premium tier by default.** Premium App Service / Cosmos throughput / SQL DTUs that the workload doesn't need.
- **No autoscaling.** Always provisioned at peak. Easy 30-40% savings if added.
- **Egress through public internet.** Cross-region traffic via public endpoints is expensive; use VNet peering or Private Link.
- **Unused Log Analytics retention.** Keeping 730 days of debug logs at default rate.
- **Cosmos DB autoscale ceiling too high.** Pays for unused capacity.
- **Premium-tier Managed Disks for non-critical workloads.** Standard SSD is usually enough.

See [references/azure-cost-optimization.md](references/azure-cost-optimization.md) for the full lever catalog and detection patterns.

---

## End-to-end workflows

### Workflow: Design a new workload

1. **Understand requirements** — traffic, data scale, latency, region requirements, compliance.
2. **Pick compute** using the decision tree.
3. **Pick data stores** using the decision tree.
4. **Design networking** — VNet topology, Private Endpoints, gateway pattern.
5. **Design identity** — Managed Identities, RBAC scopes.
6. **Plan observability** — Log Analytics workspace, Application Insights, metrics.
7. **Estimate cost** with `scripts/azure_cost_estimator.py`.
8. **Validate against WAF** with `scripts/azure_waf_scorer.py`.
9. **Document** the architecture; share for review.

### Workflow: Review an existing Azure architecture

1. **Gather artifacts** — ARM/Bicep/Terraform code, network diagrams, service inventory.
2. **Run the validator** — `scripts/azure_architecture_validator.py --bicep ./infra/*.bicep` flags structural issues.
3. **Run WAF scorer** with the workload's actual config.
4. **Identify high-cost components** with the cost estimator.
5. **Produce findings** by pillar with severity and recommendation.

### Workflow: Migrate from AWS / GCP to Azure

1. **Map services** — most have equivalents but not 1:1 (e.g., SQS → Service Bus or Storage Queue depending on use; SNS → Event Grid; Lambda → Functions; DynamoDB → Cosmos DB).
2. **Re-evaluate the architecture** in Azure-native terms before lift-and-shift; some patterns work better here, others worse.
3. **Network parity** — VPC equivalent (VNet); IAM equivalent (Entra + RBAC); private connectivity (Private Endpoint).
4. **Data migration** — Azure Database Migration Service for many SQL/PG scenarios; Storage Migration Service for files.
5. **Cost re-estimate** — Azure pricing differs from AWS / GCP per-service; don't assume parity.

---

## Anti-patterns (Azure-specific)

- **Storing secrets in App Settings / env vars instead of Key Vault.** Use Key Vault references in App Settings.
- **Single-region production** for a tier-1 workload. Use paired regions + Traffic Manager / Front Door.
- **No Resource Locks on critical resources.** A single `az group delete` away from disaster.
- **Public storage account access enabled** ("AllowBlobPublicAccess"). Disable at storage-account level unless explicitly needed.
- **Premium SKU for everything.** Premium SQL / Premium App Service / Premium Cosmos used by default without need.
- **Shared keys / connection strings** for service-to-service auth instead of Managed Identity.
- **No diagnostic settings** sending logs to Log Analytics → no audit trail.
- **Hub-and-spoke without UDRs to force egress through firewall.** Defeats the firewall's purpose.
- **AKS without managed identities, RBAC, or network policies** — provisioning Kubernetes the easy way leaves obvious gaps.
- **Functions on Consumption plan for VNet integration.** Consumption can't do VNet; need Premium or Dedicated.
- **Mixing Cosmos consistency levels** without understanding the trade-off; Session is the right default for most.

---

## Tooling outputs

| Script | Input | Output |
|--------|-------|--------|
| `scripts/azure_architecture_validator.py` | Bicep/ARM file or YAML workload spec | Structural issues, anti-pattern findings, missing best-practice settings |
| `scripts/azure_cost_estimator.py` | YAML workload spec (services + tiers + scale) | Per-service monthly cost estimate, total, optimization opportunities |
| `scripts/azure_waf_scorer.py` | YAML workload spec | Score per WAF pillar, gap analysis, recommendations |

All scripts: stdlib only, argparse CLI, JSON or markdown output.

---

## References

- [azure-services-reference.md](references/azure-services-reference.md) — per-service depth: tiers, SLAs, limits, when to upgrade
- [azure-well-architected.md](references/azure-well-architected.md) — 5-pillar WAF assessment with questions and remediations
- [azure-cost-optimization.md](references/azure-cost-optimization.md) — cost levers, anti-patterns, detection heuristics

---

## Related skills

- `engineering/senior-cloud-architect` — generic multi-cloud architecture patterns
- `engineering/aws-solution-architect` — AWS counterpart
- `engineering/gcp-cloud-architect` — GCP counterpart
- `engineering/kubernetes-operator` — for AKS operator-pattern workloads
- `ra-qm-team/information-security-manager-iso27001` — for compliance-mapped controls (Azure has built-in Defender / Compliance Manager)
- `ra-qm-team/soc2-compliance-expert` — Azure-specific SOC 2 evidence collection
