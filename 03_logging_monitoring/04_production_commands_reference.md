## 04 — Production Commands Reference — Logging & Monitoring

> Production-grade command reference for observability in Kubernetes clusters.
> Not exam-focused — use this for day-to-day operations and AKS work.

---

## Metrics Server

```bash
# Deploy Metrics Server (kubeadm / cloud clusters)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify deployment
kubectl get deployment -n kube-system metrics-server
kubectl rollout status deployment/metrics-server -n kube-system

# Check Metrics Server logs (useful if kubectl top returns errors)
kubectl logs -n kube-system -l k8s-app=metrics-server --tail=50

# Check Metrics API availability
kubectl get apiservices | grep metrics

# Node metrics
kubectl top nodes
kubectl top nodes --sort-by=cpu
kubectl top nodes --sort-by=memory

# Pod metrics
kubectl top pods -A                              # all namespaces
kubectl top pods -n <namespace>
kubectl top pods --sort-by=cpu -A
kubectl top pods --sort-by=memory -A
kubectl top pods --containers -A                 # per-container breakdown
kubectl top pods -l app=payments-api             # filter by label
```

---

## kubectl logs — Full Reference

```bash
# Basic
kubectl logs <pod-name>
kubectl logs <pod-name> -f                       # stream live
kubectl logs <pod-name> --tail=100               # last N lines
kubectl logs <pod-name> --since=1h               # last 1 hour
kubectl logs <pod-name> --since=30m              # last 30 minutes
kubectl logs <pod-name> --since-time="2024-01-15T10:00:00Z"  # since exact time
kubectl logs <pod-name> --timestamps             # prepend timestamps

# Multi-container pods
kubectl logs <pod-name> -c <container-name>
kubectl logs <pod-name> --all-containers=true
kubectl logs <pod-name> --all-containers=true -f

# Crashed / restarted containers
kubectl logs <pod-name> -c <container-name> --previous

# Deployments / ReplicaSets — logs from all pods matching a label
kubectl logs -l app=payments-api --all-containers=true
kubectl logs -l app=payments-api --all-containers=true --tail=50

# Logs from all pods in a namespace matching a label (streaming)
kubectl logs -l app=payments-api -n production -f --max-log-requests=10

# Output logs to a file
kubectl logs <pod-name> > pod-logs.txt
kubectl logs <pod-name> --since=1h >> pod-logs.txt
```

---

## Node-Level Log Access

```bash
# SSH into a node and view kubelet logs
journalctl -u kubelet -f
journalctl -u kubelet --since="1 hour ago"
journalctl -u kubelet -n 100                     # last 100 lines

# Container runtime logs (containerd)
journalctl -u containerd -f

# System logs on the node
journalctl -f

# Container log files on the node filesystem
ls /var/log/containers/                          # symlinks per container
ls /var/log/pods/                                # actual log files per pod

# Tail a specific container log file directly on the node
tail -f /var/log/containers/<pod>_<namespace>_<container>-<id>.log
```

---

## crictl — Container Runtime Debugging on Nodes

```bash
# List running containers
crictl ps

# List all containers including stopped/crashed
crictl ps -a

# View logs of a specific container
crictl logs <container-id>
crictl logs <container-id> -f                    # stream
crictl logs <container-id> --tail=50

# Inspect container details
crictl inspect <container-id>

# List pods on the node
crictl pods

# List images on the node
crictl images

# Exec into a container
crictl exec -it <container-id> sh

# Remove stopped containers (cleanup)
crictl rm <container-id>

# Set runtime endpoint if needed
export CONTAINER_RUNTIME_ENDPOINT=unix:///run/containerd/containerd.sock
```

---

## Prometheus (kube-prometheus-stack)

