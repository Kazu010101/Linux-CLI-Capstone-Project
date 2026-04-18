---
tags:
  - cybersecurity
  - linux-cli
  - disk-management
  - system-monitoring
  - cron
  - capstone
status: complete
lab: 2
topic: Investigate and Respond to a Disk Space Alert
environment: Linux (Kali) — Bash
---

# Linux Lab 02 – Investigate and Respond to a Disk Space Alert

## Scenario

> The internal monitoring tool at **NovaStack Solutions** has triggered a disk space usage alert. Identify which directories are consuming the most space, clean up unnecessary files, archive large files, and set up an automated daily disk usage report using cron.

---

## Theory & Background

### `df` vs `du` — Which to Use When

Both tools deal with disk space but answer different questions:

| Tool | Question it answers | Output scope |
|---|---|---|
| `df` | How much space is available on each **mounted filesystem**? | Partition-level view |
| `du` | How much space is used by a **directory or file**? | File/directory-level view |

Use `df` first to identify *which partition* is filling up, then `du` to drill down into *which directories* are responsible.

**Common issue with `du` on mounted filesystems:** `du ~` without `--max-depth` will recursively traverse into every subdirectory including mounted filesystems, producing hundreds of lines of output that are hard to read. Use `--max-depth=1` or `--max-depth=2` to limit scope, and pipe to `sort -hr` to rank by size.

### `fallocate` vs `dd` — Simulating Large Files

| Tool | Method | Speed | Use case |
|---|---|---|---|
| `fallocate` | Allocates disk space without writing data | Instant | Quick size simulation, filesystem tests |
| `dd` | Writes actual data block by block | Slow | Realistic data simulation, testing throughput |

### Cron Syntax Reference

```
# m  h  dom mon dow  command
# │  │   │   │   │
# │  │   │   │   └── Day of Week (0=Sun, 7=Sun)
# │  │   │   └────── Month (1–12)
# │  │   └────────── Day of Month (1–31)
# │  └────────────── Hour (0–23)
# └────────────────── Minute (0–59)
  0  8   *   *   *   command    # Every day at 8:00 AM
  *  *   *   *   *   command    # Every minute
```

### `find` + `tar` — The `-print0` / `--null` Pairing

When piping `find` output to `tar`, a critical pairing is required:

- `find -print0` separates results with a **null character `\0`** instead of a newline `\n`
- `tar --null` tells tar to read filenames separated by **null characters**

These two flags **must be used together**. If `-print0` is omitted, `find` uses `\n` as separator, but `tar --null` expects `\0` — causing a mismatch and "Cannot stat" errors (demonstrated in image18 below).

---

## Setup — Simulate Large Files

<img width="414" height="54" alt="image" src="https://github.com/user-attachments/assets/3ff9716f-25b5-4354-8260-fcc9c4543681" />

**Screenshot evidence (image1):** `fallocate -l 100M large_files.txt` instantly allocates a 100MB file.

```bash
fallocate -l 100M large_files.txt
```

<img width="665" height="91" alt="image" src="https://github.com/user-attachments/assets/46cb7347-161b-48c5-ae13-7ac8f35d5998" />

**Screenshot evidence (image2):** `ls -lh` confirms `large_files.txt` at 100M.

```bash
ls -lh
```

<img width="673" height="239" alt="image" src="https://github.com/user-attachments/assets/1610361c-62c1-49ba-aa00-fceb0373139d" />

**Screenshot evidence (image3):** `dd if=/dev/urandom of=big_files.txt bs=1M count=100` writes 100MB of random data. Output confirms `104857600 bytes (105 MB) copied`. `ls -lh` now shows both files totalling 201M.

```bash
dd if=/dev/urandom of=big_files.txt bs=1M count=100
ls -lh
```

| `dd` Flag | Meaning |
|---|---|
| `if=/dev/urandom` | Input file — read from random data generator |
| `of=big_files.txt` | Output file |
| `bs=1M` | Block size — write 1MB per block |
| `count=100` | Write 100 blocks (= 100MB total) |

