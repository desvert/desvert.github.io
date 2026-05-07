---
layout: post 
title: "Adding the Catalyst" 
date: 2026-05-05 
categories: - blog 
tags: - homelab - networking - cisco - ccna - vlan - linux 
excerpt: ""
---

# Adding the Catalyst

The C7 is a capable router but a dumb switch. All four LAN ports sit on a flat bridge with no visibility into what's crossing them and no way to segment traffic between them. That limitation was acceptable while the lab was just a router and a home server. It stopped being acceptable once the plan involved IoT devices, a dedicated monitoring stack, and a port that can mirror everything to a capture interface.

The Cisco Catalyst WS-C2960C-12PC-L was the answer. Twelve FastEthernet PoE ports, two GigabitEthernet uplinks, 124W of PoE budget, running IOS 12.2(55)EX3 with the lanbase license. L2 only, no routing. Exactly what this setup needs. Picked up used. An IOS upgrade was considered and deferred: 12.2 handles VLANs and SPAN without issue, and pulling images legally requires a Cisco service contract.

This is also where the lab starts looking like something you'd study for CCNA on, because that's exactly what it is. The switch config in this post is real cert-prep, not simulation.

---

## Getting Console Access

The Catalyst has no out-of-band management until it's configured. That means a USB console cable before anything else. The FTDI FT232-based cable works; `picocom` is the cleaner tool for serial console work compared to `screen`, with explicit baud rate flags and cleaner exit behavior.

WSL2 adds a step. USB serial devices don't pass through to the WSL environment automatically. The `usbipd` tool handles this: attach the FTDI adapter from a Windows terminal before opening the console session in WSL. Without that step, `picocom` sees nothing.

```bash
# Windows (PowerShell, as admin)
usbipd attach --wsl --busid <busid>

# WSL
picocom -b 9600 /dev/ttyUSB0
```

Baud rate is 9600 8N1, standard for IOS console.

---

## Baseline Configuration

The switch arrived with a partial config left by the previous owner: an SVI at `192.168.1.11`, the HTTP server enabled, and VTY lines configured with `login` but no password set, meaning Telnet access was open to anyone on the network with no authentication. The first order of business was a full baseline before connecting it to anything.

```
hostname C7SW
no ip domain-lookup
enable secret <redacted>
service password-encryption
no ip http server
no ip http secure-server
banner motd ^Authorized access only^
```

Console and VTY lines locked down with `login local`, a named user account, and SSH v2 only. IP default gateway set to `192.168.10.1` (the C7) so the management SVI can reach the rest of the network.

---

## The SSH Crypto Problem

Modern OpenSSH rejects IOS 12.2's algorithms by default. Attempting to SSH into the switch produced a connection failure with no useful message from the client side. Three separate flags were required, each surfacing as a distinct error:

```
# ~/.ssh/config
Host c7sw
    HostName 192.168.10.11
    User admin
    KexAlgorithms +diffie-hellman-group1-sha1
    HostKeyAlgorithms +ssh-rsa
    Ciphers +aes256-cbc
```

The key exchange error came first, then the host key algorithm rejection, then the cipher mismatch. Each one required a separate attempt to surface. The switch isn't broken. IOS 12.2 predates modern crypto standards by over a decade. The `~/.ssh/config` entry makes the workaround permanent so it doesn't need to be remembered.

---

## VLAN Design

VLAN 1 was skipped entirely. It's the default native VLAN on 802.1Q trunks, which makes it the outer tag in a double-tagging attack against any port carrying tagged traffic. Easier to leave it empty than to harden against it.

The scheme:

- **VLAN 10** — trusted LAN (`192.168.10.x`), wired ports and `ether` WiFi
- **VLAN 20** — lab (`192.168.20.x`), `labnet` WiFi, ESP32s and sensors
- **VLAN 30** — reserved stub, no devices yet
- **VLAN 99** — black hole, all unallocated ports land here with no routing and no internet access

VLAN 99 is a deliberate security posture. An empty port on an unmanaged switch is an open door. An empty port assigned to VLAN 99 and shut down is not.

---

## Configuring the Switch

VLANs created, ports named and assigned. All unallocated FastEthernet ports assigned to VLAN 99 and shut down. `Fa0/1` configured as a VLAN 10 access port for `argus` -- the t620 thin client that will run the monitoring stack -- with `Fa0/2` reserved as its dedicated capture interface. `Gi0/1` configured as an 802.1Q trunk to the C7 carrying VLANs 10 and 20. Management SVI moved from VLAN 1 to VLAN 10 at `192.168.10.11`.

