---
name: kubernetes-operator
description: >
  Design, build, and operate Kubernetes operators using the operator pattern.
  Use when extending Kubernetes with a custom controller for a domain object
  (database, message queue, ML pipeline, internal platform primitive), deciding
  between operator-SDK / Kubebuilder / controller-runtime / metacontroller,
  designing CRDs with proper schema and conversion, implementing reconciliation
  loops that are idempotent and converge, building status subresources for
  observability, or auditing an existing operator for anti-patterns
  (leaky abstractions, missing finalizers, no leader election, status as spec).
license: MIT + Commons Clause
metadata:
  version: 1.0.0
  author: borghei
  category: engineering
  domain: engineering
  updated: 2026-05-27
  tags: [kubernetes, operator-pattern, crd, controller-runtime, kubebuilder, platform-engineering, sre]
---

# Kubernetes Operator

End-to-end Kubernetes operator design and construction. Covers the operator pattern (control loops for stateful workloads), CRD design (schema, validation, conversion, status), the reconciliation loop (idempotency, convergence, level-triggered vs edge-triggered), framework selection (controller-runtime / Kubebuilder / operator-SDK / metacontroller), and the operational concerns (finalizers, leader election, RBAC scoping, status subresource, observability).

This skill works with Go-based operators (the dominant ecosystem) and notes when alternative languages or frameworks (Python / KOPF, Java / JOSDK) apply.

---

## When to use this skill

| Situation | Skill applies |
|-----------|---------------|
| Building a new operator for an internal platform primitive | Yes — start with the **operator pattern decision** |
| Auditing an existing operator for production-readiness | Yes — use **anti-patterns** + `scripts/reconciliation_audit.py` |
| Designing CRDs for a custom resource | Yes — use **CRD design** + `scripts/crd_validator.py` |
| Deciding "operator vs Helm chart vs plain manifests" | Yes — use the decision matrix below |
| Scaffolding a new operator project | Yes — `scripts/operator_scaffold.py` |
| Debugging a controller that "isn't reconciling" | Yes — use **reconciliation troubleshooting** |
| Just running someone else's operator (Postgres, Kafka, etc.) | Partially — useful for understanding what it does and how to monitor it |

---

## When NOT to write an operator

Operators are extensions of Kubernetes. They cost ongoing maintenance, RBAC review, security audit, version-skew handling, and (often) a dedicated SRE rotation. Avoid if:

- Your resource is **stateless** and fits CronJob + Deployment + ConfigMap (no extra control loop needed)
- You can model the workload as a Helm chart with values overrides
- An existing community operator (CNCF / vendor) covers your needs — adopt it instead
- The lifecycle has < 5 transitions (then a Job + readiness probe is simpler)
- You'd be the only consumer (consider a less heavyweight extension point — admission webhook, Lua/JSONNet, GitOps controller config)

A useful rule: write an operator only when the resource has at least 2-3 non-trivial state transitions AND existing primitives can't model them cleanly.

---

## The operator pattern in one paragraph

An operator is a **controller** (Kubernetes control-loop program) that watches **CustomResources** (CRs) representing application-specific state, and drives the cluster toward the **desired state** declared in the CR's spec, recording **observed state** in the CR's status. The pattern: a domain expert encodes operational knowledge as software — "when CR X is created, do A; when X.spec.replicas changes, do B; when underlying Pod fails, do C" — so users get a declarative API instead of a wiki of runbooks.

Three building blocks:

1. **CRD** (CustomResourceDefinition) — schema for the new resource type
2. **Controller** — control loop that reconciles spec → state
3. **Custom Resource** (CR) — instances users create to ask for things

---

## Operator vs alternatives — decision matrix

| Need | Use |
|------|-----|
| Run a stateless app | Deployment + Service |
| Run a stateful app with simple lifecycle | StatefulSet + headless Service |
| Manage configuration drift | Argo CD / Flux (GitOps controllers) |
| Add validation/mutation to existing K8s API | Admission webhook (no operator) |
| Manage external resources from inside cluster | Operator (if non-trivial), OR Crossplane (if matches its model) |
| Automate complex stateful workloads with domain expertise | **Operator** |
| Multi-cluster federation | Operator + push to other clusters (e.g., Karmada, KubeFed) |
| Simple "do X on YAML change" | Metacontroller (declarative composition, no Go) |

---

## CRD design — the foundation

A bad CRD = perpetual operator pain. Spend time here.

### Required design decisions

