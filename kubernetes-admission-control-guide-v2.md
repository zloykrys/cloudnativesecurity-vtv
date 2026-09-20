# Kubernetes Admission Control Implementation & Governance Guide

Admission control forms the primary operational and security boundary within the Kubernetes control plane, intercepting authenticated API requests before state persistence in `etcd`. The process operates through two sequential phases: mutating webhooks that dynamically modify incoming specifications, and validating webhooks that enforce programmatic compliance. Grounded in authoritative standards such as NIST SP 800-190, the CIS Kubernetes Benchmark, and the NSA/CISA Hardening Guide, admission policies transform enterprise security mandates into mathematical guardrails.

---

## 1. Security and Compliance Hardening

### Key Recommendations
* **Enforce Pod Security Standards (PSS):** Minimise container breakout risks by blocking root execution (`runAsRoot: false`), dropping unnecessary Linux capabilities (e.g., `CAP_SYS_ADMIN`), and disallowing privilege escalation (`allowPrivilegeEscalation: false`).
* **Isolate Host Namespaces & Storage:** Prevent host-level exploitation by rejecting pods requesting `hostPID`, `hostIPC`, or `hostNetwork`, and blocking `HostPath` volume mounts.
* **Mandate Runtime Immutability:** Require read-only root filesystems, enforce non-unconfined Seccomp/AppArmor profiles, enforce default `/proc` masks, and restrict unsafe sysctl modifications.
* **Eliminate Insecure Secrets:** Reject legacy authentication secret types such as `kubernetes.io/basic-auth` in favour of short-lived credentials.

### Implementation Details

| Use Case / Guardrail | Target API Object | Operational Mechanism & Implication | Kubewarden / NeuVector Reference |
| :--- | :--- | :--- | :--- |
| **Enforce Pod Security Standards** | Pod | Rejects `runAsRoot` and requires capability dropping; mitigates container breakout. | **Kubewarden:** `pod-privileged`, `user-group-psp`<br>**NeuVector:** `runAsRoot` or `runAsPrivileged` criteria |
| **Disallow Privilege Escalation** | Pod | Forces `allowPrivilegeEscalation: false`; prevents child process privilege acquisition. | **Kubewarden:** `psp-capabilities`<br>**NeuVector:** `Allow Privilege Escalation` criteria |
| **Block Host Namespaces** | Pod | Rejects `hostPID`, `hostIPC`, and `hostNetwork`; protects node-level process and network isolation. | **Kubewarden:** `pod-privileged`<br>**NeuVector:** `NvAdmissionControlSecurityRule` namespace criteria |
| **Restrict HostPath Volumes** | Pod | Rejects `hostPath` mounts; prevents arbitrary read/write access to the host filesystem. | **Kubewarden:** `volume-types` policy<br>**NeuVector:** `NvAdmissionControlSecurityRule` volume criteria |
| **Read-Only Root Filesystem** | Pod | Forces `readOnlyRootFilesystem: true`; ensures container immutability at runtime. | **Kubewarden:** `read-only-root-filesystem`<br>**NeuVector:** Container immutability security rule |
| **Enforce Seccomp Profiles** | Pod | Rejects `Unconfined` profiles; restricts system calls available to containers. | **Kubewarden:** `seccomp-profile`<br>**NeuVector:** System call filter criteria |

---

## 2. Supply Chain and Cryptographic Provenance

### Key Recommendations
* **Enforce Registry Allowlisting:** Restrict container image sources exclusively to internal, scanned enterprise artifact repositories.
* **Verify Cryptographic Signatures & SBOMs:** Mandate Cosign/Sigstore cryptographic attestations and Software Bill of Materials (SBOM) verification before workload scheduling.
* **Prevent Cache Poisoning:** Employ mutating webhooks to force `imagePullPolicy: Always`, ensuring node-level digest re-verification on every instantiation.
* **Disallow Floating Tags:** Reject the `latest` image tag and mandate explicit semantic versions or SHA-256 digests.

### Implementation Details

