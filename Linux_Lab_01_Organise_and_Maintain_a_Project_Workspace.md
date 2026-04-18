---
tags:
  - cybersecurity
  - linux-cli
  - file-management
  - devops
  - capstone
status: complete
lab: 1
topic: Organise and Maintain a Project Workspace
environment: Linux (Kali) — Bash
---

# Linux Lab 01 – Organise and Maintain a Project Workspace

## Scenario

> You are a Junior DevOps Technician at **NovaStack Solutions**. As part of a new internal tool rollout, you are tasked with organising project directories, creating placeholder files, archiving old content, and cleaning up unused files — all through the Linux CLI.

---

## Theory & Background

### Linux Filesystem Hierarchy

Linux organises files in a single unified tree rooted at `/`. User home directories live under `/home/<username>/`. Understanding absolute vs. relative paths is essential:

- **Absolute path** — starts from `/`, e.g. `/home/devops/workspace/project_a/final_draft.txt`
- **Relative path** — starts from current directory, e.g. `../archive/` (one level up, then into archive)

### How `rm` Works — and Why It's Dangerous

Unlike Windows Recycle Bin or macOS Trash, `rm` in Linux **does not move files to a recoverable location**. It removes the filesystem link to the data block. The data may still exist on disk, but becomes inaccessible and will be overwritten by new data. Recovery tools (e.g. `extundelete`, `testdisk`) can sometimes recover files, but only if the blocks haven't been reused yet.

> 🔗 Reference: https://unix.stackexchange.com/questions/10883/where-do-files-go-when-the-rm-command-is-issued

### `mv` as a Rename Tool

In Linux, `mv` serves a dual purpose — it moves files between directories **and** renames files. There is no dedicated `rename` command for single-file renaming. When source and destination are in the same directory, `mv` simply renames.

---

## Setup — Create and Switch to devops User

![[Pasted image 20260418134818.png]]
**Screenshot evidence (image1):** `sudo adduser devops` was run from `/opt/dev`. The interactive prompt filled in password and user info fields. The final confirmation `Is the information correct? [Y/n] y` completed user creation.

```bash
sudo adduser devops
```

![[Pasted image 20260418134838.png]]
**Screenshot evidence (image2):** `cat /etc/passwd | grep 'devops'` confirms the user was created successfully. The entry reads `devops:x:1002:1002:,,,:/home/devops:/bin/bash` — confirming UID 1002, home directory `/home/devops`, and default shell `/bin/bash`.

```bash
cat /etc/passwd | grep 'devops'
```

![[Pasted image 20260418134944.png]]

![[Pasted image 20260418135005.png]]
**Screenshot evidence (image3 & image4):** The session was switched to the devops user. `whoami` confirms the identity as `devops`. `cd /home/devops/` navigates to the home directory, and the prompt changes to `(devops@kazukali)-[~]` confirming the switch.

```bash
su - devops       # Switch to devops user (full login shell with -)
whoami            # Verify: devops
cd /home/devops/  # Navigate to home directory
```

> 💡 **`su -` vs `su`:** The `-` flag loads the full login environment of the target user (PATH, variables, home directory). Without it, you switch user but keep your current environment — which can cause commands to behave unexpectedly.

---

## Setup — Create Initial Workspace Structure

![[Pasted image 20260418135208.png]]
**Screenshot evidence (image5):** `tree` run from `~/workspace` shows the full initial structure: `archive/`, `project_a/` (containing `config.json`, `draft.txt`, `notes.md`), and `project_b/` (containing `old_report.log` and `tempt.txt`).

```bash
mkdir -p workspace/project_a workspace/project_b workspace/archive
cd workspace
touch project_a/draft.txt project_a/notes.md project_a/config.json
touch project_b/old_report.log project_b/temp.txt
tree
```

> 💡 `mkdir -p` creates all intermediate directories in one command. Without `-p`, you need to create each level separately.

---

## Objective 1 — Create `backups` Directory

![[Pasted image 20260418135340.png]]
**Screenshot evidence (image6):** `mkdir backups` was run from `~/workspace`. The subsequent `ls` confirms `backups` now appears alongside `archive`, `project_a`, and `project_b`.

```bash
mkdir backups
ls
```

---

## Objective 2 — Create Log Files in `project_b`

![[Pasted image 20260418135410.png]]
**Screenshot evidence (image7):** `touch system.log error.log` was run from `~/workspace/project_b`. The subsequent `ls` confirms both `error.log` and `system.log` are now listed alongside `old_report.log` and `tempt.txt`.

```bash
cd project_b
touch system.log error.log
ls
```

> 💡 `touch` creates an empty file if it doesn't exist, or updates its timestamp if it does. It's the standard way to create placeholder files in Linux.

---

## Objective 3 — Copy `config.json` to `backups`

![[Pasted image 20260418135508.png]]
**Screenshot evidence (image8):** `cp config.json /home/devops/workspace/backups/` was run from `~/workspace/project_a`. The absolute destination path was used to avoid ambiguity.

```bash
cd ~/workspace/project_a
cp config.json /home/devops/workspace/backups/
```

**Common cause of "No such file or directory" error:**
1. Destination path is wrong or mistyped
2. Filename is mistyped (Linux filenames are case-sensitive — `Config.json` ≠ `config.json`)

---

## Objective 4 — Move `old_report.log` to `archive`

