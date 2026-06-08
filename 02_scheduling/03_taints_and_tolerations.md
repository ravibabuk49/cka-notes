## 03 — Taints and Tolerations

### What it is
- **Taint** — a property applied to a **Node** that repels Pods from being scheduled on it unless the Pod explicitly tolerates that taint.
- **Toleration** — a property applied to a **Pod** that declares it can be scheduled onto a node carrying a matching taint.

Together they form a **node-restriction mechanism** — they control which Pods are allowed onto which Nodes. They do not attract Pods to specific Nodes; that is the job of Node Affinity.

> **Analogy:** Think of a Kubernetes node like an Azure Subnet with a **Network Security Group (NSG)**. By default, the subnet accepts all traffic (no taint). Once you attach an NSG rule that denies inbound traffic (taint), only packets that match an explicit allow rule (toleration) get through. The NSG does not route traffic to a specific subnet — it only filters what is allowed in. Taints work the same way — they filter what Pods are allowed onto a node, but do not direct a Pod to land on a specific node.

---

### Taint Effects

The taint effect defines what happens to a Pod that does **not** tolerate the taint:

| Effect | Behaviour |
|---|---|
| `NoSchedule` | Pod is never scheduled on the node. Existing Pods on the node are unaffected. |
| `PreferNoSchedule` | Scheduler avoids placing the Pod on the node but will do so if no other node is available. Soft constraint. |
| `NoExecute` | New Pods are not scheduled **and** existing Pods on the node that do not tolerate the taint are **evicted immediately**. |

---

### Applying a Taint to a Node

```bash
# Syntax
kubectl taint nodes <node-name> <key>=<value>:<effect>

# Example — dedicate node01 to the payments application
kubectl taint nodes node01 app=payments:NoSchedule

# Remove a taint (append -)
kubectl taint nodes node01 app=payments:NoSchedule-
```

---

### Adding a Toleration to a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: payments-api
spec:
  tolerations:
    - key: "app"
      operator: "Equal"
      value: "payments"
      effect: "NoSchedule"   # must match the taint effect exactly
  containers:
    - name: app
      image: payments-api:v1
```

> All toleration values must be **strings in double quotes**. Omitting quotes causes a validation error.

#### Toleration Operators

| Operator | Behaviour |
|---|---|
| `Equal` | Key, value, and effect must all match the taint exactly |
| `Exists` | Matches any taint with the specified key, regardless of value |

---

### How Scheduling Works with Taints

Given: `node01` tainted `app=payments:NoSchedule`, only Pod D has a matching toleration.

```
Pods: A, B, C (no toleration) | D (tolerates app=payments:NoSchedule)
Nodes: node01 (tainted), node02, node03 (untainted)

Scheduler result:
  Pod A → node02 or node03   (rejected by node01's taint)
  Pod B → node02 or node03   (rejected by node01's taint)
  Pod C → node02 or node03   (rejected by node01's taint)
  Pod D → node01, node02, OR node03  ← important — see gotcha below
```

---

### NoExecute — Eviction Behaviour

`NoExecute` is the only effect that acts on **already-running Pods**:

```bash
# Before taint: Pod C is running on node01
kubectl taint nodes node01 app=payments:NoExecute

# After taint:
# Pod C (no toleration) → evicted and rescheduled elsewhere
# Pod D (has toleration) → continues running on node01
```

You can also set a **tolerationSeconds** on a `NoExecute` toleration to allow a Pod to stay on the node for a grace period before eviction:

```yaml
tolerations:
  - key: "app"
    operator: "Equal"
    value: "payments"
    effect: "NoExecute"
    tolerationSeconds: 60    # Pod stays on node for 60s after taint is applied, then evicted
```

---

### Master Node Taint

When a cluster is bootstrapped (via kubeadm), the control plane node is automatically tainted to prevent workload Pods from being scheduled on it:

```bash
kubectl describe node <control-plane-node> | grep -i taint
# Taints: node-role.kubernetes.io/control-plane:NoSchedule
```

This taint can be removed, but doing so is not recommended for production clusters.

---

### Real-World Usage

| Scenario | Taint | Toleration on |
|---|---|---|
| **Dedicated GPU nodes** — reserve nodes with GPU hardware exclusively for ML workloads | `gpu=true:NoSchedule` | ML training Pods only |
| **Spot / Preemptible nodes** — prevent regular workloads landing on spot nodes that can be evicted anytime | `spot=true:NoExecute` | Only fault-tolerant, stateless Pods |
| **Node maintenance / drain** — cordon + taint a node before maintenance so new Pods are not scheduled and running Pods are gracefully evicted | `maintenance=true:NoExecute` | None — intentional full eviction |
| **Dedicated system node pool (AKS)** — AKS system node pools carry `CriticalAddonsOnly=true:NoSchedule` to reserve them for kube-system components | `CriticalAddonsOnly=true:NoSchedule` | Only kube-system Pods (CoreDNS, metrics-server etc.) |
| **Tainted ingress nodes** — isolate ingress controllers on edge nodes away from application workloads | `role=ingress:NoSchedule` | Ingress controller Pods only |

---

### Exam Gotchas

- **Taints restrict, they don't attract.** A Pod with a toleration for `node01` can still land on `node02` or `node03`. To force a Pod onto a specific node, combine taints with **Node Affinity**.
- **`NoExecute` evicts running Pods** — `NoSchedule` does not. This distinction is a frequent exam question.
- **Toleration values must be quoted strings** — missing quotes on `value`, `key`, or `effect` will cause a validation error.
- **`operator: Exists`** — do not specify `value` when using `Exists`; the key alone is matched.
- **Removing a taint** — append `-` to the taint string: `kubectl taint nodes node01 app=payments:NoSchedule-`. Forgetting the `-` adds a duplicate taint instead.
- **Master node taint** — in kubeadm clusters the control plane carries `node-role.kubernetes.io/control-plane:NoSchedule` by default. Expect a question where Pods are pending because the only available node is the control plane.

---

### In Managed Clusters (AKS / EKS / GKE)

- AKS **system node pools** are automatically tainted with `CriticalAddonsOnly=true:NoSchedule` — only kube-system Pods with a matching toleration are scheduled there.
- AKS **spot node pools** carry `kubernetes.azure.com/scalesetpriority=spot:NoSchedule` — workload Pods must explicitly tolerate this to land on spot nodes.
- In AKS you cannot taint system node pool nodes manually via `kubectl taint` — node pool taints are managed at the AKS API level (ARM / Terraform `node_taints` property).

---

