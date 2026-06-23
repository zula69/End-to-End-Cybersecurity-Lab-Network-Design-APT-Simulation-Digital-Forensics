# 🔍 APT Simulation & Digital Forensics Investigation — ICBT Campus Network

A comprehensive cybersecurity project that simulates an Advanced Persistent Threat (APT) attack against a fully configured campus network, captures forensic evidence, and performs complete digital forensic analysis across network, memory, and disk layers.

---

## 🎯 Objective

This project replicates a real-world corporate network environment based on an ICBT university campus topology built in Cisco Packet Tracer, then simulates an APT attack against a live vulnerable machine in a controlled 4-VM lab. The full incident response lifecycle is demonstrated:

- 🏛️ **Campus Network Design** — Multi-VLAN hierarchical network with DMZ, ACLs, NAT, and routing
- ⚔️ **APT Attack Simulation** — Reconnaissance, exploitation, persistence, and data exfiltration
- 📡 **Evidence Collection** — Passive network capture, memory acquisition, and disk imaging
- 🔬 **Digital Forensics** — Network, memory, and disk forensic analysis using industry-standard tools
- 📋 **Incident Response** — Full IR report with IOCs, timeline, and remediation recommendations

---

## 🧠 Skills Demonstrated

- Designing and configuring a multi-tier hierarchical campus network (Cisco Packet Tracer)
- VLAN segmentation, inter-VLAN routing, trunking, and ACL implementation
- APT attack chain execution: reconnaissance → exploitation → persistence → exfiltration
- Passive evidence collection using tcpdump without alerting the attacker
- Network forensics with Wireshark: protocol analysis, stream reconstruction, IOC identification
- Memory forensics with Volatility3
- Disk forensics with The Sleuth Kit (TSK): partition analysis, file system examination, deleted file recovery
- Super timeline generation using Plaso (62,631 events extracted)
- File carving with Foremost
- Forensic chain of custody — read-only disk mounting, evidence hashing

---

## 🛠️ Tools Used

| Tool | Purpose | Phase |
|------|---------|-------|
| **Cisco Packet Tracer** | Campus network design and simulation | Network Design |
| **Nmap** | Network reconnaissance and port scanning | Reconnaissance |
| **Metasploit Framework** | Exploitation of Samba usermap_script (CVE-2007-2447) | Exploitation |
| **Netcat** | Data exfiltration via port 5555 | Exfiltration |
| **tcpdump** | Passive network traffic capture on Kali #2 | Evidence Collection |
| **LiME** | Live memory acquisition from victim machine | Evidence Collection |
| **dd** | Forensic disk image acquisition | Evidence Collection |
| **Wireshark** | PCAP analysis, protocol inspection, stream reconstruction | Network Forensics |
| **Volatility3** | Memory dump analysis | Memory Forensics |
| **The Sleuth Kit (TSK)** | Disk forensics — mmls, fls, icat, fsstat, mactime | Disk Forensics |
| **Foremost** | File carving from disk image | Disk Forensics |
| **Plaso / log2timeline** | Super timeline generation (62,631 events) | Timeline Analysis |
| **tcpflow** | TCP stream reconstruction and data extraction | Network Forensics |
| **p0f** | Passive OS fingerprinting from PCAP | Network Forensics |

---

## 🔑 Key Findings

| Finding | Evidence Source | Detail |
|---------|----------------|--------|
| Nmap reconnaissance confirmed | Wireshark SYN filter | Rapid SYN packets to multiple ports from 192.168.56.10 |
| vsftpd exploit attempted but failed | Wireshark + Metasploit logs | CVE-2011-2523 not triggered |
| Samba exploit successful | Wireshark port 445 stream | CVE-2007-2447 payload confirmed |
| Root shell on port 6200 | Wireshark TCP stream | Attacker commands visible in plaintext |
| Backdoor user created | /etc/passwd on disk | aptbackdoor:x:1003:1003::/home/aptbackdoor:/bin/bash |
| Data exfiltration confirmed | Wireshark port 5555 stream | Employee data visible in plaintext |
| 62,631 filesystem events | Plaso disk timeline | Complete activity log of victim disk |

---

## ⚡ Challenges & Mitigations

| # | Challenge | What Happened | How It Was Fixed |
|---|-----------|---------------|-----------------|
| 1 | **vsftpd exploit failed** | Metasploit vsftpd_234_backdoor module did not trigger a shell | Switched to Samba usermap_script exploit (CVE-2007-2447) which successfully returned a root shell |
| 2 | **LiME kernel header mismatch** | LiME failed to compile due to Kali running a cutting-edge kernel with no available headers | Installed stable kernel packages with matching headers and rebooted before recompiling LiME |
| 3 | **Memory evidence lost** | Volatile memory data (processes, connections) had disappeared by the time Volatility3 analysis began | Documented as a key lesson: memory forensics must be performed live during the incident window |
| 4 | **Autopsy 2.x image format error** | apt-installed Autopsy (v2.24) returned "image format type could not be determined" | Replaced with The Sleuth Kit CLI tools (mmls, fls, icat, mactime) — the same engine Autopsy wraps |
| 5 | **Disk image using wrong partition offset** | TSK commands returned no results when using default offset | Used mmls to identify correct start sector (63 for boot, LVM at 482013) and applied -o flag |
| 6 | **LVM partition not accessible via TSK** | Root filesystem was inside LVM — fls returned no attack artifacts | Activated LVM with vgscan/vgchange and mounted root volume read-only for direct filesystem access |
| 7 | **NetworkMiner rendering broken on Parrot KDE** | Mono + KDE caused black/blank GUI on all launch attempts | Replaced with tcpflow for stream reconstruction and p0f for OS fingerprinting — native Linux tools |
| 8 | **Plaso CSV empty on first run** | psort used > redirection which corrupted the output | Used the -w flag instead of shell redirection to write the CSV file correctly |



## 🖧 Part 1 — Campus Network Design (Cisco Packet Tracer)

A reference campus network was designed and configured in Cisco Packet Tracer using a three-tier hierarchical model (access layer, destribution layer and core layer) with five VLANs, a DMZ, an edge router, and an ISP router.