![[Pasted image 20260418135608.png]]
**Screenshot evidence (image9):** `mv old_report.log /home/devops/workspace/archive/` was run from `~/workspace/project_b`. The absolute path ensures the file goes to the correct archive directory.

```bash
cd ~/workspace/project_b
mv old_report.log /home/devops/workspace/archive/
```

---

## Objective 5 — Rename `draft.txt` to `final_draft.txt`

![[Pasted image 20260418135651.png]]
**Screenshot evidence (image10):** `mv draft.txt final_draft.txt` was run from `~/workspace/project_a`. The subsequent `ls` confirms `final_draft.txt` is now listed alongside `config.json` and `notes.md`. `draft.txt` is gone.

```bash
cd ~/workspace/project_a
mv draft.txt final_draft.txt
ls
```

**Absolute path after renaming:**
```
/home/devops/workspace/project_a/final_draft.txt
```

---

## Objective 6 — Delete `temp.txt`

![[Pasted image 20260418135726.png]]
**Screenshot evidence (image11):** `rm tempt.txt` was run from `~/workspace/project_b`. The subsequent `ls` shows only `error.log` and `system.log` — `tempt.txt` is gone, confirming successful deletion.

```bash
cd ~/workspace/project_b
rm tempt.txt
ls    # Verify temp.txt is gone
```

> ⚠️ **Why `rm` is dangerous:** There is no undo. If you accidentally delete the wrong file, recovery requires forensic tools and is not guaranteed. Always double-check the filename before running `rm`. Use `rm -i` for interactive confirmation, especially when using wildcards.

---

## Objective 7 — Verify Final Structure with `tree`

![[Pasted image 20260418135902.png]]
**Screenshot evidence (image12):** `tree` run from `~/workspace` shows the final clean state:
- `archive/old_report.log` ✅ — moved from project_b
- `backups/config.json` ✅ — copied from project_a
- `project_a/config.json`, `final_draft.txt`, `notes.md` ✅
- `project_b/error.log`, `system.log` ✅ — temp.txt deleted, old_report.log moved
- **5 directories, 7 files**

```bash
cd ~/workspace
tree
```

---

## Additional — Copy Multiple File Types with Wildcards

![[Pasted image 20260418135939.png]]
**Screenshot evidence (image13):** `cp *.md *.json /home/devops/workspace/backups/` was run from `~/workspace/project_a`. The subsequent `tree` confirms `backups/` now contains both `config.json` and `notes.md` alongside the previously copied `config.json`.

```bash
cp *.md *.json /home/devops/workspace/backups/
```

The `*` wildcard matches any filename with the specified extension. This is more efficient than listing each file individually.

---

## Tools & Alternatives

| Tool Used | Purpose | Alternative |
|---|---|---|
| `mkdir` | Create directories | `install -d` (also sets permissions) |
| `touch` | Create empty files | `echo "" > file.txt` |
| `cp` | Copy files | `rsync` (better for large/incremental copies) |
| `mv` | Move or rename files | `rename` (for batch renaming with regex) |
| `rm` | Delete files permanently | `trash-put` (sends to Trash, recoverable) |
| `ls` | List directory contents | `exa` / `lsd` (modern replacements with colour and icons) |
| `tree` | Visualise directory structure | `find . \| sed ...` (manual tree with find) |

---

## Cheatsheet — Key Commands

```bash
# --- User Management ---
sudo adduser username              # Create new user (interactive)
su - username                      # Switch user with full login environment
whoami                             # Show current user
cat /etc/passwd | grep username    # Verify user exists

# --- Navigation ---
cd /absolute/path                  # Navigate to absolute path
cd relative/path                   # Navigate to relative path
cd ~                               # Go to current user's home
cd ..                              # Go up one directory
pwd                                # Print working directory

# --- Directory Operations ---
mkdir dirname                      # Create directory
mkdir -p path/to/nested/dir        # Create nested directories in one shot
tree                               # Visual directory tree (install: sudo apt install tree)
ls                                 # List contents
ls -lh                             # Detailed list with human-readable sizes
ls -la                             # Include hidden files

# --- File Operations ---
touch file.txt                     # Create empty file / update timestamp
cp source dest                     # Copy file
cp *.md *.json /dest/              # Copy multiple types with wildcards
mv source dest                     # Move file
mv oldname.txt newname.txt         # Rename file (same directory)
rm file.txt                        # Delete file (permanent, no undo)
rm -i file.txt                     # Interactive delete (asks for confirmation)
rm -rf dirname/                    # Recursively delete directory (DANGEROUS)

# --- Viewing ---
cat file.txt                       # Print file contents
```

---

## Summary

| Objective | Result |
|---|---|
| Create `backups/` directory | ✅ `mkdir backups` |
| Create `system.log`, `error.log` in `project_b` | ✅ `touch system.log error.log` |
| Copy `config.json` to `backups` | ✅ `cp config.json /home/devops/workspace/backups/` |
| Move `old_report.log` to `archive` | ✅ `mv old_report.log /home/devops/workspace/archive/` |
| Rename `draft.txt` → `final_draft.txt` | ✅ `mv draft.txt final_draft.txt` |
| Delete `temp.txt` | ✅ `rm tempt.txt` + verified with `ls` |
| Verify structure with `ls` / `tree` | ✅ 5 directories, 7 files confirmed |
| Copy multiple file types with wildcards | ✅ `cp *.md *.json /backups/` |
