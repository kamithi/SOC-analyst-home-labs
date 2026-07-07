# Lab 1 — Brute Force Detection

 
**MITRE ATT&CK:** T1110.001 (Brute Force: Password Guessing)  
**Tools:** Ncrack, Hydra, Splunk, VMware

---

## Goal

Simulate an RDP brute-force attack from Kali Linux against Windows 10 and detect failed login attempts in Splunk using EventCode 4625.

---

## Environment

| Role | Machine | IP Address |
|------|---------|------------|
| Attacker | Kali Linux | 192.168.221.128 |
| Target | Windows 10 | 192.168.221.129 |
| SIEM | Splunk Enterprise | Host Machine |

---

## Pre-requisites

Disable Network Level Authentication (NLA) on Windows 10 to allow the brute-force tool to connect:

```powershell
Set-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\Terminal Server\\WinStations\\RDP-Tcp' -Name "UserAuthentication" -Value 0
Set-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\Terminal Server\\WinStations\\RDP-Tcp' -Name "SecurityLayer" -Value 0
```

> ⚠️ **Security Note:** Only disable NLA in an isolated lab environment. Always enable NLA in production.

---

## Attack Steps

### Step 1 — Confirm Target IP

On Windows 10 Target VM:

```cmd
ipconfig
```

Note the IPv4 address (e.g., `192.168.221.129`).

![Windows 10 IP Configuration](screenshots/lab1/win10-ipconfig.png)

---

### Step 2 — Generate Wordlist

On Kali Linux Attacker VM:

```bash
head -500 /usr/share/wordlists/rockyou.txt > ~/lab_wordlist.txt
echo "Password1" >> ~/lab_wordlist.txt
```

This creates a small wordlist for faster lab execution.

---

### Step 3 — Launch Brute-Force Attack

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

![Ncrack Brute Force Attack](screenshots/lab1/ncrack-attack.png)

---

## Detection

### Phase 1 — DETECT

**Alert Trigger:** High volume of EventCode 4625 (Failed Logon) from a single source IP within a short time window.

**Core Splunk Detection Query:**

```spl
index=win_logs EventCode=4625
| stats count by src_ip, Account_Name
| where count > 10
| sort -count
```

**Time-Bucketed Pattern Analysis:**

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

![Splunk EventCode 4625 Detection](screenshots/lab1/splunk-4625.png)

---

### Phase 2 — TRIAGE

| Question | Answer Source |
|----------|---------------|
| Which account was targeted? | EventCode 4625 → `Account_Name` field |
| What is the source IP? | `src_ip` — determine if internal or external |
| How many attempts were made? | `count` in Splunk query results |
| What is the Logon Type? | `Logon_Type` field — **10 = RDP** |
| Were any logons successful? | Check for EventCode 4624 from same `src_ip` |
| Is the account locked out? | Check AD or local account status |

**Severity Classification:**

| Scenario | Severity | Action |
|----------|----------|--------|
| Failures only, no EventCode 4624 | **Medium** | Block IP and monitor |
| EventCode 4624 appears after multiple 4625s | **CRITICAL** | Breach confirmed — isolate host |
| Internal source IP | **Higher Priority** | Possible lateral movement or insider threat |

---

### Phase 3 — CONTAIN

**Immediate Actions:**

```bash
# Block attacker IP at firewall (Linux)
sudo ufw deny from <KALI_IP> to any
```

```powershell
# Lock targeted account on Windows
net user labuser /active:no

# Or via Active Directory
Disable-ADAccount -Identity "labuser"

# Verify account status
net user labuser
```

**If breach confirmed (EventCode 4624 observed):**
1. Isolate target host **immediately**
2. Preserve logs — do not clear Event Viewer
3. Screenshot Splunk dashboard as evidence

---

### Phase 4 — INVESTIGATE

**Full Attack Timeline:**

