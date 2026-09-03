## Backup and Restore Methodologies

### What it is

The strategies and tooling for protecting Kubernetes cluster state and recovering from failure — covering resource configuration backups via the API server, etcd snapshot-based backup and restore, and etcd file-level backup using `etcdutl`.

---

### What to Back Up

| Backup Target | What It Contains | Priority |
|---|---|---|
| Resource configuration (YAMLs) | Deployments, Services, ConfigMaps, Secrets, RBAC, etc. | ✅ Always |
| etcd backup (snapshot or file-level) | Complete cluster state — all objects, all namespaces | ✅ Always |
| Persistent Volumes | Application data on disk | Context-dependent |

---

### Method 1 — Resource Configuration Backup

#### Declarative approach (preferred)

Store all object definition YAML files in a Git repository. This is the most recoverable approach — losing the entire cluster means simply re-applying the files.

```bash
kubectl apply -f /path/to/manifests/
```

**Limitation:** Only covers objects that were created declaratively. Any object created imperatively (without a YAML) will not be captured.

#### API server query (safety net for imperative objects)

Query the live cluster via the Kubernetes API and dump all resource configurations to YAML:

```bash
# Basic — misses CRDs, PVs, PVCs, and non-namespaced resources
kubectl get all --all-namespaces -o yaml > all-resources.yaml

# Namespaced resources — captures all namespaced resource types the cluster knows about
kubectl get $(kubectl api-resources --verbs=list --namespaced=true -o name \
  | tr '\n' ',') \
  --all-namespaces -o yaml > all-namespaced-resources.yaml

# Cluster-scoped resources — PersistentVolumes, ClusterRoles, Namespaces, etc.
kubectl get $(kubectl api-resources --verbs=list --namespaced=false -o name \
  | tr '\n' ',') \
  -o yaml > all-cluster-scoped-resources.yaml
```

> `kubectl get all` does **not** get all resources — it only covers the core resource group (pods, deployments, services, etc.). Use the `api-resources` commands above to capture all namespaced and cluster-scoped resources separately for a complete dump.

#### Tooling — Velero

**Velero** (formerly Ark by Heptio) is the standard open-source tool for Kubernetes backup via the API. It supports:
- Scheduled backups of namespaces or label-selected resources
- Backup to object storage (S3, Azure Blob, GCS)
- Restore to the same or a different cluster
- PersistentVolume snapshots via CSI or cloud provider plugins

```bash
velero backup create my-backup --include-namespaces default
velero restore create --from-backup my-backup
```

---

### Method 2 — etcd Snapshot Backup

The etcd database contains the complete state of the cluster. A snapshot captures everything — all objects across all namespaces, cluster configuration, and RBAC — at a point in time.

#### Verify etcdctl version

Before running any etcd commands, confirm the tool version and API version:

```bash
etcdctl version
```

Expected output in Kubernetes lab environments:
```
etcdctl version: 3.5.16
API version: 3.5
```

Always set API version 3 explicitly before using `etcdctl`:
```bash
export ETCDCTL_API=3
```

#### Tool selection — `etcdctl` vs `etcdutl`

Two separate binaries exist as of etcd v3.5+:

| Operation | Tool | Online/Offline | Notes |
|---|---|---|---|
| `snapshot save` | `etcdctl` | Online | Connects to running etcd — requires endpoint + certs |
| `snapshot status` | `etcdutl` *(preferred)* / `etcdctl` | Offline | Both work; `etcdutl` preferred in v3.5+ |
| `snapshot restore` | `etcdutl` *(preferred)* / `etcdctl` | Offline | Both work; `etcdutl` preferred in v3.5+ |
| `backup` | `etcdutl` | Offline | Raw file-level copy of data dir + WAL files |

> `etcdctl snapshot restore` and `etcdctl snapshot status` still function in etcd v3.5+ but `etcdutl` is the **preferred** tool for offline operations. Use `etcdutl` by default for restore and status — it does not require endpoint or cert flags.

#### Finding etcd certificate paths

