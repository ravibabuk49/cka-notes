## 05 — Configuring Environment Variables

### What it is

Kubernetes injects environment variables into a container at runtime via the Pod spec's `env` field, without modifying the container image. Values can be set directly as plain key-value pairs, or sourced dynamically from a `ConfigMap` or `Secret`.

---

### Direct Key-Value Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
    env:
      - name: APP_COLOR
        value: pink
      - name: APP_MODE
        value: prod
```

- `env` is an **array** — each entry is a list item (`-`).
- Each item requires `name` (the environment variable name visible inside the container) and either `value` (static) or `valueFrom` (dynamic, see below).

---

### Dynamic Values — `valueFrom`

Instead of a hardcoded `value`, use `valueFrom` to source the value from a `ConfigMap` or `Secret`:

```yaml
env:
  - name: APP_COLOR
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_COLOR
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: DB_PASSWORD
```

> Full ConfigMap and Secret mechanics (creation, mounting as volumes, etc.) are covered in their own dedicated sections — this section only covers their use as an `env` value source.

---

### When to Use Which

| Need | Use |
|---|---|
| Static, non-sensitive value (e.g., log level, app mode) | `value` |
| Shared, non-sensitive config used across multiple Pods | `valueFrom: configMapKeyRef` |
| Sensitive value (passwords, API keys, tokens) | `valueFrom: secretKeyRef` |

---

### Exam Gotchas

- `env` is a list (`-` prefixed), not a map — a common YAML indentation mistake under time pressure.
- `value` and `valueFrom` are mutually exclusive per entry — never both on the same item.
- `valueFrom.configMapKeyRef` and `valueFrom.secretKeyRef` both require **both** `name` (the ConfigMap/Secret object) and `key` (the specific key within it) — missing either causes a `CreateContainerConfigError`.
- If a referenced ConfigMap/Secret or key doesn't exist, the Pod will be stuck in `CreateContainerConfigError` — check with `kubectl describe pod`.
- Environment variables set this way are visible via `kubectl exec <pod> -- env` — useful for quick verification in the exam.

---

### Real-World Usage

- **Twelve-factor app config**: environment variables are the standard mechanism for injecting environment-specific config (dev/staging/prod) into a single, unmodified container image — this is the same pattern used in Azure App Service application settings, just expressed via Kubernetes-native fields.
- **Feature flags and mode toggles**: simple flags like `APP_MODE=debug` are commonly passed directly via `value` rather than ConfigMaps, since they're trivial and Pod-specific.
- **Centralized config sharing**: when the same configuration (e.g., a feature flag or service endpoint) needs to be consistent across many Pods/Deployments, teams use a shared `ConfigMap` referenced via `configMapKeyRef` instead of duplicating `value` entries across manifests.
- **CI/CD pipeline injection**: GitHub Actions or Azure Pipelines often template the `env` block at deploy time (e.g., via Helm `values.yaml` or Kustomize overlays) so the same base manifest produces different runtime config per environment.

---

### In Managed Clusters (AKS)

Behaviour is identical to self-managed clusters. AKS Workload Identity / Key Vault integration (CSI driver) is the recommended production pattern for sourcing genuinely sensitive secrets, layered on top of the native `secretKeyRef` mechanism — covered in the Security domain.

---

