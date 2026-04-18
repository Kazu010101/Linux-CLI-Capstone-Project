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

<img width="665" height="214" alt="image" src="https://github.com/user-attachments/assets/ea572a18-af5a-4717-8f75-96ab7a9b25e1" />

**Screenshot evidence (image2):** `ufw enable` as root confirms `Firewall is active and enabled on system startup`. Default state: `deny (incoming), allow (outgoing)` — outbound works before the misconfiguration.

```bash
sudo su -           # Switch to root for ufw management
ufw enable
ufw status verbose  # Confirm: deny incoming, allow outgoing
```

### Step 2: Apply the Misconfiguration

<img width="663" height="160" alt="image" src="https://github.com/user-attachments/assets/33ec9ec0-e893-4da9-bed2-abe074d8b579" />

**Screenshot evidence (image3):** `ufw default deny incoming && ufw default deny outgoing` changes both default policies to DENY.

```bash
ufw default deny incoming && ufw default deny outgoing
```

<img width="650" height="134" alt="image" src="https://github.com/user-attachments/assets/240ed6bb-ed51-494a-8b96-7b248f8487ea" />

**Screenshot evidence (image4):** `ufw status verbose` now shows `Default: deny (incoming), deny (outgoing), disabled (routed)` — confirming the fault is active.

### Step 3: Verify the Fault Symptoms

<img width="1090" height="768" alt="image" src="https://github.com/user-attachments/assets/9dad2d82-bdef-4a04-bd6a-f8ac4de41d08" />

**Screenshot evidence (image5):** Browser shows "Server Not Found" for `www.wikipedia.org` — internet is broken.

<img width="1090" height="212" alt="image" src="https://github.com/user-attachments/assets/21fffbc3-758d-442c-891a-f630c4b45ab8" />

**Screenshot evidence (image6):** `sudo apt update` fails with `Temporary failure resolving 'http.kali.org'` — package manager cannot reach repositories.

---

## Objective 1 — Inspect Interface Status (Layer 1 & 2)

### `ifconfig` — Layer 1 Check

<img width="904" height="498" alt="image" src="https://github.com/user-attachments/assets/681bc137-b06b-405a-955e-a35d87218a6c" />

**Screenshot evidence (image7):** `ifconfig` output shows `eth0` interface with flags `UP,BROADCAST,RUNNING,MULTICAST` and `inet 192.168.1.14 netmask 255.255.255.0 broadcast 192.168.1.255` (highlighted in red). The interface is UP and has a valid IP — **Layer 1 is fine**.

```bash
ifconfig
```

### `ip addr` — Alternative Interface Check

<img width="1041" height="265" alt="image" src="https://github.com/user-attachments/assets/5a0edab1-3789-46f4-b10c-05e212c3458b" />

**Screenshot evidence (image1):** `ip addr` shows `eth0` as `UP,LOWER_UP` with `inet 192.168.1.14/24 brd 192.168.1.255` (highlighted in red). Same conclusion — interface is up.

```bash
ip addr
ip addr show eth0    # Specific interface only
```

### `ip neigh` — Layer 2 ARP Check

<img width="521" height="58" alt="image" src="https://github.com/user-attachments/assets/3b475296-ac75-4f55-9581-afbb8fbf0902" />

**Screenshot evidence (image8):** `ip neigh` shows `192.168.1.1 dev eth0 lladdr [MAC] STALE`. The gateway's MAC address IS in the ARP table — **Layer 2 worked at some point**. The `STALE` status is because the firewall is now blocking ICMP, preventing ARP refresh. This means the network hardware path to the gateway is intact.

```bash
ip neigh
```

---

## Objective 2 — Test Connectivity (Layer 3)

### `ip route` — Verify Routing Table

<img width="813" height="241" alt="image" src="https://github.com/user-attachments/assets/7b6fe462-b992-41f6-9245-b1a267663c68" />

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

<img width="704" height="139" alt="image" src="https://github.com/user-attachments/assets/e9529ecc-5483-41a0-85c2-3d40bf719231" />

**Screenshot evidence (image10):** `ping 8.8.8.8` results in `41 packets transmitted, 0 received, 100% packet loss` (highlighted in red). Confirms all outbound traffic is blocked.

```bash
ping 8.8.8.8
```

---

## Objective 3 — Test DNS and Trace Routes (Layer 7 & L3)

### `dig` — DNS Resolution Test

<img width="553" height="208" alt="image" src="https://github.com/user-attachments/assets/b9d105b6-1457-4832-948f-42159596ac06" />