### Network Topology Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    ICBT CAMPUS NETWORK                      │
├─────────────────────────────────────────────────────────────┤
│  VLAN 10 → Students    → 10.10.10.0/24                     │
│  VLAN 20 → Staff       → 10.20.20.0/24                     │
│  VLAN 30 → Admin       → 10.30.30.0/24                     │
│  VLAN 40 → Library     → 10.40.40.0/24                     │
│  VLAN 50 → Guest WiFi  → 10.50.50.0/24                     │
│  DMZ     → Servers     → 172.16.1.0/24                     │
└─────────────────────────────────────────────────────────────┘
```

---

### 🔌 Device Setup & Labelling

Configured and labelled all network devices in the topology before beginning configuration. Each device was clearly identified by role and location.

- Labelled network devices and physical connections
<!-- Screenshot: labelled devices -->
<img width="1920" height="1080" alt="PT all labled devices" src="https://github.com/user-attachments/assets/ec3cecff-5645-4c56-b200-03ea7c55765f" />

- Connected all devices in the topology
<!-- Screenshot: connected devices -->
<img width="1920" height="1080" alt="PT all connected devices" src="https://github.com/user-attachments/assets/e5dd32b0-e08b-4276-bb10-cada69fa480c" />

---

### 🎓 Student Lab Switch (VLAN 10)

Configured the access layer switch for the Student Lab, assigned VLAN 10, and set up trunk ports to connect to the distribution layer.

- Changing hostname of Student Lab switch
<!-- Screenshot -->
<img width="1920" height="1080" alt="changing hostanme of student lab" src="https://github.com/user-attachments/assets/5218d38f-f733-4bf2-8905-c9d8be74cf87" />

- Configuring VLAN 10 (Students — 10.10.10.0/24)
<!-- Screenshot -->
<img width="1920" height="1080" alt="fao1  vlan 10" src="https://github.com/user-attachments/assets/fe61209d-2008-495b-b3a3-ed3e6ded8aa4" />

<img width="1920" height="1080" alt="fao3  vlan 10" src="https://github.com/user-attachments/assets/1365bb53-dad6-4c90-8280-ee00bbefc856" />

- Creating trunk port on Student Lab switch
<!-- Screenshot -->
<img width="1920" height="1080" alt="creating trunk port" src="https://github.com/user-attachments/assets/4c04e064-2650-41bf-8b5f-c8662b4722a2" />


- Confirming VLAN 10 is active and working
<!-- Screenshot -->
<img width="1920" height="1080" alt="confirming student lab vlan working" src="https://github.com/user-attachments/assets/47359a13-b905-41cc-94ee-94b2f97f6253" />

---

### 👨‍💼 Staff Switch (VLAN 20)

- Changing hostname of Staff switch
<!-- Screenshot -->
<img width="1920" height="1080" alt="creatong hostname of staff" src="https://github.com/user-attachments/assets/de0b2bb1-82e6-484a-96d6-492b39ea76a6" />

- Creating VLAN 20 (Staff — 10.20.20.0/24)
<!-- Screenshot -->
<img width="1920" height="1080" alt="creating vlan 20" src="https://github.com/user-attachments/assets/2257abb5-383d-4119-8407-cf1e36c559f4" />

<img width="1920" height="1080" alt="fa 01 and fa 03 vlan 20" src="https://github.com/user-attachments/assets/40a5fb50-7361-41de-99da-2189f7519fc3" />


- Creating trunk port on Staff switch
<!-- Screenshot -->
<img width="1920" height="1080" alt="trunk vlan 20" src="https://github.com/user-attachments/assets/db332dbc-40ad-4083-9827-f2848e33deb7" />


- Confirming VLAN 20 is active
<!-- Screenshot -->
<img width="1920" height="1080" alt="confirmation vlan 20" src="https://github.com/user-attachments/assets/d3c50680-5be6-4f4e-aba7-1a8f9b9c2924" />

---

### 🔐 Admin Switch (VLAN 30)

- Changing hostname of Admin switch
<!-- Screenshot -->
<img width="1920" height="1080" alt="changinghostname of admin" src="https://github.com/user-attachments/assets/3bcd16a1-fb6e-40d2-8378-7f8b854d439b" />


- Creating VLAN 30 (Admin — 10.30.30.0/24)
<!-- Screenshot -->
<img width="1920" height="1080" alt="creating vlan 30" src="https://github.com/user-attachments/assets/4d62cfba-2f5f-4174-b5a0-ccd0c78d4855" />

<img width="1920" height="1080" alt="configuring interfaces to acces vlan 30" src="https://github.com/user-attachments/assets/cc4d24a0-f75f-4a88-b027-045b8c0d57d5" />

- Creating trunk port on Admin switch
<!-- Screenshot -->
<img width="1920" height="1080" alt="trunk" src="https://github.com/user-attachments/assets/d144884a-9b99-4006-8a35-36e31caab7c6" />


- Confirming VLAN 30 is active
<!-- Screenshot -->
<img width="1920" height="1080" alt="confirming vlan active" src="https://github.com/user-attachments/assets/94123bf8-d2c5-47d5-b866-9cebda082d3c" />

---

### 📚 Library Switch (VLAN 40)

- Changing hostname of Library switch
<!-- Screenshot -->
<img width="1920" height="1080" alt="changing hostname of library switch4" src="https://github.com/user-attachments/assets/1c26cf6a-8bc2-4052-865e-4469fe730474" />


- Creating VLAN 40 (Library — 10.40.40.0/24)
<!-- Screenshot -->
<img width="1920" height="1080" alt="creating vlan 40" src="https://github.com/user-attachments/assets/aac73a40-80a9-4839-a685-eecbff7f873e" />

<img width="1920" height="1080" alt="adding interfaces to ccessport vlan 40" src="https://github.com/user-attachments/assets/e0dc7efa-8847-4b15-b1e8-fbb5ad832b7a" />


- Trunk port setup on Library switch
<!-- Screenshot -->
<img width="1920" height="1080" alt="trunk port setup" src="https://github.com/user-attachments/assets/65193bec-1793-4ae8-b35b-542fac1633fb" />


- Confirming VLAN 40 is active
<!-- Screenshot -->
<img width="1920" height="1080" alt="library vlan active" src="https://github.com/user-attachments/assets/bce8c046-36c5-4545-95e2-ab5e04691f41" />

---

### 📶 Guest WiFi Switch (VLAN 50)

- Changing hostname of Guest WiFi switch
<!-- Screenshot -->
<img width="1920" height="1080" alt="hostname guest wifi switch 5" src="https://github.com/user-attachments/assets/c2bcad6f-3929-4928-9529-11512c9d4119" />

- Creating VLAN 50 (Guest WiFi — 10.50.50.0/24)
<!-- Screenshot -->
<img width="1920" height="1080" alt="creating vlan 50 guest wifi" src="https://github.com/user-attachments/assets/9374faea-960d-40e2-8e94-a28705c744f9" />

<img width="1920" height="1080" alt="adding interface to vlan 50" src="https://github.com/user-attachments/assets/d76d4ac8-ca80-4f55-b79b-9600e3448a9a" />

- Trunk port setup on Guest WiFi switch
<!-- Screenshot -->
<img width="1920" height="1080" alt="trunk 50" src="https://github.com/user-attachments/assets/53b1aa60-1011-43ba-928c-06e75ba6b25a" />

- Confirming VLAN 50 is active
<!-- Screenshot -->
<img width="1920" height="1080" alt="vlan confirmation 50" src="https://github.com/user-attachments/assets/597d4ef3-1079-4f20-93f6-1151ea7bacf6" />

---

### 🔀 Distribution Switch 1

The first distribution layer switch aggregates traffic from Student Lab, Staff, and Admin access switches and trunks upward to the core.

- Changing hostname of Distribution Switch 1
<!-- Screenshot -->
<img width="1920" height="1080" alt="hostname desswitch1" src="https://github.com/user-attachments/assets/b2290c21-1525-4662-ba03-dd634b399bbe" />

- Creating VLANs on Distribution Switch 1
<!-- Screenshot -->
<img width="1920" height="1080" alt="creating vlans in desswitch1" src="https://github.com/user-attachments/assets/027ace0b-11e1-4380-a910-0de1a9ec53fe" />

- Confirming VLANs are active
<!-- Screenshot -->
<img width="1920" height="1080" alt="vlans active in destribution switch" src="https://github.com/user-attachments/assets/58e79409-ba66-4a4a-8173-8c530cbca605" />


- Configuring uplink trunk port (to Core Switch)
<!-- Screenshot -->
<img width="1920" height="1080" alt="uplink trunk" src="https://github.com/user-attachments/assets/d46af589-162b-4005-9228-71c64794d6b8" />


- Configuring downlink trunk ports (to access switches)
<!-- Screenshot -->
<img width="1920" height="1080" alt="downlink switchport pallehata yana tika trunk" src="https://github.com/user-attachments/assets/65a2a599-9c7e-4b38-800a-1086480d470a" />

<img width="1920" height="1080" alt="2" src="https://github.com/user-attachments/assets/ef49544e-889a-461c-967a-38e4736d8a27" />

<img width="1920" height="1080" alt="downlink switchport pallehata yana tika trunk 3" src="https://github.com/user-attachments/assets/1488cb00-d197-464f-bde9-e4c0489dbac0" />

---

### 🔀 Distribution Switch 2

The second distribution switch aggregates Library and Guest WiFi traffic.

- Changing hostname of Distribution Switch 2
<!-- Screenshot -->
<img width="1920" height="1080" alt="changing hostname of desswitch 2" src="https://github.com/user-attachments/assets/4d1060a3-9a5a-491c-9040-6f54213c3329" />


- Creating VLANs on Distribution Switch 2
<!-- Screenshot -->
<img width="1920" height="1080" alt="adding vlans to destr switch 2" src="https://github.com/user-attachments/assets/1c87f3f0-a6f0-4004-afac-b9f9a39329da" />

- Configuring uplink trunk port (to Core Switch)
<!-- Screenshot -->
<img width="1920" height="1080" alt="setting uyplink trunk port" src="https://github.com/user-attachments/assets/1e92d654-e843-4a9e-91e5-c7419a308268" />

- Configuring downlink trunk ports
<!-- Screenshot -->
<img width="1920" height="1080" alt="settingup downlink trunk ports" src="https://github.com/user-attachments/assets/747f76c7-b263-4cbd-b13d-acdba34057c8" />

---

### 🖥️ Core Switch

The core switch provides Layer 3 inter-VLAN routing, assigns SVI IP addresses for each VLAN, and trunks to both distribution switches.

- Changing hostname of Core Switch
<!-- Screenshot -->
<img width="1920" height="1080" alt="changing the hostname of coreswitch" src="https://github.com/user-attachments/assets/388f4d39-a951-470b-a1f3-1b267e1bb12f" />

- Adding all VLANs to Core Switch
<!-- Screenshot -->
<img width="1920" height="1080" alt="adding vlans to core switch" src="https://github.com/user-attachments/assets/d84da289-bf4b-42da-9d36-8c38d0640e1d" />

<img width="1920" height="1080" alt="confirmation" src="https://github.com/user-attachments/assets/6b440c78-3eee-4d8c-b780-d457a15cfe96" />

- Setting up interfaces and IP addresses for each VLAN
<!-- Screenshot -->
<img width="1920" height="1080" alt="setting up interfaces and ips for each vlan in core switch" src="https://github.com/user-attachments/assets/3789b155-12de-422e-a307-cd7e9bd4f69b" />

<img width="1920" height="1080" alt="setting up interfaces and ips for each vlan in core switch           2" src="https://github.com/user-attachments/assets/62cafa5f-24dc-4093-a6fb-a15605c97b06" />

<img width="1920" height="1080" alt="setting up interfaces and ips for each vlan in core switch   3" src="https://github.com/user-attachments/assets/c0c16908-cae5-4516-be91-3dc6e6ffb41b" />

<img width="1920" height="1080" alt="4" src="https://github.com/user-attachments/assets/d9e250ff-6ce9-4e57-ba07-e4f01af82c9a" />

<img width="1920" height="1080" alt="5" src="https://github.com/user-attachments/assets/1f6f8832-706e-4597-b1fe-1f598c3ac6ba" />

- Trunk to Distribution Switch 1
<!-- Screenshot -->
<img width="1920" height="1080" alt="trunking to destribution switches 1" src="https://github.com/user-attachments/assets/8743f5f3-7f34-49d3-b7e1-6bd3efa72c66" />

<img width="1920" height="1080" alt="trunking to destribution switches 1    verifying" src="https://github.com/user-attachments/assets/c87530ac-c595-4a3a-8c3e-cecfd248ec03" />


- Trunk to Distribution Switch 2
<!-- Screenshot -->
<img width="1920" height="1080" alt="trunking to destribution switches 2" src="https://github.com/user-attachments/assets/3ce9c58e-9fcd-4a9b-857e-4536fa2d0311" />

<img width="1920" height="1080" alt="verified" src="https://github.com/user-attachments/assets/ac40d86f-f895-4997-913e-7991ef11d645" />

---

### 🔥 Firewall 

> ⚠️ The original Cisco ASA 5505 firewall was replaced with a 2911 router using ACLs and NAT due to Packet Tracer limitations with ASA configuration.

<img width="1920" height="1080" alt="setting up 2911 router as the firewwall beause assa firewall buggey as hell" src="https://github.com/user-attachments/assets/018c8148-c133-4a24-b344-28186f438b08" />

### Edge router

- Changing hostname of Edge Router
<!-- Screenshot -->
<img width="1920" height="1080" alt="changing the hostname of the edge router" src="https://github.com/user-attachments/assets/2bf7e0d0-89ff-42e1-96c8-36a1eb9f26cc" />

- Assigning IP addresses on Edge Router interfaces
<!-- Screenshot -->
<img width="1920" height="1080" alt="assinging IP address for ISP port on edge router" src="https://github.com/user-attachments/assets/76eecc37-b93e-465b-a296-d156f05065ed" />

<img width="1920" height="1080" alt="assigning IP address for student firewall router in edge router" src="https://github.com/user-attachments/assets/6cdf2028-f465-498c-af07-805982299152" />

- Permitting all VLANs to access ISP router (NAT/routing)
<!-- Screenshot -->
<img width="1920" height="1080" alt="translate all 10 x x x addresses to the public IP" src="https://github.com/user-attachments/assets/bb96b1fd-4e7a-4690-b1a0-54a4275a3d7d" />

- Setting default route to ISP
<!-- Screenshot -->
<img width="1920" height="1080" alt="stting default route to ISP" src="https://github.com/user-attachments/assets/c7ff2efc-b0d9-4c6c-933a-d7f4fca4e8ed" />

- Setting return route back to campus network
<!-- Screenshot -->
<img width="1920" height="1080" alt="setting route back to campus" src="https://github.com/user-attachments/assets/081fbe9a-55f6-4f87-a8d3-3c3481db0b18" />

---

### 🌐 ISP Router

- Configuring hostname of ISP Router
<!-- Screenshot -->
<img width="1920" height="1080" alt="changing the hostname of the isp router" src="https://github.com/user-attachments/assets/436db109-5841-4335-8984-b011730d8d5d" />

- Adding IP addresses on ISP Router
<!-- Screenshot -->
<img width="1920" height="1080" alt="configuring the edge router interface in ISP router" src="https://github.com/user-attachments/assets/cfed838d-82f1-433b-9d5c-4252c7fe277e" />

- Configuring routing to forward campus traffic
<!-- Screenshot -->
<img width="1920" height="1080" alt="traffic routing command" src="https://github.com/user-attachments/assets/a836da15-6959-4f35-803f-b7256777af20" />


---

### 🖥️ Server Configuration (DMZ — 172.16.1.0/24)

- Configuring Web Server
<!-- Screenshot -->
<img width="1920" height="1080" alt="ip configuration of the web server" src="https://github.com/user-attachments/assets/a23ea245-454a-4c3a-b1b9-eb44e8efca18" />

<img width="1920" height="1080" alt="turning on http server in web server" src="https://github.com/user-attachments/assets/9180b1d5-421e-4ed6-a2bf-e9a36e3cb71f" />

- Configuring DNS Server
<!-- Screenshot -->
<img width="1920" height="1080" alt="configuring DNS server" src="https://github.com/user-attachments/assets/f010d869-5ddf-42d5-8abd-7b90c3fb012c" />

<img width="1920" height="1080" alt="adding the DNS server" src="https://github.com/user-attachments/assets/a4f7b6c7-f7bb-48ba-b781-5a9869459f85" />

- Configuring Mail Server
<!-- Screenshot -->
<img width="1920" height="1080" alt="configuring mail server" src="https://github.com/user-attachments/assets/536aa1ab-504b-4491-96ba-8fb40e2460a0" />

<img width="1920" height="1080" alt="turning the mail server on" src="https://github.com/user-attachments/assets/531fd2fe-12c7-4d90-9371-b7a8a46fefc4" />

---

### 🗺️ Final Network Topology

<!-- Screenshot: full Packet Tracer topology -->
<img width="1920" height="1080" alt="Final network topology" src="https://github.com/user-attachments/assets/9c8b1bc4-b2d8-4554-a925-3ed3b23b3c31" />

---

### ✅ Connectivity Verification — Ping Tests

All VLANs were verified to communicate correctly through the three-tier hierarchy.

- Testing connectivity 
<!-- Screenshot -->
- from student 2 to student 1
<img width="1920" height="1080" alt="student 2 to student 1" src="https://github.com/user-attachments/assets/25c95488-f802-4f3e-95dd-a142a85c1a78" />

- from staff to student 1
<img width="1920" height="1080" alt="staff to student 1" src="https://github.com/user-attachments/assets/8ef6c3a0-f6e7-4d09-9635-05c64fb930df" />

- from admin to student 1
<img width="1920" height="1080" alt="admin to student" src="https://github.com/user-attachments/assets/0c0ac8fd-5341-46b7-bdcb-1a2ae18c6d7d" />

- from library to student 1
<img width="1920" height="1080" alt="library to student" src="https://github.com/user-attachments/assets/471cc288-8c66-451f-9994-f78e966ad452" />

- from guest to student 1
<img width="1920" height="1080" alt="guest to  student" src="https://github.com/user-attachments/assets/c65ef7ee-dc16-459c-a7bd-e57324996a0b" />

- from web server to student 1
<img width="1920" height="1080" alt="web to student" src="https://github.com/user-attachments/assets/148378d2-d4b4-4d02-ba45-35fde24488ac" />

- from DNS to student 1
<img width="1920" height="1080" alt="DNS to student" src="https://github.com/user-attachments/assets/2dc8fde6-fdda-44cb-86a5-38003e300b72" />

- from mail server to student 1
<img width="1920" height="1080" alt="mail to student" src="https://github.com/user-attachments/assets/2139644e-7f2d-421f-8382-393bfbf91c06" />

- from firewall to student 1
<img width="1920" height="1080" alt="firewall to student" src="https://github.com/user-attachments/assets/ab9de653-e91a-4bda-9208-0bb070c14912" />

- from edge router to student 1
<img width="1920" height="1080" alt="edge router to student" src="https://github.com/user-attachments/assets/4a77c0c1-8e5d-47dd-8b21-f40c86d50e06" />

- from ISP to student 1
<img width="1920" height="1080" alt="ISP router to studemt" src="https://github.com/user-attachments/assets/7a79db3b-a6c8-49d3-8bdc-0eb6a25929de" />

---

### 🔒 Access Control Lists (ACLs)

Four ACLs were implemented on the Core Switch to enforce the campus security policy.

#### ACL 1 — Guest WiFi Policy
> Guests can access the internet but are completely blocked from all internal departments

<img width="1920" height="1080" alt="configuring access list on guest  1" src="https://github.com/user-attachments/assets/84961a97-e0fc-4d81-a375-eaa1d074fadf" />

- Blocking other IPs
<img width="1920" height="1080" alt="guest blocking others" src="https://github.com/user-attachments/assets/e1ac3959-2059-41cf-ae72-a2ee2187c629" />

- Allowing access only to the internet
<img width="1920" height="1080" alt="permitting only internet" src="https://github.com/user-attachments/assets/d531007f-18ae-4a6a-b6e4-747d0b75a632" />

<img width="1920" height="1080" alt="saving guest" src="https://github.com/user-attachments/assets/283f73a2-eb0d-4b8d-81e2-76fd3c4598d1" />


Guests (VLAN 50) → ❌ VLAN 10 (Students)
<img width="1920" height="1080" alt="ping to student unsuccessful" src="https://github.com/user-attachments/assets/57dc2f24-de09-4eb9-95ea-98ec77209143" />

Guests (VLAN 50) → ❌ VLAN 20 (Staff)
<img width="1920" height="1080" alt="ping to staff unsuccessful" src="https://github.com/user-attachments/assets/a172b5f0-aac7-426c-8053-4a5088ff4a00" />

Guests (VLAN 50) → ❌ VLAN 30 (Admin)
<img width="1920" height="1080" alt="ping to admin unsuccessful" src="https://github.com/user-attachments/assets/c7823058-073a-439b-838d-9d6444194b7a" />

Guests (VLAN 50) → ❌ servers
<img width="1920" height="1080" alt="ping to servers unsuccessful" src="https://github.com/user-attachments/assets/41599f54-e09d-415c-95b8-51f7c7a9cd54" />

Guests (VLAN 50) → ✅ Internet only
<img width="1920" height="1080" alt="ping to edge router successfull internet working as f" src="https://github.com/user-attachments/assets/bf578518-fb33-4cbd-a25d-6464817c71b6" />


#### ACL 2 — Student Policy
> Students can access Library and DMZ but cannot reach Admin or Staff systems
<img width="1920" height="1080" alt="creating student policy" src="https://github.com/user-attachments/assets/ccd718ca-8dd0-41c5-b1b4-563049ee7376" />

- Blocking staff and admin LANs
<img width="1920" height="1080" alt="denying staff and admin" src="https://github.com/user-attachments/assets/e7aca8e2-7364-4b3e-badc-42a36f117b1e" />

- Permitting library and the internet
<img width="1920" height="1080" alt="permitting library servers and internet" src="https://github.com/user-attachments/assets/7133598c-22b8-4561-a69a-101b18a9a8bd" />

<img width="1920" height="1080" alt="saving the config" src="https://github.com/user-attachments/assets/756ecf4b-261e-4c6c-ac9f-835046dfa487" />



Students (VLAN 10) → ✅ VLAN 40 (Library)
<img width="1920" height="1080" alt="ping to library" src="https://github.com/user-attachments/assets/645546e3-1f71-44a1-8aef-bc59c3f0ad4a" />

Students (VLAN 10) → ✅ Edge router
<img width="1920" height="1080" alt="ping to edge router" src="https://github.com/user-attachments/assets/82ba1116-ab02-45a0-ba29-63b1e7b24396" />


Students (VLAN 10) → ❌ VLAN 30 (Admin)
<img width="1920" height="1080" alt="ping to admin not success" src="https://github.com/user-attachments/assets/c844c802-28ba-4223-bea6-b4615f0cec72" />

Students (VLAN 10) → ❌ VLAN 20 (Staff)
<img width="1920" height="1080" alt="ping to staff not working" src="https://github.com/user-attachments/assets/233e9568-71c5-4c8f-a1d4-3d5e45217593" />



#### ACL 3 — DMZ Server Protection (firewall configuration)
> Only specific traffic allowed to reach DMZ. Only IT Staff can SSH into servers.

- Allow public IPs to access the web server
<img width="1920" height="1080" alt="firewall acl 1 allow web server access to public" src="https://github.com/user-attachments/assets/d2f83579-b91e-4f5f-bf07-60a6b1b99f5a" />

<img width="1920" height="1080" alt="firewall acl 1 allow web server access to public 2" src="https://github.com/user-attachments/assets/ce94b6ff-d132-4438-9a2b-0e57adf64104" />

- Allow public IPs to access the DNS
<img width="1920" height="1080" alt="allow dns t" src="https://github.com/user-attachments/assets/2bdf0265-59bd-4763-a13d-cf82d58c5775" />

- Permit only the staff to SSH servers
<img width="1920" height="1080" alt="permit only staff to ssh servers" src="https://github.com/user-attachments/assets/97deed4a-fec5-464a-9b79-3497803848fe" />

- Block access from any other IPs to the servers
<img width="1920" height="1080" alt="block others from accessing servers" src="https://github.com/user-attachments/assets/366a3d16-da5b-4e64-9c53-1de09f967eb5" />

- Permitting all other traffic through the firewall
<img width="1920" height="1080" alt="permit other all trafic through firewall" src="https://github.com/user-attachments/assets/351cc9b8-dc94-447a-90b7-695007f2275e" />

- Saving the configuration
<img width="1920" height="1080" alt="saving the firewall config" src="https://github.com/user-attachments/assets/e8028319-46bd-4ff5-8e2a-798c9aefdc5e" />


#### ACL 4 — Admin Office Protection
> Only Admin and Staff VLANs can access the Admin network

- Allow admin pcs to work within the admin LAN
<img width="1920" height="1080" alt="admin acl allow admin to admin" src="https://github.com/user-attachments/assets/46b860a5-7c45-493e-944e-5f9b669d8018" />

- Permit staff to work with admin
<img width="1920" height="1080" alt="permit staff to work with admin" src="https://github.com/user-attachments/assets/310bf76e-d6ed-4dd0-bc4c-043ba0a754a1" />

- Block student, library and guest IPs
<img width="1920" height="1080" alt="deny student library and guest to admin" src="https://github.com/user-attachments/assets/46787572-b9f2-4961-974b-3255b3973048" />

- Permit other traffic
<img width="1920" height="1080" alt="permit other traffic" src="https://github.com/user-attachments/assets/a4fe6bd0-c7c8-41ac-bb60-6188378a5854" />

- Saving the configuration
<img width="1920" height="1080" alt="saving the config admin" src="https://github.com/user-attachments/assets/a5941807-30b3-4798-82e3-09e5e1163b1e" />

- ACL verification using `show access-lists` on Core Switch
<!-- Screenshot -->
<img width="1920" height="1080" alt="accesslists verification" src="https://github.com/user-attachments/assets/96247777-125b-4413-ab2a-f028576d869d" />


---

## 💻 Part 2 — VM Lab Setup & APT Attack Simulation

### Lab Architecture

```
┌─────────────────────────────────────────────────────────┐
│              HOST-ONLY NETWORK: 192.168.56.0/24         │
│                                                         │
│  ┌──────────────┐          ┌──────────────────────────┐ │
│  │  Kali Linux  │ ──attack─▶  Metasploitable2 Victim  │ │
│  │  (Attacker)  │          │     192.168.56.13        │ │
│  │192.168.56.10 │          └──────────────────────────┘ │
│  └──────────────┘                      │                │
│                                        │ (traffic)      │
│  ┌──────────────┐          ┌───────────▼──────────────┐ │
│  │  Parrot OS   │◀─analyze─│     Kali Linux #2        │ │
│  │  (Forensics) │          │  (Network Watcher)       │ │
│  │192.168.56.12 │          │     192.168.56.11        │ │
│  └──────────────┘          └──────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