| Use Case / Guardrail | Target API Object | Operational Mechanism & Implication | Kubewarden / NeuVector Reference |
| :--- | :--- | :--- | :--- |
| **Restrict Image Registries** | Pod | Rejects untrusted registries; forces reliance on internal, scanned repositories. | **Kubewarden:** Custom `cel-policy`<br>**NeuVector:** `Image registry` criteria |
| **Verify Image Signatures & Vulnerabilities** | Pod | Validates Cosign/Sigstore signatures and enforces maximum allowable CVE thresholds. | **Kubewarden:** `verify-image-signatures`<br>**NeuVector:** `Count of High Severity CVE`, `Image scanned`, `CVE names` criteria |
| **Enforce ImagePullPolicy** | Pod | Mutates policy to `Always`; mitigates node-level image cache poisoning attacks. | **Kubewarden:** Mutating Wasm module<br>**NeuVector:** Evaluated via image compliance policy rules |
| **Block 'Latest' Image Tags** | Pod | Rejects the `latest` tag; enforces declarative, immutable versioning across the cluster. | **Kubewarden:** Tag validation Wasm policy<br>**NeuVector:** Deployment manifest syntax criteria |
| **Require SBOM Attestations** | Pod | Validates presence of an attached SBOM; ensures deep visibility into dependencies. | **Kubewarden:** Wasm attestation verification policy<br>**NeuVector:** Integrated image scanning criteria |

---

## 3. Financial Governance and Cost Control (FinOps)

### Key Recommendations
* **Mandate Resource Requests & Limits:** Block workloads lacking explicit CPU and memory declarations to ensure stable bin-packing and prevent Out-Of-Memory (OOM) failures.
* **Enforce Cost-Allocation Metadata:** Reject namespaces, deployments, and PVCs lacking required tracking labels (e.g., `cost-center`, `owner`, `app.kubernetes.io/managed-by`).
* **Gate Premium Infrastructure:** Inspect node selectors and tolerations to prevent unauthorised workloads from scheduling onto high-cost GPU node pools.
* **Govern Networking and Storage Expenditure:** Prohibit public `LoadBalancer` Service types in non-production environments and set upper bounds on PVC storage requests.

### Implementation Details

| Use Case / Guardrail | Target API Object | Operational Mechanism & Implication | Kubewarden / NeuVector Reference |
| :--- | :--- | :--- | :--- |
| **Enforce CPU/Memory Requests & Limits** | Pod | Injects default boundaries or validates declarations within min/max thresholds. | **Kubewarden:** `container-resources` Wasm policy<br>**NeuVector:** Handled via native K8s limits or Gatekeeper |
| **Mandatory Cost-Allocation Labels** | All Resources | Rejects objects missing required FinOps labels; guarantees accurate cloud spend chargeback. | **Kubewarden:** `safe-labels`<br>**NeuVector:** *N/A* |
| **Restrict Premium Infrastructure** | Pod | Evaluates node selectors; blocks unauthorised access to expensive GPU or compute nodes. | **Kubewarden:** `node-selector` or custom CEL policy<br>**NeuVector:** *N/A* |
| **Block Public Load Balancers** | Service | Rejects `LoadBalancer` types in non-prod; forces utilization of shared Ingress controllers. | **Kubewarden:** `cel-policy` inspecting `spec.type`<br>**NeuVector:** *N/A* |
| **Limit PVC Storage Sizes** | PersistentVolumeClaim | Enforces upper bounds on requested storage capacity; prevents uncontrolled storage costs. | **Kubewarden:** Storage governance Wasm module<br>**NeuVector:** *N/A* |

---

## 4. Operational Automation (Mutating Webhooks)

### Key Recommendations
* **Automate Service Mesh Injection:** Dynamically inject Envoy sidecar proxies upon pod creation to guarantee universal mutual TLS (mTLS) encryption.
* **Automate Credential Ingestion & Telemetry:** Inject Vault agent sidecars for ephemeral secret retrieval and OpenTelemetry/Fluent Bit agents for standardised logging.
* **Optimise Registry Egress & High Availability:** Rewrite external image paths (e.g., `docker.io`) to point to internal pull-through caches and automatically inject topology spread constraints.
* **Default Security Contexts:** Automatically inject non-root user IDs (e.g., `runAsUser: 1000`) if developers omit security context definitions.

