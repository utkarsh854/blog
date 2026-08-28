---
title: "Simple CTF — TryHackMe Write-up"
date: 2026-08-26 12:00:00 +0530
categories: [TryHackMe]
tags: [simplectf, sqli, cve-2019-9053]
image:
  path: /assets/images/simple-ctf/banner.png
  alt: Simple CTF
---

Greetings, everyone! Today we'll be taking an in depth look at the TryHackMe **Simple CTF** room, which has a little bit of everything and is a great CTF for a beginner. I'm designing these walkthroughs to keep myself motivated to learn cyber security and to make sure that I remember the knowledge gained by THM's rooms. Come along with me as I learn cyber security, and I'll try to explain concepts as I go to set myself apart from other walkthroughs.

Have fun in the room!

Room link: [tryhackme.com/r/room/easyctf](https://tryhackme.com/r/room/easyctf)

## Reconnaissance

Let's begin by using nmap, which I always scan before entering a room.

```bash
nmap -sCV <TARGET_IP>
```

![2 ports are open]({{ '/assets/images/simple-ctf/nmap-scan.png' | relative_url }})

From our results, we can see ports 21 (FTP), 80 (HTTP), and 2222 (SSH) are open.

> **How many services are running under port 1000?**
>
> *2*

> **What is running on the higher port?**
>
> *SSH*

Numerous fascinating facts can be found here. We can observe an anonymous FTP login, a robots.txt file containing disallowed content, and, most importantly for our research, we find SSH functionality.

Note that anonymous login for FTP is allowed, let's see if we can access any sensitive information via FTP.

![FTP anonymous login note]({{ '/assets/images/simple-ctf/ftp-note.png' | relative_url }})

Well, I do not change my directory, nothing really interesting in the note. Moving on to port 80. As we previously discovered that port 80 is running the http service we will use the Firefox browser, so open a new tab and enter your target machine IP. This brings up an "Apache2 Ubuntu Default Page." Not too exciting.

![Apache2 Ubuntu default page]({{ '/assets/images/simple-ctf/apache-default.png' | relative_url }})

Next, we can use "gobuster" to scan the website for any additional pages.

![gobuster result]({{ '/assets/images/simple-ctf/gobuster-result.png' | relative_url }})

Using the big wordlist we supplied, gobuster was able to find there is a webpage at "/simple". Let's try browsing to it now and see what we find.

![CMS system]({{ '/assets/images/simple-ctf/cms-system.png' | relative_url }})

![CMS version 2.2.8]({{ '/assets/images/simple-ctf/cms-version.png' | relative_url }})

This seems interesting, it opens up a CMS system. Quickly reading through the page we can see that it is Simple CMS version 2.2.8.

Running a quick google search or a search on exploit-db.com for known exploits associated with it and we can see that there is indeed an exploit. This will help us answer Q3 and Q4.

![exploit-db search results]({{ '/assets/images/simple-ctf/exploitdb-search.png' | relative_url }})

In our results, we see a page on Exploit-DB that matches our search and refers to a SQL injection attack utilizing [CVE-2019-9053](https://www.exploit-db.com/exploits/46635).

> **What's the CVE you're using against the application?**
>
> *CVE-2019-9053*

> **To what kind of vulnerability is the application vulnerable?**
>
> *SQLi*

## Exploitation

To exploit this vulnerability, all we ideally need to do is download the script right from ExploitDB and run it. Optionally, if you're using Kali or ParrotOS (as I am), the script is located in:

```
/usr/share/exploitdb/exploits/php/webapps/46635.py
```

I will be downloading the script and moving it to the directory that I am working from.

Reference: [CMS Made Simple < 2.2.10 - SQL Injection, CVE-2019-9053](https://www.exploit-db.com/exploits/46635)

**Pro tip:** Reading the script can tell us a lot about how to run it. Here's a snippet from the beginning of the script:

![exploit script snippet]({{ '/assets/images/simple-ctf/exploit-script.png' | relative_url }})

The script wants us to provide three options:

- `-u`: Target URI, or the URL to the website we will be attacking
- `-c`: Crack, whether we want the script to attempt to crack any hashes it finds (which we do)
- `-w`: Wordlist, specifies a wordlist to use for cracking. I will be using `rockyou.txt`

Let's try running this script in python as-is:

```bash
python3 46635.py -u http://10.10.192.40/simple -c -w /usr/share/wordlists/rockyou.txt
```

![Bingo — cracked credentials returned]({{ '/assets/images/simple-ctf/bingo-result.png' | relative_url }})

Bingo! We got a username and a cracked password returned from the exploit.