---

## Setup — Create Simulated `/var/logs` Structure

<img width="595" height="389" alt="image" src="https://github.com/user-attachments/assets/7daadb66-2085-4dc2-9273-14e1f0a03845" />

**Screenshot evidence (image4):** A simulated log directory structure was created and populated with placeholder files.

```bash
mkdir simulated_var && cd simulated_var && mkdir logs
cd logs
touch sample.log sample.old
ls    # Confirms: sample.log  sample.old
```

---

## Objective 1 — Inspect Disk Space with `df -h`

<img width="1011" height="294" alt="image" src="https://github.com/user-attachments/assets/8515cdb0-f9ea-4445-a305-29212ad77bff" />

**Screenshot evidence (image5):** `df -h` output shows all mounted filesystems. Key finding: `/dev/sda1` is **33G total, 30G used, 1.5G available — 96% full**. This is the partition triggering the alert.

```bash
df -h
```

The `-h` flag makes sizes human-readable (KB, MB, GB instead of raw bytes).

---

## Objective 2 — Identify Top Disk-Consuming Directories

<img width="446" height="396" alt="image" src="https://github.com/user-attachments/assets/b3eff721-8f00-46ac-8911-2360d67167b6" />

**Screenshot evidence (image6):** `sudo du -h --max-depth=1 ~ | sort -hr` shows all home subdirectories ranked by size. `/home/kazukali/capstone_2` is highlighted in red at **201M** — the largest, confirming it contains the simulated large files.

```bash
sudo du -h --max-depth=1 ~ | sort -hr
```

<img width="535" height="116" alt="image" src="https://github.com/user-attachments/assets/43510bb6-363e-47d7-a8a4-2512e89413c5" />

**Screenshot evidence (image16):** To find the top 3, pipe to `head -n 3`:

```bash
du -h --max-depth 2 ~ | sort -hr | head -n 3
# Output:
# 304M  /home/kazukali
# 301M  /home/kazukali/capstone_2
# 3.0M  /home/kazukali/.cache
```

<img width="619" height="790" alt="image" src="https://github.com/user-attachments/assets/bc3b1b83-105e-47e5-baf7-d50575c79fc5" />

**Screenshot evidence (image20 -top & image21 -below):** `du -h ~` without `--max-depth` produces an overwhelming list traversing every subdirectory and cache. In contrast, `df -h ~` shows only the relevant partition (`/dev/sda1` at 96%) — far more readable for a quick health check.
<img width="521" height="96" alt="image" src="https://github.com/user-attachments/assets/3f0aec55-d87d-45e6-85cc-77810c60d7e4" />



---

## Objective 3 — Find Files Over 10MB and Older than 30 Days

<img width="446" height="51" alt="image" src="https://github.com/user-attachments/assets/5c4ca402-6d92-43b9-8a0e-6767f0e3d31d" />

**Screenshot evidence (image7):** `find ~ -type f -size +10M -mtime +30` returns no results — the Kali VM was created less than 30 days ago, so no files meet the age threshold.

```bash
find ~ -type f -size +10M -mtime +30
```
<img width="460" height="96" alt="image" src="https://github.com/user-attachments/assets/6b3b900b-0c3f-4b20-9ada-bf6a9515e7d4" />

**Screenshot evidence (image8):** Removing `-mtime +30` to find all files over 10MB:

```bash
find ~ -type f -size +10M
# Output:
# /home/kazukali/capstone_2/large_files.txt
# /home/kazukali/capstone_2/big_files.txt
```

| `find` Flag | Meaning |
|---|---|
| `-type f` | Files only (exclude directories) |
| `-size +10M` | Larger than 10 megabytes |
| `-mtime +30` | Last modified more than 30 days ago |

---

## Objective 4 — Archive and Compress Large Files

### Basic tar Archive

<img width="845" height="120" alt="image" src="https://github.com/user-attachments/assets/3135d6a4-7634-43b5-ae92-3ef72f4ff47f" />

