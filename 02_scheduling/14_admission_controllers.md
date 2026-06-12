## 14 — Admission Controllers

### What it is
An Admission Controller is a piece of code that **intercepts requests to the kube-apiserver** after authentication and authorization have passed, but **before the object is persisted to etcd**. It can validate a request, modify it, or reject it entirely — enforcing policies that go beyond what RBAC can express.

---

### Where Admission Controllers Fit in the Request Pipeline

```
kubectl / API request
        │
        ▼
┌─────────────────────┐
│   Authentication    │  Who are you? (certificates, tokens)
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│   Authorization     │  Are you allowed to do this? (RBAC)
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ Admission Controllers│ ← Should this request be allowed, modified, or rejected?
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│   Persisted to etcd │  Object is created
└─────────────────────┘
```

---

### Why RBAC Is Not Enough

RBAC answers: *"Is this user allowed to perform this operation on this resource type?"*

It cannot answer:
- Is this container image from an approved internal registry?
- Is the image using the `latest` tag (which should be blocked)?
- Is this container running as root (which should be denied)?
- Does this Pod spec include the required labels?
- Does this PVC have a storage class assigned?

These are **content-level policies** — they require inspecting the request body, not just the verb and resource type. Admission controllers are the mechanism for enforcing them.

> **Azure analogy:** Admission controllers are Kubernetes' equivalent of **Azure Policy**. Azure Policy enforces organisational rules at the ARM layer — before a resource is deployed — by auditing, denying, or modifying requests. Admission controllers do the same at the Kubernetes API layer — before an object is persisted to etcd.
>
> | Azure | Kubernetes |
> |---|---|
> | Azure Policy | Admission Controller |
> | Policy Definition | Admission Controller rule / Webhook logic |
> | Deny effect | Validating controller rejecting a request |
> | Modify / Append effect | Mutating controller modifying a request |
> | Azure Policy for AKS (Gatekeeper) | Webhook Admission Controller running inside the cluster |
> | ARM layer (before resource creation) | After AuthN/AuthZ, before etcd persistence |
>
> The key distinction: Azure Policy is a managed external service sitting outside your resources. Kubernetes admission controllers run inside the control plane pipeline — either built into the apiserver itself, or as webhook Deployments inside the cluster.

---

### What Admission Controllers Can Do

| Capability | Description |
|---|---|
| **Validate** | Inspect the request and reject it if it violates a policy |
| **Mutate** | Modify the request before it is persisted — inject defaults, add labels, set fields |
| **Both** | Some controllers validate and mutate in the same step |

---

### Built-in Admission Controllers

| Controller | What it does | Enabled by default |
|---|---|---|
| `AlwaysPullImages` | Forces every Pod to always pull its image — prevents use of cached images from other tenants | ❌ |
| `DefaultStorageClass` | Automatically adds the default storage class to PVCs that don't specify one | ✅ |
| `EventRateLimit` | Limits the rate of event requests to the API server to prevent flooding | ❌ |
| `NamespaceExists` | Rejects requests to namespaces that do not exist | ✅ (deprecated) |
| `NamespaceAutoProvision` | Automatically creates a namespace if it does not exist | ❌ (deprecated) |
| `NamespaceLifecycle` | Rejects requests to non-existent namespaces; protects `default`, `kube-system`, `kube-public` from deletion. Replaces both deprecated controllers above | ✅ |
| `LimitRanger` | Enforces `LimitRange` constraints on Pods and containers | ✅ |
| `ResourceQuota` | Enforces `ResourceQuota` limits per namespace | ✅ |
| `ServiceAccount` | Automatically assigns a ServiceAccount to Pods that don't specify one | ✅ |

---

### Viewing Enabled Admission Controllers

```bash
# On a kubeadm cluster — exec into the apiserver Pod first
kubectl exec -n kube-system kube-apiserver-controlplane -- \
  kube-apiserver -h | grep enable-admission-plugins

# Or inspect the running apiserver flags directly
kubectl exec -n kube-system kube-apiserver-controlplane -- \
  ps aux | grep admission
```

---

### Enabling and Disabling Admission Controllers

**In a kubeadm cluster — edit the apiserver static Pod manifest:**

```bash
vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

```yaml
spec:
  containers:
    - command:
        - kube-apiserver
        - --enable-admission-plugins=NodeRestriction,NamespaceAutoProvision
        - --disable-admission-plugins=DefaultStorageClass
