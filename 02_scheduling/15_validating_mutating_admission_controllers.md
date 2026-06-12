## 15 — Validating & Mutating Admission Controllers

### What it is
Two sub-types of admission controllers that differ in **what they do to a request**:

- **Validating Admission Controller** — inspects the request and either allows or rejects it. It cannot change the request.
- **Mutating Admission Controller** — can modify the request before it is persisted. It changes the object itself.
- Some controllers do **both** — mutate first, then validate the result.

---

### Validating vs Mutating — Side by Side

| | Validating | Mutating |
|---|---|---|
| **What it does** | Allows or rejects the request as-is | Modifies the request before it is persisted |
| **Can change the object?** | ❌ No | ✅ Yes |
| **Example** | `NamespaceLifecycle` — rejects requests to non-existent namespaces | `DefaultStorageClass` — injects the default storage class into PVCs that don't specify one |
| **Invocation order** | Second | First |

---

### Why Mutating Runs Before Validating

Mutating controllers always run **before** validating controllers. The reason is logical:

```
Request arrives
      │
      ▼
Mutating controllers run first
      │   e.g. NamespaceAutoProvision creates the namespace
      ▼
Validating controllers run second
      │   e.g. NamespaceExists checks if namespace exists — now it does ✅
      ▼
Request allowed
```

If validating ran first, it would reject the request before the mutating controller had a chance to fix it. For example, `NamespaceExists` would always reject requests for namespaces that don't exist — and `NamespaceAutoProvision` would never get the chance to create them.

> If **any** admission controller in the chain rejects the request, the entire request is rejected and an error is returned to the user immediately.

---

### Webhook Admission Controllers

Built-in admission controllers are compiled into Kubernetes source code. For **custom logic**, Kubernetes provides two special webhook-based controllers:

| Webhook Type | Kind |
|---|---|
| `MutatingAdmissionWebhook` | `MutatingWebhookConfiguration` |
| `ValidatingAdmissionWebhook` | `ValidatingWebhookConfiguration` |

These webhooks point to an **external server** (your own code) that Kubernetes calls at admission time. The server receives the request, applies its logic, and tells Kubernetes whether to allow, reject, or modify it.

---

### How a Webhook Admission Controller Works

```
API request arrives → AuthN → AuthZ → Built-in Admission Controllers
                                                    │
                                                    ▼
                                        Webhook fires → API server sends
                                        AdmissionReview object (JSON) to
                                        your webhook server
                                                    │
                                                    ▼
                                        Webhook server responds with
                                        AdmissionReview (allowed: true/false)
                                        + optional patch (for mutating)
                                                    │
                                          ┌─────────┴─────────┐
                                          ▼                   ▼
                                     allowed: true       allowed: false
                                     Object persisted    Request rejected
                                     to etcd             with error message
```

---

### Setting Up a Custom Webhook — Step by Step

#### Step 1 — Build and Deploy the Webhook Server

The webhook server is any HTTP server that:
- Accepts `POST` requests at `/validate` and/or `/mutate` endpoints
- Receives an `AdmissionReview` JSON object
- Responds with an `AdmissionReview` JSON object with `allowed: true` or `allowed: false`

It can be written in any language (Go, Python, etc.) and deployed as a Kubernetes Deployment with a Service:

```bash
# After building and pushing the image
kubectl apply -f webhook-deployment.yaml
kubectl apply -f webhook-service.yaml

# Verify it is running
kubectl get pods -n webhook-demo
kubectl get svc -n webhook-demo
```

> From an exam perspective — you will **not** be asked to write webhook server code. You need to understand the flow and be able to configure the webhook object.

---

#### Step 2 — Create the Webhook Configuration Object

**Validating Webhook Configuration:**

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: my-validating-webhook
webhooks:
  - name: validate.example.com
    clientConfig:
      # Option A — webhook server deployed inside the cluster as a Service
      service:
        namespace: webhook-demo
        name: webhook-service
        path: /validate
      # TLS — the API server must trust the webhook server's certificate
      caBundle: <base64-encoded-CA-certificate>
      # Option B — webhook server hosted externally
      # url: "https://my-external-webhook.company.com/validate"
    rules:
      - apiGroups:   [""]
        apiVersions: ["v1"]
        operations:  ["CREATE"]        # only fire on Pod CREATE requests
        resources:   ["pods"]
    admissionReviewVersions: ["v1"]
    sideEffects: None
```

**Mutating Webhook Configuration** — identical structure, different `kind`:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: my-mutating-webhook
webhooks:
  - name: mutate.example.com
    clientConfig:
      service:
        namespace: webhook-demo
        name: webhook-service
        path: /mutate
      caBundle: <base64-encoded-CA-certificate>
    rules:
      - apiGroups:   [""]
        apiVersions: ["v1"]
        operations:  ["CREATE", "UPDATE"]
        resources:   ["pods"]
    admissionReviewVersions: ["v1"]
    sideEffects: None
```

