## Vertical Pod Autoscaler (VPA)

### What it is

The **Vertical Pod Autoscaler** automatically adjusts CPU and memory requests/limits on Pods in a Deployment based on observed usage. Unlike HPA which adds/removes Pods, VPA right-sizes individual Pods — replacing the manual workflow of `kubectl edit deployment` to update resource definitions.

> VPA is **not built into Kubernetes** — it must be deployed separately from the [VPA GitHub repo](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler). HPA is built-in since 1.23; VPA is not.

---

### VPA Components

VPA deploys three components into the `kube-system` namespace:

| Component | Role |
|---|---|
| **Recommender** | Continuously monitors resource usage via Metrics API; collects historical and live data; generates CPU/memory recommendations. Does NOT modify Pods directly. |
| **Updater** | Reads recommendations from Recommender; evicts (terminates) Pods whose resource usage is outside the recommended range, triggering recreation. |
| **Admission Controller** | Intercepts Pod creation requests; mutates the Pod spec to inject the Recommender's suggested CPU/memory values before the Pod starts. |

**Flow:**
```
Recommender → generates recommendation
Updater → evicts out-of-range Pod
Deployment controller → recreates Pod
Admission Controller → intercepts recreation, injects new resource values
New Pod starts with right-sized resources
```

---

### Deploying VPA

```bash
# Apply VPA components from the official repo
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/latest/download/vertical-pod-autoscaler.yaml

# Verify components are running
kubectl get pods -n kube-system | grep vpa
# vpa-admission-controller-...   Running
# vpa-recommender-...            Running
# vpa-updater-...                Running
```

---

### Creating a VPA — Declarative Only

There is no imperative `kubectl` command to create a VPA — declarative manifest only.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: myapp
      minAllowed:
        cpu: 100m
        memory: 50Mi
      maxAllowed:
        cpu: "2"
        memory: 1Gi
```

---

### VPA Update Modes

| Mode | Recommender | Updater | Admission Controller | Behaviour |
|---|---|---|---|---|
| `Off` | ✅ Active | ❌ Inactive | ❌ Inactive | Recommendations generated, no changes applied — read-only audit mode |
| `Initial` | ✅ Active | ❌ Inactive | ✅ Active | Resources set at Pod creation only — existing Pods never evicted |
| `Recreate` | ✅ Active | ✅ Active | ✅ Active | Updater evicts out-of-range Pods; Admission Controller sets resources on recreation |
| `Auto` | ✅ Active | ✅ Active | ✅ Active | Same as `Recreate` today; will use in-place resizing when stable (see `13_inplace_pod_resizing.md`) |

> `Auto` is forward-looking — once `InPlacePodVerticalScaling` becomes stable, `Auto` will resize without eviction. As of Kubernetes 1.33, `Auto` behaves identically to `Recreate`.

---

### Viewing VPA Recommendations

```bash
kubectl describe vpa myapp-vpa
```

Output includes a `Recommendation` section showing suggested CPU and memory values per container — useful even in `Off` mode for understanding actual resource needs before committing to automated changes.

---

### HPA vs VPA — Comparison

| Dimension | HPA | VPA |
|---|---|---|
| Scaling method | Adds/removes Pods | Resizes CPU/memory on existing Pods |
| Pod disruption | None — new Pods added alongside | Yes — Pod evicted and recreated (until in-place resizing is stable) |
| Traffic spike handling | Excellent — instant new Pods | Poor — requires Pod restart, introduces delay |
| Cost optimisation | Removes idle Pods | Prevents over-provisioning of CPU/memory per Pod |
| Best for | Stateless workloads, web APIs, microservices | Stateful workloads, databases, JVM apps, AI workloads |

---

### When to Use Which

| Use case | Use |
|---|---|
| Web servers, APIs, message consumers — traffic-driven scaling | HPA |
| Stateless microservices with fluctuating request rates | HPA |
| Databases, JVM apps, AI workloads — right-sizing resource allocation | VPA |
| Apps with heavy startup CPU then lower steady-state usage | VPA |
| Unknown resource requirements — want data-driven recommendations first | VPA in `Off` mode |
| Large-scale production — optimal coverage | HPA + VPA together (with care — see gotchas) |

---

### Exam Gotchas

- VPA is **not built-in** — it requires separate installation. HPA is built-in since 1.23. This distinction is frequently tested.
- There is **no imperative command** to create a VPA — declarative only. `kubectl autoscale` only works for HPA.
- `apiVersion: autoscaling.k8s.io/v1` — note the full domain, not just `autoscaling/v2` as with HPA.
- **HPA and VPA should not both target CPU on the same Deployment** — they will conflict. VPA changes resource limits, which alters HPA's utilisation calculations. Use HPA for scaling Pod count and VPA for non-CPU metrics (memory), or use one at a time.
- `Off` mode is useful in production as a **recommendation-only** tool — no risk, just data.
- VPA **cannot scale to zero** — use HPA with KEDA for that.
- `Auto` mode does not yet use in-place resizing — it still evicts Pods as of Kubernetes 1.33.

---

### Real-World Usage

- **JVM applications**: Java apps often need significantly more memory at startup (class loading, JIT compilation) than at steady state. VPA in `Recreate` mode continuously right-sizes memory allocation, preventing both OOMKill (too low) and wasted reserved memory (too high).
- **Databases (PostgreSQL, MySQL)**: stateful workloads that cannot easily scale horizontally benefit from VPA ensuring they always have sufficient CPU/memory without over-provisioning.
- **`Off` mode for resource auditing**: deploying VPA in `Off` mode against all Deployments is a common cost-optimisation practice — teams review recommendations periodically and update manifests manually, using VPA as a data source rather than an automation tool.
- **HPA + VPA together**: the supported combination is HPA scaling on custom metrics (e.g. RPS) while VPA manages CPU/memory sizing. Never run both targeting the same CPU metric.

---

### In Managed Clusters (AKS)

VPA must be manually deployed in AKS — it is not a managed add-on. The [AKS VPA add-on](https://learn.microsoft.com/en-us/azure/aks/vertical-pod-autoscaler) is available in preview, which manages the VPA deployment and lifecycle. For production AKS workloads, KEDA is the preferred scaling extension for event-driven and custom metric scenarios; VPA fills the vertical right-sizing gap that KEDA does not address.

