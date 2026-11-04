# Azure SOC Lab: Load-Balanced Web Servers and Log Analysis

I built this lab to learn how a small cloud environment is set up, secured and monitored, and to see what attacks look like in the logs. It runs in Microsoft Azure and has a JumpBox, two load-balanced web servers, Ansible for automation and an ELK server for monitoring. After building it, I attacked my own lab and checked what showed up in the logs.

<p align="center">
  <img src="https://github.com/user-attachments/assets/8bf88395-c5af-4c48-8a40-ee0b279be124" alt="Architecture diagram of the Azure SOC lab" width="700">
</p>

<p align="center"><sub><i>The planned architecture. Everything was deployed, but I could not get the Filebeat → Elasticsearch → Kibana log pipeline working (see <a href="#step-15-filebeat-and-kibana-problems">Step 15</a>), so I analyzed the attacks directly in the terminal.</i></sub></p>

---

## Table of Contents
1. [Project Overview](#project-overview)
2. [Project Status](#project-status)
3. [Technologies Used](#technologies-used)
4. [Repository Structure](#repository-structure)
5. [Phase 1: Azure Foundation](#phase-1-azure-foundation)
6. [Phase 2: JumpBox and Ansible](#phase-2-jumpbox-and-ansible)
7. [Phase 3: Web Servers and Load Balancer](#phase-3-web-servers-and-load-balancer)
8. [Phase 4: ELK Monitoring Setup](#phase-4-elk-monitoring-setup)
9. [Phase 5: Attacking the Lab](#phase-5-attacking-the-lab)
10. [Phase 6: Clean Up](#phase-6-clean-up)
11. [Lessons Learned](#lessons-learned)
12. [Security Notes](#security-notes)

---

## Project Overview
A SIEM (Security Information and Event Management) is a program that collects logs from all your computers so you can spot attacks. In this project I built a network, a safe entrance, two web servers, a receptionist and a monitoring room. Then I simulated attacks against my own lab and looked at the logs they left behind.

The setup can be explained as a castle:
* **Microsoft Azure:** the land where the castle stands
* **Virtual Network:** the castle wall
* **NSG (Network Security Group):** the guards at the gate
* **JumpBox:** the only door in
* **Web-1 and Web-2:** two identical shops
* **Load Balancer:** the receptionist
* **ELK:** the security camera room
* **Ansible:** a robot helper

## Project Status
Everything in the lab was deployed and tested, except the log pipeline. Elasticsearch and Kibana were running on the ELK server, but I could not get Filebeat to send logs from the web servers, so I analyzed the attacks directly in the terminal instead (see Step 15).

## Technologies Used
* **Cloud:** Microsoft Azure (virtual networks, peering, NSGs, NAT gateway, availability set, load balancer)
* **Automation:** Ansible running in a Docker container
* **Containers:** Docker and Docker Compose
* **Monitoring:** Elasticsearch, Kibana and Filebeat (ELK)
* **Security tools:** Nmap, OpenSSH, DVWA (Damn Vulnerable Web Application)
* **Operating system:** Ubuntu Server 22.04 LTS

## Repository Structure
```text
soc-azure-lab/
├── README.md      # Screenshots are embedded directly in this file
├── images/        # Optional local copies of screenshots
├── ansible/       # Hosts file and playbooks
│   ├── hosts
│   ├── pentest.yml
│   └── beats.yml
└── elk/           # Docker Compose file for Elasticsearch and Kibana
    └── docker-compose.yml
```

Click any screenshot to enlarge it.

---

## Phase 1: Azure Foundation

### Step 1: Resource Group
<p align="center">
  <img src="https://github.com/user-attachments/assets/68bde916-1157-49ae-8b33-93a5152b97ee" alt="01-resource-group" width="650">
</p>

I created a resource group called SOC-Lab to keep everything in the project in one place. When the lab was finished, I could delete the whole group at once, which avoids leftover costs.

### Step 2: Virtual Network
<p align="center">
  <img src="https://github.com/user-attachments/assets/9b2e3d64-dbb2-431a-a849-d4f56e604cf5" alt="02-vnet" width="650">
</p>

I created a virtual network with the address space 10.0.0.0/16 and one subnet, 10.0.0.0/24. The servers can talk to each other inside the network without being exposed to the internet. The JumpBox and the web servers go in this subnet.

### Step 3: Network Security Group (NSG)
<table>
  <tr>
    <td align="center"><img src="https://github.com/user-attachments/assets/14ec25c0-ab37-4387-9b7a-157104b9d3bf" alt="03-nsg-rules" width="400"><br><sub>Inbound security rules</sub></td>
    <td align="center"><img src="https://github.com/user-attachments/assets/020567f8-522a-4d6c-8d6f-9a4d5c977d01" alt="04-subnet-association" width="400"><br><sub>NSG attached to the subnet</sub></td>
  </tr>
</table>

I created an NSG that only allows SSH (port 22) and HTTP (port 80) from my own IP address, and attached it to the subnet. An NSG works like a firewall. Because only my IP is allowed, bots on the internet cannot reach the servers. If my home IP changes, the rules have to be updated.

### Step 4: SSH Key
I generated an SSH key pair on my computer. The public key goes on the servers and the private key stays with me. Keys are safer than passwords because a bot cannot guess them.

---

## Phase 2: JumpBox and Ansible

### Step 5: JumpBox
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

I created the JumpBox, a VM with a public IP that is the only way in from the internet. The other servers have no public IP and can only be reached through it, so there is only one entrance to protect. I gave it a static private IP (10.0.0.4) so the address does not change.

### Step 6: Docker and the Ansible Container
<p align="center">
  <img src="https://github.com/user-attachments/assets/c7164fb9-5191-4936-8c3e-51ef81cb48ab" alt="08-Docker-Installed" width="650">
</p>

I installed Docker on the JumpBox and started a container with Ansible inside it. Docker bundles a program together with what it needs, so I did not have to install the tools directly on the server. Ansible is what I use to configure the other servers.

### Step 7: SSH Key for Ansible
<p align="center">
  <img src="https://github.com/user-attachments/assets/83e0ecf2-b21b-42fb-be7c-c21931c62c48" alt="09-Ansible-Container" width="650"><br>
  <img src="https://github.com/user-attachments/assets/41482f64-14f5-4119-88ae-c684a2e23ef1" alt="10-Inside-Ansible-Container" width="650">
</p>

I generated a separate key pair inside the Ansible container. The container is what logs in to the other servers, so its public key is the one that goes on them. This lets Ansible connect without passwords.

---

## Phase 3: Web Servers and Load Balancer

### Step 8: Web Servers and NAT Gateway
<table>
  <tr>
    <td align="center"><img src="https://github.com/user-attachments/assets/3a3632e7-9408-4500-89ba-a21982870ab8" alt="11-The-VM-List" width="420"><br><sub>VM list</sub></td>
    <td align="center"><img src="https://github.com/user-attachments/assets/aa75d733-f4c6-44fe-9947-20cb6dfdd273" alt="12-Web1-Overview" width="420"><br><sub>Web-1 overview (no public IP)</sub></td>
  </tr>
</table>

I created two identical web servers with no public IP, in the same availability set. An availability set puts them on different hardware, so if one fails the other keeps running. A NAT gateway lets them download software without being reachable from the internet.

### Step 9: SSH to the Web Servers
<p align="center">
  <img src="https://github.com/user-attachments/assets/eb8db450-088d-451d-a421-636a4752793d" alt="13-azureadmin-at-Web-1" width="460"><br>
  <img src="https://github.com/user-attachments/assets/cf9285e4-535a-4930-83fd-d4a2339fe52c" alt="14-azureadmin-at-Web-2" width="460">
</p>

I logged in to both web servers over SSH from the Ansible container. This shows that the servers can be reached through the JumpBox even though they have no public IP.

### Step 10: Ansible Playbook and DVWA
<table>
  <tr>
    <td align="center"><img src="https://github.com/user-attachments/assets/0f5f3d2c-fa14-4675-8e64-f91b727a19c0" alt="15-ansible-ping-SUCCESS" width="320"><br><sub>Ansible ping: SUCCESS on both servers</sub></td>
    <td align="center"><img src="https://github.com/user-attachments/assets/7dff8fb2-ee0d-4120-8b09-1fcc80ad57c7" alt="16-playbook-failed-0" width="500"><br><sub>Playbook run ending with failed=0</sub></td>
  </tr>
</table>

I added the web servers to Ansible's hosts file and wrote a playbook that installs Docker and starts DVWA on both. A playbook describes the setup I want, so one command configures both servers the same way. DVWA is intentionally vulnerable, so it gives me something safe to attack later.

Files: [`ansible/hosts`](ansible/hosts) and [`ansible/pentest.yml`](ansible/pentest.yml)

### Step 11: Load Balancer
<table>
  <tr>
    <td align="center"><img src="https://github.com/user-attachments/assets/dc80d6ad-a18e-4703-ad0f-9eaa23d10720" alt="17-BackendPool-Web1-and-Web2" width="420"><br><sub>Backend pool with Web-1 and Web-2</sub></td>
    <td align="center"><img src="https://github.com/user-attachments/assets/14856620-4f5d-47b7-9c68-e8fc5818c5a0" alt="19-loadbalancingRule" width="420"><br><sub>Load balancing rule (80 → 80)</sub></td>
  </tr>
</table>

<p align="center">
  <img src="https://github.com/user-attachments/assets/219f2caf-b5cf-4ea2-ab79-6a8210ad4576" alt="20-DVWA" width="600"><br>
  <sub>DVWA reached through the load balancer</sub>
</p>

<details>
<summary>More load balancer screenshots (health probe, backend pool)</summary>

<p align="center">
  <img src="https://github.com/user-attachments/assets/688ce25a-2d71-4fe9-aee9-fa011f869df6" alt="18-HealthProbeHTTP" width="650"><br>
  <img src="https://github.com/user-attachments/assets/53c611cc-7f7a-49e1-8c93-d3e850adb618" alt="18-HealthProbeHTTP-2" width="650"><br>
  <img src="https://github.com/user-attachments/assets/e3203fb1-3d2a-4743-a390-2d55680d594b" alt="21-BackendPool-2VM" width="650">
</p>

</details>

I created a load balancer that spreads web traffic between Web-1 and Web-2. A health probe checks that the servers answer on port 80, and the balancer stops sending traffic to one that is down.

---

## Phase 4: ELK Monitoring Setup

### Step 12: Network Peering
<p align="center">
  <img src="https://github.com/user-attachments/assets/5bd42877-0cca-4f51-9272-c6d157938886" alt="22-peering-CONNECTED" width="700">
</p>

I created a separate network for ELK (10.1.0.0/16) and connected it to the web network with peering. This keeps monitoring separate from the machines being monitored, and the traffic between the two networks stays private.

### Step 13: ELK Server and NSG
<table>
  <tr>
    <td align="center"><img src="https://github.com/user-attachments/assets/3f93fe7b-cf6e-4615-9285-ee8339bad513" alt="23-ELK-NSG-Rules" width="400"></td>
    <td align="center"><img src="https://github.com/user-attachments/assets/7452cb30-67d2-4080-94db-a55efe227d4b" alt="24-ELK-NSG-Rules" width="440"></td>
  </tr>
</table>

I created the ELK server with its own NSG. Each port is only open to the one who needs it:

| Rule | Source | Port |
|------|--------|------|
| SSH from JumpBox | 10.0.0.4 | 22 |
| Kibana from my IP | My IP | 5601 |
| Incoming logs | 10.0.0.0/16 | 9200 |

I used a VM with 4 GB of RAM because Elasticsearch needs a lot of memory.

### Step 14: Elasticsearch and Kibana with Docker Compose
<table>
  <tr>
    <td align="center"><img src="https://github.com/user-attachments/assets/3cee9762-51fd-47a5-a43b-30e058a2fae6" alt="24-dockerps-showing-both-containers" width="420"><br><sub><code>docker ps</code> showing both containers</sub></td>
    <td align="center"><img src="https://github.com/user-attachments/assets/5eadb25b-570b-440e-bdde-58b1645450fd" alt="Elk-and-Kibana-are-running" width="420"><br><sub>Elasticsearch and Kibana running</sub></td>
  </tr>
</table>

I installed Docker on the ELK server and started Elasticsearch (stores the logs) and Kibana (shows them) with Docker Compose. I turned off Elastic's security to keep the lab simple. That is only acceptable because the NSG limits access, and I would not do it in production.

File: [`elk/docker-compose.yml`](elk/docker-compose.yml)

### Step 15: Filebeat and Kibana Problems
<p align="center">
  <img src="https://github.com/user-attachments/assets/89596abd-1a21-4fbc-9c97-84549027ccc5" alt="26-KIBANA-EncryptedTunnel" width="700">
</p>

I wrote an Ansible playbook ([`ansible/beats.yml`](ansible/beats.yml)) to install Filebeat on both web servers and send their logs to Elasticsearch. The Filebeat download failed, and I never got logs flowing into Elasticsearch and Kibana. I checked the services by hand over SSH and with `curl -I`, and did the log analysis in the terminal instead (Phase 5).

So the pipeline web servers → Filebeat → Elasticsearch → Kibana was **not** completed.

---

## Phase 5: Attacking the Lab

> All scanning and attacks below were done against my own resources in my own Azure lab. Microsoft allows security testing of your own Azure resources, but no denial-of-service attacks, and I did none. Never scan or attack systems you do not own.

Because Kibana had no data, I read the logs directly on the servers.

### Step 16: Port Scan with Nmap
<table>
  <tr>
    <td align="center"><img src="https://github.com/user-attachments/assets/891a0f6a-c4cf-445f-9610-4a1c134f5dce" alt="31-starting-nmap" width="360"><br><sub>Starting the scan</sub></td>
    <td align="center"><img src="https://github.com/user-attachments/assets/025f73ea-63ce-4191-b1b8-675624654641" alt="27-nmap-result-port22-and-80-open" width="480"><br><sub>Result: ports 22 and 80 open</sub></td>
  </tr>
</table>

I scanned the web servers with nmap to see which ports and services they expose. Scanning is the first thing an attacker does, and it shows what anyone inside the network can see. Only ports 22 and 80 were open. Every open port is a possible way in.

### Step 17: Failed SSH Logins
<p align="center">
  <img src="https://github.com/user-attachments/assets/69e7ac7f-2b0d-46cc-a6d7-5a2c37297bd5" alt="32-Failed-logins-from-a-fake-user" width="750">
</p>

I simulated a brute-force attack by trying to log in as a user that does not exist, many times in a row. Since Kibana had no data, I read `/var/log/auth.log` on the web server. Every attempt is logged with a timestamp and the source IP, which is the JumpBox (`10.0.0.4`). Many failed logins in a short time is how password guessing looks in the logs.

### Step 18: SQL Injection Against DVWA
<table>
  <tr>
    <td align="center"><img src="https://github.com/user-attachments/assets/02518c07-447e-4e7d-bad8-25f951a10bcf" alt="DVWA-inside-the-terminal" width="400"><br><sub>DVWA accessed from the terminal</sub></td>
    <td align="center"><img src="https://github.com/user-attachments/assets/2d18f9ca-e51c-4dc1-bfb2-fd31942c55e3" alt="33-Logs-afterwards-web-server-logs" width="380"><br><sub>Web server logs after the attack</sub></td>
  </tr>
</table>

I exploited a SQL injection in DVWA by entering a malicious string in an input field, which made the database return data it should not. Afterwards I read the web server's access log with `sudo docker logs dvwa`. It shows the attack requests, and also the Azure load balancer's health probe (`168.63.129.16`) polling the server. The attack leaves clear traces in the web logs, which is why they should be sent to a SIEM.

### Step 19: What the Logs Showed
Since Kibana had no data, I read the raw logs directly: `/var/log/auth.log` for the failed SSH logins and `docker logs dvwa` for the web attack. This showed me what the signs of an attack look like in the raw data, before any dashboard is involved. It would still have been better to have everything in one place in Kibana.

---

## Phase 6: Clean Up

### Step 20: Deleting the Resource Group
I deleted the whole resource group when I was done. In the cloud you pay for what is running, so cleaning up matters.

---

## Lessons Learned
1. **Network interface problem:** Azure does not let you swap the network interface on a running VM. I deleted the VM but kept its OS disk, and created a new VM directly in the right subnet, so no data was lost.
2. **Filebeat download failed:** The Filebeat package could not be downloaded because the vendor's package source had changed. Instead of giving up, I checked the services by hand over SSH and with `curl -I`. It reminded me that you need to be comfortable in the command line when the tools fail.
3. **Logs never reached Elasticsearch and Kibana:** The ELK containers were running, but I could not get logs from the web servers into them. If I repeated the lab, I would check:
   * that both containers are running and not restarting (`docker ps`, `docker logs`)
   * that `vm.max_map_count` was applied
   * that the VM has enough memory for Elasticsearch and Kibana
   * that the ELK NSG rules and my current public IP are correct for ports 5601 and 9200
   * that Filebeat installed correctly on both web servers
4. **NAT gateway:** The web servers have no public IP, so they could not download anything until I attached a NAT gateway to the subnet.
5. **IP-based NSG rules:** Because the NSG only allows my own IP, the rules must be updated whenever my home IP changes.

---

## Security Notes
* The whole lab ran in one resource group (`SOC-Lab`), which was deleted afterwards to avoid costs.
* Elastic security (`xpack.security.enabled=false`) was turned off for this lab only. Do not do this in production.
* DVWA is intentionally vulnerable and must never be exposed to the open internet.
* Public IP addresses are blurred or left out of the screenshots, and no private keys are stored in this repository.
