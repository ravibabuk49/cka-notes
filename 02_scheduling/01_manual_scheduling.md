## 01 — Manual Scheduling

### What it is
Manual scheduling is the direct assignment of a Pod to a Node by explicitly setting the `nodeName` field in the Pod spec, bypassing the kube-scheduler entirely. The scheduler itself uses this same field — it selects a node via its algorithm then writes the node name by creating a `Binding` object; manual scheduling replicates that outcome directly.

---

### How the Scheduler Uses `nodeName` Internally

1. Scheduler watches for Pods where `spec.nodeName` is **not set** — these are scheduling candidates.
2. It runs filtering + scoring algorithms to select a node.
3. It creates a **`Binding`** object (API resource: `v1/Binding`) which sets `spec.nodeName` on the Pod.

Without a running scheduler, Pods with no `nodeName` remain in **`Pending`** state indefinitely.

> **Method selection:** `spec.nodeName` is the assignment mechanism for **Pods that do not yet exist** — it is declared at creation time and evaluated immediately by the API server. The Binding API is the equivalent mechanism for **Pods already in `Pending` state** — it replicates the scheduler's final binding step by POSTing a `Binding` object to the Pod's binding subresource, which causes the API server to set `spec.nodeName` on the live object.

---

### Method 1 — Set `nodeName` at Creation Time

Only valid **before** the Pod exists. Directly set in the Pod manifest:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeName: node01          # scheduler is bypassed entirely
  containers:
    - name: nginx
      image: nginx
```

```bash
kubectl apply -f pod.yaml
kubectl get pod nginx -o wide   # verify NODE column
```

> **Constraint:** `nodeName` cannot be modified on an already-created Pod via `kubectl edit` or `kubectl patch` — the API server rejects the change.

---

### Method 2 — Binding Object for Already-Running Pods

Used when a Pod already exists in `Pending` state and `nodeName` cannot be patched directly. This method replicates exactly what `kube-scheduler` does internally — it POSTs a `Binding` object to the Pod's binding subresource, which causes the API server to set `spec.nodeName` on the Pod.

---

#### Step 1 — Confirm the Pod is Pending and identify the target node

```bash
kubectl get pods -o wide                  # STATUS = Pending, NODE = <none>
kubectl get nodes                         # pick a Ready node name, e.g. node01
```

---

#### Step 2 — Write the Binding manifest (YAML)

```yaml
apiVersion: v1
kind: Binding
metadata:
  name: nginx                 # must match the Pod name exactly
  namespace: default          # must match the Pod namespace
target:
  apiVersion: v1
  kind: Node
  name: node01                # the node you want to assign to
```

Key fields:
| Field | Purpose |
|---|---|
| `metadata.name` | Must match the Pod name |
| `metadata.namespace` | Must match the Pod namespace |
| `target.kind` | Always `Node` |
| `target.name` | Name of the destination node |

---

#### Step 3 — Convert YAML to JSON

The Binding subresource API **only accepts JSON**. Convert manually or with a tool:

```bash
# Using python (available in most CKA exam environments)
python3 -c "
import json, yaml
with open('binding.yaml') as f:
    print(json.dumps(yaml.safe_load(f), indent=2))
"
```

Resulting JSON:
```json
{
  "apiVersion": "v1",
  "kind": "Binding",
  "metadata": {
    "name": "nginx",
    "namespace": "default"
  },
  "target": {
    "apiVersion": "v1",
    "kind": "Node",
    "name": "node01"
  }
}
```

---

#### Step 4 — POST to the binding subresource API

**API endpoint pattern:**
```
POST /api/v1/namespaces/{namespace}/pods/{pod-name}/binding
```

**Using curl with in-line JSON:**
```bash
curl -X POST \
  http://localhost:8080/api/v1/namespaces/default/pods/nginx/binding \
  -H "Content-Type: application/json" \
  -d '{
    "apiVersion": "v1",
    "kind": "Binding",
    "metadata": { "name": "nginx", "namespace": "default" },
    "target": { "apiVersion": "v1", "kind": "Node", "name": "node01" }
  }'
```

**Using curl with a JSON file:**
```bash
curl -X POST \
  http://localhost:8080/api/v1/namespaces/default/pods/nginx/binding \
  -H "Content-Type: application/json" \
  -d @binding.json
```

> `localhost:8080` assumes you are running on the control plane node where the API server's insecure port is accessible, or via `kubectl proxy`. In kubeadm clusters the secure port is `6443` — use `kubectl proxy` to avoid handling certificates in the exam.

**Preferred exam approach — use `kubectl proxy`:**
```bash
# Terminal 1: start proxy (forwards to API server with your kubeconfig credentials)
kubectl proxy --port=8001 &

# Terminal 2: POST the binding
curl -X POST \
  http://localhost:8001/api/v1/namespaces/default/pods/nginx/binding \
  -H "Content-Type: application/json" \
  -d '{
    "apiVersion": "v1",
    "kind": "Binding",
    "metadata": { "name": "nginx", "namespace": "default" },
    "target": { "apiVersion": "v1", "kind": "Node", "name": "node01" }
  }'
```

---

#### Step 5 — Verify

```bash
kubectl get pod nginx -o wide
# NODE column should now show node01, STATUS should move to Running
```

A successful POST returns HTTP `201 Created` with the Binding object in the response body. Any `400` or `409` response indicates a name/namespace mismatch or the Pod is already bound.

---

### Exam Gotchas

- **`nodeName` is immutable post-creation.** You cannot patch it on an existing Pod. The Binding API is the only mechanism for an already-created Pod.
- **Pending Pods → no scheduler** is the classic symptom. Check with `kubectl get pods` — if all pods are `Pending` and there are no scheduling events, `kube-scheduler` may not be running.
- **`nodeName` skips all scheduling constraints** — taints, tolerations, affinity rules, resource checks are all bypassed. The Pod lands on the node regardless.
- **Binding API requires JSON body**, not YAML. Forgetting to convert is a common trap.
- **`nodeName` vs `nodeSelector`** — `nodeName` is a hard exact match by node name; `nodeSelector` uses labels and goes through the scheduler. Do not conflate them.

---

### In Managed Clusters (AKS / EKS / GKE)

- `kube-scheduler` is always running and managed by the control plane — the "no scheduler" scenario is self-managed / kubeadm specific.
- `nodeName` override still works in AKS node pools but bypasses cluster-autoscaler awareness and any taint/toleration policies on system node pools. Avoid in production.

---

### Practice Test Lesson Learned

<!-- Fill in after completing the lab -->