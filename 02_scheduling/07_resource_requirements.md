## 07 — Resource Requirements

### What it is
Resource requirements define how much **CPU and memory** a container needs and is allowed to use. They consist of two settings:

- **Request** — the minimum guaranteed amount of CPU/memory the scheduler reserves for the container on a node. The scheduler only places a Pod on a node that has at least this much available.
- **Limit** — the maximum amount of CPU/memory a container is allowed to consume. What happens when a container exceeds this differs between CPU and memory.

---

### Units

#### CPU
| Notation | Meaning |
|---|---|
| `1` | 1 vCPU (= 1 AWS vCPU = 1 Azure/GCP core = 1 Hyperthread) |
| `0.1` | 100 millicores |
| `100m` | 100 millicores (same as `0.1`) |
| `1m` | 1 millicore — the minimum allowed value |

#### Memory
| Suffix | Meaning |
|---|---|
| `256Mi` | 256 Mebibytes (1 Mi = 1024 KB) |
| `256M` | 256 Megabytes (1 M = 1000 KB) |
| `1Gi` | 1 Gibibyte = 1024 MB |
| `1G` | 1 Gigabyte = 1000 MB |

> **`Mi` vs `M` matters** — they are not the same value. Use `Mi`/`Gi` for precise binary sizing (standard in Kubernetes manifests).

---

### Defining Requests and Limits

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-processor
spec:
  containers:
    - name: processor
      image: data-processor:v1
      resources:
        requests:
          memory: "1Gi"      # scheduler reserves this on the node
          cpu: "500m"        # 0.5 vCPU guaranteed
        limits:
          memory: "2Gi"      # container cannot exceed this
          cpu: "1"           # container cannot exceed 1 vCPU
```

> Requests and limits are set **per container**, not per Pod. Each container in a multi-container Pod has its own `resources` block.

---

### What Happens When a Container Hits Its Limit

| Resource | Behaviour at limit |
|---|---|
| **CPU** | Throttled — the container is slowed down but keeps running. It cannot use more than its CPU limit. |
| **Memory** | The container can temporarily exceed its limit, but if it **consistently** exceeds it, the Pod is **OOMKilled** (Out Of Memory Killed) and restarted. |

```bash
# Check if a Pod was OOMKilled
kubectl describe pod <pod-name>
# Look for: Last State: OOMKilled or Exit Code: 137
```

---

### Four Scenarios — Requests and Limits Combinations

#### 1. No requests, no limits (default)
Any Pod can consume all CPU and memory on a node, starving other Pods. **Not recommended.**

#### 2. No requests, limits set
Kubernetes automatically sets requests equal to limits. Each Pod gets a guaranteed and fixed amount — no bursting allowed. Predictable but inflexible.

```
requests == limits == 3 vCPU  →  Pod gets exactly 3, no more, no less
```

#### 3. Requests and limits both set
Each Pod gets its guaranteed request and can burst up to the limit. Looks ideal but unnecessarily caps Pods even when spare capacity exists.

```
requests = 1 vCPU, limits = 3 vCPU
Pod A needs 3 but is capped at 3.
Pod B is idle at 0.2 but its spare capacity cannot be used by Pod A.
```

#### 4. Requests set, no limits ✅ (recommended for CPU)
Each Pod is guaranteed its requested amount. When spare capacity exists, any Pod can use it freely. The scheduler still respects requests for placement decisions.

```
requests = 1 vCPU, no limit
Pod A can burst beyond 1 vCPU when Pod B is idle.
Pod B is still guaranteed its 1 vCPU when it needs it.
```

> **Memory is different** — unlike CPU, memory cannot be throttled. If a Pod over-consumes memory beyond its request with no limit set, the only recovery is to kill it. For memory, setting a limit is generally safer.

---

### Scenario Summary Table

| Scenario | CPU Guaranteed | CPU Burst | Memory Risk |
|---|---|---|---|
| No requests, no limits | ❌ | Unlimited | ⚠️ Any Pod can starve others |
| No requests, limits set | ✅ (= limit) | ❌ None | ✅ Capped |
| Requests + limits | ✅ | ✅ Up to limit | ✅ Capped |
| Requests only, no limits | ✅ | ✅ Unlimited | ⚠️ OOMKill risk if unchecked |

---

### LimitRange — Namespace-Level Defaults

A `LimitRange` object sets default requests/limits for containers that do not specify their own. It is scoped to a **namespace** and only applies to **newly created Pods** — existing Pods are unaffected.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-memory-defaults
  namespace: dev
spec:
  limits:
    - type: Container
      default:              # applied as limit if not specified
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:       # applied as request if not specified
        cpu: "250m"
        memory: "256Mi"
      max:                  # no container in this namespace can exceed this
        cpu: "1"
        memory: "1Gi"
      min:                  # no container can request less than this
        cpu: "100m"
        memory: "128Mi"
```

