## 02 — Managing Application Logs

### What it is
Kubernetes surfaces container logs via the `kubectl logs` command, which reads from the standard output (stdout) and standard error (stderr) streams of a container running inside a Pod. This maps directly to how Docker surfaces logs — Kubernetes delegates log retrieval to the container runtime on the node.

---

### Single Container Pod

```bash
# View logs of a pod (single container)
kubectl logs <pod-name>

# Stream logs live (equivalent to docker logs -f)
kubectl logs <pod-name> -f

# View last N lines
kubectl logs <pod-name> --tail=50

# View logs since a duration
kubectl logs <pod-name> --since=1h
kubectl logs <pod-name> --since=5m

# View logs with timestamps
kubectl logs <pod-name> --timestamps
```

---

### Multi-Container Pod

When a Pod has more than one container, `kubectl logs` **requires the container name to be specified explicitly** — otherwise the command fails with an error asking you to specify a container.

```bash
# This FAILS on a multi-container pod
kubectl logs <pod-name>
# Error: a container name must be specified for pod <pod-name>, choose one of: [event-simulator image-processor]

# Correct — specify the container name
kubectl logs <pod-name> -c <container-name>
kubectl logs <pod-name> -c event-simulator
kubectl logs <pod-name> -c image-processor

# Stream logs from a specific container
kubectl logs <pod-name> -c <container-name> -f
```

---

### Logs for Previously Terminated Containers

If a container has crashed and restarted, `kubectl logs` shows the current container's logs by default. To see logs from the previous instance:

```bash
kubectl logs <pod-name> -c <container-name> --previous
```

---

### Logs Across All Containers in a Pod

```bash
# Print logs from all containers in a pod simultaneously
kubectl logs <pod-name> --all-containers=true

# Stream from all containers
kubectl logs <pod-name> --all-containers=true -f
```

---

### Exam Gotchas

- **Multi-container pod — always specify `-c`** — forgetting the container name on a multi-container pod causes the command to fail. Check `kubectl describe pod <pod-name>` to list container names if unsure.
- **`kubectl logs` reads stdout/stderr only** — logs written to files inside the container are not surfaced by `kubectl logs`. They require `kubectl exec` to access.
- **Logs are lost when a pod is deleted** — `kubectl logs` reads from the node's container runtime log files. Once the pod is gone, so are its logs unless a log aggregation solution (Fluentd, Elastic Stack) is in place.
- **`--previous` flag for crash debugging** — if a container is in `CrashLoopBackOff`, the current container may not have useful logs yet. Use `--previous` to read the terminated instance's logs.
- **`-f` flag streams indefinitely** — use `Ctrl+C` to exit. In the exam, avoid leaving a streaming log command running in a terminal you need for other tasks.

---

### In Managed Clusters (AKS / EKS / GKE)

- In AKS with Container Insights enabled, container logs are shipped to a **Log Analytics Workspace** and queryable via KQL — persisted beyond pod lifecycle, unlike `kubectl logs`.
- `kubectl logs` works identically on AKS — no behavioural difference from self-managed clusters.
- For log aggregation at scale on AKS, **Fluentd or Fluent Bit** runs as a DaemonSet (see `09_daemonsets.md`) to forward logs to Azure Monitor, Elastic Stack, or other backends.

---