| Decision | Options | Default |
|----------|---------|---------|
| Scope | Namespaced / Cluster | Namespaced (unless cluster-wide makes no sense without it) |
| Versioning | v1alpha1 / v1beta1 / v1 | Start v1alpha1, graduate per Kubernetes API conventions |
| Conversion | None / Webhook / None-with-storage-version | Webhook once you have > 1 served version |
| Subresources | status / scale / both | Always enable `status`; `scale` if user-controlled replicas |
| Validation | OpenAPI schema / admission webhook / both | OpenAPI for shape; webhook for cross-field |
| Printer columns | Yes (recommended) | Always — kubectl get is much nicer |
| Categories | Optional | Add for grouping (e.g., `kubectl get all,databases`) |
| Short names | Optional | Add (e.g., `db` for `Database`) |

### Spec / Status separation

**Spec** = what the user wants (desired state)
**Status** = what the operator observes (actual state)

The user writes Spec. The operator writes Status. Never the other way round.

```yaml
apiVersion: example.com/v1
kind: Database
metadata:
  name: prod-orders
spec:                          # user-owned
  version: "14.10"
  storage: 100Gi
  replicas: 3
  backup:
    schedule: "0 2 * * *"
status:                        # operator-owned
  phase: Running
  observedGeneration: 5
  conditions:
    - type: Available
      status: "True"
      reason: AllReplicasReady
      lastTransitionTime: "2026-05-27T08:00:00Z"
  endpoints:
    primary: prod-orders-primary.default.svc.cluster.local
    replicas: ["prod-orders-replica-0.default.svc.cluster.local"]
  observedVersion: "14.10"
```

### CRD schema rules

- **Use OpenAPI v3 schema**, structural (no `x-kubernetes-preserve-unknown-fields` unless you really need it)
- **Validate at the API edge** — required fields, enum values, format, regex pattern, min/max
- **Use defaults sparingly** — every default is something users can't tell the operator "I don't care, you decide"
- **Add descriptions** — they show up in `kubectl explain`, which is how users learn your API
- **Use `x-kubernetes-validations`** (CEL, K8s 1.25+) for cross-field validation that doesn't need a webhook
- **Version your API thoughtfully** — v1alpha1 can break; v1 is forever

See [references/operator-pattern-and-crds.md](references/operator-pattern-and-crds.md) for the full schema design guide including conversion webhooks, status subresource semantics, and the printer-column DSL.

---

## The reconciliation loop

The control loop is the heart of an operator. It runs whenever a watched resource changes (or periodically).

```
loop {
  obj = fetch(CR_key)
  if (obj is being deleted) {
    handle_finalizer(obj)
    return
  }
  desired = compute_desired_state(obj.spec)
  actual = observe_actual_state(cluster)
  if (desired != actual) {
    apply_diff(desired, actual)
  }
  update_status(obj, actual)
  if (transient_error) requeue_after(backoff)
}
```

### Five rules of reconciliation

1. **Idempotent.** Running reconcile twice with the same inputs produces the same effect. No "create or fail" — always "ensure exists."
2. **Level-triggered, not edge-triggered.** Reconcile from current state, not from the diff of what changed. Don't say "user changed X, so increment Y" — say "Y should equal f(X), check and set if needed."
3. **Converge.** Each reconcile gets closer to desired state, or stays there. Doesn't oscillate.
4. **Fail safe.** If something errors, requeue with exponential backoff. Don't crash; don't loop tight.
5. **Observed generation tracking.** Status.observedGeneration tells users "yes, I saw your latest spec change."

### Common reconciliation patterns

| Pattern | When |
|---------|------|
| **Direct apply** | Stateless dependent resources (Deployments, Services, ConfigMaps owned by this CR) |
| **Phase machine** | Multi-step lifecycle with explicit phases (Pending → Bootstrapping → Running → Updating → Terminating) |
| **Sub-controllers** | One controller per logical concern (e.g., one for the StatefulSet, one for backups, one for network policies) |
| **Owner references** | All child resources have OwnerReference back to the CR → garbage collection is automatic on delete |

See [references/controller-runtime-patterns.md](references/controller-runtime-patterns.md) for Go-flavored controller-runtime examples per pattern, plus error handling, requeueing strategies, and indexer use.

---

## Framework selection

The three dominant Go frameworks (and others):

