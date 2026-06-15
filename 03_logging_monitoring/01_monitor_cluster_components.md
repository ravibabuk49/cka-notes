## 01 — Monitor Cluster Components

### What it is
Kubernetes does not ship with a full-featured built-in monitoring solution. Monitoring is achieved through external or add-on tooling that collects metrics at two levels:

- **Node level** — number of nodes, health status, CPU, memory, network, disk utilization
- **Pod level** — number of pods, CPU and memory consumption per pod

---

### Metrics Server

The Metrics Server is the lightweight, in-memory monitoring solution that ships as a Kubernetes add-on. It is the primary monitoring tool relevant to the CKA exam.

| Property | Detail |
|---|---|
| **Origin** | Slimmed-down successor to the deprecated Heapster project |
| **Instances per cluster** | One |
| **Storage** | In-memory only — no disk persistence |
| **Historical data** | ❌ Not supported — use Prometheus or Elastic Stack for historical data |
| **Data source** | kubelet's cAdvisor subcomponent on each node |

---

### How Metrics Are Collected

```
Pod / Container
      │
      ▼
cAdvisor (subcomponent of kubelet on each node)
      │  retrieves performance metrics from pods
      ▼
kubelet API
      │  exposes metrics via HTTP endpoint
      ▼
Metrics Server
      │  aggregates metrics from all nodes
      ▼
kubectl top (consumer)
```

**cAdvisor (Container Advisor)** — a built-in subcomponent of the kubelet responsible for collecting CPU, memory, and other resource metrics from running containers and exposing them through the kubelet API. No separate installation required.

---

### Deploying the Metrics Server

#### On Minikube

```bash
minikube addons enable metrics-server
```

#### On all other clusters (kubeadm, cloud)

```bash
# Clone the metrics server deployment manifests
git clone https://github.com/kubernetes-sigs/metrics-server.git

# Deploy
kubectl apply -f metrics-server/deploy/kubernetes/

# Verify deployment
kubectl get deployment -n kube-system metrics-server
kubectl get pods -n kube-system | grep metrics-server
```

> After deployment, wait 1–2 minutes for the Metrics Server to collect and process data before running `kubectl top` commands.

---

### Viewing Metrics

```bash
# Node-level CPU and memory consumption
kubectl top nodes

# Pod-level CPU and memory consumption
kubectl top pods

# Pod metrics in a specific namespace
kubectl top pods -n <namespace>

# Sort by CPU or memory
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory

# Show metrics for containers within pods
kubectl top pods --containers
```

**Example output — `kubectl top nodes`:**
```
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
controlplane   166m         8%     1200Mi          31%
node01         120m         6%     900Mi           23%
```

---

### Monitoring Ecosystem — Beyond Metrics Server

| Tool | Type | Persistence | Best for |
|---|---|---|---|
| Metrics Server | Add-on | In-memory only | CKA exam, basic `kubectl top` |
| Prometheus | Open source | Disk (TSDB) | Production metrics + alerting |
| Elastic Stack | Open source | Disk | Logs + metrics unified |
| Datadog | SaaS | Cloud | Enterprise observability |
| Dynatrace | SaaS | Cloud | Enterprise APM + infra |

---

### Real-World Usage

| Scenario | Tool | Why |
|---|---|---|
| CKA exam — check node/pod resource usage | Metrics Server + `kubectl top` | Built-in, no setup required in exam environment |
| Production cluster alerting on CPU/memory thresholds | Prometheus + Alertmanager | Persistent storage, rich query language (PromQL), alerting rules |
| Unified log and metric correlation | Elastic Stack (ELK) | Single pane for logs and metrics across all nodes and pods |
| Kubernetes-native dashboards | Prometheus + Grafana | Pre-built Kubernetes dashboards, long-term trend analysis |
| Managed observability on AKS | Azure Monitor + Container Insights | Native AKS integration, no agent deployment required |

---

### Exam Gotchas

- **`kubectl top` returns error if Metrics Server is not deployed** — `error: Metrics API not available`. If this appears in the exam, Metrics Server is either not installed or not yet ready.
- **Metrics Server stores data in memory only** — it cannot answer questions about historical resource usage. Do not confuse it with Prometheus.
- **cAdvisor is not a separate process** — it is a subcomponent embedded inside the kubelet. You do not install or manage it separately.
- **Give Metrics Server time after deployment** — running `kubectl top` immediately after deploying Metrics Server returns no data. Wait 60–90 seconds.
- **Heapster is deprecated** — any reference to Heapster in older documentation or exam questions refers to the predecessor of Metrics Server. Metrics Server is the current replacement.
- **`kubectl top pods` does not show historical trends** — it shows the current point-in-time snapshot only.

---

### In Managed Clusters (AKS / EKS / GKE)

- AKS ships with **Azure Monitor Container Insights** enabled by default on new clusters — it provides node and pod metrics, log collection, and dashboards without deploying Metrics Server separately.
- The Metrics Server is also available on AKS as a managed add-on (`az aks enable-addons --addons monitoring`) and is required for **Horizontal Pod Autoscaler (HPA)** to function.
- On AKS, `kubectl top nodes` and `kubectl top pods` work out of the box once the Metrics Server add-on is enabled — no manual deployment needed.
- Container Insights uses the **Azure Monitor Agent** (formerly OMS Agent) as a DaemonSet on each node — the equivalent of cAdvisor + Metrics Server + log shipping in a single managed component.

---
