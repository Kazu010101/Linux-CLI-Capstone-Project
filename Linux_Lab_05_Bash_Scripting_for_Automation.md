---
tags:
  - cybersecurity
  - linux-cli
  - bash-scripting
  - automation
  - cron
  - backup
  - capstone
status: complete
lab: 5
topic: Bash Scripting for Automation
environment: Linux (Kali) — Bash
---

# Linux Lab 05 – Bash Scripting for Automation

## Scenario

> **NovaStack's** team frequently forgets to back up their documentation folder. Write a Bash script to automate the backup process, name archives with the current date, log all operations, and schedule the script to run daily at 9AM via cron.
>
> **Source:** `/home/devops/docs/`
> **Destination:** `/mnt/backups/`
> **Archive naming:** `backup-YYYY-MM-DD.tar.gz`

---

## Theory & Background

### Bash Script Structure

Every Bash script should follow this structure:

```bash
#!/bin/bash            # Shebang — tells the OS to use bash to run this file

# Variables
VAR="value"

# Logic
if [ condition ]; then
    command
fi
```

The **shebang** (`#!/bin/bash`) on line 1 is critical — without it, the OS may use a different shell (`sh`, `dash`) that behaves differently.

### Exit Codes — `$?`

Every command in Linux returns an **exit code** when it finishes:
- `0` = success
- Non-zero = failure (the specific number indicates the type of error)

`$?` captures the exit code of the **most recently run command**. This is how scripts detect whether a step succeeded before proceeding.

```bash
tar -czvf archive.tar.gz /source
if [ $? -eq 0 ]; then
    echo "Success"
else
    echo "Failed"
fi
```

### Output Redirection — `>>` and `2>&1`

| Syntax | Meaning |
|---|---|
| `>` | Redirect stdout, overwrite file |
| `>>` | Redirect stdout, append to file |
| `2>` | Redirect stderr only |
| `2>&1` | Redirect stderr to the same destination as stdout |
| `>> file 2>&1` | Append both stdout and stderr to file |

The `2>&1` pattern is essential for capturing error messages in log files. Without it, errors print to the terminal but are not recorded in the log.

### `&&` — Conditional Command Chaining

`command1 && command2` only runs `command2` if `command1` exits with code `0` (success). This prevents partial operations — if the first step fails, the chain stops immediately rather than continuing with invalid state.

### `date +%F` — Date in Filenames

`date +%F` outputs the current date in `YYYY-MM-DD` format (ISO 8601), making it ideal for log file and backup archive naming. The format is sortable alphabetically and chronologically.

```bash
date +%F          # Output: 2026-03-01
date +%Y%m%d      # Output: 20260301 (no dashes)
date +%F_%H-%M    # Output: 2026-03-01_09-30 (with time)
```

### `awk` for Formatted Output

`awk '{print "Original:", $1, "→ Backup File:", $2}'` reads space-separated fields from a string and formats them into a readable summary. `$1` and `$2` refer to the first and second whitespace-separated tokens in the input.

---

## Setup — Verify Source and Destination

![[Pasted image 20260418153154.png]]
**Screenshot evidence (image1):** `ls` from `~/docs` shows `backup1.txt` and `backup2.txt` — the source files to be backed up.

```bash
ls ~/docs
```

![[Pasted image 20260418153206.png]]
**Screenshot evidence (image2):** `cd /mnt/backups` then `ls` shows an empty directory — the backup destination exists and is empty before the script runs.

```bash
cd /mnt/backups
ls    # Empty — no backups yet
```

---

## Step 1 — Create the Script File

![[Pasted image 20260418153227.png]]
**Screenshot evidence (image3):** `sudo touch backup.sh` creates an empty script file in the home directory.

```bash
sudo touch /home/kazukali/backup.sh
```

---

## Step 2 — Add the Shebang

![[Pasted image 20260418153312.png]]
**Screenshot evidence (image4):** nano opens `backup.sh` with `#!/bin/bash` as the first line — the shebang that declares the interpreter.

