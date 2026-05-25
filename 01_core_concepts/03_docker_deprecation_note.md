## Section 3: A Note on Docker Deprecation

### What it is

A clarification on what "Docker deprecation" actually means in the context of Kubernetes — what was removed, what wasn't.

---

### What Was Actually Deprecated

Kubernetes deprecated **Docker as a container runtime** — specifically the requirement for the full Docker stack to be present on cluster nodes.

Docker consists of multiple components:

| Component | Status in Kubernetes |
|---|---|
| Docker CLI | Not needed by Kubernetes |
| Docker API | Not needed by Kubernetes |
| Build tools | Not needed by Kubernetes |
| Volumes, auth, security | Not needed by Kubernetes |
| `containerd` (runtime daemon) | ✅ Still used — extracted and used directly via CRI |
| `runc` (OCI runtime) | ✅ Still used — invoked by containerd |

### Docker Build Tool Alternative

| Tool | One-liner |
|---|---|
| **Buildah** | OCI-compliant image build tool that constructs container images without requiring a running Docker daemon — commonly paired with Podman as a daemonless Docker build replacement. |

Kubernetes only ever needed `containerd` from the Docker stack. Once `containerd` became available as a standalone CRI-compatible runtime, the rest of Docker's tooling became unnecessary overhead for Kubernetes — hence the deprecation.

---

### What Was NOT Deprecated

- **Docker itself** — Docker remains the most widely used container tool for local development and CI/CD image builds.
- **Docker-built images** — All images built with Docker follow the OCI image spec and continue to work on any OCI-compliant runtime including `containerd`.
- **Docker as a learning tool** — Using Docker to understand container concepts before applying them in Kubernetes is still valid and common practice.

---

### Practical Note for This Course

Throughout the course, Docker is used in examples to explain container concepts. This is intentional — container fundamentals are best understood with Docker before introducing Kubernetes orchestration on top.

If Docker is not installed on your machine and you are working with `containerd` directly, replace the `docker` command with `nerdctl` — the syntax is nearly identical for most operations.

```bash
# Docker
docker run nginx

# Equivalent with nerdctl (containerd)
nerdctl run nginx
```

---

### Exam Gotchas

- "Docker deprecated" means Docker as a **runtime for Kubernetes nodes** — not Docker as a tool or image format.
- On CKA exam nodes, the runtime is `containerd`. Use `crictl` for runtime-level troubleshooting — not `docker`.
- Docker images on any registry (Docker Hub, ACR, ECR) work without any changes — image format was never deprecated.

---

