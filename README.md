# cloudnativesecurity-vtv
A set of simple-to-set up Proof of Technology cases that help to validate the business value of cloud native security tooling.

## Structure

### 📦 [Supply Chain Security](./supplychain)
This module focuses on pre-deployment gates, ensuring that only trusted, verified, and scanned code is allowed to enter the Kubernetes environment.

* **[Test Case 1: Artefact Validation](./supplychain):** Automated scanning of container registries to identify and catalogue known software vulnerabilities (CVEs) before deployment.
* **[Test Case 2: Chain-of-Custody (Signature) Verification](./supplychain):** Cryptographic signature validation using the Sigstore/Cosign framework to guarantee image provenance and prevent supply chain tampering.
* **[Test Case 3: Admission Control](./supplychain):** A strict Kubernetes Zero-Trust gatekeeper that inercepts API requests to block unsigned or unverified workloads from being scheduled.

### 🌐 [OWASP Web Application Security](./owasp)
This module demonstrates Layer 7 network protections against common web-based application attacks, providing critical "virtual patching" capabilities.

* **[Test Case 4: OWASP Top 10 Attack Mitigation (SQL Injection)](./owasp):** Utilising NeuVector's Deep Packet Inspection (DPI) Web Application Firewall (WAF) to detect, alert, and neutralise SQL Injection attacks in real-time, keeping the core application secure and operational even when no patches are available.

### ⚔️ [Zero-Day & Runtime Protection](./zeroday)
This module focuses on post-deployment, behavioural security mechanisms designed to stop unknown threats and bypasses.

* **[Test Case 5: Zero-Day Attack Prevention](./zeroday):** Utilising a patented Zero-Trust Container Runtime to build process profiles, deterministically blocking zero-day Remote Code Execution (RCE) attacks (like Log4Shell) without relying on known CVE signatures.