## 19 — Imperative vs Declarative

### What it is

Two fundamentally different approaches to managing infrastructure and Kubernetes objects. The distinction determines **how you communicate intent to the system** — by specifying exact steps, or by specifying the desired end state.

---

### Core Distinction

| | Imperative | Declarative |
|---|---|---|
| You specify | **What to do AND how to do it** — step-by-step instructions | **What the end state should be** — the system determines how to get there |
| System role | Executes exactly what you instruct | Compares current state to desired state and reconciles the difference |
| Idempotency | ❌ Not inherently idempotent — re-running the same commands may fail or produce unintended results | ✅ Idempotent — applying the same configuration repeatedly always converges to the same state |
| State tracking | ❌ No record of how objects were created — exists only in session history | ✅ Configuration files serve as the source of truth — storable in Git, reviewable, auditable |
| Examples in IaC | Shell scripts, step-by-step runbooks | Terraform, Ansible, Puppet, Chef |
| Examples in Kubernetes | `kubectl run`, `kubectl create`, `kubectl edit`, `kubectl scale` | `kubectl apply` |

---

### Imperative Approach in Kubernetes

#### 1 — Imperative Commands (no YAML files)

Direct commands that create or modify objects immediately:

```bash
# Create objects
kubectl run nginx --image=nginx                          # create a pod
kubectl create deployment nginx --image=nginx            # create a deployment
kubectl expose deployment nginx --port=80                # create a service

# Update objects
kubectl edit deployment nginx                            # edit live object directly
kubectl scale deployment nginx --replicas=5              # scale a deployment
kubectl set image deployment nginx nginx=nginx:1.18      # update container image
```

**Strengths:** Fast — no YAML required. Ideal for simple, one-off tasks in the exam.

**Weaknesses:**
- No persistent record — changes exist only in session history
- Not suitable for complex objects (multi-container pods, init containers, env vars)
- Re-running `kubectl create` on an existing object **fails with an error**
- Re-running `kubectl replace` on a non-existent object **fails with an error**

#### 2 — Imperative with Configuration Files

Uses YAML files but with imperative commands:

```bash
kubectl create -f nginx.yaml          # fails if object already exists
kubectl replace -f nginx.yaml         # fails if object does not exist
kubectl replace --force -f nginx.yaml # deletes and recreates the object
kubectl delete -f nginx.yaml          # deletes the object
```

**Strengths:** Configuration is documented in files — can be stored in Git.

**Weaknesses:**
- You must still manage the object lifecycle manually — know whether to `create` or `replace`
- `kubectl edit` changes the live object but **not your local YAML file** — the local file becomes stale and will overwrite the edit on the next `replace`

> **`kubectl edit` warning:** Changes made via `kubectl edit` are applied to the live object in Kubernetes memory only. Your local definition file retains the old configuration. If a teammate later runs `kubectl replace -f` with the local file, your edit is silently overwritten.

---

### Declarative Approach in Kubernetes

Uses `kubectl apply` exclusively — for creating, updating, and managing all objects:

```bash
# Create or update a single object
kubectl apply -f nginx.yaml

# Create or update all objects in a directory at once
kubectl apply -f ./manifests/

# Apply recursively across subdirectories
kubectl apply -f ./manifests/ -R
```

**How `kubectl apply` behaves:**
- Object **does not exist** → creates it
- Object **exists** → computes the diff and applies only the changed fields
- **Never errors** on "object already exists" or "object does not exist"
- Always converges to the state defined in the file

**Strengths:**
- Single command for all lifecycle operations
- Idempotent — safe to re-run at any time
- Configuration files are the source of truth — store in Git, review via PRs, audit changes
- Supports directory-level application — deploy entire environments in one command

---

### kubectl edit — What Actually Happens

When you run `kubectl edit deployment nginx`, Kubernetes opens a YAML file from its **in-memory representation** of the object — not your local file. This includes additional fields like `status` that are not in your original manifest.

```
kubectl edit → opens live object from Kubernetes memory (includes status fields)
     │
     ▼
You make changes and save
     │
     ▼
Changes applied to the live object in etcd
     │
     ▼
Your local .yaml file is UNCHANGED — now out of sync with the live object
```

> Use `kubectl edit` only for quick temporary changes where the local file is not being used for ongoing management. For anything tracked in Git, always edit the local file and apply it.

