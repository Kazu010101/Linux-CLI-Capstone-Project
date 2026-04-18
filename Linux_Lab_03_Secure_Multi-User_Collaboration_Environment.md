---
tags:
  - cybersecurity
  - linux-cli
  - permissions
  - user-management
  - access-control
  - capstone
status: complete
lab: 3
topic: Secure Multi-User Collaboration Environment
environment: Linux (Kali) — Bash
---

# Linux Lab 03 – Secure Multi-User Collaboration Environment

## Scenario

> The software team at **NovaStack** is collaborating on multiple microservices. Set up secure access in a shared project environment for different user groups — frontend (alice, bob) and backend (carol) — without overlapping permissions or accidental data leaks.

---

## Theory & Background

### Linux Permission Model

Every file and directory in Linux has three permission sets:

| Set | Who it applies to |
|---|---|
| **Owner** (`u`) | The user who owns the file |
| **Group** (`g`) | The group assigned to the file |
| **Others** (`o`) | Everyone else |

Each set has three permission bits:

| Bit | File meaning | Directory meaning |
|---|---|---|
| `r` (4) | Read file contents | List directory contents |
| `w` (2) | Modify file | Create/delete files inside |
| `x` (1) | Execute file | Enter (cd into) directory |

### Understanding `chmod` Numeric Notation

`chmod 770` breaks down as:
- `7` = Owner: 4+2+1 = rwx (Full Control)
- `7` = Group: 4+2+1 = rwx (Full Control)
- `0` = Others: 0 = --- (No access)

`chmod 750`:
- `7` = Owner: rwx
- `5` = Group: 4+0+1 = r-x (Read + Execute, no write)
- `0` = Others: no access

### SGID — Set Group ID (the `2` prefix)

`chmod 2770` sets the **SGID bit** on a directory. When SGID is set:
- Any **new files or subdirectories** created inside inherit the **directory's group**, not the creating user's primary group
- This means alice creates a file → group is `frontend`, not `alice` → bob can access it because he's also in `frontend`

Without SGID, alice's files would belong to group `alice` by default, and bob could not access them even though they share the `frontend` group.

### `chgrp` vs `chown`

| Command | Changes |
|---|---|
| `chgrp groupname path` | Only the group owner |
| `chown user:group path` | Both user owner and group owner |
| `chown :group path` | Only the group (colon prefix, no user) |

### `getfacl` — Detailed Permission View

`getfacl` (Get File ACL) shows detailed ownership and permission information including the ACL entries, which is more informative than plain `ls -l` for verifying group-based access.

---

## Setup — Create Users and Directory Structure

### Create Users: alice, bob, carol

![[Pasted image 20260418143706.png]]
![[Pasted image 20260418143721.png]]
**Screenshot evidence (image1 & image2):** `cat /etc/passwd` shows the root entry format, then scrolled down the list to show `alice:x:1004:1004:,,,:/home/alice:/bin/bash`, `bob:x:1005:1005:,,,:/home/bob:/bin/bash`, and `carol:x:1006:1006:,,,:/home/carol:/bin/bash` — confirming all three users were created with home directories and bash shell.

```bash
sudo adduser alice
sudo adduser bob
sudo adduser carol
cat /etc/passwd | tail -5    # Verify new entries
```

### Create Shared Directory Structure

![[Pasted image 20260418143844.png]]
**Screenshot evidence (image3):** `tree` run from `/opt/shared` shows: `projects/` containing `backend/` and `frontend/` — 4 directories, 0 files.

```bash
sudo mkdir -p /opt/shared/projects/frontend
sudo mkdir -p /opt/shared/projects/backend
tree /opt/shared
```

---

## Step 1 — Create Groups and Assign Users

### Create `frontend` Group and Add alice + bob

![[Pasted image 20260418143909.png]]
**Screenshot evidence (image4):** `sudo groupadd frontend` was run successfully (prompted for sudo password).

```bash
sudo groupadd frontend
```

![[Pasted image 20260418143932.png]]
**Screenshot evidence (image5):** `sudo usermod -aG frontend alice && sudo usermod -aG frontend bob` adds both users to `frontend` in a single chained command.

```bash
sudo usermod -aG frontend alice
sudo usermod -aG frontend bob
```

> 💡 **`-aG` flag is critical:** `-a` means **append** — add to the group without removing from other groups. Omitting `-a` would replace the user's group memberships entirely.

### Create `backend` Group and Add carol

![[Pasted image 20260418144054.png]]
![[Pasted image 20260418144106.png]]
**Screenshot evidence (image6 & image7):** `sudo groupadd backend` and `sudo usermod -aG backend carol` create the group and assign carol.

```bash
sudo groupadd backend
sudo usermod -aG backend carol
```

### Verify Group Assignments

