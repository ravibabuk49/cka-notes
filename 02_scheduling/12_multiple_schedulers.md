## 12 — Multiple Schedulers

### What it is
Kubernetes allows you to run **multiple schedulers simultaneously** in a cluster. The default `kube-scheduler` handles all Pods unless a Pod explicitly requests a specific scheduler by name. Custom schedulers can implement their own placement logic — additional checks, business rules, or algorithms — beyond what built-in mechanisms (taints, affinity, resource requests) can express.

---

### Why Multiple Schedulers

The default scheduler covers the vast majority of scheduling needs. A custom scheduler is only needed when:

- You have placement requirements that cannot be expressed through taints, tolerations, affinity, or resource requests
- You need custom pre-scheduling validation (e.g. check an external system before placing a Pod)
- You are building a platform that extends Kubernetes scheduling for a specific workload type

---

### Real-World Use Cases

#### 1. ML / AI Workload Scheduling
GPU nodes are expensive and limited. The default scheduler has no awareness of GPU topology — it can place two Pods on the same GPU node when spreading them across two different nodes would double throughput. A custom scheduler queries GPU topology via NVIDIA's device plugin before placement.

```
Default scheduler → places Pods on any node with available GPU label
Custom scheduler  → checks GPU interconnect topology (NVLink), co-locates
                    Pods that need fast GPU-to-GPU communication on the same node
```

#### 2. Data Locality Scheduling
Big data workloads (Spark, Flink, Hadoop) perform significantly better when compute Pods are placed on the same node or rack as the data they process. The default scheduler has no awareness of where data blocks live in HDFS or an object store.

```
Default scheduler → places Pods on least-loaded node
Custom scheduler  → queries HDFS NameNode for block locations,
                    places Pod on node that already holds the data blocks
```

#### 3. Licence-Aware Scheduling
Some enterprise software (MATLAB, ANSYS, Oracle DB) requires a floating licence from a licence server. If more Pods are scheduled than available licences, jobs fail at runtime. A custom scheduler checks the licence server before placing a Pod.

```
Default scheduler → places Pod as long as CPU/memory is available
Custom scheduler  → calls licence server API, only places Pod if a
                    licence slot is currently available
```

#### 4. Compliance and Data Residency
Financial or healthcare workloads may have regulatory requirements that certain Pods only run in specific physical data centre zones or on hardware certified to a specific standard. While Node Affinity can partially address this, a custom scheduler can enforce it with an audit trail and reject non-compliant placements with a clear error.

#### 5. Co-scheduling / Gang Scheduling
Some distributed workloads (MPI jobs, distributed training) require **all Pods to start simultaneously** — starting partial sets wastes resources and causes deadlocks. The default scheduler places Pods one at a time. A gang scheduler (e.g. Volcano, YuniKorn) holds all Pods until the full set can be placed atomically.

```
Default scheduler → places Pod A on node01, Pod B waits → deadlock
Gang scheduler    → waits until node01 + node02 both have capacity,
                    then places Pod A and Pod B simultaneously
```

