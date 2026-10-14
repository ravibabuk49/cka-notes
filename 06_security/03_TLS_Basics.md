## 03 — TLS Basics (Prerequisite)

### What it is

TLS (Transport Layer Security) is the protocol that secures communication between two parties by providing three guarantees:

| Guarantee | What it means |
|---|---|
| **Confidentiality** | Data is encrypted — intercepted traffic is unreadable |
| **Authentication** | Each party can verify the other is who they claim to be |
| **Integrity** | Data cannot be tampered with in transit without detection |

Certificates are the mechanism that delivers authentication within TLS. Understanding how they work is a prerequisite for every Kubernetes PKI topic that follows in this domain.

---

### Symmetric Encryption

A single shared key is used to both encrypt and decrypt data.

```
plaintext + key → ciphertext     (encryption)
ciphertext + key → plaintext     (decryption)
```

**The key exchange problem:** the shared key must be transmitted to the other party before they can decrypt anything. If it is sent over the same network as the data, an attacker who intercepts the key can decrypt everything. Symmetric encryption is fast and efficient for bulk data transfer, but cannot safely bootstrap its own key exchange over an untrusted network.

---

### Asymmetric Encryption

A mathematically linked key pair is used instead of a single shared key:

| Key | Visibility | Role |
|---|---|---|
| **Private key** | Never shared — kept secret by the owner | Decrypt data encrypted with the public key; sign data |
| **Public key** | Freely distributed | Encrypt data for the private key holder; verify signatures |

**Critical property:** you can encrypt with either key, but you can only decrypt with the other. Encrypt with the public key → only the private key can decrypt. Encrypt with the private key → only the public key can decrypt (this is the basis of digital signatures).

**SSH key pair — the simplest example:**
```bash
ssh-keygen -t rsa -b 4096
# Creates: ~/.ssh/id_rsa       (private key — never share)
#          ~/.ssh/id_rsa.pub   (public key — copy to servers)
```
The public key is placed in `~/.ssh/authorized_keys` on the target server. The private key never leaves the client. Only someone holding the private key can complete the SSH handshake.

---

### How TLS Combines Both

TLS uses asymmetric encryption to solve the key exchange problem, then switches to fast symmetric encryption for the session:

```
1. Server sends its certificate (containing its public key)
2. Client verifies the certificate is signed by a trusted CA
3. Client generates a random symmetric session key
4. Client encrypts the session key with the server's public key
5. Client sends the encrypted session key to the server
6. Server decrypts with its private key → recovers the session key
7. Both parties now share the session key → all further data uses symmetric encryption
```

