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
| 5 | **Autopsy 4.x Java conflict** | Autopsy 4.21 crashed due to system running Java 25 (Security Manager removed in Java 17+) | Fixed by editing etc/autopsy.conf to set jdkhome to the Java 17 path, bypassing system Java |
| 6 | **Disk image using wrong partition offset** | TSK commands returned no results when using default offset | Used mmls to identify correct start sector (63 for boot, LVM at 482013) and applied -o flag |
| 7 | **LVM partition not accessible via TSK** | Root filesystem was inside LVM — fls returned no attack artifacts | Activated LVM with vgscan/vgchange and mounted root volume read-only for direct filesystem access |
| 8 | **NetworkMiner rendering broken on Parrot KDE** | Mono + KDE caused black/blank GUI on all launch attempts | Replaced with tcpflow for stream reconstruction and p0f for OS fingerprinting — native Linux tools |
| 9 | **Plaso CSV empty on first run** | psort used > redirection which corrupted the output | Used the -w flag instead of shell redirection to write the CSV file correctly |
| 10 | **Snapd broken on Parrot OS** | AppArmor conflict prevented snapd from starting for Autopsy 4 install | Used manual Autopsy 4 installation with explicit jdkhome configuration instead of snap |




## 🖧 Part 1 — Campus Network Design (Cisco Packet Tracer)

A reference campus network was designed and configured in Cisco Packet Tracer using a three-tier hierarchical model with five VLANs, a DMZ, an edge router, and an ISP router.

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
![Labelled Devices](screenshots/pt_labelled_devices.png)

- Connected all devices in the topology
<!-- Screenshot: connected devices -->
![Connected Devices](screenshots/pt_connected_devices.png)

---

### 🎓 Student Lab Switch (VLAN 10)

Configured the access layer switch for the Student Lab, assigned VLAN 10, and set up trunk ports to connect to the distribution layer.

- Changing hostname of Student Lab switch
<!-- Screenshot -->
![Student Switch Hostname](screenshots/pt_student_hostname.png)

- Configuring VLAN 10 (Students — 10.10.10.0/24)
<!-- Screenshot -->
![VLAN 10 Config](screenshots/pt_vlan10_config.png)

- Creating trunk port on Student Lab switch
<!-- Screenshot -->
![Student Trunk Port](screenshots/pt_student_trunk.png)

- Confirming VLAN 10 is active and working
<!-- Screenshot -->
![VLAN 10 Confirmation](screenshots/pt_vlan10_confirm.png)

---

### 👨‍💼 Staff Switch (VLAN 20)

- Changing hostname of Staff switch
<!-- Screenshot -->
![Staff Switch Hostname](screenshots/pt_staff_hostname.png)

- Creating VLAN 20 (Staff — 10.20.20.0/24)
<!-- Screenshot -->
![VLAN 20 Config](screenshots/pt_vlan20_config.png)

- Creating trunk port on Staff switch
<!-- Screenshot -->
![Staff Trunk Port](screenshots/pt_staff_trunk.png)

- Confirming VLAN 20 is active
<!-- Screenshot -->
![VLAN 20 Confirmation](screenshots/pt_vlan20_confirm.png)

---

### 🔐 Admin Switch (VLAN 30)

- Changing hostname of Admin switch
<!-- Screenshot -->
![Admin Switch Hostname](screenshots/pt_admin_hostname.png)

- Creating VLAN 30 (Admin — 10.30.30.0/24)
<!-- Screenshot -->
![VLAN 30 Config](screenshots/pt_vlan30_config.png)

- Creating trunk port on Admin switch
<!-- Screenshot -->
![Admin Trunk Port](screenshots/pt_admin_trunk.png)

- Confirming VLAN 30 is active
<!-- Screenshot -->
![VLAN 30 Confirmation](screenshots/pt_vlan30_confirm.png)

---

### 📚 Library Switch (VLAN 40)

- Changing hostname of Library switch
<!-- Screenshot -->
![Library Switch Hostname](screenshots/pt_library_hostname.png)

- Creating VLAN 40 (Library — 10.40.40.0/24)
<!-- Screenshot -->
![VLAN 40 Config](screenshots/pt_vlan40_config.png)

- Trunk port setup on Library switch
<!-- Screenshot -->
![Library Trunk Port](screenshots/pt_library_trunk.png)

- Confirming VLAN 40 is active
<!-- Screenshot -->
![VLAN 40 Confirmation](screenshots/pt_vlan40_confirm.png)

