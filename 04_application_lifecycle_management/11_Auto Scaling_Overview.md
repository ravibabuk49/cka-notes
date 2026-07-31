## Auto Scaling Overview

> This is an introductory orientation section. HPA, VPA, and Cluster Autoscaler are each covered in detail in the sections that follow.

---

### What it is

Auto scaling in Kubernetes is the automated adjustment of resources — either workloads or cluster infrastructure — in response to demand. There are two dimensions to scaling and two approaches to doing it.

---

### Two Dimensions of Scaling

| Dimension | Horizontal | Vertical |
|---|---|---|
| **Cluster infra** | Add/remove nodes | Increase resources on existing nodes |
| **Workloads** | Add/remove Pods | Increase CPU/memory allocated to existing Pods |

> Vertical scaling of cluster nodes is rarely used in practice — it requires taking down the node and its workloads. The standard alternative is to provision a new larger node, drain the old one, and remove it.

---

### Manual vs Automated Scaling

**Manual — Cluster infra:**
```bash
# Add a node (self-managed cluster)
kubeadm join <control-plane-endpoint> --token <token> --discovery-token-ca-cert-hash <hash>
```

**Manual — Workload horizontal scaling:**
```bash
kubectl scale deployment <name> --replicas=5
```

**Manual — Workload vertical scaling:**
```bash
kubectl edit deployment <name>   # update resources.requests/limits on the container
```

---

### Automated Scaling — The Three Tools

| Tool | Scales | Direction |
|---|---|---|
| **Cluster Autoscaler (CA)** | Cluster nodes | Horizontal |
| **Horizontal Pod Autoscaler (HPA)** | Pod count | Horizontal |
| **Vertical Pod Autoscaler (VPA)** | Pod CPU/memory | Vertical |

Each is covered in its own dedicated section.

---

### Exam Gotchas

- Know which tool maps to which scaling type — CA scales nodes, HPA scales Pods horizontally, VPA scales Pods vertically. Mixing these up in an exam answer is a common mistake.
- Manual `kubectl scale` is horizontal workload scaling — not the same as HPA, which does it automatically based on metrics.
- Vertical node scaling is not a standard Kubernetes pattern — the exam will not ask you to vertically scale a node.

