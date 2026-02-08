# Day 3 – OpenVAS Setup & Vulnerability Scanning

## Overview

Day 3 focused on understanding how vulnerability scanning works and learning what CVEs are.

My goal was to set up OpenVAS, run my first scan, and focus on learning one CVE at a time so I could understand it better instead of trying to learn too many at once.

---

## Lab Setup

The lab included Kali Linux as the scanner, Msploitable2 as the vulnerable target, and Wireshark to monitor network traffic. All machines were connected inside a private lab network.

---

## OpenVAS Setup

Setting up OpenVAS was the hardest part of this lab. The installation and configuration took around two days because of different setup problems. I had to troubleshoot several issues before the scanner worked correctly.

---

## Vulnerability Scanning

After finishing the setup, I ran a vulnerability scan against the target system. The scan took about forty minutes and tested multiple services automatically to look for security weaknesses.

---

## Network Traffic Observation

I used Wireshark during the scan to watch the network traffic. The scan created more than 300,000 packets and showed how scanners communicate with different services while testing for vulnerabilities.

---

## Key Finding – Apache Tomcat

One of the main findings was Apache Tomcat using default credentials. The scan showed that the username and password "tomcat" allowed access to admin panels. This is dangerous because someone could log in and possibly take control of the server.

---

## What is Apache Tomcat

Apache Tomcat is a server that runs websites and web applications built with Java. It also has admin panels that help manage the server. If these panels are not secured properly, they can become security risks.

---

## CVE Understanding

CVEs are ID numbers given to known security problems. One service can have multiple CVEs because software can have different types of weaknesses in different parts of it.

---

## Final Notes

This lab helped me understand how vulnerability scanning works and how misconfigurations can expose systems. It also helped me learn the importance of studying one vulnerability at a time to fully understand it.