```bash
sudo nano /home/kazukali/backup.sh
# First line: #!/bin/bash
```

---

## Step 3 — Define Variables

![[Pasted image 20260418153333.png]]
**Screenshot evidence (image5):** The variable block is defined:

```bash
# Define source and destination
SOURCE="/home/devops/docs"
DEST="/mnt/backups"
DATE=$(date +%F)
FILENAME="backup-$DATE.tar.gz"
LOGFILE="/home/kazukali/backup.log"
```

**Variable breakdown:**

| Variable | Value | Purpose |
|---|---|---|
| `SOURCE` | `/home/devops/docs` | Directory to back up |
| `DEST` | `/mnt/backups` | Where to store archives |
| `DATE` | `$(date +%F)` | Today's date in YYYY-MM-DD format |
| `FILENAME` | `backup-$DATE.tar.gz` | Archive name with embedded date |
| `LOGFILE` | `/home/kazukali/backup.log` | Log file path |

> 💡 **`$(...)` — Command Substitution:** `$(date +%F)` runs the `date` command and substitutes its output inline. This is equivalent to backtick syntax `` `date +%F` `` but is preferred because it nests cleanly.

---

## Step 4 — Check if Source Directory Exists

![[Pasted image 20260418153447.png]]
**Screenshot evidence (image6 — top section, highlighted in red):** The `if` block checks for the source directory:

```bash
# Logs each operation date into a logfile
echo "[$(date)] Backup Started." >> $LOGFILE

# Check if source directory exists
if [ -d "$SOURCE" ]; then
    echo "Source directory exists." >> $LOGFILE

    # ... backup logic here ...

else
    echo "Error: Source directory does not exist." >> $LOGFILE
fi
```

| Flag | Meaning |
|---|---|
| `-d "$SOURCE"` | True if `$SOURCE` exists AND is a directory |
| `-f "$FILE"` | True if exists and is a regular file |
| `-e "$PATH"` | True if exists (file or directory) |

---

## Step 5 — Create the tar Archive

![[Pasted image 20260418153703.png]]


**Screenshot evidence (image7, highlighted in red):** Inside the `if` block, the `tar` command archives and compresses the source directory:

```bash
# Create the archive
tar -czvf "$DEST/$FILENAME" "$SOURCE" >> $LOGFILE 2>&1
```

| Part | Meaning |
|---|---|
| `tar -czvf` | Create, gzip-compress, verbose, specify filename |
| `"$DEST/$FILENAME"` | Full output path: `/mnt/backups/backup-2026-03-01.tar.gz` |
| `"$SOURCE"` | The directory to archive |
| `>> $LOGFILE 2>&1` | Append both stdout (file list) AND stderr (errors) to log |

---

## Step 6 — Check tar Success and Log Result

![[Pasted image 20260418153913.png]]
**Screenshot evidence (image8, highlighted in red):** The `$?` check immediately after `tar` determines success or failure:

```bash
# Check if tar command was successful
if [ $? -eq 0 ]; then
    echo "Backup successful!" >> $LOGFILE
else
    echo "Error: Backup failed during archive creation." >> $LOGFILE
fi
```

This inner `if` is nested inside the outer `if [ -d "$SOURCE" ]` block — it only runs if the source directory exists AND `tar` was attempted.

---

## Step 7 — Print Summary with `awk`

![[Pasted image 20260418154119.png]]
**Screenshot evidence (image9, highlighted in red):** After a successful backup, `awk` formats a human-readable summary:

```bash
# Print summary using awk
echo "$SOURCE $DEST/$FILENAME" | awk '{print "Original:", $1, "→ Backup File:", $2}' >> $LOGFILE
```

`echo` outputs two space-separated fields (`$SOURCE` and `$DEST/$FILENAME`). `awk` reads them as `$1` and `$2` and formats them into a readable line:
```
Original: /home/devops/docs → Backup File: /mnt/backups/backup-2026-03-01.tar.gz
```

