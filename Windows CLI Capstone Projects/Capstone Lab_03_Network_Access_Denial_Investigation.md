---
tags:
  - cybersecurity
  - windows-cli
  - networking
  - troubleshooting
  - incident-response
  - capstone
status: complete
lab: 3
topic: Network Access Denial Investigation
environment: Windows PowerShell / CMD
---

# Capstone Lab 03 – Network Access Denial Investigation

## Scenario

> Users in the **Sales department** cannot access the internet. Investigate the cause using CLI tools, log all evidence, and produce a report suitable for non-technical readers.
>
> **Simulation method:** The VM's default gateway was manually set to `192.168.1.200/24` — an invalid gateway that cannot route to the internet — to reproduce the symptoms of a misconfigured network.

---

## Theory & Background

### The OSI Model and Where This Problem Lives

Network connectivity issues are diagnosed layer by layer. This lab targets **Layer 3 (Network)** — the IP routing layer.

| OSI Layer | Relevant to this lab? | What to check |
|---|---|---|
| Layer 1 – Physical | ✅ (indirectly) | Is the cable plugged in / Wi-Fi associated? |
| Layer 2 – Data Link | ✅ (indirectly) | MAC address, ARP resolution |
| Layer 3 – Network | ✅ **Primary issue** | IP address, subnet mask, default gateway |
| Layer 4 – Transport | Downstream | TCP/UDP connections (netstat) |
| Layer 7 – Application | Downstream | DNS, HTTP proxy, application errors |

### What is a Default Gateway?

The **default gateway** is the router your machine sends traffic to when the destination IP is outside your local subnet. If the gateway is wrong, your machine can reach local devices but cannot route packets to the internet — exactly what happened here.

### DNS vs. Gateway Issues

A common mistake is assuming the problem is DNS when it is actually a routing issue. The correct diagnostic order is:
1. Check IP config (gateway correct?)
2. Ping the gateway (is the router reachable?)
3. Ping a public IP (e.g. `8.8.8.8`) — if this fails with a gateway ping also failing, it's routing
4. Only then test DNS (`nslookup`) — DNS failure on top of a routing failure is a symptom, not the cause

---

## Step 1 – Run ipconfig

<img width="889" height="365" alt="image" src="https://github.com/user-attachments/assets/55244293-b6db-4151-8a9f-199eae584a93" />

**Screenshot evidence:** `ipconfig` output shows:
- IPv4 Address: `192.168.1.100`
- Subnet Mask: `255.255.255.0` (/24 — correct)
- Default Gateway: `192.168.1.200` (**incorrect** — highlighted in red)

The machine has a valid IP address and correct subnet, but the gateway `192.168.1.200` is not the actual edge router. This means all traffic destined for external networks has no valid exit path.

```powershell
ipconfig
ipconfig /all          # Shows additional info: DNS servers, DHCP lease, MAC address
```

---

## Step 2 – Test Connectivity (ping, tracert, netstat)

### ping – Test Internet Reachability

<img width="970" height="321" alt="image" src="https://github.com/user-attachments/assets/34374450-95aa-4730-820c-a20909e8a746" />

