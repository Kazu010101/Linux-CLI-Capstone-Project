---
tags:
  - cybersecurity
  - linux-cli
  - networking
  - firewall
  - ufw
  - iptables
  - troubleshooting
  - capstone
status: complete
lab: 4
topic: Network Diagnostics and Firewall Recovery
environment: Linux (Kali) — Bash
---

# Linux Lab 04 – Network Diagnostics and Firewall Recovery

## Scenario

> A staging server within **NovaStack's** internal network is unable to reach the internet, preventing package updates and external API access. Troubleshoot and fix the problem using only CLI tools.
>
> **Environment:** Kali Linux VM with Bridged Adapter — IP `192.168.1.14/24`, gateway `192.168.1.1`
> **Simulated fault:** `ufw default deny outgoing` — all outbound traffic dropped

---

## Theory & Background

### OSI Layer Triage — Bottom Up

Network troubleshooting is most efficient when done layer by layer from the bottom up:

| Layer | Tool | What to check |
|---|---|---|
| L1 — Physical | `ifconfig`, `ip addr` | Is the interface UP? Is an IP assigned? |
| L2 — Data Link | `ip neigh` | Can we resolve the gateway's MAC address (ARP)? |
| L3 — Network | `ip route`, `ping` | Is the route to the gateway correct? Can we ping it? |
| L4 — Transport | `ss`, `netstat` | Are TCP/UDP connections being established? |
| L7 — Application | `dig`, `curl` | Is DNS resolving? Is HTTP/HTTPS working? |

### `ufw` vs `iptables`

`ufw` (Uncomplicated Firewall) is a frontend for `iptables` that simplifies rule management:

| | `ufw` | `iptables` |
|---|---|---|
| Syntax | Human-readable (`ufw allow out 443/tcp`) | Complex chains and targets |
| Use case | Quick firewall setup | Fine-grained packet filtering |
| Relationship | `ufw` writes `iptables` rules behind the scenes |

When `ufw default deny outgoing` is set, `iptables` reflects this as `Chain OUTPUT (policy DROP)`.

### Why `ping 8.8.8.8` Works Even When DNS Doesn't

`ping 8.8.8.8` bypasses DNS entirely — it sends ICMP packets directly to the IP address. If DNS is broken but routing is working, `ping 8.8.8.8` will succeed. This makes it the ideal first test to isolate whether the problem is routing (Layer 3) or DNS (Layer 7).

### ARP Cache Status — `STALE` vs `REACHABLE`

`ip neigh` shows the ARP table (IP-to-MAC mappings):
- `REACHABLE` — entry was recently confirmed by bidirectional communication
- `STALE` — entry exists but hasn't been confirmed recently; will be revalidated on next use
- `FAILED` — ARP resolution failed; the host is unreachable at Layer 2

---

## Setup — Simulate the Firewall Misconfiguration

### Step 1: Enable ufw

![[Pasted image 20260418150505.png]]
**Screenshot evidence (image2):** `ufw enable` as root confirms `Firewall is active and enabled on system startup`. Default state: `deny (incoming), allow (outgoing)` — outbound works before the misconfiguration.

```bash
sudo su -           # Switch to root for ufw management
ufw enable
ufw status verbose  # Confirm: deny incoming, allow outgoing
```

### Step 2: Apply the Misconfiguration

![[Pasted image 20260418150546.png]]
**Screenshot evidence (image3):** `ufw default deny incoming && ufw default deny outgoing` changes both default policies to DENY.

```bash
ufw default deny incoming && ufw default deny outgoing
```

![[Pasted image 20260418150624.png]]
**Screenshot evidence (image4):** `ufw status verbose` now shows `Default: deny (incoming), deny (outgoing), disabled (routed)` — confirming the fault is active.

### Step 3: Verify the Fault Symptoms

![[Pasted image 20260418150654.png]]
**Screenshot evidence (image5):** Browser shows "Server Not Found" for `www.wikipedia.org` — internet is broken.

![[Pasted image 20260418150719.png]]
**Screenshot evidence (image6):** `sudo apt update` fails with `Temporary failure resolving 'http.kali.org'` — package manager cannot reach repositories.

---

## Objective 1 — Inspect Interface Status (Layer 1 & 2)

### `ifconfig` — Layer 1 Check

![[Pasted image 20260418150900.png]]
**Screenshot evidence (image7):** `ifconfig` output shows `eth0` interface with flags `UP,BROADCAST,RUNNING,MULTICAST` and `inet 192.168.1.14 netmask 255.255.255.0 broadcast 192.168.1.255` (highlighted in red). The interface is UP and has a valid IP — **Layer 1 is fine**.

```bash
ifconfig
```

### `ip addr` — Alternative Interface Check

![[Pasted image 20260418150959.png]]
**Screenshot evidence (image1):** `ip addr` shows `eth0` as `UP,LOWER_UP` with `inet 192.168.1.14/24 brd 192.168.1.255` (highlighted in red). Same conclusion — interface is up.

