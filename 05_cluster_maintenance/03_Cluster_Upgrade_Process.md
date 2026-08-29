## Cluster Upgrade Process

### What it is

The procedure for upgrading a Kubernetes cluster from one minor version to the next using `kubeadm`, covering version skew policy, upgrade order, worker node strategies, and the exact command sequence.

---

### Version Skew Policy

Version skew policy defines the maximum supported version difference between Kubernetes components. kube-apiserver sets the version ceiling — no component may run a higher minor version than the API server, since all components communicate through it. Control plane components (kube-controller-manager, kube-scheduler) are allowed up to one minor version behind, while node-level components (kubelet, kube-proxy) are allowed up to two minor versions behind, reflecting their position further down the communication chain. This controlled tolerance is what makes live rolling upgrades possible — components can be upgraded incrementally without requiring a full cluster freeze.

`kube-apiserver` is the highest-versioned component in the control plane. All other components must not exceed it.

> **What does N mean?**
> `N` is just a placeholder for whatever minor version `kube-apiserver` is currently running.
> If `kube-apiserver` is at **v1.36**, then:
> - `N` = v1.36
> - `N-1` = v1.35 (one version behind)
> - `N-2` = v1.34 (two versions behind)
> - `N+1` = v1.37 (one version ahead — only `kubectl` is allowed this)
> Only the **minor** version number changes — the major version (`1`) stays fixed.

| Component | Allowed Versions Relative to kube-apiserver (N) |
|---|---|
| `kube-apiserver` | N (highest in cluster) |
| `kube-controller-manager` | N or N-1 |
| `kube-scheduler` | N or N-1 |
| `kubelet` | N, N-1, or N-2 |
| `kube-proxy` | N, N-1, or N-2 (matches kubelet on same node) |
| `kubectl` | N+1, N, or N-1 |

> `kubectl` is the only component that can run **ahead** of the API server — useful when upgrading kubectl before the cluster.

**Example with kube-apiserver at v1.36:**

| Component | Allowed |
|---|---|
| kube-controller-manager | v1.36 or v1.35 |
| kube-scheduler | v1.36 or v1.35 |
| kubelet | v1.36, v1.35, or v1.34 |
| kube-proxy | v1.36, v1.35, or v1.34 |
| kubectl | v1.37, v1.36, or v1.35 |

This skew policy is what enables **live rolling upgrades** — components can be upgraded one at a time without requiring a full cluster freeze.

---

### When to Upgrade

Kubernetes supports the **3 most recent minor versions** (N-2 policy). When a new minor version is released, the oldest supported version drops out of support.

```
Currently supported: v1.34, v1.35, v1.36
When v1.37 releases:  v1.34 drops out — upgrade before this point
```

> Upgrade your cluster **before** your current version falls out of the support window. Running an unsupported version means no security patches.

---

### Upgrade Path — One Minor Version at a Time

**Never skip minor versions.** Kubernetes only supports upgrading one minor version at a time.

```
v1.34 → v1.35 → v1.36   ✅ correct
v1.34 → v1.36            ❌ not supported
```

---

### Upgrade Order

```
1. Upgrade kubeadm on control plane node
2. Upgrade control plane (master node)
   ├── kubeadm upgrade plan       ← validates etcd health, cert status, compatibility
   ├── kubeadm upgrade apply      ← upgrades API server, controller-manager, scheduler, CoreDNS, etcd
   │                                 (also auto-renews control plane certificates)
   └── upgrade kubelet + kubectl on master → restart kubelet
3. Upgrade worker nodes (one at a time)
   ├── drain node                 ← from control plane
   ├── upgrade kubeadm on worker
   ├── kubeadm upgrade node       ← updates node kubelet configuration
   ├── upgrade kubelet + kubectl  ← then upgrade binaries
   ├── restart kubelet
   └── uncordon node              ← from control plane
```

> **kubeadm does NOT upgrade kubelet.** Kubelet must be upgraded manually on every node (master and workers).

---

### What Happens During Control Plane Upgrade

When the master is being upgraded, control plane components go down briefly:

| Impact | Detail |
|---|---|
| `kube-apiserver` down | No `kubectl` access; no API calls |
| Controller manager down | Failed pods are NOT recreated during this window |
| Scheduler down | No new pod scheduling |
| Worker nodes | **Unaffected** — running pods continue serving traffic |
| Running applications | **Unaffected** — users see no downtime |

This window is typically under 1 minute with `kubeadm`.

---

### Worker Node Upgrade Strategies

| Strategy | Downtime | Use Case |
|---|---|---|
| All nodes at once | Yes — all pods evicted simultaneously | Dev/test clusters only |
| One node at a time (rolling) | No — workloads shift between nodes | Standard production upgrade |
| Add new nodes, drain old | No — clean separation of versions | Cloud environments; preferred in AKS/EKS |

---

### Full Upgrade Command Sequence (kubeadm)

> Example: upgrading from **v1.35.x → v1.36.x**

#### Step 1 — Upgrade kubeadm on the control plane node