---

### 📶 Guest WiFi Switch (VLAN 50)

- Changing hostname of Guest WiFi switch
<!-- Screenshot -->
![Guest WiFi Hostname](screenshots/pt_guest_hostname.png)

- Creating VLAN 50 (Guest WiFi — 10.50.50.0/24)
<!-- Screenshot -->
![VLAN 50 Config](screenshots/pt_vlan50_config.png)

- Trunk port setup on Guest WiFi switch
<!-- Screenshot -->
![Guest Trunk Port](screenshots/pt_guest_trunk.png)

- Confirming VLAN 50 is active
<!-- Screenshot -->
![VLAN 50 Confirmation](screenshots/pt_vlan50_confirm.png)

---

### 🔀 Distribution Switch 1

The first distribution layer switch aggregates traffic from Student Lab, Staff, and Admin access switches and trunks upward to the core.

- Changing hostname of Distribution Switch 1
<!-- Screenshot -->
![Dist SW1 Hostname](screenshots/pt_dist1_hostname.png)

- Creating VLANs on Distribution Switch 1
<!-- Screenshot -->
![Dist SW1 VLANs](screenshots/pt_dist1_vlans.png)

- Confirming VLANs are active
<!-- Screenshot -->
![Dist SW1 VLAN Confirm](screenshots/pt_dist1_vlan_confirm.png)

- Configuring uplink trunk port (to Core Switch)
<!-- Screenshot -->
![Dist SW1 Uplink Trunk](screenshots/pt_dist1_uplink.png)

- Configuring downlink trunk ports (to access switches)
<!-- Screenshot -->
![Dist SW1 Downlink Trunks](screenshots/pt_dist1_downlinks.png)

---

### 🔀 Distribution Switch 2

The second distribution switch aggregates Library and Guest WiFi traffic.

- Changing hostname of Distribution Switch 2
<!-- Screenshot -->
![Dist SW2 Hostname](screenshots/pt_dist2_hostname.png)

- Creating VLANs on Distribution Switch 2
<!-- Screenshot -->
![Dist SW2 VLANs](screenshots/pt_dist2_vlans.png)

- Confirming VLANs active
<!-- Screenshot -->
![Dist SW2 VLAN Confirm](screenshots/pt_dist2_vlan_confirm.png)

- Configuring uplink trunk port (to Core Switch)
<!-- Screenshot -->
![Dist SW2 Uplink](screenshots/pt_dist2_uplink.png)

- Configuring downlink trunk ports
<!-- Screenshot -->
![Dist SW2 Downlinks](screenshots/pt_dist2_downlinks.png)

---

### 🖥️ Core Switch

The core switch provides Layer 3 inter-VLAN routing, assigns SVI IP addresses for each VLAN, and trunks to both distribution switches.

- Changing hostname of Core Switch
<!-- Screenshot -->
![Core Switch Hostname](screenshots/pt_core_hostname.png)

- Adding all VLANs to Core Switch
<!-- Screenshot -->
![Core VLANs](screenshots/pt_core_vlans.png)

- Setting up SVI interfaces and IP addresses for each VLAN
<!-- Screenshot -->
![Core SVIs](screenshots/pt_core_svi.png)

- Trunk to Distribution Switch 1
<!-- Screenshot -->
![Core Trunk Dist1](screenshots/pt_core_trunk_dist1.png)

- Trunk to Distribution Switch 2
<!-- Screenshot -->
![Core Trunk Dist2](screenshots/pt_core_trunk_dist2.png)

---

### 🔥 Edge Router (Firewall Replacement)

> ⚠️ The original Cisco ASA 5505 firewall was replaced with a 2911 router using ACLs and NAT due to Packet Tracer limitations with ASA configuration.

- Changing hostname of Edge Router
<!-- Screenshot -->
![Edge Router Hostname](screenshots/pt_edge_hostname.png)

- Assigning IP addresses on Edge Router interfaces
<!-- Screenshot -->
![Edge Router IPs](screenshots/pt_edge_ips.png)

- Permitting all VLANs to access ISP router (NAT/routing)
<!-- Screenshot -->
![Edge Router NAT](screenshots/pt_edge_nat.png)

- Setting default route to ISP
<!-- Screenshot -->
![Edge Default Route](screenshots/pt_edge_default_route.png)

- Setting return route back to campus network
<!-- Screenshot -->
![Edge Return Route](screenshots/pt_edge_return_route.png)

---

### 🌐 ISP Router

