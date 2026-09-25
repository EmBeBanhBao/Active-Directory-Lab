# Active-Directory-Lab
<div align="center">

# 🛡️ Active Directory Attack & Detection Environment

**HOME SECURITY LAB PROJECT**

*Building an on-premises Windows domain with centralized logging, then simulating and detecting an intrusion*

![Active Directory](https://img.shields.io/badge/Active%20Directory-0078D4?style=for-the-badge&logo=windows&logoColor=white)
![Splunk](https://img.shields.io/badge/Splunk%20SIEM-000000?style=for-the-badge&logo=splunk&logoColor=white)
![Sysmon](https://img.shields.io/badge/Sysmon-5E5E5E?style=for-the-badge&logo=windows&logoColor=white)
![MITRE ATT&CK](https://img.shields.io/badge/MITRE%20ATT%26CK-C00000?style=for-the-badge)
![Atomic Red Team](https://img.shields.io/badge/Atomic%20Red%20Team-B22222?style=for-the-badge)

**Domain:** `myproject.local` &nbsp;|&nbsp; **Network:** `192.168.10.0/24`

*Self-directed home lab · SOC / Blue Team focus*

</div>

---

## 📑 Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Objectives](#2-objectives)
3. [Environment & Architecture](#3-environment--architecture)
4. [Implementation](#4-implementation)
5. [Attack Simulation & Detection](#5-attack-simulation--detection)
6. [Outcomes & Skills Demonstrated](#6-outcomes--skills-demonstrated)

---

## 1. Executive Summary

This project documents the design, build, and testing of a fully self-contained Active Directory environment, created from scratch on-premises to practice both offensive and defensive security skills. The lab pairs a Windows domain with a Splunk-based detection pipeline so that endpoint activity can be collected, centralized, and searched the way it would be in a real Security Operations Center (SOC).

Four virtual machines were connected together using a NATNetwork: a Windows Server 2022 domain controller, a Windows 10 target workstation, an Ubuntu Server host running Splunk Enterprise, and a Kali Linux attacker machine. Sysmon and the Splunk Universal Forwarder were deployed to the Windows endpoints to stream detailed telemetry into a dedicated Splunk index.

With the environment in place, an adversary scenario was executed end-to-end: a Remote Desktop (RDP) brute-force attack was launched from Kali against a domain user, and the resulting authentication activity was then hunted and confirmed in Splunk. Atomic Red Team was used to run additional MITRE ATT&CK techniques, which surfaced both a working detection and a visibility gap — demonstrating the value of adversary emulation for validating logging coverage.

---

## 2. Objectives

- Design a logical network diagram and addressing scheme before building, to plan data flow and practice whiteboarding an environment.
- Stand up a multi-VM lab (domain controller, workstation, SIEM, and attacker) on a single isolated virtual network.
- Deploy Active Directory Domain Services, promote a server to a domain controller, create organizational units and users, and join a workstation to the domain.
- Build a centralized logging pipeline using Sysmon, the Splunk Universal Forwarder, and Splunk Enterprise.
- Simulate a realistic attack (RDP brute force), then detect and analyze it in Splunk using Windows security event codes.
- Use Atomic Red Team and the MITRE ATT&CK framework to emulate techniques, validate detections, and identify gaps in visibility.

---

## 3. Environment & Architecture

The lab was built on VirtualBox using a NAT Network, all guests share one subnet (`192.168.10.0/24`) while retaining outbound internet access. Building the diagram first made the data flow explicit: Windows endpoints forward logs to Splunk, while the attacker sits on the same segment to reach the target over RDP.

### 3.1 Virtual Machines

| Host | Operating System | Role | Resources |
|---|---|---|---|
| **ADDC01** | Windows Server 2022 | Active Directory Domain Controller / DNS | 2 vCPU · 4 GB RAM |
| **TARGET-PC** | Windows 10 Pro | Domain-joined target workstation | 1 vCPU · 4 GB RAM |
| **Splunk** | Ubuntu Server 22.04 | Splunk Enterprise (SIEM / indexer) | 2 vCPU · 8 GB RAM |
| **Kali** | Kali Linux | Attacker machine | 2 vCPU · 4 GB RAM |



### 3.2 IP Addressing Scheme

A static addressing plan was defined up front and applied to every host except the target's initial DHCP lease, which was later fixed to a static address to avoid conflicts.

| Host | IP Address | DNS | Notes |
|---|---|---|---|
| Gateway | `192.168.10.1` | — | NAT Network gateway |
| ADDC01 (DC) | `192.168.10.7` | `8.8.8.8` | Domain controller & DNS server |
| Splunk | `192.168.10.10` | `8.8.8.8` | Web UI on `:8000`, receiver on `:9997` |
| TARGET-PC | `192.168.10.100` | `192.168.10.7` | DNS points to DC to resolve the domain |
| Kali | `192.168.10.250` | `8.8.8.8` | Attacker |



### 3.3 Toolset

| Tool | Purpose |
|---|---|
| **VirtualBox** | Type-2 hypervisor hosting all four VMs on a NAT Network |
| **Active Directory Domain Services** | Directory, authentication (Kerberos), and authorization for the domain |
| **Splunk Enterprise** | SIEM — indexing, searching, and analyzing collected logs |
| **Splunk Universal Forwarder** | Lightweight agent shipping endpoint logs to Splunk on `:9997` |
| **Sysmon** | Deep Windows telemetry (process, network, and more), using a community configuration (Olaf Hartong's) |
| **Atomic Red Team** | Adversary emulation mapped to MITRE ATT&CK techniques |
| **Crowbar** | Brute-force tool used to attack RDP from Kali |
| **rockyou.txt** | Password wordlist used to seed the brute-force attempt |
| **MITRE ATT&CK** | Framework used to select and reference techniques |

---

## 4. Implementation

### Phase 1 — Design & Network Topology

The environment was first mapped in a diagram (built in draw.io) covering the two servers, the target and attacker workstations, a switch, a router, and the internet. The diagram fixed the domain name (`myproject.local`), the subnet, and each host's role and IP, and marked the forwarding paths from the Windows endpoints to Splunk. Planning this before building clarified how data would flow and served as the reference for the rest of the project.

<!-- 📷 PHOTO PLACEHOLDER: replace the path below with your network diagram image -->
<p align="center">
  <img src="images/network-diagram.png" alt="Network topology diagram" width="700">
</p>

### Phase 2 — Virtual Machine Provisioning

All four guests were installed in VirtualBox. Windows 10 and Windows Server 2022 were installed from Microsoft ISOs, Kali Linux from its prebuilt VirtualBox image, and Splunk's host from the Ubuntu Server 22.04 ISO. VirtualBox downloads were integrity-checked against published SHA-256 hashes before installation. Every VM's adapter was attached to the shared NAT Network so the hosts could communicate on `192.168.10.0/24`.

<!-- 📷 PHOTO PLACEHOLDER: replace the path below with your Server Manager / VirtualBox screenshot -->
<p align="center">
  <img src="images/vm-provisioning.png" alt="Server Manager and running VMs" width="700">
</p>

### Phase 3 — Telemetry & Splunk Pipeline

Splunk Enterprise was installed on the Ubuntu host and configured with a static IP via netplan and enabled to start on boot. A dedicated index named `endpoint` was created, and a receiving port was opened on `9997` so the indexer could accept forwarded data.

On each Windows endpoint (the target, and the domain controller) two agents were installed:

- **Sysmon**, loaded with a community configuration to generate rich process, network, and system telemetry.
- **The Splunk Universal Forwarder**, pointed at the indexer (`192.168.10.10:9997`).

A custom `inputs.conf` was placed in the forwarder's local directory to specify which logs to ship — the Application, Security, and System event logs plus the Sysmon operational log — all routed to the `endpoint` index.

**Verification:** a search of `index=endpoint` in Splunk returned live events with the host TARGET-PC and the expected Security, Application, System, and Sysmon source types — confirming the pipeline was working end-to-end.

<!-- 📷 PHOTO PLACEHOLDER: replace the path below with your Splunk index=endpoint search screenshot -->
<p align="center">
  <img src="images/splunk-endpoint-index.png" alt="Splunk search of index=endpoint" width="700">
</p>

### Phase 4 — Active Directory Deployment

1. Assigned the domain controller a static IP (`192.168.10.7`) and verified connectivity to the internet and to the Splunk host.
2. Installed the Active Directory Domain Services (AD DS) role via Server Manager.
3. Promoted the server to a domain controller by creating a new forest, myDFIR.local.
4. Created organizational units (IT and HR) and two domain users — Jack Reacher(reacher) and Tony Swan(tonyswan) — to mirror a departmental structure.
5. Pointed the Windows 10 workstation's DNS to the domain controller, then joined it to `myproject.local` and logged in as a domain user.

---

## 5. Attack Simulation & Detection

### 5.1 RDP Brute-Force Attack

Remote Desktop was enabled on the target workstation for the two domain users. On Kali, the crowbar tool was used to brute-force RDP against the user tsmith. A short password list was built from the first lines of `rockyou.txt`, with the account's real password appended so the attack would succeed and generate a clean success-and-failure pattern to hunt.

```bash
crowbar -b rdp -u tsmith -C password.txt -s 192.168.10.100/32
```

Crowbar iterated the wordlist and reported a successful RDP login on the correct password, confirming the attack against the domain account.

### 5.2 Detection in Splunk

The generated telemetry was then investigated in Splunk. Filtering the `endpoint` index to the last 15 minutes for the targeted user surfaced the attack immediately through two Windows security event codes:

| Event Code | Meaning | Observed | Significance |
|:---:|---|:---:|---|
| **4625** | An account failed to log on | 20 events | Repeated failures clustered within seconds — a classic brute-force signature |
| **4624** | An account was successfully logged on | 1 event | The single success; expanded to reveal source workstation "Kali" and its IP |

The tight timestamp clustering of the twenty 4625 failures, immediately followed by one 4624 success originating from the Kali host, is exactly the pattern an analyst would alert on. Windows event codes were cross-referenced against Ultimate Windows Security to confirm their meaning.

### 5.3 Adversary Emulation with Atomic Red Team

Atomic Red Team was installed on the target to run additional MITRE ATT&CK techniques and test detection coverage. A Microsoft Defender exclusion was set so test artifacts would not be removed mid-run.

| Technique | ATT&CK ID | Result |
|---|:---:|---|
| Create Account: Local Account | `T1136.001` | Telemetry generated for the new local account; visible in Splunk after indexing |
| Command & Scripting Interpreter: PowerShell | `T1059.001` | PowerShell execution (exec-bypass / no-profile) captured and searchable in Splunk |

> [!TIP]
> This phase reinforced a core blue-team lesson: adversary emulation validates whether logging actually captures an attacker's actions. Where an expected event does not appear, that absence is itself a finding — a visibility gap to be closed before a real intrusion exploits it.

---

## 6. Outcomes & Skills Demonstrated

The completed lab is a working, repeatable environment for practicing detection engineering and threat hunting. The project exercised the full loop a SOC analyst works within: instrument endpoints, centralize logs, emulate an adversary, and detect the activity.

### Skills demonstrated

- **Active Directory:** AD DS installation, domain-controller promotion, forest/domain creation, OUs, users, and domain join.
- **SIEM & log management:** Splunk Enterprise setup, index and receiver configuration, Universal Forwarder deployment, and `inputs.conf` tuning.
- **Endpoint telemetry:** Sysmon deployment with a community configuration for high-fidelity Windows logging.
- **Threat detection & hunting:** Building Splunk searches and interpreting Windows security event codes (4624 / 4625) to identify a brute-force attack.
- **Offensive security:** RDP brute-forcing with crowbar and wordlist preparation from `rockyou.txt`.
- **Adversary emulation:** Atomic Red Team execution mapped to MITRE ATT&CK, including detection-gap analysis.
- **Infrastructure & networking:** VirtualBox virtualization, NAT networking, static IP and DNS configuration, netplan, and file-integrity verification.
