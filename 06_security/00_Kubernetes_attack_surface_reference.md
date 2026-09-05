## Kubernetes Attack Surface: 10 Frequently Exploited Security Flaws

> **Supplementary reference — not CKA curriculum.**
> Included to ground defensive configuration in real attack mechanics.
> Each flaw maps back to a section in this domain. Taxonomy: MITRE ATT&CK for Containers.

---

### About the Taxonomy: MITRE ATT&CK for Containers

**MITRE ATT&CK** (Adversarial Tactics, Techniques, and Common Knowledge) is a publicly maintained knowledge base of real-world adversary behaviour, published by the non-profit MITRE Corporation. It is the industry-standard reference for describing, categorising, and communicating how attacks actually work — used by red teams, defenders, and security vendors globally.

The framework is organised as a two-level hierarchy:

| Level | What it describes | Example |
|---|---|---|
| **Tactic** | The adversary's goal at that stage of the attack | *Privilege Escalation* — gaining higher permissions |
| **Technique** | The specific method used to achieve that goal | *T1611: Escape to Host* — breaking out of a container to the node |

**ATT&CK for Containers** is the container-specific matrix within the broader Enterprise ATT&CK framework. It maps techniques observed in real attacks against Kubernetes clusters, Docker environments, and container registries — covering the full attack lifecycle from initial access through persistence, lateral movement, and impact.

Each flaw entry below references the most relevant tactic and technique ID, so the attack pattern can be looked up in the full MITRE matrix at `https://attack.mitre.org/matrices/enterprise/containers/` for further detail, known threat-actor usage, and detection guidance.

---

### Quick-Reference Summary

| # | Flaw | MITRE Tactic | Safe by Default? | Primary Control |
|---|---|---|---|---|
| 01 | Unauthenticated kube-apiserver | Initial Access | ❌ upstream default allows anonymous | `--anonymous-auth=false` + private endpoint |
| 02 | Overpermissive RBAC | Privilege Escalation | ⚠️ RBAC on, but operators misconfigure | Least-privilege Roles; avoid cluster-admin |
| 03 | Privileged / host-namespace pods | Privilege Escalation | ✅ not privileged by default | Pod Security Admission enforce: restricted |
| 04 | Unprotected etcd | Credential Access | ⚠️ kubeadm secures; manual installs often do not | mTLS + `--client-cert-auth=true` + bind localhost |
| 05 | Service account token abuse | Credential Access | ❌ auto-mounted by default | `automountServiceAccountToken: false` + minimal RBAC |
| 06 | Unsigned / unscanned images | Initial Access (Supply Chain) | ❌ no signature verification by default | Cosign + policy admission controller |
| 07 | Missing Network Policies | Lateral Movement | ❌ flat open network by default | Default-deny NetworkPolicy per namespace |
| 08 | Plaintext Secrets | Credential Access | ❌ no encryption at rest by default | EncryptionConfiguration + external secrets store |
| 09 | Unauthenticated kubelet API | Execution | ⚠️ kubeadm secures; legacy installs may not | `--anonymous-auth=false` + `--authorization-mode=Webhook` |
| 10 | Malicious admission webhook | Persistence | ✅ requires privileged RBAC first | Restrict webhook config RBAC; scope selectors |

---

### 01 — Unauthenticated kube-apiserver Exposure

**MITRE ATT&CK:** Initial Access — Exploit Public-Facing Application (T1190)

**How it's exploited**
The upstream Kubernetes apiserver defaults to `--anonymous-auth=true`, allowing unauthenticated requests under the `system:anonymous` identity. If the authorization mode is `AlwaysAllow` (the default when `--authorization-mode` is not explicitly set), or if a ClusterRoleBinding grants permissions to `system:unauthenticated`, an attacker reaching port 6443 from the internet can enumerate resources, exec into pods, and read Secrets — no credentials required. Shodan and Censys continuously index exposed apiservers on ports 6443 and 8080.

