# Linux CLI Cheatsheet – NovaStack Capstone Labs

> Consolidated command reference across all 5 Linux capstone labs. Use for CTFs, incident response, sysadmin tasks, and daily DevOps work.

---

## 👤 User & Group Management

```bash
# --- Users ---
sudo adduser username                          # Create user (interactive, Debian-style)
sudo useradd -m -s /bin/bash username         # Create user (manual, non-interactive)
su - username                                  # Switch user (full login environment)
sudo -u username bash                          # Open shell as user (uses your sudo)
whoami                                         # Show current user
id username                                    # Show UID, GID, all groups
cat /etc/passwd | grep username                # Verify user entry
cat /etc/passwd                                # View all users

# --- Groups ---
sudo groupadd groupname                        # Create a group
sudo usermod -aG groupname username            # Add user to group (APPEND — critical flag)
sudo gpasswd -a username groupname             # Alternative: add user to group
groups username                                # Show user's group memberships
cat /etc/group | grep groupname               # Verify group members
```

---

## 📁 File System & Directory Management

```bash
# --- Navigation ---
cd /absolute/path                              # Absolute path navigation
cd relative/path                               # Relative path
cd ~                                           # Home directory
cd ..                                          # One level up
pwd                                            # Print working directory

# --- Creating ---
mkdir dirname                                  # Create directory
mkdir -p path/to/nested/dir                    # Create full nested path
touch file.txt                                 # Create empty file / update timestamp
touch file1.txt file2.txt file3.txt            # Create multiple files

# --- Listing ---
ls                                             # Basic list
ls -lh                                         # Long format, human-readable sizes
ls -la                                         # Include hidden files (dotfiles)
ls -lhS                                        # Sort by size (largest first)
tree                                           # Visual directory tree

# --- Copying & Moving ---
cp source dest                                 # Copy file
cp -r sourcedir/ destdir/                      # Copy directory recursively
cp *.md *.json /dest/                          # Wildcard copy multiple types
mv source dest                                 # Move file or directory
mv oldname.txt newname.txt                     # Rename (same directory)

# --- Deleting ---
rm file.txt                                    # Delete file (permanent — no undo)
rm -i file.txt                                 # Interactive delete (asks first)
rm -rf dirname/                                # Recursive delete directory (DANGEROUS)

# --- Viewing ---
cat file.txt                                   # Print entire file
head -n 20 file.txt                            # First 20 lines
tail -n 20 file.txt                            # Last 20 lines
tail -f file.log                               # Follow a log file live
less file.txt                                  # Paginated view (q to quit)
```

---

## 🔐 Permissions & Ownership

```bash
# --- chmod (change permissions) ---
chmod 770 /path                                # rwxrwx--- (owner+group full, others none)
chmod 750 /path                                # rwxr-x--- (group read+exec, no write)
chmod 660 /path/to/file                        # rw-rw---- (no execute)
chmod 2770 /path                               # SGID + 770 (new files inherit group)
chmod +x script.sh                             # Add execute permission for all
chmod u+x script.sh                            # Add execute for owner only

# chmod numeric quick reference:
# 7 = rwx (4+2+1)
# 6 = rw- (4+2+0)
# 5 = r-x (4+0+1)
# 4 = r-- (4+0+0)
# 0 = --- (0+0+0)
# Prefix 2 = SGID, 4 = SUID, 1 = Sticky bit

# --- chown / chgrp (change ownership) ---
sudo chown user:group /path                    # Change both owner and group
sudo chown :group /path                        # Change group only
sudo chgrp groupname /path                     # Change group only (explicit)
sudo chown -R user:group /path/                # Recursive ownership change

# --- Viewing Permissions ---
ls -ld /path/to/dir                            # Directory permissions and ownership
ls -lh /path/to/dir                            # File permissions inside directory
getfacl filename                               # Detailed ACL (owner, group, perms)
stat filename                                  # Full file metadata

# --- umask ---
umask                                          # Show current umask
umask 007                                      # New files: rw-rw---- (no others)
```

---

## 💾 Disk Management

