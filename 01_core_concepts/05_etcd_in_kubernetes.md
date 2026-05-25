## Section 5: etcd in Kubernetes

### What it is

This section covers etcd's specific role within a Kubernetes cluster — what data it stores, how it is deployed under two different setup methods, how to query it, and how it is configured in a high-availability environment.

---

### What etcd Stores in Kubernetes

etcd is the **sole persistent storage backend** for the entire Kubernetes cluster state. Every object you create or modify in Kubernetes is ultimately written to etcd.

etcd stores:
- Nodes
- Pods
- ConfigMaps and Secrets
- Deployments, ReplicaSets, DaemonSets, StatefulSets
- ServiceAccounts
- Roles and RoleBindings (RBAC)
- Namespaces and all other Kubernetes API objects

**Critical behaviour:**
Every `kubectl get` command reads data from etcd via `kube-apiserver`. A change to the cluster — adding a node, deploying a pod, scaling a ReplicaSet — is only considered **complete** once it has been written to etcd. Until that write is confirmed, the change does not exist from the cluster's perspective.

---

### etcd Deployment Methods

etcd is deployed differently depending on how the cluster was provisioned:

#### Method 1 — From Scratch (Manual)

You download the etcd binary, install it, and configure it as a `systemd` service on the master node manually.

```bash
# Download and extract
ETCD_VER=v3.5.13
curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz \
  -o etcd.tar.gz
tar xzvf etcd.tar.gz

# Run as a service — key flags
etcd \
  --advertise-client-urls https://<master-node-ip>:2379 \
  --cert-file=/etc/etcd/etcd.crt \
  --key-file=/etc/etcd/etcd.key \
  --trusted-ca-file=/etc/etcd/ca.crt \
  --initial-cluster <node1>=https://<ip1>:2380,<node2>=https://<ip2>:2380
```

Key flags to understand at this stage:

| Flag | Purpose |
|---|---|
| `--advertise-client-urls` | The URL on which etcd listens for client connections — this is what `kube-apiserver` uses to reach etcd. Must be set to the master node IP on port `2379`. |
| `--initial-cluster` | Specifies all etcd instances in the cluster — required for HA multi-node etcd setup. |
| `--cert-file`, `--key-file`, `--trusted-ca-file` | TLS certificate configuration — covered in full in the Security section. |

#### Method 2 — kubeadm (Automated)

kubeadm deploys etcd automatically as a **static pod** in the `kube-system` namespace.

```bash
# Verify etcd is running as a pod
kubectl get pods -n kube-system | grep etcd
```

```
etcd-controlplane   1/1   Running   0   10m
```

Static pod manifest location:
```bash
/etc/kubernetes/manifests/etcd.yaml
```

---

### etcd Directory Structure in Kubernetes

Kubernetes stores all data in etcd under a specific directory hierarchy. The root is `/registry`, under which all Kubernetes constructs are organised:

```
/registry
├── minions/                  ← nodes
├── pods/
│   └── <namespace>/
│       └── <pod-name>
├── replicasets/
├── deployments/
├── configmaps/
├── secrets/
├── serviceaccounts/
├── roles/
└── rolebindings/
```

**List all keys stored in etcd (kubeadm cluster):**

```bash
kubectl exec etcd-controlplane -n kube-system -- \
  sh -c "ETCDCTL_API=3 etcdctl get / \
  --prefix --keys-only --limit=10 \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/server.crt \
  --key /etc/kubernetes/pki/etcd/server.key"
```

> TLS certificate flags are mandatory when querying etcd in a kubeadm cluster. etcd enforces mutual TLS authentication — unauthenticated requests are rejected.

---

### etcdctl with TLS — Certificate Flags

When running `etcdctl` directly against a kubeadm-provisioned etcd, you must always supply the TLS certificates. Certificate files are located at:

```bash
--cacert /etc/kubernetes/pki/etcd/ca.crt
--cert   /etc/kubernetes/pki/etcd/server.crt
--key    /etc/kubernetes/pki/etcd/server.key
```

**Full form of any `etcdctl` command in a kubeadm cluster:**

```bash
ETCDCTL_API=3 etcdctl \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert   /etc/kubernetes/pki/etcd/server.crt \
  --key    /etc/kubernetes/pki/etcd/server.key \
  <command>
```

> Without these flags, `etcdctl` will fail with a TLS handshake error. Always include them when working with etcd in a kubeadm cluster.

---

### High Availability — etcd in Multi-Master Clusters

In a high-availability cluster, multiple master nodes each run their own etcd instance. These instances form a distributed etcd cluster and must be aware of each other.

This is configured via the `--initial-cluster` flag:

```bash
--initial-cluster \
  master1=https://<master1-ip>:2380,\
  master2=https://<master2-ip>:2380,\
  master3=https://<master3-ip>:2380
```

- Port `2379` — client communication (used by `kube-apiserver`)
- Port `2380` — peer communication (used between etcd instances for Raft consensus)

> Full HA configuration and Raft protocol are covered in the High Availability section of this course.

---

### Exam Gotchas

- `kubectl get` reads from etcd via `kube-apiserver` — there is no direct kubectl-to-etcd communication.
- A change is only **complete** once written to etcd — this is fundamental to how Kubernetes consistency works.
- In a kubeadm cluster, the etcd pod is named `etcd-controlplane` (or `etcd-<node-name>`) — not just `etcd`.
- Always use `ETCDCTL_API=3` and supply TLS cert flags when running `etcdctl` inside a kubeadm cluster.
- In the CKA exam, the etcd backup command requires specifying the etcd endpoint and all three cert flags — missing any one of them will cause the command to fail silently or error.

---

### In Managed Clusters (AKS / EKS / GKE)

> etcd is fully managed and inaccessible to the operator — refer to the managed cluster block in Section 4 for the full breakdown.

The one addition relevant to this section: since all Kubernetes state is written to etcd before being considered complete, any `kubectl` operation you perform on AKS follows the same write-to-etcd confirmation path internally — you simply never see or interact with that layer directly. The behaviour is identical; the access is not.

---