In kubeadm clusters, etcd runs as a static pod. Inspect its manifest to find cert paths:
```bash
cat /etc/kubernetes/manifests/etcd.yaml
```
Look for these flags:
```
--trusted-ca-file     → CA cert
--cert-file           → Server cert
--key-file            → Server key
--listen-client-urls  → Endpoint (typically https://127.0.0.1:2379)
```

Default paths in kubeadm clusters:
```
CA:       /etc/kubernetes/pki/etcd/ca.crt
Cert:     /etc/kubernetes/pki/etcd/server.crt
Key:      /etc/kubernetes/pki/etcd/server.key
Endpoint: https://127.0.0.1:2379
```

#### Taking a snapshot

```bash
ETCDCTL_API=3 etcdctl snapshot save /opt/snapshot-pre-boot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

#### Verifying a snapshot

```bash
etcdutl snapshot status /opt/snapshot-pre-boot.db --write-out=table
```

Output shows: snapshot hash, revision, total keys, total size. Use this to verify snapshot integrity before attempting a restore.

---

### Method 3 — etcd File-level Backup (`etcdutl backup`)

An alternative to snapshot-based backup. `etcdutl backup` performs a **raw file-level copy** of the etcd data directory and WAL (Write-Ahead Log) files — no running etcd instance required.

```bash
etcdutl backup \
  --data-dir /var/lib/etcd \
  --backup-dir /backup/etcd-backup
```

This copies the backend database and WAL files directly to the target location.

#### Restoring from a file-level backup

No special restore command — stop etcd first, clear or use a new data directory, copy the backup contents, then restart etcd:

```bash
# Stop etcd first (kubeadm: move manifest; systemd: systemctl stop etcd)