**Screenshot evidence (image11):** `dig www.google.com` outputs repeated `communications error to 192.168.1.1#53: timed out` and `no servers could be reached`. DNS queries cannot leave the host — consistent with outbound DROP.

```bash
dig www.google.com
dig www.google.com @8.8.8.8    # Query a specific DNS server
```

> 💡 **`dig` vs `nslookup`:** Both test DNS. `dig` is preferred for diagnostic work because it shows full DNS response detail (query time, server used, TTL, record type), making it easier to distinguish between "no response" and "NXDOMAIN".

### `traceroute` — Route Tracing

<img width="681" height="90" alt="image" src="https://github.com/user-attachments/assets/dcfd7eea-7211-4823-9e46-2463b62129e5" />

**Screenshot evidence (image12):** `traceroute 8.8.8.8` immediately fails with `send: Operation not permitted`. The firewall drops the ICMP/UDP packets before they can even leave the host — traceroute cannot send its probes.

```bash
traceroute 8.8.8.8
```

---

## Objective 4 — Identify the Firewall Misconfiguration

### `ufw status verbose`

<img width="630" height="128" alt="image" src="https://github.com/user-attachments/assets/7504a545-341a-4941-9793-384cdac0700a" />

**Screenshot evidence (image13):** `ufw status verbose` confirms `Default: deny (incoming), deny (outgoing)` — both directions are blocked.

```bash
ufw status verbose
```

### `iptables -L --line-numbers`

<img width="1090" height="807" alt="image" src="https://github.com/user-attachments/assets/02cf2e04-6596-4f04-a904-08263d6e30e8" />

**Screenshot evidence (image14):** `iptables -L --line-numbers` shows the raw iptables rules generated by ufw. All three chains (`INPUT`, `FORWARD`, `OUTPUT`) have **policy DROP**. This is the underlying enforcement mechanism behind `ufw default deny`.

```bash
iptables -L --line-numbers
iptables -L -n -v    # Numeric output with packet counters
```

---

## Objective 5 — Fix the Firewall

### Restore Outbound Traffic

<img width="459" height="90" alt="image" src="https://github.com/user-attachments/assets/fcfc9570-7dbe-48c6-86bf-1aee437f9a98" />

**Screenshot evidence (image15):** `ufw default allow outgoing` changes the outbound policy back to ALLOW. Output: `Default outgoing policy changed to 'allow'`.

```bash
ufw default allow outgoing
```

### Verify the Fix

<img width="794" height="926" alt="image" src="https://github.com/user-attachments/assets/7d987c85-b533-44c2-82ea-0d22bb62b923" />

**Screenshot evidence (image16):** All four verification checks pass:
1. `ping 8.8.8.8` — `4 packets transmitted, 4 received, 0% packet loss` ✅ — internet is restored
2. `sudo apt update` — `Fetched 73.2 MB in 19s` ✅ — package manager works
3. `ping 192.168.1.1` — `3 packets transmitted, 3 received, 0% packet loss` ✅ — gateway now reachable
4. `ip neigh` — `192.168.1.1 dev eth0 ... REACHABLE` ✅ — ARP entry refreshed (no longer STALE)

<img width="1038" height="640" alt="image" src="https://github.com/user-attachments/assets/1e318027-395a-4db7-afa1-d18b6a871b85" />

**Screenshot evidence (image17):** Browser successfully loads `www.wikipedia.org` ✅

<img width="853" height="625" alt="image" src="https://github.com/user-attachments/assets/d00b0690-8ce0-4c98-8f0e-297786ac93f1" />

**Screenshot evidence (image18):** `iptables -L --line-numbers` now shows `Chain OUTPUT (policy ACCEPT)` — confirming the iptables policy has changed from DROP to ACCEPT.

---

## Q&A — Additional Questions

### What rule allows outbound HTTP/HTTPS traffic specifically?

<img width="671" height="415" alt="image" src="https://github.com/user-attachments/assets/ba59b500-28bc-4d17-906a-0fb864fafad5" />

**Screenshot evidence (image20):** `ufw allow out 80/tcp && ufw allow out 443/tcp` adds specific port-based allow rules. `ufw status verbose` then shows:
- `80/tcp` — `ALLOW OUT Anywhere`
- `443/tcp` — `ALLOW OUT Anywhere` (both IPv4 and IPv6)

```bash
ufw allow out 80/tcp
ufw allow out 443/tcp
ufw status verbose    # Verify the rules are listed
```

### Working DNS after fix

<img width="698" height="528" alt="image" src="https://github.com/user-attachments/assets/b5a1c2fd-0019-4d6c-b477-98504b0a3a43" />

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