| VM | Role | OS | IP Address |
|----|------|----|------------|
| Kali #1 | Attacker | Kali Linux | 192.168.56.10 |
| Kali #2 | Network Watcher / Evidence Collector | Kali Linux | 192.168.56.11 |
| Metasploitable2 | Victim | Ubuntu (Metasploitable2) | 192.168.56.13 |
| Parrot OS | Forensic Workstation | Parrot OS 7.1 KDE | 192.168.56.12 |

---

### VM setup

- setting up metasploitable
<img width="1920" height="1080" alt="metasploit setup 1" src="https://github.com/user-attachments/assets/a76df881-66bd-48b8-9970-e6163a39b4cc" />

<img width="1920" height="1080" alt="metasloitable2" src="https://github.com/user-attachments/assets/c3689e65-7c9a-4832-82e6-33cf06285ce3" />

- All VMs setup
<img width="1920" height="1080" alt="all vms set" src="https://github.com/user-attachments/assets/20c2297f-66bc-4bd7-9492-6b7206894e68" />

### creating a host - only network and setting up ips for each VM

- creating a host only network
<img width="1920" height="1080" alt="creating a host only network" src="https://github.com/user-attachments/assets/8a39d4d3-9979-4499-872b-3a9dad76dbed" />

- applying the host-only adapter to all vms
<img width="1920" height="1080" alt="applyiing host only adapter parrot" src="https://github.com/user-attachments/assets/bf9dee84-5fd5-4b0f-8d38-7ede3af021d3" />