**Enabling misconfiguration**
- apiserver bound to `0.0.0.0` with no firewall restricting port 6443
- `--anonymous-auth=true` not overridden
- `--authorization-mode` absent (defaults to `AlwaysAllow`)
- ClusterRoleBinding granting permissions to `system:unauthenticated`

**Real-world incident**
In 2018, RedLock researchers found Tesla's Kubernetes environment with its dashboard exposed to the internet without authentication. Attackers installed a cryptominer and used instance metadata credentials to access S3. The root cause — unauthenticated access to a Kubernetes management interface with no network restriction — is identical to an exposed apiserver.

**Kubernetes-native mitigation**
```
# apiserver flags
--anonymous-auth=false
--authorization-mode=Node,RBAC
```
- Bind the apiserver to a private IP; use cloud-managed private endpoints (AKS: authorized IP ranges)
- Audit for anonymous grants:
```bash
kubectl get clusterrolebinding -o json | \
  jq '.items[] | select(.subjects[]?.name == "system:anonymous" or .subjects[]?.name == "system:unauthenticated")'
```

**Maps to:** `01_kubernetes_security_primitives`, `02_authentication`, `04_authorization_rbac`

---

### 02 — Overpermissive RBAC

**MITRE ATT&CK:** Privilege Escalation — Valid Accounts (T1078)

**How it's exploited**
RBAC escalation exploits legitimate but overly broad permissions — no vulnerability required. Common patterns:

| Pattern | Why it's dangerous |
|---|---|
| `verbs: ["*"]` / `resources: ["*"]` | Any operation on any resource — effectively cluster-admin |
| `get secrets` at cluster scope | Reads every Secret in every namespace including kubeconfig and TLS keys |
| `create pods` + `exec` | Arbitrary code execution on any node |
| `cluster-admin` on a service account | Full cluster control from any compromised workload using that SA |
| `bind`, `escalate` on Roles | Allows the subject to grant themselves any permission that role contains |

**Enabling misconfiguration**
- Granting `cluster-admin` to service accounts or CI/CD pipelines for convenience during setup, never tightened
- Helm v2: Tiller deployed with a `cluster-admin` ClusterRoleBinding by default — any pod reaching Tiller had full cluster control
- No periodic RBAC audit; permissions accumulate over time

**Real-world incident**
Helm v2's Tiller was deployed by default with `cluster-admin`, making it a cluster-wide privilege escalation target reachable by any internal workload. This design flaw was a primary motivation for Helm v3 removing Tiller entirely. Bug bounty researchers at multiple organisations have demonstrated lateral escalation from a compromised developer service account with oversized RBAC grants.

**Kubernetes-native mitigation**
- Prefer namespaced `Role` + `RoleBinding` over `ClusterRole` + `ClusterRoleBinding` unless cluster-scope is genuinely required
- Never grant wildcard verbs or resources
- Audit effective permissions:
```bash
kubectl auth can-i --list --as=system:serviceaccount:<namespace>:<sa-name>
```
- Use `kubectl-who-can` or `rakkess` for cross-subject permission mapping

**Maps to:** `04_authorization_rbac`, `05_service_accounts`

---

### 03 — Privileged Pods and Host Namespace Sharing

**MITRE ATT&CK:** Privilege Escalation — Escape to Host (T1611)

**How it's exploited**
A pod with `privileged: true` receives all Linux capabilities and raw access to host devices — it is a root shell on the node. Even without full privilege, specific settings create container escape paths:

| Setting | Attack path |
|---|---|
| `privileged: true` | Direct host device access; cgroup `release_agent` write for host code execution |
| `hostPID: true` | Enumerate and signal all host processes; read `/proc/<pid>/mem` of host processes |
| `hostNetwork: true` | Bind to host interfaces; bypass NetworkPolicies; reach cloud metadata service (`169.254.169.254`) |
| `hostPath` mount to `/` or `/etc` | Read/write host filesystem including `/etc/kubernetes/pki` and kubeconfig files |
| `CAP_SYS_ADMIN` without `privileged` | Sufficient for several escape techniques independently |