```bash
# --- Disk Space Overview ---
df -h                                          # Filesystem usage (human-readable)
df -h ~                                        # Partition containing home
df -h /                                        # Root partition

# --- Directory Sizes ---
du -h --max-depth=1 ~                          # Home subdirectory sizes, 1 level
du -h --max-depth=2 ~ | sort -hr               # 2 levels, sorted by size
du -h --max-depth=2 ~ | sort -hr | head -n 3  # Top 3 largest

# --- Finding Files ---
find ~ -type f -size +10M                      # Files over 10MB
find ~ -type f -mtime +30                      # Files older than 30 days
find ~ -type f -size +10M -mtime +30           # Both conditions
find ~ -type f -name "*.log"                   # Files by extension
find ~ -type f -name "*.txt" -print0           # Null-separated output (for tar pipe)

# --- Simulating Large Files ---
fallocate -l 100M filename.txt                 # Allocate 100MB instantly (no real data)
dd if=/dev/urandom of=filename.txt bs=1M count=100  # Write 100MB of random data

# --- Archiving ---
tar -czvf archive.tar.gz file1 file2           # Create gzip-compressed archive
tar -czvf archive.tar.gz /path/to/dir/         # Archive entire directory
tar -tzf archive.tar.gz                        # List contents (verify without extracting)
tar -xzvf archive.tar.gz                       # Extract archive
tar -xzvf archive.tar.gz -C /dest/             # Extract to specific directory
tar --exclude="active.log" -czvf out.tar.gz .  # Archive, excluding a file

# --- find + tar pipeline (must pair -print0 with --null) ---
find ~ -type f -name "*.txt" -size +10M -print0 | tar -czvf combined.tar.gz --null -T -
```

---

## 🌐 Network Diagnostics

```bash
# --- Interface Inspection ---
ifconfig                                       # View all interfaces (legacy)
ip addr                                        # View all interfaces (modern)
ip addr show eth0                              # Specific interface only

# --- ARP / Layer 2 ---
ip neigh                                       # ARP table (STALE/REACHABLE/FAILED)
arp -n                                         # Legacy ARP view

# --- Routing / Layer 3 ---
ip route                                       # View routing table
ping 192.168.1.1                               # Test gateway reachability (ICMP)
ping 8.8.8.8                                   # Test internet (bypasses DNS)
ping -c 4 8.8.8.8                              # Send exactly 4 packets

# --- DNS ---
dig www.google.com                             # Full DNS query output
dig www.google.com @8.8.8.8                   # Query specific DNS server
nslookup www.google.com                        # Simple DNS test

# --- Route Tracing ---
traceroute 8.8.8.8                             # Trace route hops
mtr 8.8.8.8                                   # Live traceroute + ping combined

# --- Connections ---
netstat -an                                    # All connections, numeric (legacy)
ss -tunap                                      # Modern: TCP/UDP connections + process

# --- Firewall (ufw) ---
sudo ufw enable                                # Enable firewall
sudo ufw status verbose                        # View rules and defaults
sudo ufw default deny incoming                 # Block all inbound by default
sudo ufw default deny outgoing                 # Block all outbound by default
sudo ufw default allow outgoing               # Restore outbound
sudo ufw allow out 443/tcp                     # Allow outbound HTTPS
sudo ufw allow out 80/tcp                      # Allow outbound HTTP
sudo ufw allow out 53/udp                      # Allow outbound DNS
sudo ufw allow in 22/tcp                       # Allow inbound SSH

# --- iptables (raw rules) ---
sudo iptables -L --line-numbers                # List all rules with line numbers
sudo iptables -L -n -v                         # Numeric with packet counters
sudo iptables -L OUTPUT                        # View OUTPUT chain only

# --- Package Manager (verify fix) ---
sudo apt update                                # Test outbound HTTP to apt repos
```

---

## 📜 Bash Scripting Patterns

### Script Template

```bash
#!/bin/bash

# Variables
SOURCE="/home/devops/docs"
DEST="/mnt/backups"
DATE=$(date +%F)
FILENAME="backup-$DATE.tar.gz"
LOGFILE="/home/kazukali/backup.log"

# Log start
echo "[$(date)] Script Started." >> $LOGFILE

# Check directory exists
if [ -d "$SOURCE" ]; then
    echo "Source exists." >> $LOGFILE
    # ... do work ...
else
    echo "Error: Source not found." >> $LOGFILE
fi
```

### Conditionals