<img width="1920" height="1080" alt="applyiing host only adapter kali 2" src="https://github.com/user-attachments/assets/7d7691f8-8710-46c4-b192-d1711b6a5638" />

<img width="1920" height="1080" alt="applyiing host only adapter kali 1" src="https://github.com/user-attachments/assets/31162747-cd9c-471d-acb3-6a75541c9059" />

<img width="1920" height="1080" alt="applyiing host only adapter metasploit" src="https://github.com/user-attachments/assets/bc53eeaf-f7ce-4457-9795-08f2f7d8b775" />

- Setting up ip for kali 1
<img width="1920" height="1080" alt="setting up ip for kali 1" src="https://github.com/user-attachments/assets/e2766194-3ca3-4b8f-8ce5-2bf0dbf15e56" />

<img width="1920" height="1080" alt="setting up ip for kali 1     2" src="https://github.com/user-attachments/assets/2e4bcffa-205d-4deb-973b-e10a92210e48" />

<img width="1920" height="1080" alt="setting up ip for kali 1     3" src="https://github.com/user-attachments/assets/bfd22f1a-7f9d-46c5-8353-1732ad6a616c" />

- Setting up ip for kali 2
<img width="1920" height="1080" alt="setting up ip for kali 2" src="https://github.com/user-attachments/assets/e543fabf-2754-4fe9-9560-6ba58b63b39a" />

