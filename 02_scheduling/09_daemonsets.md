## 09 — DaemonSets

### What it is
A DaemonSet ensures that **exactly one copy of a Pod runs on every node** in the cluster. When a node is added, the DaemonSet automatically schedules a Pod on it. When a node is removed, the Pod is garbage collected. Unlike a ReplicaSet which targets a desired replica count, a DaemonSet targets **every node** as its desired state.

---

### DaemonSet vs ReplicaSet

| | ReplicaSet | DaemonSet |
|---|---|---|
| **Desired state** | N replicas across any nodes | 1 Pod per node, on every node |
| **New node added** | No action — existing replica count unchanged | Pod automatically scheduled on new node |
| **Node removed** | Pod rescheduled elsewhere | Pod removed, no replacement needed |
| **Use case** | Stateless application replicas | Node-level agents and infrastructure components |

---

### How DaemonSet Scheduling Works

Prior to Kubernetes **v1.12**, DaemonSets scheduled Pods by directly setting `spec.nodeName` on each Pod — bypassing the scheduler entirely (see `01_manual_scheduling.md`).

From **v1.12 onwards**, DaemonSets use the **default scheduler** with **Node Affinity rules** to place Pods on nodes — the same mechanism covered in `05_node_affinity.md`.

---

### Real-World Use Cases

| Use Case | Example |
|---|---|
| **Log collection** | Fluentd, Filebeat — one agent per node to ship node and container logs |
| **Monitoring** | Prometheus Node Exporter, Datadog agent — collect node-level metrics |
| **Networking** | Calico, Flannel, Cilium — CNI agents must run on every node to manage Pod networking |
| **kube-proxy** | The built-in `kube-proxy` component is itself deployed as a DaemonSet in kubeadm clusters |
| **Storage** | Ceph, Longhorn storage agents — require a presence on every node |
| **Security** | Falco, Sysdig — kernel-level security monitoring needs a Pod on every node |

---

### DaemonSet Definition

Structure is almost identical to a ReplicaSet — the key difference is `kind: DaemonSet` and no `replicas` field (it is irrelevant — one per node is the implicit desired state).

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-agent
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: monitoring-agent       # must match template labels
  template:
    metadata:
      labels:
        app: monitoring-agent
    spec:
      containers:
        - name: monitoring-agent
          image: monitoring-agent:v1
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              memory: "256Mi"
```

> There is no `replicas` field in a DaemonSet — it is not needed. The controller manages one Pod per node automatically.

---

### Essential Commands

```bash
# Create
kubectl apply -f daemonset.yaml

# List all DaemonSets
kubectl get daemonsets -n kube-system

# Shorthand
kubectl get ds -n kube-system

# Inspect — shows desired, current, ready, and node counts
kubectl describe daemonset monitoring-agent -n kube-system

# Check which nodes have the DaemonSet Pod running
kubectl get pods -l app=monitoring-agent -o wide -n kube-system
```

---

### Exam Gotchas

- **No `replicas` field** — including `replicas` in a DaemonSet spec causes a validation error. The desired count is always "one per node".
- **`kubectl create` does not support DaemonSet** — unlike Deployments and ReplicaSets, there is no imperative `kubectl create daemonset` command. You must write the manifest. The fastest approach in the exam is to generate a Deployment manifest and change `kind: Deployment` to `kind: DaemonSet` and remove the `replicas` field:
```bash
kubectl create deployment monitoring-agent \
  --image=monitoring-agent:v1 \
  --dry-run=client -o yaml > ds.yaml
# Then edit: change kind to DaemonSet, remove replicas and strategy fields
kubectl apply -f ds.yaml
```
- **DaemonSet Pods on control plane nodes** — by default, DaemonSet Pods are not scheduled on control plane nodes because of the `node-role.kubernetes.io/control-plane:NoSchedule` taint. To run on control plane nodes, add a matching toleration to the DaemonSet Pod spec.
- **`kubectl get ds` not `kubectl get daemonset`** — both work, but `ds` is the shorthand you want under exam time pressure.
- **Namespace matters** — most system DaemonSets (kube-proxy, CNI agents) live in `kube-system`. Always specify `-n kube-system` or `-A` when looking for them.

---

### In Managed Clusters (AKS / EKS / GKE)

- In AKS, core DaemonSets (`kube-proxy`, `azure-ip-masq-agent`, `csi-azuredisk-node`, `csi-azurefile-node`) run in `kube-system` and are fully managed by the platform — do not modify or delete them.
- AKS node pools run as VMSS (Virtual Machine Scale Sets) — when the VMSS scales out, AKS bootstraps the new node and the DaemonSet controller automatically schedules the Pod on it without any manual intervention.
- Custom DaemonSets (e.g. Datadog, Falco) are fully supported on AKS user node pools. For system node pools, add a `CriticalAddonsOnly` toleration if the DaemonSet must run there.

---

