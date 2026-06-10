## 11 — Priority Classes

### What it is
A `PriorityClass` is a **cluster-scoped** (non-namespaced) object that assigns a numeric priority value to a name. Pods reference a `PriorityClass` by name to declare their scheduling priority. The scheduler uses this priority to determine both **scheduling order** and **preemption behaviour** — whether a high-priority Pod is allowed to evict lower-priority Pods to claim their resources.

---

### Priority Value Ranges

| Range | Who uses it |
|---|---|
| Up to **~2 billion** | Internal Kubernetes system components (control plane Pods) |
| **1 billion to -2 billion** | User workloads and application Pods |
| **0** | Default priority for any Pod with no `priorityClassName` set |

> A **larger number = higher priority**. System-critical classes (`system-cluster-critical`, `system-node-critical`) are pre-installed and sit near the 2 billion ceiling — ensuring control plane components always outrank user workloads.

```bash
# View built-in priority classes
kubectl get priorityclass

# Output includes:
# system-cluster-critical   ~2000000000   false
# system-node-critical      ~2000001000   false
```

---

### Creating a PriorityClass

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000          # higher number = higher priority
globalDefault: false    # if true, applied to all Pods with no priorityClassName
description: "Used for critical application workloads."
```

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 1000
globalDefault: true     # default for all Pods that don't specify a class
description: "Default priority for background workloads."
```

> **`globalDefault: true` can only be set on one PriorityClass in the entire cluster.** Creating a second one with `globalDefault: true` is rejected by the API server.

---

### Assigning a PriorityClass to a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
spec:
  priorityClassName: high-priority    # references the PriorityClass by name
  containers:
    - name: app
      image: critical-app:v1
```

When no `priorityClassName` is set, the Pod gets priority `0` — unless a `globalDefault` PriorityClass exists, in which case that value is used instead.

#### Inspecting Priority Classes Across Pods

```bash
# Compare priority classes assigned to all Pods at a glance
kubectl get pods -o custom-columns="NAME:.metadata.name,PRIORITY:.spec.priorityClassName"

# Across all namespaces
kubectl get pods -A -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,PRIORITY:.spec.priorityClassName"
```

---

### Scheduling Behaviour

The scheduler processes the **scheduling queue in priority order** — higher priority Pods are scheduled before lower priority ones. When resources are insufficient, the scheduler evaluates **preemption**:

```
New high-priority Pod arrives → no resources available
        │
        ▼
Scheduler checks preemptionPolicy of the new Pod's PriorityClass
        │
        ├── PreemptLowerPriority (default)
        │     → evicts one or more lower-priority Pods to free resources
        │     → high-priority Pod takes their place
        │
        └── Never
              → Pod does NOT evict existing workloads
              → Pod waits in the scheduling queue
              → Still gets higher queue position than other waiting lower-priority Pods
```

---

### Preemption Policy

| `preemptionPolicy` value | Behaviour |
|---|---|
| `PreemptLowerPriority` (default) | Pod can evict lower-priority running Pods to claim resources |
| `Never` | Pod cannot evict any running Pod — waits in queue but with higher scheduling priority than lower-priority waiting Pods |

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-high
value: 500000
preemptionPolicy: Never       # will not evict running Pods
globalDefault: false
description: "High-priority batch jobs that should not disrupt running workloads."
```

---

### Real-World Usage

| Scenario | Priority Class | preemptionPolicy |
|---|---|---|
| **Control plane components** — must always run | `system-cluster-critical` (built-in) | `PreemptLowerPriority` |
| **Production databases** — must not be displaced by new workloads | High value (e.g. `900000`) | `PreemptLowerPriority` |
| **Critical user-facing APIs** — must get scheduled immediately even at cost of lower workloads | High value (e.g. `800000`) | `PreemptLowerPriority` |
| **Batch / background jobs** — important but should not kill running workloads | Medium value, `Never` | `Never` |
| **Dev / test workloads** — lowest priority, first to be evicted | Low value (e.g. `1000`) | `PreemptLowerPriority` |

---

### When to Use Which

| Need | Approach |
|---|---|
| Ensure critical workloads always get scheduled first | Assign a high-value `PriorityClass` with `PreemptLowerPriority` |
| Protect running workloads from being evicted by new arrivals | Assign a high-value `PriorityClass` with `Never` to the new workload |
| Set a sensible cluster-wide default for Pods with no explicit priority | Create one `PriorityClass` with `globalDefault: true` |
| Separate control plane priority from application priority | Use built-in `system-cluster-critical` / `system-node-critical` for control plane — never assign these to user workloads |

---

### Exam Gotchas

- **`PriorityClass` is cluster-scoped** — it is not created inside a namespace. `kubectl get priorityclass` not `kubectl get priorityclass -n default`.
- **`globalDefault: true` is a singleton** — only one PriorityClass in the entire cluster can have this set. A second one will be rejected.
- **Default priority is `0`, not the lowest possible** — the range goes down to `-2 billion`. You can explicitly assign negative priorities to workloads you want preempted first.
- **`preemptionPolicy: Never` does not mean lowest priority** — the Pod still gets a higher queue position than lower-priority waiting Pods. It only means it will not evict already-running Pods.
- **Preemption evicts entire Pods, not containers** — when the scheduler preempts, it kills the lowest-priority Pod(s) to free enough resources. The evicted Pod may be rescheduled elsewhere if resources exist.
- **API version** — `scheduling.k8s.io/v1`, not `v1`. Using the wrong API version is a common manifest error.

---