---

## Complete Script

![[Pasted image 20260418154203.png]]
**Screenshot evidence (image20):** The complete `backup.sh` as seen in nano:

```bash
#!/bin/bash

# Define source and destination
SOURCE="/home/devops/docs"
DEST="/mnt/backups"

# Get today's date
DATE=$(date +%F)

# Define backup filename
FILENAME="backup-$DATE.tar.gz"

# Define a logfile
LOGFILE="/home/kazukali/backup.log"

# Logs each operation date into a logfile
echo "[$(date)] Backup Started." >> $LOGFILE

# Check if source directory exists
if [ -d "$SOURCE" ]; then
    echo "Source directory exists." >> $LOGFILE

    # Create the archive (stdout + stderr → logfile)
    tar -czvf "$DEST/$FILENAME" "$SOURCE" >> $LOGFILE 2>&1

    # Check if tar command was successful
    if [ $? -eq 0 ]; then
        echo "Backup successful!" >> $LOGFILE

        # Print summary using awk
	    echo "$SOURCE $DEST/$FILENAME" | awk '{print "Original:", $1, "→ Backup File:", $2}' >> $LOGFILE
    else
        echo "Error: Backup failed during archive creation." >> $LOGFILE
    fi

else
    echo "Error: Source directory does not exist." >> $LOGFILE
fi
```

---

## Step 8 — Run and Verify the Script

![[Pasted image 20260418154351.png]]
**Screenshot evidence (image10):** `sudo /home/kazukali/backup.sh` runs the script. Terminal output:
```
Source directory exists. Starting backup...
tar: Removing leading '/' from member names
/home/devops/docs/
/home/devops/docs/backup2.txt
/home/devops/docs/backup1.txt
Backup successful!
Original: /home/devops/docs → Backup File: /mnt/backups/backup-2026-03-01.tar.gz
```

![[Pasted image 20260418154425.png]]
**Screenshot evidence (image12):** `cat /home/kazukali/backup.log` shows two full run entries — the second run (highlighted in red) shows:
```
[Mon 02 Mar 2026 00:50:29 AEDT] Backup Started.
Source directory exists.
tar: Removing leading '/' ...
/home/devops/docs/
/home/devops/docs/backup2.txt
/home/devops/docs/backup1.txt
Backup successful!
Original: /home/devops/docs → Backup File: /mnt/backups/backup-2026-03-02.tar.gz
```

The log shows what happened, when it happened, and what was produced — full auditability.

---

## Step 9 — Make the Script Executable

![[Pasted image 20260418154530.png]]
**Screenshot evidence (image13):** `sudo chmod +x /home/kazukali/backup.sh` grants execute permission.

```bash
sudo chmod +x /home/kazukali/backup.sh
```

![[Pasted image 20260418154601.png]]
**Screenshot evidence (image14):** `ls -lh backup.sh` shows `-rwxr-xr-x 1 root root 740 Mar 1 23:31 backup.sh` — the `x` bits are now set for owner, group, and others.

```bash
ls -lh backup.sh
# -rwxr-xr-x 1 root root 740 Mar  1 23:31 backup.sh
```

---

## Step 10 — Schedule with crontab

### Open Root's crontab

![[Pasted image 20260418154735.png]]
**Screenshot evidence (image15):** `sudo crontab -e` opens root's crontab. Since it's empty, it prompts for an editor — option 1 (`/bin/nano`) is selected.

```bash
sudo crontab -e
```

> 💡 **`sudo crontab -e` vs `crontab -e`:** The backup script requires root privileges (to write to `/mnt/backups/`). Running `sudo crontab -e` edits **root's** crontab. Without `sudo`, the entry would run as the current user and likely fail with permission errors.

### Add the Cron Entry

![[Pasted image 20260418154824.png]]
**Screenshot evidence (image16):** Inside the nano crontab editor, the following line is added at the bottom (highlighted in red):

```bash
0 9 * * * /home/kazukali/backup.sh
```

