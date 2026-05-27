## 10 — kube-proxy

### What it is

`kube-proxy` is a network proxy that runs on every node in the cluster. It is responsible for maintaining network rules that allow pods to communicate with Services. Every time a new Service is created, `kube-proxy` programs the appropriate forwarding rules on every node so that traffic destined for the Service IP is forwarded to the correct backend pod IP.

---

### The Problem kube-proxy Solves

Pods communicate with each other over an internal virtual network that spans all nodes — the **pod network**. Direct pod-to-pod communication using pod IPs works, but pod IPs are ephemeral — they change when a pod is rescheduled.

The solution is a **Service** — a stable virtual endpoint with a fixed ClusterIP and DNS name that fronts one or more pods.

```
Web pod → DB Service (ClusterIP: 10.96.0.12) → DB pod (PodIP: 10.32.0.15)
```

However, a Service is not a real network entity:
- It is not a container
- It has no network interface
- It has no actively listening process
- It exists only as an object in the Kubernetes API (etcd)

`kube-proxy` is what makes a Service actually reachable — it translates the virtual Service IP into real pod IP forwarding rules on every node.

---

### How kube-proxy Works

`kube-proxy` watches `kube-apiserver` for new Services and Endpoints. When a Service is created, it programs forwarding rules on every node so that traffic to the Service IP is redirected to the backing pod IP.

**Default mechanism — iptables:**

```
Packet destined for Service IP 10.96.0.12
         │
         ▼
   iptables rule on node (programmed by kube-proxy)
         │
         ▼
Redirected to Pod IP 10.32.0.15
```

`kube-proxy` supports three forwarding modes:

| Mode | Mechanism | Notes |
|---|---|---|
| `iptables` | Linux netfilter rules | Default mode. Efficient for most clusters. Rules evaluated per-packet. |
| `ipvs` | Linux IP Virtual Server | Better performance and scalability for large clusters with many Services. Uses hash tables — O(1) lookup vs O(n) for iptables. |
| `userspace` | Proxy in userspace | Legacy mode. Deprecated. Not used in modern clusters. |

---

### Installation

#### Method 1 — kubeadm

kubeadm deploys `kube-proxy` as a **DaemonSet** in the `kube-system` namespace — ensuring exactly one `kube-proxy` pod runs on every node in the cluster, including nodes added later.

```bash
# View kube-proxy DaemonSet
kubectl get daemonset -n kube-system kube-proxy

# View kube-proxy pods — one per node
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
```

```
NAME               READY   STATUS    NODE
kube-proxy-abc12   1/1     Running   node01
kube-proxy-xyz34   1/1     Running   node02
```

#### Method 2 — From Scratch (Manual)

Download the binary and run it as a `systemd` service on each node:

```bash
wget https://dl.k8s.io/v1.29.0/bin/linux/amd64/kube-proxy

kube-proxy \
  --config=/var/lib/kube-proxy/config.conf \
  --hostname-override=<node-name>
```

---

### Viewing kube-proxy Options on a Running Cluster

```bash
# View kube-proxy ConfigMap (kubeadm clusters)
kubectl get configmap kube-proxy -n kube-system -o yaml

# View running process on a node
ps -aux | grep kube-proxy
```

---

### Exam Gotchas

- A **Service is not a real network entity** — it has no interface or process. `kube-proxy` is what makes it reachable by programming iptables/IPVS rules on every node.
- `kube-proxy` is deployed as a **DaemonSet** in kubeadm clusters — one pod per node, always. This is different from other control plane components which run as static pods only on the master.
- If a Service is unreachable, check whether `kube-proxy` pods are running and healthy on the relevant nodes.
- `kube-proxy` operates at the **node level** — it programs rules on the node's kernel, not inside pods.
- Networking, Services, and `kube-proxy` internals are covered in full in the Networking section of this course.

---

### In Managed Clusters (AKS / EKS / GKE)

> `kube-proxy` runs on worker nodes in managed clusters. In AKS specifically, the default networking model affects whether `kube-proxy` is used at all.

| Aspect | Self-managed | AKS / EKS / GKE |
|---|---|---|
| Deployment | DaemonSet or systemd service | Provider-managed DaemonSet — pre-deployed on all nodes |
| Forwarding mode | Configurable (`iptables` / `ipvs`) | Provider-configured — typically `iptables` |
| Configuration | Full control via ConfigMap or flags | Managed internally by the provider |

**AKS-specific note:**
AKS supports two networking models — **kubenet** and **Azure CNI**. With Azure CNI, pod IPs are directly routable on the Azure VNet — reducing reliance on `kube-proxy` for some traffic paths. With **Azure CNI Overlay** or **Cilium** (available in AKS), eBPF-based networking can replace `kube-proxy` entirely for Service routing, offering better performance at scale. The underlying Service abstraction and `kube-proxy` behaviour remains consistent from a Kubernetes perspective regardless of the CNI in use.

---