- Configuring hostname of ISP Router
<!-- Screenshot -->
![ISP Router Hostname](screenshots/pt_isp_hostname.png)

- Adding IP addresses on ISP Router
<!-- Screenshot -->
![ISP Router IPs](screenshots/pt_isp_ips.png)

- Configuring routing to forward campus traffic
<!-- Screenshot -->
![ISP Routing](screenshots/pt_isp_routing.png)

---

### 🖥️ Server Configuration (DMZ — 172.16.1.0/24)

- Configuring Web Server
<!-- Screenshot -->
![Web Server Config](screenshots/pt_web_server.png)

- Configuring DNS Server
<!-- Screenshot -->
![DNS Server Config](screenshots/pt_dns_server.png)

- Configuring Mail Server
<!-- Screenshot -->
![Mail Server Config](screenshots/pt_mail_server.png)

---

### 🗺️ Final Network Topology

<!-- Screenshot: full Packet Tracer topology -->
![Final Topology](screenshots/pt_final_topology.png)

---

### ✅ Connectivity Verification — Ping Tests

All VLANs were verified to communicate correctly through the three-tier hierarchy.

- Pinging from Student PC to all other VLANs (all successful)
<!-- Screenshot -->
![Ping Tests](screenshots/pt_ping_tests.png)

---

### 🔒 Access Control Lists (ACLs)

Four ACLs were implemented on the Core Switch to enforce the campus security policy.

#### ACL 1 — Guest WiFi Policy
> Guests can access the internet but are completely blocked from all internal departments

```
Guests (VLAN 50) → ❌ VLAN 10 (Students)
Guests (VLAN 50) → ❌ VLAN 20 (Staff)
Guests (VLAN 50) → ❌ VLAN 30 (Admin)
Guests (VLAN 50) → ❌ VLAN 40 (Library)
Guests (VLAN 50) → ✅ Internet only
```

#### ACL 2 — Student Policy
> Students can access Library and DMZ but cannot reach Admin or Staff systems

```
Students (VLAN 10) → ✅ VLAN 40 (Library)
Students (VLAN 10) → ✅ DMZ (Web/DNS)
Students (VLAN 10) → ❌ VLAN 30 (Admin)
Students (VLAN 10) → ❌ VLAN 20 (Staff)
```

#### ACL 3 — DMZ Server Protection
> Only specific traffic allowed to reach DMZ. Only IT Staff can SSH into servers.

```
Internet → ✅ Port 80/443 (HTTP/HTTPS)
Internet → ✅ Port 53 (DNS)
Internet → ❌ All other ports
VLAN 20 (Staff) → ✅ Port 22 (SSH management)
All others → ❌ Port 22
```

#### ACL 4 — Admin Office Protection
> Only Admin and Staff VLANs can access the Admin network

```
VLAN 30 (Admin) → ✅ VLAN 30
VLAN 20 (Staff) → ✅ VLAN 30
VLAN 10 (Students) → ❌ VLAN 30
VLAN 50 (Guests) → ❌ VLAN 30
```

- ACL verification using `show access-lists` on Core Switch
<!-- Screenshot -->
![ACL Verification](screenshots/pt_acl_verify.png)

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

### Step 1 — Network Verification

Verified all VMs can communicate over the host-only network before beginning the attack simulation.

- Pinging from Kali #1 to Metasploitable2
<!-- Screenshot -->
![Ping Attacker to Victim](screenshots/vm_ping_attacker_victim.png)

- Pinging from Kali #2 to all VMs
<!-- Screenshot -->
![Ping Watcher](screenshots/vm_ping_watcher.png)

- Pinging from Parrot OS to all VMs
<!-- Screenshot -->
![Ping Parrot](screenshots/vm_ping_parrot.png)

---

### Step 2 — Evidence Collection Setup (Kali #2 Watcher)

Kali Linux #2 was configured as a **passive network monitor**. It silently captured all traffic on the network segment using tcpdump without sending any packets that could alert the attacker.

- Starting tcpdump to capture all network traffic
<!-- Screenshot -->
![tcpdump Start](screenshots/kali2_tcpdump_start.png)

- Collecting Nmap scan results from Kali #2 to Metasploitable2
<!-- Screenshot -->
![Nmap Scan Evidence](screenshots/kali2_nmap_evidence.png)

- Collecting journalctl system logs as supporting evidence
<!-- Screenshot -->
![journalctl Evidence](screenshots/kali2_journalctl.png)

---

### Step 3 — Reconnaissance (Kali #1 Attacker)

