🛡️ SOC Analyst Home Lab

> **Hands-on cybersecurity lab simulating real-world Security Operations Center scenarios.**
> Built with VMware Workstation Pro, Splunk Enterprise, Kali Linux, and Windows 10.

![Lab Status](https://img.shields.io/badge/Labs-3%20of%207%20Complete-blue)
![MITRE](https://img.shields.io/badge/MITRE%20ATT%26CK-Mapped-red)
![Splunk](https://img.shields.io/badge/SIEM-Splunk%20Enterprise-green)

---

## 📋 Table of Contents

- [Lab Overview](#lab-overview)
- [Environment](#environment)
- [Setup Guide](#setup-guide)
  - [Phase 1: VMware Workstation Pro](#phase-1-vmware-workstation-pro)
  - [Phase 2: Kali Linux VM](#phase-2-kali-linux-vm)
  - [Phase 3: Windows 10 VM](#phase-3-windows-10-vm)
  - [Phase 4: Splunk Enterprise](#phase-4-splunk-enterprise)
  - [Phase 5: Splunk Universal Forwarder](#phase-5-splunk-universal-forwarder)
  - [Phase 6: Network Verification](#phase-6-network-verification)
- [Labs](#labs)
  - [Lab 1: Brute Force Detection](#lab-1--brute-force-detection)
  - [Lab 2: Network Reconnaissance & Port Scan Detection](#lab-2--network-reconnaissance--port-scan-detection)
  - [Lab 3: Beaconing / C2 Detection](#lab-3--beaconing--c2-detection)
- [MITRE ATT&CK Mapping](#mitre-attck-mapping)
- [Key Event IDs](#key-event-ids)
- [Author](#author)

---

## 🎯 Lab Overview

This home lab provides hands-on experience in cybersecurity by simulating real-world systems and scenarios. Throughout the setup, you will work with virtual machines, networking, and security tools to understand how threats are detected and managed.

By the end, you will have practical skills in:
- **System configuration** and virtualization
- **Log analysis** and SIEM query writing (SPL)
- **Incident response** lifecycle (Detect → Triage → Contain → Investigate → Escalate → Remediate)
- **Threat detection** using Windows Event Logs and Sysmon
- **MITRE ATT&CK framework** mapping

### Lab Roadmap

| # | Scenario | Key Skills | MITRE |
|---|----------|----------|-------|
| 1 | [Brute Force Detection](#lab-1--brute-force-detection) | Failed logon analysis, threshold alerting | T1110.001 |
| 2 | [Network Reconnaissance](#lab-2--network-reconnaissance--port-scan-detection) | Port scan detection, SYN analysis | T1046, T1595 |
| 3 | [C2 Beaconing](#lab-3--beaconing--c2-detection) | Timing analysis, connection patterns | T1071 |
| 4 | Privilege Escalation Detection | Process analysis, token manipulation | T1134 |
| 5 | Malware Detection | Behavioral analysis, hash verification | T1055 |
| 6 | Rule Tuning | False positive reduction, threshold optimization | — |
| 7 | Phishing Analysis | Email forensics, URL analysis | T1566 |

---

## 🖥️ Environment

| Component | Details |
|-----------|---------|
| **Host Machine** | Physical laptop/desktop (Windows 10/11) |
| **Hypervisor** | VMware Workstation Pro 25.x |
| **Attacker VM** | Kali Linux (VM 1) |
| **Target VM** | Windows 10 (VM 2) |
| **SIEM** | Splunk Enterprise (on host) |
| **Forwarder** | Splunk Universal Forwarder (on Win10) |
| **EDR** | Microsoft Sysmon (on Win10) |

### Network Architecture

```
+-----------------------------------------+
|           HOST MACHINE                  |
|  +---------------------------------+    |
|  |     Splunk Enterprise           |    |
|  |     http://localhost:8000       |    |
|  +---------------------------------+    |
|              |                          |
|         VMnet1 (Host-only)              |
|         192.168.x.0/24                  |
|         |              |                |
|    +----+----+    +----+----+           |
|    | Kali    |    | Win10  |           |
|    | Linux   |<--> | Target |           |
|    | Attacker|    | Victim |           |
|    +---------+    +--------+           |
+-----------------------------------------+
```

> ⚠️ **All attack traffic stays inside the isolated Host-only network. Never expose these VMs to the internet.**

---

## 🔧 Setup Guide

### Prerequisites

| Requirement | Specification | Notes |
|-------------|---------------|-------|
| Host RAM | 16–32 GB | 12–16 GB allocated to VMs |
| Storage | 150+ GB free | Kali: 80 GB, Win10: 60 GB |
| CPU | 4+ cores with virtualization | Intel VT-x / AMD-V in BIOS |
| OS | Windows 10/11 host | Administrator access required |
| Network | Internet for downloads | VMs use isolated network |

---

### Phase 1: VMware Workstation Pro

#### Step 1 — Download VMware

VMware is now owned by Broadcom. All downloads require a free Broadcom account.

1. Go to: [https://support.broadcom.com](https://support.broadcom.com)
2. Register for a free account (verify email)
3. Navigate to: **My Downloads → VMware Workstation Pro**
4. Select version **25.x** (latest stable) for Windows
5. Download the `.exe` installer

> ⚠️ **Do NOT download VMware Workstation Pro 17 or earlier** — they are end-of-life and no longer receive security updates.

#### Step 2 — Install VMware

Run the installer as Administrator. Use default settings with these exceptions:
- Uncheck **"Join the VMware Customer Experience..."** for privacy
- On first launch, select **"I want to use VMware Workstation 25 for Personal Use"** (free tier — fully featured)

#### Step 3 — Configure Virtual Networks

Open **Edit → Virtual Network Editor** (click "Change Settings" and approve UAC).

Confirm these networks exist:

| Name | Type | Subnet | Purpose |
|------|------|--------|---------|
| VMnet1 | Host-only | 192.168.x.0 | Isolated attack lab network |
| VMnet8 | NAT | 192.168.y.0 | Internet access for updates |

> 💡 **Note your VMnet1 subnet IP.** Run `ipconfig` on your host and look for:
> ```
> Ethernet adapter VMware Network Adapter VMnet1:
>    IPv4 Address......: 192.168.x.1
> ```

#### Step 4 — Set Memory Preferences

**Edit → Preferences → Memory**
- Reserved memory: **12–16 GB** total across all running VMs

**Edit → Preferences → Workspace**
- Default VM location: `C:\Users\YourName\Documents\Virtual Machines\`

#### Step 5 — Verify VMware Services

```powershell
Get-Service -Name "VMware*" | Select-Object Name, Status
```

All key services should show **Running**.

---

### Phase 2: Kali Linux VM

#### Step 6 — Download Kali Linux ISO

1. Go to: [https://www.kali.org/get-kali/#kali-installer-images](https://www.kali.org/get-kali/#kali-installer-images)
2. Download the **64-bit Installer Image**
3. Verify SHA256 checksum:

```powershell
Get-FileHash "kalilinux-2025.2-installer-amd64.iso" -Algorithm SHA256
```

#### Step 7 — Create Kali VM

**File → New Virtual Machine → Typical**

| Setting | Value |
|---------|-------|
| Installer disc image | Browse to Kali ISO |
| Guest OS | Linux → Debian 12.x 64-bit |
| VM Name | `kali-attacker` |
| Location | `C:\Users\YourName\Documents\Virtual Machines\Kali Linux\` |
| Disk Size | 80 GB (Store as single file) |
| Memory | 4096 MB |
| Processors | 2 cores, 2 per processor |
| Network | NAT (for installation/updates) |

#### Step 8 — Install Kali Linux

Boot the VM and select **Graphical Install**:

| Setting | Value |
|---------|-------|
| Language | English |
| Location | Your region |
| Keyboard | American English |
| Hostname | `kali-attacker` |
| Domain | (leave blank) |
| Username | Your name |
| Password | Strong password (min 12 chars) |
| Partitioning | Guided — use entire disk → All files in one partition |
| Software | ✅ `top10tools` ✅ `SSH server` |

#### Step 9 — Update System & Install VMware Tools

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install open-vm-tools open-vm-tools-desktop
sudo reboot
```

#### Step 10 — Verify Key Tools

```bash
nmap --version
hydra -h | head -5
msfconsole --version
wireshark --version
```

Install any missing tools:
```bash
sudo apt install -y nmap wireshark metasploit-framework hydra netcat-traditional
```

> ✅ **Take a snapshot:** `Kali-Clean Install` — Fresh install, tools verified, no attacks run yet.

---

### Phase 3: Windows 10 VM

#### Step 11 — Create Windows 10 ISO

1. Go to: [https://www.microsoft.com/software-download/windows10](https://www.microsoft.com/software-download/windows10)
2. Download the **Media Creation Tool**
3. Run as Administrator → **"Create installation media" → ISO file**
4. Save to: `C:\Users\YourName\Documents\Virtual Machines\Win10-Target\`

#### Step 12 — Create Windows 10 VM

**File → New Virtual Machine → Typical**

| Setting | Value |
|---------|-------|
| Installer disc image | Browse to Windows ISO |
| Guest OS | Microsoft Windows → Windows 10 x64 |
| VM Name | `Win10-Target` |
| Location | `C:\Users\YourName\Documents\Virtual Machines\Win10-Target\` |
| Disk Size | 60 GB (Store as single file) |
| Memory | 4096 MB |
| Processors | 2 cores, 2 per processor |
| Network | **Host-only** |

> ⚠️ **Select Windows 10 Pro** during installation — Pro has RDP server built-in.

#### Step 13 — Windows OOBE Setup

| Setting | Value |
|---------|-------|
| Region | United States |
| Keyboard | US |
| Sign in | Click **"Domain join instead"** (creates local account) |
| Full name | `labuser` |
| Password | (set a password for RDP) |
| Privacy | Turn **ALL toggles OFF** |

#### Step 14 — Install VMware Tools

In VMware menu: **VM → Install VMware Tools**

Inside Win10 VM: File Explorer → DVD Drive (D:) → Setup → Next → Next → Install → Restart

#### Step 15 — Configure Windows 10

**Rename computer:**
```powershell
Rename-Computer -NewName "Win10-Target" -Restart
```

**Enable RDP:**
```powershell
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
```

**Create weak victim account (for brute-force lab):**
```powershell
New-LocalUser -Name "victim" -Password (ConvertTo-SecureString "Password123" -AsPlainText -Force) -FullName "Lab Victim" -Description "Weak account for brute-force lab"
Add-LocalGroupMember -Group "Remote Desktop Users" -Member "victim"
Add-LocalGroupMember -Group "Users" -Member "victim"
```

> 🚨 **Security Warning:** This account is intentionally insecure. Only use inside an isolated Host-only network.

#### Step 16 — Install Sysmon

```powershell
New-Item -ItemType Directory -Path "C:\Tools" -Force
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "C:\Tools\Sysmon.zip"
Expand-Archive -Path "C:\Tools\Sysmon.zip" -DestinationPath "C:\Tools\Sysmon\"
cd C:\Tools\Sysmon\
.\Sysmon64.exe -accepteula -i
```

Verify:
```powershell
Get-Service Sysmon64
```

#### Step 17 — Enable Audit Logging

```powershell
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Credential Validation" /success:enable /failure:enable
```

Verify:
```powershell
auditpol /get /category:*
```

> ✅ **Take a snapshot:** `Win10-Clean Install` — Fresh install, RDP enabled, victim account created, Sysmon running.

---

### Phase 4: Splunk Enterprise

#### Step 18 — Download & Install Splunk

1. Go to: [https://www.splunk.com/en_us/download/splunk-enterprise.html](https://www.splunk.com/en_us/download/splunk-enterprise.html)
2. Create a free Splunk account
3. Download the Windows `.msi` installer
4. Run as Administrator

| Setting | Value |
|---------|-------|
| License | Splunk Free (60-day trial / 500 MB/day) |
| Username | `admin` |
| Password | (set strong password) |
| Service Account | Local System |

#### Step 19 — Open Splunk Web

```
http://localhost:8000
```

Log in with admin credentials. Accept the license agreement.

#### Step 20 — Enable Receiving Port

**Settings → Forwarding and receiving → Configure receiving → New Receiving Port**

- Port number: `9997`
- Click **Save**

Verify port is listening:
```powershell
netstat -ano | findstr :9997
```

---

### Phase 5: Splunk Universal Forwarder

#### Step 21 — Download Forwarder

On your host machine:
1. Go to: [https://www.splunk.com/en_us/download/universal-forwarder.html](https://www.splunk.com/en_us/download/universal-forwarder.html)
2. Download Windows 64-bit installer
3. Copy to Win10-Target (drag-and-drop or shared folder)

#### Step 22 — Install Forwarder on Win10

Run as Administrator inside Win10-Target:

| Setting | Value |
|---------|-------|
| License | On-premises Splunk Enterprise |
| Username | `admin` |
| Password | (set forwarder admin password) |
| Deployment Server | (leave blank) |
| Receiving Indexer | Host's VMnet1 IP (e.g., `192.168.x.1`) |
| Port | `9997` |

#### Step 23 — Configure Log Inputs

```powershell
cd "C:\Program Files\SplunkUniversalForwarder\bin"

.\splunk.exe add monitor "WinEventLog://Security" --accept-license
.\splunk.exe add monitor "WinEventLog://System" --accept-license
.\splunk.exe add monitor "WinEventLog://Application" --accept-license
.\splunk.exe add monitor "WinEventLog://Microsoft-Windows-Sysmon/Operational" --accept-license

.\splunk.exe restart
```

#### Step 24 — Verify Forwarder Connection

```powershell
.\splunk.exe list forward-server
```

Expected output:
```
Active forwards: 192.168.x.x:9997
```

#### Step 25 — Test in Splunk Web

Run a search:
```spl
index=* host=Win10-Target
```

Set time range: **Last 15 minutes**

You should see Windows Event Log entries from Win10-Target.

> ✅ **Take a snapshot:** `Win10-Forwarder Configured` — Full log pipeline working.

---

### Phase 6: Network Verification

#### Step 26 — Verify VM Connectivity

Set both VMs to **Host-only** network mode.

**Check IPs:**
```bash
# On Kali
ip a
```
```cmd
:: On Win10
ipconfig
```

**Test ping:**
```bash
# From Kali
ping <Win10-IP>
```
```powershell
# From Win10
ping <Kali-IP>
```

If ping is blocked, allow ICMP on Windows:
```powershell
New-NetFirewallRule -DisplayName "Allow ICMPv4" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow
```

🎉 **Setup Complete!** Your lab environment is ready.

---

## 🧪 Labs

---

### Lab 1 — Brute Force Detection

> **Goal:** Simulate an RDP brute-force attack from Kali against Windows 10 and detect it in Splunk using failed login events (EventCode 4625).

**MITRE ATT&CK:** T1110.001 (Brute Force: Password Guessing)

#### Pre-requisites on Win10-Target

Disable Network Level Authentication (NLA) to allow Hydra to connect:

```powershell
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name "UserAuthentication" -Value 0
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name "SecurityLayer" -Value 0
```

#### Step 1 — Confirm Target IP

On Win10-Target:
```cmd
ipconfig
```
Note the IP (e.g., `192.168.221.129`).

#### Step 2 — Generate Wordlist

On Kali:
```bash
head -500 /usr/share/wordlists/rockyou.txt > ~/lab_wordlist.txt
echo "Password1" >> ~/lab_wordlist.txt
```

#### Step 3 — Launch Brute-Force Attack

**Option A — Ncrack** (most reliable for RDP):
```bash
ncrack -u labuser -P ~/lab_wordlist.txt rdp://192.168.221.129 --connection-limit 1
```

**Option B — Hydra:**
```bash
hydra -l labuser -P ~/lab_wordlist.txt rdp://192.168.221.129 -t 1 -V -W 3
```

**Option C — NetExec:**
```bash
nxc rdp 192.168.221.129 -u labuser -p ~/lab_wordlist.txt
```

---

#### Phase 1 — DETECT

**Alert Trigger:** High volume of EventCode 4625 (Failed Logon) from a single source IP.

**Core Detection Query:**
```spl
index=win_logs EventCode=4625
| stats count by src_ip, Account_Name
| where count > 10
| sort -count
```

**Time-bucketed Pattern:**
```spl
index=win_logs EventCode=4625
| timechart span=1m count by src_ip
```

**Check for Successful Logons:**
```spl
index=win_logs (EventCode=4624 OR EventCode=4625)
| stats count by EventCode, src_ip, Account_Name
| sort src_ip
```

---

#### Phase 2 — TRIAGE

| Question | Answer Source |
|----------|---------------|
| Which account targeted? | EventCode 4625 → `Account_Name` field |
| Source IP? | `src_ip` — internal or external? |
| How many attempts? | `count` in Splunk query |
| Logon Type? | `Logon_Type` — 10 = RDP |
| Any successes? | EventCode 4624 from same `src_ip` |
| Account locked? | Check AD or local account status |

**Severity Classification:**
- **Medium** — Failures only, no 4624 → Block and monitor
- **CRITICAL** — 4624 appears after multiple 4625s → Breach confirmed
- **Higher Priority** — Internal source IP → Lateral movement or insider threat

---

#### Phase 3 — CONTAIN

```powershell
# Block attacker IP at firewall
sudo ufw deny from <KALI_IP> to any

# Lock targeted account
net user labuser /active:no

# Or via AD
Disable-ADAccount -Identity "labuser"

# Check account status
net user labuser
```

---

#### Phase 4 — INVESTIGATE

**Full Attack Timeline:**
```spl
index=win_logs (EventCode=4625 OR EventCode=4624 OR EventCode=4648) src_ip="<KALI_IP>"
| table _time, EventCode, Account_Name, src_ip, Logon_Type
| sort _time
```

**If Breach Confirmed — Post-Exploitation:**
```spl
index=win_logs EventCode=4688 OR EventCode=1
| where _time > "[time of 4624]"
| table _time, Account_Name, Image, CommandLine
```

---

#### Phase 5 — ESCALATE & DOCUMENT

**Escalate to L2 if:**
- Any EventCode 4624 confirmed from attacker IP
- Attack came from external IP
- Multiple accounts targeted simultaneously
- Attacker IP matches threat intel / blacklist

**Incident Ticket Template:**
```markdown
## INCIDENT REPORT — RDP Brute Force Attack

| Field | Value |
|-------|-------|
| Detection Time | [timestamp] |
| Alert Source | Splunk — EventCode 4625 threshold breach |
| Attacker IP | <KALI_IP> (internal/external) |
| Target Account | labuser |
| Target Host | Win10-Target |
| Total Attempts | [count] |
| Timeframe | [start] to [end] |
| Logon Type | 10 (RemoteInteractive / RDP) |
| Breach Confirmed | Yes / No |
| 4624 Observed | Yes / No |
| Actions Taken | IP blocked, account locked, host isolated |
| Escalated To | [L2 name] |
| MITRE ATT&CK | T1110.001 — Brute Force: Password Guessing |
```

---

#### Phase 6 — REMEDIATE

| Fix | Reason |
|-----|--------|
| Account lockout after 5 attempts | Stops brute force cold |
| Enable MFA on RDP | Password alone is insufficient |
| Enable NLA | Auth required before RDP session opens |
| Disable RDP if not needed | Remove attack surface entirely |
| Geo-block RDP from outside country | Reduce external exposure |
| Deploy fail2ban equivalent | Auto-block threshold-breaching IPs |
| Use non-standard RDP port | Reduces automated scanning hits |

---

### Lab 2 — Network Reconnaissance & Port Scan Detection

> **Goal:** Simulate Nmap port scans from Kali against Windows 10 and detect scanning activity in Splunk by counting distinct destination ports from the same source IP.

**MITRE ATT&CK:** T1046 (Network Service Discovery), T1595 (Active Scanning)

#### Step 1 — Confirm Target IP

On Win10-Target:
```cmd
ipconfig
```

#### Step 2 — Start Packet Capture on Windows

Open **Wireshark** on Win10-Target, select the correct network adapter, and start capturing.

#### Step 3 — Run Nmap Scans from Kali

```bash
# Stealth SYN scan
nmap -sS <Win10-IP>

# Aggressive scan with service/OS detection
nmap -A <Win10-IP>

# All ports scan
nmap -p- <Win10-IP>

# OS fingerprinting
nmap -O <Win10-IP>

# Host discovery (ping sweep)
nmap -sn <Win10-IP>

# Full SYN port scan
nmap -sS -p- <Win10-IP>

# Service and version detection
nmap -sV -sC -p 22,80,443,3389,445,139 <Win10-IP>
```

#### Step 4 — Observe SYN Packets in Wireshark

Apply filter:
```
tcp.flags.syn==1 and tcp.flags.ack==0
```

This reveals a high volume of SYN packets from Kali IP toward many ports — typical of port scanning behavior.

---

#### Phase 1 — DETECT

**Alert Triggers:**
- Sysmon EventCode 3 — burst of network connections to multiple destination ports from a single source IP
- Windows Firewall EventCode 5157 — blocked inbound connection attempts across multiple ports
- No corresponding application traffic pattern — pure port enumeration

**Port Scan Detection:**
```spl
index=win_logs EventCode=3
| stats dc(DestinationPort) as unique_ports, count by SourceIp, DestinationIp
| where unique_ports > 15
| sort -unique_ports
```

**Timeline of Scan:**
```spl
index=win_logs EventCode=3 SourceIp="<KALI_IP>"
| timechart span=10s count
```

**Which Ports Were Probed:**
```spl
index=win_logs EventCode=3 SourceIp="<KALI_IP>"
| stats count by DestinationPort
| sort -count
```

**Firewall Blocked Traffic:**
```spl
index=win_logs EventCode=5157
| stats dc(DestinationPort) as unique_ports by SourceAddress, DestAddress
| where unique_ports > 15
| sort -unique_ports
```

---

#### Phase 2 — TRIAGE

| Question | Where to Look |
|----------|---------------|
| Which IP is scanning? | Sysmon EventCode 3 — `SourceIp` field |
| Internal or external source? | Compare `SourceIp` against internal subnet |
| How many ports probed? | `dc(DestinationPort)` in Splunk |
| How long did scan last? | Earliest vs latest time in results |
| Did scan find open ports? | Check which Destination Ports got responses |
| Did anything follow the scan? | EventCode 4624/4625 from same IP post-scan |

**Severity:**
- Internal IP scanning internally → **Medium** (possible compromised host)
- External IP scanning perimeter → **High** (active external reconnaissance)
- Scan followed by exploitation → **Critical** (escalate now)

---

#### Phase 3 — CONTAIN

```powershell
# Block scanning IP at perimeter firewall

# Block at Windows Firewall (if internal attacker)
netsh advfirewall firewall add rule name="Block Scanner" dir=in action=block remoteip=<KALI_IP>

# Block at Linux perimeter
sudo ufw deny from <KALI_IP> to any
```

---

#### Phase 4 — INVESTIGATE

**Did scanner connect to open ports after scanning?**
```spl
index=win_logs EventCode=3 SourceIp="<KALI_IP>"
| stats count by DestinationPort
| sort -count
```

**Any login attempts from scanning IP after scan?**
```spl
index=win_logs (EventCode=4624 OR EventCode=4625) src_ip="<KALI_IP>"
| table _time, EventCode, Account_Name, src_ip, Logon_Type
| sort _time
```

**Any other hosts scanned from same source?**
```spl
index=win_logs EventCode=3 SourceIp="<KALI_IP>"
| stats dc(DestinationIp) as hosts_scanned, values(DestinationIp) as targets by SourceIp
```

**Attack Timeline:**
```
[T+0]   First Sysmon EventCode 3 from KALI_IP
[T+xs]  Burst of connections across multiple ports — scan pattern confirmed
[T+ym]  Scan ends — unique_ports count peaks
[T+zm]  Any EventCode 4624/4625 from same IP? → exploitation attempt
```

---

#### Phase 5 — ESCALATE & DOCUMENT

**Escalate to L2 when:**
- Scan source is confirmed external IP
- Scan is immediately followed by login attempts or exploitation
- Multiple internal hosts scanned (lateral recon after initial compromise)
- Scan pattern matches known threat actor tooling

**Incident Report Template:**
```markdown
## INCIDENT REPORT — Network Port Scan / Reconnaissance

| Field | Value |
|-------|-------|
| Detection Time | [timestamp] |
| Alert Source | Splunk — Sysmon EventCode 3 dc(DestinationPort) threshold |
| Scanning IP | <KALI_IP> (internal/external) |
| Target Host | Win10-Target |
| Ports Probed | [count] unique ports |
| Scan Duration | [start_time] to [end_time] |
| Open Ports Found | [list ports with inbound connections] |
| Follow-on Action | Yes / No — [login attempts / exploit attempts] |
| Actions Taken | Scanning IP blocked, L2 notified |
| MITRE ATT&CK | T1046 / T1595 |
```

---

#### Phase 6 — REMEDIATE

| Fix | Reason |
|-----|--------|
| Enable host-based firewall on all endpoints | Blocks unsolicited inbound probes |
| Deploy IDS/IPS (Snort / Suricata) | Detects nmap scan signatures automatically |
| Enable Sysmon with network logging (EventCode 3) | Core visibility for scan detection |
| Network segmentation / VLANs | Limits which hosts an attacker can reach |
| Alert rule: dc(DestinationPort) > 15 in 60s | Automates detection |
| Geo-block external IP ranges at perimeter | Reduces external reconnaissance exposure |
| Disable unnecessary open ports/services | Shrinks the attack surface |

---

### Lab 3 — Beaconing / C2 Detection

> **Goal:** Simulate command-and-control style beaconing traffic from Kali to Windows 10 and detect the repeated connection pattern in Splunk using timing-based analysis.

**MITRE ATT&CK:** T1071 (Application Layer Protocol)

#### Step 1 — Confirm Target IP

On Win10-Target:
```cmd
ipconfig
```

#### Step 2 — Start a Listener on Windows

Download Nmap on Win10 VM to get Ncat, then open Command Prompt as Administrator:

```cmd
ncat -lvp 4444
```

If `nc` is not available, use `ncat` (included with Nmap).

#### Step 3 — Start Repeated Connections from Kali

From Kali, run a loop to generate repeated outbound connections every 30 seconds:

```bash
while true; do nc 192.168.221.129 4444; sleep 30; done
```

This creates a predictable callback pattern similar to beaconing or C2 activity.

#### Step 4 — Observe Traffic in Wireshark

On Windows, open Wireshark and watch repeated connection attempts come in at regular intervals.

Filter: `tcp.port == 4444`

---

#### Detection

**Data Source:** Network or traffic logs forwarded to Splunk.

**Key Idea:** Look for repeated connections from the same source to the same destination at consistent time gaps.

**Beaconing Detection Query:**
```spl
index=main
| sort 0 src_ip dest_ip _time
| streamstats current=f last(_time) as prev_time by src_ip dest_ip
| eval interval = _time - prev_time
| stats count, avg(interval) as avg_interval, stdev(interval) as stdev_interval by src_ip dest_ip
| where count > 5 AND avg_interval < 300 AND stdev_interval < 10
| sort -count
```

This type of timing analysis is commonly used to identify beaconing because malicious check-ins often happen at regular intervals with low standard deviation.

---

#### Triage Notes

- **True positive** in the lab
- **Source IP:** Kali attacker VM
- **Destination IP:** Windows 10 VM
- **Interval:** Highly consistent between connections — main clue for beaconing
- **Confirmed by:** Both Wireshark and Splunk

---

## 🗺️ MITRE ATT&CK Mapping

### Lab 1 — Brute Force

| Technique ID | Technique Name | Detail |
|--------------|----------------|--------|
| T1110 | Brute Force | Parent technique |
| T1110.001 | Password Guessing | Wordlist against RDP |
| T1078 | Valid Accounts | If brute force succeeds |
| T1021.001 | Remote Services: RDP | Access method used |
| T1133 | External Remote Services | RDP exposed externally |

**Kill Chain:**
```
Credential Access → [T1110.001] Brute Force RDP
    └── Success → [T1078] Valid Accounts
        └── [T1021.001] RDP Lateral Movement
```

### Lab 2 — Port Scanning

| ID | Technique | Context |
|----|-----------|---------|
| T1595 | Active Scanning | Attacker scans IP ranges and ports |
| T1595.001 | Scanning IP Blocks | nmap ping sweep (-sn) |
| T1595.002 | Vulnerability Scanning | nmap service detection (-sV -sC) |
| T1046 | Network Service Discovery | Full port enumeration (-p-) |
| T1590 | Gather Victim Network Information | OS fingerprinting (-O) |

**Kill Chain:**
```
Reconnaissance
    └── [T1595.001] Ping Sweep — host discovery
        └── [T1046] Port Scan — service enumeration
            └── [T1595.002] Version Detection — vulnerability mapping
                └── Weaponization → Exploitation
```

### Lab 3 — C2 Beaconing

| ID | Technique | Context |
|----|-----------|---------|
| T1071 | Application Layer Protocol | C2 communication channel |
| T1071.001 | Web Protocols | HTTP/HTTPS beaconing |
| T1001 | Data Obfuscation | Jitter / randomization in beacon timing |

---

## 📊 Key Event IDs

### Windows Security Log

| Event ID | Description |
|----------|-------------|
| 4624 | Successful logon — breach confirmation |
| 4625 | Failed logon — core brute force indicator |
| 4648 | Logon with explicit credentials |
| 4776 | NTLM credential validation attempt |
| 4740 | Account locked out |

### Sysmon

| Event ID | Description |
|----------|-------------|
| 1 | Process creation |
| 3 | Network connection — core scan/beacon detection |
| 7 | Image loaded |
| 8 | CreateRemoteThread |
| 10 | ProcessAccess |

### Windows Firewall

| Event ID | Description |
|----------|-------------|
| 5156 | Allowed connection through firewall |
| 5157 | Blocked connection — scan traffic |

### Logon Types

| Type | Name | Meaning |
|------|------|---------|
| 2 | Interactive | Logged in at keyboard/console |
| 3 | Network | Access over network (SMB, DC auth) |
| 4 | Batch | Scheduled tasks |
| 5 | Service | Service account logon |
| 7 | Unlock | Workstation unlock |
| 8 | NetworkCleartext | Cleartext credentials used |
| 9 | NewCredentials | RunAs with different creds |
| 10 | RemoteInteractive | RDP logon |
| 11 | CachedInteractive | Cached domain creds |

---

## 👤 Author

**SOC Analyst Home Lab** — Built for educational purposes in an isolated environment.

> ⚠️ **Disclaimer:** This lab was built for educational purposes in an isolated environment. No production systems were used. All attack traffic stays within the Host-only virtual network.

---

*Last updated: July 2026*
"""