> **Real tools:** [Volcano](https://volcano.sh) and [Apache YuniKorn](https://yunikorn.apache.org) are production-grade custom schedulers widely used for batch and ML workloads on Kubernetes. They deploy exactly as described in this section — as a Deployment in `kube-system` alongside the default scheduler.

---

### How Schedulers Are Named

Every scheduler must have a unique name, defined in its configuration file:

```yaml
# kube-scheduler default config (name is implicit if omitted)
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
```

```yaml
# Custom scheduler config
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: my-custom-scheduler
```

---

### Deploying a Custom Scheduler as a Pod

The most common deployment method in kubeadm clusters — the scheduler runs as a Pod in `kube-system`.

**Step 1 — Create a ConfigMap for the scheduler config:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-scheduler-config
  namespace: kube-system
data:
  my-scheduler-config.yaml: |
    apiVersion: kubescheduler.config.k8s.io/v1
    kind: KubeSchedulerConfiguration
    profiles:
      - schedulerName: my-custom-scheduler
    leaderElection:
      leaderElect: false
```

**Step 2 — Deploy the scheduler as a Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-custom-scheduler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      component: my-custom-scheduler
  template:
    metadata:
      labels:
        component: my-custom-scheduler
    spec:
      serviceAccountName: my-scheduler-sa     # needs RBAC permissions
      containers:
        - name: kube-second-scheduler
          image: registry.k8s.io/kube-scheduler:v1.30.0
          command:
            - /usr/local/bin/kube-scheduler
            - --config=/etc/kubernetes/my-scheduler-config.yaml
          volumeMounts:
            - name: config-vol
              mountPath: /etc/kubernetes
      volumes:
        - name: config-vol
          configMap:
            name: my-scheduler-config
```

> The scheduler config is passed in via a **ConfigMap mounted as a volume** — this is the standard pattern in kubeadm clusters rather than baking the config into the image.

---

### leaderElect Option

The `leaderElection` setting is relevant when running **multiple replicas of the same scheduler** in a high-availability setup across multiple control plane nodes:

| Scenario | `leaderElect` setting |
|---|---|
| Single scheduler replica | `false` — no election needed |
| Multiple replicas (HA control plane) | `true` — only one replica is active at a time, others are on standby |

```yaml
leaderElection:
  leaderElect: true          # enables leader election
  resourceName: my-custom-scheduler   # distinguishes this scheduler's lock from others
```

> Without a unique `resourceName`, multiple custom schedulers competing for leader election can interfere with each other.

---

### Assigning a Pod to a Custom Scheduler

Add `schedulerName` to the Pod spec:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-scheduled-pod
spec:
  schedulerName: my-custom-scheduler    # must match the scheduler's configured name exactly
  containers:
    - name: app
      image: nginx
```

- If the named scheduler is not running or misconfigured, the Pod stays in **`Pending`** state indefinitely.
- If `schedulerName` is omitted, the Pod is handled by `default-scheduler`.

---

### Verifying Which Scheduler Placed a Pod

```bash
# View scheduling events — shows which scheduler picked up the Pod
kubectl get events -o wide
# Look for: Reason=Scheduled, Source=my-custom-scheduler

# View scheduler Pod logs directly
kubectl logs -n kube-system <scheduler-pod-name>

# Check if custom scheduler Pod is running
kubectl get pods -n kube-system | grep scheduler

# Confirm schedulerName on a Pod
kubectl get pod custom-scheduled-pod -o jsonpath='{.spec.schedulerName}'
```

---

### RBAC Requirements

A custom scheduler needs permissions to watch nodes, Pods, and create binding objects. The minimum required:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-scheduler-sa
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: my-scheduler-crb
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-scheduler      # reuse the built-in scheduler ClusterRole
subjects:
  - kind: ServiceAccount
    name: my-scheduler-sa
    namespace: kube-system
```

---

### Exam Gotchas

- **Pod stuck in `Pending`** with a `schedulerName` set — the custom scheduler is either not running, misconfigured, or the name in the Pod spec does not exactly match the name in the scheduler config.
- **`schedulerName` is case-sensitive** — `my-custom-scheduler` and `My-Custom-Scheduler` are treated as different schedulers.
- **`leaderElect: false` for single replica** — forgetting to set this causes the scheduler to wait indefinitely for a leader election that will never complete when only one replica exists.
- **ConfigMap name mismatch** — if the volume references a ConfigMap that does not exist or is in the wrong namespace, the scheduler Pod will fail with a `MountVolume` error.
- **Verify with `kubectl get events -o wide`**, not just `kubectl describe pod` — the events output shows the `Source` field which directly tells you which scheduler handled the Pod.
- **Custom schedulers run in `kube-system`** — always check this namespace when debugging scheduler issues.

---

### In Managed Clusters (AKS / EKS / GKE)

- In AKS, the `kube-scheduler` runs on managed control plane nodes — you cannot access or modify it directly.
- Deploying a **custom scheduler as a Deployment** in `kube-system` is fully supported on AKS worker nodes — the custom scheduler runs alongside the managed default scheduler.
- For most AKS use cases, the **Kubernetes Scheduling Framework** (webhook-based scheduler extensions) is preferred over a full custom scheduler — it allows you to inject custom logic as plugins without deploying an entirely separate scheduler binary.

---

