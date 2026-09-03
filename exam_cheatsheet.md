# Exam Cheatsheet

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
# Check etcdctl version and API version
etcdctl version

# Backup (online — requires running etcd + certs)
ETCDCTL_API=3 etcdctl snapshot save /opt/snapshot-pre-boot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot (etcdutl preferred in v3.5+; etcdctl also works)
etcdutl snapshot status /opt/snapshot-pre-boot.db --write-out=table

# Restore (etcdutl preferred in v3.5+ — offline, no certs needed)
etcdutl snapshot restore /opt/snapshot-pre-boot.db \
  --data-dir=/var/lib/etcd-from-backup

# File-level backup (offline, no certs needed, etcd can be stopped)
etcdutl backup \
  --data-dir /var/lib/etcd \
  --backup-dir /backup/etcd-backup

# List all keys in etcd
kubectl exec etcd-controlplane -n kube-system -- \
  sh -c "ETCDCTL_API=3 etcdctl get / --prefix --keys-only --limit=10 \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/server.crt \
  --key /etc/kubernetes/pki/etcd/server.key"
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
kubectl get pods --watch (or -w)                # streams real-time lifecycle changes of your Pods.
kubectl get pods -o wide                        # includes node and IP
kubectl get pods -n <namespace>                 # specific namespace
kubectl get pods --all-namespaces               # all namespaces
kubectl get pods -l app=myapp                   # filter by label
kubectl get pods --all-namespaces --field-selector spec.nodeName=<NODE_NAME>    # to find all pods across all namespaces running on a specific node
kubectl get pods -n <NAMESPACE_NAME> --field-selector spec.nodeName=<NODE_NAME> # to find all pods in a single namespace running on a specific node

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

```
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

| Component                 | Port  | Purpose                                      |
| ------------------------- | ----- | -------------------------------------------- |
| `kube-apiserver`          | 6443  | HTTPS API — all client and component traffic |
| `etcd`                    | 2379  | Client connections (kube-apiserver → etcd)   |
| `etcd`                    | 2380  | Peer communication (etcd ↔ etcd in HA)       |
| `kubelet`                 | 10250 | HTTPS — apiserver → kubelet communication    |
| `kube-scheduler`          | 10259 | HTTPS metrics / health                       |
| `kube-controller-manager` | 10257 | HTTPS metrics / health                       |

---

## Quick Reference — apiVersion by Kind

| Kind                             | apiVersion                        |
| -------------------------------- | --------------------------------- |
| `Pod`                            | `v1`                              |
| `Service`                        | `v1`                              |
| `Namespace`                      | `v1`                              |
| `ServiceAccount`                 | `v1`                              |
| `LimitRange`                     | `v1`                              |
| `ResourceQuota`                  | `v1`                              |
| `Binding`                        | `v1`                              |
| `ConfigMap`                      | `v1`                              |
| `Secret`                         | `v1`                              |
| `ReplicaSet`                     | `apps/v1`                         |
| `Deployment`                     | `apps/v1`                         |
| `DaemonSet`                      | `apps/v1`                         |
| `StatefulSet`                    | `apps/v1`                         |
| `Job`                            | `batch/v1`                        |
| `CronJob`                        | `batch/v1`                        |
| `Ingress`                        | `networking.k8s.io/v1`            |
| `NetworkPolicy`                  | `networking.k8s.io/v1`            |
| `PersistentVolume`               | `v1`                              |
| `PersistentVolumeClaim`          | `v1`                              |
| `PodDisruptionBudget`            | `policy/v1`                       |
| `ClusterRole`                    | `rbac.authorization.k8s.io/v1`    |
| `ClusterRoleBinding`             | `rbac.authorization.k8s.io/v1`    |
| `Role`                           | `rbac.authorization.k8s.io/v1`    |
| `RoleBinding`                    | `rbac.authorization.k8s.io/v1`    |
| `PriorityClass`                  | `scheduling.k8s.io/v1`            |
| `ValidatingWebhookConfiguration` | `admissionregistration.k8s.io/v1` |
| `MutatingWebhookConfiguration`   | `admissionregistration.k8s.io/v1` |
| `HorizontalPodAutoscaler`        | `autoscaling/v2`                  |
| `VerticalPodAutoscaler`          | `autoscaling.k8s.io/v1`           |

---

## 02 — Scheduling

### Manual Scheduling

```bash
# Check why a pod is Pending (no scheduler / no nodeName)
kubectl get pods -o wide                        # NODE column shows <none>
kubectl describe pod <pod-name>                 # check Events for scheduling errors

