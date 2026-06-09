## 10 — Static Pods

### What it is
Static Pods are Pods created and managed **directly by the kubelet** on a node — without any involvement from the kube-apiserver, kube-scheduler, or any other control plane component. The kubelet watches a designated directory on the host filesystem for Pod definition files and acts on them autonomously.

> **Key distinction:** Every other Pod in Kubernetes is created by the control plane and assigned to a node by the scheduler. A Static Pod is created by the node itself, from a file on disk.

---

### How It Works

```
Host filesystem (manifest directory)
        │
        │  kubelet polls this directory periodically
        ▼
   kubelet reads .yaml files
        │
        ├── Pod does not exist → creates it
        ├── File changed       → deletes and recreates the Pod
        └── File removed       → deletes the Pod
```

- The kubelet **continuously reconciles** the running Pods against the files in the manifest directory.
- If the Pod crashes, the kubelet **restarts it automatically** — same as any other Pod.
- You can only create **Pods** this way — not ReplicaSets, Deployments, Services, or any other resource. Those require control plane components the kubelet does not have.

---

### Configuring the Manifest Directory

There are two ways the manifest directory path is configured on a node:

#### Method 1 — Direct flag in kubelet service

```bash
# /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
ExecStart=/usr/bin/kubelet \
  --pod-manifest-path=/etc/kubernetes/manifests \
  ...
```

#### Method 2 — Config file reference (kubeadm default)

```bash
# kubelet service references a config file
ExecStart=/usr/bin/kubelet \
  --config=/var/lib/kubelet/config.yaml \
  ...

# Inside config.yaml
staticPodPath: /etc/kubernetes/manifests
```

#### How to find the manifest directory on any cluster

```bash
# Step 1 — check for direct flag
ps aux | grep kubelet | grep pod-manifest-path

# Step 2 — if not found, find the config file path
ps aux | grep kubelet | grep config

# Step 3 — read the config file for staticPodPath
cat /var/lib/kubelet/config.yaml | grep staticPodPath
```

---

### Static Pods in a Cluster — Mirror Pods

When a node running static Pods is part of a cluster, the kubelet automatically creates a **mirror object** in the kube-apiserver for each static Pod. This allows you to see static Pods via `kubectl` — but with important constraints:

```bash
# Static Pods ARE visible via kubectl
kubectl get pods -n kube-system

# Static Pod names are automatically suffixed with the node name
# e.g. kube-apiserver-controlplane, etcd-controlplane
```

| Operation | Via kubectl | Via manifest file |
|---|---|---|
| View | ✅ | ✅ |
| Edit | ❌ Read-only mirror | ✅ Edit the file on disk |
| Delete | ❌ Rejected | ✅ Remove the file |

> **Mirror Pods are read-only.** Any attempt to `kubectl delete` or `kubectl edit` a static Pod is either rejected or immediately undone — the kubelet recreates it from the manifest file.

---

### Primary Use Case — Bootstrapping the Control Plane

This is the most important real-world use of static Pods. Since static Pods are self-sufficient and don't depend on a running control plane, **kubeadm uses them to deploy the control plane components themselves**:

```
/etc/kubernetes/manifests/
├── etcd.yaml                  ← etcd static Pod
├── kube-apiserver.yaml        ← kube-apiserver static Pod
├── kube-controller-manager.yaml
└── kube-scheduler.yaml
```

The kubelet on the control plane node reads these files and keeps the control plane running as Pods. If any component crashes, the kubelet restarts it automatically — no systemd service management needed.

```bash
# Verify on a kubeadm cluster
kubectl get pods -n kube-system
# You will see: etcd-<node>, kube-apiserver-<node>, kube-controller-manager-<node>, kube-scheduler-<node>
```

---

### Static Pods vs DaemonSets

Both result in Pods running on nodes outside the scheduler's control — but they are fundamentally different mechanisms:

| | Static Pod | DaemonSet |
|---|---|---|
| **Created by** | kubelet directly, from files on disk | DaemonSet controller via kube-apiserver |
| **Requires control plane?** | ❌ No | ✅ Yes |
| **Scope** | Single node (the node hosting the file) | All nodes in the cluster |
| **Managed via** | Files in manifest directory | `kubectl` / DaemonSet spec |
| **Visible via kubectl?** | ✅ As read-only mirror | ✅ Fully manageable |
| **kube-scheduler involvement** | ❌ Ignored | ❌ Ignored |
| **Primary use case** | Control plane bootstrapping | Node-level agents (logging, monitoring, networking) |

---

### Exam Gotchas

- **Finding the manifest directory is a core exam skill** — the path is not always `/etc/kubernetes/manifests`. Always follow the three-step lookup above to confirm it on the specific cluster.
- **Static Pod names always end with the node name** — `etcd-controlplane`, `kube-apiserver-controlplane`. This is how you identify them in `kubectl get pods -n kube-system` output.
- **Cannot delete a static Pod via `kubectl delete`** — it will reappear immediately. The only way to delete it is to remove or rename the manifest file from the directory on the node.
- **Cannot `kubectl edit` a static Pod** — the mirror object is read-only. SSH into the node and edit the file directly in the manifest directory.
- **Only Pods can be static** — if an exam question asks you to create a static Deployment or ReplicaSet, it is a trap. Only Pod definitions work in the manifest directory.
- **Both static Pods and DaemonSet Pods are ignored by kube-scheduler** — neither goes through the normal scheduling pipeline.
- **`docker ps` is dead — use `crictl ps`** — Docker was deprecated as a container runtime in Kubernetes v1.24. The CKA exam environment uses **containerd**. If no kube-apiserver is available and you need to inspect running containers on a node, use `crictl`:

```bash
# List running containers (equivalent to docker ps)
crictl ps

# List all containers including stopped
crictl ps -a

# Inspect a specific container
crictl inspect <container-id>

# View container logs
crictl logs <container-id>
```

---

### Creating a Static Pod in the Exam — Workflow

The exam commonly asks you to create a static Pod on a specific node. Follow this sequence:

```bash
# Step 1 — find the manifest directory on the target node
ssh <node-name>
ps aux | grep kubelet | grep -E 'pod-manifest-path|config'
# If --config flag found:
grep staticPodPath /var/lib/kubelet/config.yaml

# Step 2 — generate the Pod manifest using dry-run (fastest method)
kubectl run static-pod-name \
  --image=nginx \
  --dry-run=client -o yaml > /etc/kubernetes/manifests/static-pod-name.yaml

# Step 3 — verify the kubelet picked it up
crictl ps | grep static-pod-name

# Step 4 — if the cluster has an apiserver, verify the mirror Pod
kubectl get pods -A | grep static-pod-name
# Name will appear as: static-pod-name-<node-name>
```

> **SSH is required** to place a file in the manifest directory on a remote node. The exam environment always has SSH access between nodes.

---

### In Managed Clusters (AKS / EKS / GKE)

- In AKS, the control plane (etcd, kube-apiserver, kube-controller-manager, kube-scheduler) is fully managed by Microsoft and runs outside the customer's node pool — you cannot SSH into control plane nodes or access their manifest directories.
- Static Pods on AKS **worker nodes** are still functional — you can SSH into a worker node and place a manifest in the `staticPodPath` directory, but this is rarely needed or recommended in managed clusters.
- AKS node bootstrapping uses a managed kubelet config — the `staticPodPath` is set internally and not directly exposed to the operator.

---

