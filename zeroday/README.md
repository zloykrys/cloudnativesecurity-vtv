# Use Case 4: Zero-Day Attack Prevention (Runtime Security)

## Overview
Pre-deployment supply chain scanning is critical, but it cannot stop zero-day exploits—vulnerabilities that have no known CVE signature yet, such as the initial Log4Shell outbreak. 

To mitigate this, NeuVector utilises a patented Zero-Trust Container Runtime. By observing a container's baseline behaviour during regular operation, NeuVector builds a strict "Process Profile". If an attacker exploits a zero-day vulnerability to spawn an anomalous process (like a reverse shell), NeuVector intercepts and terminates the process at the kernel level, regardless of whether a CVE signature exists.

**Business Value:** Provides deterministic, signature-less protection against unknown zero-day attacks and remote code execution (RCE), ensuring applications remain secure even if an underlying library is compromised.

**Threat Vector:** A zero-day Remote Code Execution (RCE) vulnerability (CVE-2021-44228 / Log4Shell) allowing an attacker to inject an LDAP payload, execute arbitrary code, and open a reverse shell into the container.

**Success Criteria:** In `Protect` mode, NeuVector's process enforcement immediately blocks the anomalous `/bin/sh` process spawned by the Java application, preventing the attacker from establishing a reverse shell connection.

---

## Prerequisites for this Test
* A terminal with `netcat` (`nc`), `python3`, `pip`, and `docker` installed.
* A web browser to interact with the vulnerable application's GUI.
* Ensure you map `log4shell.demo.com` to your Kubernetes Ingress IP in your local `/etc/hosts` file.
* Reference PoC Exploit Repository: [kozmer/log4j-shell-poc](https://github.com/kozmer/log4j-shell-poc)

---

## Verification Steps (Interactive Tutorial)

### Phase 1: The Exploit (Simulating an Unprotected Cluster)
First, we will demonstrate the devastating impact of the exploit without runtime protection.

1. **Deploy the vulnerable application** to your cluster from this directory:
```bash
   kubectl apply -f log4shell-vulnerable-app.yaml
   ```
2. Open the NeuVector UI, navigate to **Policy -> Groups**, and search for `nv.log4shell.log4shell-rogue-ns`. Ensure this group is in **Discover** or **Monitor** mode (which allows anomalous processes to run so we can baseline them).
3. **Clone the PoC repository and configure the Python environment:**
```bash
   git clone [https://github.com/kozmer/log4j-shell-poc.git](https://github.com/kozmer/log4j-shell-poc.git)
   cd log4j-shell-poc

   # Create and activate a dedicated Python virtual environment
   python3 -m venv venv
   source venv/bin/activate
   
   # Install the required dependencies (like colorama)
   pip install -r requirements.txt
   ```
4. **Configure the Legacy Java JDK:**
   The exploit script requires an older version of Java to compile the malicious payload.
   * Download the Java JDK archive: `jdk-8u20-linux-x64.tar.gz` (Available via Oracle archives or mirrors).
   * Extract it directly into the root of the `log4j-shell-poc` directory:
```bash
     tar -xf jdk-8u20-linux-x64.tar.gz