## 07 — Secrets

### What it is

A **Secret** is a Kubernetes API object for storing sensitive data — passwords, tokens, keys — separately from Pod definitions and application code. Structurally identical to a ConfigMap, except values are **base64-encoded** at rest (not encrypted by default) and Kubernetes treats them with additional access controls.

There are two phases:
1. **Create** the Secret (imperative or declarative)
2. **Inject** it into a Pod (as environment variables or mounted volume files)

---

### ConfigMap vs Secret — When to Use Which

| Data type | Use |
|---|---|
| Non-sensitive config (log level, app mode, hostnames) | ConfigMap |
| Sensitive data (passwords, API keys, tokens, certificates) | Secret |

> A database hostname → ConfigMap. A database password → Secret. Never store passwords in a ConfigMap.

---

### Creating Secrets — Imperative

```bash
# Inline key-value pairs — Kubernetes base64-encodes the values automatically
kubectl create secret generic app-secret \
  --from-literal=DB_HOST=mysql \
  --from-literal=DB_USER=root \
  --from-literal=DB_PASSWORD=paswrd

# From a file
kubectl create secret generic app-secret --from-file=app_secret.properties
```

> With the imperative method, you pass plain text — Kubernetes handles the base64 encoding for you.

---

### Creating Secrets — Declarative

With the declarative approach, **you must base64-encode the values yourself** before putting them in the manifest — Kubernetes does not encode them for you here.

**Encode values on Linux:**
```bash
echo -n 'mysql' | base64      # bXlzcWw=
echo -n 'root' | base64       # cm9vdA==
echo -n 'paswrd' | base64     # cGFzd3Jk
```

**Secret manifest with pre-encoded values:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  DB_HOST: bXlzcWw=
  DB_USER: cm9vdA==
  DB_PASSWORD: cGFzd3Jk
```

```bash
kubectl create -f secret.yaml
```

> ⚠️ base64 is **encoding, not encryption**. Anyone who can read the Secret object can decode the values trivially. It is not a security mechanism — it is just the required storage format.

---

### Viewing Secrets

```bash
# List secrets
kubectl get secrets

# Describe — shows keys but hides values
kubectl describe secret app-secret

# View encoded values in YAML
kubectl get secret app-secret -o yaml
```

**Decode a value:**
```bash
echo -n 'cGFzd3Jk' | base64 --decode    # paswrd
```

---

### Injecting Secrets into Pods

**As all environment variables (`envFrom`):**

```yaml
spec:
  containers:
  - name: webapp
    image: webapp
    envFrom:
      - secretRef:
          name: app-secret
```

Every key in the Secret becomes an environment variable in the container — same "grab everything" behaviour as `envFrom` with ConfigMaps.

**As a single environment variable:**

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-secret
        key: DB_PASSWORD
```

**As a mounted volume (one file per key):**

```yaml
volumes:
- name: app-secret-volume
  secret:
    secretName: app-secret
spec:
  containers:
  - name: webapp
    volumeMounts:
    - name: app-secret-volume
      mountPath: /etc/secrets
      readOnly: true
```

Each key in the Secret becomes a **file** inside `/etc/secrets/`. The file's content is the decoded secret value:
```
/etc/secrets/DB_HOST      → mysql
/etc/secrets/DB_USER      → root
/etc/secrets/DB_PASSWORD  → paswrd
```

This is the preferred injection method for applications that read config from the filesystem (e.g., most database drivers, TLS certificate consumers) rather than from environment variables.

---

### When to Use Which (Injection Method)

| Need | Use |
|---|---|
| Inject all secret keys as env vars | `envFrom` + `secretRef` |
| Inject one specific key as an env var | `env` + `valueFrom.secretKeyRef` |
| App reads secrets from the filesystem | Volume mount |
| TLS certificates, kubeconfig, SSH keys | Volume mount (binary-safe, no env var size limits) |

---

### Exam Gotchas

- **Declarative = you encode; Imperative = Kubernetes encodes.** Forgetting to base64-encode values in a declarative manifest is the most common mistake with Secrets.
- **base64 is not encryption.** The exam won't ask you to "encrypt" a Secret value — just encode it. But expect questions on how to encode/decode.
- **`secretRef` vs `secretKeyRef`** — `envFrom` uses `secretRef` (no `key` field); `env`/`valueFrom` uses `secretKeyRef` (requires both `name` and `key`). Mixing these up causes `CreateContainerConfigError`.
- **`describe secret` hides values** — use `kubectl get secret -o yaml` to see the encoded values, then decode manually with `base64 --decode`.
- **Volume-mounted secrets are updated automatically** when the Secret object changes (with a short sync delay) — environment variable secrets are NOT updated until the Pod restarts. This is a meaningful operational difference.
- A Pod referencing a non-existent Secret will be stuck in `CreateContainerConfigError` — same failure mode as a missing ConfigMap reference.

---

### Real-World Usage

- **Never commit Secret manifests to Git with real values.** In GitOps pipelines, Secrets are managed via tools like **Sealed Secrets** (Bitnami), **External Secrets Operator**, or **Azure Key Vault CSI Driver** (AKS) — the manifest in Git contains a reference or encrypted blob, not a raw base64 value.
- **AKS + Key Vault CSI Driver**: the production pattern in Azure is to store secrets in Azure Key Vault and sync them into the cluster as Kubernetes Secrets or directly as volume mounts via the Secrets Store CSI Driver — eliminating the need to manage base64-encoded manifests entirely.
- **TLS certificates as Secrets**: Ingress controllers (NGINX, Azure Application Gateway Ingress) consume TLS certificates stored as `kubernetes.io/tls` typed Secrets — cert-manager automates their creation and renewal.
- **ImagePullSecrets**: container registry credentials (e.g., ACR pull credentials) are stored as `kubernetes.io/dockerconfigjson` typed Secrets and referenced in Pod specs via `imagePullSecrets` — this is how AKS authenticates to ACR when not using managed identity.

---

### In Managed Clusters (AKS)

- By default, Secrets in AKS (and all Kubernetes clusters) are stored **unencrypted** in etcd — only base64-encoded. Enable **etcd encryption at rest** via Azure Disk Encryption or AKS encryption policy for true at-rest security.
- The recommended AKS-native pattern is **Azure Key Vault + Secrets Store CSI Driver** — Secrets live in Key Vault (RBAC-controlled, audited, rotatable) and are mounted into Pods without ever being stored as Kubernetes Secret objects.
- AKS Workload Identity integrates with the CSI Driver so Pods authenticate to Key Vault using their managed identity — no credentials needed in the cluster at all.


