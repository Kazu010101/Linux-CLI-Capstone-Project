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

<img width="489" height="120" alt="image" src="https://github.com/user-attachments/assets/52c4a3ee-88b3-4cc6-a4cc-4bd6ebbcce26" />

<img width="488" height="65" alt="image" src="https://github.com/user-attachments/assets/e9f5330f-5b29-408d-83ed-4a3fb3be6509" />

**Screenshot evidence (image1 & image2):** `cat /etc/passwd` shows the root entry format, then scrolled down the list to show `alice:x:1004:1004:,,,:/home/alice:/bin/bash`, `bob:x:1005:1005:,,,:/home/bob:/bin/bash`, and `carol:x:1006:1006:,,,:/home/carol:/bin/bash` — confirming all three users were created with home directories and bash shell.

```bash
sudo adduser alice
sudo adduser bob
sudo adduser carol
cat /etc/passwd | tail -5    # Verify new entries
```

### Create Shared Directory Structure

<img width="408" height="179" alt="image" src="https://github.com/user-attachments/assets/1b72641c-6dfd-41d6-96c4-f3d179073484" />

**Screenshot evidence (image3):** `tree` run from `/opt/shared` shows: `projects/` containing `backend/` and `frontend/` — 4 directories, 0 files.

```bash
sudo mkdir -p /opt/shared/projects/frontend
sudo mkdir -p /opt/shared/projects/backend
tree /opt/shared
```

---

## Step 1 — Create Groups and Assign Users

### Create `frontend` Group and Add alice + bob

<img width="340" height="78" alt="image" src="https://github.com/user-attachments/assets/2cbd2b0f-4fdb-465d-a9db-ba661e653532" />

**Screenshot evidence (image4):** `sudo groupadd frontend` was run successfully (prompted for sudo password).

```bash
sudo groupadd frontend
```

<img width="756" height="59" alt="image" src="https://github.com/user-attachments/assets/f2e037c2-8ae2-48ea-bd49-2ef637b6c96c" />

**Screenshot evidence (image5):** `sudo usermod -aG frontend alice && sudo usermod -aG frontend bob` adds both users to `frontend` in a single chained command.

```bash
sudo usermod -aG frontend alice
sudo usermod -aG frontend bob
```

> 💡 **`-aG` flag is critical:** `-a` means **append** — add to the group without removing from other groups. Omitting `-a` would replace the user's group memberships entirely.

### Create `backend` Group and Add carol

<img width="298" height="54" alt="image" src="https://github.com/user-attachments/assets/46d71107-c4db-4b34-995d-45403a9cd5e7" />

<img width="484" height="72" alt="image" src="https://github.com/user-attachments/assets/5e30a99e-01cc-4782-b274-078adb0a53fd" />

**Screenshot evidence (image6 & image7):** `sudo groupadd backend` and `sudo usermod -aG backend carol` create the group and assign carol.

```bash
sudo groupadd backend
sudo usermod -aG backend carol
```

### Verify Group Assignments

<img width="633" height="145" alt="image" src="https://github.com/user-attachments/assets/6fc14ae0-d90a-4c37-b669-cdf70ffc7a1a" />

**Screenshot evidence (image8):** `groups alice && groups bob && groups carol` shows:
- `alice : alice users frontend` ✅
- `bob : bob users frontend` ✅
- `carol : carol users backend` ✅

```bash
groups alice && groups bob && groups carol
```

---

## Step 2 — Change Group Ownership with `chgrp`

<img width="748" height="158" alt="image" src="https://github.com/user-attachments/assets/6898e3d8-d955-4232-835b-93850fd4d5ce" />

**Screenshot evidence (image9):** Before `chgrp`, `ls -ld /opt/shared/projects/frontend` shows `drwxr-xr-x 2 root root` — both directories still belong to group `root`.

<img width="1056" height="71" alt="image" src="https://github.com/user-attachments/assets/bd907790-dcb3-47dd-8eb5-89fb72d5ee41" />

**Screenshot evidence (image10):** `sudo chgrp frontend /opt/shared/projects/frontend && sudo chgrp backend /opt/shared/projects/backend` updates group ownership.

```bash
sudo chgrp frontend /opt/shared/projects/frontend
sudo chgrp backend  /opt/shared/projects/backend
```

<img width="785" height="160" alt="image" src="https://github.com/user-attachments/assets/d60f9a00-fc65-4a43-98a9-cd37995e7be6" />

**Screenshot evidence (image11):** After `chgrp`, `ls -ld` now shows `drwxr-xr-x 2 root frontend` and `drwxr-xr-x 2 root backend` — group ownership updated correctly (highlighted in red).

---

## Step 3 — Set Permissions with `chmod 770`

<img width="985" height="53" alt="image" src="https://github.com/user-attachments/assets/cb8fed67-568c-4ed4-8554-559e7de7c972" />

**Screenshot evidence (image12):** `sudo chmod 770 /opt/shared/projects/frontend && sudo chmod 770 /opt/shared/projects/backend` sets permissions for both directories in one command.

