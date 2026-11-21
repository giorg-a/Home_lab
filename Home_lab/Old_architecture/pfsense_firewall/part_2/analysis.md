# Suricata IDS on pfSense

## What this session was about

Added Suricata to pfSense as an actual IDS, watching traffic on both DMZ and LAN interfaces, then ran different nmap scan types against Ubuntu to see what actually gets caught and what doesn't. Point of the exercise was comparing loud scanning behavior vs quiet stealth scanning and seeing how Suricata reacts to each.

## Installing Suricata

Found it under System > Package Manager > Available Packages, searched "suric", installed version 7.0.9.

```
Name       Version   Description
suricata   7.0.9     High Performance Network IDS, IPS and Security Monitoring engine by OISF
```

## Setting it up on an interface

Went to Services > Suricata > Interfaces, added a new instance for DMZ. Got hit with a warning right away saying hardware checksum offloading needed to be disabled or Suricata wouldn't inspect packets correctly:

```
WARNING! Suricata now requires that Hardware Checksum Offloading, Hardware TCP 
Segmentation Offloading and Hardware Large Receive Offloading all be disabled for 
proper operation.
```

Fixed that under System > Advanced > Networking:

```
Hardware Checksum Offloading: [x] Disable hardware checksum offload
```

Rebooted pfSense for that to take effect, then came back and enabled the DMZ interface for Suricata:

```
Enable: [x] Checking this box enables Suricata inspection on the interface.
Interface: DMZ (em2)
Description: DMZ
```

Left Blocking Mode disabled the whole session, wanted pure detection/logging first, no reason to risk dropping legit traffic before knowing what the rules actually catch.

## Getting real rules downloaded

The default categories that show up out of the box are just protocol sanity checks (dns-events, dhcp-events, kerberos-events, etc), not actual threat signatures. To get real detection rules had to go to Services > Suricata > Global Settings and enable ETOpen:

```
Install ETOpen Emerging Threats rules: [x] ETOpen is a free open source set of 
Suricata rules whose coverage is more limited than ETPro.
```

Then went to Updates tab and forced a download. Before that it showed:

```
Rule Set Name/Publisher          MD5 Signature Hash    MD5 Signature Date
Emerging Threats Open Rules      Not Downloaded        Not Downloaded
```

After updating, the hash actually populated, confirmed the rules downloaded.

Once that was done, went back into DMZ Categories and turned on a few rulesets to start with instead of enabling everything:

- emerging-scan.rules
- emerging-exploit.rules
- emerging-attack_response.rules

Repeated the whole interface setup for LAN too, since I wanted to see if traffic looks different depending on which side of the segment boundary you're watching from.

## First scan, nothing happened

Ran a basic version scan from Kali against Ubuntu on the DMZ ip, checked the Alerts tab, completely empty. Spent a while debugging this, went through a checklist:

- confirmed Suricata was actually running (green status, yes it was)
- confirmed categories were actually checked and saved
- confirmed traffic was crossing an interface Suricata was watching

Eventually went to check the raw log file directly under Logs View, picked eve.json, and got this:

```
Status/Result: Log file does not exist or that logging feature is not enabled.
Log File Path: Not Available
```

That was the actual problem. Went into the interface Settings tab (not Categories) and found EVE JSON Log was just unchecked the whole time:

```
EVE JSON Log: [x] Suricata will output selected info in JSON format to a single 
file or to syslog. Default is Not Checked.
EVE Output Type: FILE
EVE Log Alerts: [x] Suricata will output Alerts via EVE
```

Checked that box on both DMZ and LAN instances, saved, did a full stop then start on both interfaces to force them to actually pick up the change.

## Second attempt, aggressive scan

Ran an aggressive scan this time to generate more obvious signal:

```
$ nmap -sV -A -p- 192.168.2.100
Starting Nmap 7.98 at 2026-08-16 15:33 +0400
Nmap scan report for 192.168.2.100
Host is up (0.00079s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.18 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 03:ca:c3:c0:34:e6:6c:46:35:f3:da:df:73:a7:62:0e (ECDSA)
|_  256 39:31:f5:01:38:f6:0a:57:be:3f:5d:71:45:f3:1f:ac (ED25519)
Device type: general purpose
Running: Linux 4.X
OS CPE: cpe:/o:linux:linux_kernel:4
OS details: Linux 4.19 - 5.15
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 587/tcp)
HOP RTT     ADDRESS
1   0.95 ms pfSense.home.arpa (192.168.1.1)
2   1.62 ms 192.168.2.100
```

