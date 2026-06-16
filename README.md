# CKA Notes

Personal study notes for the **Certified Kubernetes Administrator (CKA)** exam.

**Course:** Certified Kubernetes Administrator
**Exam provider:** Cloud Native Computing Foundation (CNCF)

---

## Repo Structure

```
CKA-NOTES/
├── 01_core_concepts/
├── 02_scheduling/
├── 03_logging_monitoring/
├── 04_application_lifecycle_management/
├── 05_cluster_maintenance/
├── 06_security/
├── 07_storage/
├── 08_networking/
├── 09_design_and_install_kubernetes_cluster/
├── 10_install_kubernetes_kubeadm/
├── 11_troubleshooting/
├── 12_helm/
├── 13_kubectl_advanced/
├── exam_cheatsheet.md
└── README.md
```

Each folder represents a CKA exam domain. Every lecture has its own `.md` file inside the relevant domain folder.

---

## Note Format

Every section follows this structure:

| Block | Purpose |
|---|---|
| **What it is** | Precise technical definition |
| **Key concepts / tables** | Core mechanics, comparisons, YAML structure |
| **Commands** | kubectl, crictl, etcdctl — exam-ready |
| **Exam Gotchas** | Common mistakes and exam-specific traps |
| **In Managed Clusters (AKS / EKS / GKE)** | Real-world context — only where behaviour differs |
| **Practice Test Lesson Learned** | Filled in after completing the hands-on lab |

---

## Progress Tracker

### 01 — Core Concepts

| # | Section | Status |
|---|---|---|
| 01 | Cluster Architecture | ✅ Done |
| 02 | Docker vs containerd | ✅ Done |
| 03 | A Note on Docker Deprecation | ✅ Done |
| 04 | etcd for Beginners | ✅ Done |
| 05 | etcd in Kubernetes | ✅ Done |
| 06 | kube-apiserver | ✅ Done |
| 07 | kube-controller-manager | ✅ Done |
| 08 | kube-scheduler | ✅ Done |
| 09 | kubelet | ✅ Done |
| 10 | kube-proxy | ✅ Done |
| 11 | Pods | ✅ Done |
| 12 | Pods with YAML | ✅ Done |
| 13 | ReplicaSets | ✅ Done |
| 14 | Deployments | ✅ Done |
| 15 | Services | ✅ Done |
| 16 | Namespaces | ✅ Done |
| 17 | Imperative vs Declarative | ✅ Done |
| 18 | kubectl apply | ✅ Done |

### 02 — Scheduling

| # | Section | Status |
|---|---|---|
| 01 | Manual Scheduling | ✅ Done |
| 02 | Labels_Selectors_Annotations | ✅ Done |
| 03 | Taints and Tolerations | ✅ Done |
| 04 | Node Selectors | ✅ Done |
| 05 | Node Affinity | ✅ Done |
| 06 | Taints & Tolerations vs Node Affinity | ✅ Done |
| 07 | Resource Requirements and Limits | ✅ Done |
| 08 | Editing Pods and Deployments | ✅ Done |
| 09 | DaemonSets | ✅ Done |
| 10 | Static Pods | ✅ Done |
| 11 | Priority Classes | ✅ Done |
| 12 | Multiple Schedulers | ✅ Done |
| 13 | Scheduler Profiles | ✅ Done |
| 14 | Admission Controllers | ✅ Done |
| 15 | Validating and Mutating Admission Controllers | ✅ Done |

### 03 — Logging & Monitoring

| # | Section | Status |
|---|---|---|
| 01 | Monitor Cluster Components | ✅ Done |
| 02 | Managing Application Logs | ✅ Done |
| 03 | Production Logging Monitoring Stack | ✅ Done |
| 04 | Production Commands Reference | ✅ Done |

### 04 — Application Lifecycle Management

