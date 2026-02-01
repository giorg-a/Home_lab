Day 2 — Loud vs Stealth Nmap Scans
Network Behavior Analysis
Overview

— Day 2 focused on analyzing how different Nmap scanning strategies behave on the network
— The goal was to compare loud and stealth-ier scans from a defensive and monitoring perspective
— Wireshark was used to visualize packet volume, timing, and interaction with background traffic

Lab Setup

— Attacker machine: Kali Linux (192.168.56.105)
— Target machine: Metasploitable (192.168.56.109)
— Traffic analysis tool: Wireshark
— All systems were connected to the same isolated internal network

Loud Scan

— A loud Nmap scan (nmap -sS -sV -A -T4 -p- <TARGET>
) was executed to maximize speed and information gathering
— The scan generated an extremely high volume of traffic
— Normal background network activity was overwhelmed and visually drowned out
— Approximately 133,000 packets were captured during the scan
— Only around 1,300 packets represented successful connections (SYN-ACK)
— The scan traffic dominated the capture and was very easy to identify
— Despite being highly detectable, the scan provided a large amount of useful information about the target

Stealth Scan

— A stealth-ier Nmap scan was performed (nmap -sS -Pn -T1 --top-ports 100 <TARGET>
) using slower timing and a limited port scope
— Packet volume was significantly lower than the loud scan
— Traffic was spread out over time instead of appearing in bursts
— Scan traffic blended into normal background activity such as ARP, DNS, and SSDP
— The scan took approximately 1,652 seconds (~27 minutes) to complete
— Much less information was gathered compared to the loud scan
— Detection required focused filtering and correlation rather than visual inspection

Loud vs Stealth

— Loud scans are fast, noisy, and information-rich
— Stealth scans are slow, quiet, and less visually distinguishable
— Loud scans create strong contrast with baseline traffic
— Stealth scans reduce contrast rather than eliminating visibility

Loud scans look like noise.
Stealth scans look like background activity.

Final Notes

— Most scan traffic consists of noise, not meaningful connections
— Only a small portion of packets represent real exposure
— Stealth does not mean invisible, it means less noticeable
— Understanding baseline network behavior is critical for effective detection


