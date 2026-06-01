## 20 — kubectl explain and kubectl apply Internals

### What it is

Two essential operational tools for working declaratively with Kubernetes: `kubectl explain` for discovering resource fields without leaving the terminal, and a deep look at how `kubectl apply` tracks and reconciles state using a three-way merge strategy.

---

## kubectl explain

### What it does

`kubectl explain` provides inline documentation for any Kubernetes resource and its fields directly in the terminal — no browser required. Essential during the exam when you need to look up a field name or type quickly.

---

### kubectl api-resources — Discover Resource Names

Before explaining a resource, use `kubectl api-resources` to find the correct resource name, short name, API group, and version:

```bash
kubectl api-resources
```

```
NAME                    SHORTNAMES   APIVERSION   NAMESPACED   KIND
pods                    po           v1           true         Pod
deployments             deploy       apps/v1      true         Deployment
replicasets             rs           apps/v1      true         ReplicaSet
services                svc          v1           true         Service
namespaces              ns           v1           false        Namespace
persistentvolumes       pv           v1           false        PersistentVolume
persistentvolumeclaims  pvc          v1           true         PersistentVolumeClaim
```

> Use this when you forget a resource short name, are unsure of the API version, or need the exact `kind` name for a definition file.

---

### kubectl explain — Resource and Field Documentation

```bash
# List all top-level fields for a resource
kubectl explain pod
kubectl explain deployment
kubectl explain service
```

```
KIND:     Pod
VERSION:  v1

FIELDS:
   apiVersion   <string>    APIVersion defines the versioned schema...
   kind         <string>    Kind is a string value representing the REST resource...
   metadata     <Object>    Standard object's metadata...
   spec         <Object>    Specification of the desired behavior of the pod...
   status       <Object>    Most recently observed status of the pod...
```

```bash
# Drill into a specific field
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl explain deployment.spec.strategy
```

```bash
# List ALL fields recursively — full YAML structure in one view
kubectl explain pod --recursive
kubectl explain deployment --recursive
```

> `--recursive` outputs the complete field hierarchy exactly as it would appear in a YAML file — the fastest way to construct a definition file from scratch during the exam.

---

### Exam Usage Pattern

```bash
# Step 1 — find the resource and its API version
kubectl api-resources | grep ingress

# Step 2 — see all top-level fields
kubectl explain ingress

# Step 3 — drill into spec
kubectl explain ingress.spec

# Step 4 — get full field list for YAML construction
kubectl explain ingress --recursive
```

---

## kubectl apply — How It Works Internally

### The Three-Way Merge

`kubectl apply` does not simply overwrite the live object. It uses a **three-way merge** strategy, comparing three sources to determine exactly what changes to make:

| Source | Location | Contains |
|---|---|---|
| **Local file** | Your filesystem (`nginx.yaml`) | The desired state you are declaring |
| **Live object configuration** | Kubernetes memory (etcd) | The current state of the object including `status` fields added by Kubernetes |
| **Last applied configuration** | Stored as an annotation on the live object | A JSON snapshot of the local file from the last time `kubectl apply` was run |

```
Local file  ──────────────────────────────────────────────────┐
                                                               ▼
Last applied configuration  ──────────────────►  Three-way merge  ──►  Updated live object
                                                               ▲
Live object configuration  ───────────────────────────────────┘
```

---

### How the Three-Way Merge Works — Field-by-Field Logic

| Scenario | Action taken |
|---|---|
| Field exists in local file, different from live object | Update live object with value from local file |
| Field exists in local file, same as live object | No change |
| Field added to local file, not in live object | Add field to live object |
| Field deleted from local file, was present in last applied configuration | **Remove field from live object** |
| Field not in local file, not in last applied configuration, present in live object | Leave it — was added by Kubernetes or another process |

> The **last applied configuration** is what enables accurate deletion detection. Without it, `kubectl apply` cannot distinguish between "this field was intentionally removed from the local file" and "this field was added externally and should be left alone."

---

### Where the Last Applied Configuration is Stored

The last applied configuration is stored as an **annotation on the live object itself** — in JSON format:

```bash
kubectl get pod myapp-pod -o yaml
```

```yaml
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Pod","metadata":{"labels":{"app":"myapp",
      "type":"front-end-service"},"name":"myapp-pod"},"spec":{"containers":
      [{"image":"nginx:1.19","name":"nginx-container"}]}}
```

> This annotation is **only created when you use `kubectl apply`**. Objects created with `kubectl create` or `kubectl replace` do not have this annotation — which is why mixing imperative and declarative approaches on the same object causes unpredictable merge behaviour.

---

### Concrete Example — Image Update