**Enabling misconfiguration**
- No Pod Security Admission policy enforced on namespaces
- DaemonSets (node exporters, CNI agents) granted `privileged: true` without isolation in a separate namespace
- `hostPath` volumes to sensitive paths approved without review

**Real-world incident**
CVE-2022-0492 (CVSS 7.8): Linux kernel flaw in cgroup v1 `release_agent`. A container with `CAP_DAC_OVERRIDE` — or any privileged pod — could write to the `release_agent` file and execute arbitrary commands on the host. Affected all clusters running kernel < 5.17 without cgroup v2 enabled. Independently, Felix Wilhelm (2019) demonstrated the same `release_agent` vector requiring only a privileged container — no kernel CVE needed.

**Kubernetes-native mitigation**
Apply Pod Security Admission at namespace level:
```yaml
# Namespace labels
pod-security.kubernetes.io/enforce: restricted
pod-security.kubernetes.io/enforce-version: latest
```
The `restricted` profile blocks: `privileged`, `hostPID`, `hostIPC`, `hostNetwork`, all `hostPath` volumes, `allowPrivilegeEscalation: true`, and requires `runAsNonRoot: true`.

For workloads genuinely requiring elevated access (CNI plugins, monitoring DaemonSets), isolate them in a dedicated namespace with `baseline` or `privileged` PSA and compensate with NetworkPolicy and strict RBAC.

**Maps to:** `07_security_contexts`, `06_pod_security_admission`

---

### 04 — Unprotected etcd

**MITRE ATT&CK:** Credential Access — Data from Configuration Repository (T1213)

**How it's exploited**
etcd is the cluster's source of truth — it stores every object including Secrets. Kubernetes Secrets are base64-encoded in etcd by default, not encrypted. Base64 is reversible in one command. If etcd is reachable without TLS or without client certificate authentication, an attacker can:

1. Enumerate all stored keys: `etcdctl get / --prefix --keys-only`
2. Read any Secret directly: `etcdctl get /registry/secrets/production/db-creds`
3. Write arbitrary objects — ClusterRoleBindings, new admin users — directly to etcd, bypassing the apiserver and all admission controls entirely

**Enabling misconfiguration**
- etcd bound to `0.0.0.0:2379` with no firewall
- `--client-cert-auth=false` — unauthenticated connections accepted
- `--cert-file` and `--key-file` not set — no TLS on the client port
- No encryption at rest configured (see Flaw 08)

**Real-world incident**
A 2018 analysis by security researchers found thousands of etcd instances exposed on the public internet without authentication, storing API keys, database credentials, and kubeconfig files from production clusters. Several cryptocurrency exchanges had infrastructure credentials extracted via exposed etcd ports.

**Kubernetes-native mitigation**
```
# etcd flags
--cert-file=/etc/kubernetes/pki/etcd/server.crt
--key-file=/etc/kubernetes/pki/etcd/server.key
--client-cert-auth=true
--trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
--listen-client-urls=https://127.0.0.1:2379        # bind to localhost only

# apiserver flags (client cert presented to etcd)
--etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
--etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
--etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
```
- Firewall port 2379 — only the apiserver IP should reach it; no external access
- kubeadm configures all of the above correctly; verify manually on non-kubeadm clusters

**Maps to:** TLS certificate sections of this domain, `08_secrets_encryption`

---

### 05 — Service Account Token Abuse

**MITRE ATT&CK:** Credential Access — Steal Application Access Token (T1528)

**How it's exploited**
By default, every pod has a service account token auto-mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`. When a pod is compromised via an application vulnerability, the attacker reads this token and authenticates directly to the apiserver:

```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
APISERVER=https://kubernetes.default.svc

# Enumerate namespaces — scope of access depends on the SA's RBAC bindings
curl -sk $APISERVER/api/v1/namespaces \
  -H "Authorization: Bearer $TOKEN"
