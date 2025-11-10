# 🧾 **README – Linux User and Group Administration**

## 📘 **Overview**

This document explains key **user administration tasks** in a **Linux-based operating system**, including creating, modifying, and deleting user accounts, managing groups, setting file permissions, and monitoring user activity.  
It serves as a concise reference for system administrators to maintain **security**, **policy compliance**, and **efficient resource utilization**.

---

## 👤 **1. User Account Management**

### 🟩 **Create a New User**
```bash
sudo useradd [options] username
```
**Examples:**
```bash
sudo useradd dev
sudo useradd -m -s /bin/bash -c "Developer Account" dev
```
**Options:**
- `-m` → create a home directory  
- `-s` → specify shell (e.g., `/bin/bash`)  
- `-c` → add a comment  
- `-u` → specify UID  
- `-g` → assign primary group  
- `-G` → assign secondary groups  

---

### 🟨 **Set or Change Password**
```bash
sudo passwd username
sudo passwd -x 30 username   # expire password in 30 days
```

---

### 🔳 **Modify User Account**
```bash
sudo usermod [options] username
```
**Examples:**
```bash
sudo usermod -l newname dev        # rename user
sudo usermod -s /bin/zsh dev       # change shell
sudo usermod -aG sudo dev          # add to sudo group
```

---

### 🔴 **Delete a User**
```bash
sudo userdel [options] username
```
**Examples:**
```bash
sudo userdel dev
sudo userdel -r dev   # remove user and home directory
```

---

## 👥 **2. Group Management**

### 🟩 **Create Group**
```bash
sudo groupadd [options] groupname
```
**Example:**
```bash
sudo groupadd developers
```

---

### 🟨 **Manage Group Members**
```bash
sudo gpasswd [options] groupname
```
**Examples:**
```bash
sudo gpasswd -a dev developers  # add user
sudo gpasswd -d dev developers  # remove user
```

---

### 🔴 **Delete Group**
```bash
sudo groupdel groupname
```
**Example:**
```bash
sudo groupdel developers
```

---

## 🗂️ **3. File and Directory Permissions**

Every file/directory has an **owner**, **group**, and **others** with permission bits for **read (r = 4)**, **write (w = 2)**, and **execute (x = 1)**.

### 🟩 **Change Permissions (chmod)**
```bash
chmod [permissions] filename
```
**Examples:**
```bash
chmod 755 script.sh        # rwxr-xr-x
chmod u+x script.sh        # add execute permission for user
chmod g-w file.txt         # remove write permission for group
```

---

### 🟨 **Change Ownership (chown)**
```bash
sudo chown [newowner] filename
```
**Examples:**
```bash
sudo chown dev file.txt
sudo chown dev:developers file.txt
```

---

### 🔴 **Change Group Ownership (chgrp)**
```bash
sudo chgrp groupname filename
```
**Example:**
```bash
sudo chgrp developers project.txt
```

---

### 🛠️ **Advanced Permissions**

- **SUID (Set User ID):** Run with file owner's privileges.
  ```bash
  chmod u+s filename
  ```
- **SGID (Set Group ID):** New files inherit group ownership.
  ```bash
  chmod g+s directory_name
  ```
- **Sticky Bit:** Only file owner can delete their own files.
  ```bash
  chmod +t /shared
  ```

---

## 🔍 **4. Monitoring User Activities**

### **who** — Show logged-in users
```bash
who
```

### **w** — Show who is logged in and what they are doing
```bash
w
```

### **last** — Display login history
```bash
last
```

### **Log Files**
- `/var/log/auth.log` (Ubuntu/Debian)
- `/var/log/secure` (RHEL/CentOS)
```bash
sudo tail /var/log/auth.log
sudo grep "FAILED" /var/log/secure
```

---

## ✅ **Summary Table**

| Task | Command | Description |
|------|----------|-------------|
| Create User | `useradd` | Add new user |
| Modify User | `usermod` | Edit user info |
| Delete User | `userdel` | Remove user |
| Create Group | `groupadd` | Add new group |
| Manage Group | `gpasswd` | Add/remove group members |
| Delete Group | `groupdel` | Remove group |
| Change Permissions | `chmod` | Modify file permissions |
| Change Owner | `chown` | Change file ownership |
| Change Group | `chgrp` | Change group ownership |
| View Users | `who`, `w`, `last` | Display user activity |
| Monitor Logs | `/var/log/auth.log` | Authentication logs |

---

### 📄 **Author:** System Administration Reference
### 🕛 **Version:** 1.0
### 🔒 **Purpose:** For learning and practical lab use on Linux user a