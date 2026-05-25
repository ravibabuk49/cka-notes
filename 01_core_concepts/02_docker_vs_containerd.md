## Section 2: Docker vs containerd

### What it is

This section explains the historical evolution of container runtimes in Kubernetes — why Docker was removed, what containerd is, and which CLI tools to use when working with containerd-based clusters.

---

### Historical Context — How We Got Here

**Phase 1 — Tight coupling (early Kubernetes):**
Kubernetes was originally built to orchestrate Docker specifically. Docker was the dominant container tool due to its user experience, and Kubernetes only supported Docker as its container runtime.

**Phase 2 — CRI introduced:**
As Kubernetes grew, other runtimes (e.g., Rocket/rkt) needed support. Kubernetes introduced the **Container Runtime Interface (CRI)** — a standard interface that allows any OCI-compliant container runtime to integrate with Kubernetes without requiring changes to Kubernetes core code.

**OCI — Open Container Initiative** defines two specifications:
- **Image spec** — how a container image must be built
- **Runtime spec** — how a container runtime must behave

Any runtime that adheres to OCI standards can be used with Kubernetes via CRI.

**Phase 3 — Dockershim (the hack):**
Docker predated CRI and was not built to implement it. To maintain Docker support, Kubernetes introduced **Dockershim** — a compatibility shim that allowed Docker to work outside of the CRI standard. This was explicitly a temporary and hacky solution.

**Phase 4 — Dockershim removed (K8s 1.24):**
Maintaining Dockershim added unnecessary complexity. In **Kubernetes v1.24**, Dockershim was permanently removed and Docker support was dropped as a container runtime.

> Docker images continue to work because Docker follows the OCI image spec. The runtime was removed — not image compatibility.

---

### What Docker Actually Is

Docker is not just a container runtime. It is a collection of tools:

| Docker component | Description |
|---|---|
| Docker CLI | Command-line interface for user interaction |
| Docker API | REST API for programmatic access |
| Build tools | Tools for building container images |
| Volumes & security | Storage and security management features |
| `containerd` | The actual container runtime daemon within Docker |
| `runc` | The low-level OCI runtime that containerd uses to run containers |

`containerd` was always embedded inside Docker. It is CRI-compatible and can be extracted and used as a standalone runtime — which is exactly what Kubernetes does from v1.24 onwards.

`containerd` is now a standalone CNCF **graduated** project and can be installed independently without Docker.

---

### CLI Tools Comparison

Three CLI tools exist in the containerd ecosystem — understanding which to use and when is critical:

| Tool | Maintained by | Works with | Purpose | Use in production? |
|---|---|---|---|---|
| `ctr` | containerd community | containerd only | Low-level debugging of containerd | ❌ No — limited features, not user-friendly |
| `nerdctl` | containerd community | containerd only | Docker-like general purpose CLI for containerd | ✅ Yes — replaces Docker CLI |
| `crictl` | Kubernetes community | All CRI-compatible runtimes | Inspect and debug CRI runtimes from the Kubernetes perspective | ⚠️ Debugging only — not for creating containers |

---

### `ctr` — containerd CLI

Installed automatically with containerd. Intended strictly for debugging containerd internals. Not recommended for general use.

```bash
# Pull an image
ctr images pull docker.io/library/redis:latest

# Run a container
ctr run docker.io/library/redis:latest redis
```

---

### `nerdctl` — Docker-like CLI for containerd

The recommended general-purpose CLI when working with containerd directly. Syntax is nearly identical to Docker — replace `docker` with `nerdctl`.

```bash
# Run a container
nerdctl run nginx

# Run with port mapping
nerdctl run -p 80:80 nginx

# Pull an image
nerdctl pull redis
```

Additional capabilities beyond Docker CLI:
- Encrypted container images
- Lazy pulling of images
- P2P image distribution
- Image signing and verification
- Kubernetes namespaces support

---

### `crictl` — Kubernetes CRI Debugging Tool

Developed and maintained by the Kubernetes community. Works across **all CRI-compatible runtimes** (containerd, CRI-O, etc.) — not containerd-specific.

Used to inspect and debug container runtimes from the Kubernetes perspective. Must be installed separately.

```bash
# List running containers
crictl ps

# Execute a command inside a container
crictl exec -it <container-id> sh

# View container logs
crictl logs <container-id>

# Pull an image
crictl pull nginx

# List pods
crictl pods

# List ports (unique to crictl — not available in Docker)
crictl ports <container-id>
```

> **Critical:** Do not use `crictl` to create containers in a running cluster. `kubelet` manages the desired state of pods on the node. Any container created via `crictl` outside of `kubelet`'s knowledge will be detected and deleted by `kubelet`.

**Configuring the runtime endpoint** (if multiple runtimes are configured):

By default, `crictl` attempts to connect to endpoints in this order:
1. `dockershim` (legacy)
2. `containerd`
3. `CRI-O`
4. `cri-dockerd`

To explicitly set the endpoint:

```bash
# Via flag
crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps

# Via environment variable
export CONTAINER_RUNTIME_ENDPOINT=unix:///run/containerd/containerd.sock
```

---

### Summary Table

| | `ctr` | `nerdctl` | `crictl` |
|---|---|---|---|
| Maintained by | containerd community | containerd community | Kubernetes community |
| Works with | containerd only | containerd only | All CRI runtimes |
| Primary use | Debugging | General purpose | Debugging / inspection |
| Docker-like syntax | ❌ | ✅ | Mostly |
| Safe to create containers | ❌ | ✅ | ❌ |

---

### Other Container Runtimes — Context

| Runtime | Category | One-liner |
|---|---|---|
| **Podman** | OCI app container runtime | Daemonless, rootless Docker-compatible container engine; integrates with Kubernetes via CRI-O as its CRI-compatible runtime layer. |
| **LXC / LXD** | OS-level container runtime | Provides full Linux system containers (not OCI app containers); not CRI-compatible and not used as a Kubernetes runtime. |

> **LXC/LXD vs Docker/containerd/Podman:** LXC/LXD runs an entire OS userspace per container (closer to a lightweight VM you SSH into). Docker, containerd, and Podman run isolated application processes sharing the host kernel — these are what Kubernetes manages.

### When You Would Actually Use Each

| Scenario | Use |
|---|---|
| Running a microservice in Kubernetes | containerd / Podman via CRI |
| Isolating a full dev environment with multiple services | LXD |
| Running a legacy monolithic app that expects a full OS | LXD |
| CI/CD pipeline building and running OCI images | Docker / Podman + Buildah |
| Lightweight VM alternative for infrastructure isolation | LXD with `--vm` |

---

### Exam Gotchas

- In older clusters (pre-1.24), `docker` commands were used to troubleshoot containers on worker nodes. In current clusters, use `crictl` instead.
- `crictl` is the correct tool for CKA troubleshooting tasks involving containers on nodes — not `docker`, not `nerdctl`.
- Docker images built before v1.24 still work — OCI image spec compliance is what matters, not the runtime that built them.
- If `crictl` shows containers that you don't expect, `kubelet` reconciliation may have already deleted them.

---

