## 18 — Namespaces

### What it is

A **Namespace** is a logical isolation boundary within a Kubernetes cluster. It partitions cluster resources into distinct scopes — each with its own set of objects, access policies, and resource quotas. Resources within the same namespace can reference each other by short name. Resources across namespaces require a fully qualified DNS name.

---

### Default Namespaces Created by Kubernetes

| Namespace | Purpose |
|---|---|
| `default` | The working namespace when no namespace is specified. All user-created resources land here unless explicitly targeted to another namespace. |
| `kube-system` | Reserved for Kubernetes internal components — DNS, networking, control plane pods. Isolated to prevent accidental modification by users. |
| `kube-public` | Resources that must be accessible to all users across the cluster, including unauthenticated users. Rarely used directly. |
| `kube-node-lease` | Holds node lease objects used by the control plane to track node heartbeats. |

---

### Why Namespaces Matter

| Use case | How namespaces help |
|---|---|
| Environment isolation | Separate `dev`, `staging`, and `prod` on the same cluster — preventing accidental cross-environment changes |
| Access control | Apply RBAC policies per namespace — teams only access their own namespace |
| Resource quotas | Assign CPU, memory, and object count limits per namespace — preventing one team from consuming cluster-wide resources |
| Multi-team clusters | Each team operates independently in their own namespace without visibility into others |

> For small or learning clusters, the `default` namespace is sufficient. Namespaces become essential at enterprise or production scale.

---

### DNS and Cross-Namespace Communication

Resources within the same namespace reference each other by **short name**:

```
# Pod in 'default' namespace reaching a service in the same namespace
curl http://db-service
```

To reach a resource in a **different namespace**, use the fully qualified DNS name:

```
curl http://db-service.dev.svc.cluster.local
```

**DNS name format breakdown:**

```
db-service  .  dev  .  svc  .  cluster.local
     │            │      │           │
 service name  namespace  service   default cluster
                          subdomain  domain
```

> This DNS entry is created automatically by CoreDNS when the service is created — no manual configuration required.

---

### Namespace Definition File

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

---

### Resource Quota Definition File

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "10"
    limits.memory: 16Gi
```

---

### Specifying Namespace in a Pod Definition

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  namespace: dev          # ensures pod is always created in dev namespace
  labels:
    app: myapp
spec:
  containers:
    - name: nginx
      image: nginx
```

> Defining `namespace` in the manifest ensures the resource is always created in the correct namespace regardless of the current `kubectl` context — preferred over relying on command-line flags.

---

### kubectl Commands

```bash
# List pods in the default namespace
kubectl get pods

# List pods in a specific namespace
kubectl get pods -n kube-system
kubectl get pods --namespace=dev

# List pods across all namespaces
kubectl get pods --all-namespaces
kubectl get pods -A                             # shorthand

# Create a resource in a specific namespace
kubectl create -f pod.yaml -n dev
kubectl apply -f pod.yaml -n dev

# Create a namespace imperatively
kubectl create namespace dev

# Switch current context to a different namespace permanently
kubectl config set-context --current --namespace=dev

# Verify current namespace
kubectl config view --minify | grep namespace

# Create a resource quota
kubectl create -f resource-quota.yaml
kubectl get resourcequota -n dev
```

---

### Exam Gotchas

- `kubectl get pods` only lists pods in the **current namespace** — always use `-A` or `--all-namespaces` when looking for a resource you cannot locate.
- `kubectl config set-context --current --namespace=dev` switches the default namespace for all subsequent commands in the current context — easy to forget this is set and wonder why resources are missing.
- Cross-namespace DNS format is `<service>.<namespace>.svc.cluster.local` — the exam frequently tests cross-namespace service access.
- `kube-system` namespace contains control plane pods in kubeadm clusters — always specify `-n kube-system` when inspecting `etcd`, `kube-apiserver`, `kube-scheduler`, or `kube-controller-manager` pods.
- Deleting a namespace **deletes all resources inside it** — there is no confirmation prompt.
- `namespace` defined in the manifest takes precedence over `-n` flag only if you use `kubectl apply` without `-n`. Using `-n` on the command line with `kubectl create` overrides the manifest namespace.

---

### In Managed Clusters (AKS / EKS / GKE)

> Namespace behaviour is identical across managed and self-managed clusters. AKS adds a few provider-managed namespaces you will encounter:

| Namespace | Purpose on AKS |
|---|---|
| `kube-system` | Contains AKS-managed system pods — `coredns`, `azure-ip-masq-agent`, `kube-proxy`, `metrics-server` |
| `gatekeeper-system` | Present when Azure Policy add-on is enabled — runs OPA Gatekeeper for policy enforcement |
| `monitoring` | Present when Azure Monitor / Container Insights add-on is enabled |

> Never modify or delete resources in AKS-managed namespaces — changes may be reconciled back by the provider or break cluster functionality.

---