```bash
if [ -d "$DIR" ]; then ... fi              # Directory exists?
if [ -f "$FILE" ]; then ... fi             # File exists?
if [ -e "$PATH" ]; then ... fi             # Exists (any type)?
if [ $? -eq 0 ]; then ... fi              # Last command succeeded?
if [ "$VAR" = "value" ]; then ... fi      # String equals?
if [ "$NUM" -gt 80 ]; then ... fi         # Number greater than?
```

### Loops

```bash
for i in $(seq 1 10); do echo $i; done    # For loop (counter)
for file in *.log; do echo $file; done    # For loop (glob)
while true; do command; sleep 60; done    # Infinite loop (Ctrl+C to stop)
```

### Output Redirection

```bash
command > file.log                         # Overwrite stdout
command >> file.log                        # Append stdout
command 2>&1 >> file.log                  # Append stdout + stderr
command | tee -a file.log                 # Print to terminal AND append to file
command 2>/dev/null                       # Discard stderr
```

### Date Formatting

```bash
date +%F                                   # YYYY-MM-DD
date +%Y%m%d                               # YYYYMMDD
date +%F_%H-%M                             # YYYY-MM-DD_HH-MM
echo "[$(date)]"                           # Full date string
```

### Error Handling with Exit Codes

```bash
some_command
if [ $? -eq 0 ]; then
    echo "Success" >> $LOGFILE
else
    echo "Failed" >> $LOGFILE
fi

# Or with &&  / ||
some_command && echo "Success" || echo "Failed"
```

### awk Formatting

```bash
echo "a b" | awk '{print $1}'                        # First field
echo "a b" | awk '{print "X:", $1, "Y:", $2}'        # Labelled output
du -h --max-depth=2 ~ | sort -hr | head -n 3         # Real-world awk pipeline
```

---

## ⏰ Cron Scheduling

```bash
# Edit crontab
crontab -e                                 # Edit current user's crontab
sudo crontab -e                            # Edit root's crontab
crontab -l                                 # List current entries

# Cron syntax
# m  h  dom mon dow  command
  0  8  *   *   *    df -h >> /path/sample.log   # Daily 8AM disk report
  0  9  *   *   *    /home/kazukali/backup.sh     # Daily 9AM backup
  *  *  *   *   *    command                       # Every minute (for testing)

# Cron daemon
sudo systemctl restart cron                # Restart cron
systemctl status cron                      # Check cron is running
```

---

## 🧪 Useful Exploration Commands

```bash
# Who & Where
whoami                                     # Current user
id                                         # UID, GID, groups
hostname                                   # Machine hostname
uname -a                                   # Kernel + OS info

# Processes
ps aux                                     # All running processes
ps aux | grep processname                  # Filter processes
kill -9 PID                               # Force kill a process
top / htop                                 # Live process monitor

# System
systemctl status servicename              # Check service status
systemctl restart servicename             # Restart a service
journalctl -u servicename                  # View service logs

# Text Processing
grep "pattern" file.txt                    # Search in file
grep -r "pattern" /path/                   # Recursive search
sort file.txt                              # Sort lines alphabetically
sort -hr                                   # Sort human-readable sizes (reverse)
head -n 3                                  # First 3 lines
tail -n 3                                  # Last 3 lines
wc -l file.txt                             # Count lines
```

---

## 🗂️ Lab Index

| Lab | Topic | Key Tools |
|---|---|---|
| [[Linux_Lab_01_Organise_and_Maintain_a_Project_Workspace]] | File & directory management | `mkdir`, `cp`, `mv`, `rm`, `touch`, `tree` |
| [[Linux_Lab_02_Investigate_and_Respond_to_a_Disk_Space_Alert]] | Disk monitoring & cleanup | `df`, `du`, `find`, `tar`, `fallocate`, `cron` |
| [[Linux_Lab_03_Secure_Multi-User_Collaboration_Environment]] | Permissions & user management | `chmod`, `chgrp`, `usermod`, `groupadd`, `getfacl` |
| [[Linux_Lab_04_Network_Diagnostics_and_Firewall_Recovery]] | Network triage & firewall | `ip`, `ping`, `dig`, `traceroute`, `ufw`, `iptables` |
| [[Linux_Lab_05_Bash_Scripting_for_Automation]] | Bash scripting & cron | `bash`, `tar`, `date`, `awk`, `crontab`, `chmod +x` |
