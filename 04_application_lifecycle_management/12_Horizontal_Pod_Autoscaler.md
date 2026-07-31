## Horizontal Pod Autoscaler (HPA)

### What it is

The **Horizontal Pod Autoscaler** automatically increases or decreases the number of Pods in a Deployment, StatefulSet, or ReplicaSet based on observed resource utilisation (CPU, memory) or custom/external metrics. It replaces the manual workflow of monitoring `kubectl top pod` and running `kubectl scale` by doing both continuously and automatically.

**Prerequisite:** Metrics Server must be running in the cluster — HPA pulls current resource utilisation from it.

---

### Manual Scaling vs HPA

| Manual approach | HPA |
|---|---|
| Run `kubectl top pod` continuously | Polls Metrics Server automatically |
| Run `kubectl scale` when threshold is hit | Adjusts replica count automatically |
| Operator must be available at all times | Reacts immediately, including during traffic spikes |
| Human error and reaction delay | Consistent, policy-driven |

---

### How HPA Works

1. Reads the resource **limits** configured on the Pod (e.g. `500m` CPU).
2. Calculates target utilisation as a percentage of that limit (e.g. 50% of 500m = 250m threshold).
3. Continuously polls Metrics Server for current usage.
4. Scales the replica count up or down to keep utilisation near the target.
5. Never scales below `minReplicas` or above `maxReplicas`.

---

### Creating HPA — Imperative

```bash
kubectl autoscale deployment myapp \
  --cpu-percent=50 \
  --min=1 \
  --max=10
```

This creates an HPA targeting the `myapp` Deployment, scaling between 1 and 10 replicas to keep CPU utilisation at or below 50% of the Pod's configured limit.

---

### Creating HPA — Declarative

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

> `apiVersion: autoscaling/v2` — use v2, not v1. v2 supports multiple metrics and is the current standard. HPA is built into Kubernetes since **v1.23** — no separate installation required.

---

### Viewing and Managing HPA

```bash
# List HPAs and current status
kubectl get hpa

# Detailed view — shows current vs target metrics and replica counts
kubectl describe hpa myapp-hpa

# Delete HPA (deployment continues running at its current replica count)
kubectl delete hpa myapp-hpa
```

The `kubectl get hpa` output shows:

```
NAME        REFERENCE          TARGETS   MINPODS   MAXPODS   REPLICAS
myapp-hpa   Deployment/myapp   23%/50%   1         10        3
```

`TARGETS` = current utilisation / configured threshold.

---

### Metrics Sources

| Source type | Description | Example |
|---|---|---|
| **Resource metrics** (default) | CPU and memory from Metrics Server | `cpu`, `memory` |
| **Custom metrics** | Application-level metrics from internal adapters | Requests per second, queue depth |
| **External metrics** | Metrics from tools outside the cluster via external adapters | Datadog, Dynatrace |

> Custom and external metrics adapters are beyond CKA exam scope — covered in the dedicated Kubernetes Auto Scaling course.

---

### When to Use Which (Metrics)

| Use case | Metric type |
|---|---|
| CPU-bound workloads (compute-heavy APIs, batch jobs) | CPU utilisation |
| Memory-bound workloads (caches, in-memory data stores) | Memory utilisation |
| Request-driven workloads (web servers, message consumers) | Custom metrics (requests/sec, queue depth) |

---

### Exam Gotchas

- HPA requires **Metrics Server** — if Metrics Server is not running, HPA will show `<unknown>` in the TARGETS column and will not scale.
- HPA reads the Pod's **resource limits**, not requests, to calculate utilisation percentage. If no limit is set on the container, CPU-based HPA cannot function.
- `kubectl autoscale` is the imperative shortcut — remember the flags: `--cpu-percent`, `--min`, `--max`.
- Declarative HPA uses `apiVersion: autoscaling/v2` — not `v1`. The v2 API supports multiple metrics simultaneously.
- Deleting an HPA does **not** delete or scale down the Deployment — the Deployment stays at whatever replica count it was at when the HPA was deleted.
- HPA and manual `kubectl scale` conflict — if you manually scale a Deployment managed by HPA, HPA will override it on the next reconciliation loop.
- `scaleTargetRef` requires both `kind` and `name` — missing either causes the HPA to fail silently.

---

### Real-World Usage

- **Web APIs with variable traffic**: HPA on CPU or RPS (requests per second via custom metrics) is the standard pattern for API deployments that experience predictable daily load cycles (e.g. business hours vs. off-hours).
- **AKS + KEDA**: in Azure production workloads, KEDA (Kubernetes Event-Driven Autoscaling) extends HPA with native support for Azure Service Bus queue depth, Event Hub lag, and other Azure-native metrics — more expressive than raw CPU/memory for event-driven architectures.
- **HPA + Cluster Autoscaler together**: HPA scales Pods up; when nodes run out of capacity to schedule those Pods, Cluster Autoscaler adds new nodes. The two work in tandem — HPA handles workload scaling, CA handles infrastructure scaling.
- **Cost optimisation**: `minReplicas: 0` (supported in HPA v2) allows scaling a Deployment to zero during off-hours, eliminating compute cost for non-critical workloads — requires KEDA or a custom metric since CPU-based HPA cannot scale from zero (no Pods = no metrics).

---

### In Managed Clusters (AKS)

- Metrics Server is **pre-installed** in AKS — no manual setup required.
- AKS integrates natively with Cluster Autoscaler — enabling both HPA and CA together is the recommended production scaling configuration.
- For Azure-native metric sources (Service Bus, Event Hubs, Blob Storage), use **KEDA** (also available as an AKS add-on) rather than raw HPA custom metrics adapters.