![[Pasted image 20260418144128.png]]
**Screenshot evidence (image8):** `groups alice && groups bob && groups carol` shows:
- `alice : alice users frontend` ✅
- `bob : bob users frontend` ✅
- `carol : carol users backend` ✅

```bash
groups alice && groups bob && groups carol
```

---

## Step 2 — Change Group Ownership with `chgrp`

![[Pasted image 20260418144204.png]]
**Screenshot evidence (image9):** Before `chgrp`, `ls -ld /opt/shared/projects/frontend` shows `drwxr-xr-x 2 root root` — both directories still belong to group `root`.

![[Pasted image 20260418144229.png]]
**Screenshot evidence (image10):** `sudo chgrp frontend /opt/shared/projects/frontend && sudo chgrp backend /opt/shared/projects/backend` updates group ownership.

```bash
sudo chgrp frontend /opt/shared/projects/frontend
sudo chgrp backend  /opt/shared/projects/backend
```

![[Pasted image 20260418144307.png]]
**Screenshot evidence (image11):** After `chgrp`, `ls -ld` now shows `drwxr-xr-x 2 root frontend` and `drwxr-xr-x 2 root backend` — group ownership updated correctly (highlighted in red).

---

## Step 3 — Set Permissions with `chmod 770`

![[Pasted image 20260418144344.png]]
**Screenshot evidence (image12):** `sudo chmod 770 /opt/shared/projects/frontend && sudo chmod 770 /opt/shared/projects/backend` sets permissions for both directories in one command.

```bash
sudo chmod 770 /opt/shared/projects/frontend
sudo chmod 770 /opt/shared/projects/backend
```

![[Pasted image 20260418144406.png]]
**Screenshot evidence (image13):** `ls -ld` now shows `drwxrwx---` for both directories (highlighted in red):
- Owner (root): `rwx`
- Group (frontend/backend): `rwx`
- Others: `---` (no access)

This means alice and bob can fully access `frontend/`, carol can fully access `backend/`, and none of them can access the other team's directory.

**`chmod 770` vs `chmod 750`:**

| Permission  | Group can write? | Use case                                           |
| ----------- | ---------------- | -------------------------------------------------- |
| `chmod 770` | ✅ Yes            | Collaborative folder — members create/delete files |
| `chmod 750` | ❌ No             | Read-only group access                             |

---

## Step 4 — Set SGID with `chmod 2770`

![[Pasted image 20260418144800.png]]
**Screenshot evidence (image14):** `sudo chmod 2770 /opt/shared/projects/frontend && sudo chmod 2770 /opt/shared/projects/backend` sets the SGID bit on both directories.

```bash
sudo chmod 2770 /opt/shared/projects/frontend
sudo chmod 2770 /opt/shared/projects/backend
```

The `2` prefix sets the SGID bit. Combined with `770`, any new file created inside inherits the directory's group (`frontend` or `backend`) automatically.

---

## Step 5 — Test Access with `su`

### Test 1: Alice can access and create files in `frontend/`

![[Pasted image 20260418144904.png]]
**Screenshot evidence (image15):** `su - alice`, then `cd /opt/shared/projects/frontend/`, then `touch frontend.txt` — all succeed. Alice has full access to her team's directory.

```bash
su - alice
cd /opt/shared/projects/frontend/
touch frontend.txt    # Success
```

### Test 2: Alice cannot access `backend/`

![[Pasted image 20260418144928.png]]
**Screenshot evidence (image16):** From alice's session, `cd /opt/shared/projects/backend` returns `-bash: cd: /opt/shared/projects/backend: Permission denied` ✅

```bash
cd /opt/shared/projects/backend    # Permission denied — expected
```

### Test 3: Bob can access and create files in `frontend/`

