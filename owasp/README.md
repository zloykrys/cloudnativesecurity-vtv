# Test Case 2: OWASP Top 10 Attack Mitigation (SQL Injection)

## Description
A03:2021-Injection occupies third place in the OWASP Top 10 vulnerabilities list. This test case demonstrates how NeuVector programmatically detects, alerts, and neutralizes SQL Injection (SQLi) attacks inside a GitOps pipeline, keeping the core application secured and operational.

Business value: keep application running when a known exploit exists but no patches available

## Success Criteria
* **Automated Protection:** The SQL injection authentication bypass exploit is successfully blocked by the upstream NeuVector Deep Packet Inspection (DPI) web application firewall engine.
* **Forensics & Diagnostics:** NeuVector generates a high-severity Security Event alert and captures an automated packet capture (PCAP) of the threat payload for compliance auditing.

## Threat Vector Details
* **Target Application:** OWASP Juice Shop (intentionally vulnerable application)
* **Exploit Mechanism:** SQL Injection login bypass
* **Exploit Payload String:**
  * **Email:** `' or 1=1; --`
  * **Password:** `any_random_string`

## GitOps Implementation Architecture
To automate this scenario without manual UI intervention, the state is declared using two primary layers:
1. **Application Layer:** Deploys the application inside a dedicated isolated namespace (`juice-shop`).
2. **Security Layer:** Deploys an `NvWafSecurityRule` defining the custom SQLi regular expression sensor and an `NvSecurityRule` setting the workload group to `Protect` mode with the sensor attached in a `deny` action state.

## Verification Steps
1. Push the deployment and security configurations to your Git repo monitored by your GitOps engine (e.g., Fleet, ArgoCD or Flux CD).
2. Attempt to log into the Juice Shop console using the SQL injection vector: `' or 1=1; --`.
3. Verify that the login execution fails immediately.
4. Open the NeuVector console or query the logs to view the blocked threat under **Notifications -> Security Events** along with its diagnostic packet capture file.
