## Multi-Container Pods

### What it is

A **multi-container Pod** runs more than one container in a single Pod. All containers in the Pod share the same network namespace (communicate via `localhost`), the same storage volumes, and the same lifecycle (created and destroyed together). This allows tightly coupled services to be developed and deployed independently while operating together at runtime — without needing inter-Pod networking or volume sharing mechanisms.

---

### Why Multi-Container Pods Exist

Microservices architecture favours small, independently deployable services. But some services are inherently coupled at runtime — a web server that must run alongside every instance of an app, or a log shipper that must capture every log line the app produces. Merging them into a single image defeats the purpose of separation; running them in separate Pods introduces unnecessary networking overhead and lifecycle complexity.

Multi-container Pods solve this by co-locating containers that need to:
- Communicate on `localhost` without a Service
- Share a volume without PersistentVolumeClaim sharing
- Scale together — one Pod unit, not two independently managed Pods

---

### The Three Multi-Container Patterns

| Pattern | Startup order | Lifecycle | Typical use case |
|---|---|---|---|
| **Co-located (ambient)** | No guarantee — both start together | Both run for the full Pod lifetime | Two services equally dependent on each other |
| **Init Container** | Init runs first, exits, then main starts | Init exits before main starts | Pre-flight checks, DB readiness, config seeding |
| **Sidecar Container** | Sidecar starts first, then main starts | Both run for full Pod lifetime; sidecar ends after main | Log shipping, proxies, service mesh agents |

---

### Pattern 1 — Co-located Containers

Both containers are defined as elements in the `containers` array. No startup order is guaranteed — both start concurrently.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
  - name: web-app
    image: nginx
  - name: main-app
    image: main-app:v1
```

Use when: the two services have no startup dependency on each other and just need shared network/storage.

---

### Pattern 2 — Init Containers

Defined under a separate `initContainers` field (not inside `containers`). Init containers run **sequentially** in the order listed, each must exit successfully before the next starts, and all must complete before any container in `containers` starts.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z db-service 3306; do sleep 2; done']
  - name: api-checker
    image: busybox
    command: ['sh', '-c', 'until curl -s http://api-service/health; do sleep 2; done']
  containers:
  - name: main-app
    image: main-app:v1
```

Execution order: `wait-for-db` runs and exits → `api-checker` runs and exits → `main-app` starts.

> If any init container fails, Kubernetes restarts the Pod (subject to `restartPolicy`) — the main container never starts until all init containers complete successfully.

---

### Pattern 3 — Sidecar Containers

Sidecar containers use the `initContainers` field but with `restartPolicy: Always` — this makes them start before the main container (like an init container) but **continue running** throughout the Pod lifecycle (unlike a regular init container).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  initContainers:
  - name: filebeat
    image: elastic/filebeat:8.10.0
    restartPolicy: Always       # ← this is what makes it a sidecar, not an init container
    volumeMounts:
    - name: app-logs
      mountPath: /var/log/app
  containers:
  - name: main-app
    image: main-app:v1
    volumeMounts:
    - name: app-logs
      mountPath: /var/log/app
  volumes:
  - name: app-logs
    emptyDir: {}
```

**Sidecar lifecycle:**
- Starts before `main-app` → captures startup logs
- Runs alongside `main-app` → ships live logs to Elasticsearch
- Ends after `main-app` stops → captures termination logs

> `restartPolicy: Always` on an `initContainers` entry is the Kubernetes-native sidecar pattern introduced in Kubernetes 1.29. Prior to this, sidecars were implemented by simply adding them to the `containers` array and accepting the lack of startup order guarantee.

---

### When to Use Which

| Need | Pattern |
|---|---|
| Two services with no startup dependency | Co-located (`containers` array) |
| Pre-flight task that must complete before app starts | Init Container |
| Helper that must start before app AND run alongside it | Sidecar (`initContainers` + `restartPolicy: Always`) |
| Log shipping, proxies, service mesh agents (Envoy/Istio) | Sidecar |
| Database readiness checks, config seeding | Init Container |

---

### Shared Resources Between Containers

All containers in a Pod automatically share:

```
Network:   localhost — containers communicate on 127.0.0.1:<port>
Volumes:   any volume defined in spec.volumes is mountable by any container
Lifecycle: created together, destroyed together
```

No Service, no PVC sharing, no inter-Pod networking needed for intra-Pod communication.

---

### Exam Gotchas

- `initContainers` is a **separate field** from `containers` — do not add init containers inside the `containers` array.
- Init containers run **sequentially** in list order — not in parallel. All must exit with code 0 before the main container starts.
- If an init container fails, the Pod restarts from the first init container — not from the failed one.
- A sidecar is distinguished from an init container by `restartPolicy: Always` on the `initContainers` entry — without it, the container behaves as a regular init container and exits.
- Co-located containers and sidecars both run for the full Pod lifetime — the key difference is startup order guarantee, not duration.
- All containers in a Pod must be healthy for the Pod to be considered `Running` — a crashing sidecar will bring down the Pod.

---

### Real-World Usage

- **Service mesh (Istio/Linkerd)**: Envoy proxy is injected as a sidecar container into every application Pod automatically by a mutating admission webhook — the app container never changes, but all traffic is transparently intercepted and managed by the proxy sidecar.
- **Log aggregation (EFK stack)**: Filebeat or Fluentd sidecars collect logs from a shared `emptyDir` volume that the main app writes to, and ship them to Elasticsearch — without requiring the app to have any logging SDK.
- **AKS + Azure Monitor**: Azure Monitor Agent is deployed as a DaemonSet, not a sidecar — but the pattern of co-located log collection is the same concept, just implemented cluster-wide rather than per-Pod.
- **Database readiness init containers**: A common init container pattern in Helm charts waits for a dependent database Service to be reachable before the app starts — preventing `CrashLoopBackOff` storms during cluster cold starts or rolling restarts.
- **Config seeding init containers**: An init container clones a Git repo or fetches a config file from an API and writes it to a shared `emptyDir` volume — the main container reads it at startup without needing that logic in its own image.

---

### In Managed Clusters (AKS)

Behaviour is identical to self-managed clusters. AKS uses the sidecar pattern internally — for example, the OMS Agent for Azure Monitor and the Secrets Store CSI Driver both operate via DaemonSet Pods with multiple containers. The native sidecar feature (`restartPolicy: Always` on `initContainers`) requires Kubernetes 1.29+ — AKS supports this on current node pool versions.

