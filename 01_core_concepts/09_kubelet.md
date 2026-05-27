## 09 — kubelet

### What it is

`kubelet` is the primary node agent that runs on every worker node. It is the sole point of contact between the worker node and the control plane — registering the node with the cluster, executing pod assignments from `kube-scheduler` via `kube-apiserver`, and continuously reporting node and pod status back to the control plane.

---

### Responsibilities

| Responsibility | Description |
|---|---|
| Node registration | Registers the worker node with the Kubernetes cluster on startup |
| Pod execution | Receives pod specs from `kube-apiserver` and instructs the container runtime to pull images and start containers |
| Health monitoring | Continuously monitors the state of running pods and containers on the node |
| Status reporting | Reports node and pod status back to `kube-apiserver` at regular intervals |

---

### How kubelet Works

```
kube-apiserver
      │
      │  pod spec assigned to this node (by scheduler)
      ▼
   kubelet
      │
      ├──► instructs container runtime (containerd) to pull image
      ├──► instructs container runtime to start container
      ├──► monitors pod and container state continuously
      └──► reports status back to kube-apiserver at regular intervals
```

---

### Critical Installation Difference — kubeadm

> **kubeadm does NOT automatically deploy `kubelet` on worker nodes.**

This is the key difference from every other control plane component. `kube-apiserver`, `etcd`, `kube-scheduler`, and `kube-controller-manager` are all deployed automatically by kubeadm as static pods. `kubelet` must always be installed manually on each worker node.

```bash
# Download the kubelet binary
wget https://dl.k8s.io/v1.29.0/bin/linux/amd64/kubelet

# Install and run as a systemd service
chmod +x kubelet
mv kubelet /usr/local/bin/

# Enable and start
systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
```

> On kubeadm-provisioned clusters, `kubeadm join` handles the kubelet installation and configuration on worker nodes as part of the join process — but the kubelet binary itself must already be present on the node.

---

### Viewing kubelet Options on a Running Node

Unlike control plane components, kubelet runs as a `systemd` service directly on the node — not as a pod.

```bash
# View running process and effective options
ps -aux | grep kubelet

# View kubelet service configuration
cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# View kubelet config file
cat /var/lib/kubelet/config.yaml
```

---

### Key File Paths

```bash
/var/lib/kubelet/config.yaml          # kubelet configuration file
/etc/systemd/system/kubelet.service.d/ # kubelet systemd drop-in config
/var/lib/kubelet/kubeconfig            # kubelet's kubeconfig to authenticate with apiserver
/etc/kubernetes/pki/                   # TLS certificates used by kubelet
```

---

### Exam Gotchas

- `kubelet` is **never deployed by kubeadm** — it must be installed manually on every worker node. This is the single most important distinction about kubelet.
- `kubelet` runs as a **systemd service** on the node — not as a static pod. You cannot inspect it with `kubectl get pods -n kube-system`.
- If a node shows `NotReady`, the first thing to check is whether `kubelet` is running: `systemctl status kubelet`.
- `kubelet` is the component that actually **creates pods** — the scheduler only decides where; kubelet does the work.
- TLS bootstrapping, certificate generation, and kubelet configuration in depth are covered in the Security section of this course.

---

### In Managed Clusters (AKS / EKS / GKE)

> `kubelet` runs on worker nodes in managed clusters exactly as it does in self-managed clusters — the behaviour is identical. The difference is only in how it is provisioned.

| Aspect | Self-managed | AKS / EKS / GKE |
|---|---|---|
| Installation | Manual on every worker node | Pre-installed by the provider on node pool VMs |
| Configuration | You configure `kubelet` flags and config file | Provider-managed — default configuration applied |
| TLS certificates | You generate and bootstrap manually | Provider handles certificate issuance and rotation |
| Visibility | `systemctl status kubelet` on the node | Node access available via `kubectl debug node` or SSH (if enabled) |

**Practical implication for AKS:**
On AKS, `kubelet` is pre-installed and pre-configured on every node in a node pool. When AKS provisions a new node (e.g., during autoscaling), `kubelet` is already running and registered with the cluster before the node becomes `Ready`. You interact with its effects — pod scheduling, node status — but rarely with `kubelet` directly.

---