<img width="1920" height="1080" alt="setting up ip for kali 2   2" src="https://github.com/user-attachments/assets/517deb64-d2ae-4621-805e-06f2f3a55525" />

<img width="1920" height="1080" alt="setting up ip for kali 2   3" src="https://github.com/user-attachments/assets/3f98b5aa-620b-42d7-a694-ca5f10dc2bcb" />

- Setting up ip for parrot os vm
<img width="1920" height="1080" alt="setting up ip for parrot" src="https://github.com/user-attachments/assets/aaa4436a-b2a9-4800-8a46-b21f065a8386" />

<img width="1920" height="1080" alt="setting up ip for parrot 2" src="https://github.com/user-attachments/assets/f230854b-a3ac-4188-8e5d-497bec21a025" />

<img width="1920" height="1080" alt="setting up ip for parrot 3" src="https://github.com/user-attachments/assets/4f8c41fd-236c-4053-ab4d-44c78b01fe7a" />

- Setting up ip for metasploit
<img width="1920" height="1080" alt="setting up ip for metasploit" src="https://github.com/user-attachments/assets/c909512a-4755-48df-9788-4e1b72895099" />

<img width="1920" height="1080" alt="setting up ip for metasploit   2" src="https://github.com/user-attachments/assets/8494a1e5-8ffe-4f8a-a3c5-a646c2686377" />

<img width="1920" height="1080" alt="setting up ip for metasploit    3" src="https://github.com/user-attachments/assets/c560ea9c-c0a1-48d8-bc7a-b661237ac21c" />

### Step 1 — Network Verification

Verified all VMs can communicate over the host-only network before beginning the attack simulation.

- Pinging from Kali #1 to all VMs
<!-- Screenshot -->
<img width="1920" height="1080" alt="ping from kali 1 to kali 2" src="https://github.com/user-attachments/assets/2aa6ea03-8391-47ec-a019-309ebc031d3f" />

<img width="1920" height="1080" alt="ping from kali 1 to parot" src="https://github.com/user-attachments/assets/44d100ba-ca36-40f0-a345-233a7f5c2d50" />

<img width="1920" height="1080" alt="ping from kali 1 to metasploit" src="https://github.com/user-attachments/assets/89ca1ab2-6a32-4b0d-82c8-3dc2d7517e48" />