```

If the service account has overpermissive RBAC (see Flaw 02), this escalates an application-level compromise into cluster-level access without touching the host.

**Enabling misconfiguration**
- `automountServiceAccountToken: true` — the default on all service accounts and pods
- Applications using the `default` service account, which may have inherited ClusterRoleBindings
- Service accounts with `get secrets` or `list pods` at cluster scope
- Pre-1.22 clusters where tokens are long-lived, non-expiring Secrets rather than projected volumes

**Real-world incident**
Palo Alto Unit 42's Azurescape disclosure (2021): a cross-tenant container escape in Azure Container Instances chained multiple vulnerabilities; one stage involved leveraging a Kubernetes service account token to authenticate to the cluster API and pivot to access other tenants' resources within the shared control plane.

**Kubernetes-native mitigation**
```yaml
# Disable auto-mount at service account level
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: production
automountServiceAccountToken: false
```
```yaml
# Or override at pod level
spec:
  automountServiceAccountToken: false
```
- Grant each service account only the specific verbs and resources it requires — verify with `kubectl auth can-i --list --as=system:serviceaccount:<ns>:<sa>`
- In cloud environments, use projected tokens with short TTLs and audience binding: AKS Workload Identity / IRSA replace the default mounted token with short-lived, audience-scoped credentials

**Maps to:** `05_service_accounts`, `04_authorization_rbac`

---

### 06 — Unsigned and Unscanned Container Images

**MITRE ATT&CK:** Initial Access — Supply Chain Compromise (T1195.001)

**How it's exploited**
Without image signature verification at admission time, nothing prevents a malicious image from entering the cluster:

| Vector | Mechanism |
|---|---|
| **Typosquatting** | `nginix`, `ubuntuu` on public registries — legitimate-looking names pulling attacker-controlled images |
| **Tag mutation** | `image: app:latest` resolves to a different digest on each pull — the same tag can deliver different code |
| **Registry compromise** | Attacker pushes a backdoored image under a legitimate tag; downstream deployments pull it silently |
| **Dependency confusion** | Public package name shadows an internal one; build pipeline pulls the public (attacker) version |
| **No CVE scanning** | Known-vulnerable base image deployed to production; exploited after the fact |

**Enabling misconfiguration**
- Using mutable tags (`latest`, `stable`) instead of digest pinning (`image: name@sha256:<digest>`)
- No admission controller enforcing signature verification before a pod is scheduled
- Pulling from public registries directly with no allowlist or pull-through private registry
- No vulnerability scanning gate in CI before images are pushed to the registry

**Real-world incident**
Multiple Docker Hub cryptojacking campaigns (2018–2023) published typosquatted images closely resembling popular base images. Automated deployments pulling unreviewed images from public registries executed cryptominers. Several incidents resulted in sustained resource abuse before detection because the container otherwise appeared functional.

**Kubernetes-native mitigation**
- Sign images with **Cosign** (Sigstore) in CI; enforce at admission time via **Sigstore Policy Controller**, **Kyverno**, or **OPA/Gatekeeper**
- Pin images by digest in production manifests:
```yaml
image: myregistry.azurecr.io/app@sha256:a1b2c3d4...
```
- Use a private registry (ACR, ECR) as the single pull source; configure a pull-through cache for approved public images
- Integrate **Trivy** or **Grype** in CI to fail builds on critical CVEs before the image reaches the registry

**Maps to:** `09_securing_images`

---

### 07 — Missing Network Policies: Unrestricted Lateral Movement

**MITRE ATT&CK:** Lateral Movement — Remote Services (T1021)

**How it's exploited**
The Kubernetes flat network model allows a compromised pod to immediately reach, with no restrictions:
- Every other pod in every namespace on every port
- The kubelet HTTPS API on each node (port 10250) — see Flaw 09
- The cloud instance metadata service (`169.254.169.254` on AWS/Azure, `metadata.google.internal` on GCP) — yields IAM credentials for the node's cloud identity
- Internal databases, caches, and message queues that operators assume are "internal only"

The cluster produces no warning — NetworkPolicy absence is silent.

**Enabling misconfiguration**
- No NetworkPolicy objects created in any namespace
- CNI plugin that does not enforce NetworkPolicy (vanilla Flannel silently ignores NetworkPolicy objects)
- Overly broad allow policies that open more than intended

**Real-world incident**
TeamTNT's Hildegard campaign (2021) used unrestricted pod networking to scan for and connect to Redis instances, the Weave Scope management UI, and other internal cluster services from a single compromised container — using standard network tooling (`masscan`, `nmap`) installed in the attacking image.

**Kubernetes-native mitigation**
Apply a default-deny policy to every namespace at creation time:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```
Then add explicit allow policies for required traffic paths only.

