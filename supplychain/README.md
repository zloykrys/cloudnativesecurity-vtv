# Comprehensive Zero-Trust Supply Chain Security Pipeline

## Overview
This repository demonstrates a fully automated, defence-in-depth GitOps pipeline using NeuVector. By deploying this single architecture, we validate three critical pillars of container supply chain security: Vulnerability Management, Chain of Custody Verification, and Zero-Trust Admission Control.

---

## Use Case 1: Artefact Scanning in Container Registry (Vulnerability Management)

**Description:** Before a container is ever allowed to run, NeuVector must inspect its contents. This phase automatically connects to upstream container registries (SUSE AppCo and Docker Hub), downloads the image layers, and cross-references the packages against current CVE databases to identify known software vulnerabilities.

**Business Value:** Proactively shifts security left by identifying and cataloguing exploitable vulnerabilities at the registry level, reducing the cluster's attack surface and fulfilling compliance auditing requirements.

**Threat Vector:** Deploying container images that contain known, exploitable software vulnerabilities (e.g., outdated OpenSSL libraries) that attackers can use to gain a foothold.

**Success Criteria:** NeuVector successfully authenticates to the target registries, downloads the image manifests, parses the OS and application layers, and generates a comprehensive CVE vulnerability report.

**Verification Steps:**
1. Open the NeuVector console and navigate to **Assets -> Registries**.
2. Select either the `SUSE-AppCo` or `DockerHub-Test` registry.
3. Click into the **Images** tab and select the targeted Caddy image.
4. Verify that a detailed vulnerability scan report is populated, displaying the Risk Score, CVE IDs, and impacted modules.

---

## Use Case 2: Signature Verification (Chain of Custody)

**Description:** Scanning for vulnerabilities is not enough; we must also guarantee provenance. This phase utilises the Sigstore/Cosign framework to cryptographically verify that the container image was explicitly signed by a trusted entity (SUSE AppCo) and has not been tampered with since compilation.

**Business Value:** Guarantees software integrity and provenance, ensuring that only trusted, unaltered code produced by approved build pipelines enters the infrastructure environment.

**Threat Vector:** Supply chain poisoning, Man-in-the-Middle (MITM) attacks altering the image in the registry, or unauthorised image substitution by compromised third-party vendors.

**Success Criteria:** NeuVector successfully locates the `.sig` manifest in the registry, validates it against the securely injected SUSE AppCo public key (`suseappcokey`), and caches the verified status.

**Verification Steps:**
1. Open the NeuVector console and navigate to **Assets -> Registries**.
2. Select the `SUSE-AppCo` registry and view the scanned Caddy image.
3. Look for the explicit "Sigstore" badge or signature validation status next to the image ID, proving the cryptographic signature was verified against the Root of Trust.

---

## Use Case 3: Admission Control Enforcement (Zero-Trust Gatekeeper)

**Description:** This is the final enforcement mechanism. The Kubernetes Validating Webhook intercepts K8s API requests in real-time. It evaluates incoming Pods against a strict Default-Deny policy, allowing execution *only* if the image has passed the signature verification phase.

**Business Value:** Acts as an automated, physical fail-safe that strictly enforces security policies, eliminating the risk of human error and preventing untrusted workloads from consuming compute resources.

**Threat Vector:** Rogue deployments, insider threats, or compromised CI/CD pipelines attempting to bypass security checks and run unverified code in a secure namespace.

**Success Criteria:** The Webhook successfully allows the deployment of the cryptographically signed SUSE AppCo image, while instantaneously and explicitly denying the deployment of the unsigned Docker Hub image.

**Verification Steps:**
1. Check the Kubernetes cluster deployment state: verify that the `suse-appco-caddy` pod successfully reaches a `Running` state.
2. Check the cluster state for `malicious-unsigned-caddy`: verify that the deployment was blocked and the Pod failed to schedule.
3. Open the NeuVector console and navigate to **Notifications -> Security Events**.
4. Locate the high-severity **Admission Control Deny** log, verifying that the unsigned image was blocked specifically by the enforcement policy.

---

## Pre-Requisite Cluster Instructions
Because this repository is public, credentials and tokens must be decoupled from Git. The cluster operator must execute the following commands to provision the required target namespace and cryptographic/pull secrets **before** the GitOps engine synchronises the repository:

```bash
# 1. Create the targeted isolated deployment namespace
kubectl create namespace caddy-secure

# 2. Create the NeuVector API Authentication Secret (Used by the bootstrap job)
kubectl create secret generic neuvector-api-credentials \
  --namespace caddy-secure \
  --from-literal=username=admin \
  --from-literal=password='YOUR_NEUVECTOR_PASSWORD'

# 3. Create the SUSE AppCo Registry Pull Secret (Used by Kubelet to pull the signed image)
kubectl create secret docker-registry suse-appco-registry-secret \
  --namespace caddy-secure \
  --docker-server=dp.apps.rancher.io \
  --docker-username='YOUR_APPCO_USERNAME' \
  --docker-password='YOUR_APPCO_TOKEN'
```

## GitOps Implementation Architecture
To automate this scenario while avoiding Kubernetes/GitOps race conditions, the state is declared as a **Strict Helm Chart** utilising execution sequence hooks:

1. **Security Layer (Hook Weight -5):** Deploys an `NvAdmissionControlSecurityRule` *before* anything else. This enforces a default-deny policy for the namespace, allowing only images explicitly verified by the `suseappco/suseappcokey` Root of Trust.
2. **Bootstrapping Layer (Hook Weight -1 & 0):** A `pre-install` ConfigMap and Kubernetes Job securely inject the SUSE AppCo public key into NeuVector's internal Sigstore engine, authenticate the scanner to the private registries, enable the Admission Webhook, and trigger asynchronous signature scans. The Job intentionally holds the deployment pipeline open for 60 seconds to allow the scans to complete.
3. **Application Layer (Standard Install):** Once the pipeline hold is released, Helm deploys the signed AppCo Caddy workload alongside an explicitly unsigned public Caddy workload to prove the Zero-Trust enforcement mechanism against the fully-armed webhook.