# Assign existing Pending pod to a node via Binding API
kubectl proxy --port=8001 &
curl -X POST \
  http://localhost:8001/api/v1/namespaces/default/pods/<pod-name>/binding \
  -H "Content-Type: application/json" \
  -d '{
    "apiVersion": "v1",
    "kind": "Binding",
    "metadata": { "name": "<pod-name>", "namespace": "default" },
    "target": { "apiVersion": "v1", "kind": "Node", "name": "<node-name>" }
  }'
```

---

### Labels and Selectors

```bash
# Filter pods by label
kubectl get pods -l app=payments-api
kubectl get pods -l app=payments-api,env=prod   # AND logic

# Filter any resource type
kubectl get all -l env=prod

# Show labels on pods
kubectl get pods --show-labels

# Show labels on nodes
kubectl get nodes --show-labels

# Add a label to a pod
kubectl label pod <pod-name> env=prod

# Remove a label from a pod
kubectl label pod <pod-name> env-

# Compare priority classes across pods
kubectl get pods -o custom-columns="NAME:.metadata.name,PRIORITY:.spec.priorityClassName"
kubectl get pods -A -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,PRIORITY:.spec.priorityClassName"
```

---

### Taints and Tolerations

```bash
# Apply a taint to a node
kubectl taint nodes <node-name> app=payments:NoSchedule
kubectl taint nodes <node-name> app=payments:NoExecute
kubectl taint nodes <node-name> app=payments:PreferNoSchedule

# Remove a taint (append -)
kubectl taint nodes <node-name> app=payments:NoSchedule-

# View taints on a node
kubectl describe node <node-name> | grep -i taint

# View master node taint
kubectl describe node <control-plane-node> | grep -i taint
```

---

### Node Labels and Selectors

```bash
# Add a label to a node
kubectl label nodes <node-name> size=large

# Remove a label from a node
kubectl label nodes <node-name> size-

# Verify node labels
kubectl get node <node-name> --show-labels
```

---

### Resource Requirements

```bash
# View resource usage across pods
kubectl top pods
kubectl top pods -n <namespace>
kubectl top nodes

# Check why pod is Pending (insufficient resources)
kubectl describe pod <pod-name>                 # look for: Insufficient cpu / memory

# View LimitRange in a namespace
kubectl get limitrange -n <namespace>
kubectl describe limitrange -n <namespace>

# View ResourceQuota in a namespace
kubectl get resourcequota -n <namespace>
kubectl describe resourcequota -n <namespace>   # shows used vs hard limits
```

---

### Editing Pods and Deployments

```bash
# Edit a live deployment (no restrictions)
kubectl edit deployment <deployment-name>

# Edit a pod — will be rejected for immutable fields, saves to temp file
kubectl edit pod <pod-name>

# Export pod spec to file, edit, delete, recreate
kubectl get pod <pod-name> -o yaml > my-pod.yaml
kubectl delete pod <pod-name>
kubectl create -f my-pod.yaml

# One-command force delete and recreate from file
kubectl replace --force -f my-pod.yaml
```

---

### DaemonSets

```bash
# List DaemonSets
kubectl get daemonsets -n kube-system
kubectl get ds -n kube-system                   # shorthand

# Inspect DaemonSet — shows desired/current/ready/node counts
kubectl describe daemonset <ds-name> -n kube-system

# Check which nodes have the DaemonSet pod running
kubectl get pods -l <ds-label> -o wide -n kube-system

