## 06 — ConfigMaps

### What it is

A **ConfigMap** is a Kubernetes API object that stores non-sensitive configuration data as key-value pairs, decoupled from Pod definitions. It solves the problem of scattering environment variables across many Pod manifests — configuration is centralized in one object and referenced by name.

There are two phases:
1. **Create** the ConfigMap (imperative or declarative)
2. **Inject** it into a Pod (as environment variables, individual values, or mounted volumes)

---

### Creating ConfigMaps — Imperative

```bash
# Inline key-value pairs
kubectl create configmap app-config \
  --from-literal=APP_COLOR=blue \
  --from-literal=APP_MODE=prod

# From a file (data stored under the filename as key)
kubectl create configmap app-config --from-file=app_config.properties
```

> Multiple `--from-literal` flags can be chained for additional key-value pairs. This approach gets unwieldy past a handful of keys — prefer the declarative method for anything non-trivial.

---

### Creating ConfigMaps — Declarative

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: blue
  APP_MODE: prod
```

```bash
kubectl create -f config-map.yaml
```

> Note the structural difference from a Pod definition: ConfigMaps have **no `spec`** — configuration data lives directly under `data`.

---

### When to Use Which

| Scenario | Use |
|---|---|
| Quick, one-off testing or scripting | Imperative (`--from-literal`) |
| Few key-value pairs, ad hoc | Imperative |
| Version-controlled, repeatable, part of GitOps | Declarative (YAML manifest) |
| Importing an existing config file as-is | `--from-file` |

---

### Viewing ConfigMaps

```bash
kubectl get configmaps
kubectl describe configmap app-config    # shows data section with key-value pairs
```

Naming matters in practice — with multiple ConfigMaps per cluster (one per app, one for MySQL, one for Redis, etc.), clear naming is what lets you correctly associate the right ConfigMap with the right Pod later.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  MYSQL_DATABASE: orders_db
  MYSQL_PORT: "3306"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
data:
  REDIS_MAXMEMORY: 256mb
  REDIS_MAXMEMORY_POLICY: allkeys-lru
```

> Each ConfigMap is named after the component it configures (`mysql-config`, `redis-config`) rather than something generic like `config1` — this is what makes `envFrom`/`configMapRef` references unambiguous when a Pod spec is read months later.

---

### Injecting ConfigMaps into Pods

**As all environment variables (`envFrom`):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  containers:
  - name: webapp
    image: webapp-color
    envFrom:
      - configMapRef:
          name: app-config
```

- Think of it like this: `envFrom` says "take **everything** in this ConfigMap and turn it into environment variables." If `app-config` has 5 keys, you get 5 environment variables in the container — automatically, with no need to list them one by one.
- `envFrom` is a **list**, so you can point it at more than one ConfigMap if needed — each one contributes its keys as env vars.
- This is different from the `env` + `valueFrom.configMapKeyRef` approach in the previous section, where you pick out **just one** key from the ConfigMap by name. Use `envFrom` when you want all the keys; use `env` + `valueFrom` when you only want one or two specific ones.

**As a single environment variable (covered in previous section, repeated here for contrast):**

```yaml
env:
  - name: APP_COLOR
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_COLOR
```

**As a mounted volume (files in the container filesystem):**

> Full volume-mount mechanics are covered in the Storage domain — the existence of this option is noted here for completeness, since the lecture references it as a third injection method.

---

### When to Use Which (Injection Method)

| Need | Use |
|---|---|
| Inject every key in the ConfigMap as env vars | `envFrom` + `configMapRef` |
| Inject just one specific key as an env var | `env` + `valueFrom.configMapKeyRef` |
| Application reads config from a file path, not env vars | Volume mount (Storage domain) |

---

### Exam Gotchas

- ConfigMap manifests have **no `spec`** field — data goes directly under `data`. Writing `spec.data` is a common mistake under time pressure.
- `envFrom` injects **all** keys from the ConfigMap — there's no way to selectively exclude a key using `envFrom` alone; use individual `env`/`valueFrom` entries if you need only some keys.
- `envFrom` is a list (note the `-` before `configMapRef`) — you can reference multiple ConfigMaps in one container this way.
- If the referenced ConfigMap doesn't exist, the Pod fails with `CreateContainerConfigError` — same failure mode as a missing Secret reference.
- ConfigMap key names become environment variable names verbatim — invalid env var characters in keys (rare, but possible from `--from-file`) can cause issues.

---

### Real-World Usage

- **Centralizing shared config across microservices**: a `common-config` ConfigMap holding things like `LOG_LEVEL` or `FEATURE_FLAGS` is referenced via `envFrom` by multiple Deployments, avoiding duplication.
- **GitOps-managed configuration**: ConfigMap YAML manifests are committed to Git alongside Deployment manifests; ArgoCD syncs both together, so a config change and a code change can be reviewed and deployed through the same pipeline.
- **Importing existing config files**: `--from-file` is commonly used to load an existing `nginx.conf`, `application.properties`, or `.env` file directly into a ConfigMap without manually re-typing key-value pairs.
- **ConfigMap reloading caveat**: changing a ConfigMap does **not** automatically restart Pods using it via `env`/`envFrom` — environment variables are only read at container start. Tools like Reloader (Stakater) or a manual rollout restart are used in production to pick up changes.

---

### In Managed Clusters (AKS)

Behaviour is identical to self-managed clusters. AKS Azure Policy add-on can enforce naming conventions or restrict ConfigMap usage; otherwise no managed-cluster-specific differences apply.

---