```
vlan 10
 name TRUSTED
vlan 20
 name LAB
vlan 30
 name RESERVED
vlan 99
 name BLACKHOLE

interface range Fa0/3 - 12
 switchport mode access
 switchport access vlan 99
 shutdown

interface Fa0/1
 description argus
 switchport mode access
 switchport access vlan 10
 no shutdown

interface Gi0/1
 description uplink-C7
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 no shutdown

interface vlan 10
 ip address 192.168.10.11 255.255.255.0
 no shutdown

ip default-gateway 192.168.10.1
```

---

## The C7 Factory Reset

Configuring the C7's LAN ports to pass tagged VLAN traffic hit the same failure mode described in [Post 1](https://desvert.github.io/blog/2026/04/26/homelab-1-taking-control.html): manipulating the internal bridge via `swconfig` brought down all ports simultaneously. Another factory reset.

The difference from Post 1: the lesson held. WiFi was already enabled as a fallback before any switch configuration was attempted. Access to the C7 survived the reset and reconfiguration proceeded without needing physical access to the router.

After the reset, the decision was made to stop fighting the C7's internal switch entirely. The C7 becomes a pure router: one trunk port out of LAN3 to the Catalyst's `Gi0/1`. The Catalyst handles all switching, which is what a managed switch is for.

---

## The VLAN Migration

Moving `thing` from `192.168.1.111` to `192.168.10.111` and `quantum` from `192.168.1.136` to `192.168.10.136` required updates in several places: DHCP reservations on the C7, port forwarding rules for the Bedrock server (19132/UDP), TCP wrappers on `thing` (which needed `192.168.10.x` added to `/etc/hosts.allow`, the third time this has come up across the series), and `~/.ssh/config` on `quantum`.

The `quantum` static reservation targeting `.222` never took. Windows held `.136` through the migration the same way it had before. It stopped mattering once everything was stable at `.136`, so the reservation was updated to match reality rather than fighting it.

Two post-migration surprises. The Bedrock server stopped working after `thing` reconnected to the Verizon WiFi network during the migration, making `wlp2s0` the primary default route. Inbound traffic was arriving via `enp1s0` on `192.168.10.x`; outbound Xbox Live authentication was leaving via WiFi on `192.168.0.x`. The asymmetric path broke the connection handshake. `ip route show` made it visible immediately. The fix was permanent:

```bash
sudo nmcli radio wifi off
sudo systemctl disable wpa_supplicant
```

A wired server has no business having WiFi enabled. The second surprise was simpler: Minecraft port forwarding was closed entirely. Every player is on the local network. There's no reason to leave 19132/UDP open to the internet.

---

## SPAN

SPAN (Switched Port Analyzer) mirrors traffic from one port to another. One port on the Catalyst sees everything crossing a given source port and sends a copy to a designated monitor port. That monitor port connects to a capture interface on `argus`. Every packet crossing `Fa0/1`, in both directions, becomes visible to whatever is listening on `capture0`.

`argus` connects to the Catalyst on two ports: `Fa0/1` carries its normal VLAN 10 network traffic, and `Fa0/2` connects to `capture0`, a dedicated USB NIC (ASIX AX88179) used exclusively for packet capture.

```
monitor session 1 source interface Fa0/1 both
monitor session 1 destination interface Fa0/2
```

`Fa0/2` was already up from the earlier port configuration. Once the `monitor session` command ran, IOS moved it into monitoring mode automatically. `show interfaces Fa0/2 status` confirms it:

```
Port      Name      Status       Vlan       Duplex  Speed Type
Fa0/2               monitoring   99         a-full  a-100 10/100BaseTX
```

Confirmation that traffic was flowing to `capture0` on `argus`:

```bash
sudo tcpdump -i capture0
```

Traffic appeared immediately.

---

## What Came Out of Building It

The SSH crypto problem on IOS 12.2 took longer than it should have because the OpenSSH error messages are not specific about which negotiation step failed. The fix, once the flags were known, was a single `~/.ssh/config` block. Check `ssh -vvv` output early rather than guessing at which algorithm is the problem.

The C7 factory reset was frustrating but not a surprise. Same failure mode as Post 1, triggered the same way. The difference was that the WiFi fallback was in place before it happened. The architectural outcome was worth it: the C7 does routing, the Catalyst does switching, and each piece does one job cleanly.

---

## Where This Is Going

`argus` is cabled, `capture0` is live, and the SPAN session is running. The next post covers the monitoring stack on the t620: Rocky Linux, Prometheus, Grafana, Mosquitto, Telegraf, and eventually Zeek and Suricata consuming everything `capture0` sees.