```bash
ip addr
ip addr show eth0    # Specific interface only
```

### `ip neigh` — Layer 2 ARP Check

![[Pasted image 20260418151103.png]]
**Screenshot evidence (image8):** `ip neigh` shows `192.168.1.1 dev eth0 lladdr [MAC] STALE`. The gateway's MAC address IS in the ARP table — **Layer 2 worked at some point**. The `STALE` status is because the firewall is now blocking ICMP, preventing ARP refresh. This means the network hardware path to the gateway is intact.

```bash
ip neigh
```

---

## Objective 2 — Test Connectivity (Layer 3)

### `ip route` — Verify Routing Table

![[Pasted image 20260418151219.png]]
**Screenshot evidence (image9 — top):** `ip route` shows `default via 192.168.1.1 dev eth0` and `192.168.1.0/24 dev eth0`. The default gateway `192.168.1.1` is correctly configured — **Layer 3 routing configuration is correct**.

```bash
ip route
```

### `ping` Gateway — Confirm Firewall is Blocking

**Screenshot evidence (image9 — bottom):** `ping 192.168.1.1` results in `18 packets transmitted, 0 received, 100% packet loss` — the gateway is unreachable despite being correctly configured. Combined with the correct route, this confirms the firewall is **blocking ICMP outbound** — the issue is at the firewall layer.

```bash
ping 192.168.1.1 # Ctrl+c to stop the ping
```

### `ping 8.8.8.8` — Confirm No Internet

![[Pasted image 20260418151718.png]]
**Screenshot evidence (image10):** `ping 8.8.8.8` results in `41 packets transmitted, 0 received, 100% packet loss` (highlighted in red). Confirms all outbound traffic is blocked.

```bash
ping 8.8.8.8
```

---

## Objective 3 — Test DNS and Trace Routes (Layer 7 & L3)

### `dig` — DNS Resolution Test

![[Pasted image 20260418151756.png]]
**Screenshot evidence (image11):** `dig www.google.com` outputs repeated `communications error to 192.168.1.1#53: timed out` and `no servers could be reached`. DNS queries cannot leave the host — consistent with outbound DROP.

```bash
dig www.google.com
dig www.google.com @8.8.8.8    # Query a specific DNS server
```

> 💡 **`dig` vs `nslookup`:** Both test DNS. `dig` is preferred for diagnostic work because it shows full DNS response detail (query time, server used, TTL, record type), making it easier to distinguish between "no response" and "NXDOMAIN".

### `traceroute` — Route Tracing

![[Pasted image 20260418151948.png]]
**Screenshot evidence (image12):** `traceroute 8.8.8.8` immediately fails with `send: Operation not permitted`. The firewall drops the ICMP/UDP packets before they can even leave the host — traceroute cannot send its probes.

```bash
traceroute 8.8.8.8
```

---

## Objective 4 — Identify the Firewall Misconfiguration

### `ufw status verbose`

![[Pasted image 20260418152053.png]]
**Screenshot evidence (image13):** `ufw status verbose` confirms `Default: deny (incoming), deny (outgoing)` — both directions are blocked.

```bash
ufw status verbose
```

### `iptables -L --line-numbers`

![[Pasted image 20260418152117.png]]
**Screenshot evidence (image14):** `iptables -L --line-numbers` shows the raw iptables rules generated by ufw. All three chains (`INPUT`, `FORWARD`, `OUTPUT`) have **policy DROP**. This is the underlying enforcement mechanism behind `ufw default deny`.

```bash
iptables -L --line-numbers
iptables -L -n -v    # Numeric output with packet counters
```

---

## Objective 5 — Fix the Firewall

### Restore Outbound Traffic

![[Pasted image 20260418152209.png]]
**Screenshot evidence (image15):** `ufw default allow outgoing` changes the outbound policy back to ALLOW. Output: `Default outgoing policy changed to 'allow'`.

```bash
ufw default allow outgoing
```

### Verify the Fix

![[Pasted image 20260418152300.png]]
**Screenshot evidence (image16):** All four verification checks pass:
1. `ping 8.8.8.8` — `4 packets transmitted, 4 received, 0% packet loss` ✅ — internet is restored
2. `sudo apt update` — `Fetched 73.2 MB in 19s` ✅ — package manager works
3. `ping 192.168.1.1` — `3 packets transmitted, 3 received, 0% packet loss` ✅ — gateway now reachable
4. `ip neigh` — `192.168.1.1 dev eth0 ... REACHABLE` ✅ — ARP entry refreshed (no longer STALE)

![[Pasted image 20260418152344.png]]
**Screenshot evidence (image17):** Browser successfully loads `www.wikipedia.org` ✅