---

### AdmissionReview — Request and Response

**What the API server sends to your webhook server:**

```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "request": {
    "uid": "705ab4f5-6393-11e8-b7cc-42010a800002",
    "kind": { "group": "", "version": "v1", "kind": "Pod" },
    "operation": "CREATE",
    "userInfo": { "username": "ravi", "groups": ["developers"] },
    "object": { ... }       // full Pod spec
  }
}
```

**What your webhook server must respond with:**

```json
// Allow
{ "response": { "uid": "<same-uid>", "allowed": true } }

// Reject with message
{ "response": { "uid": "<same-uid>", "allowed": false,
    "status": { "message": "Images must be from registry.company.internal" } } }

// Mutate — allow with a patch (mutating only)
{ "response": { "uid": "<same-uid>", "allowed": true,
    "patchType": "JSONPatch",
    "patch": "<base64-encoded-JSON-patch>" } }
```

**JSON Patch operations (for mutating webhooks):**

| Operation | What it does |
|---|---|
| `add` | Add a new field or value |
| `remove` | Remove an existing field |
| `replace` | Replace an existing field value |
| `move` | Move a field to a new path |
| `copy` | Copy a field to a new path |

---

### TLS Requirement

Communication between the kube-apiserver and the webhook server **must be over TLS**. The webhook server needs:

1. A TLS certificate and private key
2. The CA certificate that signed it, base64-encoded and placed in `clientConfig.caBundle`

```bash
# Base64 encode the CA cert for the caBundle field
cat ca.crt | base64 -w 0
```

---

### Real-World Use Cases

#### Use Case 1 — Auto-inject a Sidecar Container (Mutating)
Service mesh tools like **Istio** and **Linkerd** use a mutating webhook to automatically inject a proxy sidecar container into every Pod — without developers needing to add it manually to their manifests. The webhook intercepts every Pod CREATE request and patches in the sidecar container spec before the Pod is persisted.

```
Developer applies pod.yaml (1 container)
      │
      ▼
MutatingWebhookConfiguration fires
      │
      ▼
Istio webhook adds istio-proxy sidecar container
      │
      ▼
Pod created with 2 containers (app + istio-proxy) ✅
```

#### Use Case 2 — Block Privileged Containers (Validating)
A security team deploys a validating webhook that rejects any Pod where a container has `securityContext.privileged: true` or `securityContext.runAsUser: 0` (root). This enforces a security baseline that RBAC cannot express — it does not matter who creates the Pod, the content policy applies universally.

---

### Exam Gotchas

- **Mutating always runs before Validating** — this order is fixed by Kubernetes, not configurable. If an exam question asks why a validating controller is not catching something, check if a mutating controller is changing the request before validation.
- **`caBundle` is mandatory** — omitting it causes the webhook to fail with a TLS error. The API server cannot communicate with the webhook server without trusting its certificate.
- **`rules` scopes when the webhook fires** — a webhook without rules fires on every single API request. Always scope it to specific operations and resources to avoid impacting cluster performance.
- **`sideEffects: None`** — required for webhooks that do not produce side effects (most webhooks). Omitting it causes a deprecation warning in newer Kubernetes versions.
- **`failurePolicy`** — not covered in the transcript but exam-relevant. Defaults to `Fail` — if the webhook server is unreachable, the request is rejected. Set to `Ignore` if you want the request to proceed when the webhook is unavailable:
```yaml
failurePolicy: Ignore    # request allowed if webhook server is down
failurePolicy: Fail      # request rejected if webhook server is down (default)
```
- **API version** — `admissionregistration.k8s.io/v1`. Using `v1beta1` is deprecated and will be rejected on newer clusters.

---

### In Managed Clusters (AKS / EKS / GKE)

- AKS fully supports both `MutatingWebhookConfiguration` and `ValidatingWebhookConfiguration` — deploy the webhook server as a Deployment in any namespace and create the configuration object normally.
- **Azure Policy for Kubernetes (Gatekeeper/OPA)** is AKS's managed validating webhook — it runs as a Deployment in `gatekeeper-system` and enforces policy as code across the cluster. Under the hood it is a `ValidatingWebhookConfiguration` registered against the apiserver.
- **Istio on AKS** uses a `MutatingWebhookConfiguration` to inject the Envoy sidecar — you can inspect it with `kubectl get mutatingwebhookconfigurations`.

```bash
# Inspect all webhook configurations in a cluster
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations
```

---

