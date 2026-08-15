## Kubernetes Releases and Versioning

### What it is

Kubernetes follows **semantic versioning** (`MAJOR.MINOR.PATCH`) for all control plane component releases. Understanding the versioning scheme is a prerequisite for planning cluster upgrades safely.

---

### Version Number Anatomy

```
v1.36.3
│  │   └── PATCH — bug fixes and security patches; released frequently
│  └────── MINOR — new features and API additions; released every ~4 months
└───────── MAJOR — breaking changes; has been v1 since July 2015
```
[🔗 Latest stable release](https://kubernetes.io/releases/)

Example from `kubectl get nodes`:
```
NAME        STATUS   VERSION
node-1      Ready    v1.36.3
```

---

### Release Types

| Release Type | Tag Example | Features Enabled by Default | Stability |
|---|---|---|---|
| Alpha | `v1.38.0-alpha.0` | No | Potentially buggy; may be removed |
| Beta | `v1.38.0-beta.0` | Yes | Well tested; API may still change |
| Stable | `v1.36.3` | Yes | Production-ready; API committed |

**Release pipeline:** Bug fixes and new features enter Alpha → promoted to Beta once tested → land in a stable minor release.

> Alpha features are off by default and must be explicitly enabled via feature gates. Beta features are on by default but not yet locked into the API contract.

---

### Release Cadence

| Release Type | Frequency |
|---|---|
| Minor release | Every ~15 weeks (approximately 3–4 releases per year) |
| Patch release | As needed — critical bug fixes and CVE patches |
| Alpha / Beta | Continuous — tagged during the development cycle |

**Current release state (August 2026):**

| Version | Status | Latest Patch | End of Life |
|---|---|---|---|
| v1.36 | ✅ Active support | v1.36.3 (Jul 22, 2026) | Jun 28, 2027 |
| v1.35 | ✅ Active support | v1.35.7 (Jul 22, 2026) | Feb 28, 2027 |
| v1.34 | ✅ Active support | v1.34.10 (Jul 22, 2026) | Oct 27, 2026 |
| v1.33 | ❌ End of Life | — | Jun 28, 2026 |
| v1.37 | 🔜 RC phase | v1.37.0-rc.0 | Releasing Aug 26, 2026 |

**Notable historical releases:**
- `v1.0.0` — July 2015 (GA launch)
- `v1.20` — December 2020
- `v1.33` — April 2025
- `v1.34` — August 2025
- `v1.35` — December 2025
- `v1.36` — ~April 2026 (latest stable minor)

---

### Support Policy — N-2

Kubernetes maintains the **3 most recent minor versions** in active support at any time (N, N-1, N-2). Once a version falls outside this window it receives no further patches — including security fixes.

Each minor version receives approximately **14 months of total support**:
- 12 months of active patch support
- 2 months of upgrade grace period (maintenance mode)

> This means if you are on v1.33 today, you are running an **unsupported, unpatched** cluster. Minimum supported version is currently v1.34.

---

### Component Versioning — What Ships Together

When you download a Kubernetes release tarball, all **core control plane components** are at the same version:

| Component | Versioned With Kubernetes |
|---|---|
| `kube-apiserver` | ✅ Yes |
| `kube-controller-manager` | ✅ Yes |
| `kube-scheduler` | ✅ Yes |
| `kubelet` | ✅ Yes |
| `kube-proxy` | ✅ Yes |
| `kubectl` | ✅ Yes |

**Independent versioning** — these are separate projects with their own release cycles:

| Component | Why Independent |
|---|---|
| `etcd` | CNCF project; separate release cycle |
| `CoreDNS` | CNCF project; separate release cycle |
| Container runtime (containerd, CRI-O) | Independent upstream projects |

> The Kubernetes release notes for each version document the **supported version ranges** for etcd and CoreDNS. Always check these before upgrading — a mismatch can break the cluster.

---

### Checking Current Versions

```bash
# kubelet version on each node
kubectl get nodes

# kubectl client version + kube-apiserver version
kubectl version

# All control plane component versions via their pod images (kube-system namespace)
kubectl -n kube-system get pods -o jsonpath="{range .items[*]}{.metadata.name}{'\t'}{.spec.containers[*].image}{'\n'}{end}"

# etcd version (-c etcd required — pod has a single container but good practice)
kubectl -n kube-system exec -it etcd-<node> -c etcd -- etcd --version

# CoreDNS version
kubectl -n kube-system get deployment coredns -o jsonpath='{.spec.template.spec.containers[0].image}'
```

---

### Where to Find Releases

- GitHub releases: `https://github.com/kubernetes/kubernetes/releases`
- Official release page: `https://kubernetes.io/releases/`
- Support end dates and patch history: `https://kubernetes.io/releases/patch-releases/`
- EOL tracking: `https://endoflife.date/kubernetes`

---

### When to Use Which

| Goal | What to Check |
|---|---|
| Plan a cluster upgrade | Current minor → nearest supported minor (N-2 policy) |
| Verify component compatibility | Release notes for supported etcd/CoreDNS versions |
| Assess security exposure | Is current version within the 3-version support window? |
| Enable an experimental feature | Check feature gate status (Alpha/Beta/GA) for that version |

---

### Real-World Usage

**Version skew in multi-node clusters:** During a rolling upgrade, nodes may temporarily run different patch versions. Kubernetes defines a supported **version skew policy** (covered in the cluster upgrade section) governing which component version combinations are valid during the upgrade window.

**AKS versioning:** AKS surfaces available versions via `az aks get-versions --location <region>`. AKS qualifies and tests versions before making them available — not every upstream minor version is immediately offered. AKS also enforces deprecation windows, auto-upgrading clusters that fall outside supported versions.

---

### Exam Gotchas

- `etcd` and `CoreDNS` versions are **not** the same as the Kubernetes version — they are independent projects. Assuming all cluster components share one version number is a common trap.
- `kubectl version` shows both client and server version — client and server can differ within the supported skew.
- Kubernetes supports only the **3 most recent minor versions** — anything older receives no security patches. Know this when asked about upgrade planning.
- Alpha features are **disabled by default** — enabled via `--feature-gates` on the relevant component. Beta features are enabled by default.
- The major version has been `1` since 2015 — `v2.0` does not exist. Exam scenarios always use `v1.x.y`.
- Release cadence is approximately **every 15 weeks (3–4 per year)** — not every 3 months exactly.

---

### In Managed Clusters (AKS / EKS / GKE)

- Managed control planes abstract etcd and CoreDNS version management — upgraded by the cloud provider as part of the Kubernetes version upgrade.
- AKS enforces support windows: clusters on deprecated versions are auto-upgraded if left unmaintained.
- Available versions per region: `az aks get-versions --location eastus --output table`
- AKS does not expose every upstream patch — only qualified patch versions are offered.
- EKS follows a similar N-2 model with extended support options available for an additional fee.