| Framework | Use when | Skip when |
|-----------|----------|-----------|
| **controller-runtime** | You want full control; building a complex multi-controller operator; library, not framework | You want batteries-included scaffolding |
| **Kubebuilder** | Greenfield operator; want scaffolding, conventions, makefile, CI templates | You don't want code generation in your repo |
| **operator-SDK** | Came from Operator Lifecycle Manager world; want OLM packaging | OLM isn't your target |
| **Metacontroller** | Operator logic fits a simple "given parent + children YAML → output YAML" model, written in any language | You need fine-grained control or complex state |
| **KOPF (Python)** | Team is Python-first; operator is mostly orchestration, not perf-critical | Need maximum K8s API surface |
| **JOSDK (Java)** | Team is Java-first; integrating with Java ecosystem libs | Same as above |
| **kube-rs (Rust)** | Team is Rust-first; perf or reliability concerns where Go isn't enough | No Rust expertise on the team |

**Default recommendation:** Kubebuilder for greenfield operators. It uses controller-runtime under the hood, gives you scaffolding without lock-in, and you can drop into raw controller-runtime any time.

---

## Operational concerns

### Finalizers

Without finalizers, when a CR is deleted, the controller doesn't get to clean up external resources (database in cloud, DNS record, S3 bucket).

Pattern:
1. On CR create, add finalizer string to `metadata.finalizers`
2. On CR delete (when `metadata.deletionTimestamp` is set), do cleanup
3. After successful cleanup, remove the finalizer
4. K8s sees no finalizers, completes the delete

Anti-pattern: finalizer that can't complete (cleanup waits on something unavailable). User can't delete the CR; must manually edit out the finalizer (`kubectl patch`). Always have a max-retry / timeout for cleanup.

### Leader election

With > 1 controller replica (for HA), only one should reconcile at a time. Use lease-based leader election (built into controller-runtime).

```go
mgr, err := ctrl.NewManager(cfg, ctrl.Options{
    LeaderElection:          true,
    LeaderElectionID:        "myop.example.com",
    LeaderElectionNamespace: "myop-system",
})
```

Without leader election, two controllers fight, status flaps, you've created your own chaos.

### RBAC scoping

Default RBAC generated by Kubebuilder is wide ("get/list/watch/create/update/delete/patch on everything in my group + my CRD"). Tighten it:

- **Get/list/watch** on resources the controller observes
- **Create/update/patch** only on resources the controller manages
- **Delete** only on resources it owns
- **No `*` verbs** unless absolutely necessary
- **Cluster-scoped vs namespace-scoped** carefully — many operators don't need cluster scope

A controller with `cluster-admin` RBAC is a privilege-escalation vector.

### Status subresource

Always enable status as a subresource:

```yaml
versions:
- name: v1
  served: true
  storage: true
  subresources:
    status: {}
```

Why:
- `kubectl get -o yaml` shows status without ambiguity
- Updates to status don't change spec's `resourceVersion` (no infinite reconcile loop)
- RBAC can grant status-only updates separately

### Observability

Operators are silent until they aren't. Wire metrics:

- `reconcile_total{controller="db"}` — count of reconciliations
- `reconcile_errors_total{controller="db"}` — errored reconciliations
- `reconcile_duration_seconds{controller="db"}` — histogram
- `workqueue_depth{controller="db"}` — backlog
- `workqueue_unfinished_work_seconds{controller="db"}` — pending work age
- `controller_runtime_active_workers{controller="db"}` — concurrency

Controller-runtime exposes these via the `/metrics` endpoint. Scrape with Prometheus.

Logging:
- Structured (logr / zap)
- Include `namespace/name` of the reconciled CR
- Include `reconcileID` (UUID per reconciliation, threading through sub-calls)
- WARN/ERROR on requeue with reason
- INFO on phase transitions

---

## End-to-end workflows

### Workflow: Scaffold a new operator

1. **Decide.** Confirm operator is the right pattern (see anti-section above).
2. **Bootstrap.**
   ```
   kubebuilder init --domain example.com --repo example.com/db-operator
   kubebuilder create api --group example.com --version v1alpha1 --kind Database
   ```
   Or use `scripts/operator_scaffold.py --name db-operator --group example.com --kind Database` for our pre-templated structure including stricter RBAC, observability, and finalizer scaffolding.
3. **Design CRD.** Write the spec/status schema; validate with `scripts/crd_validator.py`.
4. **Implement controller.** Start with: fetch → handle delete (finalizer) → reconcile children → update status.
5. **Add tests.** Use envtest (sets up a real K8s API server for tests).
6. **Wire observability.** Prometheus metrics, structured logging.
7. **Document.** README with example CR, available fields, status meanings, common errors.

### Workflow: Audit an existing operator