**Screenshot evidence (image9):** `tar -czvf big_largefiles.tar.gz big_files.txt large_files.txt` creates a compressed archive of both large files.

```bash
tar -czvf big_largefiles.tar.gz big_files.txt large_files.txt
```

<img width="730" height="151" alt="image" src="https://github.com/user-attachments/assets/c8c842f4-7efb-4062-9f9b-15065a64e684" />

**Screenshot evidence (image10):** `ls -lh` confirms `big_largefiles.tar.gz` at 101M exists alongside the original files.

| `tar` Flag | Meaning |
|---|---|
| `-c` | Create a new archive |
| `-z` | Compress with gzip |
| `-v` | Verbose — list files being processed |
| `-f` | Specify the archive filename |

### Advanced — Combine `find` + `tar` in One Pipeline

<img width="960" height="234" alt="image" src="https://github.com/user-attachments/assets/12380514-30ce-45d5-bd57-04c05ca1f397" />

**Screenshot evidence (image17):** The correct combined command uses `-print0` and `--null` together:


```bash
find ~ -type f -name "*.txt" -size +10M -print0 | tar -czvf combined.tar.gz --null -T -
```

| Flag | Command | Meaning |
|---|---|---|
| `-print0` | `find` | Separate output with null `\0` instead of newline |
| `--null` | `tar` | Read filenames separated by null `\0` |
| `-T -` | `tar` | Read file list from stdin (the pipe) |

<img width="1090" height="139" alt="image" src="https://github.com/user-attachments/assets/ab14585f-dd65-43fb-9d39-59a59259dfa5" />

**Screenshot evidence (image18):** The error case — running without `-print0` causes `tar` to fail with `Cannot stat: No such file or directory`. The filenames include `\n` in the path because `find` used newline separation, which `tar --null` cannot parse.

### Excluding Active Log Files from Archive

<img width="778" height="159" alt="image" src="https://github.com/user-attachments/assets/8576229e-0f62-43bb-8c15-d7484ed75e20" />

**Screenshot evidence (image19):** `tar --exclude="sample.log" -czvf exclude_active_log.tar.gz sample.old` archives only `sample.old` while leaving the active `sample.log` untouched. The subsequent `ls` confirms both the archive and `sample.log` remain.

```bash
tar --exclude="sample.log" -czvf exclude_active_log.tar.gz sample.old
```

> 💡 Always identify active log files before archiving. Archiving a log file that a running service is writing to can corrupt both the archive and the service's output.

---

## Objective 5 — Cron Job for Daily Disk Reports

### Open crontab

<img width="623" height="164" alt="image" src="https://github.com/user-attachments/assets/f96644ea-1d93-4a68-916e-0cc796880948" />

**Screenshot evidence (image11):** `crontab -e` is run. Since no crontab exists yet, it prompts for an editor. Option 1 (`/bin/nano`) is selected as the easiest.

```bash
crontab -e
```

### Add the Cron Entry

<img width="814" height="514" alt="image" src="https://github.com/user-attachments/assets/aa79c65b-76bd-4e59-9053-6dc94a668978" />

**Screenshot evidence (image12):** Inside the nano crontab editor, the following line is added at the bottom:

```bash
0 8 * * * df -h >> /home/kazukali/capstone_2/simulated_var/logs/sample.log
```

This means: at minute 0 of hour 8 (8:00 AM), every day, every month, every day of week — run `df -h` and **append** the output to `sample.log`.

### Test the Cron Job Without Waiting Until 8AM

<img width="810" height="44" alt="image" src="https://github.com/user-attachments/assets/c46e737d-43a2-4cd0-9c7b-353e04679620" />

**Screenshot evidence (image14):** To verify without waiting, the crontab was temporarily changed to run every minute:

```bash
* * * * * df -h >> /home/kazukali/capstone_2/simulated_var/logs/sample.log
```

<img width="1070" height="435" alt="image" src="https://github.com/user-attachments/assets/c34680c2-0141-4f9e-9f05-512d71757a42" />

