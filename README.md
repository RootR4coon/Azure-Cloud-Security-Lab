# Azure Cloud Security Lab: Load-Balanced Infrastructure & Live Forensic Log Auditing

A hands-on SOC (Security Operations Center) lab built in Microsoft Azure: a secured network with a JumpBox, two load-balanced web servers, Ansible automation, and an ELK monitoring zone. I then attacked my own lab and audited the traces.

<p align="center">
  <img src="https://github.com/user-attachments/assets/8bf88395-c5af-4c48-8a40-ee0b279be124" alt="Architecture diagram of the Azure SOC lab" width="700">
</p>

<p align="center"><sub><i>Intended architecture. All components were deployed, but the Filebeat → Elasticsearch → Kibana path could not be completed (see <a href="#step-15-log-shipping-attempt-filebeat-and-kibana-limitation">Step 15</a>), so attack detection was done directly in the terminal.</i></sub></p>

---

## Table of Contents
1. [Project Overview](#project-overview)
2. [Project Status](#project-status)
3. [Core Technologies](#core-technologies)
4. [Directory Structure](#directory-structure)
5. [Phase 1: The Azure Cloud Foundation](#phase-1-the-azure-cloud-foundation)
6. [Phase 2: Gateway Tier and Automation Infrastructure](#phase-2-gateway-tier-and-automation-infrastructure)
7. [Phase 3: High-Availability Web Architecture](#phase-3-high-availability-web-architecture)
8. [Phase 4: Monitoring Zone Setup (ELK and Peering)](#phase-4-monitoring-zone-setup-elk-and-peering)
9. [Phase 5: Threat Simulation and Live Terminal Log Auditing](#phase-5-threat-simulation-and-live-terminal-log-auditing)
10. [Phase 6: Infrastructure Decommissioning and Clean Up](#phase-6-infrastructure-decommissioning-and-clean-up)
11. [Lessons Learned and Technical Solutions](#lessons-learned-and-technical-solutions)
12. [Cleanup & Security Notes](#cleanup--security-notes)

---

## Project Overview
This repository documents the end-to-end architecture, deployment, and live security auditing of a secure cloud infrastructure built within Microsoft Azure. The project simulates an enterprise network environment containing an administrative gateway tier, a high-availability web tier, and an isolated monitoring zone.

A SIEM (Security Information and Event Management) is a program that collects logs from all your computers so you can spot attacks. In this project, I built a network, a safe entrance, two web servers, a receptionist, and a monitoring room. Then, I simulated real-world attacks against my own lab and audited the resulting telemetry.

> **Note on the monitoring stack:** Elasticsearch and Kibana were deployed, but I could not get the web servers' logs flowing into them. I therefore performed the attack detection directly in the Linux terminal (`/var/log/auth.log` and `docker logs`). See [Step 15](#step-15-log-shipping-attempt-filebeat-and-kibana-limitation) and [Lessons Learned](#lessons-learned-and-technical-solutions) for details.

### Architectural Mapping (The Castle Blueprint)
To explain the environment's defense-in-depth layout, the infrastructure maps directly to historical fortresses:

| Component | Castle analogy |
|-----------|----------------|
| Microsoft Azure | The land where the castle stands |
| Virtual Network | The perimeter castle wall |
| NSG (Security Group) | The guards at the gate |
| JumpBox | The only door in |
| Web-1, Web-2 | Two identical shops |
| Load Balancer | The receptionist |
| ELK | The security camera room |
| Ansible | Your robot helper |

---

## Project Status

| Component | Status |
|-----------|--------|
| Azure network, NSGs, peering | ✅ Completed |
| JumpBox + containerized Ansible | ✅ Completed |
| Two web servers with DVWA behind a Load Balancer | ✅ Completed |
| ELK server (Elasticsearch + Kibana in Docker) | ✅ Deployed |
| Filebeat → Elasticsearch → Kibana log pipeline | ❌ Not completed (see Step 15) |
| Attack simulations (nmap, SSH brute-force, SQL injection) | ✅ Completed, audited in the terminal |
| Clean-up of all resources | ✅ Completed |

---

## Core Technologies
* **Cloud Platform:** Microsoft Azure (VNets, Peerings, NSGs, NAT Gateway, Availability Sets, Standard Load Balancers)
* **Configuration Management:** Ansible (containerized, playbooks)
* **Containerization:** Docker & Docker Compose
* **Monitoring Stack:** Elasticsearch, Kibana, Filebeat (ELK)
* **Security & Scanning:** Nmap, OpenSSH, DVWA (Damn Vulnerable Web Application)
* **Operating System:** Ubuntu Server 22.04 LTS

---

## Directory Structure
```text
soc-azure-lab/
├── README.md      # Screenshots are embedded directly in this file
├── images/        # Optional local copies of screenshots
├── ansible/       # Host files, deployment scripts, and configuration playbooks
│   ├── hosts
│   ├── pentest.yml
│   └── beats.yml
└── elk/           # Docker Compose YAML definition file
    └── docker-compose.yml
```

> 💡 *Click any screenshot to enlarge it.*

---

## Phase 1: The Azure Cloud Foundation

### Step 1: Resource Group Deployment
<p align="center">
  <img src="https://github.com/user-attachments/assets/68bde916-1157-49ae-8b33-93a5152b97ee" alt="01-resource-group" width="650">
</p>

**Documentation:** I created a Resource Group called SOC-Lab. A Resource Group keeps every resource in the project in one place. This makes it easy to stay organized and to delete everything at once when the lab is finished, so I avoid unnecessary costs.

### Step 2: Virtual Network Topology
<p align="center">
  <img src="https://github.com/user-attachments/assets/9b2e3d64-dbb2-431a-a849-d4f56e604cf5" alt="02-vnet" width="650">
</p>

**Documentation:** I created a private virtual network with the address space 10.0.0.0/16 and a subnet of 10.0.0.0/24. The network is like my own fenced-off zone in the cloud, where my servers can talk to each other without being exposed to the whole internet. The subnet is where I later place the JumpBox and the web servers.

### Step 3: Perimeter Security Enforcement (NSG)
<table>
  <tr>
    <td align="center"><img src="https://github.com/user-attachments/assets/14ec25c0-ab37-4387-9b7a-157104b9d3bf" alt="03-nsg-rules" width="400"><br><sub>Inbound security rules</sub></td>
    <td align="center"><img src="https://github.com/user-attachments/assets/020567f8-522a-4d6c-8d6f-9a4d5c977d01" alt="04-subnet-association" width="400"><br><sub>NSG attached to the subnet</sub></td>
  </tr>
</table>

**Documentation:** I created an NSG that only allows SSH (port 22) and HTTP (port 80) from my own IP address, and attached it to the subnet. An NSG works like a firewall that decides which traffic may come in. Limiting the source to my IP reduces the attack surface, because bots on the internet cannot reach my servers.

### Step 4: Cryptographic Key Infrastructure
**Documentation:** I generated an SSH key pair on my computer/terminal. The public key goes on the servers, while the private key stays with me. This is safer than passwords because a key cannot be guessed by a bot.

---

## Phase 2: Gateway Tier and Automation Infrastructure

### Step 5: JumpBox Provisioning
<table>
  <tr>
    <td align="center"><img src="https://github.com/user-attachments/assets/22f39acd-44dc-4588-bed9-1b616caf5b72" alt="05-ubuntu" width="380"><br><sub>JumpBox VM overview</sub></td>
    <td align="center"><img src="https://github.com/user-attachments/assets/3853dc36-68ba-43e4-b3b7-84864dcc7334" alt="07-Terminal-loggedIn" width="420"><br><sub>Logged in over SSH</sub></td>
  </tr>
</table>

<p align="center">
  <img src="https://github.com/user-attachments/assets/dee2caca-bb61-4951-bfa4-c45dd36c6e48" alt="06-IP-static" width="620"><br>
  <sub>Static private IP 10.0.0.4</sub>
</p>

**Documentation:** I created a JumpBox, a server with a public IP that is the only way into the network from outside. All other servers get no public IP and can only be reached through the JumpBox. This means I only have to protect one entrance, and I gave it a fixed private IP (10.0.0.4) so the address never changes.

### Step 6: Containerized Ansible Engine
<p align="center">
  <img src="https://github.com/user-attachments/assets/c7164fb9-5191-4936-8c3e-51ef81cb48ab" alt="08-Docker-Installed" width="650">
</p>

**Documentation:** I installed Docker on the JumpBox and started a container with Ansible inside it. Docker packages a program together with everything it needs, so I don't have to install and mix tools directly on the server. Ansible is the tool that later configures all my other servers automatically.

### Step 7: Automation Credentials
<p align="center">
  <img src="https://github.com/user-attachments/assets/83e0ecf2-b21b-42fb-be7c-c21931c62c48" alt="09-Ansible-Container" width="650"><br>
  <img src="https://github.com/user-attachments/assets/41482f64-14f5-4119-88ae-c684a2e23ef1" alt="10-Inside-Ansible-Container" width="650">
</p>

**Documentation:** I generated a separate key pair inside the Ansible container. The container is the one that logs into the other servers, so its public key is what must be installed on them. This lets Ansible connect without passwords, which automation requires.

---

## Phase 3: High-Availability Web Architecture

### Step 8: Redundant Private Web Servers & NAT Gateway
<table>
  <tr>
    <td align="center"><img src="https://github.com/user-attachments/assets/3a3632e7-9408-4500-89ba-a21982870ab8" alt="11-The-VM-List" width="420"><br><sub>VM list</sub></td>
    <td align="center"><img src="https://github.com/user-attachments/assets/aa75d733-f4c6-44fe-9947-20cb6dfdd273" alt="12-Web1-Overview" width="420"><br><sub>Web-1 overview (no public IP)</sub></td>
  </tr>
</table>

**Documentation:** I created two identical web servers without public IPs, placed in the same Availability Set. An Availability Set spreads them across different physical hardware, so if one fails the other keeps running. The NAT Gateway lets them download software from the internet without anyone from outside being able to reach them.

### Step 9: Perimeter Access Verification
<p align="center">
  <img src="https://github.com/user-attachments/assets/eb8db450-088d-451d-a421-636a4752793d" alt="13-azureadmin-at-Web-1" width="460"><br>
  <img src="https://github.com/user-attachments/assets/cf9285e4-535a-4930-83fd-d4a2339fe52c" alt="14-azureadmin-at-Web-2" width="460">
</p>

**Documentation:** I logged in to both web servers over SSH from the Ansible container on the JumpBox. This proves the JumpBox design works: the servers have no public IP but can still be reached through the one secure entrance. These are the SSH arrows in the network diagram.

### Step 10: Configure Ansible and Install DVWA
<table>
  <tr>
    <td align="center"><img src="https://github.com/user-attachments/assets/0f5f3d2c-fa14-4675-8e64-f91b727a19c0" alt="15-ansible-ping-SUCCESS" width="320"><br><sub>Ansible ping: SUCCESS on both servers</sub></td>
    <td align="center"><img src="https://github.com/user-attachments/assets/7dff8fb2-ee0d-4120-8b09-1fcc80ad57c7" alt="16-playbook-failed-0" width="500"><br><sub>Playbook run ending with failed=0</sub></td>
  </tr>
</table>

**Documentation:** I listed my web servers in Ansible's hosts file and wrote a playbook that installs Docker and starts the DVWA web app on both servers. A playbook describes the desired state, so I can configure many servers identically with one command instead of doing everything by hand. DVWA is intentionally vulnerable, which gives me something safe to attack later.

* **Files:** [`ansible/hosts`](ansible/hosts), [`ansible/pentest.yml`](ansible/pentest.yml)
* **Verification:** `ansible -m ping all` returned two green `SUCCESS` results, and the playbook run ended with `failed=0`.

### Step 11: Standard Load Balancer Optimization
<table>
  <tr>
    <td align="center"><img src="https://github.com/user-attachments/assets/dc80d6ad-a18e-4703-ad0f-9eaa23d10720" alt="17-BackendPool-Web1-and-Web2" width="420"><br><sub>Backend pool with Web-1 and Web-2</sub></td>
    <td align="center"><img src="https://github.com/user-attachments/assets/14856620-4f5d-47b7-9c68-e8fc5818c5a0" alt="19-loadbalancingRule" width="420"><br><sub>Load balancing rule (80 → 80)</sub></td>
  </tr>
</table>

<p align="center">
  <img src="https://github.com/user-attachments/assets/219f2caf-b5cf-4ea2-ab79-6a8210ad4576" alt="20-DVWA" width="600"><br>
  <sub>DVWA reached through the Load Balancer</sub>
</p>

<details>
<summary><b>More Load Balancer screenshots (health probe, backend pool)</b></summary>

<p align="center">
  <img src="https://github.com/user-attachments/assets/688ce25a-2d71-4fe9-aee9-fa011f869df6" alt="18-HealthProbeHTTP" width="650"><br>
  <img src="https://github.com/user-attachments/assets/53c611cc-7f7a-49e1-8c93-d3e850adb618" alt="18-HealthProbeHTTP-2" width="650"><br>
  <img src="https://github.com/user-attachments/assets/e3203fb1-3d2a-4743-a390-2d55680d594b" alt="21-BackendPool-2VM" width="650">
</p>

</details>

**Documentation:** I created a Load Balancer that spreads web traffic between Web-1 and Web-2. A health probe checks that the servers answer on port 80, and the balancer stops sending traffic to a server that is down. This gives higher availability and represents the HTTP arrows in the diagram.

---

## Phase 4: Monitoring Zone Setup (ELK and Peering)

### Step 12: Network Peering
<p align="center">
  <img src="https://github.com/user-attachments/assets/5bd42877-0cca-4f51-9272-c6d157938886" alt="22-peering-CONNECTED" width="700">
</p>

**Documentation:** I created a separate network for ELK (10.1.0.0/16) and connected it to the web network with peering. Keeping the log server in its own network separates monitoring from the things being monitored. Peering lets the networks talk privately without the traffic crossing the internet.

### Step 13: ELK Server & NSG Configuration
<table>
  <tr>
    <td align="center"><img src="https://github.com/user-attachments/assets/3f93fe7b-cf6e-4615-9285-ee8339bad513" alt="23-ELK-NSG-Rules" width="400"></td>
    <td align="center"><img src="https://github.com/user-attachments/assets/7452cb30-67d2-4080-94db-a55efe227d4b" alt="24-ELK-NSG-Rules" width="440"></td>
  </tr>
</table>

**Documentation:** I created the ELK server and its own NSG, which only allows SSH from the JumpBox, Kibana from my IP, and log collection from the web network. This follows the principle of least privilege: every port is opened only for those who need it. The server got 4 GB RAM because Elasticsearch is memory-hungry.

| Rule | Source | Port |
|------|--------|------|
| SSH from JumpBox | 10.0.0.4 | 22 |
| Kibana from my IP | My IP | 5601 |
| Incoming logs | 10.0.0.0/16 | 9200 |

### Step 14: Docker Compose Infrastructure
<table>
  <tr>
    <td align="center"><img src="https://github.com/user-attachments/assets/3cee9762-51fd-47a5-a43b-30e058a2fae6" alt="24-dockerps-showing-both-containers" width="420"><br><sub><code>docker ps</code> showing both containers</sub></td>
    <td align="center"><img src="https://github.com/user-attachments/assets/5eadb25b-570b-440e-bdde-58b1645450fd" alt="Elk-and-Kibana-are-running" width="420"><br><sub>Elasticsearch and Kibana running</sub></td>
  </tr>
</table>

**Documentation:** I installed Docker on the ELK server and started Elasticsearch (stores the logs) and Kibana (displays them) with Docker Compose. Compose starts both services from a single file, which makes the setup repeatable. I turned off Elastic security to keep the lab simple, which is only acceptable because the NSG limits access, and I would never do this in production.

* **File:** [`elk/docker-compose.yml`](elk/docker-compose.yml)

### Step 15: Log Shipping Attempt (Filebeat) and Kibana Limitation
<p align="center">
  <img src="https://github.com/user-attachments/assets/89596abd-1a21-4fbc-9c97-84549027ccc5" alt="26-KIBANA-EncryptedTunnel" width="700">
</p>

**Documentation:** I wrote an Ansible playbook ([`ansible/beats.yml`](ansible/beats.yml)) to install Filebeat on both web servers and point it at Elasticsearch on the ELK server, so that system and login logs would appear in Kibana. However, the Filebeat deployment hit package download problems, and I could not get log data flowing into Elasticsearch/Kibana. Instead of getting stuck, I verified the services manually over SSH and with `curl -I`, and moved the log analysis to the Linux terminal (Phase 5).

**Result:** The full "web servers → Filebeat → Elasticsearch → Kibana dashboard" chain was **not** completed. This is documented honestly here and in the Lessons Learned section.

---

## Phase 5: Threat Simulation and Live Terminal Log Auditing

> *Legal & Compliance Disclaimer: All security scanning and penetration testing simulations documented below were performed strictly within an isolated private lab environment against owned and authorized cloud assets. Microsoft permits security testing against your own Azure resources, but no denial-of-service (DoS) attacks were performed. Never scan or attack systems you do not own.*

Because no logs reached Kibana, the attacks below were audited directly in the terminal.

### Step 16: Reconnaissance & Enumeration (Port Scan)
<table>
  <tr>
    <td align="center"><img src="https://github.com/user-attachments/assets/891a0f6a-c4cf-445f-9610-4a1c134f5dce" alt="31-starting-nmap" width="360"><br><sub>Starting the scan</sub></td>
    <td align="center"><img src="https://github.com/user-attachments/assets/025f73ea-63ce-4191-b1b8-675624654641" alt="27-nmap-result-port22-and-80-open" width="480"><br><sub>Result: ports 22 and 80 open</sub></td>
  </tr>
</table>

**Documentation:** I scanned my web servers with nmap to see which ports and services they expose. This is the first thing an attacker does (reconnaissance), and it shows what is visible to anyone who gets into the network. I learned that every open port is a possible way in.

### Step 17: SSH Brute-Force Log Detection
<p align="center">
  <img src="https://github.com/user-attachments/assets/69e7ac7f-2b0d-46cc-a6d7-5a2c37297bd5" alt="32-Failed-logins-from-a-fake-user" width="750">
</p>

**Documentation:** I simulated a brute-force attack by trying to log in with a wrong user many times. Instead of relying on a graphical SIEM frontend, I pivoted directly to the Linux authentication layer to inspect the raw system logs (`/var/log/auth.log`). The system successfully logged each unauthorized attempt, revealing the timestamps and the JumpBox source IP (`10.0.0.4`). This demonstrates how a SOC analyst detects brute-force behaviors through rapid-succession authentication failures.

### Step 18: Web Application Exploitation (SQL Injection)
<table>
  <tr>
    <td align="center"><img src="https://github.com/user-attachments/assets/02518c07-447e-4e7d-bad8-25f951a10bcf" alt="DVWA-inside-the-terminal" width="400"><br><sub>DVWA accessed from the terminal</sub></td>
    <td align="center"><img src="https://github.com/user-attachments/assets/2d18f9ca-e51c-4dc1-bfb2-fd31942c55e3" alt="33-Logs-afterwards-web-server-logs" width="380"><br><sub>Web server logs after the attack</sub></td>
  </tr>
</table>

**Documentation:** I exploited a SQL injection in DVWA by submitting malicious string signatures into an input field to bypass database logic. To verify the attack traces at the root level, I extracted the web server access logs using `sudo docker logs dvwa`. The output documents the exploit patterns alongside the Azure Load Balancer Active Health Agent (`168.63.129.16`) polling the environment. This highlights that input validation errors leave forensic traces, proving why application metrics must be monitored.

### Step 19: Security Analysis & Log Evaluation
**Documentation:** Instead of generating a standard graphical chart inside the Kibana interface, I mapped out and dissected the threat telemetry directly within the core Linux system files and container logs. This provided an advanced, hands-on understanding of raw indicators of compromise (IoC). Relying on direct system diagnostics ensured total visibility of the attack surface, completing the defensive engineering lifecycle: infrastructure build, penetration simulation, and low-level detection auditing.

---

## Phase 6: Infrastructure Decommissioning and Clean Up

### Step 20: Resource Group Destruction
**Documentation:** I deleted the whole Resource Group when the lab was finished. In the cloud you pay for what is running, so cleaning up is an important habit to prevent unexpected cloud costs.

---

## Lessons Learned and Technical Solutions
1. **Virtual NIC Lockup Issues:** Discovered that Azure restricts swapping network interfaces on hot-running virtual nodes. Solved the roadblock by deleting the compute wrapper while preserving the underlying OS disk, and provisioning a new server natively inside the target subnet to prevent data loss.
2. **Automated Pipeline Fault Resilience:** Navigated external vendor mirror package changes during the Filebeat deployment. By bypassing the automated interface, I performed manual core logging verifications via SSH and `curl -I`, proving that true security operations depend on mastery of the CLI and operating system architecture when interfaces fail.
3. **Log Ingestion Into Elasticsearch/Kibana Failed:** The ELK containers were deployed, but I could not get logs from the web servers into Elasticsearch and Kibana. If I repeated the lab, I would check:
   * that both containers are actually running and not restarting (`docker ps`, `docker logs`)
   * that `vm.max_map_count` was applied
   * that the VM has enough memory for Elasticsearch plus Kibana
   * that the ELK NSG rules and my current public IP are correct for ports 5601 and 9200
   * that Filebeat installed correctly on both web servers
4. **No Outbound Internet Without a NAT Gateway:** The web servers have no public IP, so they could not download packages until I attached a NAT Gateway to the subnet.
5. **Source-IP NSG Rules Need Maintenance:** Because the NSG only allows my own IP, the rules must be updated whenever my home IP changes.

---

## Cleanup & Security Notes
* The entire lab ran in one Resource Group (`SOC-Lab`), which was deleted afterwards to avoid costs.
* Elastic security (`xpack.security.enabled=false`) was disabled for lab simplicity only. Do not do this in production.
* DVWA is intentionally vulnerable and must never be exposed to the open internet.
* Public IP addresses are blurred or omitted in screenshots, and no private keys are stored in this repository.
