# 09 — Exam Cheatsheet

> Commands only. No explanations. Rapid-fire revision before the exam.
> Rule: if you hesitate on a command during revision, you need more practice with it.

---

## Environment Setup (do this first in the exam)

```bash
# Enable kubectl autocomplete
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

# Set alias (saves keystrokes across hundreds of commands)
alias k=kubectl
complete -o default -F __start_kubectl k

# Set ETCDCTL API version
export ETCDCTL_API=3
```

---

## etcd

```bash
# Backup
ETCDCTL_API=3 etcdctl snapshot save /opt/snapshot-pre-boot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Restore
ETCDCTL_API=3 etcdctl snapshot restore /opt/snapshot-pre-boot.db \
  --data-dir=/var/lib/etcd-from-backup

# Verify snapshot
ETCDCTL_API=3 etcdctl snapshot status /opt/snapshot-pre-boot.db

# List all keys in etcd
kubectl exec etcd-controlplane -n kube-system -- \
  sh -c "ETCDCTL_API=3 etcdctl get / --prefix --keys-only --limit=10 \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/server.crt \
  --key /etc/kubernetes/pki/etcd/server.key"

# Check etcdctl version and API version
etcdctl version
```

---

## Inspect Control Plane Components

```bash
# Static pod manifests (kubeadm clusters)
cat /etc/kubernetes/manifests/kube-apiserver.yaml
cat /etc/kubernetes/manifests/etcd.yaml
cat /etc/kubernetes/manifests/kube-scheduler.yaml
cat /etc/kubernetes/manifests/kube-controller-manager.yaml

# Live process inspection
ps -aux | grep kube-apiserver
ps -aux | grep kube-controller-manager
ps -aux | grep kube-scheduler
ps -aux | grep kubelet

# systemd service files (manual/scratch setup)
cat /etc/systemd/system/kube-apiserver.service
cat /etc/systemd/system/kube-controller-manager.service

# kubelet config and service
systemctl status kubelet
cat /var/lib/kubelet/config.yaml
cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

---

## crictl (container troubleshooting on nodes)

```bash
crictl ps                             # list running containers
crictl ps -a                          # list all containers including stopped
crictl pods                           # list pods
crictl images                         # list pulled images
crictl logs <container-id>            # view container logs
crictl exec -it <container-id> sh     # exec into container
crictl inspect <container-id>         # inspect container details
crictl ports <container-id>           # list ports

# Set runtime endpoint explicitly
crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps
export CONTAINER_RUNTIME_ENDPOINT=unix:///run/containerd/containerd.sock
```

---

## Pods

```bash
# Create a pod imperatively
kubectl run nginx --image=nginx

# Generate pod YAML without creating it — fastest way in the exam
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# Create from file
kubectl create -f pod.yaml
kubectl apply -f pod.yaml

# List pods
kubectl get pods
kubectl get pods -o wide                        # includes node and IP
kubectl get pods -n <namespace>                 # specific namespace
kubectl get pods --all-namespaces               # all namespaces
kubectl get pods -l app=myapp                   # filter by label

# Inspect
kubectl describe pod <pod-name>                 # full detail + events
kubectl get pod <pod-name> -o yaml              # full YAML definition

# Logs
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>     # multi-container pod

# Exec into a pod
kubectl exec -it <pod-name> -- sh
kubectl exec -it <pod-name> -c <container> -- sh

# Delete
kubectl delete pod <pod-name>
kubectl delete pod <pod-name> --force           # immediate deletion
```

---

## ReplicaSets

```bash
# Create
kubectl create -f rs-definition.yaml

# List
kubectl get replicaset
kubectl get rs                                  # shorthand

# Inspect
kubectl describe replicaset <rs-name>

# Scale — method 1: edit file then replace
kubectl replace -f rs-definition.yaml

# Scale — method 2: scale command via file (does NOT update file)
kubectl scale --replicas=6 -f rs-definition.yaml

# Scale — method 3: scale command via name (does NOT update file)
kubectl scale --replicas=6 replicaset <rs-name>

# Delete RS and all its pods
kubectl delete replicaset <rs-name>

# Delete RS but keep pods
kubectl delete replicaset <rs-name> --cascade=orphan

# Generate RS YAML template
kubectl create -f rs.yaml --dry-run=client -o yaml
```

---

## Deployments

```bash
# Create imperatively
kubectl create deployment nginx --image=nginx
kubectl create deployment nginx --image=nginx --replicas=3

# Generate Deployment YAML without creating
kubectl create deployment nginx --image=nginx --replicas=3 --dry-run=client -o yaml > deployment.yaml

# Apply from file
kubectl apply -f deployment.yaml

# List deployments
kubectl get deployments
kubectl get deploy                              # shorthand

# View full object hierarchy at once
kubectl get all

# Inspect
kubectl describe deployment <deployment-name>

# Scale
kubectl scale deployment nginx --replicas=5

# Update container image
kubectl set image deployment nginx nginx=nginx:1.18

# Edit live deployment
kubectl edit deployment nginx

# Delete deployment (also deletes RS and pods)
kubectl delete deployment <deployment-name>
```

---

## Services

```bash
# Create from file
kubectl apply -f service.yaml

# Expose a pod — uses pod labels as selectors automatically
kubectl expose pod nginx --type=NodePort --port=80 --name=nginx-service --dry-run=client -o yaml
kubectl expose pod redis --port=6379 --name=redis-service --dry-run=client -o yaml

# Expose a deployment
kubectl expose deployment nginx --port=80 --dry-run=client -o yaml