```bash
# Switch to the new Kubernetes apt repository for the target minor version
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' \
  | tee /etc/apt/sources.list.d/kubernetes.list

apt-get update

# Unhold, upgrade, re-hold kubeadm
apt-mark unhold kubeadm
apt-get install -y kubeadm=1.36.3-1.1
apt-mark hold kubeadm

# Verify
kubeadm version
```

#### Step 2 — Plan the upgrade

```bash
kubeadm upgrade plan
```

Output shows:
- Current cluster version
- kubeadm version
- Latest available stable version
- What each control plane component will upgrade to
- Reminder to manually upgrade kubelet after

#### Step 3 — Apply the upgrade to the control plane

```bash
kubeadm upgrade apply v1.36.3
```

This pulls required images and upgrades: `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `kube-proxy`, and CoreDNS/etcd if needed.

> **Certificate renewal:** `kubeadm upgrade apply` automatically renews all control plane certificates as part of the upgrade. If your certificates are close to expiry and you are not ready to upgrade yet, you can renew them manually with `kubeadm certs renew all`.

#### Step 4 — Upgrade kubelet and kubectl on the master node

```bash
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.36.3-1.1 kubectl=1.36.3-1.1
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet
```

Verify:
```bash
kubectl get nodes
# Master now shows v1.36.3; workers still show v1.35.x
```

> `kubectl get nodes` shows the **kubelet version** on each node — not the API server version. This is a common exam trap.

#### Step 5 — Upgrade each worker node (repeat per node)

```bash
# On the CONTROL PLANE node — drain the worker
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data

# SSH into worker node-1
ssh node-1

# Switch repository and upgrade kubeadm on the worker
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' \
  | tee /etc/apt/sources.list.d/kubernetes.list

apt-get update
apt-mark unhold kubeadm
apt-get install -y kubeadm=1.36.3-1.1
apt-mark hold kubeadm

# Upgrade node configuration (worker nodes use this, NOT upgrade apply)
kubeadm upgrade node

# Upgrade kubelet and kubectl
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.36.3-1.1 kubectl=1.36.3-1.1
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet

# Back on the CONTROL PLANE node — uncordon
exit
kubectl uncordon node-1
```

Repeat for node-2, node-3, etc.

---

### Key Command Differences: Master vs Worker

| Command | Master | Worker |
|---|---|---|
| `kubeadm upgrade apply v1.x.y` | ✅ Used | ❌ Not used |
| `kubeadm upgrade node` | ❌ Not used | ✅ Used |
| Drain before upgrade | Optional (no workloads usually) | ✅ Required |
| Uncordon after upgrade | N/A | ✅ Required |

---

### When to Use Which (Worker Strategies)

| Situation | Strategy |
|---|---|
| Cloud environment (AKS, EKS, GKE) | Add new nodes, drain old — most reliable |
| On-prem, enough capacity to absorb load | Rolling one-at-a-time |
| Dev cluster, downtime acceptable | All at once |
| PodDisruptionBudgets present | Rolling — drain respects PDBs |

---

### Real-World Usage

**AKS cluster upgrade:** `az aks upgrade --resource-group <rg> --name <cluster> --kubernetes-version 1.36.3` — AKS handles the entire drain → upgrade → uncordon cycle per node automatically. For node pools: `az aks nodepool upgrade`. Monitor with `az aks show` — stalled upgrades are almost always PDB violations blocking drain.

**kubeadm clusters in production:** Always run `kubeadm upgrade plan` first — it validates etcd health, certificate status, and component compatibility before touching anything. Never run `upgrade apply` without reviewing plan output.

**Version pinning with apt-mark hold:** Always `apt-mark hold kubelet kubeadm kubectl` after installation to prevent accidental upgrades via `apt-get upgrade`.

---

### Exam Gotchas

- `kubectl get nodes` shows **kubelet version**, not API server version — do not confuse these when checking upgrade status.
- `kubeadm upgrade apply` is only run on the **master/control plane** node. Worker nodes use `kubeadm upgrade node`.
- **kubeadm does not upgrade kubelet** — this is always a manual step on every node.
- You must upgrade **kubeadm itself** before running `kubeadm upgrade apply` — the tool version must match the target cluster version.
- Upgrade only **one minor version at a time** — skipping minor versions is unsupported.
- After draining a worker, always **uncordon** it after the upgrade — pods do not return automatically but the node must be marked schedulable.
- `apt-mark unhold` before upgrading packages, `apt-mark hold` after — forgetting the unhold will silently skip the upgrade.
- The repository URL changes per minor version (`/v1.36/`, `/v1.35/`) — update the apt source list before installing the new version.

---

### In Managed Clusters (AKS / EKS / GKE)

- The control plane upgrade is fully managed — you never touch `kubeadm upgrade apply` or kubelet on master nodes.
- AKS node pool upgrades handle drain → OS upgrade → kubelet upgrade → uncordon per node automatically.
- AKS respects PDBs during node pool upgrades — a stuck upgrade is almost always a PDB violation.
- Max surge (`--max-surge`) controls how many extra nodes AKS provisions during upgrade to reduce downtime window.
- GKE Autopilot manages all node upgrades entirely — no manual intervention possible or needed.
- EKS managed node groups handle rolling upgrades; self-managed node groups require manual drain/upgrade/uncordon.

