## 13 — Scheduler Profiles

### What it is
Scheduler Profiles allow you to run **multiple scheduling behaviours within a single scheduler binary** by defining multiple named profiles in one `KubeSchedulerConfiguration` file. Each profile acts as an independent scheduler with its own plugin configuration. Introduced in Kubernetes **v1.18** as a cleaner alternative to running separate scheduler processes.

---

### How the Scheduler Works — The Four Phases

Before understanding profiles, it helps to understand the four phases every Pod goes through during scheduling:

> **Visual reference:** A complete flow diagram showing all four phases, extension points, and plugins at each stage was generated in the chat as part of this section. Refer to it alongside the table below.

| Phase | What happens |
|---|---|
| **Scheduling Queue** | Pod enters the queue and is sorted by priority — higher priority = scheduled first |
| **Filter** | Nodes that cannot run the Pod are eliminated (insufficient resources, taints, affinity mismatches) |
| **Score** | Remaining nodes are ranked — the node with the highest score wins |
| **Bind** | Pod is bound to the winning node via a Binding object — `spec.nodeName` is set |

---

### Extension Points

Each phase has one or more **extension points** — hooks where plugins can be plugged in:

> **Extension Point vs Plugin — the distinction:**
> - **Extension Point** — the specific *slot* in the scheduling pipeline where logic is allowed to run. Think of it as an electrical socket on the wall.
> - **Plugin** — the actual *logic* that runs at that slot. Think of it as the appliance you plug into the socket.
>
> One plugin can plug into **multiple extension points** — `NodeResourcesFit` runs at both `filter` (eliminates nodes without enough resources) and `score` (ranks remaining nodes by free space after allocation).
>
> One extension point can have **multiple plugins** plugged in simultaneously — the `filter` extension point runs `NodeResourcesFit`, `NodeName`, `NodeUnschedulable`, `TaintToleration`, and `NodeAffinity` all at once.

| Phase | Extension Points (in order) | Purpose |
|---|---|---|
| **Scheduling Queue** | `queueSort` | Sort Pods by priority before scheduling begins |
| **Filter** | `preFilter` → `filter` → `postFilter` | Eliminate nodes that cannot run the Pod |
| **Score** | `preScore` → `score` → `reserve` → `permit` | Rank remaining nodes; reserve resources; gate final placement |
| **Bind** | `preBind` → `bind` → `postBind` | Assign Pod to the winning node; run post-bind cleanup |

> The four phases are fixed. The extension points are the fine-grained hooks **within** each phase where plugins can inject logic — before, during, or after the core action of that phase.

---

### Default Plugins Per Phase

| Phase | Plugin | What it does |
|---|---|---|
| **Scheduling Queue** | `PrioritySort` | Sorts Pods by priority value — higher priority moves to the front |
| **Filter** | `NodeResourcesFit` | Eliminates nodes without sufficient CPU/memory for the Pod |
| | `NodeName` | Eliminates nodes that do not match `spec.nodeName` on the Pod |
| | `NodeUnschedulable` | Eliminates nodes with `spec.unschedulable: true` (cordoned nodes) |
| | `TaintToleration` | Eliminates nodes whose taints the Pod does not tolerate |
| | `NodeAffinity` | Eliminates nodes that do not satisfy the Pod's node affinity rules |
| **Score** | `NodeResourcesFit` | Scores nodes by free resources remaining after the Pod is allocated |
| | `ImageLocality` | Scores nodes higher if they already have the container image cached |
| | `TaintToleration` | Scores nodes with fewer matching taints higher |
| | `NodeAffinity` | Scores nodes that better match the Pod's preferred affinity rules higher |
| **Bind** | `DefaultBinder` | Creates the Binding object that sets `spec.nodeName` on the Pod |

> `NodeResourcesFit`, `TaintToleration`, and `NodeAffinity` each appear in **both Filter and Score** — they first eliminate invalid nodes, then rank the remaining ones. This is why a single plugin can plug into multiple extension points.

