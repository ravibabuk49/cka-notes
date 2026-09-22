## 02 — Authentication

### What it is

Authentication is the process by which the kube-apiserver verifies the identity of every inbound request before any processing or authorisation occurs. It is the first of three sequential layers in the API server's request pipeline:

```
Request → [1] Authentication → [2] Authorization → [3] Admission Control → etcd
```

This section covers the identity side of that pipeline: who is making the request, and how that identity is established.

For the full list of supported authentication mechanisms and when to use each, see `01_kubernetes_security_primitives`.

---

### User Types in Kubernetes

Kubernetes recognises two categories of identity:

| Category | Examples | Managed by Kubernetes? |
|---|---|---|
| **Human users** | Cluster administrators, developers | ❌ No — always external |
| **Service accounts** | Pods, controllers, CI/CD pipelines | ✅ Yes — first-class API objects |

**Human users have no native representation in Kubernetes.** There is no `User` resource, no `kubectl create user`, and no `kubectl get users`. Identity for human users must come from an external source: an X.509 certificate, an OIDC token issued by an external IdP, or a static credential file. The kube-apiserver trusts the external source; it never stores the user itself.

**Service accounts are namespace-scoped Kubernetes objects** created and managed through the standard API. They are the correct identity mechanism for any workload or automation that needs to call the Kubernetes API. Covered in full in the Service Accounts section of this domain.

---

### All Requests Route Through the kube-apiserver

Whether a request originates from `kubectl`, a direct REST call, or an internal component, it always reaches the kube-apiserver first. The apiserver authenticates the request before doing anything else — unauthenticated requests are rejected at this boundary.

```
kubectl       ──┐
REST client   ──┼──► kube-apiserver (authenticate first) ──► process request
internal cmpt ──┘
```

---

### Authentication Mechanisms

The kube-apiserver supports several authentication strategies, configured via flags at startup. Multiple strategies can be active simultaneously — the apiserver tries each in sequence until one succeeds.

| Mechanism | Flag | Status |
|---|---|---|
| Static password file | `--basic-auth-file` | ❌ **Removed in Kubernetes 1.19** |
| Static token file | `--token-auth-file` | ⚠️ **Deprecated in Kubernetes 1.29** |
| X.509 client certificates | `--client-ca-file` | ✅ Current — standard for components |
| Bootstrap tokens | `--enable-bootstrap-token-auth` | ✅ Current — kubeadm node join |
| Service account tokens | `--service-account-key-file` | ✅ Current — in-cluster workloads |
| OIDC tokens | `--oidc-issuer-url` | ✅ Current — production human auth |
| Authenticating proxy | `--requestheader-*` | ✅ Current — LDAP via proxy |

