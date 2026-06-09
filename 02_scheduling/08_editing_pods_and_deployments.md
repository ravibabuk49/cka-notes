## 08 — Editing Pods and Deployments

### What it is
A reference for what can and cannot be edited on a live Pod, and how to work around the immutability constraints. Deployments have no such restrictions since they manage Pod recreation automatically.

---

### Editing a Pod — Mutable Fields Only

Most Pod spec fields are **immutable** after creation. The only fields you can edit on a running Pod are:

| Field | Description |
|---|---|
| `spec.containers[*].image` | Container image |
| `spec.initContainers[*].image` | Init container image |
| `spec.activeDeadlineSeconds` | Maximum duration a Pod can run |
| `spec.tolerations` | Toleration entries (can be added, not removed) |

Attempting to edit any other field — environment variables, resource limits, service accounts, volumes, etc. — will be rejected by the API server.

---

### Workaround — How to Edit an Immutable Field

There are two approaches:

#### Option 1 — `kubectl edit` + delete + recreate from temp file

```bash
# Step 1 — open the Pod spec in the editor, make your changes, attempt to save
kubectl edit pod webapp
# API server rejects the save — but kubectl writes your changes to a temp file
# Note the temp file path shown in the error output e.g. /tmp/kubectl-edit-ccvrq.yaml

# Step 2 — delete the existing Pod
kubectl delete pod webapp

# Step 3 — recreate from the temp file with your changes
kubectl create -f /tmp/kubectl-edit-ccvrq.yaml
```

#### Option 2 — export + edit + delete + recreate from your own file

```bash
# Step 1 — export the live Pod spec to a local YAML file
kubectl get pod webapp -o yaml > my-new-pod.yaml

# Step 2 — edit the file
vi my-new-pod.yaml

# Step 3 — delete the existing Pod
kubectl delete pod webapp

# Step 4 — recreate from your edited file
kubectl create -f my-new-pod.yaml
```

> **Which option to use:** Option 1 is faster in the exam — fewer commands. Option 2 gives you a clean file you control, which is safer if you need to verify your changes before recreating.

---

### Editing a Deployment — No Restrictions

Every field in the Pod template (`spec.template`) of a Deployment is editable. When you save a change, the Deployment controller **automatically deletes the old Pods and recreates them** with the updated spec via a rolling update.

```bash
kubectl edit deployment my-deployment
# Save → Deployment triggers rolling Pod replacement automatically
```

This is why editing resource limits, environment variables, or service accounts is straightforward on a Deployment but requires delete-and-recreate on a standalone Pod.

---

### Pod vs Deployment Edit — Summary

| | Standalone Pod | Deployment |
|---|---|---|
| **Editable fields** | 4 specific fields only | Any field in `spec.template` |
| **Edit command** | `kubectl edit pod <name>` | `kubectl edit deployment <name>` |
| **How changes apply** | Manual delete + recreate required | Automatic rolling Pod replacement |
| **Risk of downtime** | Yes — Pod is deleted before recreate | No — rolling update handles it |

---

### Exam Gotchas

- **`kubectl edit pod` does not silently fail** — it writes your changes to a temp file even when rejected. Note the path — you will need it for Option 1.
- **Temp file path is random** — it looks like `/tmp/kubectl-edit-<random>.yaml`. Do not close the terminal before noting it or you will have to fall back to Option 2.
- **`kubectl apply` will also be rejected** for immutable fields — the restriction is enforced by the API server, not just `kubectl edit`.
- **Always prefer editing at the Deployment level** — if the Pod is part of a Deployment, editing the Pod directly is pointless. The Deployment will recreate the old Pod spec the moment the manually edited Pod is deleted. Edit the Deployment instead.
- **`kubectl replace --force -f my-new-pod.yaml`** — a one-command shortcut that deletes and recreates the Pod from a file in a single step. Useful in the exam to save time:
```bash
kubectl get pod webapp -o yaml > my-new-pod.yaml
# edit the file
kubectl replace --force -f my-new-pod.yaml
```

---