# Generate DaemonSet manifest via Deployment dry-run (no native create ds command)
kubectl create deployment <name> --image=<image> --dry-run=client -o yaml > ds.yaml
# Then edit: change kind to DaemonSet, remove replicas and strategy fields
kubectl apply -f ds.yaml
```

---

### Static Pods

```bash
# Find the static pod manifest directory
ps aux | grep kubelet | grep -E 'pod-manifest-path|config'
grep staticPodPath /var/lib/kubelet/config.yaml

# Generate static pod manifest (fastest method)
kubectl run <pod-name> --image=<image> --dry-run=client -o yaml > \
  /etc/kubernetes/manifests/<pod-name>.yaml

# View static pods (when no apiserver available)
crictl ps

# Verify mirror pod created by kubelet (when part of a cluster)
kubectl get pods -A | grep <pod-name>           # name appears as <pod-name>-<node-name>

# Delete a static pod — remove the manifest file (kubectl delete will not work)
rm /etc/kubernetes/manifests/<pod-name>.yaml

# View control plane static pod manifests
ls /etc/kubernetes/manifests/
```

---

### Priority Classes

```bash
# List all priority classes
kubectl get priorityclass
kubectl get pc                                  # shorthand

# Inspect a priority class
kubectl describe priorityclass <name>

# Check priority class assigned to pods
kubectl get pods -o custom-columns="NAME:.metadata.name,PRIORITY:.spec.priorityClassName"
```

---

### Multiple Schedulers

```bash
# Verify custom scheduler pod is running
kubectl get pods -n kube-system | grep scheduler

# Check which scheduler placed a pod
kubectl get events -o wide | grep Scheduled     # Source column shows scheduler name

# View scheduler logs
kubectl logs -n kube-system <scheduler-pod-name>

# Confirm schedulerName on a pod
kubectl get pod <pod-name> -o jsonpath='{.spec.schedulerName}'
```

---

### Admission Controllers

```bash
# View enabled admission controllers (kubeadm cluster)
kubectl exec -n kube-system kube-apiserver-controlplane -- \
  kube-apiserver -h | grep enable-admission-plugins

# Inspect running apiserver admission flags
kubectl exec -n kube-system kube-apiserver-controlplane -- \
  ps aux | grep admission

# Edit apiserver manifest to enable/disable admission controllers
vi /etc/kubernetes/manifests/kube-apiserver.yaml
# Add flags under command:
#   --enable-admission-plugins=NodeRestriction,NamespaceAutoProvision
#   --disable-admission-plugins=DefaultStorageClass

# List validating webhook configurations
kubectl get validatingwebhookconfigurations
kubectl get validatingwebhookconfigurations -o yaml

# List mutating webhook configurations
kubectl get mutatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations -o yaml
```

---

## 03 — Logging & Monitoring

### Metrics Server

```bash
# Enable on Minikube
minikube addons enable metrics-server

# Deploy on kubeadm / cloud clusters
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify Metrics Server is running
kubectl get deployment -n kube-system metrics-server
kubectl get pods -n kube-system | grep metrics-server

# Node-level CPU and memory
kubectl top nodes

# Pod-level CPU and memory
kubectl top pods
kubectl top pods -n <namespace>

# Sort by CPU or memory
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory

# Show per-container metrics
kubectl top pods --containers
```

---

### Application Logs

```bash
# View logs — single container pod
kubectl logs <pod-name>

# Stream logs live
kubectl logs <pod-name> -f

# Last N lines
kubectl logs <pod-name> --tail=50

# Logs since a duration
kubectl logs <pod-name> --since=1h
kubectl logs <pod-name> --since=5m

# Logs with timestamps
kubectl logs <pod-name> --timestamps

# Multi-container pod — must specify container name
kubectl logs <pod-name> -c <container-name>
kubectl logs <pod-name> -c <container-name> -f

# Logs from a previously terminated/crashed container
kubectl logs <pod-name> -c <container-name> --previous