```bash
# Deploy via Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=<your-password>

# Verify all components
kubectl get pods -n monitoring
kubectl get svc -n monitoring

# Access Prometheus UI
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090

# Access Grafana UI
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
# Default credentials: admin / prom-operator

# Access Alertmanager UI
kubectl port-forward -n monitoring svc/kube-prometheus-stack-alertmanager 9093:9093

# Check Prometheus targets (is it scraping correctly?)
# Open http://localhost:9090/targets after port-forward

# View Prometheus rules
kubectl get prometheusrules -n monitoring
kubectl describe prometheusrule <rule-name> -n monitoring

# View ServiceMonitors (what Prometheus is scraping)
kubectl get servicemonitors -n monitoring
kubectl get servicemonitors -A

# Upgrade the stack
helm upgrade kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring

# Uninstall
helm uninstall kube-prometheus-stack -n monitoring
```

---

## Fluent Bit

```bash
# Deploy via Helm
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
helm install fluent-bit fluent/fluent-bit \
  --namespace logging \
  --create-namespace

# Verify DaemonSet — one pod per node
kubectl get ds -n logging
kubectl get pods -n logging -o wide              # confirms one per node

# View Fluent Bit logs (useful for debugging log forwarding issues)
kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit --tail=50
kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit -f

# View Fluent Bit config
kubectl get configmap -n logging fluent-bit -o yaml

# Edit Fluent Bit config (add outputs, filters, parsers)
kubectl edit configmap -n logging fluent-bit
# Restart pods to pick up config changes
kubectl rollout restart daemonset/fluent-bit -n logging

# Check Fluent Bit metrics endpoint
kubectl port-forward -n logging <fluent-bit-pod> 2020:2020
curl http://localhost:2020/api/v1/metrics
```

---

## AKS-Specific — Azure Monitor & Container Insights

```bash
# Enable Container Insights on an existing AKS cluster
az aks enable-addons \
  --resource-group <rg-name> \
  --name <aks-cluster-name> \
  --addons monitoring \
  --workspace-resource-id <log-analytics-workspace-id>

# Check if monitoring addon is enabled
az aks show \
  --resource-group <rg-name> \
  --name <aks-cluster-name> \
  --query addonProfiles.omsagent

# View Azure Monitor Agent DaemonSet (managed by AKS)
kubectl get ds -n kube-system | grep ama

# View Container Insights pods
kubectl get pods -n kube-system | grep ama

# Enable Azure Managed Prometheus
az aks update \
  --resource-group <rg-name> \
  --name <aks-cluster-name> \
  --enable-azure-monitor-metrics

# Query logs in Log Analytics (KQL examples)
# Run in Azure Portal → Log Analytics Workspace → Logs

# All container logs in last 1 hour
# ContainerLogV2
# | where TimeGenerated > ago(1h)
# | project TimeGenerated, PodName, ContainerName, LogMessage

# OOMKilled containers
# KubePodInventory
# | where LastKnownStatus == "OOMKilled"
# | project TimeGenerated, Name, Namespace, LastKnownStatus

# Node CPU usage over time
# Perf
# | where ObjectName == "K8SNode" and CounterName == "cpuUsageNanoCores"
# | summarize avg(CounterValue) by bin(TimeGenerated, 5m), Computer
```

---

## Debugging Observability Issues — Quick Reference

```bash
# Metrics Server not working — check API service
kubectl get apiservices v1beta1.metrics.k8s.io
kubectl describe apiservices v1beta1.metrics.k8s.io

# kubectl top returns "error: Metrics API not available"
kubectl get pods -n kube-system | grep metrics-server   # is it running?
kubectl logs -n kube-system -l k8s-app=metrics-server   # check logs

# Pod logs not appearing in log aggregator
kubectl get pods -n logging                              # is Fluent Bit running?
kubectl logs -n logging <fluent-bit-pod> | grep error   # any forwarding errors?
ls /var/log/containers/ | grep <pod-name>               # log file exists on node?

# Prometheus not scraping a target
kubectl get servicemonitor -A                           # ServiceMonitor exists?
kubectl get svc -n <namespace> | grep <app>             # Service exists and labelled?
kubectl get endpoints <service-name> -n <namespace>     # Endpoints populated?

# Check events for observability-related issues
kubectl get events -n monitoring --sort-by='.lastTimestamp'
kubectl get events -n logging --sort-by='.lastTimestamp'
```