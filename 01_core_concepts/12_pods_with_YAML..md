## 12 — Pods with YAML

### What it is

Kubernetes uses YAML definition files to create and manage all objects — pods, deployments, services, and more. Every Kubernetes YAML definition file follows the same structure and always contains four mandatory top-level fields.

---

### The Four Mandatory Top-Level Fields

```yaml
apiVersion:
kind:
metadata:

spec:
```

Every Kubernetes definition file — regardless of the object type — must have exactly these four root-level fields. Missing any one of them will cause the `kubectl` command to fail.

| Field | Type | Description |
|---|---|---|
| `apiVersion` | String | Version of the Kubernetes API used to create this object |
| `kind` | String | Type of Kubernetes object to create |
| `metadata` | Dictionary | Data about the object — name, labels, namespace, annotations |
| `spec` | Dictionary | Desired state specification — varies per object type |

---

### apiVersion and kind — Common Combinations

| `kind` | `apiVersion` |
|---|---|
| `Pod` | `v1` |
| `Service` | `v1` |
| `ReplicaSet` | `apps/v1` |
| `Deployment` | `apps/v1` |
| `Namespace` | `v1` |
| `ServiceAccount` | `v1` |

> Always match `apiVersion` to `kind` — using the wrong API version will cause a validation error or the object will not be found.

---

### metadata — Structure and Rules

```yaml
metadata:
  name: myapp-pod
  labels:
    app: myapp
    tier: frontend
```

**Rules:**
- `name` and `labels` must be indented equally — they are siblings under `metadata`
- `labels` is a dictionary — it accepts any arbitrary key-value pairs you define
- Under `metadata` itself, you can only use fields Kubernetes recognises (`name`, `labels`, `namespace`, `annotations`) — you cannot add arbitrary custom fields directly under `metadata`
- Under `labels`, you can define any key-value pairs you need — there is no restriction

**Why labels matter:**
Labels are how you identify, filter, and select objects at scale. In a cluster running hundreds of pods across multiple tiers, labels like `app: frontend`, `app: backend`, or `app: database` let you target specific groups using selectors in Services, Deployments, and `kubectl` queries.

---

### spec — Pod-Specific Structure

```yaml
spec:
  containers:
    - name: nginx-container
      image: nginx
```

**Rules:**
- `containers` is a **list** — a pod can have multiple containers, each as a separate list item
- Each list item is denoted by a leading `-` (dash)
- Each item in the list is a dictionary with at minimum `name` and `image`
- `image` refers to the Docker image name — pulled from Docker Hub by default, or a private registry if configured

---

### Complete Pod Definition File

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    tier: frontend
spec:
  containers:
    - name: nginx-container
      image: nginx
```

---

### Creating and Inspecting a Pod from a YAML File

```bash
# Create the pod from the definition file
kubectl create -f pod-definition.yaml

# OR — create or update (preferred for declarative workflow)
kubectl apply -f pod-definition.yaml

# List all pods
kubectl get pods

# Detailed pod information — labels, node, containers, events
kubectl describe pod myapp-pod

# Output the pod definition in YAML format
kubectl get pod myapp-pod -o yaml

# Edit a running pod's definition directly
kubectl edit pod myapp-pod

# Delete the pod
kubectl delete pod myapp-pod
```

**Example output of `kubectl describe pod myapp-pod`:**
```
Name:         myapp-pod
Namespace:    default
Node:         node01/192.168.1.2
Labels:       app=myapp
              tier=frontend
Status:       Running
Containers:
  nginx-container:
    Image:    nginx
    State:    Running
Events:
  ...
```

---

### YAML Indentation — Common Mistakes

YAML is indentation-sensitive. These are the most frequent errors:

```yaml
# ✅ Correct — name and labels are siblings under metadata
metadata:
  name: myapp-pod
  labels:
    app: myapp

# ❌ Wrong — labels is a child of name, not a sibling
metadata:
  name: myapp-pod
    labels:       # over-indented — now a child of name
      app: myapp

# ❌ Wrong — name, labels, and metadata are all siblings (same level)
metadata:
name: myapp-pod   # same indentation as metadata — not a child
labels:
  app: myapp
```

---

### Exam Gotchas

- The four top-level fields — `apiVersion`, `kind`, `metadata`, `spec` — are **mandatory** in every definition file without exception.
- `containers` under `spec` is a **list** — the `-` before `name` is not optional. Omitting it causes a parse error.
- `kubectl create` fails if the object already exists. `kubectl apply` creates or updates — prefer `apply` for idempotent workflows.
- In the CKA exam, use `kubectl run --dry-run=client -o yaml` to generate a pod YAML template instantly rather than writing from scratch:

```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod-definition.yaml
```
- `--dry-run=client` tells `kubectl` to process the command entirely on the client side and generate the object definition without sending it to the API server — so nothing is actually created in the cluster.
- Combined with `-o yaml` Kubernetes outputs the complete, correctly structured YAML for you. You then open the file, make only the changes specific to your task — add a label, change a port, set an env variable — and apply it. That's seconds of work instead of minutes.
- `kubectl describe pod` is the primary command for diagnosing pod issues — always check the **Events** section at the bottom of the output first.

---

### In Managed Clusters (AKS / EKS / GKE)

> Pod YAML structure and `kubectl` commands are identical across all Kubernetes clusters — managed or self-managed. No difference at this level.

---