```

> Editing this file causes the kubelet to automatically restart the kube-apiserver static Pod — no manual restart needed. Wait 30–60 seconds for the apiserver to come back up.

**In a non-kubeadm cluster — edit the kube-apiserver service file:**

```bash
vi /etc/systemd/system/kube-apiserver.service
# Add flags to ExecStart
systemctl daemon-reload && systemctl restart kube-apiserver
```

---

### NamespaceExists vs NamespaceAutoProvision vs NamespaceLifecycle

This is a common source of confusion:

| Controller | Behaviour | Status |
|---|---|---|
| `NamespaceExists` | Rejects requests to non-existent namespaces | Deprecated |
| `NamespaceAutoProvision` | Creates the namespace automatically if it doesn't exist | Deprecated |
| `NamespaceLifecycle` | Rejects non-existent namespace requests **and** protects default namespaces from deletion | ✅ Active replacement |

> Both deprecated controllers are now replaced by `NamespaceLifecycle`. Do not enable the deprecated ones on new clusters.

---

### Exam Gotchas

- **Admission controllers run after RBAC** — a request that passes authentication and authorization can still be rejected by an admission controller. If a Pod creation is rejected with a policy-related error, check admission controllers before assuming it is an RBAC issue.
- **Editing `kube-apiserver.yaml` restarts the apiserver** — after editing the static Pod manifest, wait for the apiserver to come back up before running further `kubectl` commands. It can take up to 60 seconds.
- **`--enable-admission-plugins` is additive** — it adds to the default set, not replaces it. `--disable-admission-plugins` explicitly removes from the default set.
- **`NamespaceAutoProvision` is disabled by default** — if an exam question asks you to allow Pod creation in a non-existent namespace automatically, you need to explicitly enable it (or check if `NamespaceLifecycle` is already handling this).
- **Two types of admission controllers exist** — validating and mutating. These are covered in detail in the next section (`15_validating_mutating_admission_controllers.md`).

---

### Real-World Use Cases

#### Use Case 1 — Block Images from Public Docker Hub

**Problem:** Developers are deploying Pods using public Docker Hub images (`nginx:latest`, `python:3.11`) which bypasses security scanning, introduces supply chain risk, and violates organisational policy. RBAC cannot prevent this — it only controls whether a user can create a Pod, not what image they use inside it.

**Solution:** Enable the `AlwaysPullImages` admission controller as a short-term measure, and implement an `ImagePolicyWebhook` as a long-term solution that rejects any image not from the internal registry (`registry.company.internal`).

**What happens without the controller:**
```bash
# Developer runs this — succeeds even though nginx is from public Docker Hub
kubectl run web --image=nginx:latest
# Pod created ✅ — no policy enforcement
```

**What happens with the controller enforcing internal registry:**
```bash
kubectl run web --image=nginx:latest
# Error: admission webhook denied the request:
# image "nginx:latest" is not from an approved registry.
# Use registry.company.internal/nginx:latest instead.

kubectl run web --image=registry.company.internal/nginx:1.25
# Pod created ✅
```

**Why admission controllers, not RBAC:**
RBAC could deny a user the `create pods` verb entirely — but that blocks all Pod creation, not just ones with unapproved images. Admission controllers inspect the request body and make policy decisions at the content level.

---

#### Use Case 2 — Enforce Required Labels on All Pods

**Problem:** A platform team requires every Pod to carry `app` and `env` labels for cost attribution, monitoring dashboards, and alert routing. Developers frequently forget to add them, causing Pods to disappear from dashboards and alerts to misfire.

**Solution:** A validating admission webhook rejects any Pod creation or update request where `app` or `env` labels are missing from `metadata.labels`.

**What happens without enforcement:**
```bash
# Pod created without required labels — no error
kubectl run payments --image=payments-api:v1
kubectl get pod payments --show-labels
# NAME       LABELS
# payments   <none>    ← invisible to dashboards and alert rules
```

**What happens with the admission controller:**
```bash
kubectl run payments --image=payments-api:v1
# Error: admission webhook denied the request:
# Pod must include labels: "app" and "env".
# Add them to metadata.labels before resubmitting.

# Correct request
kubectl run payments --image=payments-api:v1 \
  -l app=payments,env=prod
# Pod created ✅
```

**Why this matters operationally:**
Without enforcement, label compliance depends entirely on developer discipline. With an admission controller, it is a hard cluster-level guarantee — no Pod enters the cluster without the required labels, regardless of who creates it or how.

---



- In AKS, the kube-apiserver admission controller configuration is fully managed — you cannot modify `--enable-admission-plugins` or `--disable-admission-plugins` directly.
- Custom admission logic in AKS is implemented via **Webhook Admission Controllers** (covered in the next section) — these run as Deployments inside the cluster and are fully supported.
- AKS uses **Azure Policy for Kubernetes** (built on Gatekeeper/OPA) as its managed admission controller solution — it enforces organisational policies (image registries, resource limits, label requirements) across AKS clusters without modifying the apiserver directly.

---