This means: at minute 0, hour 9 (9:00 AM), every day, every month, every day of week — run the backup script.

**Full crontab entry:**
```bash
# m  h  dom mon dow  command
  0  9  *   *   *    /home/kazukali/backup.sh
```

---

## Tools & Alternatives

| Tool Used | Purpose | Alternative |
|---|---|---|
| `bash` script | Automate backup workflow | Python script (more readable for complex logic) |
| `tar -czvf` | Archive + compress | `rsync` (incremental), `zip`, `7z` |
| `date +%F` | Date-stamped filenames | `date +%Y%m%d` (no dashes), `date +%F_%T` (with time) |
| `echo >> $LOGFILE` | Append log entries | `tee -a $LOGFILE` (prints to both terminal and file) |
| `$?` | Check last command exit code | `set -e` (exit script on any error — simpler but less control) |
| `awk` | Format output | `printf`, `sed` |
| `crontab -e` | Schedule recurring task | `systemd timers` (more robust, survives reboots better) |
| `chmod +x` | Make script executable | `bash script.sh` (run without execute permission) |

---

## Cheatsheet — Key Commands

```bash
# --- Script Creation ---
touch script.sh                          # Create empty script file
nano script.sh                           # Edit script
chmod +x script.sh                       # Make executable
sudo ./script.sh                         # Run with sudo
bash script.sh                           # Run without execute permission

# --- Date Formatting ---
date +%F                                 # YYYY-MM-DD
date +%Y%m%d                             # YYYYMMDD
date +%F_%H-%M                           # YYYY-MM-DD_HH-MM
echo "$(date)"                           # Full date string in script

# --- Bash Conditionals ---
if [ -d "$DIR" ]; then ... fi            # Check if directory exists
if [ -f "$FILE" ]; then ... fi           # Check if file exists
if [ $? -eq 0 ]; then ... fi            # Check last command succeeded
if [ "$VAR" = "value" ]; then ... fi    # String comparison

# --- Output Redirection ---
command >> file.log                      # Append stdout to file
command >> file.log 2>&1                 # Append stdout + stderr to file
command 2>/dev/null                      # Discard errors
command | tee -a file.log               # Print to terminal AND append to file

# --- tar Archiving ---
tar -czvf archive.tar.gz /source/        # Create compressed archive
tar -tzf archive.tar.gz                  # Verify/list archive contents
tar -xzvf archive.tar.gz                 # Extract archive

# --- Cron ---
crontab -e                               # Edit current user's crontab
sudo crontab -e                          # Edit root's crontab
crontab -l                               # List crontab entries
# m  h  dom mon dow  command
  0  9  *   *   *    /path/to/script.sh  # Daily at 9AM

# --- awk Quick Reference ---
echo "a b" | awk '{print $1}'           # Print first field: a
echo "a b" | awk '{print $2}'           # Print second field: b
echo "a b" | awk '{print "X:", $1, "Y:", $2}'   # Formatted output
```

---

## Summary

| Objective | Result |
|---|---|
| Create `backup.sh` with shebang | ✅ `#!/bin/bash` |
| Define variables (SOURCE, DEST, DATE, FILENAME, LOGFILE) | ✅ `$(date +%F)` for date-stamped filename |
| Check if source directory exists | ✅ `if [ -d "$SOURCE" ]` |
| Create tar.gz archive | ✅ `tar -czvf "$DEST/$FILENAME" "$SOURCE"` |
| Redirect stdout + stderr to log | ✅ `>> $LOGFILE 2>&1` |
| Check tar success with `$?` | ✅ `if [ $? -eq 0 ]` |
| Format summary with `awk` | ✅ `echo "..." \| awk '{print "Original:", $1, ...}'` |
| Log all operations | ✅ Verified with `cat backup.log` |
| Make script executable | ✅ `chmod +x backup.sh` → `-rwxr-xr-x` |
| Schedule daily at 9AM via crontab | ✅ `0 9 * * * /home/kazukali/backup.sh` |
