## Self-Healing Applications

### What it is

Kubernetes provides self-healing at two levels:

1. **ReplicaSet / ReplicationController** — ensures the desired number of Pod replicas is always running. If a Pod crashes or is deleted, the ReplicaSet automatically creates a replacement.
2. **Liveness and Readiness Probes** — allow Kubernetes to check the health of a running application inside a Pod and take action (restart container, remove from Service endpoints) if it becomes unhealthy.

> Liveness and Readiness Probes are **not in scope for the CKA exam** — they are CKAD topics. ReplicaSet-based self-healing is covered in `01_core_concepts`.

---

### Liveness and Readiness Probes — High Level

> ⚠️ Not required for CKA — CKAD scope only. Included here for awareness.

Both probes are defined per container in the Pod spec and are executed by the Kubelet on the node.

**Liveness Probe**
Answers: *is this container still alive and worth keeping running?*
If the liveness probe fails, Kubelet kills the container and restarts it according to the Pod's `restartPolicy`. Used to recover from deadlocks or frozen application states that don't crash the process but render it non-functional.

**Readiness Probe**
Answers: *is this container ready to receive traffic?*
If the readiness probe fails, Kubernetes removes the Pod's IP from the Endpoints of any matching Service — traffic stops being routed to it, but the container is not restarted. Used to prevent traffic from hitting a Pod that is still warming up (loading cache, waiting for a dependency) or temporarily overwhelmed.

| Probe | Failure action | Restart? |
|---|---|---|
| Liveness | Kill + restart container | Yes |
| Readiness | Remove from Service endpoints | No |

Both support three check mechanisms: HTTP GET, TCP socket, and exec (run a command inside the container).

---

### Exam Gotchas

- Self-healing via ReplicaSet is automatic — no manual intervention needed when a Pod crashes, as long as the Pod is managed by a controller (ReplicaSet, Deployment, StatefulSet).
- Pods created directly with `kubectl run` or a bare Pod manifest (no controller) are **not** self-healed — if the Pod dies, it stays dead.
- The CKA exam tests your ability to identify whether a Pod is controller-managed and why a crashed Pod may or may not restart.