---

### Problem with Separate Scheduler Binaries

Running multiple schedulers as separate binaries (covered in `12_multiple_schedulers.md`) has two issues:

| Problem | Impact |
|---|---|
| **Operational overhead** | Each scheduler is a separate Deployment, ConfigMap, ServiceAccount, and RBAC config to maintain |
| **Race conditions** | Two separate processes can simultaneously decide to place Pods on the same node without awareness of each other, causing resource over-commitment |

Scheduler Profiles solve both problems — multiple behaviours, one binary, one process.

---

### Defining Multiple Profiles

Multiple profiles are defined under the `profiles` list in a single `KubeSchedulerConfiguration`. Each profile has a unique name and its own plugin configuration.

**Profile 1 — default scheduler, no changes:**
```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
```
This behaves identically to the standard kube-scheduler. No plugins are modified.

---

**Profile 2 — disable TaintToleration:**
Pods using this profile ignore taints entirely and can land on any node regardless of taints set on it.
```yaml
  - schedulerName: my-scheduler
    plugins:
      filter:
        disabled:
          - name: TaintToleration
      score:
        disabled:
          - name: TaintToleration
```

---

**Profile 3 — disable all scoring:**
Pods using this profile skip the scoring phase entirely — the first node that passes the filter is selected. Useful for workloads where placement speed matters more than optimal node selection.
```yaml
  - schedulerName: no-scoring-scheduler
    plugins:
      preScore:
        disabled:
          - name: "*"     # "*" disables ALL plugins at this extension point
      score:
        disabled:
          - name: "*"
```

All three profiles run inside a single scheduler binary — one Deployment, one config file.

---

### Assigning a Pod to a Profile

Identical to assigning a Pod to a custom scheduler binary — use `spec.schedulerName`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  schedulerName: my-scheduler     # matches the profile name exactly
  containers:
    - name: app
      image: nginx
```

---

### Multiple Schedulers vs Scheduler Profiles

| | Separate Scheduler Binaries | Scheduler Profiles |
|---|---|---|
| **Introduced** | Original Kubernetes | v1.18 |
| **Process count** | One process per scheduler | One process, multiple profiles |
| **Race condition risk** | ✅ Yes — separate processes | ❌ No — single process, shared state |
| **Operational overhead** | High — separate Deployments, RBAC | Low — single Deployment, one config |
| **Custom logic** | Requires separate binary build | Plugins enabled/disabled per profile |
| **When to use** | Completely different scheduler algorithm (e.g. Volcano) | Variations of default scheduler behaviour |

---

### Exam Gotchas

- **Profile name = scheduler name** — the `schedulerName` in each profile entry is what Pods reference in `spec.schedulerName`. They must match exactly.
- **`"*"` disables all plugins** in an extension point — useful when you want to strip out all scoring or filtering logic for a specific profile.
- **Disabling `TaintToleration` in a profile** means Pods using that profile completely ignore taints — they can land on any node regardless of taints. Dangerous if done accidentally.
- **Profiles share the same process** — they are not isolated. A bug in one profile's custom plugin can affect the entire scheduler process including the default profile.
- **Race conditions are eliminated within profiles** — but if you still run a completely separate scheduler binary alongside profiles (e.g. Volcano), race conditions between that binary and the profile-based scheduler can still occur.

---

### In Managed Clusters (AKS / EKS / GKE)

- In AKS, the default `kube-scheduler` is fully managed — you cannot modify its profiles or plugins directly.
- Custom scheduler profiles can be deployed on AKS as a separate Deployment in `kube-system` (same as a custom scheduler binary) — AKS does not prevent this.
- For production AKS workloads requiring custom scheduling behaviour, deploying a custom scheduler profile as a Deployment in `kube-system` is the supported approach — the same pattern covered in `12_multiple_schedulers.md`.

---

