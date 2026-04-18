# 🐧 Linux CLI Capstone Projects

> A series of hands-on labs practising Linux system administration and security tasks using **Bash and CLI tools exclusively** — no GUI allowed.
>
> These labs simulate real-world DevOps and junior sysadmin scenarios at a fictional company, **NovaStack Solutions**, and cover file management, disk monitoring, multi-user access control, network troubleshooting, and Bash scripting automation.

---

## 📋 Lab Overview

| # | Lab | Key Skills |
|---|-----|-----------|
| 01 | <a href="https://github.com/Kazu010101/Linux-CLI-Capstone-Project/blob/main/Linux_Lab_01_Organise_and_Maintain_a_Project_Workspace.md">Organise and Maintain a Project Workspace</a> | File/directory management, Linux paths |
| 02 | <a href="https://github.com/Kazu010101/Linux-CLI-Capstone-Project/blob/main/Linux_Lab_02_Investigate_and_Respond_to_a_Disk_Space_Alert.md">Investigate and Respond to a Disk Space Alert| Disk monitoring, archiving, cron |
| 03 | <a href="https://github.com/Kazu010101/Linux-CLI-Capstone-Project/blob/main/Linux_Lab_03_Secure_Multi-User_Collaboration_Environment.md">Secure Multi User Collaboration Environment | chmod, SGID, group-based access control |
| 04 | <a href="https://github.com/Kazu010101/Linux-CLI-Capstone-Project/blob/main/Linux_Lab_04_Network_Diagnostics_and_Firewall_Recovery.md">Network Diagnostics and Firewall Recovery | OSI triage, ufw, iptables |
| 05 | [Bash Scripting for Automation](#-lab-05--bash-scripting-for-automation) | Bash scripting, error handling, crontab | 
📎 **[Linux CLI Cheatsheet](./Linux_CLI_Cheatsheet.md)** — consolidated command reference across all labs

---

## 📁 Lab 01 – Organise and Maintain a Project Workspace

**Scenario:** Set up and maintain a structured project workspace for the software engineering team at NovaStack, using only Linux CLI file management commands.

**What I did:**
- Created a new `devops` user and switched to that account using `su -`
- Built a multi-level workspace directory structure using `mkdir -p`
- Created placeholder files with `touch`, copied files with `cp`, moved files with `mv`
- Renamed a file using `mv` (Linux's dual-purpose move/rename command)
- Safely deleted a file with `rm` and verified the result with `ls` and `tree`
- Used wildcards (`*.md`, `*.json`) to copy multiple file types in one command

**Security concepts applied:** Principle of least privilege (working as `devops`, not root), understanding `rm` permanence vs. Trash-based deletion, absolute vs. relative path safety

**Tools:** `adduser`, `su`, `mkdir`, `touch`, `cp`, `mv`, `rm`, `ls`, `tree`

👉 **[View Full Lab Notes](./Linux_Lab_01_Organise_and_Maintain_a_Project_Workspace.md)**

---

## 💾 Lab 02 – Investigate and Respond to a Disk Space Alert

**Scenario:** A disk space alert has fired. Identify the largest directories, archive large files to free space, and set up an automated daily disk usage report.

**What I did:**
- Simulated large files using `fallocate` (instant allocation) and `dd` (writes real random data)
- Created a simulated `/var/logs` structure with `.log` and `.old` files
- Used `df -h` to identify the partition at 96% capacity and `du --max-depth | sort -hr | head -n 3` to find the top consuming directories
- Used `find` with `-size` and `-mtime` flags to locate large old files
- Archived large files with `tar -czvf` and used the advanced `find -print0 | tar --null` pipeline for scripted archiving
- Excluded active log files from archives using `tar --exclude`
- Set up a cron job (`0 8 * * * df -h >> sample.log`) and verified it worked by temporarily changing to `* * * * *`

**Security concepts applied:** Log retention policy, `-print0`/`--null` pairing for safe filename handling, active file exclusion before archiving

**Tools:** `df`, `du`, `find`, `fallocate`, `dd`, `tar`, `crontab`, `systemctl`

👉 **[View Full Lab Notes](./Linux_Lab_02_Investigate_and_Respond_to_a_Disk_Space_Alert.md)**

---

## 🔐 Lab 03 – Secure Multi-User Collaboration Environment

**Scenario:** Configure a shared `/opt/shared/projects/` directory so the frontend team (alice, bob) and backend team (carol) each have full access to their own subdirectory but are completely blocked from the other's.

**What I did:**
- Created three users (`alice`, `bob`, `carol`) and two groups (`frontend`, `backend`)
- Used `usermod -aG` with the `-a` (append) flag to add users to groups without removing existing memberships
- Changed directory group ownership with `chgrp`
- Applied `chmod 770` — full access for owner and group, zero access for others
- Applied `chmod 2770` — SGID bit ensures new files inherit the directory's group, not the creator's primary group
- Verified all six access combinations using `su - user` to switch identities
- Confirmed SGID works: bob can write into alice's file because it belongs to group `frontend`
- Used `getfacl` to inspect detailed ownership and permission information

**Security concepts applied:** RBAC with Linux groups, SGID for collaborative directories, principle of least privilege, testing access restrictions systematically

**Tools:** `adduser`, `groupadd`, `usermod -aG`, `chgrp`, `chmod`, `su`, `getfacl`

👉 **[View Full Lab Notes](./Linux_Lab_03_Secure_Multi-User_Collaboration_Environment.md)**

---

## 🌐 Lab 04 – Network Diagnostics and Firewall Recovery

**Scenario:** A staging server cannot reach the internet. Troubleshoot the problem layer-by-layer and fix the firewall misconfiguration that is blocking all outbound traffic.

**What I did:**
- Simulated the fault: `ufw default deny outgoing` blocked all outbound traffic
- Diagnosed layer-by-layer using the OSI model:
  - L1/L2: `ifconfig`, `ip addr`, `ip neigh` — interface UP, ARP entry STALE (not FAILED)
  - L3: `ip route` — gateway configured correctly; `ping 192.168.1.1` — 100% packet loss (firewall confirmed)
  - L7: `dig www.google.com` — DNS timeouts; `traceroute` — `Operation not permitted`
- Identified root cause using `ufw status verbose` (deny outgoing) and confirmed with `iptables -L` (OUTPUT policy DROP)
- Fixed with `ufw default allow outgoing`
- Verified fix: ping, `apt update`, browser, and `ip neigh` (STALE → REACHABLE) all confirmed working
- Added specific port-based allow rules for HTTP/HTTPS (`ufw allow out 80/tcp` and `443/tcp`)

**Security concepts applied:** OSI layer-by-layer triage methodology, ufw/iptables relationship, Principle of Least Privilege for egress filtering

**Tools:** `ifconfig`, `ip addr`, `ip neigh`, `ip route`, `ping`, `dig`, `traceroute`, `ufw`, `iptables`

👉 **[View Full Lab Notes](./Linux_Lab_04_Network_Diagnostics_and_Firewall_Recovery.md)**

---

## 🤖 Lab 05 – Bash Scripting for Automation

**Scenario:** Write a Bash script to automate daily backups of the documentation folder, name archives with the current date, log all operations, and schedule it via cron.

**What I did:**
- Created `backup.sh` with a proper `#!/bin/bash` shebang and modular variable definitions
- Used `$(date +%F)` to embed the current date in the archive filename (`backup-2026-03-01.tar.gz`)
- Used `[ -d "$SOURCE" ]` to check if the source directory exists before attempting backup
- Ran `tar -czvf` with `>> $LOGFILE 2>&1` to capture both stdout and stderr into the log
- Used `$?` to check the exit code of `tar` and branch on success or failure
- Formatted a human-readable summary line using `echo | awk`
- Made the script executable with `chmod +x` and scheduled it with `sudo crontab -e`
- Scheduled daily execution at 9AM: `0 9 * * * /home/kazukali/backup.sh`

**Security concepts applied:** Exit code-driven error handling, stdout/stderr log capture (`2>&1`), `&&` for safe command chaining, run-as-root cron for privileged operations

**Tools:** `bash`, `touch`, `nano`, `chmod`, `tar`, `date`, `awk`, `echo`, `crontab`

👉 **[View Full Lab Notes](./Linux_Lab_05_Bash_Scripting_for_Automation.md)**

---

## 🔧 Tools & Technologies Used

![Linux](https://img.shields.io/badge/Linux-FCC624?style=flat&logo=linux&logoColor=black)
![Bash](https://img.shields.io/badge/Bash-4EAA25?style=flat&logo=gnubash&logoColor=white)
![Kali](https://img.shields.io/badge/Kali_Linux-557C94?style=flat&logo=kalilinux&logoColor=white)

| Category | Tools |
|---|---|
| File Management | `mkdir`, `cp`, `mv`, `rm`, `touch`, `tree` |
| Disk Monitoring | `df`, `du`, `find`, `fallocate`, `dd` |
| Archiving | `tar`, `gzip` |
| Permissions | `chmod`, `chgrp`, `chown`, `getfacl` |
| User Management | `adduser`, `groupadd`, `usermod`, `su` |
| Networking | `ip`, `ifconfig`, `ping`, `dig`, `traceroute` |
| Firewall | `ufw`, `iptables` |
| Automation | Bash scripting, `crontab`, `date`, `awk` |

---

## 📎 Quick Reference

📄 **[Linux CLI Cheatsheet](./Linux_CLI_Cheatsheet.md)** — all commands from every lab in one place, organised by category

---

*← Back to [Portfolio Home](../README.md)*