1. **Run** `scripts/reconciliation_audit.py --controller-path ./internal/controllers --crd ./config/crd/bases/*.yaml`.
2. **Review** flagged anti-patterns:
   - Missing finalizer + creates external resources
   - No leader election + replicas > 1
   - Status writes spec fields
   - Reconcile is not idempotent (creates duplicates on retry)
   - No requeue backoff (tight loop on error)
   - Wide RBAC verbs / wide scopes
   - Missing observed generation tracking
3. **File issues** for each finding.
4. **Re-audit** after fixes.

### Workflow: Design a CRD

1. **Sketch** the spec: minimum fields the user must provide to make sense of the resource.
2. **Sketch** the status: what the operator will report back.
3. **Validate** with `scripts/crd_validator.py --schema my-crd.yaml`. Output: warnings on missing descriptions, dangerous fields (preserve-unknown), missing printer columns, etc.
4. **Add OpenAPI validation:** required, enum, pattern, min/max.
5. **Add `x-kubernetes-validations`** for cross-field rules.
6. **Add printer columns:** at minimum, `Phase` and `Age`.
7. **Add `kubectl explain` descriptions** for every field.
8. **Iterate** with users — read the YAML they write, identify confusion.

### Workflow: Upgrade CRD versions

The v1alpha1 → v1 upgrade is one of the most error-prone parts of operator development.

1. **Add the new version** (v1beta1) alongside v1alpha1. Both `served: true`. Pick the **storage version** carefully (whichever you'd rather have in etcd).
2. **Write a conversion webhook** if fields differ. Or use "round-tripping" conversion — write to the storage version, read from the served version.
3. **Test conversion both directions** with test cases.
4. **Deploy** the new operator with both versions served.
5. **Migrate** users to v1beta1 (update tooling, docs, examples).
6. **Remove** v1alpha1 (`served: false`, then later remove from CRD).

Don't delete an old version while users are still writing it.

---

## Anti-patterns

- **Status fields in spec.** User writes "phase: Running", operator never overrides — chaos. Status is operator-only.
- **No finalizer + external resources.** User deletes CR, cloud resources leak.
- **No leader election + multiple replicas.** Two controllers race; status flaps.
- **Tight loop on error.** Reconcile returns error, K8s requeues immediately, infinite spin. Always backoff.
- **Reconcile creates resources directly with name = CR name (no ownerRef).** Garbage collection won't clean up; orphaned resources accumulate.
- **Spec field is array; controller appends instead of setting.** Each reconcile grows the array; eventually hits etcd object size limit.
- **Cross-CR coupling without watchers.** Controller for A reads B; if B changes, A doesn't reconcile until something else triggers it.
- **Wide RBAC.** `cluster-admin` for convenience; massive blast radius.
- **No status conditions.** User has no way to tell why a CR is failing.
- **Operator that requires `kubectl edit` for normal operations.** Defeats the declarative API.
- **No observed generation.** User can't tell if their spec change has been processed.
- **Operator that mutates Pods directly.** Bypasses StatefulSet/Deployment semantics; gets very confusing very fast.

---

## Tooling outputs

| Script | Input | Output |
|--------|-------|--------|
| `scripts/crd_validator.py` | Path to one or more CRD YAML files | Markdown report of schema issues, missing descriptions, dangerous fields, printer column suggestions |
| `scripts/operator_scaffold.py` | Operator name, group, kind, namespace-scope | Bootstrapped project structure with stricter RBAC, observability, finalizer skeleton |
| `scripts/reconciliation_audit.py` | Controller source directory (Go) + CRD YAMLs | Findings: missing finalizers, no leader election, tight loops, wide RBAC, status anti-patterns |

All scripts: stdlib only, argparse CLI, JSON or markdown output.

---

## References

- [operator-pattern-and-crds.md](references/operator-pattern-and-crds.md) — pattern fundamentals, CRD schema design, versioning, conversion, status subresource
- [controller-runtime-patterns.md](references/controller-runtime-patterns.md) — Go controller-runtime examples, reconciliation patterns, finalizers, leader election, indexers
- [operator-anti-patterns.md](references/operator-anti-patterns.md) — the full anti-pattern catalog with detection heuristics and fixes

---

## Related skills

- `engineering/chaos-engineering` — chaos-test operators (kill the controller, partition from API server)
- `engineering/observability-designer` — wire metrics + logging for operators
- `engineering/incident-commander` — operators amplify blast radius; incident response matters more
- `engineering/feature-flags-architect` — operators with `spec.feature.<x>.enabled` fields effectively become flag systems; consider the trade-off
