## Section 4: etcd for Beginners

### What it is

etcd is a distributed, reliable key-value store designed to be simple, secure, and fast. In Kubernetes, it serves as the sole persistent storage backend — the complete cluster state lives in etcd.

---

### Storage Models — How etcd Differs

Understanding key-value stores requires contrasting them against the two more familiar storage models:

| | Relational DB (SQL) | Document Store | Key-Value Store (etcd) |
|---|---|---|---|
| Structure | Tables — rows and columns | Documents (JSON/BSON) per record | Key mapped to a value |
| Schema | Strict, predefined schema | Flexible, schema-optional | No schema |
| Complex queries | ✅ Full SQL joins and aggregations | ⚠️ Limited | ❌ Not supported |
| Flexibility | Rigid — adding a column affects all rows | High — each document is independent | Maximum — any value against any key |
| Performance | Good | Good | Very fast — optimised for simple lookup |
| Best suited for | Structured, relational data | Semi-structured data | Simple, fast lookups — configuration, state, metadata |

**Why relational DBs are unsuitable for cluster state:**
Adding a new field (e.g., a new node attribute) requires altering the entire table schema and leaves null cells for all unaffected rows. In a dynamic cluster where every node and pod can have varying attributes, this rigidity is unacceptable.

**Why key-value store is the right fit:**
Each entry is independent. A key can store a simple scalar or a complex JSON document. Adding a new attribute to one entry does not affect any other entry — ideal for storing heterogeneous cluster state.

---

### Installing and Running etcd

```bash
# Download the latest binary from GitHub releases
ETCD_VER=v3.5.13
curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz \
  -o etcd.tar.gz

# Extract
tar xzvf etcd.tar.gz

# Run etcd server
./etcd
```

When started, etcd listens on **port `2379`** by default for client connections.

> In practice, etcd is run as a **systemd service** on bare-metal clusters or as a **static pod** on kubeadm-provisioned clusters — not as a standalone binary.

---

### etcdctl — Command-Line Client

`etcdctl` is the official CLI client for etcd. It is used to store and retrieve key-value pairs and to perform administrative operations.

**Set the API version explicitly (always do this):**

```bash
export ETCDCTL_API=3
```

**Core operations:**

```bash
# Store a key-value pair
etcdctl put name John

# Retrieve a value by key
etcdctl get name

# Delete a key
etcdctl del name

# Check etcdctl version and API version in use
etcdctl version
```

**Example output of `etcdctl get`:**

```
name
John
```

> etcd returns the key on the first line and the value on the second line.

---

### API v2 vs API v3 — Command Differences

The transcript references older blogs and documentation that use v2 commands. Know the difference to avoid confusion:

| Operation | API v2 (deprecated) | API v3 (current) |
|---|---|---|
| Write a key | `etcdctl set key value` | `etcdctl put key value` |
| Read a key | `etcdctl get key` | `etcdctl get key` |
| Delete a key | `etcdctl rm key` | `etcdctl del key` |
| Transactions | ❌ Not supported | ✅ Supported |

**Check which API version is active:**

```bash
etcdctl version
```

```
etcdctl version: 3.5.13
API version: 3.5
```

> If you encounter older articles using `set` and `rm` commands, they are targeting the v2 API. Always use v3 in current clusters.

---

### etcd Version Timeline — What Matters

| Version | Date | What changed |
|---|---|---|
| v2.0 | Feb 2015 | First stable release. Raft consensus algorithm redesigned. Supported 1000+ writes/second. |
| v3.0 | Jan 2017 | Major API overhaul — `put/del` replaced `set/rm`. Added transactions. Significant performance improvements. |
| v3.5 | Jun 2021 | Current stable release line. Further performance, security, and observability improvements. |
| — | Nov 2020 | etcd became a **CNCF graduated project**. |

> v3.6 is currently in active development as of 2025. All production Kubernetes clusters run v3.x. v2 is fully deprecated.

---

### Exam Gotchas

- etcd listens on port **`2379`** for client requests and port **`2380`** for peer-to-peer communication in a cluster.
- Always export `ETCDCTL_API=3` before running `etcdctl` commands in the exam — the default may vary by environment.
- etcd backup uses `etcdctl snapshot save` and restore uses `etcdctl snapshot restore` — covered in the Cluster Maintenance section.
- In a kubeadm cluster, etcd runs as a static pod — its manifest is at `/etc/kubernetes/manifests/etcd.yaml`.
- The `--advertise-client-urls` flag in the etcd configuration defines the endpoint that clients (including `kube-apiserver`) use to reach etcd — relevant for backup and HA configurations.

---

### In Managed Clusters (AKS / EKS / GKE)

> **etcd is fully managed by the cloud provider — you have zero direct access to it.**

| Aspect | Self-managed cluster | AKS / EKS / GKE |
|---|---|---|
| etcd hosting | You provision and manage etcd nodes | Runs on control plane nodes managed entirely by the provider |
| Backup / restore | Your responsibility — `etcdctl snapshot save/restore` | Provider handles automated backups — no manual intervention required |
| Direct access | Full access via `etcdctl` with TLS certs | No access — etcd endpoint is not exposed to the cluster operator |
| HA configuration | You configure the etcd cluster size and Raft quorum | Provider manages HA and quorum internally |
| Failure recovery | You restore from snapshot manually | Provider handles etcd failure and recovery transparently |

**Practical implication for AKS:**
When working with AKS, you will never run `etcdctl` against the cluster's etcd directly. All state inspection is done through the Kubernetes API (`kubectl`) — not etcd. The etcd layer is completely abstracted.

**Why you still need to understand etcd deeply:**
- CKA exam tests etcd on self-managed clusters — backup and restore is a near-certain exam task
- When debugging Kubernetes API failures or data inconsistencies, understanding that etcd is the source of truth helps you reason about the failure correctly
- If you ever work with self-managed clusters (on-premise, Azure Stack HCI, or bare-metal), etcd management is your direct responsibility

---