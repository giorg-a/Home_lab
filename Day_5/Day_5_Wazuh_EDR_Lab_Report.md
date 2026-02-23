# Day 5 --- Endpoint Detection & Response (Wazuh) Lab

## Objective

In previous labs, the focus was mainly on **network detection tools**.\
However, attacks do not only exist on the network --- they also occur
**inside endpoints (machines)**.

The goal of this lab was to understand how **Endpoint Detection and
Response (EDR)** works by monitoring activity directly on a host system.

To achieve this, the Wazuh platform was deployed and tested against
different authentication scenarios, including brute-force attacks.

------------------------------------------------------------------------

## Lab Environment

**EDR Platform:** Wazuh\
**Agent Host (Victim Machine):** Ubuntu Server\
**Attacking Machine:** Kali Linux\
**Additional Target Attempted:** Metasploitable (agent not supported due
to outdated system)

### Architecture

-   Dedicated virtual machine running **Wazuh Manager**
-   Wazuh agent installed on Ubuntu victim machine
-   Attacks initiated from Kali Linux over SSH

------------------------------------------------------------------------

## Wazuh Overview

Wazuh operates by deploying **agents** on monitored endpoints.\
Each agent continuously monitors system activity such as:

-   Authentication attempts
-   File integrity changes
-   Process activity
-   System anomalies
-   Port interaction
-   Security policy violations

All collected telemetry is sent to the **Wazuh Manager**, where it is
analyzed and converted into alerts.

------------------------------------------------------------------------

## Deployment Challenges

Setting up Wazuh and configuring agents required the most time in this
lab.

Initially, an attempt was made to deploy an agent on Metasploitable, but
the system is extremely outdated and not compatible with modern Wazuh
agents.\
To resolve this, a fresh Ubuntu Server instance was used instead.

This highlighted an important real-world lesson:

Legacy systems often create monitoring limitations and security
visibility gaps.

------------------------------------------------------------------------

## Attack Scenarios & Observations

### 1. Failed Login --- Existing User (Wrong Password)

Test: Attempted SSH login using a valid username but incorrect password.

Wazuh Detection: - Alert generated immediately - MITRE tactic:
Credential Access - MITRE tactic: Lateral Movement - Authentication
failure recorded

Insight: Wazuh clearly identifies credential misuse attempts even when
the account exists.

------------------------------------------------------------------------

### 2. Login Attempt --- Non-Existent User

Test: Attempted SSH login using a username that does not exist.

Wazuh Detection: - Same tactical classification (Credential Access /
Lateral Movement) - Alert specifically indicated attempt to authenticate
with non-existent account

Insight: Wazuh distinguishes between invalid password attempts and
account enumeration attempts.

------------------------------------------------------------------------

### 3. Fast SSH Brute Force (Hydra)

Method: - Generated 50 random passwords (including correct password) -
Executed Hydra brute force with rapid attempts

Result: - Massive alert spike - Approximately 10 pages of alerts
generated in seconds - Repeated authentication failures clearly
visible - Brute-force pattern extremely obvious

Detection Quality: Very high --- even basic monitoring would detect this
attack.

Key Takeaway: Fast brute force attacks create strong noise signatures
and are easy to detect.

------------------------------------------------------------------------

### 4. Slow / Stealth Brute Force

Method: - Same brute force approach - Added delay of 30 seconds between
login attempts

Result: - Alert volume dramatically reduced - Activity appeared
quieter - Pattern spread over time instead of clustering

Detection Reality: This type of attack could blend into normal
background noise in large environments where thousands of alerts are
generated daily.

However: A skilled analyst would still identify the repetitive pattern
over time.

------------------------------------------------------------------------

## Security Lessons Learned

-   Endpoint monitoring provides visibility that network monitoring
    alone cannot.
-   Authentication activity is a critical behavioral indicator.
-   Fast brute force attacks are easy to detect due to alert clustering.
-   Slow brute force attacks are stealthier but still traceable through
    behavioral analysis.
-   Legacy systems reduce monitoring capability.
-   EDR tools are essential for internal host visibility.

------------------------------------------------------------------------

## Why This Lab Matters

This lab did not focus on exploitation.

Instead, it focused on something more important:

Understanding one of the most fundamental defensive security mechanisms
--- endpoint monitoring.

Security is not only about blocking attacks.\
It is about observing behavior, recognizing patterns, and detecting
anomalies.

------------------------------------------------------------------------

## Screenshots

All visual evidence of alerts, attack execution, and system responses
are available in:

/screenshots

------------------------------------------------------------------------

## Conclusion

Wazuh demonstrated strong visibility into authentication behavior and
host activity.\
The platform successfully detected:

-   Incorrect password attempts
-   Invalid account usage
-   High-volume brute force attacks
-   Slow stealth brute force patterns

This lab reinforced the importance of EDR in modern defensive security
architecture.

Understanding detection is the foundation of effective defense.