---

### ResourceQuota — Namespace-Level Total Cap

A `ResourceQuota` sets a **hard ceiling on total resource consumption** across all Pods in a namespace. It prevents any single team or application from monopolising cluster resources.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "4"          # total CPU requests across all Pods cannot exceed 4
    requests.memory: "4Gi"     # total memory requests cannot exceed 4Gi
    limits.cpu: "10"           # total CPU limits cannot exceed 10
    limits.memory: "10Gi"      # total memory limits cannot exceed 10Gi
```

---

### LimitRange vs ResourceQuota

| | LimitRange | ResourceQuota |
|---|---|---|
| **Scope** | Per container / Pod defaults | Total namespace consumption |
| **Purpose** | Fill in missing requests/limits on individual containers | Cap the total resources a namespace can consume |
| **Affects existing Pods?** | ❌ No | ✅ Yes — new Pods will be rejected if quota is exhausted |
| **Object kind** | `LimitRange` | `ResourceQuota` |

---

### Practical Behaviour — LimitRange

When you create or update a `LimitRange`, Kubernetes only enforces it at **Pod admission time** — the moment a Pod is submitted to the API server. It is not a continuous enforcement mechanism. Already-running Pods are never touched.

```
10:00 AM → Pod A created (no requests/limits set) → admitted as-is, no LimitRange exists
10:05 AM → LimitRange created (default cpu request: 250m)
10:10 AM → Pod B created (no requests/limits set) → LimitRange injects cpu request: 250m automatically

Pod A → still running with zero requests. LimitRange never touched it.
Pod B → running with cpu request: 250m applied by LimitRange.
```

> **Exam implication:** Creating a `LimitRange` to fix missing resource definitions does not protect already-running Pods. You must **delete and recreate** them to get the defaults applied.

---

### Practical Behaviour — ResourceQuota

When a `ResourceQuota` is created, it immediately starts **tracking total consumption** of all existing Pods in the namespace — but it does not evict or modify them. The hard limits only **block new Pod admissions** once the namespace total is exhausted.

```
10:00 AM → Pod A created (requests: cpu=3)  → admitted, no quota exists
10:02 AM → Pod B created (requests: cpu=3)  → admitted, no quota exists
           Total CPU requests in namespace  = 6

10:05 AM → ResourceQuota created (requests.cpu: 4)
           Quota immediately sees 6 consumed > 4 limit
           BUT Pod A and Pod B keep running — they are not evicted

10:10 AM → Pod C created (requests: cpu=1)  → REJECTED ❌
           Reason: quota already exceeded by existing Pods

10:15 AM → Pod A deleted
           Total CPU requests in namespace  = 3

10:20 AM → Pod C created (requests: cpu=1)  → admitted ✅
           Total CPU requests in namespace  = 4 (at quota limit)

10:25 AM → Pod D created (requests: cpu=1)  → REJECTED ❌
           Reason: would push total to 5, exceeds quota of 4
