# New Architecture

This is the rebuilt version of my home lab. The old one (see [Old_architecture](../Old_architecture/)) worked, but it was flat and limited, with no management network and no way to test from outside.

In this version, pfSense runs with 4 interfaces: WAN, LAN, DMZ, and MGMT. Kali sits on LAN and Ubuntu Server sits on DMZ, each on its own internal network. The pfSense web UI is only reachable through a separate host-only MGMT interface, and a physical Parrot OS laptop on my home network tests the lab from outside through the WAN side.

I kept the old lab in the repo on purpose, so you can see how the setup improved over time.
