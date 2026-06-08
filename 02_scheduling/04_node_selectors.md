## 04 — Node Selectors

### What it is
`nodeSelector` is the simplest form of node-targeting in Kubernetes. It is a field in the Pod spec that specifies a set of key-value label pairs — the scheduler only places the Pod on nodes that carry **all** the specified labels. It is an exact-match, AND-only mechanism with no support for expressions, OR logic, or exclusions.

---

### How It Works

1. Label the target node with a key-value pair.
2. Add `nodeSelector` to the Pod spec referencing that same label.
3. The scheduler filters nodes by matching the Pod's `nodeSelector` against node labels — only matching nodes are considered.

---

### Step 1 — Label the Node

```bash
# Syntax
kubectl label nodes <node-name> <key>=<value>

# Example — mark node01 as a large node
kubectl label nodes node01 size=large

# Verify
kubectl get node node01 --show-labels
```

---

### Step 2 — Add nodeSelector to the Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-processor
spec:
  nodeSelector:
    size: large          # must match the label on the target node exactly
  containers:
    - name: processor
      image: data-processor:v1
```

When this Pod is created, the scheduler only considers nodes that carry the label `size=large`. All other nodes are filtered out.

---

### Limitations

`nodeSelector` only supports exact key-value matching. It cannot express:

| Requirement | Possible with nodeSelector? |
|---|---|
| Place on `large` node | ✅ |
| Place on `large` OR `medium` node | ❌ |
| Place on any node that is NOT `small` | ❌ |
| Prefer a node but don't hard-require it | ❌ |

For these requirements, use **Node Affinity** (covered in the next section).

---

### Real-World Usage

`nodeSelector` is appropriate when the targeting requirement is simple and unlikely to change:

| Scenario | Label | nodeSelector on |
|---|---|---|
| **OS-specific workloads** — Windows containers must land on Windows nodes | `kubernetes.io/os=windows` | Windows workload Pods |
| **Architecture targeting** — ARM vs AMD64 node pools | `kubernetes.io/arch=arm64` | Pods built for ARM images |
| **Storage-attached nodes** — Pods that require local SSD must land on nodes with SSDs physically attached | `disk=ssd` | Database or cache Pods needing low-latency disk |
| **Licensed software nodes** — nodes with specific software licences installed | `license=matlab` | Simulation workload Pods |
| **AKS node pool targeting** — route workloads to a specific user node pool by its agentpool label | `agentpool=gpupool` | ML inference Pods |

> For anything beyond a single exact-match label — use **Node Affinity**. `nodeSelector` is intentionally kept simple; complexity belongs in affinity rules.

---

### Exam Gotchas

- **Label the node before creating the Pod** — if the node is not labelled when the Pod is scheduled, the Pod stays `Pending` indefinitely. Labelling the node afterwards will trigger the scheduler to place the pending Pod.
- **`nodeSelector` vs `nodeName`** — `nodeName` bypasses the scheduler entirely and pins to an exact node by name. `nodeSelector` works through the scheduler using labels and can match multiple nodes.
- **`nodeSelector` vs Node Affinity** — `nodeSelector` is always a hard requirement (equivalent to `requiredDuringSchedulingIgnoredDuringExecution`). There is no soft/preferred variant in `nodeSelector`.
- **All labels must match** — if you specify multiple key-value pairs under `nodeSelector`, the node must carry all of them (AND logic). There is no way to express OR within `nodeSelector`.
- **Built-in node labels** — Kubernetes automatically assigns well-known labels to every node (e.g. `kubernetes.io/os`, `kubernetes.io/arch`, `node.kubernetes.io/instance-type`). These are usable in `nodeSelector` without manual labelling.

---

### In Managed Clusters (AKS / EKS / GKE)

- AKS automatically labels nodes with `agentpool=<pool-name>` and `kubernetes.azure.com/agentpool=<pool-name>` — use either in `nodeSelector` to target a specific node pool without manual labelling.
- AKS also injects `kubernetes.io/os=linux` or `kubernetes.io/os=windows` automatically based on the node pool OS type — Windows container workloads rely on this label.
- In AKS, `nodeSelector` is the recommended approach for basic node pool targeting; Node Affinity is preferred when dealing with spot pools or multi-pool soft preferences.

---