| # | Section | Status |
|---|---|---|
| 01 | Rolling Updates and Rollbacks | ✅ Done |
| 02 | Configure Applications Intro | ✅ Done |
| 02 | Commands and Arguments | ⬜ Pending |
| 03 | Environment Variables | ⬜ Pending |
| 04 | ConfigMaps | ⬜ Pending |
| 05 | Secrets | ⬜ Pending |
| 06 | Multi-Container Pods | ⬜ Pending |
| 07 | Init Containers | ⬜ Pending |

### 05 — Cluster Maintenance

| # | Section | Status |
|---|---|---|
| 01 | OS Upgrades | ⬜ Pending |
| 02 | Kubernetes Releases | ⬜ Pending |
| 03 | Cluster Upgrade Process | ⬜ Pending |
| 04 | Backup and Restore Methods | ⬜ Pending |

### 06 — Security

| # | Section | Status |
|---|---|---|
| 01 | Kubernetes Security Primitives | ⬜ Pending |
| 02 | Authentication | ⬜ Pending |
| 03 | TLS Certificates | ⬜ Pending |
| 04 | Certificate API | ⬜ Pending |
| 05 | KubeConfig | ⬜ Pending |
| 06 | API Groups | ⬜ Pending |
| 07 | RBAC | ⬜ Pending |
| 08 | Cluster Roles | ⬜ Pending |
| 09 | Service Accounts | ⬜ Pending |
| 10 | Image Security | ⬜ Pending |
| 11 | Security Contexts | ⬜ Pending |
| 12 | Network Policies | ⬜ Pending |

### 07 — Storage

| # | Section | Status |
|---|---|---|
| 01 | Volumes | ⬜ Pending |
| 02 | Persistent Volumes | ⬜ Pending |
| 03 | Persistent Volume Claims | ⬜ Pending |
| 04 | Storage Classes | ⬜ Pending |
| 05 | StatefulSets | ⬜ Pending |

### 08 — Networking

| # | Section | Status |
|---|---|---|
| 01 | Networking Concepts | ⬜ Pending |
| 02 | CNI | ⬜ Pending |
| 03 | Pod Networking | ⬜ Pending |
| 04 | CoreDNS | ⬜ Pending |
| 05 | Ingress | ⬜ Pending |

### 09 — Design and Install a Kubernetes Cluster

| # | Section | Status |
|---|---|---|
| 01 | Cluster Design Considerations | ⬜ Pending |
| 02 | Choosing a Kubernetes Infrastructure | ⬜ Pending |
| 03 | High Availability Setup | ⬜ Pending |

### 10 — Install Kubernetes the kubeadm Way

| # | Section | Status |
|---|---|---|
| 01 | kubeadm Introduction | ⬜ Pending |
| 02 | Provision VMs with Vagrant | ⬜ Pending |
| 03 | Deploy Kubernetes with kubeadm | ⬜ Pending |

### 11 — Troubleshooting

| # | Section | Status |
|---|---|---|
| 01 | Application Failure | ⬜ Pending |
| 02 | Control Plane Failure | ⬜ Pending |
| 03 | Worker Node Failure | ⬜ Pending |
| 04 | Network Troubleshooting | ⬜ Pending |

### 12 — Helm

| # | Section | Status |
|---|---|---|
| 01 | Helm Concepts | ⬜ Pending |
| 02 | Helm Charts | ⬜ Pending |
| 03 | Helm Commands | ⬜ Pending |

### 13 — kubectl Advanced

| # | Section | Status |
|---|---|---|
| 01 | JSONPath | ⬜ Pending |
| 02 | Custom Columns | ⬜ Pending |
| 03 | Sorting Output | ⬜ Pending |

---

## Exam Cheatsheet

See [`exam_cheatsheet.md`](./exam_cheatsheet.md) — commands only, rapid-fire revision.

---

## CKA Exam Domain Weights

| Domain | Weight |
|---|---|
| Cluster Architecture, Installation & Configuration | 25% |
| Workloads & Scheduling | 15% |
| Services & Networking | 20% |
| Storage | 10% |
| Troubleshooting | 30% |