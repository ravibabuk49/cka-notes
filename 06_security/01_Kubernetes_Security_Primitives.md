## 01 — Kubernetes Security Primitives

### What it is

Kubernetes security is layered across four distinct planes:

| Plane | What it secures |
|---|---|
| **Host / infrastructure** | The physical or virtual nodes forming the cluster |
| **API access control** | Who can reach the kube-apiserver and what they can do |
| **Component communication** | TLS-encrypted traffic between all control-plane and node components |
| **Pod networking** | East-west traffic between pods inside the cluster |

The kube-apiserver is the single entry point for all cluster operations — every `kubectl` call and direct REST request passes through it, making it the primary security boundary. Every other control decision flows from this point outward.

---

### Host Security Baseline

Before any Kubernetes-specific hardening, the underlying nodes must be secured:

| Requirement | Detail |
|---|---|
| Root access | Disabled |
| Password-based SSH | Disabled |
| Authentication method | SSH key-based only |
| Infrastructure hardening | CIS Benchmarks, OS-level firewall rules, minimal installed packages |

A compromised node gives an attacker direct access to the kubelet, pod filesystems, mounted secrets, and potentially cluster-admin credentials stored in kubeconfig files on control-plane nodes. Node security is a prerequisite, not an afterthought.

---

### API Server Access Control — Two Questions

Every access-control decision at the kube-apiserver reduces to two questions:

```
1. Who are you?      → Authentication  (identity verification)
2. What can you do?  → Authorization   (permission enforcement)
```

#### Authentication — Who Can Access

The kube-apiserver supports multiple authentication strategies, evaluated in order until one succeeds:

| Mechanism | Used for | Notes |
|---|---|---|
| X.509 client certificates | Components, admin users | Standard approach; configured via `--client-ca-file` on the apiserver |
| Static token files | Dev/test only | Plaintext tokens in a CSV file; deprecated pattern — avoid |
| Bootstrap tokens | Node join via kubeadm | Short-lived; used during cluster bootstrap only |
| Service account tokens | In-cluster workloads (pods) | JWT signed by the apiserver; auto-mounted into pods |
| OIDC tokens | Enterprise human users | Production-grade external auth (Entra ID, Okta, Dex, etc.) |
| LDAP (via webhook) | External directory integration | Delegated through an authentication webhook |

> **Kubernetes has no native `User` object.** Human users exist only through external mechanisms (certificates, OIDC). `kubectl get users` returns an error — there is nothing to list.

> **Service accounts are the Kubernetes-native identity for machine workloads.** They are namespace-scoped objects backed by signed JWTs. Covered in detail in the service accounts section of this domain.

#### Authorization — What They Can Do

After authentication succeeds, the apiserver evaluates one or more authorization modes in the order they appear in `--authorization-mode`. It allows on the **first ALLOW** and denies if all modes return DENY or NO_OPINION.

| Mode | Description |
|---|---|
| **RBAC** | Role-based access control — Roles and ClusterRoles bound to users, groups, or service accounts. The standard for production clusters. |
| **Node** | Special-purpose authorizer that allows kubelets to read/write only the resources belonging to their own node (pods scheduled on it, its node object, etc.). |
| **ABAC** | Attribute-based access control using a static policy file on disk. Requires apiserver restart to update. Largely superseded by RBAC. |
| **Webhook** | Delegates the authorization decision to an external HTTP service (e.g., OPA/Gatekeeper). |
| **AlwaysAllow** | Permits all authenticated requests. Testing only — never production. |
| **AlwaysDeny** | Denies all requests. Testing only. |

Standard production value: `--authorization-mode=Node,RBAC`

---

### When to Use Which

#### Authentication Mechanism

| Scenario | Mechanism |
|---|---|
| Control-plane components (scheduler, controller-manager, kubelet) authenticating to the apiserver | X.509 client certificates |
| Human operator access via `kubectl` in a corporate environment | OIDC (federated with enterprise IdP) |
| In-cluster workload calling the Kubernetes API | Service account token |
| Temporary local admin access (e.g., break-glass kubeconfig) | X.509 client certificate |
| Static tokens, basic auth | Do not use — deprecated and insecure |

#### Authorization Mode

