## 14 — Deployments

### What it is

A **Deployment** is a higher-level Kubernetes object that manages ReplicaSets and provides declarative control over application rollouts. It sits above ReplicaSets in the hierarchy — a Deployment creates and manages a ReplicaSet, which in turn creates and manages Pods.

```
Deployment
    └── ReplicaSet
            └── Pod
            └── Pod
            └── Pod
```

---

### Why Deployments Exist — Production Requirements

| Requirement | Deployment capability |
|---|---|
| Run multiple instances of an application | Manages a ReplicaSet with a desired replica count |
| Upgrade instances without downtime | **Rolling updates** — replaces pods one at a time |
| Recover from a bad release | **Rollback** — reverts to a previous known-good state |
| Apply multiple changes at once | **Pause and resume** — batch changes before rolling out |

---

### Deployment vs ReplicaSet

| | ReplicaSet | Deployment |
|---|---|---|
| Purpose | Maintains desired pod count | Manages ReplicaSets + adds rollout control |
| Rolling updates | ❌ Not supported | ✅ Built-in |
| Rollback | ❌ Not supported | ✅ Built-in |
| Pause / resume changes | ❌ Not supported | ✅ Built-in |
| Use in production | ❌ Rarely used directly | ✅ Standard workload object |

> In practice, you always create Deployments — never bare ReplicaSets. The Deployment manages the ReplicaSet automatically.

---

### Deployment Definition File

The structure is identical to a ReplicaSet definition — the only difference is `kind: Deployment`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    app: myapp
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        tier: frontend
    spec:
      containers:
        - name: nginx-container
          image: nginx
```

---

### Object Hierarchy Created by a Deployment

When a Deployment is created, it automatically creates a ReplicaSet, which in turn creates Pods. The naming reflects this hierarchy:

```
Deployment:   myapp-deployment
ReplicaSet:   myapp-deployment-<rs-hash>          e.g. myapp-deployment-7d9f8b6c4
Pod:          myapp-deployment-<rs-hash>-<pod-id>  e.g. myapp-deployment-7d9f8b6c4-x7k2p
```

---

### kubectl Commands

```bash
# Create a Deployment
kubectl create -f deployment-definition.yaml

# Generate Deployment YAML without creating — fastest method in exam
kubectl create deployment myapp --image=nginx --replicas=3 --dry-run=client -o yaml > deployment.yaml

# List Deployments
kubectl get deployments

# List all objects created by the Deployment at once
kubectl get all

# Detailed information
kubectl describe deployment myapp-deployment

# View the ReplicaSet created by the Deployment
kubectl get replicaset

# View Pods created by the Deployment
kubectl get pods

# Delete a Deployment (also deletes its ReplicaSet and Pods)
kubectl delete deployment myapp-deployment
```

**Example output of `kubectl get all`:**
```
NAME                                       READY   STATUS    RESTARTS
pod/myapp-deployment-7d9f8b6c4-x7k2p      1/1     Running   0
pod/myapp-deployment-7d9f8b6c4-p9q3r      1/1     Running   0
pod/myapp-deployment-7d9f8b6c4-m2n5s      1/1     Running   0

NAME                                  DESIRED   CURRENT   READY
replicaset/myapp-deployment-7d9f8b6c4    3         3         3

NAME                            READY   UP-TO-DATE   AVAILABLE
deployment/myapp-deployment     3/3     3            3
```

---

### Exam Gotchas

- `apiVersion` for Deployment is `apps/v1` — same as ReplicaSet. Using `v1` produces a validation error.
- A Deployment **automatically creates a ReplicaSet** — you will see it in `kubectl get replicaset` even though you never defined one.
- Deleting a Deployment **also deletes** its ReplicaSet and all Pods.
- `kubectl get all` is the fastest way to see the full object hierarchy in one command — use it constantly during the exam.
- Rolling updates, rollbacks, and pause/resume are covered in the Application Lifecycle Management section of this course.
- Use `kubectl create deployment` for imperative creation — not `kubectl run` which creates a bare Pod.

---

### In Managed Clusters (AKS / EKS / GKE)

> Deployment behaviour is identical across managed and self-managed clusters. In AKS, Deployments are the standard way to run all stateless workloads — no difference in how they are defined or managed.

---