# Clear the existing data directory to avoid stale data conflicts
rm -rf /var/lib/etcd/*

# Copy backup files
cp -r /backup/etcd-backup/* /var/lib/etcd/

# Restart etcd (kubeadm: move manifest back; systemd: systemctl start etcd)
```

> If you prefer not to wipe the existing data directory, restore to a new path and update `--data-dir` in the etcd config or static pod manifest — same pattern as snapshot restore.

#### Snapshot backup vs file-level backup

| | `etcdctl snapshot save` | `etcdutl backup` |
|---|---|---|
| etcd running? | ✅ Required | ❌ Not required |
| Output format | Single `.db` file | Directory of data + WAL files |
| Restore tool | `etcdutl snapshot restore` | Manual copy + restart |
| Use case | Standard CKA exam backup | Offline/maintenance window backup |
| Cert flags needed? | ✅ Yes | ❌ No |

---

### Restoring from an etcd Snapshot

#### Restore process differs by cluster setup

| Setup | How etcd runs | Stop method |
|---|---|---|
| kubeadm | Static pod (`/etc/kubernetes/manifests/etcd.yaml`) | Move manifest out of directory |
| Systemd (bare metal) | `etcd.service` systemd unit | `systemctl stop etcd` |
| Managed (AKS/EKS/GKE) | Control plane managed by cloud provider | etcd access not available |

---

#### Full restore sequence — kubeadm cluster

**Step 1 — Stop kube-apiserver** (prevents writes to etcd during restore)
```bash
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
# kubelet detects the manifest is gone and stops the pod
# Wait ~30s for the pod to terminate
```

**Step 2 — Restore the snapshot to a new data directory**
```bash
etcdutl snapshot restore /opt/snapshot-pre-boot.db \
  --data-dir=/var/lib/etcd-from-backup
```

> This creates a **new** etcd data directory. It does not touch the existing one. A new cluster token is also generated to prevent accidental merging with an existing cluster.

**Step 3 — Update the etcd static pod manifest**

Edit `/etc/kubernetes/manifests/etcd.yaml` and update two places:

```yaml
# 1. Update the --data-dir flag
- --data-dir=/var/lib/etcd-from-backup

# 2. Update the hostPath volume to point to the new directory
volumes:
  - hostPath:
      path: /var/lib/etcd-from-backup   # ← change this
      type: DirectoryOrCreate
    name: etcd-data
```

**Step 4 — Restart etcd and kube-apiserver**
```bash
# kubelet auto-restarts etcd since the manifest changed
# Restore kube-apiserver
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# Verify
watch kubectl get pods -n kube-system
```

---

#### Full restore sequence — systemd cluster

```bash
# Stop API server
systemctl stop kube-apiserver

# Restore snapshot
etcdutl snapshot restore /opt/snapshot-pre-boot.db \
  --data-dir=/var/lib/etcd-from-backup

# Update etcd config to point to new data dir
# Edit /etc/etcd/etcd.conf or the systemd unit file:
# --data-dir=/var/lib/etcd-from-backup

# Reload and restart
systemctl daemon-reload
systemctl restart etcd
systemctl start kube-apiserver
```

---

### Backup Method Comparison

| Method | Captures Imperative Objects | etcd Access Required | Works in Managed Clusters | Tooling |
|---|---|---|---|---|
| Git + declarative YAMLs | ❌ No | ❌ No | ✅ Yes | kubectl apply |
| kubectl API query | ✅ Yes | ❌ No | ✅ Yes | kubectl / Velero |
| etcd snapshot | ✅ Yes | ✅ Yes (online) | ❌ No | etcdctl snapshot save |
| etcd file-level backup | ✅ Yes | ❌ No (offline) | ❌ No | etcdutl backup |

> In managed clusters (AKS, EKS, GKE), the etcd control plane is not accessible. Use API server querying or Velero for backups.

---

### When to Use Which

| Scenario | Recommended Approach |
|---|---|
| Self-managed cluster, full DR capability needed | etcd snapshot + Velero |
| Managed cluster (AKS/EKS/GKE) | Velero with cloud object storage |
| Pre-upgrade safety backup | etcd snapshot (`etcdctl snapshot save`) |
| etcd down / maintenance window backup | File-level backup (`etcdutl backup`) |
| Migrating workloads to a new cluster | kubectl API query or Velero |
| Team uses mixed imperative/declarative | API query + Git for declarative objects |

---

### Real-World Usage

**Pre-upgrade snapshot:** Always take an etcd snapshot before running `kubeadm upgrade apply`. If the upgrade corrupts cluster state, restore is a single command:
```bash
ETCDCTL_API=3 etcdctl snapshot save /opt/pre-upgrade-$(date +%F).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

**AKS backup:** AKS does not expose etcd. Use Velero with Azure Blob Storage as the backup target:
```bash
velero install --provider azure --bucket <blob-container> \
  --secret-file ./credentials-velero \
  --backup-location-config resourceGroup=<rg>,storageAccount=<sa>
```

**Scheduled backups:** In production, etcd snapshots should be automated via a CronJob or external scheduler with retention policies. Velero supports this natively with `velero schedule create`.

---

### Exam Gotchas

- Always `export ETCDCTL_API=3` before any `etcdctl` command — without it, v2 API is used and commands fail or return wrong results.
- `etcdctl snapshot save` requires `--endpoints`, `--cacert`, `--cert`, `--key` — missing any one fails the command.
- `etcdutl snapshot restore` and `etcdutl snapshot status` are **preferred** over their `etcdctl` equivalents in etcd v3.5+ — but both still work. On the CKA exam, either is acceptable; `etcdutl` is the safer choice.
- After restore, **both** the `--data-dir` flag and the `hostPath` volume in `etcd.yaml` must be updated — changing only one leaves etcd pointing at the wrong directory.
- `kubectl get all` does **not** capture all resources — know this when asked about complete cluster backup via the API.
- In kubeadm clusters, stopping kube-apiserver means moving its manifest out of `/etc/kubernetes/manifests/` — there is no `systemctl stop kube-apiserver` because it runs as a static pod.
- After restore, etcd initialises a **new cluster** with a new cluster token — this is by design to prevent stale members from joining.

---

### In Managed Clusters (AKS / EKS / GKE)

- etcd is not accessible in any managed Kubernetes offering — etcd snapshot commands cannot be run.
- AKS backs up etcd internally on its own schedule — this is not user-configurable or visible.
- For user-controlled backup in AKS: use **Velero** with Azure Blob Storage, or **Azure Backup for AKS** (GA since 2023) which integrates natively with AKS and supports scheduled backup and restore of workloads and PVs.
- `az aks backup` CLI commands are available for Azure Backup for AKS integration.