- Block metadata service: add an Egress deny for `169.254.169.254/32` (and `fd00:ec2::254/128` on AWS IPv6)
- Verify your CNI enforces NetworkPolicy — Calico, Cilium, Azure CNI with Azure Network Policy, and Weave Net (with policy) do; vanilla Flannel does not

**Maps to:** `10_network_policies`

---

### 08 — Secrets Stored or Transmitted in Plaintext

**MITRE ATT&CK:** Credential Access — Unsecured Credentials (T1552)

**How it's exploited**
Kubernetes Secrets are base64-encoded, not encrypted. Base64 is a reversible encoding — it provides zero confidentiality:

```bash
# Any subject with get secrets permission reads plaintext in one step
kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d
```

In etcd without encryption at rest configured, the same Secret sits as:
```
/registry/secrets/production/db-creds → {"password":"<base64>"}
```

An attacker with etcd access (Flaw 04) reads all cluster Secrets with no RBAC bypass needed. Separately, Secrets injected as environment variables are visible in `kubectl describe pod`, in process listings (`/proc/<pid>/environ`), and in application crash dumps — leaking to any subject with `get pods` permission.

**Enabling misconfiguration**
- `--encryption-provider-config` not set on the apiserver (the default) — Secrets stored as base64 in etcd
- Secrets passed as `env` rather than mounted as volumes
- Secrets committed to version control or baked into container image layers during build
- Overly broad `get secrets` RBAC grants (see Flaw 02)

**Real-world incident**
The GitGuardian State of Secrets Sprawl reports consistently find Kubernetes Secrets, service account tokens, and kubeconfig files exposed in public Git repositories — committed during development or debugging and never cleaned up. Several major breach disclosures have included Kubernetes credentials sourced from public repositories.

**Kubernetes-native mitigation**
```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}   # fallback: reads pre-existing unencrypted Secrets
```
```
# apiserver flag
--encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```
After enabling, force re-encryption of all existing Secrets:
```bash
kubectl get secrets -A -o json | kubectl replace -f -
```
- **Production**: use a KMS provider (Azure Key Vault, AWS KMS) instead of `aescbc` — encryption keys never live on the cluster nodes
- Mount Secrets as volumes rather than env vars where possible
- Use the **External Secrets Operator** or **Azure Key Vault CSI driver** to source Secrets from a dedicated secrets store rather than Kubernetes Secrets at all

**Maps to:** Secrets encryption section of this domain

---

### 09 — Unauthenticated Kubelet API (Port 10250)

**MITRE ATT&CK:** Execution — Container Administration Command (T1609)

**How it's exploited**
The kubelet exposes an HTTPS API on port 10250 capable of exec-ing into running containers, streaming logs, and running arbitrary commands. If `anonymous-auth` is enabled on the kubelet, these endpoints require no credentials and bypass the apiserver entirely — meaning no RBAC check, no audit log entry through the apiserver, and no admission control:

```bash
# List all pods on a node — no token required on a vulnerable kubelet
curl -sk https://<node-ip>:10250/pods

# Execute a command inside a running container
curl -sk https://<node-ip>:10250/run/<namespace>/<pod>/<container> \
  -d "cmd=cat /etc/shadow"
```

Any pod in the cluster can reach node IPs on port 10250 unless a NetworkPolicy prevents it — combining Flaw 07 with this one creates a direct path from pod compromise to node-level execution.

