## 04 — Commands and Arguments in Kubernetes

### What it is

Kubernetes Pod specs provide two fields — `command` and `args` — that map directly to Docker's `ENTRYPOINT` and `CMD` instructions. They allow you to override container startup behaviour without rebuilding the image.

---

### Docker → Kubernetes Mapping

| Docker Dockerfile | Kubernetes Pod spec | What it overrides |
|---|---|---|
| `ENTRYPOINT ["sleep"]` | `command: ["sleep2.0"]` | The executable itself |
| `CMD ["5"]` | `args: ["10"]` | The arguments passed to the executable |

> **The naming is counterintuitive and a guaranteed exam trap:** the Kubernetes field called `command` overrides Docker's `ENTRYPOINT` — NOT Docker's `CMD`. The field called `args` overrides Docker's `CMD`.

---

### Mental Model

Carry forward the `az` analogy from the previous section:

- `command` = swapping the binary (`az` → `az2`) — you're changing what runs
- `args` = changing the subcommand (`vm list` → `account show`) — you're changing what it does

In practice, `args` is changed far more often than `command`. You rarely need to swap the binary; you almost always need to change what it does.

---

### Pod Definition

**Base image (from previous section):**
```dockerfile
ENTRYPOINT ["sleep"]
CMD ["5"]
```

**Pod using image defaults (sleeps 5s):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
```

**Override args only — sleep 10s instead of 5s:**
```yaml
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
    args: ["10"]          # overrides CMD ["5"] in Dockerfile
```

**Override both command and args — use a different binary:**
```yaml
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
    command: ["sleep2.0"]   # overrides ENTRYPOINT ["sleep"]
    args: ["10"]            # overrides CMD ["5"]
```

---

### When to Use Which

| Scenario | Use |
|---|---|
| Change arguments to the existing binary | `args` only |
| Swap the executable entirely | `command` only |
| Swap executable and change its arguments | Both `command` and `args` |
| Use image defaults unchanged | Neither — omit both fields |

---

### Real-World Usage

**1. Multi-environment configuration without image rebuilds**
A single application image is promoted across dev → staging → prod. Each environment's Pod spec passes different `args` (e.g., `["--config", "/etc/app/staging.yaml"]`) while `command` stays fixed — no image rebuild needed per environment.

**2. Init containers running one-shot tasks**
Init containers frequently use `command` + `args` to run database migrations, config validation, or secret fetching before the main container starts:
```yaml
initContainers:
- name: db-migrate
  image: myapp
  command: ["python"]
  args: ["manage.py", "migrate", "--noinput"]
```

**3. Debugging a CrashLoopBackOff pod**
When a Pod is crash-looping, override `command` to get a shell instead of running the broken entrypoint:
```yaml
command: ["sh"]
args: ["-c", "sleep 3600"]   # keep container alive to exec into it
```
Then `kubectl exec -it <pod> -- sh` to inspect the filesystem, config, or environment.

**4. Kubernetes Jobs**
`command` and `args` are the primary mechanism for parameterising batch Job Pods — each Job run passes different `args` to the same image (e.g., processing a different date range or shard).

**5. ArgoCD / GitOps**
In a GitOps workflow, `args` values in the Pod spec YAML are committed to Git. Changing a config value means a Git commit → ArgoCD sync → rolling update. The image never changes; only the args do. This is the standard pattern for config-driven deployments without ConfigMaps.

---

### Exam Gotchas

- **`command` ≠ CMD.** `command` overrides `ENTRYPOINT`. `args` overrides `CMD`. This is the single most common mistake in this topic — the naming is deliberately misleading.
- **Both fields take array format** — not a plain string:
  ```yaml
  args: ["10"]        # correct
  args: "10"          # wrong — will error
  ```
- **If you specify `command` without `args`**, the container runs the new executable with no arguments — the original `CMD` from the Dockerfile is also discarded when `command` is set.
- **Omitting both fields** means the image's `ENTRYPOINT` and `CMD` run unchanged — this is valid and common.
- In the exam, when asked to modify what a container runs, read carefully whether the question wants you to change the executable (`command`) or just the parameters (`args`).

---

### In Managed Clusters (AKS)

Behaviour is identical to self-managed clusters. AKS does not restrict or modify `command` / `args` fields. Azure Policy can be used to enforce or block specific command overrides if required by organisational policy.

---