**Screenshot evidence (image13):** `sudo systemctl restart cron` then `systemctl status cron` confirms cron is `active (running)` — essential to verify before relying on scheduled jobs.

```bash
sudo systemctl restart cron
systemctl status cron
```

<img width="839" height="239" alt="image" src="https://github.com/user-attachments/assets/4e8cbd86-6c5a-48da-8082-f56c178e0ce5" />

**Screenshot evidence (image15):** After one minute, `cat /home/kazukali/capstone_2/simulated_var/logs/sample.log` shows a full `df -h` output appended to the file — confirming the cron job ran successfully.

```bash
cat /home/kazukali/capstone_2/simulated_var/logs/sample.log
```

**Final daily crontab entry:**
```bash
0 8 * * * df -h >> /home/kazukali/capstone_2/simulated_var/logs/sample.log
```

---

## Tools & Alternatives

| Tool Used | Purpose | Alternative |
|---|---|---|
| `df -h` | View filesystem disk usage | `lsblk`, `fdisk -l` |
| `du -h --max-depth` | View directory sizes | `ncdu` (interactive, much friendlier) |
| `find` | Locate files by size/age/name | `fd` (faster modern alternative) |
| `fallocate` | Quickly allocate disk space | `truncate -s 100M file` |
| `dd` | Write data block-by-block | `pv` (with progress bar) |
| `tar -czvf` | Archive + compress | `zip`, `7z`, `zstd` (faster compression) |
| `crontab -e` | Schedule recurring tasks | `systemd timers` (more robust on modern Linux) |

---

## Cheatsheet — Key Commands

```bash
# --- Disk Space ---
df -h                                          # Filesystem usage (human-readable)
df -h ~                                        # Usage for the partition containing home
du -h --max-depth=1 ~                          # Directory sizes, 1 level deep
du -h --max-depth=2 ~ | sort -hr | head -n 3  # Top 3 largest directories

# --- Finding Large/Old Files ---
find ~ -type f -size +10M                      # Files over 10MB
find ~ -type f -mtime +30                      # Files older than 30 days
find ~ -type f -size +10M -mtime +30           # Both conditions

# --- Creating Test Files ---
fallocate -l 100M filename.txt                 # Allocate 100MB instantly
dd if=/dev/urandom of=filename.txt bs=1M count=100  # Write 100MB of random data

# --- Archiving ---
tar -czvf archive.tar.gz file1 file2           # Create compressed archive
tar -czvf archive.tar.gz /path/to/dir/         # Archive a directory
tar -tzf archive.tar.gz                        # List contents without extracting
tar -xzvf archive.tar.gz                       # Extract archive
tar --exclude="active.log" -czvf out.tar.gz .  # Archive, excluding a file

# --- find + tar pipeline ---
find ~ -type f -name "*.txt" -size +10M -print0 | tar -czvf combined.tar.gz --null -T -

# --- Cron ---
crontab -e                                     # Edit crontab
crontab -l                                     # List current crontab
sudo systemctl restart cron                    # Restart cron daemon
systemctl status cron                          # Check cron status

# Cron syntax: m h dom mon dow command
0 8 * * * df -h >> /path/to/sample.log         # Daily 8AM disk report
* * * * * df -h >> /path/to/sample.log         # Every minute (for testing)
```

---

## Summary

| Objective | Result |
|---|---|
| Simulate large files | ✅ `fallocate` (100M) + `dd` (100M) |
| Inspect disk space with `df -h` | ✅ `/dev/sda1` at 96% identified |
| Find top 3 consuming directories | ✅ `du \| sort -hr \| head -n 3` |
| Find files >10MB and >30 days old | ✅ `find -type f -size +10M -mtime +30` |
| Archive large files to tar.gz | ✅ `tar -czvf` + `find -print0 \| tar --null` |
| Exclude active logs from archive | ✅ `tar --exclude="sample.log"` |
| Set up daily disk report cron job | ✅ `0 8 * * * df -h >> sample.log` |
| Test cron without waiting | ✅ Changed to `* * * * *`, verified after 1 min |