![[Pasted image 20260418152402.png]]
**Screenshot evidence (image18):** `iptables -L --line-numbers` now shows `Chain OUTPUT (policy ACCEPT)` — confirming the iptables policy has changed from DROP to ACCEPT.

---

## Q&A — Additional Questions

### What rule allows outbound HTTP/HTTPS traffic specifically?

![[Pasted image 20260418152455.png]]
**Screenshot evidence (image20):** `ufw allow out 80/tcp && ufw allow out 443/tcp` adds specific port-based allow rules. `ufw status verbose` then shows:
- `80/tcp` — `ALLOW OUT Anywhere`
- `443/tcp` — `ALLOW OUT Anywhere` (both IPv4 and IPv6)

```bash
ufw allow out 80/tcp
ufw allow out 443/tcp
ufw status verbose    # Verify the rules are listed
```

### Working DNS after fix

![[Pasted image 20260418152536.png]]
**Screenshot evidence (image19):** `dig www.google.com` now returns a full answer section with 6 A records for `www.google.com`, query time 12ms, and server `192.168.1.1#53` — DNS is fully functional.

---

## Tools & Alternatives

| Tool Used | Purpose | Alternative |
|---|---|---|
| `ifconfig` | View interface status | `ip addr` (modern replacement) |
| `ip addr` | View interface + IP info | `ifconfig` (legacy but widely known) |
| `ip neigh` | View ARP table | `arp -n` (legacy) |
| `ip route` | View routing table | `route -n` (legacy), `netstat -r` |
| `ping` | ICMP reachability test | `fping` (batch ping multiple hosts) |
| `traceroute` | Trace routing hops | `mtr` (live combining ping + traceroute) |
| `dig` | DNS resolution test | `nslookup` (simpler), `host` |
| `ufw` | Firewall management | `firewalld` (RHEL/CentOS), raw `iptables` |
| `iptables -L` | View raw firewall rules | `nft list ruleset` (nftables) |

---

## Cheatsheet — Key Commands

```bash
# --- Interface Inspection ---
ifconfig                              # View all interfaces (legacy)
ip addr                               # View all interfaces (modern)
ip addr show eth0                     # Specific interface

# --- Layer 2 (ARP) ---
ip neigh                              # View ARP table (STALE/REACHABLE/FAILED)
arp -n                                # Legacy ARP table view

# --- Layer 3 (Routing) ---
ip route                              # View routing table
ping 192.168.1.1                      # Test gateway reachability
ping 8.8.8.8                          # Test internet reachability (bypasses DNS)

# --- DNS (Layer 7) ---
dig www.google.com                    # Full DNS query output
dig www.google.com @8.8.8.8          # Query specific DNS server
nslookup www.google.com              # Simple DNS test

# --- Route Tracing ---
traceroute 8.8.8.8                    # Trace route (requires outbound ICMP/UDP)
mtr 8.8.8.8                          # Live traceroute + ping combined

# --- ufw Firewall ---
ufw enable                            # Enable firewall
ufw status verbose                    # View current rules and defaults
ufw default deny incoming             # Block all inbound by default
ufw default deny outgoing             # Block all outbound by default
ufw default allow outgoing            # Allow all outbound (restore)
ufw allow out 443/tcp                 # Allow outbound HTTPS specifically
ufw allow out 80/tcp                  # Allow outbound HTTP specifically
ufw allow out 53/udp                  # Allow outbound DNS specifically

# --- iptables (raw rules behind ufw) ---
iptables -L --line-numbers            # List all rules with line numbers
iptables -L -n -v                     # Numeric output with packet counters
iptables -L OUTPUT                    # View only OUTPUT chain

# --- Package Management (verify fix) ---
sudo apt update                       # Test if outbound HTTP works
```

---

## Summary

| Objective | Result |
|---|---|
| Simulate ufw misconfiguration | ✅ `ufw default deny incoming && deny outgoing` |
| Layer 1 — Interface UP? | ✅ `ifconfig` / `ip addr` — eth0 UP, 192.168.1.14 assigned |
| Layer 2 — ARP resolved? | ✅ `ip neigh` — gateway MAC in table (STALE due to firewall) |
| Layer 3 — Route correct? | ✅ `ip route` — default via 192.168.1.1 correct |
| Layer 3 — ping fails? | ✅ 100% packet loss to 192.168.1.1 and 8.8.8.8 |
| Layer 7 — DNS broken? | ✅ `dig` — no servers could be reached |
| `traceroute` blocked? | ✅ `Operation not permitted` |
| Identify root cause | ✅ `ufw default deny outgoing` → `iptables OUTPUT policy DROP` |
| Fix: restore outbound | ✅ `ufw default allow outgoing` |
| Verify: ping, apt, browser | ✅ All working, `ip neigh` shows REACHABLE |
| Specific port allow rules | ✅ `ufw allow out 80/tcp && 443/tcp` |
| Working DNS confirmed | ✅ `dig www.google.com` returns A records |