- Pinging from Kali #2 to all VMs
<!-- Screenshot -->
<img width="1920" height="1080" alt="ping from kali 2 to kali 1" src="https://github.com/user-attachments/assets/505245c8-5d10-4c72-b64f-14f700b6528e" />

<img width="1920" height="1080" alt="ping from kali 2 to parrot" src="https://github.com/user-attachments/assets/e1451e94-0dfc-4d0b-8090-e89679d91ba2" />

<img width="1920" height="1080" alt="ping from kali 2 to metasloit" src="https://github.com/user-attachments/assets/e0929b3f-f741-4a76-9fa3-0a40e890f681" />

- Pinging from Parrot OS to all VMs
<!-- Screenshot -->
<img width="1920" height="1080" alt="ping parrot to kali 1" src="https://github.com/user-attachments/assets/c03dc386-4909-4e88-9571-959469fc7f23" />

<img width="1920" height="1080" alt="ping from parrot to kali 2" src="https://github.com/user-attachments/assets/17e61766-9560-4902-9572-291faf41462d" />

<img width="1920" height="1080" alt="ping from parrot to metaslploit" src="https://github.com/user-attachments/assets/5e196413-72be-405a-a0f3-faa2d3bd73e7" />

- Pinging from metasploit to all VMs
<img width="1920" height="1080" alt="ping from meta to kali 1" src="https://github.com/user-attachments/assets/cf93bf3c-2455-4d54-a6ef-7062506292bd" />

<img width="1920" height="1080" alt="ping from meta to kali 2" src="https://github.com/user-attachments/assets/46f36f84-990a-4c5c-aa43-9a233e0cd282" />

<img width="1920" height="1080" alt="ping from meta to parot" src="https://github.com/user-attachments/assets/283b85c8-fc67-47bd-af58-d6238124a59c" />

---

### Step 2 — Evidence Collection Setup (Kali #2 Watcher)

Kali Linux #2 was configured as a **passive network monitor**. It silently captured all traffic on the network segment using tcpdump without sending any packets that could alert the attacker.

- Starting tcpdump to capture all network traffic
<!-- Screenshot -->
installing tcpdump
<img width="1920" height="1080" alt="kali 2 installing tcpdump" src="https://github.com/user-attachments/assets/abce049e-8aff-4e2a-8ef8-0aedcbcf9443" />

creating a folder to collect evidences on kali 2
<img width="1920" height="1080" alt="kali 2 creating a folder to store evidence" src="https://github.com/user-attachments/assets/cfec5485-e9d2-4748-a648-929d9d1fa598" />

<img width="1920" height="1080" alt="creating evidence structure kali 2" src="https://github.com/user-attachments/assets/a48d0d79-6dc6-4277-b4f9-51d4035dbd7a" />

running tcpdump on the host-only network
<img width="1920" height="1080" alt="running tcpdump in kali 2" src="https://github.com/user-attachments/assets/ed32a400-99dd-488d-9da8-39506f167eaf" />

- Collecting Nmap scan results from Kali #2 to Metasploitable2
<!-- Screenshot -->
<img width="1920" height="1080" alt="nmap scan for metasloitable2 from kali 2 to victim and saved it to a baseline txt evidence" src="https://github.com/user-attachments/assets/2a60080f-0403-4803-83d1-94816fc09820" />

- Collecting journalctl system logs as supporting evidence
<!-- Screenshot -->
<img width="1920" height="1080" alt="saving activities in the journbalctl in to a log file" src="https://github.com/user-attachments/assets/28b9ceb2-4296-4619-8af6-d534c41a7a49" />


---

### Step 3 — Reconnaissance (Kali #1 Attacker)

The attacker performed network reconnaissance using Nmap to identify open ports and running services on the victim machine.

- Nmap SYN scan to identify open ports on Metasploitable2
<!-- Screenshot -->
basic nmap port scan
<img width="1920" height="1080" alt="kali 1 basic recoassistance phase basic nmap scan" src="https://github.com/user-attachments/assets/d42c9a68-2f29-4848-930f-c1b0855fa8dd" />

Detailed nmap port scan
<img width="1920" height="1080" alt="kali 1 detailed nmap scan of metasploit" src="https://github.com/user-attachments/assets/a9f5b193-3b69-4b99-b480-949f8fc3dd68" />

saving nmap scan results
<img width="1920" height="1080" alt="kali 1 it saved to attack txt" src="https://github.com/user-attachments/assets/4b7df8a3-0717-4fc0-9bda-bd7e8eea64f1" />

**Key ports identified:**

| Port | Service | Version |
|------|---------|---------|
| 21 | FTP | vsftpd 2.3.4 |
| 22 | SSH | OpenSSH 4.7p1 |
| 139/445 | SMB/Samba | Samba 3.x |
| 80 | HTTP | Apache 2.2.8 |
| 3306 | MySQL | 5.0.51a |


known vulnerabilities scan
<img width="1920" height="1080" alt="kali 1 known ulnerbilities scan" src="https://github.com/user-attachments/assets/1f9a0657-f296-4b2a-aeeb-b70e300c9bcf" />

Saving known vulnerabilities scan
<img width="1920" height="1080" alt="kali 1 vulnerabilities text is saved" src="https://github.com/user-attachments/assets/3fc5990f-f54e-4bbd-8390-35b76d6d2dc7" />


### Step 4 — Exploitation (Kali #1 Attacker)

#### Attempt 1 — vsftpd 2.3.4 (Failed)

The attacker first attempted to exploit the vsftpd 2.3.4 backdoor vulnerability (CVE-2011-2523) using Metasploit. The exploit did not succeed on this target.

- Attempting vsftpd exploit — failed

#### Attempt 2 — Samba usermap_script (Succeeded ✅)

The attacker then targeted the Samba service using the usermap_script vulnerability (CVE-2007-2447), which allows unauthenticated remote command execution as root.

- Configuring Metasploit
<img width="1920" height="1080" alt="kali 1 current meta settings" src="https://github.com/user-attachments/assets/494a237f-f42d-45c2-b5e9-66914a550b73" />

<img width="1920" height="1080" alt="kali 1 meta samba" src="https://github.com/user-attachments/assets/c3e8728a-8e61-4b49-8d54-f9ece17ee3fb" />

- Running the exploit — root shell obtained
<!-- Screenshot -->
<img width="1920" height="1080" alt="whoami" src="https://github.com/user-attachments/assets/807e5112-87f9-46fd-a26e-8349f6db0dbe" />

---

### Step 5 — Persistence (Kali #1 Attacker)

With root access established, the attacker created a backdoor user account to maintain persistent access even if the initial shell was closed.

- Viewing victim's details ( hostname)
<img width="1920" height="1080" alt="hostname" src="https://github.com/user-attachments/assets/d050a4da-046a-4193-b804-09f0340f6afa" />

- Viewing victim's details ( operating system details )
<img width="1920" height="1080" alt="operating system details" src="https://github.com/user-attachments/assets/3a566f59-f089-4843-9c8c-61451119318c" />

- Viewing victim's details ( full user and group ids)
<img width="1920" height="1080" alt="shows full user and group ids" src="https://github.com/user-attachments/assets/f09092f0-e165-47b0-9be5-e643b7e1206f" />

- Viewing victim's details ( all user account hashes)
<img width="1920" height="1080" alt="shows all user accounts hashes" src="https://github.com/user-attachments/assets/1d9c905a-406f-44ad-a65d-7797a0e6331b" />

- Viewing victim's details ( all user home directories)
<img width="1920" height="1080" alt="shows all user home directories" src="https://github.com/user-attachments/assets/2296aa5b-fd71-4178-95aa-c06a9d473acc" />