```bash
sudo chmod 770 /opt/shared/projects/frontend
sudo chmod 770 /opt/shared/projects/backend
```

<img width="771" height="154" alt="image" src="https://github.com/user-attachments/assets/dfa51525-ae10-4379-97bc-b920def97148" />

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

<img width="1003" height="59" alt="image" src="https://github.com/user-attachments/assets/a8ee46c6-8659-4c4e-9686-30f7235b68d8" />

**Screenshot evidence (image14):** `sudo chmod 2770 /opt/shared/projects/frontend && sudo chmod 2770 /opt/shared/projects/backend` sets the SGID bit on both directories.

```bash
sudo chmod 2770 /opt/shared/projects/frontend
sudo chmod 2770 /opt/shared/projects/backend
```

The `2` prefix sets the SGID bit. Combined with `770`, any new file created inside inherits the directory's group (`frontend` or `backend`) automatically.

---

## Step 5 — Test Access with `su`

### Test 1: Alice can access and create files in `frontend/`

<img width="569" height="256" alt="image" src="https://github.com/user-attachments/assets/c5c4de42-6f81-4c00-8f0a-267e46484cc6" />

**Screenshot evidence (image15):** `su - alice`, then `cd /opt/shared/projects/frontend/`, then `touch frontend.txt` — all succeed. Alice has full access to her team's directory.

```bash
su - alice
cd /opt/shared/projects/frontend/
touch frontend.txt    # Success
```

### Test 2: Alice cannot access `backend/`

<img width="623" height="76" alt="image" src="https://github.com/user-attachments/assets/461f6dd5-c756-49a9-b330-819d78d56854" />

**Screenshot evidence (image16):** From alice's session, `cd /opt/shared/projects/backend` returns `-bash: cd: /opt/shared/projects/backend: Permission denied` ✅

```bash
cd /opt/shared/projects/backend    # Permission denied — expected
```

### Test 3: Bob can access and create files in `frontend/`

<img width="539" height="171" alt="image" src="https://github.com/user-attachments/assets/53672531-c4ec-4923-96eb-ee69eaf77ba8" />

**Screenshot evidence (image17):** `su - bob` (from alice's session), `cd /opt/shared/projects/frontend/`, `touch bob_frontend.txt` — all succeed.

```bash
su - bob
cd /opt/shared/projects/frontend/
touch bob_frontend.txt    # Success
```

### Test 4: Bob cannot access `backend/`

<img width="625" height="75" alt="image" src="https://github.com/user-attachments/assets/a733352e-b754-4c5b-9acc-f9e4a92821ad" />

**Screenshot evidence (image18):** `cd /opt/shared/projects/backend/` returns `Permission denied` ✅

### Test 5: Carol can access and create files in `backend/`

<img width="706" height="216" alt="image" src="https://github.com/user-attachments/assets/5ffd101b-b774-4bf4-b245-902ac2059e81" />

**Screenshot evidence (image19):** `su - carol`, `cd /opt/shared/projects/backend/`, `touch backend.txt` — all succeed.

```bash
su - carol
cd /opt/shared/projects/backend/
touch backend.txt    # Success
```

### Test 6: Carol cannot access `frontend/`

<img width="631" height="76" alt="image" src="https://github.com/user-attachments/assets/be494af3-cbfa-4e49-8e49-13f167686807" />

**Screenshot evidence (image20):** `cd /opt/shared/projects/frontend/` returns `Permission denied` ✅

### Test 7: Bob can write into alice's file (SGID in action)

<img width="808" height="345" alt="image" src="https://github.com/user-attachments/assets/3eeaebea-3b9d-4049-8b52-a83476e02828" />

**Screenshot evidence (image21):** From bob's session, `ls` shows both `bob_frontend.txt` and `frontend.txt`. `echo "I can write in alice's file" >> frontend.txt` succeeds. `cat frontend.txt` confirms the content was written. This works because SGID ensures `frontend.txt` belongs to group `frontend`, and bob is a member.

```bash
echo "I can write in alice's file" >> frontend.txt
cat frontend.txt    # Output: I can write in alice's file
```

### Verify File Ownership with SGID

<img width="653" height="129" alt="image" src="https://github.com/user-attachments/assets/814128db-66ab-48f5-8162-1ffe97d4aa54" />

**Screenshot evidence (image22):** `ls -lh` from alice's session inside `frontend/` shows:
- `bob_frontend.txt` — owner: `bob`, group: `frontend` ✅
- `frontend.txt` — owner: `alice`, group: `frontend` ✅
- `new_file.txt` — owner: `alice`, group: `frontend` ✅

All files belong to group `frontend` regardless of who created them — SGID is working correctly.

### Confirm with `getfacl`

<img width="548" height="171" alt="image" src="https://github.com/user-attachments/assets/7fdcc2e6-443a-41be-b545-0933db3e2469" />

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
