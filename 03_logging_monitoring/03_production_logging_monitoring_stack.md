## 03 — Production Logging & Monitoring Stack

### What it is
A reference for how production Kubernetes clusters implement observability beyond the built-in Metrics Server. Production observability covers three pillars:

- **Metrics** — numerical measurements over time (CPU, memory, request rate, error rate)
- **Logs** — structured or unstructured event records from containers and nodes
- **Traces** — distributed request flows across services (out of CKA scope)

This section covers the standard open-source and cloud-native stacks used in production, how they integrate with Kubernetes, and how they map to AKS-native equivalents.

---

### The Two Core Stacks

#### Stack 1 — Metrics: Prometheus + Grafana

| Component | Role |
|---|---|
| **Prometheus** | Scrapes metrics from Kubernetes components and applications, stores them in a time-series database (TSDB) on disk |
| **Grafana** | Queries Prometheus via PromQL and renders dashboards |
| **kube-state-metrics** | Exposes Kubernetes object state as metrics (Pod restarts, Deployment replicas, PVC status) |
| **Node Exporter** | DaemonSet that exposes node-level OS metrics (disk I/O, filesystem, network) |
| **Alertmanager** | Receives alerts from Prometheus rules and routes them to Slack, PagerDuty, email, etc. |

**How Prometheus scrapes Kubernetes:**
```
Pods / Nodes / Control plane components
        │  expose /metrics endpoint (HTTP)
        ▼
Prometheus scrapes every N seconds
        │  stores in local TSDB
        ▼
Grafana / Alertmanager query Prometheus
```

Prometheus discovers scrape targets via **ServiceMonitor** and **PodMonitor** objects (from the `kube-prometheus-stack` Helm chart), or via static scrape configs.

---

#### Stack 2 — Logs: Fluent Bit → Elastic Stack (ELK)

| Component | Role |
|---|---|
| **Fluent Bit** | Lightweight log shipper — runs as a DaemonSet on every node, tails container log files from `/var/log/containers/`, and forwards to a backend |
| **Fluentd** | Heavier log processor — used when log transformation or aggregation logic is complex |
| **Elasticsearch** | Stores and indexes log data |
| **Kibana** | Queries and visualises logs stored in Elasticsearch |
| **Logstash** | Optional — transforms/enriches logs before indexing (often replaced by Fluent Bit filters) |

**How Fluent Bit integrates with Kubernetes:**
```
Container writes to stdout/stderr
        │
        ▼
Container runtime writes to /var/log/containers/<pod>_<namespace>_<container>.log
        │
        ▼
Fluent Bit DaemonSet tails these files on each node
        │  enriches with Kubernetes metadata (pod name, namespace, labels)
        ▼
Forwards to Elasticsearch / Azure Monitor / CloudWatch / Loki
```

Fluent Bit is the preferred choice over Fluentd for most clusters today — lower memory footprint, faster, and sufficient for the majority of log routing use cases.

---

### Deployment Pattern in Kubernetes

Both stacks follow the same Kubernetes-native deployment pattern:

| Component | Kubernetes Object | Why |
|---|---|---|
| Fluent Bit / Node Exporter | DaemonSet | Must run on every node |
| Prometheus / Elasticsearch | StatefulSet | Requires persistent storage and stable identity |
| Grafana / Kibana | Deployment | Stateless UI, multiple replicas for HA |
| Alertmanager | StatefulSet or Deployment | Depends on HA requirements |

---

### Deploying with Helm (Standard Approach)

```bash
# Add Prometheus community Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Deploy full Prometheus + Grafana + Alertmanager + kube-state-metrics + Node Exporter stack
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace

# Verify all components are running
kubectl get pods -n monitoring

# Access Grafana (port-forward)
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
# Default credentials: admin / prom-operator
```

```bash
# Add Elastic Helm repo
helm repo add elastic https://helm.elastic.co
helm repo update

# Deploy Elasticsearch
helm install elasticsearch elastic/elasticsearch -n logging --create-namespace

# Deploy Fluent Bit
helm repo add fluent https://fluent.github.io/helm-charts
helm install fluent-bit fluent/fluent-bit -n logging

# Deploy Kibana
helm install kibana elastic/kibana -n logging
```

---

### When to Use Which

| Need | Stack | Reason |
|---|---|---|
| CKA exam — check resource usage | Metrics Server + `kubectl top` | Built-in, no setup |
| Production metrics + alerting | Prometheus + Grafana + Alertmanager | Persistent TSDB, PromQL, rich alerting |
| Production log aggregation | Fluent Bit + Elasticsearch + Kibana | Persistent, searchable, Kubernetes-aware |
| Lightweight log forwarding only | Fluent Bit → CloudWatch / Azure Monitor | No Elasticsearch overhead |
| Unified metrics + logs | Grafana Loki + Fluent Bit + Grafana | Single Grafana UI for both pillars |
| Managed AKS observability | Azure Monitor + Container Insights | Zero deployment, native integration |

---

### Real-World Usage

#### Pattern 1 — Standard OSS Stack (self-managed or AKS)
```
Node Exporter (DaemonSet) ──→ Prometheus ──→ Grafana dashboards
kube-state-metrics         ──→ Prometheus ──→ Alertmanager ──→ Slack / PagerDuty
Fluent Bit (DaemonSet)     ──→ Elasticsearch ──→ Kibana
```
Used by most teams running self-managed kubeadm or EKS/GKE clusters. Full control, no vendor lock-in.

#### Pattern 2 — AKS Native
```
Azure Monitor Agent (DaemonSet, managed) ──→ Log Analytics Workspace
                                          ──→ Azure Monitor Metrics
                                          ──→ Container Insights dashboards
                                          ──→ Azure Alerts
```
Zero deployment effort on AKS. Metrics, logs, and alerts in one managed service. Trade-off: vendor lock-in and KQL instead of PromQL.

#### Pattern 3 — Hybrid (AKS + OSS)
```
Azure Monitor Agent    ──→ Log Analytics (compliance logs, audit)
Prometheus (managed)   ──→ Azure Managed Prometheus ──→ Azure Managed Grafana
Fluent Bit (DaemonSet) ──→ Elasticsearch (application logs)
```
Common in enterprises that need compliance-grade log retention in Azure but want Grafana/PromQL for operational dashboards.

---

### Key File Paths on Nodes

```bash
# Container log files on each node (tailed by Fluent Bit)
/var/log/containers/                    # symlinks to actual log files
/var/log/pods/                          # actual log files per pod/container

# Kubelet logs (systemd-managed nodes)
journalctl -u kubelet -f

# Node-level system logs
journalctl -f
```

---

### Exam Relevance

This section is beyond CKA exam scope — the exam only tests `kubectl top` and `kubectl logs`. However the following concepts from this section do appear in adjacent exam topics:

| Concept | Relevant CKA domain |
|---|---|
| DaemonSet for log/metric agents | `02_scheduling` → `09_daemonsets.md` |
| PersistentVolume for Prometheus/Elasticsearch storage | `07_storage` |
| RBAC for Prometheus scraping | `06_security` |
| Helm deployment | `12_helm` |

---