**Enabling misconfiguration**
- `authentication.anonymous.enabled: true` in kubelet config
- `authorization.mode: AlwaysAllow` in kubelet config
- Port 10250 reachable from the pod network or externally
- Read-only port 10255 enabled on older clusters (removed from kubeadm defaults post-1.16 but may persist on long-running clusters)

**Real-world incident**
TeamTNT's Hildegard malware (2021) automated scanning for port 10250 across RFC1918 ranges and used the unauthenticated `/run` endpoint to install a cryptominer directly inside running containers — no pod compromise or credential theft required. The attack required only network reachability to the kubelet port.

**Kubernetes-native mitigation**
```yaml
# /var/lib/kubelet/config.yaml  (KubeletConfiguration)
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true        # delegate authentication to the apiserver
authorization:
  mode: Webhook          # delegate authorization to apiserver (honours RBAC)
```
kubeadm sets both of these correctly. To verify an existing cluster:
```bash
# Check kubelet config via apiserver proxy (requires apiserver access)
kubectl get --raw /api/v1/nodes/<node-name>/proxy/configz | jq .kubeletconfig.authentication

# Direct check from within the cluster — should return 401, not pod data
curl -sk https://<node-ip>:10250/pods
```
- Firewall port 10250 to allow only the apiserver and monitoring agents; deny from the pod CIDR

**Maps to:** Kubelet security / TLS sections of this domain

---

### 10 — Persistence via Malicious Admission Webhook

**MITRE ATT&CK:** Persistence — Implant Internal Image (T1525)

**How it's exploited**
MutatingAdmissionWebhooks intercept every matching API request before the object is persisted and can silently rewrite any pod spec. An attacker who gains `create` or `update` on `mutatingwebhookconfigurations` — a ClusterRole-level permission — can register a webhook that:
- Injects a malicious sidecar container into every new pod in the cluster
- Exfiltrates pod environment variables and mounted Secret paths from every spec before they reach etcd
- Adds `hostPID: true` or `privileged: true` to targeted workloads without the owner seeing it in their submitted manifest
- Sets `failurePolicy: Fail` with an unavailable backend, causing a denial-of-service on all new pod scheduling

The mutation is applied server-side — `kubectl describe pod` shows the mutated spec, not what the user submitted. The original intent is lost.

**Enabling misconfiguration**
- RBAC allows non-admin subjects to `create` or `update` `mutatingwebhookconfigurations`
- `namespaceSelector` and `objectSelector` not configured — webhook applies to all namespaces and all objects
- No alerting on changes to webhook configurations
- Legitimate webhook backends (OPA, Kyverno) compromised or their serving certificates rotated by an attacker

**Real-world incident**
CyberArk researchers (2020) published a proof-of-concept demonstrating that a MutatingAdmissionWebhook can be used as a persistent implant following an initial RBAC compromise. The technique maintains cluster-wide code execution even after the original foothold — a compromised pod or stolen token — is revoked and cleaned up.

**Kubernetes-native mitigation**
- Restrict `mutatingwebhookconfigurations` and `validatingwebhookconfigurations` to cluster-admin:
```bash
# Audit who can create/update webhook configs
kubectl get clusterrolebinding -o json | \
  jq '.items[] | select(.subjects != null) | 
  select(.roleRef.name | test("admin|webhook")) | 
  {role: .roleRef.name, subjects: .subjects}'
```
- Scope all webhook configurations explicitly — never leave `namespaceSelector` and `objectSelector` empty (cluster-wide catch-all):
```yaml
namespaceSelector:
  matchLabels:
    webhook.myorg.io/enabled: "true"   # opt-in, not opt-out
```
- Enable audit logging and alert on any `create` or `update` event on `mutatingwebhookconfigurations` or `validatingwebhookconfigurations`
- Review periodically: `kubectl get mutatingwebhookconfigurations -o yaml`

**Maps to:** Admission controllers section of this domain