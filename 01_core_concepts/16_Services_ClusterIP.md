## 16 — Services — ClusterIP

### What it is

`ClusterIP` is the **default** Kubernetes Service type. It creates a virtual IP address and DNS name inside the cluster that provides a stable, single interface to a group of pods. It is the standard mechanism for internal service-to-service communication in a microservices architecture.

> ClusterIP was introduced in [Section 15 — Services](./15_services.md). This section covers its purpose and configuration in detail.

---

### The Problem ClusterIP Solves

In a multi-tier application, pods across tiers need to communicate with each other:

```
Frontend pods → Backend pods → Redis pods → MySQL pods
```

Pod IPs cannot be used directly for this because:
- Pod IPs are **ephemeral** — they change whenever a pod is rescheduled or recreated
- Multiple pods exist per tier — a frontend pod cannot decide which of three backend pods to call
- There is no built-in mechanism to load balance across pods without a Service

**ClusterIP solves both problems:**
- Provides a **single stable IP and DNS name** per tier — regardless of how many pods are behind it or how often they change
- **Randomly load balances** requests across all healthy pods matching the selector

---

### ClusterIP in a Multi-Tier Architecture

```
Frontend pods (10.244.0.2 / .0.3 / .0.4)
        │
        ▼
   "backend" ClusterIP Service
        │
        ▼
Backend pods (10.244.0.5 / .0.6 / .0.7)
        │
        ▼
   "redis" ClusterIP Service
        │
        ▼
Redis pods (10.244.0.8 / .0.9 / .0.10)
```

Each tier scales and moves independently. Because pods always communicate via the Service name — not pod IPs — no configuration changes are required when pods are added, removed, or rescheduled.

---

### ClusterIP Definition File

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  ports:
    - targetPort: 80      # port on the backend pods
      port: 80            # port exposed by the service
  selector:
    app: myapp
    type: backend
```

> `type: ClusterIP` is the default — omitting the `type` field produces a ClusterIP service automatically.

---

### Accessing a ClusterIP Service

Once created, the Service is accessible from any pod in the cluster using either:

```bash
# Via ClusterIP (assigned automatically)
curl http://10.96.0.10:80

# Via Service name (preferred — DNS-based, stable)
curl http://backend:80

# Via fully qualified domain name (cross-namespace)
curl http://backend.default.svc.cluster.local:80
```

> Always use the **Service name** for inter-pod communication — not the ClusterIP. The name is stable and human-readable. DNS resolution is handled by CoreDNS.

---

### kubectl Commands

```bash
# Create the service
kubectl create -f clusterip-service.yaml
kubectl apply -f clusterip-service.yaml

# Generate ClusterIP YAML without creating
kubectl expose deployment backend --port=80 --target-port=80 --dry-run=client -o yaml

# List services
kubectl get services
kubectl get svc

# Verify endpoints — confirms pods are correctly selected
kubectl get endpoints backend

# Detailed service info — selector, IP, endpoints
kubectl describe service backend
```

---

### Exam Gotchas

- `ClusterIP` is the **default** service type — if `type` is omitted from the spec, a ClusterIP service is created.
- A ClusterIP Service is **only reachable from within the cluster** — it is not accessible externally.
- Pods should always reference services by **name**, not ClusterIP — names are stable, IPs can theoretically change if a service is recreated.
- If a ClusterIP service returns no response, run `kubectl get endpoints <service-name>` — if it shows `<none>`, the `selector` does not match any pod labels.
- The fully qualified DNS name format is: `<service-name>.<namespace>.svc.cluster.local` — required when accessing a service from a different namespace.

---

### In Managed Clusters (AKS / EKS / GKE)

> ClusterIP behaviour is identical across managed and self-managed clusters — no difference. Internal DNS resolution via CoreDNS works the same way on AKS as on any Kubernetes cluster.

---

