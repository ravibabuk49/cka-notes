## 11 — Pods

### What it is

A **Pod** is the smallest deployable unit in Kubernetes. Kubernetes does not deploy containers directly onto worker nodes — containers are always encapsulated inside a pod. A pod represents a single instance of a running application.

---

### Pod-to-Container Relationship

Pods have a **1-to-1 relationship with application containers** in the typical case — one container per pod.

| Operation | How it is done |
|---|---|
| Scale up | Create a **new pod** with a new instance of the application |
| Scale down | Delete an existing pod |
| Add capacity beyond a node | Deploy new pods on a **new node** in the cluster |

> You do not add containers to an existing pod to scale. You create new pods.

---

### Multi-Container Pods

A pod *can* contain multiple containers, but they must not be multiple instances of the same application — those go in separate pods. The valid use case is a **helper container** that performs a supporting task alongside the main application container.

**Example:** A web application container paired with a sidecar container that processes uploaded files or fetches data on behalf of the main app.

#### Why group them in the same pod?

Without pods, running tightly coupled containers on plain Docker requires manual coordination:
- Manually link containers over a custom network
- Create and mount shared volumes between them
- Monitor the app container and manually kill the helper when the app dies
- Redeploy the helper every time the app container is redeployed

Kubernetes handles all of this automatically when containers are defined within the same pod:

| Shared resource | Behaviour |
|---|---|
| Network namespace | All containers in the pod share the same IP and port space — they communicate via `localhost` |
| Storage | All containers have access to the same pod volumes |
| Lifecycle | All containers are created together and destroyed together |

> Multi-container pods are a **rare use case**. The vast majority of pods in production run a single container.

---

### Basic kubectl Commands

```bash
# Create a pod running nginx (imperative)
kubectl run nginx --image=nginx

# List all pods in the current namespace
kubectl get pods

# List pods with additional detail (node assignment, IP)
kubectl get pods -o wide

# Describe a pod — events, status, container details
kubectl describe pod nginx

# Delete a pod
kubectl delete pod nginx
```

**Example output of `kubectl get pods`:**

```
NAME    READY   STATUS              RESTARTS   AGE
nginx   0/1     ContainerCreating   0          3s
nginx   1/1     Running             0          6s
```
> `Ready` column means "Ready containers in pod/Total containers in the pod."
> `ContainerCreating` means the image is being pulled. `Running` means the container is live.

---

### How kubectl run Works

```
kubectl run nginx --image=nginx
         │
         ▼
kube-apiserver creates a Pod object in etcd
         │
         ▼
kube-scheduler assigns the pod to a node
         │
         ▼
kubelet on the target node instructs containerd to pull nginx from Docker Hub
         │
         ▼
Container starts → pod status transitions to Running
```

The image is pulled from **Docker Hub** by default. A private registry can be configured — covered in the Security section.

---

### What a Pod Does NOT Do (at this stage)

A running pod is not yet accessible to external users. The nginx web server inside the pod is reachable only from within the node itself. Exposing the application externally requires a **Service** — covered in the Networking and Services lecture.

---

### Exam Gotchas

- The smallest object you can create in Kubernetes is a **pod** — not a container. Kubernetes has no concept of a standalone container.
- Scaling means creating new pods — **never** adding containers to an existing pod.
- All containers in a pod share the same **network namespace** — they communicate via `localhost` and cannot bind to the same port.
- All containers in a pod share the same **lifecycle** — if the pod is deleted, all containers in it are terminated simultaneously.
- `kubectl run` creates a pod directly — it does not create a Deployment unless `--restart=Always` is specified in older versions. In current Kubernetes, use `kubectl create deployment` for managed workloads.

---

### In Managed Clusters (AKS / EKS / GKE)

> Pod behaviour is identical across self-managed and managed clusters — no meaningful difference at this level. The pod abstraction is fundamental Kubernetes and fully portable.

---