# Logs from all containers in a pod simultaneously
kubectl logs <pod-name> --all-containers=true
kubectl logs <pod-name> --all-containers=true -f

# Find container names in a pod (if unsure)
kubectl describe pod <pod-name> | grep -i "container"
```

---

## 04 — Application Lifecycle Management

### Rollouts and Rollbacks

```bash
# Check rollout status (blocks until complete — useful in scripts)
kubectl rollout status deployment/<name>

# View revision history
kubectl rollout history deployment/<name>

# View a specific revision's details
kubectl rollout history deployment/<name> --revision=2

# Trigger a rollout — declarative (preferred)
kubectl apply -f deployment.yaml

# Trigger a rollout — imperative (does NOT update local YAML)
kubectl set image deployment/<name> <container>=<image>:<tag>

# Roll back to previous revision
kubectl rollout undo deployment/<name>

# Roll back to a specific revision
kubectl rollout undo deployment/<name> --to-revision=1

# View ReplicaSets to confirm old/new RS state
kubectl get rs

# Inspect deployment events — shows strategy used and scale transitions
kubectl describe deployment <name>
```

---

### Environment Variables

```bash
# Verify env vars are set correctly inside a running container
kubectl exec <pod-name> -- env

# Verify a specific env var
kubectl exec <pod-name> -- env | grep <VAR_NAME>
```

---

### ConfigMaps

```bash
# Create — imperative, inline key-value pairs
kubectl create configmap <name> --from-literal=KEY=value --from-literal=KEY2=value2

# Create — imperative, from file
kubectl create configmap <name> --from-file=<path-to-file>

# Create — declarative
kubectl apply -f configmap.yaml

# List ConfigMaps
kubectl get configmaps
kubectl get cm                                  # shorthand

# Inspect — shows keys and values
kubectl describe configmap <name>

# View full YAML including values
kubectl get configmap <name> -o yaml
```

---

### Secrets

```bash
# Create — imperative (Kubernetes base64-encodes values automatically)
kubectl create secret generic <name> \
  --from-literal=KEY=value \
  --from-literal=KEY2=value2

# Create — from file
kubectl create secret generic <name> --from-file=<path-to-file>

# Create — declarative (YOU must base64-encode values manually)
echo -n 'plaintext' | base64                   # encode
echo -n 'encodedvalue' | base64 --decode        # decode

kubectl apply -f secret.yaml

# List Secrets
kubectl get secrets

# Inspect — shows keys but hides values
kubectl describe secret <name>

# View encoded values
kubectl get secret <name> -o yaml

# Decode a specific value from the YAML output
echo -n '<base64-value>' | base64 --decode
```

---

### Multi-Container Pods / Init Containers

```bash
# Check init container status — shown separately from main containers
kubectl describe pod <pod-name>                 # Init Containers section

# View init container logs
kubectl logs <pod-name> -c <init-container-name>

# View logs of all containers including init
kubectl logs <pod-name> --all-containers=true

# Debug a stuck init container
kubectl logs <pod-name> -c <init-container-name>
kubectl describe pod <pod-name>                 # check Events for failure reason
```

---

### Horizontal Pod Autoscaler (HPA)

```bash
# Create HPA — imperative
kubectl autoscale deployment <name> \
  --cpu-percent=50 \
  --min=1 \
  --max=10

# Create HPA — declarative
kubectl apply -f hpa.yaml

# List HPAs — shows current vs target metrics and replica counts
kubectl get hpa

# Inspect HPA
kubectl describe hpa <name>

# Delete HPA (Deployment keeps running at current replica count)
kubectl delete hpa <name>

# Verify Metrics Server is running (HPA prerequisite)
kubectl get pods -n kube-system | grep metrics-server
```

---

### Vertical Pod Autoscaler (VPA)

```bash
# Deploy VPA components (not built-in — must install separately)
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/latest/download/vertical-pod-autoscaler.yaml