# Create ClusterIP service — does NOT use pod labels (assumes app=redis)
kubectl create service clusterip redis --tcp=6379:6379 --dry-run=client -o yaml

# Create NodePort service — can specify nodePort but does NOT use pod labels
kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run=client -o yaml

# List services
kubectl get services
kubectl get svc                                 # shorthand

# Inspect — check endpoints to verify selector is matching pods
kubectl describe service <service-name>

# Verify pods are being selected by the service
kubectl get endpoints <service-name>

# Access NodePort service
curl http://<node-ip>:<node-port>

# Watch for EXTERNAL-IP assignment (LoadBalancer)
kubectl get svc <service-name> --watch

# Delete
kubectl delete service <service-name>
```

---

## Namespaces

```bash
# Create namespace imperatively
kubectl create namespace dev

# Create from file
kubectl apply -f namespace.yaml

# List all namespaces
kubectl get namespaces
kubectl get ns                                  # shorthand

# List pods in a specific namespace
kubectl get pods -n kube-system
kubectl get pods -n dev

# List pods across ALL namespaces
kubectl get pods --all-namespaces
kubectl get pods -A                             # shorthand

# Create resource in a specific namespace
kubectl apply -f pod.yaml -n dev

# Switch current context namespace permanently
kubectl config set-context --current --namespace=dev

# Verify current namespace
kubectl config view --minify | grep namespace

# Create resource quota
kubectl apply -f resource-quota.yaml
kubectl get resourcequota -n dev
kubectl describe resourcequota -n dev
```

---

## Imperative Commands — Exam Quick Reference

```bash
# --- CREATE OBJECTS ---

# Pod
kubectl run nginx --image=nginx
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# Deployment
kubectl create deployment nginx --image=nginx
kubectl create deployment nginx --image=nginx --replicas=4 --dry-run=client -o yaml > deploy.yaml

# Service — ClusterIP (uses pod labels as selector)
kubectl expose pod redis --port=6379 --name=redis-service

# Service — NodePort (uses pod labels, cannot set nodePort — edit YAML manually)
kubectl expose pod nginx --type=NodePort --port=80 --name=nginx-service --dry-run=client -o yaml > svc.yaml

# Namespace
kubectl create namespace dev

# --- UPDATE OBJECTS ---

# Edit live object directly
kubectl edit deployment nginx
kubectl edit pod nginx

# Scale
kubectl scale deployment nginx --replicas=5

# Update image
kubectl set image deployment nginx nginx=nginx:1.18

# Replace from file (object must exist)
kubectl replace -f deployment.yaml

# Force delete and recreate (for immutable field changes)
kubectl replace --force -f deployment.yaml

# --- DECLARATIVE ---

# Apply single file — create or update
kubectl apply -f nginx.yaml

# Apply all files in a directory
kubectl apply -f ./manifests/

# Apply recursively
kubectl apply -f ./manifests/ -R
```

---

## kubectl explain and api-resources

```bash
# Discover all resource names, short names, API versions
kubectl api-resources

# Filter for a specific resource
kubectl api-resources | grep ingress
kubectl api-resources | grep deploy

# Explain top-level fields of a resource
kubectl explain pod
kubectl explain deployment
kubectl explain service

# Drill into a specific field
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl explain deployment.spec.strategy

# Full recursive field list — fastest way to build YAML
kubectl explain pod --recursive
kubectl explain deployment --recursive
kubectl explain persistentvolume --recursive
```

---

## Key File Paths

```bash
/etc/kubernetes/manifests/          # static pod definitions (control plane)
/etc/kubernetes/admin.conf          # cluster admin kubeconfig
/etc/kubernetes/pki/                # all cluster TLS certificates
/etc/kubernetes/pki/etcd/           # etcd-specific certificates
/var/lib/etcd/                      # etcd data directory
/var/lib/kubelet/config.yaml        # kubelet configuration
/etc/systemd/system/                # systemd service files (manual setup)
```

---

## Key Ports

| Component | Port | Purpose |
|---|---|---|
| `kube-apiserver` | 6443 | HTTPS API — all client and component traffic |
| `etcd` | 2379 | Client connections (kube-apiserver → etcd) |
| `etcd` | 2380 | Peer communication (etcd ↔ etcd in HA) |
| `kubelet` | 10250 | HTTPS — apiserver → kubelet communication |
| `kube-scheduler` | 10259 | HTTPS metrics / health |
| `kube-controller-manager` | 10257 | HTTPS metrics / health |

---

## Quick Reference — apiVersion by Kind

| Kind | apiVersion |
|---|---|
| `Pod` | `v1` |
| `Service` | `v1` |
| `Namespace` | `v1` |
| `ServiceAccount` | `v1` |
| `ReplicaSet` | `apps/v1` |
| `Deployment` | `apps/v1` |
| `DaemonSet` | `apps/v1` |
| `StatefulSet` | `apps/v1` |
| `Job` | `batch/v1` |
| `CronJob` | `batch/v1` |
| `Ingress` | `networking.k8s.io/v1` |
| `NetworkPolicy` | `networking.k8s.io/v1` |
| `PersistentVolume` | `v1` |
| `PersistentVolumeClaim` | `v1` |
| `ClusterRole` | `rbac.authorization.k8s.io/v1` |
| `ClusterRoleBinding` | `rbac.authorization.k8s.io/v1` |
| `Role` | `rbac.authorization.k8s.io/v1` |
| `RoleBinding` | `rbac.authorization.k8s.io/v1` |