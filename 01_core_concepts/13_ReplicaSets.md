## 13 — ReplicaSets

### What it is

A **ReplicaSet** ensures a specified number of pod replicas are running at all times. If a pod fails, the ReplicaSet automatically creates a replacement. It is the current recommended replication mechanism, replacing the older **Replication Controller**.

---

### Why ReplicaSets Exist

| Problem | ReplicaSet solution |
|---|---|
| Single pod fails → application goes down | Maintains multiple pod instances — if one fails, others continue serving |
| Single pod running → no automatic recovery | Even with `replicas: 1`, a failed pod is automatically replaced |
| Traffic increases → need more instances | Scale replicas up to distribute load across pods and nodes |
| Node runs out of resources | ReplicaSet spans multiple nodes — pods are scheduled wherever capacity exists |

---

### Replication Controller vs ReplicaSet

| | Replication Controller | ReplicaSet |
|---|---|---|
| API version | `v1` | `apps/v1` |
| Status | Legacy — being replaced | Current recommended approach |
| `selector` field | Optional — defaults to pod template labels | **Mandatory** — must be explicitly defined |
| Label matching | Basic equality only | `matchLabels` and `matchExpressions` (richer options) |
| Manages pre-existing pods | No | ✅ Yes — can adopt pods created before the ReplicaSet |

> Use ReplicaSet in all new implementations. Replication Controller is covered here for context only.

---

### ReplicaSet Definition File

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-rs
  labels:
    app: myapp
    type: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        type: frontend
    spec:
      containers:
        - name: nginx-container
          image: nginx
```

**Structure breakdown:**

```
spec
├── replicas        ← desired number of pods
├── selector        ← how the ReplicaSet identifies which pods to manage
│   └── matchLabels
└── template        ← pod definition used to create new replicas
    ├── metadata
    │   └── labels  ← MUST match selector.matchLabels
    └── spec
        └── containers
```

---

### Replication Controller Definition File (Legacy Reference)

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: myapp-rc
  labels:
    app: myapp
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: nginx-container
          image: nginx
```

> No `selector` field required — it defaults to the pod template labels.

---

### Labels and Selectors — Why They Matter

A ReplicaSet monitors pods by matching their labels against `selector.matchLabels`. This mechanism serves two purposes:

**1. Monitoring existing pods:**
If pods with matching labels already exist in the cluster before the ReplicaSet is created, the ReplicaSet adopts them and counts them toward the desired replica count. No new pods are created if the count is already satisfied.

**2. Creating replacement pods:**
If a monitored pod fails, the ReplicaSet uses the `template` section to create a replacement. This is why `template` is required even when all replicas already exist — the ReplicaSet needs the template to recover from future failures.

```
selector.matchLabels: app: myapp
         │
         ▼
ReplicaSet monitors all pods with label app=myapp
         │
         ├── 3 matching pods exist → desired state met, no action
         ├── 1 pod fails → creates 1 new pod from template
         └── 0 pods exist → creates 3 new pods from template
```

> `selector.matchLabels` in the ReplicaSet **must match** `template.metadata.labels`. A mismatch causes an immediate validation error.

---

### kubectl Commands

```bash
# Create a ReplicaSet
kubectl create -f rs-definition.yaml

# List ReplicaSets
kubectl get replicaset

# Detailed information
kubectl describe replicaset myapp-rs

# List pods created by the ReplicaSet (named with RS prefix)
kubectl get pods

# Delete a ReplicaSet (also deletes all its pods)
kubectl delete replicaset myapp-rs

# Replace / update a ReplicaSet from a modified file
kubectl replace -f rs-definition.yaml

# Scale without modifying the file
kubectl scale --replicas=6 -f rs-definition.yaml
kubectl scale --replicas=6 replicaset myapp-rs
```

---

### Scaling a ReplicaSet — Three Methods

| Method | Command | Updates the file? |
|---|---|---|
| Edit the file and replace | Update `replicas` in YAML → `kubectl replace -f rs-definition.yaml` | ✅ Yes |
| Scale via file reference | `kubectl scale --replicas=6 -f rs-definition.yaml` | ❌ No |
| Scale via name | `kubectl scale --replicas=6 replicaset myapp-rs` | ❌ No |

> `kubectl scale` with a file as input scales the live ReplicaSet but does **not** update the replica count in the file. The file remains at the original value — a common source of confusion.

---

### Exam Gotchas

- ReplicaSet `apiVersion` is `apps/v1` — not `v1`. Using `v1` produces: `no matches for kind "ReplicaSet"`.
- `selector.matchLabels` is **mandatory** for ReplicaSet — omitting it causes a validation error.
- `selector.matchLabels` must exactly match `template.metadata.labels` — a mismatch causes an immediate error on creation.
- Deleting a ReplicaSet **also deletes all pods** it manages. To keep pods, delete with `--cascade=orphan`.
- Pods created by a ReplicaSet are named `<replicaset-name>-<random-suffix>` — e.g., `myapp-rs-x7k2p`.
- ReplicaSet does not support rolling updates — use **Deployment** for that. Deployments manage ReplicaSets under the hood.
- `kubectl scale` does not modify your YAML file — always update the file manually if you want the desired state persisted in version control.

---

### In Managed Clusters (AKS / EKS / GKE)

> ReplicaSet behaviour is identical across managed and self-managed clusters. In practice, you will rarely create ReplicaSets directly — **Deployments** are the standard workload object, and they manage ReplicaSets automatically.

---

