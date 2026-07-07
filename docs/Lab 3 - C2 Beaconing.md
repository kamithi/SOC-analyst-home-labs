# Lab 3 — Beaconing / C2 Detection

**MITRE ATT&CK:** T1071 (Application Layer Protocol)  
**Tools:** Netcat, Ncat, Wireshark, Splunk, VMware

---

## Goal

Simulate command-and-control (C2) style beaconing traffic from Kali Linux to Windows 10 and detect the repeated connection pattern in Splunk using timing-based analysis.

---

## Environment

| Role | Machine | IP Address |
|------|---------|------------|
| Attacker | Kali Linux | 192.168.221.128 |
| Target | Windows 10 | 192.168.221.129 |
| SIEM | Splunk Enterprise | Host Machine |
| Listener Port | 4444 | TCP |

---

## Lab Setup

1. Both VMs connected to the isolated Host-only network
2. Windows 10 logging and forwarding to Splunk configured
3. Wireshark available on Windows 10 VM
4. Clean snapshot taken before starting

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

### Step 2 — Start a Listener on Windows

Download and install Nmap on the Windows 10 VM to get `ncat`. Then open **Command Prompt as Administrator** and start a listener on port 4444:

```cmd
ncat -lvp 4444
```

If `nc` is not available by default, `ncat` (included with Nmap) works identically.

![Ncat Listener](screenshots/lab3/kali-beacon.png)

**Expected output:**
```
Ncat: Version 7.99
Ncat: Listening on [::]:4444
Ncat: Listening on 0.0.0.0:4444
```

---

### Step 3 — Start Repeated Connections from Kali

From the Kali VM, run a simple loop to generate repeated outbound connections every 30 seconds:

```bash
while true; do nc 192.168.221.129 4444; sleep 30; done
```

This creates a predictable callback pattern similar to real-world beaconing or C2 activity.

![Kali Beacon Loop](screenshots/lab3/kali-beacon.png)

**What this simulates:**
- Malware calling back to a C2 server at regular intervals
- Beaconing to check for new commands
- Persistent communication channel

---

### Step 4 — Observe Traffic in Wireshark

On Windows 10, open Wireshark and watch the repeated connection attempts come in at regular intervals.

**Wireshark Display Filter:**
```
tcp.port == 4444
```

![Wireshark Beaconing Traffic](screenshots/lab3/Wireshark-showing-periodic-Kali-IP.png)



![Wireshark Beaconing Traffic](screenshots/lab3/C2-Beaconing.png)

**What to look for:**
- Connections from `192.168.221.128` (Kali) to `192.168.221.129:4444`
- Regular timing pattern (approximately every 30 seconds)
- TCP SYN packets followed by connection attempts

---

## Detection

### Data Source

Network or traffic logs forwarded to Splunk (Sysmon EventCode 3 or firewall logs).

### Key Detection Concept

Look for **repeated connections from the same source to the same destination at consistent time gaps**. Real-world beaconing often has:
- Regular intervals (e.g., every 30 seconds, 1 minute, 5 minutes)
- Low standard deviation in timing (highly predictable)
- Same source/destination pair
- Minimal data transfer per connection

### Beaconing Detection Query

``` spl
index=main
| sort 0 src_ip dest_ip _time
| streamstats current=f last(_time) as prev_time by src_ip dest_ip
| eval interval = _time - prev_time
| stats count, avg(interval) as avg_interval, stdev(interval) as stdev_interval by src_ip dest_ip
| where count > 5 AND avg_interval < 300 AND stdev_interval < 10
| sort -count
```

**How this works:**
1. `sort 0 src_ip dest_ip _time` - Orders events by connection pair and time
2. `streamstats current=f last(_time) as prev_time` - Gets the previous connection time for each pair
3. `eval interval = _time - prev_time` - Calculates time between connections
4. `stats avg(interval), stdev(interval)` - Computes average interval and standard deviation
5. `where count > 5 AND avg_interval < 300 AND stdev_interval < 10` - Filters for frequent, regular connections

**Why this catches beaconing:**
- `count > 5` - At least 5 connections (eliminates one-offs)
- `avg_interval < 300` - Average gap under 5 minutes (beaconing is frequent)
- `stdev_interval < 10` - Low standard deviation = highly regular timing

---

### Triage Notes

