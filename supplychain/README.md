# Test Case 1: Zero-Trust Image Signature Verification (SUSE AppCo)

## Description
Software supply chain attacks are increasingly targeting container registries to inject malicious code into trusted infrastructure. This test case demonstrates how NeuVector programmatically detects, validates, and enforces a Zero-Trust architecture by verifying the Sigstore/Cosign cryptographic signatures of SUSE Application Collection (AppCo) images inside a GitOps pipeline. 

**Business value:** Guarantee the integrity and authenticity of software deployments before they are scheduled in the cluster, neutralizing supply chain poisoning attacks.

## Success Criteria
* **Automated Protection:** The NeuVector Admission Control engine intercepts the Kubernetes deployment API request and strictly denies any unverified or tampered container images from running in the target namespace.
* **Forensics & Diagnostics:** NeuVector generates a high-severity Admission Control block event and logs the cryptographic verification failure for compliance auditing.

## Threat Vector Details
* **Target Application:** SUSE Application Collection Caddy 
* **Exploit Mechanism:** Supply Chain Poisoning / Unauthorized Image Deployment
* **Exploit Payload:** An unsigned or maliciously modified container image bypassing CI/CD checks.

## Pre-Requisite Cluster Instructions
Because this repository is public, credentials and tokens must be decoupled from Git. The cluster operator must execute the following commands to provision the required target namespace and cryptographic/pull secrets **before** the GitOps engine synchronizes the repository:

```bash
# 1. Create the targeted isolated deployment namespace
kubectl create namespace caddy-secure

# 2. Create the NeuVector API Authentication Secret (Used by the bootstrap job)
kubectl create secret generic neuvector-api-credentials \
  --namespace caddy-secure \
  --from-literal=username=admin \
  --from-literal=password='YOUR_NEUVECTOR_PASSWORD'

# 3. Create the SUSE AppCo Registry Pull Secret (Used by Kubelet to pull the image)
kubectl create secret docker-registry suse-appco-registry-secret \
  --namespace caddy-secure \
  --docker-server=dp.apps.rancher.io \
  --docker-username='YOUR_APPCO_USERNAME' \
  --docker-password='YOUR_APPCO_TOKEN'
```

## GitOps Implementation Architecture
To automate this scenario while keeping secrets out of public GitHub repositories, the state is declared using three primary layers:
1. **Bootstrapping Layer:** A Kubernetes Job securely injects the SUSE AppCo public key into NeuVector's internal Sigstore engine using parameterized references to the pre-created cluster secret. It then automatically enables the NeuVector Admission Control Webhook.
2. **Security Layer:** Deploys an `NvAdmissionControlSecurityRule` that enforces a default-deny policy for the namespace, allowing only images explicitly verified by the `suse-appco/suse-appco-key` Root of Trust.
3. **Application Layer:** Deploys the Caddy workload, referencing the pre-created `imagePullSecret` to safely authenticate against the AppCo registry.

## Verification Steps
1. Execute the commands outlined in the **Pre-Requisite Cluster Instructions** section above.
2. Push the deployment, bootstrap job, and security configurations to your Git repo monitored by your GitOps engine (e.g., Fleet).
3. Verify that the signed SUSE AppCo Caddy deployment spins up successfully.
4. Modify the `caddy-deployment.yaml` to use an unsigned image (e.g., `docker.io/library/caddy:latest`) and push the update.
5. Verify that the deployment is instantly rejected by Kubernetes.
6. Open the NeuVector console or query the logs to view the blocked threat under **Notifications -> Security Events**.