# Verify VPA components are running
kubectl get pods -n kube-system | grep vpa
# Expected: vpa-admission-controller, vpa-recommender, vpa-updater

# Create VPA — declarative only (no imperative command)
kubectl apply -f vpa.yaml

# List VPAs
kubectl get vpa

# View VPA recommendations
kubectl describe vpa <name>

# Delete VPA
kubectl delete vpa <name>
```

---

### base64 Encoding (Secrets)

```bash
# Encode — required for declarative Secret manifests
echo -n 'mysql' | base64                        # bXlzcWw=
echo -n 'root' | base64                         # cm9vdA==
echo -n 'mypassword' | base64                   # bXlwYXNzd29yZA==

# Decode — inspect a Secret value from kubectl get secret -o yaml
echo -n 'bXlzcWw=' | base64 --decode            # mysql
```

---

## 05 — Cluster Maintenance

### OS Upgrades — Node Drain / Cordon / Uncordon

```bash
# Safely evict all pods and cordon a node before maintenance
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Drain with custom grace period and timeout
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data \
  --grace-period=30 --timeout=120s

# Cordon only — mark unschedulable, existing pods stay running
kubectl cordon <node-name>

# Uncordon — mark schedulable again after maintenance
kubectl uncordon <node-name>

# Verify node scheduling status
kubectl get nodes                               # STATUS: Ready,SchedulingDisabled = cordoned

# Check node taints (unschedulable taint applied by cordon/drain)
kubectl describe node <node-name> | grep -i taint

# Check PodDisruptionBudgets before draining (stalled drain = PDB violation)
kubectl get pdb -A
kubectl describe pdb <name> -n <namespace>
```

---

### Cluster Upgrade (kubeadm)

```bash
# --- CONTROL PLANE NODE ---

# Step 1: Switch to new minor version repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' \
  | tee /etc/apt/sources.list.d/kubernetes.list

apt-get update

# Step 2: Upgrade kubeadm
apt-mark unhold kubeadm
apt-get install -y kubeadm=1.36.3-1.1
apt-mark hold kubeadm

# Verify kubeadm version
kubeadm version

# Step 3: Plan the upgrade
kubeadm upgrade plan

# Step 4: Apply upgrade (control plane only)
kubeadm upgrade apply v1.36.3

# Step 5: Upgrade kubelet and kubectl on master
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.36.3-1.1 kubectl=1.36.3-1.1
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet

# Verify master upgraded
kubectl get nodes

# --- WORKER NODES (repeat per node) ---

# Step 1: Drain worker (from control plane)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Step 2: SSH into worker, switch repo and upgrade kubeadm
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' \
  | tee /etc/apt/sources.list.d/kubernetes.list

apt-get update
apt-mark unhold kubeadm
apt-get install -y kubeadm=1.36.3-1.1
apt-mark hold kubeadm

# Step 3: Update node configuration
kubeadm upgrade node

# Step 4: Upgrade kubelet and kubectl on worker
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.36.3-1.1 kubectl=1.36.3-1.1
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet

# Step 5: Uncordon worker (from control plane)
kubectl uncordon <node-name>

# Verify all nodes upgraded
kubectl get nodes
```

---

### etcd Backup / Restore (Cluster Maintenance context)

```bash
# Pre-upgrade snapshot (run before kubeadm upgrade apply)
ETCDCTL_API=3 etcdctl snapshot save /opt/pre-upgrade-$(date +%F).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Find cert paths from etcd static pod manifest
grep -E 'cacert|certfile|keyfile|listen-client' /etc/kubernetes/manifests/etcd.yaml

# Full kubeadm cluster restore sequence
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/      # stop apiserver
etcdutl snapshot restore <snapshot.db> --data-dir=/var/lib/etcd-from-backup
# Edit /etc/kubernetes/manifests/etcd.yaml:
#   --data-dir=/var/lib/etcd-from-backup
#   hostPath.path: /var/lib/etcd-from-backup
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/      # restore apiserver
watch kubectl get pods -n kube-system                       # confirm recovery
```