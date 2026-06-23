## Section 1: Cluster Architecture

### What it is

A Kubernetes cluster is a distributed system composed of two distinct layers: a **Control Plane** and a set of **Worker Nodes**.

The **Control Plane** is the cluster's management layer. It maintains the *desired state* of the cluster — what applications should run, with how many replicas, on which nodes, and with what configuration — by continuously running a set of control loops that detect and reconcile any drift between desired and actual state. It exposes a single API surface (`kube-apiserver`) through which all cluster operations are performed, and persists the entire cluster state in `etcd`, a distributed key-value store.

The **Worker Nodes** are the compute layer. Each node runs a local agent (`kubelet`) that receives pod specifications from the control plane and instructs the container runtime (`containerd`) to pull images and start or stop containers accordingly. Nodes do not make scheduling decisions — they only execute what the control plane assigns to them.

Together, they implement the **declarative model**: you describe the desired end state (e.g., *"run 3 replicas of this container"*), submit it to the API, and the control plane continuously drives the cluster toward that state — self-healing, rescheduling failed pods, and scaling as needed — without manual intervention.

---

### Purpose of Kubernetes

To host applications in the form of containers in an automated fashion — making it easy to deploy as many instances of an application as required, and to enable communication between different services within the application.

---

### Mental Model — Azure Resource Manager (ARM) Analogy

You already know this architecture from daily Terraform and AKS work — Kubernetes control plane and ARM solve the exact same class of problem: providing a single, mediated, auditable control surface over a fleet of resources.

| Azure concept | Kubernetes equivalent | Why the mapping holds |
|---|---|---|
| Azure Resource Manager (ARM) | `kube-apiserver` | Every operation — Portal, CLI, Terraform, SDK — goes through ARM first. Nothing modifies a resource directly. Same as `kube-apiserver` being the only entry point for kubectl, controllers, and kubelets |
| ARM resource state (subscription's resource state) | `etcd` | The authoritative record of every resource's current configuration and status — exactly what a `terraform plan` compares against |
| Azure Fabric Controller (host/placement engine) | `kube-scheduler` | Decides which physical host a VM lands on based on capacity and constraints. It decides placement — it does not provision the VM itself |
| Azure Policy / Azure Automation (compliance engine) | `kube-controller-manager` | Continuously evaluates actual state against desired/policy state and remediates drift — same pattern as a ReplicaSet controller noticing a missing pod and recreating it |
| Azure VM Guest Agent (`waagent`) | `kubelet` | The agent running inside each compute instance that receives instructions from the control plane, applies them, and reports status back — exactly what `kubelet` does on each node |
| Azure Load Balancer / NSG rules | `kube-proxy` | Maintains the routing and firewall rules that let independently placed compute instances reach each other at a stable address |
| Virtual Machine | Pod | The smallest unit of compute the control plane schedules, tracks, and reports status for |
| Processes inside a VM | Containers inside a pod | Multiple processes share the VM's network stack — similar to multiple containers sharing a pod's network namespace and reachable via `localhost` |

> **Why this matters technically:** When you run `terraform apply`, Terraform talks only to ARM — never directly to the underlying compute fabric. ARM validates, persists state, and orchestrates the rest. `kube-apiserver` plays the identical role: every component — scheduler, controllers, kubelets — only ever talks to the apiserver, never to etcd or to each other directly. This is the same architectural reason Azure (and every major cloud) centralises control through one API: mediated access to shared state is what makes the system consistent, auditable, and safe to automate against.

---

### Control Plane Components

| Component | What it does |
|---|---|
| `kube-apiserver` | Primary management component of the cluster. Responsible for orchestrating all operations within the cluster. Exposes the Kubernetes API used by external users, internal controllers, and worker nodes. The only component that reads/writes to etcd. |
| `etcd` | Distributed key-value store. Holds the complete cluster state — nodes, pods, configs, and metadata. Source of truth — if it's gone, the cluster has no memory. |
| `kube-scheduler` | Identifies the right node to place a container on, based on the container's resource requirements, the worker node's capacity, and any constraints such as taints/tolerations or node affinity rules. Does not start pods. |
| `kube-controller-manager` | Runs all built-in control loops. The **Node controller** handles node onboarding and unavailability. The **Replication controller** ensures the desired number of containers are running at all times. Each controller watches state via the API and reconciles actual → desired. |
| `cloud-controller-manager` | Integrates with the underlying cloud provider (AKS, EKS, GKE). Manages cloud-specific resources: load balancers, node lifecycle, and routes. Absent in bare-metal clusters. |

---

### Worker Node Components

| Component | What it does |
|---|---|
| `kubelet` | Primary node agent. Registers the node, watches for pod specs assigned to it, instructs the container runtime to start/stop containers, and reports node and pod status back to the apiserver. |
| `kube-proxy` | Maintains `iptables`/`IPVS` rules on every node so that Service IPs route correctly to the backing pod IPs across the cluster. |
| Container runtime | Pulls images and manages container lifecycle. Must be installed on **all nodes**, including master nodes (if hosting control plane components as containers). Kubernetes supports **Docker**, **containerd**, and **Rocket (rkt)**. `containerd` is the default from K8s 1.24+. |

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
- In a high-availability setup, there are **multiple master nodes** — etcd runs as a distributed cluster across them.

---

### In Managed Clusters (AKS / EKS / GKE)

> **The entire control plane is provisioned, managed, and operated by the cloud provider — you interact with it only through the Kubernetes API.**

| Control Plane Component | Self-managed cluster | AKS / EKS / GKE |
|---|---|---|
| `kube-apiserver` | You deploy and configure it | Managed by provider — exposed via a cloud-hosted endpoint |
| `etcd` | Your responsibility to run, back up, and recover | Fully managed — no operator access |
| `kube-scheduler` | You deploy and can customise | Managed by provider |
| `kube-controller-manager` | You deploy and configure | Managed by provider |
| `cloud-controller-manager` | Optional, self-configured | Pre-integrated — handles Azure/AWS/GCP resources automatically |
| Control plane nodes | You provision and maintain VMs | Provider-managed — not visible in `kubectl get nodes` |

**Practical implication for AKS:**
Running `kubectl get nodes` on an AKS cluster returns only worker nodes. Control plane nodes are managed infrastructure — they are not visible or accessible to the operator. The `kube-apiserver` endpoint is a provider-managed URL (e.g., `https://<cluster>.hcp.<region>.azmk8s.io`).

**What you are still responsible for on AKS:**
- Worker node sizing, scaling, and node pool configuration
- `kubelet` and `kube-proxy` behaviour on worker nodes
- Workload deployment, networking policies, RBAC, and storage configuration
- Cluster upgrades — initiated by the operator via `az aks upgrade`, executed by the provider

---


