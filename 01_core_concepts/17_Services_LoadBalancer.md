## 17 — Services — LoadBalancer

### What it is

`LoadBalancer` is a Service type that extends `NodePort` by additionally provisioning a cloud provider's native load balancer. It provides a **single stable external URL** to route traffic to the application — replacing the need to expose multiple node IPs and ports to end users.

> LoadBalancer was introduced in [Section 15 — Services](./15_services.md). This section covers its purpose and behaviour in detail.

---

### The Problem LoadBalancer Solves

A `NodePort` service exposes an application on a port across **all nodes** in the cluster. In a 4-node cluster, this results in multiple access points:

```
Voting app:  http://192.168.1.70:30001
             http://192.168.1.71:30001
             http://192.168.1.72:30001
             http://192.168.1.73:30001

Result app:  http://192.168.1.70:30002
             http://192.168.1.71:30002
             ...
```

> Even if pods are only deployed on 2 of the 4 nodes, the NodePort is still accessible on all node IPs.

End users need a single URL — not a list of IP:port combinations. `LoadBalancer` solves this by fronting all nodes with a single cloud-managed load balancer endpoint:

```
http://vote.example.com   →   Cloud Load Balancer   →   NodePort on all nodes   →   Pods
```

---

### LoadBalancer Definition File

```yaml
apiVersion: v1
kind: Service
metadata:
  name: voting-app-service
spec:
  type: LoadBalancer
  ports:
    - targetPort: 80
      port: 80
      nodePort: 30005
  selector:
    app: voting-app
```

> The only change from a `NodePort` definition is `type: LoadBalancer`. Everything else is identical.

---

### Behaviour by Environment

| Environment | LoadBalancer behaviour |
|---|---|
| AKS / EKS / GKE (supported cloud) | Provisions a cloud-native load balancer with a public IP automatically |
| Bare-metal / VirtualBox / unsupported | Behaves exactly like `NodePort` — no external load balancer is provisioned. `EXTERNAL-IP` stays `<pending>` indefinitely |

---

### kubectl Commands

```bash
# Create LoadBalancer service
kubectl apply -f loadbalancer-service.yaml

# Check external IP assigned by cloud provider
kubectl get svc voting-app-service

# Watch until EXTERNAL-IP is assigned
kubectl get svc voting-app-service --watch
```

**Example output on a cloud cluster:**
```
NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)
voting-app-service   LoadBalancer   10.96.0.25     52.174.10.30    80:30005/TCP
```

**Example output on bare-metal:**
```
NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)
voting-app-service   LoadBalancer   10.96.0.25     <pending>     80:30005/TCP
```

---

### Exam Gotchas

- `LoadBalancer` on unsupported environments behaves identically to `NodePort` — it does not error, it just never provisions an external load balancer.
- `EXTERNAL-IP` stuck at `<pending>` on a cloud cluster means the cloud controller manager is not running or misconfigured — not a Kubernetes issue.
- Do not use `LoadBalancer` for every service — each one provisions a separate cloud load balancer which incurs cost. Use **Ingress** to route multiple services through a single load balancer instead.
- On bare-metal clusters, **MetalLB** is the standard solution to enable `LoadBalancer` type services — it assigns IPs from a configured pool.

---

### In Managed Clusters (AKS / EKS / GKE)

> `LoadBalancer` is the primary external exposure mechanism on managed clusters and is fully integrated with the cloud provider.

| Aspect | AKS behaviour |
|---|---|
| Public LoadBalancer | Provisions an Azure Public Load Balancer with a public IP automatically |
| Internal LoadBalancer | Add annotation `service.beta.kubernetes.io/azure-load-balancer-internal: "true"` — provisions an Azure Internal Load Balancer with a private VNet IP |
| DNS | Assign a DNS label via annotation `service.beta.kubernetes.io/azure-dns-label-name: "myapp"` — gives a stable FQDN instead of relying on the IP |
| Cost consideration | Each `LoadBalancer` service provisions a separate Azure Load Balancer — use **Azure Application Gateway Ingress Controller (AGIC)** or **nginx Ingress** to consolidate multiple services behind one load balancer |

---

