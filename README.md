# Container Security & Runtime Hardening Lab
## Attack & Defense Lifecycle at the Microservice Layer

## 1. Executive Summary
This project demonstrates a comprehensive approach to securing containerized applications on Linux infrastructure. It covers the entire lifecycle of microservice security: from proactive static image scanning during development to real-time kernel-level threat detection at runtime, concluding with robust host-level configuration hardening to achieve isolation.

---

## 2. Lab Infrastructure Breakdown
* **Host Platform:** Ubuntu Server running Docker Engine
* **Security Monitoring Matrix:** Falco (Cloud-Native Runtime Telemetry) + Wazuh Agent (SIEM Aggregator)
* **Target Environment:** Intentionally vulnerable microservice container deployment

---

## 3. Phase 1: Static Image Auditing (Proactive Defense)
Before a container is built or deployed into production, its blueprint must be analyzed for known vulnerabilities and embedded secrets. 

### Step 1: Install the Trivy Scanner Engine
```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - [https://aquasecurity.github.io/trivy/public.key](https://aquasecurity.github.io/trivy/public.key) | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] [https://aquasecurity.github.io/trivy/repo/deb](https://aquasecurity.github.io/trivy/repo/deb) $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update && sudo apt-get install trivy -y

trivy image --severity HIGH,CRITICAL ubuntu:20.04

curl -fsSL [https://falco.org/repo/falcosecurity-packages.asc](https://falco.org/repo/falcosecurity-packages.asc) | sudo gpg --dearmor -o /usr/share/keyrings/falco-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/falco-archive-keyring.gpg] [https://download.falco.org/packages/deb](https://download.falco.org/packages/deb) stable main" | sudo tee /etc/apt/sources.list.d/falcosecurity.list
sudo apt-get update && sudo apt-get install -y falco

<localfile>
  <log_format>json</log_format>
  <location>/var/log/falco_alerts.json</location>
</localfile>

version: '3.8'
services:
  secure-app:
    image: vulnerable-app:latest
    container_name: production_microservice
    # Enforce non-root execution path
    user: "10001:10001"
    # Prevent container privilege escalation to host kernel
    privileged: false
    security_opt:
      - no-new-privileges:true
    # Drop dangerous Linux capabilities to minimize risk profile
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    # Protect host file integrity by locking down the container OS
    read_only: true
    tmpfs:
      - /tmp
      - /run
    # Implement strict physical resource boundaries
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M


Attack Simulation: Attempting to spawn an interactive bash shell (docker exec -it production_microservice bash) or running an unauthorized process like whoami inside the container.

Telemetry Output: Falco triggers an immediate kernel alert (Notice A shell was spawned in a container with an attached terminal), which is cleanly ingested by the Wazuh Agent and visualized on the SIEM interface.
