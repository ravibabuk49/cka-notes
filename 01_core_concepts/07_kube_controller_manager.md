## Section 7: kube-controller-manager

### What it is

`kube-controller-manager` is a single binary that packages and runs all built-in Kubernetes controllers. A **controller** is a control loop — a process that continuously monitors the state of specific cluster resources via `kube-apiserver` and takes corrective action to drive the actual state toward the desired state.

Every piece of intelligence built into Kubernetes constructs — Deployments, ReplicaSets, Services, Namespaces, PersistentVolumes — is implemented by a dedicated controller running inside `kube-controller-manager`.

---

### What a Controller Does

Every controller follows the same pattern:

```
1. Watch — continuously observes the state of specific resources via kube-apiserver
2. Compare — detects drift between actual state and desired state
3. Act — takes corrective action to reconcile the difference
```

---

### Built-in Controllers — Key Examples

#### Node Controller

Monitors the health of all nodes in the cluster via `kube-apiserver`.

| Event | Behaviour |
|---|---|
| Normal operation | Checks node status every **5 seconds** |
| Heartbeat stops | Waits **40 seconds** before marking the node as `Unreachable` |
| Node unreachable | Gives the node **5 minutes** to recover |
| Node does not recover | Evicts all pods from the node and reschedules them on healthy nodes |

> Pod eviction only applies if the pods are part of a ReplicaSet, Deployment, or similar managed workload. Standalone pods are not rescheduled.

#### Replication Controller

Monitors ReplicaSets and ensures the desired number of pods are running at all times within a replication group. If a pod dies, it creates a replacement immediately.

#### Other Controllers

Every Kubernetes construct has a corresponding controller:

| Construct | Controller |
|---|---|
| Deployments | Deployment controller |
| Services | Endpoints controller |
| Namespaces | Namespace controller |
| PersistentVolumes | PV / PVC controller |
| ServiceAccounts | ServiceAccount controller |
| Jobs | Job controller |
| CronJobs | CronJob controller |

> All of these are packaged into the single `kube-controller-manager` process. Installing it installs all controllers.

---

### Installation

#### Method 1 — kubeadm

kubeadm deploys `kube-controller-manager` as a static pod in the `kube-system` namespace:

```bash
kubectl get pods -n kube-system | grep kube-controller-manager
```

```
kube-controller-manager-controlplane   1/1   Running   0   20m
```

Static pod manifest location:

```bash
/etc/kubernetes/manifests/kube-controller-manager.yaml
```

#### Method 2 — From Scratch (Manual)

Download the binary from the Kubernetes release page and run it as a `systemd` service:

```bash
# Download
wget https://dl.k8s.io/v1.29.0/bin/linux/amd64/kube-controller-manager

# Run as a service — key flags
kube-controller-manager \
  --node-monitor-period=5s \
  --node-monitor-grace-period=40s \
  --pod-eviction-timeout=5m0s \
  --controllers=* \
  ...
```

---

### Key Configuration Options

| Flag | Default | Purpose |
|---|---|---|
| `--node-monitor-period` | `5s` | How frequently the node controller checks node status |
| `--node-monitor-grace-period` | `40s` | How long to wait before marking a node `Unreachable` after heartbeat loss |
| `--pod-eviction-timeout` | `5m0s` | How long to wait before evicting pods from an unreachable node |
| `--controllers` | `*` (all) | Comma-separated list of controllers to enable. Use this to enable a subset if needed. |

> If a controller does not appear to be functioning, `--controllers` is the first flag to check — the controller may have been explicitly excluded.

---

### Viewing kube-controller-manager Options on a Running Cluster

| Setup method | How to inspect |
|---|---|
| kubeadm | `cat /etc/kubernetes/manifests/kube-controller-manager.yaml` |
| Manual (systemd) | `cat /etc/systemd/system/kube-controller-manager.service` |
| Either (live process) | `ps -aux \| grep kube-controller-manager` |

---

### Exam Gotchas

- All controllers are a single process — `kube-controller-manager`. There is no separate binary per controller.
- Node controller timing values are frequently tested: **5s** check interval, **40s** grace period, **5m** eviction timeout — memorise all three.
- Standalone pods (not part of a ReplicaSet or Deployment) are **not rescheduled** after node eviction — they are simply lost.
- If a controller is misbehaving or missing, check the `--controllers` flag in the manifest — it may have been disabled.
- In a kubeadm cluster, changes to controller-manager configuration require editing `/etc/kubernetes/manifests/kube-controller-manager.yaml` — the static pod restarts automatically.

---

### In Managed Clusters (AKS / EKS / GKE)

> `kube-controller-manager` is fully managed by the cloud provider — you have no access to it or its configuration.

| Aspect | Self-managed | AKS / EKS / GKE |
|---|---|---|
| Deployment | Static pod or systemd service, configured by you | Provisioned and managed by the provider |
| Controller configuration | Full control via flags | Not accessible |
| Node eviction behaviour | Configurable via `--pod-eviction-timeout` | Provider-managed — default values applied |
| Custom controllers | Deployed separately alongside `kube-controller-manager` | Deployed as separate workloads in the cluster — unaffected by provider management |

**Practical implication for AKS:**
You cannot modify node controller timing or disable specific controllers on AKS. The provider runs `kube-controller-manager` with its own configuration. Custom controllers (e.g., operators built with controller-runtime) are deployed as regular pods in your cluster and are entirely under your control.

---