This time the Alerts tab actually showed something:

```
Date              Action  Pri  Proto  Class                            Src            SPort  Dst            DPort  GID:SID    Description
08/16/2026 15:33  !       3    ICMP   Generic Protocol Command Decode  192.168.2.100  0      192.168.1.100  9      1:2200025  SURICATA ICMPv4 unknown code
08/16/2026 15:33  !       3    ICMP   Generic Protocol Command Decode  192.168.1.100  8      192.168.2.100  9      1:2200025  SURICATA ICMPv4 unknown code
```

Took a bit to understand this one since the src/dst ports looked weird, turns out ICMP doesn't actually have ports at all, that's a TCP/UDP thing. What's showing in those columns for ICMP is actually the type and code fields instead, not real ports. This alert fired because nmap's aggressive/OS-fingerprinting probes send deliberately malformed ICMP packets to see how the target reacts, different OSes respond differently, that's literally how OS fingerprinting works. Suricata's decoder flagged the code value on the packet as not matching anything valid, hence "unknown code."

Important thing to note, this alert came from decoder-events.rules, which just checks if protocols look properly formed, not from emerging-scan.rules which is actual signature matching against known attack patterns. Two different detection mechanisms, this one just happened to be the one that caught something first.

Also checked eve.json directly around the same time and saw regular SSH connection logs from the scan, but those were just protocol logging, not alerts:

```
"in_iface":"em2","event_type":"ssh","src_ip":"192.168.1.100","src_port":39770,"dest_ip":"192.168.2.100","dest_port":22,"proto":"TCP"
"in_iface":"em2","event_type":"ssh","src_ip":"192.168.1.100","src_port":39772,"dest_ip":"192.168.2.100","dest_port":22,"proto":"TCP"
```

This is the difference between logging and alerting basically. Suricata logs any protocol traffic it recognizes regardless of whether it's malicious, SSH connecting normally to port 22 isn't inherently suspicious so it just gets recorded, doesn't trigger an alert. Only stuff that actually matches a loaded rule becomes an alert.

## Third test, quiet SYN scan

This was the actual point of the session, comparing loud vs quiet scanning.

```
$ nmap -sS -p 22,80,443 192.168.2.100
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-16 15:45 +0400
Nmap scan report for 192.168.2.100
Host is up (0.00097s latency).

PORT    STATE  SERVICE
22/tcp  open   ssh
80/tcp  closed http
443/tcp closed https

Nmap done: 1 IP address (1 host up) scanned in 0.63 seconds
```

Checked the Alerts tab after this, nothing showed up at all. Checked eve.json, only found an SSH protocol entry for port 22, same as before, nothing for 80 or 443. Checked http.json specifically too, completely empty.

Makes sense once you think about it. Port 22 has something actually listening, sshd, so a real connection attempt happens and gets logged. Ports 80 and 443 are closed, nothing's listening, the SYN just gets an immediate RST back, there's no actual protocol conversation happening at all for any parser to log. SYN scan also doesn't send any of those weird fingerprinting probes the aggressive scan does, so no decoder anomaly either.

So end result comparing the two scans:

- Aggressive scan: triggered a real alert (ICMP decoder anomaly from OS fingerprinting probes), plus normal SSH protocol logs
- SYN scan: zero alerts, only a plain SSH log entry for the one open port, literally nothing logged anywhere for the closed ports

That's basically proof that quiet reconnaissance techniques are built specifically to avoid tripping signature and anomaly based detection, while loud aggressive scanning generates way more surface area for something to catch. Good hands on demonstration of something that's usually just explained in theory.

## What I'd do next with an alert like this

Talked through this conceptually even though it didn't really apply to the ICMP alert I got, since I already knew the cause was my own scan. In a real scenario the flow would be:

1. Triage it first, figure out if it's a true positive and whether it's actually concerning or expected
2. Decide whether to suppress it (if it's noise you expect regularly, like your own vuln scanner), escalate it, or just let it sit logged
3. Only look at writing a new custom rule if there's a real detection gap, something with no rule coverage at all. In this case the rule already existed and did its job, no reason to write anything new.