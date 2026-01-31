# Day 1 – Reconnaissance to Exploitation (Network Behavior)

## Overview
Day 1 focused on understanding **how attacks look on the network** from a defender’s perspective.

The goal was to observe reconnaissance behavior, exploitation behavior, and how these phases differ visually and logically.

---

## Lab Setup
- **Kali Linux (Attacker)** – `192.168.56.105`
- **Metasploitable2 (Victim)** – `192.168.56.109`
- **Security Onion** – Network sensor / SIEM

All machines were on the same isolated internal network, with Security Onion monitoring all traffic.

---

## Reconnaissance
A basic scan was used to identify open ports and services on the victim.

**Observed behavior**
- Many ports scanned very quickly
- Short-lived connections
- No data exchange

**Defender view**
- High volume of traffic from one source
- Chaotic and automated patterns

**Takeaway**
Reconnaissance traffic is **loud and easy to detect**.

---

## Exploitation
After reconnaissance, a vulnerable FTP service was targeted:
- **vsftpd 2.3.4 backdoor**

**Observed behavior**
- Traffic focused on a single service
- Longer connections
- Data exchanged in both directions

The exploitation succeeded and resulted in **root-level access**.

---

## Recon vs Exploit
- **Recon:** wide, fast, chaotic  
- **Exploit:** narrow, ordered, conversational  

Recon looks like noise.  
Exploit looks like a conversation.

---

## Final Notes
This lab demonstrated a full attack flow:
**recon → exploitation → compromise**

The key lesson was learning to identify attacks based on **behavior and context**, not just tools.