**The starting point:**
You have a pod running `nginx:1.18`. You created it using `kubectl apply -f nginx.yaml`. At this point Kubernetes has stored three things:

```
Local file (your machine)     →  image: nginx:1.18
Last applied annotation       →  image: nginx:1.18   (JSON copy of your local file)
Live object (etcd)            →  image: nginx:1.18
```

All three match — no action needed.

---

**Now you update your local file to `nginx:1.19` and run `kubectl apply -f nginx.yaml`:**

Kubernetes runs a three-way comparison:

```
Local file          →  image: nginx:1.19   ← you changed this
Last applied        →  image: nginx:1.18
Live object         →  image: nginx:1.18
```

Kubernetes asks: *"Is the local file different from the live object?"*
Answer: **Yes** — `1.19` vs `1.18`

Action: **Update the live object to `nginx:1.19`**. Also update the last applied annotation to `nginx:1.19`.

**Result:**
```
Local file          →  image: nginx:1.19
Last applied        →  image: nginx:1.19   ← updated
Live object         →  image: nginx:1.19   ← updated
```

All three match again. Clean state.

---

### Deletion Example

**The starting point:**
Your pod has two labels — `app: myapp` and `type: front-end-service`. You created it with `kubectl apply`.

```
Local file          →  labels: { app: myapp, type: front-end-service }
Last applied        →  labels: { app: myapp, type: front-end-service }
Live object         →  labels: { app: myapp, type: front-end-service }
```

---

**Now you remove `type: front-end-service` from your local file and run `kubectl apply`:**

```
Local file          →  labels: { app: myapp }                          ← type removed
Last applied        →  labels: { app: myapp, type: front-end-service } ← type was here before
Live object         →  labels: { app: myapp, type: front-end-service } ← type still here
```

Kubernetes asks two questions:

**Question 1:** *"Is `type` in the local file?"*
Answer: **No**

**Question 2:** *"Was `type` in the last applied configuration?"*
Answer: **Yes**

Conclusion: **The field was intentionally removed** — it was there last time, and now it's gone from the local file. Remove it from the live object.

**Result:**
```
Local file          →  labels: { app: myapp }
Last applied        →  labels: { app: myapp }   ← updated
Live object         →  labels: { app: myapp }   ← type label removed
```

---

### Why Question 2 Matters — The Critical Distinction

Consider this scenario — Kubernetes itself adds a label to your pod automatically (e.g., a controller adds `controller-uid: abc123`). This label is:

```
Local file          →  NOT present (you never added it)
Last applied        →  NOT present (it wasn't there when you last applied)
Live object         →  PRESENT (Kubernetes added it)
```

Now Kubernetes asks the same two questions:

**Question 1:** *"Is it in the local file?"* → **No**
**Question 2:** *"Was it in the last applied configuration?"* → **No**

Conclusion: **This field was added externally — not intentionally removed by you. Leave it alone.**

This is exactly why the last applied configuration exists. Without it, `kubectl apply` cannot tell the difference between *"you deliberately removed this field"* and *"this field was added by something else and you never knew about it."* The last applied annotation is the memory of what **you** last declared — giving `kubectl apply` the context to make the right decision every time.

---

### kubectl apply — Key Behaviours

```bash
# Apply a single file — creates if not exists, updates if exists
kubectl apply -f nginx.yaml

# Apply all manifests in a directory
kubectl apply -f ./manifests/

# Apply recursively across subdirectories
kubectl apply -f ./manifests/ -R
```

---

### Exam Gotchas

- `kubectl apply` stores the last applied configuration as a JSON annotation on the live object — `kubectl create` and `kubectl replace` do **not** do this.
- **Never mix imperative and declarative approaches on the same object.** If an object was created with `kubectl create`, subsequent `kubectl apply` calls will be missing the last applied annotation — deletion detection will not work correctly.
- `kubectl explain --recursive` is the fastest way to get a complete field reference for any resource during the exam — faster than navigating the documentation website.
- `kubectl api-resources` is the authoritative source for short names and API versions — use it instead of guessing.
- The last applied configuration annotation is in **JSON** format — not YAML. This is by design; it is stored as a compact string value inside a YAML annotation field.
- If you see unexpected fields persisting on an object after `kubectl apply`, check whether they were externally applied without going through `kubectl apply` — they will not appear in the last applied annotation and will therefore be left alone by the merge.

---

### In Managed Clusters (AKS / EKS / GKE)

> `kubectl apply` three-way merge behaviour is identical on managed clusters. In production AKS environments using GitOps (ArgoCD, Flux), the same three-way merge logic is used — tools like ArgoCD compare the Git-stored manifest (local file equivalent), the last applied configuration, and the live cluster state to detect and reconcile drift automatically.

---