**Screenshot evidence:** `ping 8.8.8.8` (Google's public DNS server) results in 4 timeouts with `100% packet loss` (highlighted in red). No packets reached the internet.

```powershell
ping 8.8.8.8
ping 8.8.8.8 -n 10     # Send 10 packets instead of the default 4
```

### tracert – Trace the Route

**Screenshot evidence:** `tracert 8.8.8.8` shows only 1 hop: `DESKTOP-NBVFIGL [192.168.1.100]` reports `Destination host unreachable` (highlighted in red). The trace never left the local machine — confirming no gateway was reachable to forward the packets.

```cmd
tracert 8.8.8.8
```

> 💡 **Reading tracert:** Each line is a hop (router). `*` means the hop timed out. If hop 1 itself is unreachable, the gateway is the problem. If hop 1 replies but later hops fail, the issue is further upstream in the network.

### netstat – Check Active Connections

<img width="635" height="133" alt="image" src="https://github.com/user-attachments/assets/592f0071-86f6-4861-8c2b-99c3041bbd72" />

**Screenshot evidence:** `netstat` output shows `Active Connections` with an empty table — no foreign address connections. This is consistent with a machine that cannot reach external hosts.

```cmd
netstat
netstat -an            # All connections + listening ports, numeric format
netstat -b             # Show which process owns each connection (requires admin)
```

---

## Step 3 – Log Evidence to File

<img width="824" height="845" alt="image" src="https://github.com/user-attachments/assets/7ab1742d-aa26-4d49-8f8b-4874ae03db5b" />

**Screenshot evidence:** A series of commands redirect and append all diagnostic outputs into a single report file at `C:\networklab\network_outage_report.txt`. The subsequent `Get-Content` shows the consolidated output — ipconfig, both ping results, tracert, and netstat — all in one file.

```powershell
# Create the output directory first
mkdir C:\networklab

# Run diagnostics and capture output (note: first command uses > to create, rest use >> to append)
ipconfig > network_outage_report.txt
ping 8.8.8.8 >> .\network_outage_report.txt
ping 8.8.8.8 >> .\network_outage_report.txt
tracert 8.8.8.8 >> .\network_outage_report.txt
netstat >> .\network_outage_report.txt

# Verify the report contents
Get-Content .\network_outage_report.txt
```

> 💡 **`>` vs `>>`:** `>` creates or overwrites a file. `>>` appends to an existing file. Always use `>>` after the first command to avoid erasing previous output.

---

## Step 4 – Non-Technical Report

The following report was produced for the Sales department manager and IT management — written in plain language without technical jargon.

---

**Network Access Issue Report – Sales Department**
**Report created: 14/03/2026**

**Issue Summary**
Users in the Sales department reported that they were unable to access the internet from their computers.

**Investigation**
Basic network checks were performed to review the computer's connection settings and test connectivity. The system had a valid internal IP address, but it has the wrong default gateway — the network setting that allows the computer to communicate with external networks such as the internet. Connectivity tests confirmed that the computer could not reach external internet addresses.

**Root Cause**
The network configuration on the affected system was misconfigured, which prevented internet traffic from leaving the local network.

**Solution**
The network setting of the affected system needs to be configured correctly by pointing the default gateway to the main router. Once corrected, internet connectivity will be restored.

**Expected Result**
Users in the Sales department can access the internet normally.

**Expected time to resolve:** ~1 minute

**Evidence**
All network connectivity test evidence is logged at `C:\networklab\network_outage_lab.txt`.

---

## Q&A – Additional Investigations

### What command helps verify DNS resolution?

<img width="406" height="320" alt="image" src="https://github.com/user-attachments/assets/e67a9249-1780-4279-a3cd-2bf94ae590eb" />

**Screenshot evidence:** `nslookup google.com` was run. The output shows repeated `DNS request timed out` messages — the DNS server (`8.8.8.8`) could not be reached because the gateway is broken. This confirms DNS failure is a **downstream symptom** of the gateway misconfiguration, not the root cause.

```cmd
nslookup google.com
nslookup google.com 8.8.8.8    # Query a specific DNS server
nslookup google.com 1.1.1.1    # Test with Cloudflare DNS
```

### How to check for proxy issues?

<img width="454" height="143" alt="image" src="https://github.com/user-attachments/assets/9902255e-9fec-4f1d-bd72-145d2614ea20" />

**Screenshot evidence:** `netsh winhttp show proxy` returns `Direct access (no proxy server)`. This confirms that no proxy is configured and proxy is not contributing to the issue.

```cmd
netsh winhttp show proxy
```

### How to check for firewall issues?

<img width="944" height="903" alt="image" src="https://github.com/user-attachments/assets/dea10592-ddd9-481b-9cc4-3235548b65bb" />

**Screenshot evidence:** `netsh advfirewall show allprofiles` shows all three profiles (Domain, Private, Public) have `Firewall Policy: BlockInbound, AllowOutbound` (highlighted in red). Since outbound traffic is **allowed**, the firewall is not blocking the user's internet access.

```powershell
netsh advfirewall show allprofiles
```

> 💡 **Interpreting firewall profiles:**
> - **Domain Profile** — applies when connected to a corporate domain network
> - **Private Profile** — applies on trusted home/office networks
> - **Public Profile** — applies on untrusted networks (cafes, airports)
>
> `AllowOutbound` on all profiles means outbound connections are not being blocked. If it showed `BlockOutbound`, that would be the firewall blocking internet access.

---

## Fix – Correcting the Gateway

```powershell
# Set the correct default gateway (replace with actual router IP)
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.1.100 -PrefixLength 24 -DefaultGateway 192.168.1.1

# Or use netsh (CMD)
netsh interface ip set address "Ethernet" static 192.168.1.100 255.255.255.0 192.168.1.1

# Verify the fix
ipconfig
ping 8.8.8.8
```

---

## Tools & Alternatives

| Tool Used | Purpose | Alternative |
|---|---|---|
| `ipconfig` | View IP configuration | `Get-NetIPConfiguration` (PowerShell) |
| `ping` | Test ICMP reachability | `Test-Connection` (PowerShell) |
| `tracert` | Trace routing hops | `Test-NetConnection -TraceRoute` (PowerShell) |
| `netstat` | View active TCP/UDP connections | `Get-NetTCPConnection` (PowerShell) |
| `nslookup` | Test DNS resolution | `Resolve-DnsName` (PowerShell) |
| `netsh winhttp show proxy` | Check WinHTTP proxy settings | Check `Internet Options > Connections > LAN Settings` in GUI |
| `netsh advfirewall show allprofiles` | View firewall rules | `Get-NetFirewallProfile` (PowerShell) |

---

## Cheatsheet – Key Commands

```powershell
# --- IP Configuration ---
ipconfig                                        # Basic IP info
ipconfig /all                                   # Full details inc. DNS, DHCP, MAC
ipconfig /release                               # Release DHCP lease
ipconfig /renew                                 # Renew DHCP lease
ipconfig /flushdns                              # Clear DNS resolver cache

# --- Connectivity Testing ---
ping 8.8.8.8                                    # Test internet reachability (ICMP)
ping 8.8.8.8 -n 10                              # Send 10 packets
tracert 8.8.8.8                                 # Trace routing hops (CMD)
Test-NetConnection -ComputerName 8.8.8.8        # PowerShell equivalent of ping
Test-NetConnection -ComputerName google.com -Port 443   # Test specific TCP port

# --- Active Connections ---
netstat                                         # Show active connections
netstat -an                                     # All connections, numeric
netstat -b                                      # Show owning process (admin)

# --- DNS ---
nslookup google.com                             # Test DNS resolution
nslookup google.com 8.8.8.8                     # Query a specific DNS server
Resolve-DnsName google.com                      # PowerShell DNS lookup

# --- Proxy & Firewall ---
netsh winhttp show proxy                        # Check WinHTTP proxy
netsh advfirewall show allprofiles              # View all firewall profiles
Get-NetFirewallProfile                          # PowerShell firewall info

# --- Output Logging ---
ipconfig > report.txt                           # Create/overwrite file
ping 8.8.8.8 >> report.txt                      # Append to file
Get-Content report.txt                          # Read the file
```

---

## Summary

| Requirement | Implemented |
|---|---|
| Run `ipconfig` to identify misconfiguration | ✅ Wrong gateway `192.168.1.200` identified |
| Use `ping` to test internet reachability | ✅ 100% packet loss to `8.8.8.8` |
| Use `tracert` to trace routing | ✅ Route fails at hop 1 — gateway unreachable |
| Use `netstat` to check active connections | ✅ No active connections — consistent with no gateway |
| Log all output to file | ✅ Redirected to `C:\networklab\network_outage_report.txt` |
| Produce a non-technical report | ✅ Formatted report with issue, root cause, solution |
| Check DNS resolution | ✅ `nslookup` — DNS fails because gateway is broken |
| Check proxy configuration | ✅ `netsh winhttp show proxy` — no proxy, not an issue |
| Check firewall policy | ✅ `netsh advfirewall` — outbound allowed, not an issue |