| Field | Value |
|-------|-------|
| **True Positive** | Yes - confirmed lab simulation |
| **Source IP** | Kali attacker VM (192.168.221.128) |
| **Destination IP** | Windows 10 VM (192.168.221.129) |
| **Destination Port** | 4444 |
| **Interval** | Highly consistent (~30 seconds) |
| **Key Clue** | Low standard deviation in timing |
| **Confirmed By** | Both Wireshark and Splunk |

---

### Severity Assessment

| Scenario | Severity |
|----------|----------|
| Internal beaconing (same subnet) | **High** - Possible compromised host |
| External beaconing (outbound to internet) | **Critical** - Active C2 communication |
| Beaconing followed by data exfiltration | **Critical** - Escalate immediately |

---

### Containment Actions

```powershell
# Block the beaconing IP at firewall
netsh advfirewall firewall add rule name="Block C2 Beacon" dir=out action=block remoteip=192.168.221.129
```

```bash
# On perimeter firewall
sudo ufw deny from 192.168.221.128 to any
```

1. Isolate the affected host from the network
2. Capture memory dump before shutdown
3. Preserve logs for forensic analysis
4. Check for additional compromised hosts (same beacon pattern)

---

## MITRE ATT&CK Mapping

| ID | Technique | Context |
|----|-----------|---------|
| T1071 | Application Layer Protocol | C2 communication channel |
| T1071.001 | Web Protocols | HTTP/HTTPS beaconing (common variant) |
| T1001 | Data Obfuscation | Jitter / randomization in beacon timing (evasion) |
| T1021 | Remote Services | Alternative to direct C2 for persistence |

**Kill Chain:**

```
Command and Control
    └── [T1071] Application Layer Protocol
        └── [T1071.001] Web Protocols (HTTP/HTTPS beaconing)
            └── [T1001] Data Obfuscation (jitter to evade detection)
```

---

## Key Differences: Lab vs. Real-World

| Aspect | Lab | Real-World |
|--------|-----|------------|
| **Interval** | Fixed 30 seconds | Variable, often with jitter |
| **Port** | 4444 (obvious) | Common ports (80, 443, 53) |
| **Protocol** | Raw TCP | HTTP/HTTPS, DNS, ICMP |
| **Data** | Minimal | Encrypted commands/responses |
| **Evasion** | None | Domain fronting, fast-flux, DGA |

**Why real C2 is harder to detect:**
- **Jitter:** Random intervals (e.g., 30s ± 10s) to break timing patterns
- **Protocol mimicry:** Using HTTPS to blend with normal traffic
- **Domain Generation Algorithms (DGA):** Changing C2 domains frequently
- **Fast-flux DNS:** Rapidly changing IP addresses behind a domain

---

## Enhanced Detection Queries

### With Jitter Tolerance

```spl
index=main
| sort 0 src_ip dest_ip _time
| streamstats current=f last(_time) as prev_time by src_ip dest_ip
| eval interval = _time - prev_time
| stats count, avg(interval) as avg_interval, stdev(interval) as stdev_interval, median(interval) as med_interval by src_ip dest_ip
| where count > 5 AND avg_interval < 600 AND stdev_interval < 60
| sort -count
```

### Detecting Common C2 Ports

```spl
index=win_logs EventCode=3
| where DestinationPort IN (80, 443, 8080, 53, 123, 4444, 5555)
| stats count, dc(DestinationPort) as unique_ports by SourceIp, DestinationIp
| where count > 10
| sort -count
```

### Long-Term Beaconing (Low and Slow)

```spl
index=main earliest=-7d
| sort 0 src_ip dest_ip _time
| streamstats current=f last(_time) as prev_time by src_ip dest_ip
| eval interval = _time - prev_time
| stats count, avg(interval) as avg_interval, stdev(interval) as stdev_interval by src_ip dest_ip
| where count > 50 AND avg_interval > 3600 AND stdev_interval < 300
| sort -count
```

---

## What I Learned

- **Beaconing uses predictable timing intervals** — this is both its purpose and its weakness
- **Low standard deviation is the key detection signal** — real users don't connect at exactly 30-second intervals
- **Real-world C2 adds jitter** (randomization) to evade timing-based detection
- **Streamstats in SPL** enables powerful time-series analysis for identifying patterns
- **Protocol and port choice matters** — C2 often hides on ports 80/443 to blend in
- **Context is critical** — a single connection means nothing; patterns over time reveal beaconing
- **Memory forensics** (e.g., Volatility) can find beaconing malware even if logs are cleared
