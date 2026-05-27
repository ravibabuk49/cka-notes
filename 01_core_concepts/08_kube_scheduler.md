## 08 — kube-scheduler

### What it is

`kube-scheduler` is the control plane component responsible for assigning pods to nodes. It makes scheduling **decisions** only — it does not place pods on nodes. The actual pod creation on the node is performed by `kubelet` once the scheduler has made its assignment.

---

### Scheduling Process — Two Phases

For every unscheduled pod, the scheduler executes two sequential phases:

#### Phase 1 — Filter

Eliminates all nodes that do not satisfy the pod's hard requirements:

- Insufficient CPU or memory resources
- Node taints that the pod does not tolerate
- Node affinity/selector rules that the node does not satisfy
- Missing required volumes or topology constraints

The result is a set of **feasible nodes** — nodes the pod can physically run on.

#### Phase 2 — Score

Ranks all feasible nodes using a set of priority functions, each assigning a score on a scale of **0 to 10**. The node with the highest aggregate score wins.

Example priority function — **Least Requested Resources:**
After simulating pod placement, the node with the most free resources remaining scores higher. This spreads workloads evenly across the cluster rather than packing them onto a single node.

```
Node A — 2 CPUs free after placement → lower score
Node B — 6 CPUs free after placement → higher score → selected
```

The pod's `spec.nodeName` field is then updated with the selected node — this write goes to `kube-apiserver`, which updates etcd. `kubelet` on the target node picks up the assignment and creates the pod.

---

### What Influences Scheduling Decisions

| Factor | Description |
|---|---|
| Resource requests | CPU and memory requested by the pod — nodes without sufficient allocatable capacity are filtered out |
| Taints and Tolerations | Nodes can repel pods unless the pod explicitly tolerates the taint |
| Node Selectors | Pods can require specific node labels |
| Node Affinity rules | Fine-grained rules for attracting pods to specific nodes |
| Pod Affinity / Anti-affinity | Co-locate or separate pods relative to other pods |
| Topology constraints | Spread pods across zones, regions, or racks |

> Each of these is covered in full in the Scheduling section of this course.

---

### Installation

#### Method 1 — kubeadm

kubeadm deploys `kube-scheduler` as a static pod in the `kube-system` namespace:

```bash
kubectl get pods -n kube-system | grep kube-scheduler
```

```
kube-scheduler-controlplane   1/1   Running   0   25m
```

Static pod manifest location:

```bash
/etc/kubernetes/manifests/kube-scheduler.yaml
```

#### Method 2 — From Scratch (Manual)

Download the binary and run it as a `systemd` service with a configuration file:

```bash
# Download
wget https://dl.k8s.io/v1.29.0/bin/linux/amd64/kube-scheduler

# Run as a service — key flag
kube-scheduler \
  --config=/etc/kubernetes/scheduler-config.yaml
```

The scheduler configuration file defines the scheduling profile — which plugins run in the Filter and Score phases, custom weights, and multiple scheduler profiles.

---

### Viewing kube-scheduler Options on a Running Cluster

| Setup method | How to inspect |
|---|---|
| kubeadm | `cat /etc/kubernetes/manifests/kube-scheduler.yaml` |
| Either (live process) | `ps -aux \| grep kube-scheduler` |

---

### Exam Gotchas

- `kube-scheduler` decides which node — `kubelet` creates the pod. These are two distinct steps. Never conflate them.
- If a pod is stuck in `Pending` state, the scheduler could not find a feasible node — check resource availability, taints, and affinity rules.
- If no scheduler is running, pods remain in `Pending` indefinitely — they will not self-schedule.
- You can bypass the scheduler entirely by setting `spec.nodeName` directly in the pod manifest — the pod is assigned to that node without going through the scheduling process. This is called **manual scheduling**.
- Custom schedulers can be deployed alongside the default scheduler — pods select which scheduler to use via `spec.schedulerName`.

---

### In Managed Clusters (AKS / EKS / GKE)

> `kube-scheduler` is fully managed by the cloud provider — you cannot modify its configuration or replace it directly.

| Aspect | Self-managed | AKS / EKS / GKE |
|---|---|---|
| Deployment | Static pod or systemd service, configured by you | Provisioned and managed by the provider |
| Scheduler configuration | Full control via `--config` and scheduling profiles | Not accessible |
| Custom schedulers | Deploy alongside default scheduler as a pod | Fully supported — deploy as a regular workload |
| Node pools (AKS-specific) | Managed via taints, labels, and affinity rules | AKS node pools use taints and labels — standard scheduling rules apply |

**Practical implication for AKS:**
AKS node pools are surfaced to the scheduler as standard Kubernetes nodes with labels and taints. Scheduling to a specific node pool is done via `nodeSelector`, node affinity, or tolerations — the same mechanisms used in any Kubernetes cluster. The scheduler itself is managed; your scheduling rules are not.

---

