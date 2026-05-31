## 15 — Services

### What it is

A **Service** is a Kubernetes object that provides stable network access to a set of pods. Since pod IPs are ephemeral and change when pods are rescheduled, a Service provides a fixed endpoint — a stable IP, DNS name, and port — that abstracts the underlying pods. Services enable loose coupling between application components and allow external access to workloads running inside the cluster.

---

### Service Types

| Type | Purpose |
|---|---|
| `NodePort` | Exposes a pod on a static port on every node — enables external access to the cluster |
| `ClusterIP` | Creates a virtual IP inside the cluster — enables internal communication between services |
| `LoadBalancer` | Provisions a cloud load balancer — distributes external traffic across pods. Only works on supported cloud providers (AKS, EKS, GKE) |

---

## NodePort

### How NodePort Works

Three ports are involved in a NodePort service — all from the viewpoint of the service:

```
External user
      │
      ▼
Node IP : NodePort (30008)        ← port on the node (30000–32767)
      │
      ▼
Service : Port (80)               ← port on the service object (ClusterIP)
      │
      ▼
Pod : TargetPort (80)             ← port on the pod where the app listens
```

| Port term | What it refers to | Mandatory? |
|---|---|---|
| `targetPort` | Port on the pod where the application listens | No — defaults to `port` if omitted |
| `port` | Port on the Service object itself | ✅ Yes |
| `nodePort` | Port on the node exposed externally | No — auto-assigned in range 30000–32767 if omitted |

---

### NodePort Definition File

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: NodePort
  ports:
    - targetPort: 80
      port: 80
      nodePort: 30008
  selector:
    app: myapp
```

**Key structural points:**
- `ports` is an **array** — note the `-` before `targetPort`. Multiple port mappings can exist in one Service.
- `selector` links the Service to pods — it must match the labels on the target pods exactly, same as ReplicaSet selectors.
- Without `selector`, the Service has no endpoints and forwards traffic nowhere.

---

### NodePort — Multiple Pods and Multiple Nodes

The Service behaves consistently regardless of how many pods or nodes are involved:

| Scenario | Service behaviour |
|---|---|
| Single pod, single node | Forwards traffic to that one pod |
| Multiple pods, single node | Automatically selects all matching pods as endpoints — uses a **random** load balancing algorithm |
| Multiple pods, multiple nodes | Service spans all nodes automatically — the same `nodePort` is accessible on every node's IP |

> No additional configuration is required for any of these scenarios. When pods are added or removed, the Service endpoints are updated automatically.

**Accessing the application via NodePort:**

```bash
curl http://192.168.1.2:30008      # via node IP + nodePort
```

---

## ClusterIP

### How ClusterIP Works

`ClusterIP` is the default Service type. It creates a virtual IP address inside the cluster that is only reachable from within the cluster — not from outside.

Used for internal service-to-service communication:

```
Frontend pods → ClusterIP Service → Backend pods
Backend pods  → ClusterIP Service → Database pods
```

Each tier gets a stable internal IP and DNS name regardless of how many pods are behind it or how often they are rescheduled.

---

### ClusterIP Definition File

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  ports:
    - targetPort: 80
      port: 80
  selector:
    app: backend
    type: backend
```

> `ClusterIP` is the default — if `type` is omitted from the spec, Kubernetes creates a ClusterIP service.

---

## LoadBalancer

### How LoadBalancer Works

`LoadBalancer` extends `NodePort` by additionally provisioning a cloud provider load balancer (Azure Load Balancer, AWS ELB, GCP Load Balancer) that routes external traffic to the NodePort on all nodes. The cloud LB gets a public IP that external users connect to.

```
External user
      │
      ▼
Cloud Load Balancer (public IP)
      │
      ▼
NodePort on Node 1 / Node 2 / Node 3
      │
      ▼
Pods
```

> `LoadBalancer` only works on supported cloud providers. On bare-metal clusters, it behaves like `NodePort` — no external load balancer is provisioned.

---

### kubectl Commands

```bash
# Create a service from file
kubectl create -f service-definition.yaml
kubectl apply -f service-definition.yaml

# Generate Service YAML without creating
kubectl expose pod myapp-pod --type=NodePort --port=80 --dry-run=client -o yaml

# List services
kubectl get services
kubectl get svc                              # shorthand

# Detailed service information — endpoints, selectors, ports
kubectl describe service myapp-service

# Delete a service
kubectl delete service myapp-service
```

---

### Exam Gotchas

- `NodePort` range is **30000–32767** — specifying a port outside this range causes a validation error.
- The only mandatory port field is `port` — `targetPort` and `nodePort` are both optional with sensible defaults.
- `selector` in the Service must **exactly match** the labels on the target pods — a mismatch results in a service with no endpoints.
- `ClusterIP` is the **default** service type — omitting `type` creates a ClusterIP service, not a NodePort.
- On multi-node clusters, `NodePort` is automatically available on **every node's IP** — you can use any node's IP with the same port to reach the application.
- A Service with no matching pods shows `Endpoints: <none>` in `kubectl describe` — always check endpoints when a service is unreachable.
- `LoadBalancer` on bare-metal clusters (without a cloud controller) stays in `<pending>` state for `EXTERNAL-IP` indefinitely — use `NodePort` or MetalLB instead.

---

### In Managed Clusters (AKS / EKS / GKE)

> `LoadBalancer` services are the primary way to expose applications externally on managed clusters — the cloud provider automatically provisions and manages the load balancer infrastructure.

| Service type | AKS behaviour |
|---|---|
| `ClusterIP` | Identical to self-managed — internal only |
| `NodePort` | Works but rarely used directly — `LoadBalancer` is preferred for external access |
| `LoadBalancer` | Provisions an **Azure Load Balancer** with a public IP automatically. The external IP appears in `kubectl get svc` under `EXTERNAL-IP` once provisioned. |

**AKS-specific additions:**
- Annotate a `LoadBalancer` service with `service.beta.kubernetes.io/azure-load-balancer-internal: "true"` to create an **internal** Azure Load Balancer (private IP on the VNet) instead of a public one.
- AKS also supports **Ingress controllers** (nginx, Azure Application Gateway) as a more flexible alternative to multiple LoadBalancer services — covered in the Networking section.

---