The attacker intercepting traffic sees the certificate (public — not a secret) and the encrypted session key (useless without the server's private key). Once the handshake is complete, all data is encrypted with the session key which the attacker never obtained.

---

### Certificates — What They Are and What They Contain

A certificate is a digitally signed document that binds a public key to an identity. When a server presents its certificate, you can verify:
- **Who it belongs to** — the Subject field (Common Name / CN)
- **What names it is valid for** — Subject Alternative Names (SANs)
- **Who vouches for it** — the Issuer (the CA that signed it)
- **Whether it has expired** — the validity period
- **The public key** — the key you use to encrypt the session key

```
Subject:    CN=kubernetes, O=system:masters
Issuer:     CN=cluster-ca
Valid:      2024-01-01 → 2025-01-01
Public Key: RSA 2048-bit ...
SANs:       kubernetes, kubernetes.default, kubernetes.default.svc,
            kubernetes.default.svc.cluster.local, 10.96.0.1, 192.168.1.10
Signature:  (signed with the CA's private key)
```

The **Subject Alternative Names** field is critical — for a server certificate to be valid, the hostname the client used to connect must match either the CN or one of the SANs. In Kubernetes, the apiserver certificate must list every name and IP address through which the apiserver is reachable.

---

### Certificate Authorities (CAs)

Anyone can generate a certificate, but a self-signed certificate provides no trust — the browser (or a Kubernetes component) has no way to verify it is legitimate. Certificate Authorities solve this by acting as trusted third parties.

**How it works:**

```
1. Owner generates a key pair
2. Owner creates a Certificate Signing Request (CSR)
   containing the public key + subject information
3. Owner submits CSR to a CA
4. CA verifies the owner's identity / domain ownership
5. CA signs the certificate with its own private key
6. Signed certificate is returned to the owner
7. Any client with the CA's public key can verify the certificate's signature
```

```bash
# Step 1: generate private key
openssl genrsa -out server.key 2048

# Step 2: generate CSR
openssl req -new -key server.key -subj "/CN=my-server" -out server.csr

# Step 3: CA signs the CSR → issues a certificate
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out server.crt -days 365

# Inspect a certificate
openssl x509 -in server.crt -text -noout
```

**CA trust chain:** CAs have their own key pair. The CA's public key is pre-installed in browsers (public CAs) or distributed across an organisation's systems (private CAs). When a client receives a certificate, it uses the CA's public key to verify the signature — if the signature is valid, the certificate is trusted.

**Public CAs:** DigiCert, Comodo, Sectigo, Let's Encrypt — used for internet-facing services.
**Private CAs:** deployed internally within an organisation — used for internal services. **Kubernetes runs its own private CA** — every cluster has a root CA whose public certificate is distributed to all components and used to sign every component certificate.

---

### Mutual TLS (mTLS)

Standard TLS: only the **server** presents a certificate. The client verifies the server is legitimate, but the server has no cryptographic proof of who the client is.

Mutual TLS: **both** sides present certificates. The server requests a certificate from the client; the client presents a certificate signed by a CA the server trusts.

```
Standard TLS:  Server ──cert──► Client        (client verifies server)
mTLS:          Server ←─cert─── Client        (server also verifies client)
               Server ──cert──► Client
```

**This is exactly how all Kubernetes component-to-component communication works.** The apiserver, etcd, scheduler, controller-manager, kubelet, and kube-proxy all hold client certificates signed by the cluster CA, and they use them to authenticate to each other. A compromised component without a valid client certificate cannot communicate with the rest of the cluster.

---

### PKI — Public Key Infrastructure

The complete system comprising CAs, certificates, key pairs, revocation mechanisms, and the processes governing all of the above is called **Public Key Infrastructure (PKI)**.

In Kubernetes: the cluster CA is the root of the cluster's PKI. All component certificates trace their trust back to this one CA. The CA certificate (`ca.crt`) and private key (`ca.key`) are the most sensitive assets in a self-managed cluster — anyone holding `ca.key` can issue certificates for any identity in the cluster.

---

### File Naming Conventions

Kubernetes PKI files follow consistent naming patterns. Knowing these on sight saves time during exam troubleshooting:

| Content | Common extensions | Kubernetes examples |
|---|---|---|
| Certificate (public key + identity + signature) | `.crt`, `.pem` | `ca.crt`, `apiserver.crt`, `server.pem` |
| Private key | `.key`, `-key.pem` | `ca.key`, `apiserver.key`, `server-key.pem` |
| Certificate Signing Request | `.csr` | `apiserver.csr` |

**Rule of thumb: if the filename contains the word `key`, it is a private key.** If it does not, it is a certificate or public key. This holds consistently across the Kubernetes PKI file tree at `/etc/kubernetes/pki/`.

---

### When to Use Which

#### Certificate Type by Role

| Certificate type | Who holds it | Purpose |
|---|---|---|
| **CA certificate** | Distributed to all clients/components | Used to verify signatures on other certificates |
| **Server certificate** | The server being connected to | Proves the server's identity to connecting clients |
| **Client certificate** | The entity initiating a connection | Proves the client's identity to the server (mTLS) |

In Kubernetes, most components hold **both** a server certificate (for incoming connections) and a client certificate (for outgoing connections to other components).

#### Symmetric vs Asymmetric

| Use | Mechanism | Reason |
|---|---|---|
| Bulk data transfer (TLS session data) | Symmetric (AES, ChaCha20) | Fast; key already securely exchanged |
| Key exchange and authentication (TLS handshake) | Asymmetric (RSA, ECDSA) | Solves key distribution; enables identity verification |
| SSH server access | Asymmetric | No shared secret to exchange; public key pre-placed on server |

These are not competing choices — TLS uses both. Asymmetric encryption bootstraps the trust that lets symmetric encryption operate safely.

---

### Real-World Usage

| Context | How TLS/PKI applies |
|---|---|
| Kubernetes cluster internals | Private CA per cluster; every component holds a client cert signed by it; mTLS on all component channels |
| Public-facing Ingress | cert-manager + Let's Encrypt (public CA) for auto-issued, auto-renewed certificates on Ingress resources |
| Admission webhooks | Webhook server must present a TLS certificate; `caBundle` in the webhook config tells the apiserver which CA to trust for that webhook |
| etcd peer communication | Separate etcd peer CA and certificates for etcd-to-etcd traffic — distinct from the apiserver-to-etcd client certs |
| kubeconfig | Embeds the cluster CA cert (`certificate-authority-data`) so kubectl can verify the apiserver's certificate |

---

### Exam Gotchas

- **Private keys have "key" in the filename** — `server.key`, `server-key.pem`. Certificates do not: `server.crt`, `server.pem`. This naming convention is consistent in the Kubernetes PKI directory.
- **Asymmetric encryption is bidirectional** — either key can encrypt; only the other can decrypt. The "public key encrypts, private key decrypts" framing is a useful simplification but incomplete. In TLS, the client encrypts the session key with the server's public key. In signatures, the server encrypts a hash with its private key and the client decrypts with the public key.
- **mTLS is the Kubernetes default** — it is not an optional enhancement. Every internal Kubernetes channel uses mutual certificate authentication. Components without a valid client cert are rejected.
- **Self-signed ≠ untrusted in a closed system** — the cluster CA is self-signed, but it is explicitly trusted by every component via `--client-ca-file` and `--certificate-authority` flags. "Self-signed" only means untrusted to parties who don't have the CA cert — within a cluster, all components have it.
- **SANs matter for apiserver certificates** — the apiserver certificate must include every IP address and hostname through which the apiserver is reachable (cluster DNS names, external IPs, `127.0.0.1`). A missing SAN is a common source of TLS errors when adding a new access path to an existing cluster.
- **CSR is not the certificate** — generating a CSR does not issue a certificate. The CSR must be signed by a CA. Kubernetes has a native `CertificateSigningRequest` (CSR) API object that allows requesting certificates from the cluster CA programmatically — this will be covered in the upcoming TLS sections.
- **CA key compromise = full cluster compromise** — anyone holding `ca.key` can generate a valid certificate for any identity, including `system:masters` (cluster-admin). Guard `/etc/kubernetes/pki/ca.key` accordingly.

---

### In Managed Clusters (AKS / EKS / GKE)

| Aspect | Managed behaviour |
|---|---|
| **Cluster CA** | Managed by the cloud provider — `ca.key` is never exposed to the operator |
| **Component certificates** | Auto-issued and auto-rotated by the platform; no operator action required |
| **Certificate inspection** | You can retrieve `ca.crt` via `kubectl config view --raw` (embedded in kubeconfig); private keys are never accessible |
| **Application certificates** | cert-manager with Let's Encrypt or private CA integration — fully operator-managed; this is where PKI knowledge is applied on managed clusters |
| **Admission webhook TLS** | Operator-managed — webhooks you deploy must have their own TLS certificate; cert-manager + a self-signed or CA-backed issuer is the standard approach |

