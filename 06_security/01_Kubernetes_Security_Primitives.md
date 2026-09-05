## 01 — Kubernetes Security Primitives

### What it is

Kubernetes security is layered across four distinct planes:

| Plane | What it secures |
|---|---|
| **Host / infrastructure** | Physical or virtual nodes forming the cluster |
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
| Infrastructure hardening | CIS Kubernetes Benchmark; OS-level firewall; minimal installed packages |

A compromised node gives an attacker direct access to the kubelet, pod filesystems, mounted Secrets, and potentially cluster-admin credentials stored in kubeconfig files on control-plane nodes. Node security is a prerequisite, not an afterthought.

---

### API Server Access Control — Three Layers

Every request to the kube-apiserver passes through three sequential layers:

```
Request → [1] Authentication → [2] Authorization → [3] Admission Control → etcd
```

This note covers layers 1 and 2. Admission controllers are covered in their own section later in this domain.

#### Authentication — Who Can Access

The kube-apiserver supports multiple authentication strategies, evaluated in order until one succeeds or all fail:

| Mechanism | Used for | Notes |
|---|---|---|
| X.509 client certificates | Components, admin users | Standard approach; CA configured via `--client-ca-file` on the apiserver |
| Static token files | Avoid entirely | `--token-auth-file` deprecated in Kubernetes 1.29 — plaintext tokens in a CSV file; never use |
| Bootstrap tokens | Node join via kubeadm | Short-lived; used only during cluster bootstrap |
| Service account tokens | In-cluster workloads (pods) | Projected JWTs (time-limited, audience-bound) since Kubernetes 1.22 — not long-lived Secrets |
| OIDC tokens | Enterprise human users | Production-grade external auth; integrates with Entra ID, Okta, Dex, etc. |
| Authenticating proxy / webhook | External directories (LDAP, etc.) | LDAP has no native Kubernetes support — requires an OIDC proxy (e.g., Dex backed by LDAP) or an authenticating proxy |

> **Kubernetes has no native `User` object.** Human users exist only through external mechanisms (X.509 certificates, OIDC). `kubectl get users` returns an error — there is nothing to list.

> **Service accounts** are the Kubernetes-native identity for machine workloads. They are namespace-scoped. Since Kubernetes 1.22, the auto-mounted token is a time-limited projected volume — not a long-lived Secret stored in etcd. Covered in the Service Accounts section of this domain.

#### Authorization — What They Can Do

After authentication, the apiserver evaluates authorization modes in the order specified by `--authorization-mode`. The evaluation is short-circuit:

| Result from any mode | Behaviour |
|---|---|
| **Allow** | Request is permitted immediately; remaining modes are skipped |
| **Deny** | Request is rejected immediately; remaining modes are skipped |
| **No Opinion** | Evaluation continues to the next mode |
| All modes return No Opinion | Request is denied |

| Mode | Description |
|---|---|
| **RBAC** | Role-based access control — Roles and ClusterRoles bound to users, groups, or service accounts. The standard for production clusters. |
| **Node** | Special-purpose authorizer granting kubelets read/write access only to resources belonging to their own node (scheduled pods, their node object, etc.). |
| **ABAC** | Attribute-based access control using a static policy file on disk. Requires apiserver restart to apply changes. Superseded by RBAC. |
| **Webhook** | Delegates authorization decisions to an external HTTP service via SubjectAccessReview (e.g., OPA with kube-mgmt). |
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
| In-cluster workload calling the Kubernetes API | Service account (projected token) |
| Temporary local admin access (break-glass kubeconfig) | X.509 client certificate |
| LDAP directory integration | Dex (OIDC proxy) backed by LDAP, or an authenticating webhook — never LDAP directly |
| Static tokens, basic auth | Do not use — deprecated or removed |

#### Authorization Mode

| Scenario | Mode |
|---|---|
| All production clusters | `Node,RBAC` — Node for kubelet requests, RBAC for all other subjects |
| External policy engine (OPA with kube-mgmt, Styra) required | `Node,RBAC,Webhook` — webhook evaluated after RBAC |
| Legacy clusters pre-RBAC | ABAC — migrate to RBAC at the earliest opportunity |
| Isolated test environment | AlwaysAllow — must be removed before any real use |

---

### TLS Between Cluster Components

All communication between Kubernetes control-plane and node components is mutually TLS-authenticated. Each component holds a certificate signed by the cluster CA and presents it during the TLS handshake:

```
kube-apiserver  ↔  etcd
kube-apiserver  ↔  kube-controller-manager
kube-apiserver  ↔  kube-scheduler
kube-apiserver  ↔  kubelet              (apiserver initiates; presents client cert; kubelet presents server cert)
kube-proxy      →  kube-apiserver       (kube-proxy initiates outbound watch connection)
```

> kubelet-to-container-runtime communication (CRI) uses a local Unix domain socket — not TLS. It does not appear in the cluster certificate architecture.

Full certificate architecture, file paths, flags, rotation, and troubleshooting are covered in the TLS sections later in this domain.

---

### Network Policies — Pod-to-Pod Communication

By default, Kubernetes uses a **flat, open network model**: every pod can reach every other pod across all namespaces on any port, with no restrictions.

**Network Policies** are Kubernetes-native objects that restrict this east-west traffic. They are namespace-scoped and enforced by the CNI plugin — not the apiserver. Covered in full in the Network Policies section of this domain.

> Without a Network Policy, a compromised pod can freely probe any other pod in the cluster — including those in `kube-system`. The open default is a deliberate design choice for connectivity flexibility, not a misconfiguration.

---

### Real-World Usage

| Concern | Production approach |
|---|---|
| Human user access | OIDC integration (Entra ID, Okta, Dex) — no static tokens in kubeconfig |
| Workload identity | Service accounts with projected tokens; AKS Workload Identity / IRSA (EKS) for cloud API access |
| Least privilege | RBAC with minimal ClusterRoleBindings; namespaced Roles over ClusterRoles wherever scope allows |
| Node hardening | CIS Kubernetes Benchmark (current release track); AppArmor/seccomp profiles; gVisor for untrusted workloads |
| Pod network isolation | Default-deny NetworkPolicy applied at namespace creation time |
| Certificate lifecycle | cert-manager with automated renewal — manual certificate management does not scale |
| Permission inspection | `kubectl auth can-i --list --as=<user>` for effective permissions; audit logs for retrospective review |

---

### Exam Gotchas

- **`kubectl get users` does not work** — Kubernetes has no User resource. Human identities exist only through external auth mechanisms (certificates, OIDC).
- **DENY is immediately terminal in authorization** — any single mode returning Deny stops evaluation and rejects the request. This is distinct from No Opinion, which continues the chain. "All modes must deny" is incorrect.
- **`Node,RBAC` is the Kubernetes-documented convention** — the Node authorizer is purpose-built for kubelet requests (keyed on the `system:nodes` group and `system:node:<name>` username). This is the standard order per Kubernetes documentation and the CKA exam.
- **ABAC requires an apiserver restart** to pick up policy file changes; RBAC changes take effect immediately. This distinction appears in exam questions.
- **`--authorization-mode` not set = AlwaysAllow** — if the flag is absent, all authenticated requests are permitted. Know this for misconfiguration troubleshooting.
- **Service accounts are namespace-scoped** — a pod in `namespace-a` cannot use a service account from `namespace-b`.
- **Default pod networking is fully open** — no NetworkPolicy exists by default. The exam regularly asks you to write default-deny ingress or egress policies from scratch.
- **401 vs 403** — authentication failure returns HTTP 401 (Unauthorized); authorization failure returns HTTP 403 (Forbidden). Identifying which layer rejected a request is essential for troubleshooting.
- **LDAP has no native Kubernetes authentication support** — any LDAP integration requires an OIDC proxy (Dex) or authenticating proxy in front of it. Do not confuse LDAP with OIDC in exam scenarios.
- **OPA/Gatekeeper is an admission controller, not an authorization webhook** — OPA with kube-mgmt implements the authorization webhook (SubjectAccessReview); Gatekeeper runs as a validating admission webhook. These are different layers of the request pipeline.

---

### In Managed Clusters (AKS / EKS / GKE)

| Aspect | Managed behaviour |
|---|---|
| **Authentication** | AKS integrates with Microsoft Entra ID via OIDC issuer (enabled by default); operators configure federated credentials and RBAC bindings — not the auth infrastructure |
| **Authorization** | RBAC is always enabled on AKS and cannot be disabled |
| **Node authorizer** | Active by default; not operator-managed |
| **TLS / certificates** | Control-plane certificates (apiserver, etcd, controller-manager) are fully managed by the cloud provider — operators never rotate these manually |
| **Network Policies** | Not enforced by default; requires a supported CNI (Azure CNI or Azure CNI Overlay) with Azure Network Policy or Calico engine. Can be enabled post-creation via `az aks update --network-policy` on supported CNI configurations |
| **Workload identity** | AKS Workload Identity (OIDC federation + federated credentials on a managed identity) is the current standard — replaces legacy pod-managed identity; no token files or Secrets in pods |

