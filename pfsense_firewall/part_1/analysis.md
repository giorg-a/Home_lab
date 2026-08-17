# Network Segmentation and Host Firewall

## What happened this session

Ripped out Security Onion since running it headless over CLI just isn't how it's meant to be used, the whole point of that tool is the web dashboard. Also pulled OpenVAS.

Bigger thing though, I rebuilt the whole lab network. Before this it was flat, Kali and Ubuntu just sat on the same Host-only network and could talk to each other directly, no firewall in the middle. Deployed pfSense as an actual router/firewall between them instead.

## pfSense setup

Gave pfSense 3 adapters:

- NAT, for internet access
- LAN (`lab_lan`), Kali lives here
- DMZ (`lab_dmz`), Ubuntu lives here

So now Kali and Ubuntu are on separate segments and have to go through pfSense to reach each other, instead of just seeing each other on the same switch. Basically the same idea as a real DMZ, exposed/less trusted stuff gets its own segment instead of sitting flat with everything else.

Assigned the interfaces in pfSense (WAN/LAN/OPT1, renamed OPT1 to DMZ), set the DMZ interface to a static IP of 192.168.2.1/24 with no upstream gateway since it's just a local interface, and turned on DHCP for the DMZ range (192.168.2.100 to 192.168.2.200).

Result, checked with `ip addr` on both boxes:

Ubuntu, DMZ side:

```
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:61:34:81 brd ff:ff:ff:ff:ff:ff
    inet 192.168.2.100/24 metric 100 brd 192.168.2.255 scope global dynamic enp0s3
```

Kali, LAN side:

```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:d0:2c:11 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.100/24 brd 192.168.1.255 scope global dynamic noprefixroute eth0
```

By default pfSense blocks everything between interfaces unless you tell it not to, so I added a Pass rule on the LAN interface (source any, dest any, protocol any) to let Kali actually reach the DMZ.

## Verification

Pinged both directions.

Kali to Ubuntu:

```
$ ping 192.168.2.100
PING 192.168.2.100 (192.168.2.100) 56(84) bytes of data.
64 bytes from 192.168.2.100: icmp_seq=1 ttl=63 time=0.450 ms
64 bytes from 192.168.2.100: icmp_seq=2 ttl=63 time=1.52 ms
64 bytes from 192.168.2.100: icmp_seq=3 ttl=63 time=1.47 ms
64 bytes from 192.168.2.100: icmp_seq=4 ttl=63 time=1.50 ms
64 bytes from 192.168.2.100: icmp_seq=5 ttl=63 time=1.53 ms

--- 192.168.2.100 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4161ms
```

Ubuntu to Kali:

```
ubuntu@ubuntu:~$ ping 192.168.1.100
PING 192.168.1.100 (192.168.1.100) 56(84) bytes of data.
64 bytes from 192.168.1.100: icmp_seq=1 ttl=63 time=0.440 ms
64 bytes from 192.168.1.100: icmp_seq=2 ttl=63 time=1.90 ms
64 bytes from 192.168.1.100: icmp_seq=3 ttl=63 time=1.97 ms
64 bytes from 192.168.1.100: icmp_seq=4 ttl=63 time=1.58 ms

--- 192.168.1.100 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3072ms
```

Both work fine, ttl is 63 instead of 64 which makes sense since there's now a hop through pfSense in between.

Ran an nmap scan from Kali against Ubuntu:

```
$ nmap -Pn -A -sV -p 22,80,443 192.168.2.100
Nmap scan report for 192.168.2.100
Host is up (0.0010s latency).

PORT    STATE  SERVICE VERSION
22/tcp  open   ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.18 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 03:ca:c3:c0:34:e6:6c:46:35:f3:da:df:73:a7:62:0e (ECDSA)
|_  256 39:31:f5:01:38:f6:0a:57:be:3f:5d:71:45:f3:1f:ac (ED25519)
80/tcp  closed http
443/tcp closed https

Network Distance: 2 hops

TRACEROUTE (using port 443/tcp)
HOP RTT     ADDRESS
1   0.25 ms pfSense.home.arpa (192.168.1.1)
2   0.53 ms 192.168.2.100
```

Only port 22 showed open, 80 and 443 closed, exactly what should be exposed. Traceroute shows the path going through pfSense first then landing on Ubuntu, so the routing is actually doing what it's supposed to.

Also pulled up the firewall logs on pfSense (Status > System Logs > Firewall) and filtered by source/dest IP while running the scan. Sample of what showed up:

```
Action  Time       Interface  Rule                          Source                  Destination
Pass    17:29:45   LAN        USER_RULE (1786796232)         192.168.1.100:37920     192.168.2.100:443
Pass    17:29:45   LAN        USER_RULE (1786796232)         192.168.1.100:37920     192.168.2.100:22
Pass    17:29:45   LAN        USER_RULE (1786796232)         192.168.1.100:37920     192.168.2.100:80
Block   17:29:46   LAN        Default deny rule IPv4          192.168.1.100:33025     192.168.2.100:22
Block   17:29:46   LAN        Default deny rule IPv4          192.168.1.100:33027     192.168.2.100:80
```

Could see my Pass rule matching allowed traffic, and other stuff getting caught by the "Default deny rule IPv4" that pfSense adds automatically. Good confirmation the firewall's actually doing something instead of just letting everything through.

## The nft mess

Earlier in the week I had built out an nftables ruleset on Ubuntu, default deny inbound, only allowing loopback, established/related connections, and SSH on port 22. Final state looked like this:

```
table inet filter {
    chain input {
        type filter hook input priority filter; policy drop;
        ct state established,related accept
        iifname "lo" accept
        tcp dport 22 accept
    }

    chain output {
        type filter hook output priority filter; policy accept;
    }
}
```

That was working fine at the time.

After I rebuilt the network with pfSense though, Kali couldn't connect to Ubuntu anymore. Turned out that old nft ruleset was still active and was built around the old flat network, it wasn't accounting for the new routed path through pfSense. That was the actual problem, not a pfSense misconfig.

Fixed it by just nuking the whole ruleset:

```bash
sudo nft flush ruleset
```

That clears every table, chain, and rule in one shot, back to zero filtering. Connectivity from Kali came back right after.

So right now Ubuntu has no host level firewall at all, it's fully relying on pfSense at the network level for now. Need to rebuild the nft ruleset properly for the new DMZ setup and actually persist it this time (/etc/nftables.conf + enable the nftables service) so it survives a reboot, since last time it just lived in kernel memory and vanished.
