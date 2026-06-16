## 01 — Rolling Updates and Rollbacks

### What it is

A **rollout** is the process by which Kubernetes applies a change to a Deployment's Pod template. Each rollout creates a new **ReplicaSet** and is recorded as a numbered **revision**. This revision history enables controlled rollbacks to any prior state.

---

### Rollout Revisions

Every change that modifies the Pod template (`.spec.template`) triggers a new rollout and increments the revision counter.

```bash
# Check live rollout progress
kubectl rollout status deployment/<deployment-name>

# View revision history
kubectl rollout history deployment/<deployment-name>

# View a specific revision's details
kubectl rollout history deployment/<deployment-name> --revision=2
```

> **Note:** Changes to replica count alone do NOT trigger a new rollout revision — only Pod template changes do.

---

### Deployment Strategies

| Strategy | Behaviour | Downtime | Default? |
|---|---|---|---|
| `Recreate` | Terminates all old Pods before creating new ones | Yes — gap between old down and new up | No |
| `RollingUpdate` | Replaces Pods incrementally — old ones come down one-by-one as new ones come up | None | **Yes** |

Specified in the Deployment manifest:

```yaml
spec:
  strategy:
    type: RollingUpdate          # or Recreate
    rollingUpdate:
      maxUnavailable: 1          # max Pods that can be down during update (int or %)
      maxSurge: 1                # max extra Pods above desired count during update (int or %)
```

If `.spec.strategy` is omitted entirely, Kubernetes defaults to `RollingUpdate` with `maxUnavailable: 25%` and `maxSurge: 25%`.

---

### How Rolling Updates Work Under the Hood

1. Kubernetes creates a **new ReplicaSet** for the updated Pod template.
2. It scales the new RS up and the old RS down simultaneously, one Pod at a time (subject to `maxUnavailable` / `maxSurge`).
3. Once complete: old RS has 0 Pods, new RS has the full replica count.
4. The old RS is **retained** (with 0 replicas) — this is what enables rollback.

```bash
# Observe both ReplicaSets during/after a rollout
kubectl get rs
# NAME                        DESIRED   CURRENT   READY
# myapp-deployment-7d9f6      5         5         5    ← new
# myapp-deployment-6b4c8      0         0         0    ← old (retained for rollback)
```

---

### Triggering an Update

**Method 1 — Edit the manifest and apply (preferred)**

```bash
# Edit deployment YAML (e.g., bump image tag)
kubectl apply -f deployment.yaml
```

**Method 2 — Imperative set image**

```bash
kubectl set image deployment/<deployment-name> <container-name>=<new-image>:<tag>
```

> ⚠️ `set image` updates the live object in the cluster but does NOT modify your local YAML file. The local file and cluster state diverge — future `kubectl apply` from the old file will revert the change. Avoid in production; use it in exams when speed matters.

---

### Rollback

```bash
# Roll back to the previous revision
kubectl rollout undo deployment/<deployment-name>

# Roll back to a specific revision
kubectl rollout undo deployment/<deployment-name> --to-revision=1
```

What happens internally: the old retained ReplicaSet is scaled back up, and the current ReplicaSet is scaled down to 0. The revision counter advances (the rolled-back-to revision becomes the newest revision, not a re-use of the old number).

```bash
# Verify after rollback
kubectl get rs
# myapp-deployment-6b4c8      5         5         5    ← old RS now active again
# myapp-deployment-7d9f6      0         0         0    ← new RS now idle
```

---

### Command Summary

| Goal | Command |
|---|---|
| Create deployment | `kubectl create -f deployment.yaml` |
| List deployments | `kubectl get deployments` |
| Update (declarative) | `kubectl apply -f deployment.yaml` |
| Update image (imperative) | `kubectl set image deployment/<name> <container>=<image>` |
| Check rollout status | `kubectl rollout status deployment/<name>` |
| View rollout history | `kubectl rollout history deployment/<name>` |
| Rollback (previous) | `kubectl rollout undo deployment/<name>` |
| Rollback (specific revision) | `kubectl rollout undo deployment/<name> --to-revision=<n>` |
| Describe deployment events | `kubectl describe deployment <name>` |

---

### Exam Gotchas

- **Default strategy is RollingUpdate, not Recreate.** If a question asks "which strategy avoids downtime," the answer is RollingUpdate — and you don't need to specify it explicitly since it's the default.
- **`set image` does not update your YAML.** If you then `kubectl apply -f` the same file, it will revert to the image in the file. In the exam, use `set image` when working imperatively and you don't need file-level consistency.
- **Old ReplicaSets are kept after rollout.** `kubectl get rs` will show a zero-replica RS — this is expected and intentional, not a problem.
- **Rollback re-uses the old RS**, it doesn't create a brand new one. The revision counter still increments.
- **Revision history is limited** by `.spec.revisionHistoryLimit` (default: 10). If you need to roll back beyond 10 revisions, the RS may already be garbage-collected.
- **`kubectl rollout status`** blocks and streams progress — useful in scripts to gate subsequent steps.
- **`describe deployment`** events section will explicitly show which strategy was used and scale transitions — this is what exam questions about "how did the update happen" reference.

---

### In Managed Clusters (AKS)

Behaviour is identical to self-managed clusters. The `RollingUpdate` defaults (`maxUnavailable: 25%`, `maxSurge: 25%`) apply.

In AKS production contexts, `kubectl rollout undo` is rarely used directly — GitOps tools like ArgoCD handle rollback by reverting the Git commit, which re-triggers the pipeline. The Kubernetes rollback mechanism is the underlying mechanism ArgoCD calls, but you interact with Git, not kubectl.

---

### Real-World Usage

- **Blue/green deployments** are not a native Kubernetes strategy — they are implemented by maintaining two Deployments and switching a Service selector. `RollingUpdate` is Kubernetes' native zero-downtime mechanism.
- **Canary releases** are similarly implemented via separate Deployments or traffic-split tools (Argo Rollouts, Flagger, Istio), not the built-in strategy field.
- **`revisionHistoryLimit: 3`** is commonly set in production to avoid accumulating stale ReplicaSets, especially in clusters with many frequently-deployed applications.
- **CI/CD pipelines** typically use `kubectl set image` or `helm upgrade` to trigger rollouts — they then call `kubectl rollout status` to block the pipeline until the rollout completes before marking the deployment successful.

---

