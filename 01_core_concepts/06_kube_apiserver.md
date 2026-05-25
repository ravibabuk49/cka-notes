## Section 6: kube-apiserver

### What it is

`kube-apiserver` is the **primary management component** of Kubernetes. It is the single entry point for all operations — internal and external. Every component in the cluster communicates exclusively through the apiserver; no component communicates directly with another.

---

### Responsibilities

| Responsibility | Description |
|---|---|
| **Authentication** | Verifies the identity of every incoming request |
| **Validation** | Ensures the request is well-formed and permitted |
| **Data retrieval and storage** | The only component that reads from and writes to etcd directly |
| **Coordination** | Relays information between scheduler, controller-manager, and kubelet |

---

### How a Request Flows — kubectl Example

When you run any `kubectl` command:

```
kubectl get pods
```

```
kubectl → kube-apiserver → authenticates → validates
                        → retrieves data from etcd
                        → responds to kubectl
```

You can also interact with the API directly without `kubectl` by sending HTTP requests:

```bash
curl -X GET https://<apiserver>:6443/api/v1/pods \
  --cacert ca.crt \
  --cert admin.crt \
  --key admin.key
```

---

### Pod Creation — End-to-End Request Flow

This is the most important flow to understand — it illustrates how all control plane components interact through the apiserver:

```
1. kubectl run nginx --image=nginx
         │
         ▼
2. kube-apiserver
   → Authenticates and validates the request
   → Creates a Pod object in etcd (no node assigned yet)
   → Responds to the user: "pod/nginx created"
         │
         ▼
3. kube-scheduler (watches apiserver continuously)
   → Detects a new pod with no node assigned
   → Identifies the most suitable node
   → Communicates the node assignment back to kube-apiserver
         │
         ▼
4. kube-apiserver
   → Updates the pod's node assignment in etcd
   → Forwards the pod spec to the kubelet on the assigned worker node
         │
         ▼
5. kubelet (on the worker node)
   → Receives the pod spec from kube-apiserver
   → Instructs the container runtime (containerd) to pull the image and start the container
   → Reports pod status back to kube-apiserver
         │
         ▼
6. kube-apiserver
   → Updates the pod status in etcd
```

> **Key principle:** `kube-apiserver` is at the centre of every operation. The scheduler, controller-manager, and kubelet never communicate with each other directly — all communication is mediated through the apiserver.

---

### Installation

#### Method 1 — kubeadm

kubeadm deploys `kube-apiserver` automatically as a static pod in the `kube-system` namespace:

```bash
kubectl get pods -n kube-system | grep kube-apiserver
```

```
kube-apiserver-controlplane   1/1   Running   0   15m
```

Static pod manifest location:

```bash
/etc/kubernetes/manifests/kube-apiserver.yaml
```

#### Method 2 — From Scratch (Manual)

Download the binary from the Kubernetes release page and configure it as a `systemd` service on the master node:

```bash
# Download
wget https://dl.k8s.io/v1.29.0/bin/linux/amd64/kube-apiserver

# Run as a service — key flags
kube-apiserver \
  --etcd-servers=https://<etcd-ip>:2379 \
  --client-ca-file=/etc/kubernetes/pki/ca.crt \
  --tls-cert-file=/etc/kubernetes/pki/apiserver.crt \
  --tls-private-key-file=/etc/kubernetes/pki/apiserver.key \
  ...
```

---

### Key Configuration Options

The apiserver runs with many flags. Most relate to TLS certificates — covered in the Security section. The ones relevant now:

| Flag | Purpose |
|---|---|
| `--etcd-servers` | URL of the etcd cluster. This is how kube-apiserver connects to etcd. e.g. `https://<etcd-ip>:2379` |
| `--client-ca-file` | CA certificate used to authenticate clients (kubectl, kubelets, controllers) |
| `--tls-cert-file` | TLS certificate served by the apiserver for HTTPS |
| `--tls-private-key-file` | Private key for the apiserver's TLS certificate |
| `--authorization-mode` | Defines how requests are authorised — typically `Node,RBAC` |
| `--service-cluster-ip-range` | CIDR range for Service ClusterIPs |

> All other components — kubelet, scheduler, controller-manager — need certificates to authenticate with the apiserver. These are configured in the relevant sections of this course.

---

### Viewing kube-apiserver Options on a Running Cluster

| Setup method | How to inspect |
|---|---|
| kubeadm | `cat /etc/kubernetes/manifests/kube-apiserver.yaml` |
| Manual (systemd) | `cat /etc/systemd/system/kube-apiserver.service` |
| Either (live process) | `ps -aux \| grep kube-apiserver` |

---

### Exam Gotchas

- `kube-apiserver` is the **only** component that communicates directly with etcd. Every other component reaches etcd indirectly through the apiserver.
- If `kube-apiserver` is down, `kubectl` commands fail entirely — no reads or writes to the cluster are possible.
- In a kubeadm cluster, modifying apiserver configuration means editing `/etc/kubernetes/manifests/kube-apiserver.yaml` — the static pod restarts automatically on save.
- The `--etcd-servers` flag must point to the correct etcd endpoint with the correct port (`2379`) — a misconfigured value here breaks the entire cluster.
- In the CKA exam, when asked to inspect a component's configuration on a kubeadm cluster, always check the static pod manifest at `/etc/kubernetes/manifests/` first.

---

### In Managed Clusters (AKS / EKS / GKE)

> `kube-apiserver` is fully managed by the cloud provider — you never deploy, configure, or access it directly.

| Aspect | Self-managed | AKS / EKS / GKE |
|---|---|---|
| Deployment | Binary or static pod, configured manually | Provisioned and managed by the provider |
| Endpoint | Configured by you | Provider-issued URL (e.g., `https://<cluster>.hcp.<region>.azmk8s.io:443`) |
| TLS certificates | Generated and managed by you | Provider-managed and auto-rotated |
| Configuration flags | Full control via manifest or service file | Not accessible — managed internally |
| Scaling / HA | Configured manually | Provider handles automatically |

**Practical implication for AKS:**
Your `~/.kube/config` (or the output of `az aks get-credentials`) contains the provider-issued apiserver endpoint. Every `kubectl` command you run hits that endpoint — the underlying apiserver infrastructure is invisible to you. This is why `kubectl` works identically on AKS as it does on a self-managed cluster — the API contract is the same; only the infrastructure behind it differs.

---
