## Section 1: Cluster Architecture

### What it is

A Kubernetes cluster is a distributed system composed of two distinct layers: a **Control Plane** and a set of **Worker Nodes**.

The **Control Plane** is the cluster's management layer. It maintains the *desired state* of the cluster — what applications should run, with how many replicas, on which nodes, and with what configuration — by continuously running a set of control loops that detect and reconcile any drift between desired and actual state. It exposes a single API surface (`kube-apiserver`) through which all cluster operations are performed, and persists the entire cluster state in `etcd`, a distributed key-value store.

The **Worker Nodes** are the compute layer. Each node runs a local agent (`kubelet`) that receives pod specifications from the control plane and instructs the container runtime (`containerd`) to pull images and start or stop containers accordingly. Nodes do not make scheduling decisions — they only execute what the control plane assigns to them.

Together, they implement the **declarative model**: you describe the desired end state (e.g., *"run 3 replicas of this container"*), submit it to the API, and the control plane continuously drives the cluster toward that state — self-healing, rescheduling failed pods, and scaling as needed — without manual intervention.

---

### Control Plane Components

| Component | What it does |
|---|---|
| `kube-apiserver` | Single entry point for all REST operations. Authenticates, validates, and processes every request. The only component that reads/writes to etcd. |
| `etcd` | Distributed key-value store. Holds the complete cluster state. Source of truth — if it's gone, the cluster has no memory. |
| `kube-scheduler` | Watches for unscheduled pods and assigns them to the most suitable node based on resource availability, constraints, and policies. Does not start pods. |
| `kube-controller-manager` | Runs all built-in control loops (Node, Replication, Endpoints, ServiceAccount controllers). Each loop watches state via the API and reconciles actual → desired. |
| `cloud-controller-manager` | Integrates with the underlying cloud provider (AKS, EKS, GKE). Manages cloud-specific resources: load balancers, node lifecycle, and routes. Absent in bare-metal clusters. |

---

### Worker Node Components

| Component | What it does |
|---|---|
| `kubelet` | Primary node agent. Registers the node, watches for pod specs assigned to it, instructs the container runtime to start/stop containers, and reports node and pod status back to the apiserver. |
| `kube-proxy` | Maintains `iptables`/`IPVS` rules on every node so that Service IPs route correctly to the backing pod IPs across the cluster. |
| Container runtime | Pulls images and manages container lifecycle. `containerd` is the default from K8s 1.24+. Implements the **CRI** (Container Runtime Interface). |

---

### How Components Communicate

```
kubectl
  │
  ▼  (HTTPS / REST)
kube-apiserver ──────────► etcd              (read / write cluster state)
       │
       ├───────────────────► kube-scheduler   (watches for unbound pods)
       │
       ├───────────────────► kube-controller-manager (watches for state drift)
       │
       ▼  (kubelet watch / HTTPS)
    kubelet  (on each worker node)
       │
       ▼
  container runtime  ──────► pulls image, starts container
```

> **Golden rule:** Nothing talks to etcd directly except `kube-apiserver`. Every component goes through the API.

---

### Key File Paths (memorise these)

```bash
/etc/kubernetes/manifests/     # Static pod definitions for control plane components
/etc/kubernetes/admin.conf     # kubeconfig for cluster admin
/var/lib/etcd/                 # etcd data directory (critical for backup/restore)
```

---

### Exam Gotchas

- `kube-scheduler` only **decides** which node — it does NOT start the pod. `kubelet` starts it.
- `etcd` backup and restore is a **guaranteed task** in the CKA exam. Know the `etcdctl snapshot save/restore` commands cold.
- Docker was deprecated as a runtime in K8s 1.20 and **removed in 1.24**. The runtime is now `containerd`. Docker-built images still work fine — only the runtime shim changed.
- Control plane components often run as **static pods** (defined in `/etc/kubernetes/manifests/`). If a control plane component is broken, check that path first.
- `cloud-controller-manager` is only present in **managed/cloud clusters**. On a bare-metal self-hosted cluster, it doesn't exist.
- In a high-availability setup, there are **multiple master nodes** — etcd runs as a distributed cluster across them.

---