```spl
index=win_logs (EventCode=4625 OR EventCode=4624 OR EventCode=4648) src_ip="<KALI_IP>"
| table _time, EventCode, Account_Name, src_ip, Logon_Type
| sort _time
```

**Logon Type Breakdown:**

```spl
index=win_logs EventCode=4625 src_ip="<KALI_IP>"
| stats count by Logon_Type
| eval Logon_Meaning=case(
    Logon_Type=="3", "Network",
    Logon_Type=="10", "RDP RemoteInteractive",
    Logon_Type=="7", "Unlock",
    true(), "Other")
```

**If Breach Confirmed — Post-Exploitation Analysis:**

```spl
index=win_logs EventCode=4688 OR EventCode=1
| where _time > "[time of 4624]"
| table _time, Account_Name, Image, CommandLine
```

**Timeline to Build:**

```
[T+0]    First EventCode 4625 from KALI_IP
[T+Xm]   Flood of 4625s — brute force confirmed
[T+Ym]   EventCode 4624 appears — SUCCESS (if wordlist hit)
[T+Zm]   Post-auth activity — new processes? New connections?
```

---

### Phase 5 — ESCALATE & DOCUMENT

**Escalate to L2 if:**
- Any EventCode 4624 confirmed from attacker IP
- Attack originated from external IP address
- Multiple accounts targeted simultaneously
- Attacker IP matches known threat intelligence or blacklist

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
| EventCode 4624 Observed | Yes / No — [timestamp if yes] |
| Actions Taken | IP blocked, account locked, host isolated |
| Escalated To | [L2 analyst name] |
| MITRE ATT&CK | T1110.001 — Brute Force: Password Guessing |
```

---

### Phase 6 — REMEDIATE

| Fix | Reason |
|-----|--------|
| Account lockout after 5 failed attempts | Stops brute force cold |
| Enable Multi-Factor Authentication (MFA) on RDP | Password alone is insufficient |
| Enable Network Level Authentication (NLA) | Authentication required before RDP session opens |
| Disable RDP if not business-critical | Remove attack surface entirely |
| Geo-block RDP from outside the country | Reduce external exposure |
| Deploy fail2ban or equivalent | Auto-block IPs that breach thresholds |
| Use non-standard RDP port (not 3389) | Reduces automated scanning hits |
| Alert tuning | Lower threshold for slow/low-and-slow attacks |

---

## MITRE ATT&CK Mapping

| Technique ID | Technique Name | Context |
|--------------|----------------|---------|
| T1110 | Brute Force | Parent technique |
| T1110.001 | Password Guessing | Wordlist attack against RDP |
| T1078 | Valid Accounts | If brute force succeeds |
| T1021.001 | Remote Services: RDP | Access method used |
| T1133 | External Remote Services | RDP exposed externally |

**Kill Chain:**

```
Credential Access
    └── [T1110.001] Brute Force: Password Guessing (RDP)
        └── Success → [T1078] Valid Accounts
            └── [T1021.001] RDP Lateral Movement
```

---

## Key Event IDs

| Event ID | Source | Description |
|----------|--------|-------------|
| 4625 | Security | Failed logon — core brute force indicator |
| 4624 | Security | Successful logon — breach confirmation |
| 4648 | Security | Logon with explicit credentials |
| 4776 | Security | NTLM credential validation attempt |
| 4740 | Security | Account locked out |

---

## What I Learned

- **EventCode 4625** is the primary indicator for brute-force attacks in Windows environments
- **Threshold-based alerting** (e.g., >10 failures in 60 seconds) effectively catches automated attacks
- **Network Level Authentication (NLA)** should always be enabled in production to prevent unauthenticated RDP attempts
- **Low-and-slow attacks** require lower thresholds and longer time windows for detection
- **MITRE ATT&CK T1110.001** provides a standardized way to classify and communicate brute-force incidents
- The full **incident response lifecycle** (Detect → Triage → Contain → Investigate → Escalate → Remediate) ensures no step is missed during a real incident
"""