```

> **Exam implication:** A `ResourceQuota` applied to a namespace with heavy existing consumption will **immediately block new Pods** even though it never touched the existing ones. If Pods are stuck in `Pending` after a quota is applied, check:
> ```bash
> kubectl describe resourcequota -n <namespace>
> # Shows: current usage vs hard limits side by side
> ```

---

### When to Use Which

#### Use `resources.requests` and `resources.limits` when:
- You know the CPU and memory profile of your application
- You want to **guarantee** a minimum amount of resources for a specific container
- You want to **cap** a container from consuming runaway resources (e.g. a memory leak)
- You are running in a shared cluster where noisy neighbours are a concern

#### Use `LimitRange` when:
- You want to **enforce a baseline** across all Pods in a namespace without modifying every manifest individually
- Teams are deploying Pods without any `resources` block and you want safe defaults applied automatically
- You want to **prevent extreme values** — e.g. no single container should request more than 2 CPUs or less than 50m

> **Real-world example:** A platform team managing a shared dev namespace sets a `LimitRange` so that developers who forget to set resource requests don't accidentally starve the node. Every Pod gets sensible defaults without the platform team reviewing every manifest.

#### Use `ResourceQuota` when:
- You are running a **multi-team cluster** and need to give each team a fair share of cluster resources
- You want to **prevent a single namespace from monopolising** CPU or memory across the entire cluster
- You need to enforce **budget-like constraints** — e.g. the `dev` namespace cannot consume more than 8 CPUs total regardless of how many Pods are running

> **Real-world example:** A platform team gives each product team their own namespace with a `ResourceQuota` of `requests.cpu: 8` and `requests.memory: 16Gi`. This ensures no team can scale out of control and impact other teams running in the same cluster.

#### Use both together when:
- You want **individual container defaults** (LimitRange) AND **namespace-level total caps** (ResourceQuota) enforced simultaneously — this is the standard production pattern in shared clusters

```
LimitRange  →  ensures every container has a request set (so ResourceQuota can track it)
ResourceQuota →  ensures the namespace total never exceeds the allocated share
```

> **Why both?** `ResourceQuota` only tracks Pods that have requests set. If a Pod has no requests and no `LimitRange` exists to inject defaults, `ResourceQuota` cannot account for it — that Pod flies under the quota radar. Using both together closes this gap.

#### Decision Guide

```
Do you need to control a single container's resource usage?
  └── YES → resources.requests / resources.limits in the Pod spec

Do you need to auto-apply defaults across a namespace?
  └── YES → LimitRange

Do you need to cap total resource consumption across all Pods in a namespace?
  └── YES → ResourceQuota

Are you running a shared multi-team cluster?
  └── YES → LimitRange + ResourceQuota together (standard production pattern)
```

---

### Exam Gotchas

- **Pod in `Pending` with no node match** — check `kubectl describe pod` for `Insufficient CPU` or `Insufficient memory` events. The scheduler cannot find a node with enough available resources.
- **`OOMKilled` = Exit Code 137** — memory limit exceeded. The container is killed and restarted by the kubelet according to the Pod's `restartPolicy`.
- **CPU is throttled, memory is killed** — this distinction is directly exam-testable. CPU over-limit = slow. Memory over-limit = dead.
- **No requests set + no limits set** — this is the cluster default. Any Pod can starve others silently. Always set at least requests.
- **`LimitRange` only affects new Pods** — changing or creating a `LimitRange` has zero effect on already-running Pods.
- **Requests = scheduling currency** — the scheduler uses requests, not limits, to decide node placement. A Pod with no requests set appears to need zero resources and can be placed anywhere — including on an already-saturated node.

---

### In Managed Clusters (AKS / EKS / GKE)

- AKS enforces resource quotas at the node pool level through VM SKU sizing — a node pool on `Standard_D2s_v3` (2 vCPU, 8 GB RAM) physically caps what any single node can provide.
- AKS Vertical Pod Autoscaler (VPA) can automatically tune requests based on observed usage — directly relevant if you are running workloads without well-defined resource profiles.
- Azure Container Insights surfaces OOMKill events and CPU throttling metrics — use these to tune requests and limits in production rather than guessing upfront.
- In AKS, `LimitRange` objects in the `kube-system` namespace are managed by the platform — do not modify them.

---

