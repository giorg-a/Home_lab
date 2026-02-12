Day 4 — Suricata IDS Behavior
Reconnaissance vs Exploitation Detection

Overview

— Today I wanted to understand how Suricata inside Security Onion reacts to different attacker behaviors
— The hardest part of this lab was troubleshooting the monitoring interface and log pipeline
— Once detection started working, I tested loud scanning, stealth scanning, and exploitation to compare visibility

Loud Scan

— I executed an aggressive Nmap scan (nmap --top-ports 1000 -T4 -A <target>)
— Suricata generated alerts almost immediately
— It was able to fingerprint Nmap and detect scripting engine usage
— Multiple service probing alerts were triggered
— The IDS also detected my attacking machine’s hostname through DHCP traffic

This showed me how noisy reconnaissance creates strong fingerprints that are easy to attribute.

Stealth Scan

— I performed a slower SYN scan (nmap -sS -T2 <target>)
— Alerts were still generated, but attribution changed
— Instead of confirming Nmap usage, Suricata reported suspicious port interactions
— Tool fingerprints disappeared, but behavioral detection remained
— The attacker hostname was still visible due to DHCP broadcasts

This made it clear that stealth reduces attribution clarity but does not eliminate visibility.

Exploitation

— I targeted the vulnerable vsftpd service using Metasploit
— The exploit successfully provided a root shell
— Suricata did not generate a clear exploit-specific signature
— Network activity was logged, but no explicit “exploit detected” alert appeared

This highlighted a limitation of signature-based detection. Successful compromise does not always guarantee a clear IDS alert.

Conclusion

— Loud scans are easy to detect and attribute
— Stealth scans reduce identifiable fingerprints but remain visible
— Exploitation can succeed even when explicit exploit signatures are not triggered
— Monitoring effectiveness depends on traffic patterns, rule coverage, and attacker behavio