---

### Exam Strategy — When to Use Each

| Scenario | Best approach |
|---|---|
| Create a simple pod or deployment quickly | Imperative command — `kubectl run` or `kubectl create deployment` |
| Edit a single field on a live object | `kubectl edit` — fast, no file needed |
| Complex object — multiple containers, env vars, init containers | Generate YAML with `--dry-run=client -o yaml`, edit the file, then `kubectl apply` |
| Any change you need to re-apply or verify later | `kubectl apply -f` — idempotent, safe to re-run |
| Made a mistake in a running object | Edit the local file → `kubectl apply -f` to reconcile |

---

### Exam Tips — Imperative Commands Reference

**Create objects:**

```bash
kubectl apply -f nginx.yaml
kubectl run nginx --image=nginx
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80
```

**Update objects:**

```bash
kubectl apply -f nginx.yaml
kubectl edit deployment nginx
kubectl scale deployment nginx --replicas=5
kubectl set image deployment nginx nginx=nginx:1.18
```

---

### Certification Tips — Imperative Commands Quick Reference

Two flags that work together to save significant time in the exam:

| Flag | Behaviour |
|---|---|
| `--dry-run=client` | Validates the command and tells you if the resource **can** be created — without actually creating it |
| `-o yaml` | Outputs the full resource definition in YAML format to the terminal |

Combined: `--dry-run=client -o yaml` generates a complete, correctly structured YAML definition instantly — redirect it to a file, edit only what's needed, then apply.

---

#### Pod

```bash
# Create a pod directly
kubectl run nginx --image=nginx

# Generate pod YAML without creating
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
```

---

#### Deployment

```bash
# Create a deployment directly
kubectl create deployment nginx --image=nginx

# Generate deployment YAML without creating
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deployment.yaml

# Create with specific replica count
kubectl create deployment nginx --image=nginx --replicas=4

# Scale an existing deployment
kubectl scale deployment nginx --replicas=4

# Generate YAML, save to file, then edit replicas or any field before applying
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > nginx-deployment.yaml
# edit nginx-deployment.yaml → kubectl apply -f nginx-deployment.yaml
```

---

#### Service

Two commands exist for creating services imperatively — each with a specific limitation:

**ClusterIP:**

```bash
# Recommended — automatically uses pod labels as selectors
kubectl expose pod redis --port=6379 --name=redis-service --dry-run=client -o yaml

# Alternative — does NOT use pod labels, assumes selector app=redis
# Cannot pass custom selectors — generate file and edit selectors manually
kubectl create service clusterip redis --tcp=6379:6379 --dry-run=client -o yaml
```

**NodePort:**

```bash
# Recommended — automatically uses pod labels as selectors
# Limitation: cannot specify nodePort — generate file and add nodePort manually
kubectl expose pod nginx --type=NodePort --port=80 --name=nginx-service --dry-run=client -o yaml

# Alternative — can specify nodePort but does NOT use pod labels as selectors
kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run=client -o yaml
```

> **Rule of thumb:** Use `kubectl expose` for correct selector inheritance. If you need to specify a `nodePort`, use `kubectl expose` to generate the YAML, then manually add the `nodePort` field before applying.

**Reference:**
- https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands
- https://kubernetes.io/docs/reference/kubectl/conventions/

---

### Exam Gotchas

- `kubectl create` on an existing object → **error**. `kubectl apply` on an existing object → **no error**, applies diff only.
- `kubectl replace` on a non-existent object → **error**. Always verify existence before using `replace`.
- `kubectl edit` does **not** update your local file — your local file becomes stale immediately after an edit.
- `kubectl apply` can target a **directory** — use this to apply all manifests in one command instead of running apply per file.
- In the exam, use imperative commands for speed on simple tasks — switch to `--dry-run=client -o yaml` + `kubectl apply` for anything complex.
- `kubectl replace --force -f` deletes and recreates the object — use when a field is immutable and cannot be updated in place (e.g., changing a pod's container image directly).

---

### In Managed Clusters (AKS / EKS / GKE)

> The imperative vs declarative distinction applies identically on managed clusters. In production AKS environments, the declarative approach via `kubectl apply` (or GitOps tools like ArgoCD / Flux) is the standard — imperative commands are reserved for debugging and emergency interventions only.

---

