## 02 — Labels, Selectors & Annotations

### What it is
- **Label** — a key-value pair attached to a Kubernetes object under `metadata.labels`. Used to identify and group objects.
- **Selector** — a query that filters objects by their labels. Used by both `kubectl` and controllers (ReplicaSet, Service, Deployment) to find their target objects.
- **Annotation** — also a key-value pair under `metadata.annotations`, but purely informational. Cannot be used in selectors.

---

### Why They Exist

A real cluster can have hundreds of Pods, Services, and Deployments across multiple applications and environments. Labels give you a structured way to tag every object, and selectors let you query exactly what you need.

> **Think of it like Azure resource tags** — you tag resources with `env=prod`, `app=payments`, and then filter or apply policy using those tags. Labels are the tags; selectors are the filter query.

---

### Defining Labels

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: payments-api
  labels:
    app: payments-api    # what application
    env: prod            # which environment
    tier: backend        # what function
spec:
  containers:
    - name: app
      image: payments-api:v1
```

```bash
# Filter pods by a single label
kubectl get pods -l app=payments-api

# Filter by multiple labels (AND logic — both must match)
kubectl get pods -l app=payments-api,env=prod

# Works on any object type
kubectl get all -l env=prod
```

> `--selector` and `-l` are identical flags — both are valid in the exam.

---

### The Two-Label Pattern in Controllers

This is the most commonly confused concept. In a ReplicaSet (and Deployment), labels appear in **three places**, each serving a distinct purpose:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: payments-rs
  labels:
    app: payments-api      # ① Labels ON the ReplicaSet object itself
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payments-api    # ② Which Pods this ReplicaSet owns and manages
      env: prod
  template:
    metadata:
      labels:
        app: payments-api  # ③ Labels stamped onto every Pod this RS creates
        env: prod          #    MUST match ② — API server rejects if they don't
    spec:
      containers:
        - name: app
          image: payments-api:v1
```

| # | Location | What it labels | Used by |
|---|---|---|---|
| ① | `metadata.labels` | The ReplicaSet object itself | Other objects that want to select this RS (e.g. a future HPA) |
| ② | `spec.selector.matchLabels` | Declares which Pods this RS owns | ReplicaSet controller — continuously reconciles Pod count |
| ③ | `spec.template.metadata.labels` | Every Pod the RS creates | Must satisfy ② — this is how the RS recognises its own Pods |

**How the ReplicaSet controller uses this:** It watches all Pods in the namespace matching ②. If the count drops below `replicas`, it creates new Pods using `spec.template`. If ② and ③ don't match, the RS can never recognise its own Pods — so the API server rejects it at creation.

---

### Labels in Services

A Service uses `spec.selector` to dynamically discover which Pods to route traffic to — no static IP binding:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payments-svc
spec:
  selector:
    app: payments-api    # routes traffic to all Pods with this label
    env: prod
  ports:
    - port: 80
      targetPort: 8080
```

The Service continuously watches for Pods matching its selector and keeps its **Endpoints** list updated automatically — Pods that come and go are picked up in real time.

---

### Annotations

Annotations are key-value pairs stored under `metadata.annotations`. They are **non-identifying** — the Kubernetes control plane does not use them for scheduling, selection, or ownership. Their purpose is to attach arbitrary metadata to an object for consumption by humans, external tooling, or integrations.

> **Labels answer "which objects?" — Annotations answer "what do we know about this object?"**

> **Analogy:** Think of an Azure VM's **resource tags** vs its **properties blade**. Tags (`env=prod`, `app=payments`) are how you filter and group VMs in Resource Graph — that's labels. The properties blade shows the VM's OS disk ID, NIC details, boot diagnostics URI, provisioning state — metadata that describes the resource but is never used to query it. That's annotations. External tools like Azure Monitor or Defender read those properties to do their job, just as Prometheus reads `prometheus.io/scrape` to decide whether to scrape a Pod.

#### What Annotations Are Used For

| Category | Example keys | Consumer |
|---|---|---|
| Build / release info | `build/version`, `build/commit`, `release/date` | CI/CD pipelines, audit logs |
| Ownership / contact | `owner`, `team`, `on-call-slack` | Humans, incident tooling |
| Monitoring config | `prometheus.io/scrape`, `prometheus.io/port` | Prometheus auto-discovery |
| Ingress behaviour | `nginx.ingress.kubernetes.io/rewrite-target` | NGINX Ingress controller |
| Service mesh config | `sidecar.istio.io/inject: "true"` | Istio control plane |
| GitOps metadata | `argocd.argoproj.io/sync-wave` | ArgoCD sync ordering |

#### Example — Full Annotation Block

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments-api
  labels:
    app: payments-api          # used for selection — keep it here
    env: prod
  annotations:
    # --- Build metadata ---
    build/version: "3.1.0"
    build/commit: "f4a91bc"
    build/pipeline: "https://github.com/org/repo/actions/runs/9812"

    # --- Ownership ---
    owner: "payments-team@company.com"
    on-call-slack: "#payments-oncall"

    # --- Prometheus scraping ---
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"

    # --- Ingress rewrite ---
    nginx.ingress.kubernetes.io/rewrite-target: "/"
```

#### Key Characteristics

- Values are always **strings** — even booleans and numbers must be quoted (`"true"`, `"9090"`).
- No size limit enforced by the schema, but etcd has a 1MB object size limit in practice.
- Keys follow the same `prefix/name` format as labels — prefix is optional but recommended for tooling-owned annotations (e.g. `prometheus.io/`, `nginx.ingress.kubernetes.io/`).
- Annotations are **not queryable** — `kubectl get pods -l prometheus.io/scrape=true` returns nothing. Use labels for anything you need to filter on.

#### Viewing Annotations

```bash
# See all annotations on an object
kubectl describe pod payments-api

# Get raw annotation value
kubectl get pod payments-api -o jsonpath='{.metadata.annotations}'

# Get a specific annotation
kubectl get pod payments-api \
  -o jsonpath='{.metadata.annotations.prometheus\.io/scrape}'
```

> Note the escaped dot (`\.`) in the jsonpath query — dots in annotation keys must be escaped.

---

### Exam Gotchas

- **Two label locations confusion** — `metadata.labels` is on the controller itself; `spec.template.metadata.labels` is on its Pods. Mixing them up is the #1 mistake in this topic.
- **`spec.selector` is immutable** — once a ReplicaSet or Deployment is created, you cannot change its selector without deleting and recreating the resource.
- **Selector mismatch = immediate API rejection** — `spec.selector.matchLabels` must overlap with `spec.template.metadata.labels`. Mismatch returns a validation error on `kubectl apply`.
- **Multiple selectors = AND logic only** — `app=payments,env=prod` means both must be true. There is no OR operator in equality-based selectors via `kubectl`.
- **Annotations are not selectors** — you cannot do `kubectl get pods -l prometheus.io/scrape=true`. Annotations are invisible to the selector mechanism.

---

### In Managed Clusters (AKS / EKS / GKE)

- AKS injects system labels onto nodes automatically (e.g. `kubernetes.azure.com/agentpool`, `kubernetes.io/os`) — usable in `nodeSelector` and affinity rules.
- AKS system node pools carry `kubernetes.azure.com/mode: system` — do not target these with workload selectors unless intentional.

---