The remainder of this section covers static files in detail (the transcript's focus). Certificate-based authentication and OIDC are covered in their own sections.

---

### Static Password File (Removed — Kubernetes 1.19)

> ⛔ `--basic-auth-file` was **removed in Kubernetes 1.19**. This mechanism no longer exists in any supported cluster. It is documented here for conceptual completeness and because older exam materials and blog posts still reference it.

**CSV format** — four columns, fourth optional:

```
password,username,uid,group
P@ssword1,alice,u001,admin
P@ssword2,bob,u002,developers
P@ssword3,carol,u003,"developers,qa"
```

**apiserver flag (non-functional on current Kubernetes):**
```
--basic-auth-file=/etc/kubernetes/auth/users.csv
```

**Request authentication (Basic Auth header):**
```bash
curl -k https://<apiserver>:6443/api/v1/pods \
  --user alice:P@ssword1
```

**Why it was removed:**
- Passwords stored in cleartext on disk
- No credential rotation — changing a password required editing the file and restarting the apiserver
- No token expiry
- apiserver restart required to pick up any file change

---

### Static Token File (Deprecated — Kubernetes 1.29)

> ⚠️ `--token-auth-file` was **deprecated in Kubernetes 1.29**. It may still function on current clusters but will be removed in a future release. Do not use in production.

Tokens replace the password column but the structure and limitations are identical to the static password file.

**CSV format:**

```
token,username,uid,group
eyJhbGciOiJIUzI....,alice,u001,admin
dGhpcyBpcyBub3Q....,bob,u002,developers
```

**apiserver flag:**
```
--token-auth-file=/etc/kubernetes/auth/tokens.csv
```

**Request authentication (Bearer token header):**
```bash
curl -k https://<apiserver>:6443/api/v1/pods \
  --header "Authorization: Bearer eyJhbGciOiJIUzI...."
```

**Why it is deprecated (same fundamental issues as static password file):**
- Tokens stored in cleartext on disk
- Tokens never expire — a leaked token is valid indefinitely until manually removed and the apiserver restarted
- Any change to the file requires an apiserver restart to take effect
- No mechanism for per-token auditing or revocation

---

### kubeadm Consideration: Volume Mounts

In a kubeadm cluster, the apiserver runs as a **static pod** defined in `/etc/kubernetes/manifests/kube-apiserver.yaml`. The token file lives on the host node, outside the container. To make it accessible to the apiserver process, it must be explicitly mounted via a `hostPath` volume in the manifest:

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml (relevant sections)
spec:
  containers:
  - name: kube-apiserver
    command:
    - kube-apiserver
    - --token-auth-file=/etc/kubernetes/auth/tokens.csv   # path inside container
    volumeMounts:
    - mountPath: /etc/kubernetes/auth
      name: auth-files
      readOnly: true
  volumes:
  - name: auth-files
    hostPath:
      path: /etc/kubernetes/auth                          # path on the host node
      type: DirectoryOrCreate
```

Modifying the static pod manifest causes the kubelet to automatically restart the apiserver pod. Remember to also configure RBAC bindings for any users defined in the file — authentication establishes identity; it grants no permissions on its own.

---

### When to Use Which

Static password and token files have no legitimate use case in any current cluster:

| Mechanism | Use? | Reason |
|---|---|---|
| Static password file | Never | Removed in Kubernetes 1.19 |
| Static token file | Never in production; lab only | Deprecated in 1.29; cleartext; no expiry; requires restart to change |
| X.509 certificates | For components and break-glass admin access | Covered in TLS sections |
| OIDC | For human users in any real environment | Covered in later sections |
| Service accounts | For all in-cluster workloads | Covered in Service Accounts section |

---

### Real-World Usage

Static credential files were an early Kubernetes authentication pattern designed for simplicity. They were replaced because they violate basic secrets hygiene:

| Requirement | Static files | Production alternative |
|---|---|---|
| Credentials not stored in cleartext | ❌ | OIDC (tokens never touch the cluster's disk), X.509 certs |
| Credential rotation without downtime | ❌ Requires apiserver restart | OIDC token TTLs; cert rotation via cert-manager |
| Token expiry | ❌ Static tokens never expire | OIDC JWTs with configurable TTL; projected service account tokens |
| Per-identity audit trail | ❌ All requests share the same file | OIDC with username claim in audit log |
| Revocation | ❌ File edit + restart | OIDC: revoke at IdP; certs: CRL or short-lived issuance |

In any real cluster today: human users authenticate via OIDC (Entra ID, Okta, Dex). Workloads use projected service account tokens. Cluster components use X.509 client certificates. Static files are a historical artefact.

---

### Exam Gotchas

- **Static password file was removed in 1.19, not just deprecated** — `--basic-auth-file` does not exist on any supported Kubernetes version. If an exam question references it, read it as historical context, not current behaviour.
- **Static token file is deprecated in 1.29** — it may still appear in older CKA lab environments; know the flag (`--token-auth-file`) and the Bearer token header syntax.
- **Authentication ≠ authorisation** — a user successfully authenticated via a token file has no permissions until an RBAC RoleBinding or ClusterRoleBinding grants them. Authentication failure returns HTTP 401; authorisation failure returns HTTP 403.
- **apiserver restart is required** for any change to a static credential file to take effect — this is a hard operational limitation and a common exam distractor.
- **kubeadm apiserver is a static pod** — the auth file must be on the host node and mounted into the container via `hostPath`. Placing the file only inside the container solves nothing; the manifest controls what is mounted.
- **Group assignment in CSV is the fourth column** — `token,username,uid,group`. Multiple groups use a quoted comma-separated value: `"group1,group2"`. The group field flows directly into RBAC group matching.
- **Service accounts are the answer for any non-human workload** — if an exam scenario involves a process, pod, or pipeline needing cluster access, the answer is a service account, not a user credential of any kind.

---

### In Managed Clusters (AKS / EKS / GKE)

Static credential files are irrelevant to managed clusters — the control plane is not operator-managed and these flags cannot be set. Authentication in managed clusters:

| Mechanism | AKS |
|---|---|
| Human users | Microsoft Entra ID via OIDC (managed by platform) |
| Workload identity | AKS Workload Identity — federated credentials on a managed identity; replaces static tokens entirely |
| kubectl auth | `az aks get-credentials` populates kubeconfig with a short-lived Entra ID token; `kubelogin` handles token refresh |
| Service accounts | Standard Kubernetes SA objects; projected tokens used for in-cluster calls |