### Implementation Details

| Use Case / Guardrail | Target API Object | Mutation Mechanism & Operational Impact | Kubewarden / NeuVector Reference |
| :--- | :--- | :--- | :--- |
| **Service Mesh Proxy Injection** | Pod | Injects Envoy sidecars; guarantees universal mTLS and traffic observability. | **Kubewarden:** Mutating Wasm ClusterAdmissionPolicy<br>**NeuVector:** Custom mutating webhook integration |
| **Vault Secret Ingestion** | Pod | Injects credential retrieval sidecars; eliminates hardcoded secrets. | **Kubewarden:** Mutating Wasm ClusterAdmissionPolicy<br>**NeuVector:** *N/A* |
| **Observability Agent Injection** | Pod | Attaches OpenTelemetry/Fluent Bit agents; standardises enterprise telemetry collection. | **Kubewarden:** Mutating Wasm ClusterAdmissionPolicy<br>**NeuVector:** *N/A* |
| **Image Registry Rewriting** | Pod | Mutates external registry paths to internal proxies; secures supply chain and saves egress costs. | **Kubewarden:** Mutating Wasm ClusterAdmissionPolicy<br>**NeuVector:** Evaluated via registry criteria |
| **Default Security Contexts** | Pod | Injects non-root user IDs; acts as a safety net for incomplete developer manifests. | **Kubewarden:** Mutating Wasm ClusterAdmissionPolicy<br>**NeuVector:** Evaluated via security rules |

---

## 5. Reliability, Resiliency, and Traffic Management

### Key Recommendations
* **Require Health Probes:** Block deployments lacking explicit liveness and readiness probes to guarantee self-healing and zero-downtime routing.
* **Enforce High Availability (HA) Topology:** Require `podAntiAffinity` or `topologySpreadConstraints` and enforce minimum replica counts in production namespaces.
* **Prevent Ingress Host Collisions:** Validate Ingress host strings cluster-wide to prevent catastrophic domain collisions and traffic misdirection.
* **Enforce TLS Configurations:** Mandate valid `tls` blocks on all Ingress resources to satisfy data-in-transit security requirements.

### Implementation Details

| Use Case / Guardrail | Target API Object | Operational Mechanism & Implication | Kubewarden / NeuVector Reference |
| :--- | :--- | :--- | :--- |
| **Mandatory Health Probes** | Deployment | Rejects workloads lacking liveness/readiness probes; guarantees self-healing capabilities. | **Kubewarden:** `cel-policy` checking probe paths<br>**NeuVector:** *N/A* |
| **Enforce HA Topology** | Deployment | Requires anti-affinity or spread constraints; prevents single-node or single-AZ outages. | **Kubewarden:** Topology validation Wasm module<br>**NeuVector:** *N/A* |
| **Block Overlapping Ingress Hosts** | Ingress | Validates domain strings globally; prevents catastrophic traffic routing conflicts. | **Kubewarden:** `cel-policy` (`unique_ingress` context-aware policy)<br>**NeuVector:** *N/A* |
| **Enforce Ingress TLS** | Ingress | Evaluates `has(object.spec.tls) && size(object.spec.tls) > 0` to block HTTP routes. | **Kubewarden:** `cel-policy` (`require-ingress-tls`)<br>**NeuVector:** *N/A* |
| **Mandate Pod Disruption Budgets** | Deployment | Requires PDB configurations; protects workloads during voluntary node maintenance. | **Kubewarden:** Cluster-wide PDB validation policy<br>**NeuVector:** *N/A* |

---

## 6. Identity, Access, and RBAC Guardrails

### Key Recommendations
* **Block High-Risk RBAC Bindings:** Prohibit the creation of `ClusterRoleBindings` that assign `cluster-admin` privileges to non-system entities.
* **Protect Control Plane Namespaces:** Block unauthorised modifications to protected core namespaces such as `kube-system`.
* **Restrict Default Service Accounts:** Reject pods utilising default service accounts to enforce dedicated, least-privilege identities.
* **Eliminate Wildcard Verbs & Environment Secrets:** Disallow wildcard (`*`) RBAC verbs and block pods injecting sensitive credentials into process environment variables.