- Viewing victim's details ( Viewing open ports)
<img width="1920" height="1080" alt="shows open ports netstat -tulnp" src="https://github.com/user-attachments/assets/2ae6f6e5-7474-4e04-889d-9865d92d32f8" />

- Viewing victim's details ( All running processes )
<img width="1920" height="1080" alt="ps aux shows all running processos" src="https://github.com/user-attachments/assets/4a18fad1-9cc9-43d2-bbd3-4e189b10e443" />

- Creating backdoor user account (aptbackdoor)
<!-- Screenshot -->
<img width="1920" height="1080" alt="adding a user in meta aptbackdoor" src="https://github.com/user-attachments/assets/fca72d25-ef89-4eb7-b53e-f70760267e52" />

<img width="1920" height="1080" alt="adding a password for aptbackdoor" src="https://github.com/user-attachments/assets/4ed0a8d6-e06d-445f-88f5-a5c0896ebcc2" />

- Verifying backdoor account in /etc/passwd
<!-- Screenshot -->
<img width="1920" height="1080" alt="verifying password and the user added" src="https://github.com/user-attachments/assets/bf6ef3f7-eecb-4f27-b341-e26160f33afa" />

```
aptbackdoor:x:1003:1003::/home/aptbackdoor:/bin/bash
```
- Adding user to sudors and giving the root access to aptbackdoor
<img width="1920" height="1080" alt="add to sudors and gives root access" src="https://github.com/user-attachments/assets/655544eb-8f5a-4f18-94d8-12dcc22f3127" />

- Openning a reverse shell backdoor
<img width="1920" height="1080" alt="opened a reverse shell backdoor in metasploit" src="https://github.com/user-attachments/assets/b70ef6e7-5826-4f27-ac61-8143eee03088" />

---

### Step 6 — Data Exfiltration (Kali #1 Attacker)

The attacker located and exfiltrated confidential employee records from the victim machine using Netcat on port 5555.

- Creating a directory to store stolen files
<img width="1920" height="1080" alt="creatd a directory to store stolen files" src="https://github.com/user-attachments/assets/ef44e067-567e-408f-a3c5-da3b92843c0e" />

- Copping sensitive data
<img width="1920" height="1080" alt="also copy real sentivie data to tmp confidential" src="https://github.com/user-attachments/assets/5421a0e1-d892-4501-a6ee-75efd5b52d3d" />

- Exfiltrating data via Netcat on port 5555
<!-- Screenshot -->
<img width="1920" height="1080" alt="created a listingg port on kali 1 to recieve data from metsploit" src="https://github.com/user-attachments/assets/36e6b846-24a3-424d-a104-2547ac8bae45" />

<img width="1920" height="1080" alt="sending the file to kali 1" src="https://github.com/user-attachments/assets/6f7159d4-6a01-41ad-bb50-5a954a9311bd" />

- Confirmation

<img width="1920" height="1080" alt="confirmation employee data recieved" src="https://github.com/user-attachments/assets/bf7449c1-fbc3-43cc-bf2d-d0cef8911d72" />

<img width="1920" height="1080" alt="passwd succesful" src="https://github.com/user-attachments/assets/df37f294-edf4-4fca-9026-fdc910d31768" />

<img width="1920" height="1080" alt="shadow successful" src="https://github.com/user-attachments/assets/2796b315-0999-4d2d-9cdb-ee0147e217ec" />

- Removing traces and folders
<img width="1920" height="1080" alt="removing traces" src="https://github.com/user-attachments/assets/71d63700-06b9-4b69-abd2-cf2d9877405a" />

<img width="1920" height="1080" alt="removing files" src="https://github.com/user-attachments/assets/4feec492-2471-46c5-b7e8-cbd021aabf88" />

<img width="1920" height="1080" alt="exit" src="https://github.com/user-attachments/assets/93a510c5-de0f-4e87-a8ca-7ebabe9ba5f1" />

### Step 7 — Evidence Transfer to Parrot OS

All evidence collected by Kali #2 was transferred securely to the Parrot OS forensic workstation via SCP for analysis.

- All evidence in kali 2
<img width="1920" height="1080" alt="all evidence in watcher kali 2" src="https://github.com/user-attachments/assets/8df0269b-8ec9-43a8-a1a2-48db1d234111" />

- Creating a directory in parrot os to store evidence
<img width="1920" height="1080" alt="create a directory in parrot os to store evience file" src="https://github.com/user-attachments/assets/69619886-b8ec-49de-a5b6-656adc4fa1c3" />

- Transferring PCAP, memory dump, disk image to Parrot OS
<!-- Screenshot -->
<img width="1920" height="1080" alt="obtaining evidence from kali 2 for forensics" src="https://github.com/user-attachments/assets/a84d5953-f5f5-49ba-b609-1509eb68a3d9" />

<img width="1920" height="1080" alt="verification" src="https://github.com/user-attachments/assets/07310ec9-352e-4671-940c-126087dcb54a" />


---

## 🔬 Part 3 — Digital Forensic Analysis (Parrot OS)

### Step 8 — Network Forensics (Wireshark)

The captured PCAP file was loaded into Wireshark on the Parrot OS workstation for deep packet inspection.

- Packet statistics overview
<!-- Screenshot -->
<img width="1920" height="1080" alt="statics of the packet" src="https://github.com/user-attachments/assets/28da7f3b-792b-473d-9621-9c7d827a1457" />

- Protocol hierarchy statistics
<!-- Screenshot -->
<img width="1920" height="1080" alt="list of protocol statics" src="https://github.com/user-attachments/assets/62a23fd7-cdef-497b-82ab-2824adc2558c" />

<img width="1920" height="1080" alt="lis of protocol statics  2" src="https://github.com/user-attachments/assets/65ba9547-3ea7-4931-9a37-ff8e694ef637" />

<img width="1920" height="1080" alt="list of protocol statics 3" src="https://github.com/user-attachments/assets/0127781c-7250-46c1-938a-8bf797e70e4f" />

- IPv4 conversations — identifying attacker-victim traffic
<!-- Screenshot -->
<img width="1920" height="1080" alt="statics conversation from who to who" src="https://github.com/user-attachments/assets/a4adbd74-acc7-4c9e-aff7-6e833adca6e0" />


- Traffic statistics by endpoint (ipv4 info)
<!-- Screenshot -->
<img width="1920" height="1080" alt="statics ipv4 info activity" src="https://github.com/user-attachments/assets/70898840-34a7-4ace-b413-de00490eea83" />


- Applying SYN filter to identify Nmap reconnaissance scan (`tcp.flags.syn==1 && tcp.flags.ack==0`)
<!-- Screenshot -->
<img width="1920" height="1080" alt="applying syn filter because nmapo scans send 1000 of em to identify target" src="https://github.com/user-attachments/assets/ad80f594-ed0f-4a92-bdf3-8d24e9dd663e" />


- Confirming Samba exploit traffic on port 445
<!-- Screenshot -->
<img width="1920" height="1080" alt="samba attack confirmation metasploitable on tcp port 445" src="https://github.com/user-attachments/assets/8fe0e520-68ad-446c-b21a-26bbe8f1a286" />

<img width="1920" height="1080" alt="samba exploitation 2" src="https://github.com/user-attachments/assets/bfced547-dab7-4961-8f94-2d8577a5756e" />

- Recovering attacker commands from TCP stream (port 6200)
<!-- Screenshot -->
<img width="1920" height="1080" alt="commands of the attacker" src="https://github.com/user-attachments/assets/facc1a48-cae8-4ac5-a1f6-fc9aca2dd40f" />


- Confirming data exfiltration on port 5555 — employee data visible in plaintext
<!-- Screenshot -->
<img width="1920" height="1080" alt="data exfiltration on port 5555 from meta to kali attakcer 1" src="https://github.com/user-attachments/assets/5cc2d9e3-a7c1-447d-828a-898e7a624539" />

