## 06 — Taints & Tolerations vs Node Affinity

### What it is
A conceptual comparison of two node-targeting mechanisms and the pattern of combining them. Taints & Tolerations and Node Affinity each solve only half of the node-dedication problem — using them together is the only way to fully dedicate a node to specific Pods.

---

### The Problem

You share a Kubernetes cluster with other teams. You have three nodes and three Pods — each colour-matched (blue, red, green). The goal:

- Your Pods must land **only on your nodes**
- Other teams' Pods must **never land on your nodes**

---

### Attempt 1 — Taints & Tolerations Only

**What you do:**
- Taint each node with its colour (`color=blue:NoSchedule`, etc.)
- Add a matching toleration to each Pod

**Result:**

```
blue node   → accepts only blue Pod   ✅
red node    → accepts only red Pod    ✅
green node  → accepts only green Pod  ✅

BUT:
red Pod → may still land on an untainted node (other team's node) ❌
```

**Why it fails half the requirement:**
Taints & Tolerations tell a node to **reject** Pods without the right toleration. They do not tell a Pod **where to go**. A tolerated Pod is free to land on any untainted node in the cluster — including other teams' nodes.

---

### Attempt 2 — Node Affinity Only

**What you do:**
- Label each node with its colour (`color=blue`, etc.)
- Add `nodeAffinity` rules to each Pod to target the matching node

**Result:**

```
blue Pod    → lands on blue node      ✅
red Pod     → lands on red node       ✅
green Pod   → lands on green node     ✅

BUT:
other teams' Pods → may still land on your nodes ❌
```

**Why it fails half the requirement:**
Node Affinity tells a Pod **where to go**. It does not stop other Pods from landing on the same node. A node with no taint is open to any Pod the scheduler sends its way.

---

### The Gap Each Mechanism Leaves

| Mechanism | Prevents others' Pods on your nodes | Prevents your Pods on others' nodes |
|---|---|---|
| Taints & Tolerations | ✅ | ❌ |
| Node Affinity | ❌ | ✅ |
| **Both combined** | ✅ | ✅ |

---

### The Solution — Combine Both

**Step 1 — Taints & Tolerations** (node protection)
Apply a taint to each node and a matching toleration to each Pod:

```bash
kubectl taint nodes blue-node  color=blue:NoSchedule
kubectl taint nodes red-node   color=red:NoSchedule
kubectl taint nodes green-node color=green:NoSchedule
```

```yaml
# blue Pod toleration
tolerations:
  - key: "color"
    operator: "Equal"
    value: "blue"
    effect: "NoSchedule"
```

This stops other teams' Pods from landing on your nodes.

---

**Step 2 — Node Affinity** (Pod targeting)
Label each node and add affinity rules to each Pod:

```bash
kubectl label nodes blue-node  color=blue
kubectl label nodes red-node   color=red
kubectl label nodes green-node color=green
```

```yaml
# blue Pod affinity
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: color
              operator: In
              values:
                - blue
```

This stops your Pods from drifting onto other teams' nodes.

---

### Combined Pod Spec

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: blue-pod
spec:
  tolerations:
    - key: "color"
      operator: "Equal"
      value: "blue"
      effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: color
                operator: In
                values:
                  - blue
  containers:
    - name: app
      image: blue-app:v1
```

---

### Mental Model

> **Taints & Tolerations = a door with a lock.**
> The lock (taint) keeps strangers out. But it does not stop the residents (tolerated Pods) from wandering into someone else's house.

> **Node Affinity = a home address on the Pod.**
> The Pod knows exactly where to go. But having an address does not prevent strangers from entering your house.

> **Combined = a lock on your door AND an address on your Pod.**
> Strangers cannot enter. Your Pods go exactly where they should. Full dedication achieved.

---

### Exam Gotchas

- **Toleration ≠ attraction** — a Pod with a toleration for a tainted node is *permitted* on that node, not *directed* to it. Without Node Affinity, it can land anywhere.
- **Node Affinity ≠ protection** — affinity directs your Pod to the right node but leaves the node open to all other Pods. Without a taint, other Pods can still land there.
- **The combination is the only complete solution** — any exam question asking you to "fully dedicate nodes to specific Pods in a shared cluster" requires both mechanisms.
- **Taint effect matters** — use `NoSchedule` when existing Pods should not be disturbed. Use `NoExecute` if you also want to evict any Pods already running on the node that don't belong there.

---

