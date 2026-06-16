## 03 — Commands and Arguments in Docker

### What it is

Docker containers run a single process. That process is defined at image build time via `CMD` and/or `ENTRYPOINT` in the Dockerfile. Understanding how these two instructions interact is prerequisite knowledge for configuring `command` and `args` in Kubernetes Pod specs (covered in the next section).

---

### Container Lifecycle Basics

A container lives only as long as its main process is alive. When the process exits, the container exits. There is no background OS keeping it alive — unlike a VM.

```bash
docker run ubuntu        # exits immediately — bash starts, finds no terminal, exits
docker run nginx         # stays running — nginx process keeps running
```

---

### CMD vs ENTRYPOINT

> **Mental model:** Think of `ENTRYPOINT` as the CLI binary (`az`) and `CMD` as the default subcommand. Running `az` with no subcommand gives you the default behaviour — but pass `vm list` and that default is replaced. The binary (`az`) never changes unless you explicitly swap it with `--entrypoint`.

| Dockerfile Instruction | Purpose | Behaviour when `docker run` passes extra args |
|---|---|---|
| `CMD` | Defines the default command (executable + args) | Entire CMD is **replaced** by the runtime args |
| `ENTRYPOINT` | Defines the fixed executable | Runtime args are **appended** to ENTRYPOINT |
| `ENTRYPOINT` + `CMD` together | ENTRYPOINT = executable, CMD = default args | CMD is replaced if args are passed; ENTRYPOINT stays fixed |

---

### CMD — Default Command

```dockerfile
FROM ubuntu
CMD ["sleep", "5"]       # JSON array format — required when combining with ENTRYPOINT
```

```bash
docker run ubuntu-sleeper          # runs: sleep 5
docker run ubuntu-sleeper sleep 10 # runs: sleep 10  (CMD is fully replaced)
```

> Always use **JSON array format** (`["executable", "arg"]`) when combining `CMD` with `ENTRYPOINT`. Shell form (`CMD sleep 5`) works standalone but cannot be combined correctly.

---

### ENTRYPOINT — Fixed Executable

```dockerfile
FROM ubuntu
ENTRYPOINT ["sleep"]
```

```bash
docker run ubuntu-sleeper 10       # runs: sleep 10  (10 appended to entrypoint)
docker run ubuntu-sleeper          # runs: sleep      → ERROR: missing operand
```

---

### ENTRYPOINT + CMD — Fixed Executable with Default Args

```dockerfile
FROM ubuntu
ENTRYPOINT ["sleep"]
CMD ["5"]              # default arg — used only if nothing passed at runtime
```

```bash
docker run ubuntu-sleeper          # runs: sleep 5   (CMD default used)
docker run ubuntu-sleeper 10       # runs: sleep 10  (CMD overridden by runtime arg)
```

---

### Overriding ENTRYPOINT at Runtime

```bash
docker run --entrypoint sleep2.0 ubuntu-sleeper 10
# runs: sleep2.0 10
```

---

### Shell Form vs JSON Array Form

```dockerfile
# Shell form — runs via /bin/sh -c, problematic with ENTRYPOINT+CMD combo
CMD sleep 5

# JSON array form — exec form, no shell wrapper, correct behaviour
CMD ["sleep", "5"]
```

> ⚠️ In JSON array form, the **first element must be the executable** — never combine command + args into one string element: `["sleep 5"]` is wrong; `["sleep", "5"]` is correct.

---

### Real-World Usage

**1. Sidecar and init containers with fixed tools**
Images like `busybox`, `curl`, or custom CLI tools are built with `ENTRYPOINT` set to the binary. In Kubernetes, these are used as init containers or sidecars where the tool is fixed but the arguments vary per deployment — e.g., a database migration container where the binary is always `flyway` but the target schema is passed as an arg.

**2. Application containers with environment-specific defaults**
A backend API image might be built as:
```dockerfile
ENTRYPOINT ["python", "app.py"]
CMD ["--env", "production"]
```
In staging, the Kubernetes Pod spec overrides `CMD` via `args: ["--env", "staging"]` without rebuilding the image. The same image runs in all environments — only the args change.

**3. Debugging a running container's entrypoint**
When a container exits immediately in production and you need to inspect it:
```bash
docker run --entrypoint sh myimage        # override entrypoint to get a shell
docker run --entrypoint cat myimage /etc/config.yaml   # read a file and exit
```
This is the same pattern used with `kubectl debug` in Kubernetes when overriding the command to troubleshoot a crashing Pod.

**4. CI/CD pipeline images**
Tools like `terraform`, `kubectl`, `helm` are packaged as Docker images with `ENTRYPOINT` set to the binary. In GitHub Actions or Azure Pipelines, these images are pulled and `CMD` (or Kubernetes `args`) passes the subcommand — e.g., `["apply", "-auto-approve"]` — keeping the image generic and reusable across pipeline stages.

**5. Health-check and one-shot job containers**
CronJob or Job Pods in Kubernetes commonly use images where `ENTRYPOINT` is a script (`entrypoint.sh`) that handles signal trapping and cleanup, and `CMD` passes the actual job-specific command. This pattern ensures graceful shutdown regardless of what the job does.

---



- `CMD` is fully replaced at runtime; `ENTRYPOINT` is not — this distinction maps directly to `args` vs `command` in Kubernetes (next section).
- JSON array format is mandatory when using `ENTRYPOINT` + `CMD` together; shell form breaks the append behaviour.
- First element of the JSON array must be the executable binary — not a combined string.
- A container that exits immediately is almost always caused by the main process finishing (or not finding what it needs, e.g., bash with no terminal).

---

