## In-Place Resizing of Pod Resources

### What it is

By default, any change to a Pod's resource requests or limits causes the existing Pod to be **deleted and recreated** with the new values. **In-place Pod vertical scaling** is a feature that allows CPU and memory resources to be updated on a running Pod without restarting or replacing it.

> This feature is **alpha in Kubernetes 1.27**, targeting **beta in 1.33**, and is **not enabled by default**. It must be explicitly enabled via a feature gate. Covered here as prerequisite context before Vertical Pod Autoscaler.

---

### Default Behaviour (Without Feature)

```
Change resource limits on Deployment
  → Old Pod deleted
  → New Pod created with updated resources
```

This is disruptive for stateful workloads — connections are dropped, in-flight requests are lost, and warm-up time is required.

---

### Enabling In-Place Resizing

Set the feature gate on the API server and Kubelet:

```
--feature-gates=InPlacePodVerticalScaling=true
```

Once enabled, Pod specs gain a new `resizePolicy` field per container.

---

### `resizePolicy` — Restart Behaviour Per Resource

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: myapp
    image: myapp:v1
    resources:
      requests:
        cpu: "500m"
        memory: "128Mi"
      limits:
        cpu: "1"
        memory: "256Mi"
    resizePolicy:
    - resourceName: cpu
      restartPolicy: NotRequired    # CPU can be updated without restarting the container
    - resourceName: memory
      restartPolicy: RestartContainer  # Memory change requires container restart
```

| `restartPolicy` value | Behaviour |
|---|---|
| `NotRequired` | Resource updated in place — container keeps running |
| `RestartContainer` | Container is restarted to apply the change |

---

### How an In-Place Resize Works

```
Update CPU request/limit on running Pod
  → Kubernetes updates cgroup settings on the node
  → Container receives new CPU allocation immediately
  → No Pod deletion, no downtime (if restartPolicy: NotRequired)
```

For memory with `RestartContainer`, only the container restarts — the Pod itself is not deleted and recreated.

---

### Limitations

- Only **CPU and memory** resources can be resized — no other resource types.
- **Pod QoS class** cannot be changed by resizing (e.g. you cannot move from Burstable to Guaranteed in place).
- **Init containers** and **ephemeral containers** cannot be resized.
- Resource requests and limits **cannot be removed** once set — only updated.
- A container's **memory limit cannot be reduced below its current usage** — the resize status stays `InProgress` until feasible.
- **Windows Pods** do not support in-place resizing.

---

### Relationship to Vertical Pod Autoscaler

In-place resizing is the **mechanism** — the ability to change resources without pod disruption. VPA is the **automation layer** that decides *when* and *how much* to resize. Without in-place resizing, VPA must evict and recreate Pods to apply new resource values, which is disruptive. When in-place resizing becomes stable, VPA can apply recommendations without downtime.

---

### Exam Gotchas

- This feature is **alpha / not enabled by default** as of Kubernetes 1.33 — do not assume it is available unless the question states it is enabled.
- The feature gate name is `InPlacePodVerticalScaling` — exact spelling matters.
- Default Kubernetes behaviour (without this feature) is always Pod delete + recreate on resource changes — this is what the exam will test unless explicitly told otherwise.
- `resizePolicy` is a **per-resource, per-container** field — CPU and memory can have different restart policies on the same container.
- Memory limit reduction below current usage will not error immediately — it will be stuck in `InProgress` resize status until memory usage drops.