![[Pasted image 20260418144953.png]]
**Screenshot evidence (image17):** `su - bob` (from alice's session), `cd /opt/shared/projects/frontend/`, `touch bob_frontend.txt` — all succeed.

```bash
su - bob
cd /opt/shared/projects/frontend/
touch bob_frontend.txt    # Success
```

### Test 4: Bob cannot access `backend/`

![[Pasted image 20260418145206.png]]
**Screenshot evidence (image18):** `cd /opt/shared/projects/backend/` returns `Permission denied` ✅

### Test 5: Carol can access and create files in `backend/`

![[Pasted image 20260418145226.png]]
**Screenshot evidence (image19):** `su - carol`, `cd /opt/shared/projects/backend/`, `touch backend.txt` — all succeed.

```bash
su - carol
cd /opt/shared/projects/backend/
touch backend.txt    # Success
```

### Test 6: Carol cannot access `frontend/`

![[Pasted image 20260418145246.png]]
**Screenshot evidence (image20):** `cd /opt/shared/projects/frontend/` returns `Permission denied` ✅

### Test 7: Bob can write into alice's file (SGID in action)

![[Pasted image 20260418145333.png]]
**Screenshot evidence (image21):** From bob's session, `ls` shows both `bob_frontend.txt` and `frontend.txt`. `echo "I can write in alice's file" >> frontend.txt` succeeds. `cat frontend.txt` confirms the content was written. This works because SGID ensures `frontend.txt` belongs to group `frontend`, and bob is a member.

```bash
echo "I can write in alice's file" >> frontend.txt
cat frontend.txt    # Output: I can write in alice's file
```

### Verify File Ownership with SGID

![[Pasted image 20260418145429.png]]
**Screenshot evidence (image22):** `ls -lh` from alice's session inside `frontend/` shows:
- `bob_frontend.txt` — owner: `bob`, group: `frontend` ✅
- `frontend.txt` — owner: `alice`, group: `frontend` ✅
- `new_file.txt` — owner: `alice`, group: `frontend` ✅

All files belong to group `frontend` regardless of who created them — SGID is working correctly.

### Confirm with `getfacl`

![[Pasted image 20260418145536.png]]
**Screenshot evidence (image23):** `getfacl frontend.txt` from bob's session shows:
```
# file: frontend.txt
# owner: alice
# group: frontend
user::rw-
group::rw-
other::r--
```

This confirms: owner is alice, group is `frontend` (not `alice`), and both user and group have read+write — bob can access it.

```bash
getfacl frontend.txt
```

---

## Tools & Alternatives

| Tool Used | Purpose | Alternative / Production Tool |
|---|---|---|
| `groupadd` | Create a local group | `sudo addgroup` (Debian-style wrapper) |
| `usermod -aG` | Add user to group | `gpasswd -a user group` |
| `chgrp` | Change group ownership | `chown :group path` |
| `chmod 2770` | Set permissions + SGID | `setfacl` (fine-grained ACL control) |
| `su - user` | Switch user | `sudo -u user bash` (no target password needed) |
| `getfacl` | View file ACL | `ls -l` (basic), `stat file` (more detail) |
| `groups username` | View group memberships | `id username` (shows UID, GID, all groups) |

---

## Cheatsheet — Key Commands

```bash
# --- User & Group Management ---
sudo adduser username                          # Create user (interactive)
sudo groupadd groupname                        # Create a group
sudo usermod -aG groupname username            # Add user to group (append)
groups username                                # Show user's group memberships
id username                                    # Show UID, GID, all groups
cat /etc/passwd | grep username                # Verify user in passwd file
cat /etc/group  | grep groupname               # Verify group members

# --- Ownership ---
sudo chgrp groupname /path/to/dir             # Change group owner of directory
sudo chown user:group /path/to/file           # Change both user and group owner
sudo chown :group /path/to/file               # Change group only (colon prefix)

# --- Permissions ---
sudo chmod 770  /path                          # rwxrwx--- (owner+group full, others none)
sudo chmod 750  /path                          # rwxr-x--- (group read+exec only)
sudo chmod 660  /path/to/file                  # rw-rw---- (file: owner+group read/write)
sudo chmod 2770 /path                          # SGID + 770 (new files inherit group)

# chmod numeric quick reference:
# 7 = rwx, 6 = rw-, 5 = r-x, 4 = r--, 0 = ---

# --- Verification ---
ls -ld /path/to/dir                            # Show directory permissions and ownership
ls -lh /path/to/dir                            # Show file permissions inside directory
getfacl filename                               # Detailed ACL view (owner, group, perms)
stat filename                                  # Full file metadata

# --- Switching Users ---
su - username                                  # Switch user (full login environment)
sudo -u username bash                          # Open shell as user (uses your sudo)
whoami                                         # Confirm current user
exit                                           # Return to previous user

# --- Testing Access ---
cd /path/to/dir                                # Test directory access (Permission denied = works)
touch testfile.txt                             # Test write access
cat filename                                   # Test read access
```

---

## Summary

| Objective | Result |
|---|---|
| Create alice, bob, carol | ✅ `adduser` × 3, verified in `/etc/passwd` |
| Create `/opt/shared/projects/frontend` and `backend` | ✅ `mkdir -p` |
| Create `frontend` and `backend` groups | ✅ `groupadd` |
| Assign alice+bob → frontend, carol → backend | ✅ `usermod -aG` |
| Change group ownership with `chgrp` | ✅ `frontend` and `backend` groups own directories |
| Set `chmod 770` — group full, others none | ✅ `drwxrwx---` confirmed |
| Set `chmod 2770` — SGID for group inheritance | ✅ New files inherit `frontend`/`backend` group |
| Test: alice/bob can access frontend, not backend | ✅ Permission denied confirmed |
| Test: carol can access backend, not frontend | ✅ Permission denied confirmed |
| Test: bob can write into alice's file (SGID) | ✅ `echo >>` succeeded, `getfacl` confirmed |