### Implementation Details

| Use Case / Guardrail | Target API Object | Operational Mechanism & Implication | Kubewarden / NeuVector Reference |
| :--- | :--- | :--- | :--- |
| **Block cluster-admin Bindings** | ClusterRoleBinding | Rejects assignment of ultimate privileges; prevents total cluster compromise. | **Kubewarden:** RBAC validation Wasm module<br>**NeuVector:** `NvAdmissionControlSecurityRule` RBAC rule |
| **Restrict Environment Variable Secrets** | Pod | Prevents credential extraction from process environment variable definitions. | **Kubewarden:** Custom environment inspection Wasm module<br>**NeuVector:** `Environment variables with secrets` criteria |
| **Restrict Default Service Accounts** | Pod | Rejects default service account usage; forces adoption of least-privilege identities. | **Kubewarden:** `service-account` policy<br>**NeuVector:** `NvAdmissionControlSecurityRule` service account criteria |
| **Protect System Namespaces** | All Resources | Blocks modifications to `kube-system`; protects control plane from tampering. | **Kubewarden:** Namespace protection policy<br>**NeuVector:** Namespace scope restriction rule |

---

## Sample Policy Manifests

### Example 1: Kubewarden Mutating Wasm Policy (Container Resources)
This `ClusterAdmissionPolicy` uses Kubewarden's WebAssembly policy engine to inject default CPU and memory limits/requests if omitted, while validating that requested values lie within specified bounds.

```yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: enforce-container-resources
  annotations:
    io.kubewarden.policy.category: FinOps
    io.kubewarden.policy.severity: high
spec:
  policyServer: default
  module: registry://ghcr.io/kubewarden/policies/container-resources:v1.1.0
  mutating: true
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations:
        - CREATE
        - UPDATE
  settings:
    memory:
      defaultRequest: "256Mi"
      defaultLimit: "512Mi"
      minRequest: "128Mi"
      maxLimit: "2Gi"
    cpu:
      defaultRequest: "200m"
      defaultLimit: "500m"
      minRequest: "100m"
      maxLimit: "2000m"
    ignoreImages:
      - "ghcr.io/trusted-system-agent/*"
```

---

### Example 2: Kubewarden Validating CEL Policy (Ingress TLS)
This policy uses Common Expression Language (CEL) inside Kubewarden to enforce that all Ingress resources contain a non-empty `tls` block, blocking insecure HTTP endpoints.

```yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: require-ingress-tls
  annotations:
    io.kubewarden.policy.category: Security
    io.kubewarden.policy.severity: critical
spec:
  policyServer: default
  module: registry://ghcr.io/kubewarden/policies/cel-policy:latest
  mutating: false
  rules:
    - apiGroups: ["networking.k8s.io"]
      apiVersions: ["v1"]
      resources: ["ingresses"]
      operations:
        - CREATE
        - UPDATE
  settings:
    variables:
      - name: has_tls
        expression: "has(object.spec.tls) && size(object.spec.tls) > 0"
    validations:
      - expression: "variables.has_tls"
        message: "Security violation: All Ingress resources must define a TLS block with a valid secretName."
```

---

### Example 3: NeuVector Admission Control Security Rule (Root & CVE Guardrails)
This NeuVector Custom Resource Definition (`NvAdmissionControlSecurityRule`) blocks deployments attempting to run as root or containing high severity CVEs.

```yaml
apiVersion: neuvector.com/v1
kind: NvAdmissionControlSecurityRule
metadata:
  name: block-root-and-high-cves
spec:
  action: deny
  category: deployment
  comment: "Reject root execution and unpatched high severity vulnerabilities"
  criteria:
    - name: runAsRoot
      op: "="
      value: "true"
    - name: Count of High Severity CVE
      op: ">="
      value: "1"
    - name: Image scanned
      op: "="
      value: "true"
  disable: false
  rule_type: user
```