- filtering all samba related traffic
<img width="1920" height="1080" alt="all samba related traffic filtered" src="https://github.com/user-attachments/assets/1433d4ec-2223-4b34-afac-9e80521dcb33" />

- filtering all traffic related to attacker and the victim
<img width="1920" height="1080" alt="all traffic filtered from attacker to metasploit" src="https://github.com/user-attachments/assets/becdbde9-9f6f-45e3-979d-081fdecd35e5" />

- Saving all filtered traffic as evidence
<img width="1920" height="1080" alt="saving all transferred files as evience" src="https://github.com/user-attachments/assets/5e339187-114a-4d68-b7fa-34286fef3dbb" />

<img width="1920" height="1080" alt="saving tranfered files for evidence 2" src="https://github.com/user-attachments/assets/4d6c7ddc-27d4-469c-9da1-58cde2f0fbec" />

<img width="1920" height="1080" alt="saving tranfered files for evidence  3" src="https://github.com/user-attachments/assets/9f2abc4b-9324-4f17-9ca8-b4697fcb1194" />

- Saving it as a CSV file (because it is easier to work with)
<img width="1920" height="1080" alt="saving as a csv so easy to work with it" src="https://github.com/user-attachments/assets/9cdc8096-54df-4be8-b60b-69df9e37894d" />

<img width="1920" height="1080" alt="evidence csv saved" src="https://github.com/user-attachments/assets/7b40dd6e-6108-4e68-aabc-d553401fec6f" />

---

### Step 9 — Memory Forensics (Volatility3)

A memory dump was acquired from the victim machine during the attack window using the LiME kernel module and analysed with Volatility3.

- Running Volatility3 linux.pslist to list processes
<!-- Screenshot -->
![Volatility PSList](screenshots/vol_pslist.png)

- Running linux.netstat to check network connections
<!-- Screenshot -->
![Volatility Netstat](screenshots/vol_netstat.png)

> ⚠️ **Finding:** By the time memory analysis was conducted, the attack processes had already terminated. No active malicious processes or connections were recovered from the memory dump. This highlights the critical importance of performing memory forensics **live during the incident** to capture volatile evidence before it disappears.

---

### Step 10 — Disk Forensics (The Sleuth Kit)

The victim disk image was forensically examined using The Sleuth Kit CLI tools. The image was mounted **read-only** throughout the entire analysis to preserve forensic integrity and maintain chain of custody.

- Viewing partition table with mmls
<!-- Screenshot -->
![mmls Partitions](screenshots/tsk_mmls.png)

```
002: 000:000   Start: 63        Linux (0x83)   ← Boot partition
006: 001:000   Start: 482013    Linux LVM      ← Root filesystem
```

- File system information using fsstat
<!-- Screenshot -->
![fsstat Output](screenshots/tsk_fsstat.png)

- Listing all files including deleted ones with fls
<!-- Screenshot -->
![fls All Files](screenshots/tsk_fls_all.png)

- Viewing deleted files only (entries marked with *)
<!-- Screenshot -->
![fls Deleted Files](screenshots/tsk_fls_deleted.png)

- Activating LVM and mounting root filesystem read-only
<!-- Screenshot -->
![LVM Mount](screenshots/tsk_lvm_mount.png)

- Browsing mounted victim filesystem — folder structure
<!-- Screenshot -->
![Victim Folders](screenshots/tsk_victim_folders.png)

- Viewing /tmp directory contents
<!-- Screenshot -->
![tmp Files](screenshots/tsk_tmp_files.png)

- **CRITICAL FINDING** — Backdoor user confirmed in /home/
<!-- Screenshot -->
![aptbackdoor Home](screenshots/tsk_aptbackdoor_home.png)

- **CRITICAL FINDING** — aptbackdoor confirmed in /etc/passwd
<!-- Screenshot -->
![aptbackdoor Passwd](screenshots/tsk_aptbackdoor_passwd.png)

- Viewing auth.log — login events and SSH connections
<!-- Screenshot -->
![Auth Log](screenshots/tsk_auth_log.png)

- Viewing system messages log
<!-- Screenshot -->
![Messages Log](screenshots/tsk_messages_log.png)

- Listing all users who logged in
<!-- Screenshot -->
![User List](screenshots/tsk_user_list.png)

- All attacker-related connections filtered from logs
<!-- Screenshot -->
![Attacker Connections](screenshots/tsk_attacker_connections.png)

---

### Step 11 — File Carving (Foremost)

Foremost was used to carve and recover files from the victim disk image based on file headers and footers.

- Running Foremost on victim disk image
<!-- Screenshot -->
![Foremost Carving](screenshots/foremost_run.png)

- Viewing carved files recovered from disk
<!-- Screenshot -->
![Carved Files](screenshots/foremost_carved.png)

---

### Step 12 — Timeline Analysis (Plaso / Log2Timeline)

#### 12a — Network Timeline (PCAP)

Plaso was used to generate a super timeline from the captured PCAP file, providing timestamped forensic events.

- Running log2timeline on PCAP file
<!-- Screenshot -->
![Log2Timeline PCAP](screenshots/plaso_pcap_run.png)

- Exporting to CSV with psort
<!-- Screenshot -->
![psort PCAP Export](screenshots/plaso_pcap_psort.png)

- Viewing timeline in Kate — attack date, timezone, and timestamps confirmed
<!-- Screenshot -->
![PCAP Timeline Kate](screenshots/plaso_pcap_kate.png)

#### 12b — Disk Super Timeline (62,631 Events)

A comprehensive super timeline was generated from the entire victim disk image, producing 62,631 filesystem events.

- Running log2timeline on victim disk image (LVM volume selected)
<!-- Screenshot -->
![Log2Timeline Disk](screenshots/plaso_disk_run.png)

- Processing completed — 62,631 events extracted
<!-- Screenshot -->
![Plaso Disk Complete](screenshots/plaso_disk_complete.png)

- Exporting disk timeline to CSV
<!-- Screenshot -->
![Disk Timeline CSV](screenshots/plaso_disk_csv.png)

- Viewing disk timeline — searching for attack artifacts
<!-- Screenshot -->
![Disk Timeline View](screenshots/plaso_disk_view.png)

---

---

## 📁 Evidence Inventory

```
forensic/
├── evidence/
│   ├── attack_capture.pcap       # Full network capture (tcpdump)
│   ├── filtered_evidence.pcap    # Filtered attack traffic (Wireshark)
│   ├── baseline_scan.txt         # Nmap scan results from Kali #2
│   ├── memory.dump               # Victim memory image (LiME)
│   ├── victim_disk.img           # Full disk image (dd)
│   ├── super_timeline.csv        # Plaso PCAP timeline
│   ├── disk_timeline.csv         # Plaso disk timeline (62,631 events)
│   ├── auth.log                  # Victim authentication logs
│   ├── passwd.txt                # Victim /etc/passwd (backdoor confirmed)
│   ├── networkminer_findings.txt # tcpflow/p0f network analysis findings
│   ├── FINDINGS_SUMMARY.txt      # Master forensic findings document
│   └── reports/
│       └── IR_Report_ICBT.docx   # Full Incident Response Report
└── carved_files/                 # Files recovered by Foremost
```

---

## ⚠️ Disclaimer

This project is built entirely for **educational and ethical purposes**. All attacks were performed in a fully isolated virtual lab environment with no connection to any real-world systems, networks, or infrastructure. The techniques demonstrated are intended to develop defensive cybersecurity skills and forensic investigation capabilities.

---

## 👤 Author

**Nihara Sulochana Samaranayake**
HND in Networking and Cybersecurity
ICBT Kandy Campus — Cardiff Metropolitan University
GitHub: [zula69](https://github.com/zula69)
