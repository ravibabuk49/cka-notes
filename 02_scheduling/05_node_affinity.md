## 05 — Node Affinity

### What it is
Node Affinity lets you tell the Kubernetes scheduler **which nodes a Pod is allowed to land on**, using expression-based rules. It is the advanced replacement for `nodeSelector` — it does the same job but supports OR logic, exclusions, and soft preferences that `nodeSelector` cannot express.

---

### Why Node Affinity Exists

`nodeSelector` only does exact label matching. The moment your requirement goes beyond a single label, it falls short:

| Requirement | nodeSelector | Node Affinity |
|---|---|---|
| Place on `large` node | ✅ | ✅ |
| Place on `large` OR `medium` node | ❌ | ✅ |
| Place on any node that is NOT `small` | ❌ | ✅ |
| Prefer a node but don't hard-require it | ❌ | ✅ |

---

### What It Looks Like

The same goal — place a Pod on a `large` node — expressed in both ways:

```yaml
# nodeSelector — simple but limited
nodeSelector:
  size: large
```

```yaml
# Node Affinity — more verbose, but fully expressive
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: size
              operator: In
              values:
                - large
```

Both achieve the same result here. Node Affinity just opens the door for more complex rules.

---

### Operators

The `operator` field is what gives Node Affinity its power over `nodeSelector`:

| Operator | What it means |
|---|---|
| `In` | Node label value must be one of the listed values |
| `NotIn` | Node label value must NOT be any of the listed values |
| `Exists` | Node must have the label key, value does not matter |
| `DoesNotExist` | Node must NOT have the label key at all |
| `Gt` | Node label value must be greater than our specified number |
| `Lt` | Node label value must be less than our specified number |

```yaml
matchExpressions:
  # Large OR medium — using In with multiple values
  - key: size
    operator: In
    values:
      - large
      - medium

  # Anything that is NOT small
  - key: size
    operator: NotIn
    values:
      - small

  # Any node that has a 'size' label at all — value is irrelevant
  - key: size
    operator: Exists
    # no values field — omit it entirely when using Exists
```

> **AND vs OR logic:**
> - Multiple expressions inside one `nodeSelectorTerms` block → **AND** (all must match)
> - Multiple `nodeSelectorTerms` blocks → **OR** (any one matching is enough)

---

### Affinity Types — The Long Property Name Explained

The type name under `nodeAffinity` looks intimidating but it is just two questions joined together:

```
required/preferred   DuringScheduling
ignored/required     DuringExecution
```

**DuringScheduling** — what should the scheduler do when the Pod is first being placed?
- `required` → must find a matching node, otherwise Pod stays `Pending`
- `preferred` → try to find a matching node, but place on any node if none found

**DuringExecution** — what should happen to a running Pod if a node label changes after scheduling?
- `ignored` → nothing, the Pod keeps running (this is the behaviour today)
- `required` → evict the Pod if the node no longer matches (planned, not yet available)

#### Available Types Today

| Type | At Scheduling | After Scheduling |
|---|---|---|
| `requiredDuringSchedulingIgnoredDuringExecution` | Hard requirement — Pod stays `Pending` if no match | No action — Pod keeps running even if node label changes |
| `preferredDuringSchedulingIgnoredDuringExecution` | Soft preference — falls back to any node if no match | No action — Pod keeps running even if node label changes |

#### Planned (not yet available)

| Type | At Scheduling | After Scheduling |
|---|---|---|
| `requiredDuringSchedulingRequiredDuringExecution` | Hard requirement | Pod is **evicted** if node label changes and no longer matches |

---

### How to Choose the Right Type

```
Does the workload NEED a specific node to function correctly?
  │
  ├── YES → required
  │         Pod waits in Pending until a matching node exists.
  │         Example: GPU training job, SSD-backed database.
  │
  └── NO  → preferred
            Pod schedules anywhere if no match is found.
            Example: batch job that prefers spot nodes but can run on on-demand.
```

---

### Full Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-processor
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: size
                operator: In
                values:
                  - large
                  - medium
  containers:
    - name: processor
      image: data-processor:v1
```

---

### Weight — Only for `preferred`

When using `preferred`, you can assign a `weight` (1–100) to each rule. The scheduler scores candidate nodes — higher weight means stronger preference:

```yaml
preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 80              # strongly prefer large nodes
    preference:
      matchExpressions:
        - key: size
          operator: In
          values:
            - large
  - weight: 20              # weakly prefer medium nodes
    preference:
      matchExpressions:
        - key: size
          operator: In
          values:
            - medium
```

---

### Real-World Usage

| Scenario | Type | Rule |
|---|---|---|
| ML training must run on GPU nodes | `required` | `key: gpu, operator: Exists` |
| Batch jobs prefer spot nodes, fall back to on-demand | `preferred` | `key: node-lifecycle, operator: In, values: [spot]` |
| Pods must stay in a specific availability zone for compliance | `required` | `key: topology.kubernetes.io/zone, operator: In, values: [eastus-1]` |
| Avoid placing memory-intensive Pods on small nodes | `required` | `key: size, operator: NotIn, values: [small]` |
| Route inference workloads to a dedicated AKS GPU node pool | `required` | `key: agentpool, operator: In, values: [gpupool]` |

---

### Exam Gotchas

- **Pod stuck in `Pending`** with node affinity set → first suspect is `required` type with no matching node. Check node labels with `kubectl get nodes --show-labels`.
- **`Exists` operator — no `values` field** — including a `values` field with `Exists` causes a validation error.
- **Node label removed at runtime** — today both available types have `IgnoredDuringExecution`, so a running Pod is never evicted due to a node label change.
- **Node Affinity only attracts — it does not repel.** It cannot stop other Pods from landing on the same node. For that, you need **Taints**. Production setups combine both.
- **`nodeSelectorTerms` = OR, `matchExpressions` = AND** — this logic distinction is directly exam-testable.

---

### In Managed Clusters (AKS / EKS / GKE)

- AKS auto-injects topology labels on every node — `topology.kubernetes.io/zone`, `topology.kubernetes.io/region`, `agentpool` — usable in affinity rules without any manual labelling.
- AKS spot node pools carry `kubernetes.azure.com/scalesetpriority=spot` — use `preferred` affinity with this label to route batch workloads to spot nodes with on-demand fallback.
- Combining `preferred` node affinity for a user node pool with the system pool's `CriticalAddonsOnly` taint is the standard AKS pattern to keep workloads off system pools without hard-failing when the user pool is full.

---

