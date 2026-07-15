## Init and Sidecar Containers

> This section is a technical deep-dive. For the conceptual overview of multi-container patterns, see `08_multi_container_pods.md`.

---

### What it is

**Init containers** are specialised containers that run to completion before any app container in the Pod starts. **Sidecar containers** are containers that start before the main app and continue running alongside it for the full Pod lifetime. Both are declared under `initContainers` — what separates them is `restartPolicy`.

---

### Container `restartPolicy` Mechanics

`restartPolicy` is a **Pod-level field** but Kubernetes applies it at the **individual container level**. When a single container exits, only that container is restarted in place — other containers in the same Pod are unaffected and keep running.

| `restartPolicy` | Behaviour |
|---|---|
| `Always` (default) | Restarts after any exit, regardless of exit code |
| `OnFailure` | Restarts only on non-zero exit code |
| `Never` | Never restarts |

This policy applies to **app containers** and **regular init containers**.

**Native sidecar exception:** an `initContainers` entry with its own `restartPolicy: Always` always restarts — overriding the Pod-level policy. Even if the Pod is set to `restartPolicy: Never`, the sidecar restarts independently.

> Kubernetes does **not** restart the entire Pod when one container fails. The Pod is only recreated by its controller (ReplicaSet, Deployment, etc.) when the node dies or the Pod object is deleted.

---

### Init Containers — Deep Dive

#### Execution Rules

- Run **sequentially** in the order listed under `initContainers` — not in parallel.
- Each must exit with code `0` before the next starts.
- All init containers must complete successfully before **any** container in `containers` starts.
- If an init container fails, Kubernetes restarts **all init containers from the beginning** (subject to Pod `restartPolicy`) — not from the failed one.
- Init containers do not support `livenessProbe`, `readinessProbe`, or `lifecycle` hooks — only `startupProbe`.

#### Example — Sequential DNS readiness checks

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  initContainers:
  - name: init-myservice
    image: busybox:1.31
    command: ["sh", "-c", "until nslookup myservice; do echo waiting for myservice; sleep 2; done"]
  - name: init-mydb
    image: busybox:1.31
    command: ["sh", "-c", "until nslookup mydb; do echo waiting for mydb; sleep 2; done"]
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ["sh", "-c", "echo The app is running! && sleep 3600"]
```

Execution order: `init-myservice` runs until DNS resolves → exits → `init-mydb` runs until DNS resolves → exits → `myapp-container` starts.

---

### Native Sidecar Containers — Deep Dive

Native sidecar support is **stable from Kubernetes 1.33**. Sidecars are declared in `initContainers` with `restartPolicy: Always` — this single field is what tells Kubernetes to treat the container as a sidecar rather than a regular init container.

#### How Kubernetes Handles Native Sidecars

- Starts **before** main containers (like an init container).
- Kubernetes waits for the sidecar's `startupProbe` or `readinessProbe` to pass before starting main containers — ensuring the sidecar is actually ready, not just started.
- Runs **alongside** main containers for the full Pod lifetime.
- Shuts down **after** main containers complete — allowing it to capture termination logs or flush buffers.
- Restarts independently if it crashes, regardless of Pod-level `restartPolicy`.

#### Example — Log shipping sidecar

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-example
spec:
  initContainers:
  - name: sidecar-logger
    image: busybox:1.31
    restartPolicy: Always        # ← makes this a sidecar, not an init container
    command: ["sh", "-c", "while true; do echo Sidecar running; sleep 10; done"]
  containers:
  - name: main-app
    image: busybox:1.31
    command: ["sh", "-c", "echo Main app starting; sleep 60"]
```

#### Sidecar vs Regular Init Container — the only difference

```yaml
# Regular init container — runs and exits before main app starts
initContainers:
- name: wait-for-db
  image: busybox
  command: [...]
  # no restartPolicy field

# Native sidecar — starts first, keeps running alongside main app
initContainers:
- name: log-shipper
  image: filebeat
  restartPolicy: Always          # ← this one field is the entire distinction
  command: [...]
```

---

### When to Use Which

| Need | Use |
|---|---|
| Pre-flight task — DB ready, DNS ready, config fetched | Init container (no `restartPolicy`) |
| Helper that must be running before main app starts AND stay running | Native sidecar (`restartPolicy: Always`) |
| Log shipping, proxy, service mesh agent | Native sidecar |
| One-time secret fetch or volume seeding | Init container |
| Startup order doesn't matter, both run full lifecycle | Co-located in `containers` array (see `08_multi_container_pods.md`) |

---

### Exam Gotchas

- Init containers restart **from the beginning of the list** on failure — not from the failed container. If `init-mydb` (second in list) fails, `init-myservice` runs again first.
- Init containers do **not** support `livenessProbe` or `readinessProbe` — only `startupProbe`. Specifying either on an init container will error.
- `restartPolicy: Always` on an `initContainers` entry is the **only** field that makes it a native sidecar — everything else (field placement, image, command) is identical to a regular init container.
- Native sidecar `restartPolicy: Always` overrides the Pod-level `restartPolicy` — the sidecar restarts even if Pod policy is `Never`.
- Kubernetes waits for the native sidecar's readiness probe before starting main containers — if the sidecar never becomes ready, main containers never start. A misconfigured sidecar probe will block the entire Pod.
- Native sidecar feature is **stable from Kubernetes 1.33** — on older clusters, the workaround was placing the sidecar in the `containers` array with no startup order guarantee.
- `kubectl describe pod` will show init containers and sidecar containers in the `Init Containers` section — not under `Containers`. Check there when debugging startup issues.

---

### Real-World Usage

- **Istio service mesh**: prior to native sidecar support, Istio injected the Envoy proxy into `containers` with no startup order guarantee — this caused race conditions where app traffic was routed before the proxy was ready. Native sidecar support (1.33) resolves this by ensuring Envoy is ready before the app container starts.
- **Filebeat / Fluentd log shipping**: declared as a native sidecar sharing an `emptyDir` log volume with the main app. Starts before the app (captures startup logs), ends after (captures termination logs) — exactly what the EFK stack requires.
- **Helm chart init containers**: almost every production Helm chart includes at least one init container for dependency readiness — checking that a database, message queue, or downstream API is reachable before the app Pod becomes active.
- **Certificate injection**: an init container fetches a TLS certificate from Vault or Key Vault, writes it to a shared `emptyDir` volume, exits — the main app reads the cert at startup without needing Vault SDK integration in the application image.

---

### In Managed Clusters (AKS)

Native sidecar support (`restartPolicy: Always` on `initContainers`) is stable from Kubernetes 1.33. AKS supports this on current node pool versions — check your node pool Kubernetes version before using native sidecar syntax in production manifests. On older AKS node pools, fall back to placing the sidecar in the `containers` array.