| Scenario | Mode |
|---|---|
| Virtually all production clusters | `Node,RBAC` — Node authorizer for kubelets, RBAC for everything else |
| External policy engine (OPA, Styra) needed | Add `Webhook` after `Node,RBAC` |
| Legacy clusters pre-RBAC | `ABAC` — migrate to RBAC |
| Testing a fresh component without access control | `AlwaysAllow` — remove before any real use |

---

### TLS Between Cluster Components

All communication between Kubernetes components is mutually TLS-authenticated:

```
kube-apiserver  ↔  etcd
kube-apiserver  ↔  kube-controller-manager
kube-apiserver  ↔  kube-scheduler
kube-apiserver  ↔  kubelet              (on each worker node)
kube-apiserver  ↔  kube-proxy
kubelet         ↔  container runtime
```

Each component holds a certificate signed by the cluster CA, and presents it during the TLS handshake. Full certificate setup, rotation, and troubleshooting are covered in the TLS sections of this domain.

---

### Network Policies — Pod-to-Pod Communication

By default, Kubernetes applies a **flat, open network model**: every pod can reach every other pod across all namespaces on any port, with no restrictions.

**Network Policies** are the Kubernetes-native mechanism to restrict this east-west traffic. They are namespace-scoped, CNI-enforced, and covered in full in the Network Policies section of this domain.

> Without a Network Policy in place, a compromised pod can freely probe any other pod in the cluster — including those in `kube-system`. The default-open model is a deliberate design choice for flexibility, but requires explicit policies to secure.

---

### Real-World Usage

| Concern | Production approach |
|---|---|
| Human user access | OIDC integration (Entra ID, Okta, Dex) — no static tokens in kubeconfig |
| Workload identity | Service accounts with projected tokens; AKS Workload Identity / IRSA (EKS) in cloud environments |
| Least privilege | RBAC with minimal ClusterRoleBindings; prefer namespaced Roles over ClusterRoles wherever possible |
| Node hardening | CIS Kubernetes Benchmark v1.9+; AppArmor/seccomp profiles; gVisor for untrusted workloads |
| Pod network isolation | Namespace-scoped default-deny NetworkPolicy applied at namespace creation time |
| Certificate lifecycle | cert-manager with automated renewal — manual certificate management does not scale |
| Authorization audit | `kubectl auth can-i --list --as=<user>` to inspect effective permissions; audit logs for retrospective review |

---

### Exam Gotchas

- **`kubectl get users` does not work** — Kubernetes has no User resource. Human identities live outside the cluster in certificates or external IdPs.
- **Authorization mode order matters** — `--authorization-mode=RBAC,Node` and `--authorization-mode=Node,RBAC` behave differently. The apiserver checks modes left to right; `Node` must precede `RBAC` in standard clusters to ensure kubelet permissions are handled by the Node authorizer, not RBAC.
- **ABAC requires an apiserver restart** to pick up policy file changes; RBAC changes take effect immediately. This distinction has appeared in exam questions.
- **AlwaysAllow is the effective default** if `--authorization-mode` is not set — all authenticated requests succeed. Know this for misconfiguration troubleshooting questions.
- **Service accounts are namespace-scoped** — a pod in `namespace-a` cannot use a service account defined in `namespace-b`.
- **Default pod networking is fully open** — no NetworkPolicy exists by default. The exam may ask you to create a default-deny ingress or egress policy from scratch.
- **Failing authentication vs failing authorization** — authentication failure returns HTTP 401; authorization failure returns HTTP 403. Knowing which layer rejected a request is essential for troubleshooting.

---

### In Managed Clusters (AKS / EKS / GKE)

| Aspect | Managed behaviour |
|---|---|
| **Authentication** | AKS integrates with Microsoft Entra ID natively — OIDC issuer is configured by the platform; operators configure role bindings, not auth infrastructure |
| **Authorization** | RBAC is always enabled on AKS and cannot be disabled |
| **Node authorizer** | Active by default; not operator-managed |
| **TLS / certificates** | Control-plane certificates (apiserver, etcd, controller-manager) are fully managed by the cloud provider — operators never rotate these manually |
| **Network Policies** | Not enforced by default on AKS — requires Azure CNI + Azure Network Policy or Calico add-on selected **at cluster creation time**; cannot be retrofitted without cluster recreation |
| **Workload identity** | AKS Workload Identity (OIDC federation + federated credentials on managed identity) replaces legacy pod-managed identity — no token files, no secrets in pods |