The attacker performed network reconnaissance using Nmap to identify open ports and running services on the victim machine.

- Nmap SYN scan to identify open ports on Metasploitable2
<!-- Screenshot -->
![Nmap Recon](screenshots/kali1_nmap_recon.png)

**Key ports identified:**

| Port | Service | Version |
|------|---------|---------|
| 21 | FTP | vsftpd 2.3.4 |
| 22 | SSH | OpenSSH 4.7p1 |
| 139/445 | SMB/Samba | Samba 3.x |
| 80 | HTTP | Apache 2.2.8 |
| 3306 | MySQL | 5.0.51a |

---

### Step 4 — Exploitation (Kali #1 Attacker)

#### Attempt 1 — vsftpd 2.3.4 (Failed)

The attacker first attempted to exploit the vsftpd 2.3.4 backdoor vulnerability (CVE-2011-2523) using Metasploit. The exploit did not succeed on this target.

- Attempting vsftpd exploit — failed
<!-- Screenshot -->
![vsftpd Fail](screenshots/kali1_vsftpd_fail.png)

#### Attempt 2 — Samba usermap_script (Succeeded ✅)

The attacker then targeted the Samba service using the usermap_script vulnerability (CVE-2007-2447), which allows unauthenticated remote command execution as root.

- Configuring Metasploit with exploit/multi/samba/usermap_script
<!-- Screenshot -->
![Samba Exploit Config](screenshots/kali1_samba_config.png)

- Running the exploit — root shell obtained
<!-- Screenshot -->
![Root Shell Obtained](screenshots/kali1_root_shell.png)

---

### Step 5 — Persistence (Kali #1 Attacker)

With root access established, the attacker created a backdoor user account to maintain persistent access even if the initial shell was closed.

- Creating backdoor user account (aptbackdoor)
<!-- Screenshot -->
![Backdoor User Created](screenshots/kali1_backdoor_user.png)

- Verifying backdoor account in /etc/passwd
<!-- Screenshot -->
![Backdoor in Passwd](screenshots/kali1_backdoor_passwd.png)

```
aptbackdoor:x:1003:1003::/home/aptbackdoor:/bin/bash
```

---

### Step 6 — Data Exfiltration (Kali #1 Attacker)

The attacker located and exfiltrated confidential employee records from the victim machine using Netcat on port 5555.

- Locating confidential employee data on victim
<!-- Screenshot -->
![Finding Data](screenshots/kali1_find_data.png)

- Exfiltrating data via Netcat on port 5555
<!-- Screenshot -->
![Data Exfiltration](screenshots/kali1_exfiltration.png)

---

### Step 7 — Evidence Transfer to Parrot OS

All evidence collected by Kali #2 was transferred securely to the Parrot OS forensic workstation via SCP for analysis.

- Transferring PCAP, memory dump, disk image to Parrot OS
<!-- Screenshot -->
![Evidence Transfer](screenshots/kali2_scp_transfer.png)

---

## 🔬 Part 3 — Digital Forensic Analysis (Parrot OS)

### Step 8 — Network Forensics (Wireshark)

The captured PCAP file was loaded into Wireshark on the Parrot OS workstation for deep packet inspection.

- Packet statistics overview
<!-- Screenshot -->
![Packet Statistics](screenshots/ws_packet_stats.png)

- Protocol hierarchy statistics
<!-- Screenshot -->
![Protocol Stats](screenshots/ws_protocol_stats.png)

- IPv4 conversations — identifying attacker-victim traffic
<!-- Screenshot -->
![IPv4 Conversations](screenshots/ws_ipv4_conversations.png)

- Traffic statistics by endpoint
<!-- Screenshot -->
![Traffic Stats](screenshots/ws_traffic_stats.png)

- Applying SYN filter to identify Nmap reconnaissance scan (`tcp.flags.syn==1 && tcp.flags.ack==0`)
<!-- Screenshot -->
![SYN Filter Nmap](screenshots/ws_syn_filter.png)

- Confirming Samba exploit traffic on port 445
<!-- Screenshot -->
![Samba Attack Confirmed](screenshots/ws_samba_attack.png)

- Recovering attacker commands from TCP stream (port 6200)
<!-- Screenshot -->
![Attacker Commands](screenshots/ws_attacker_commands.png)

- Confirming data exfiltration on port 5555 — employee data visible in plaintext
<!-- Screenshot -->
![Exfiltration Evidence](screenshots/ws_exfiltration.png)

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
