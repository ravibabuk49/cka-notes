## OS Upgrades

### What it is

The process of safely removing a node from active scheduling before performing OS-level maintenance (patching, rebooting, kernel upgrades), ensuring workloads are not lost during the downtime window.

---

### Node Loss Behaviour

When a node goes offline, Kubernetes does not immediately reschedule its pods. The **Node Lifecycle Controller** waits for the node to recover before taking action.

| Scenario | Behaviour |
|---|---|
| Node recovers within eviction timeout | Pods restart on same node (if kubelet restarts them) |
| Node down beyond eviction timeout | Pods terminated; ReplicaSet pods recreated elsewhere; standalone pods are lost |

**Pod Eviction Timeout** — configured on `kube-controller-manager`:
```
--pod-eviction-timeout=5m0s   # default: 5 minutes
```

> This is a cluster-wide setting. In managed clusters (AKS/EKS/GKE), this is not user-configurable.

---

### Modern Eviction Mechanism — Taint-Based Eviction

Since Kubernetes 1.18+, taint-based eviction is the default and preferred mechanism, replacing the older `pod-eviction-timeout` approach.

When a node becomes `NotReady` or `Unreachable`, the Node Lifecycle Controller automatically applies **NoExecute** taints:

| Taint | Applied When |
|---|---|
| `node.kubernetes.io/not-ready:NoExecute` | Node condition `Ready=False` |
| `node.kubernetes.io/unreachable:NoExecute` | Node condition `Ready=Unknown` |

Pods without explicit tolerations for these taints are evicted after **`tolerationSeconds=300`** (5 minutes) — matching the legacy `pod-eviction-timeout` default.

Pods can extend their eviction grace period by declaring a toleration:
```yaml
tolerations:
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 600   # stay on node for 10 min before eviction
```

---

### The Unsafe Approach — Unplanned Node Downtime

If a node reboots unexpectedly:
- Pods with ReplicaSet/Deployment backing → recreated on other nodes after eviction timeout
- **Standalone pods (no controller)** → permanently lost
- Node returns blank — no pods are rescheduled back onto it automatically

**Risk:** You cannot guarantee a node returns within 5 minutes. Treat unplanned downtime as a data-loss scenario for standalone pods.

---

### The Safe Approach — Drain → Maintain → Uncordon

Three commands control node scheduling eligibility:

#### `kubectl drain`
Gracefully evicts all eligible pods from a node **and** cordons it in a single operation.

```bash
kubectl drain <node-name> \
  --ignore-daemonsets \       # required — DaemonSet pods cannot be evicted
  --delete-emptydir-data \    # required if any pod uses an emptyDir volume
  --grace-period=30 \         # seconds to wait per pod termination
  --timeout=120s              # total time before drain gives up
```

Drain will **block or fail** if:
- Pods not managed by a controller exist on the node — use `--force` to forcibly delete them (data loss risk)
- A PodDisruptionBudget prevents eviction — drain waits until satisfiable or times out

#### `kubectl cordon`
Marks a node `SchedulingDisabled` — **no new pods are scheduled**, existing pods continue running untouched.

```bash
kubectl cordon <node-name>
```

Use cordon (not drain) when you want to stop new scheduling without disturbing running workloads — e.g., before a long maintenance window where you want in-flight work to complete naturally.

#### `kubectl uncordon`
Removes the `SchedulingDisabled` state. Node becomes eligible for scheduling again.

```bash
kubectl uncordon <node-name>
```

> **Important:** Uncordoning does not reschedule existing pods back to the node. Pods placed on other nodes during maintenance stay there. Only future scheduling decisions include this node.

---

### Node State Transitions

```
Normal ──cordon──► SchedulingDisabled  (pods stay, no new pods)
Normal ──drain───► SchedulingDisabled  (pods evicted, no new pods)
SchedulingDisabled ──uncordon──► Normal (schedulable again)
```

Verify node status:
```bash
kubectl get nodes
# STATUS: Ready (normal) | Ready,SchedulingDisabled (cordoned/drained)

kubectl describe node <node-name>
# Taints: node.kubernetes.io/unschedulable:NoSchedule
```

---

### PodDisruptionBudgets (PDB) and Drain

A **PodDisruptionBudget** limits how many pods of a workload can be simultaneously unavailable during voluntary disruptions such as drain.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 1       # at least 1 pod must remain available at all times
  selector:
    matchLabels:
      app: myapp
```

If draining a node would violate a PDB, `kubectl drain` blocks until the budget is satisfiable or the `--timeout` expires. This is the correct production safeguard — do not blindly use `--force` to bypass PDBs in production.

---

### When to Use Which

| Scenario | Command |
|---|---|
| Planned OS reboot or patch — evacuate workloads first | `kubectl drain` |
| Stop new pods landing while allowing running pods to finish | `kubectl cordon` |
| Node isolated for extended maintenance, existing pods okay | `kubectl cordon` |
| Node back online after maintenance | `kubectl uncordon` |
| Node offline unexpectedly — controller-managed pods | No manual action — Node Lifecycle Controller handles recreation |

---

### Real-World Usage

**Rolling OS patch in production:**
```
1. kubectl drain node-3 --ignore-daemonsets --delete-emptydir-data
2. Apply OS patch, reboot node
3. kubectl get nodes   # confirm node rejoins as Ready
4. kubectl uncordon node-3
```

**AKS Node Pool Upgrades:** AKS automates the drain → upgrade → rejoin cycle per node during `az aks nodepool upgrade`. It respects PDBs. Stalled AKS upgrades are almost always a PDB blocking drain — check with `kubectl get pdb -A`.

**Cluster Autoscaler interaction:** The autoscaler will not scale in a cordoned node (already unschedulable), but will still remove it if the node is fully empty and meets scale-down criteria.

---

### Exam Gotchas

- `kubectl drain` **always** cordons the node implicitly — no need to run `kubectl cordon` first.
- Omitting `--ignore-daemonsets` causes drain to fail immediately — DaemonSet pods cannot be evicted by design.
- Omitting `--delete-emptydir-data` causes drain to fail if any pod uses an emptyDir volume.
- After `kubectl uncordon`, **pods do not migrate back** — only future pod scheduling considers the node.
- Standalone pods (no owning controller) are permanently lost after eviction timeout — and `kubectl drain` refuses to evict them without `--force`.
- `kubectl cordon` does NOT evict or move any pods — running pods stay exactly where they are.
- The `--pod-eviction-timeout` flag on `kube-controller-manager` is **deprecated** in favour of taint-based eviction with `tolerationSeconds`.

---

### In Managed Clusters (AKS / EKS / GKE)

- Pod eviction timeout is not user-configurable — controlled by the managed control plane.
- AKS automates drain → patch → rejoin per node during node pool upgrades; respects PDBs natively.
- AKS **max surge** setting controls how many extra nodes are provisioned during an upgrade to reduce drain wait time (`--max-surge` on `az aks nodepool update`).
- Manual node drain in AKS is only relevant when performing custom OS-level operations via SSH — standard upgrades should go through the AKS upgrade APIs.
- Taint-based eviction behaviour is identical to self-managed clusters.

