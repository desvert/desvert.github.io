---
layout: post
title: "Homelab 2: First Services Up"
date: 2026-04-30
categories:
    - blog
tags:
    - homelab
    - networking
    - docker
    - minecraft
    - linux
excerpt: ""
---

# First Services Up

The network foundation from the last post is infrastructure. It doesn't do anything visible until something runs on top of it. The first thing I wanted to answer after getting the C7 stable was a practical question: can this setup actually run a household server without being a maintenance burden?

`thing` is a Toshiba laptop repurposed as a headless server. Six gigabytes of RAM, Ubuntu 24.04, connected via ethernet to LAN1 on the C7 and running continuously. It handles Jellyfin for video, Navidrome for music, Filebrowser for remote file access, and a Minecraft Bedrock server for the family. Pi-hole for DNS filtering has been running on it since before the C7 came into the picture. Jellyfin, Navidrome, Filebrowser, and Portainer run in Docker. Pi-hole and the Bedrock server run natively. Pi-hole's reasons are covered in [the last post](https://desvert.github.io/blog/2026/04/26/homelab-1-taking-control.html). Bedrock's are covered below.

Getting these services running was mostly unremarkable. What wasn't unremarkable were the network failures that came up along the way — each one a variation on the same theme.

---

## The Stack

The current service configuration on `thing`:

| Service | Image | Port | Notes |
|---|---|---|---|
| Jellyfin | `jellyfin/jellyfin` | 8096 | Media server |
| Navidrome | `deluan/navidrome` | 4533 | Music streaming |
| Filebrowser | `filebrowser/filebrowser` | 8080 | File management |
| Portainer | — | — | Docker management |
| Pi-hole | native install | 53 | DNS filtering |
| Bedrock server | native + systemd + tmux | 19132/udp | Minecraft |

The Bedrock server runs as a systemd service (`bedrock-direct.service`) that launches the server binary directly on the host inside a `tmux` session. It replaced an earlier Docker-based setup that was pulled after a version mismatch during a reboot caused the containerized server to fail to start — the full story is in [this post](https://desvert.github.io/blog/2026/03/24/home-server-recovery-when-a-simple-reboot-took-everything-down.html). The transition left behind a companion unit, `bedrock-rules.service`, that was written to apply game rules after startup by sending commands to a container called `mc-bedrock`. That container no longer exists, so `bedrock-rules.service` fails on every boot. The failure is benign, but it's on the list.

---

## Bedrock Through Double-NAT

Getting Minecraft Bedrock accessible from outside the LAN required port forwarding through two routers in sequence. The path is:

```
Xbox (internet)
    |
Verizon gateway: 19132/udp → 192.168.0.195  (C7 WAN IP)
    |
C7: 19132/udp → 192.168.10.111  (thing)
    |
bedrock-direct.service
```

Both rules have to exist. If either is missing, the connection fails silently. The C7 rule was lost during a factory reset that happened in the middle of the VLAN migration work and had to be recreated — which is why the server appeared to break during that migration without any obvious cause.

Double-NAT adds one forwarding hop but doesn't change the fundamental problem: when a port forward is missing, you get nothing, and both routers are candidates. Checking the chain from the outside in is faster than guessing.

---

## The Asymmetric Routing Failure

After the VLAN migration, the Bedrock server stopped working. The port forwarding rules were in place. The service was running. Packets were arriving at `thing` on `enp1s0` via `192.168.10.x`. But Xbox Live authentication was failing.

The issue was the outbound path. `thing` had reconnected to the Verizon WiFi network at some point, and `wlp2s0` had become the primary default route. Outbound traffic was leaving via WiFi with source IP `192.168.0.111` instead of going back out through the C7 the way the inbound path expected. The handshake was asymmetric: packets in via `192.168.10.x`, packets out via `192.168.0.x`. Xbox Live doesn't tolerate that.

```bash
ip route show
```

That command is where the diagnosis started. The default route pointed at the Verizon gateway via `wlp2s0`. Once that was visible, the fix was obvious:

```bash
sudo nmcli radio wifi off
sudo systemctl disable wpa_supplicant
```

A wired server has no reason to have WiFi enabled. Disabling it permanently removes the failure mode. The CCNA material on routing and traffic flow is what made this readable immediately: asymmetric routing is a concept you study in the abstract, and it's clarifying when you see it cause an actual problem.

---

## TCP Wrappers, Twice

`thing` has `/etc/hosts.deny` set to `ALL: ALL`. SSH access is controlled entirely through `/etc/hosts.allow`. That configuration caught me off guard twice during the same series of infrastructure changes.

The first time was after adding the C7. The allowlist only covered `192.168.0.x`, which was the subnet `thing` had always been accessed from. Connecting from the C7's LAN side (`192.168.1.x`) was a new source and got rejected silently at the connection handshake. The SSH journal made it obvious:

```
refused connect from 192.168.1.136 (192.168.1.136)
```

The second time was after the VLAN migration. The same problem, new subnet. `192.168.10.x` wasn't allowlisted.

Both times the fix was the same: add the new subnet to `/etc/hosts.allow`. TCP wrappers evaluate live, so no service restart is needed. The current allowlist:

```
sshd : 192.168.0. 192.168.1. 192.168.10.
```

One detail that isn't in most documentation: the file needs a trailing newline. Without it, the last rule may not parse correctly.

The pattern is predictable now. Any time `thing` becomes accessible from a new subnet, `/etc/hosts.allow` needs updating before anything else is attempted. It's a fast fix, but only if you know to look there.

---

## Check Your Bind Addresses

Filebrowser had been running before the C7 was added, when `thing` was directly on the Verizon network. It was bound to `192.168.0.111`. After the routing change, it was inaccessible from the C7's LAN side — not because anything was broken, but because it was listening on an address that was no longer the right one.

Updating the bind address to `192.168.1.111` (now `192.168.10.111` post-VLAN) fixed it. The lesson is simple: after any IP or routing change, check what your services are actually bound to. `ss -tlnp` shows listening sockets and their addresses. It takes thirty seconds and catches this class of problem immediately.

---

## What Came Out of Building It

The asymmetric routing failure was the most instructive problem in this post. Everything looked correct at the service level. The port forward was in place, the service was running, packets were arriving. The failure was in the outbound path, and you can't see that without looking at the routing table. `ip route show` should be the first command after any network change that breaks a service, before touching anything else.

TCP wrappers as a failure mode is now something I anticipate rather than troubleshoot. The configuration is deliberate and it works, but it requires active maintenance every time the network topology changes. That's a reasonable tradeoff for a box that's SSH-accessible and running continuously.

---

## Where This Is Going

`thing` is stable as a household server. The next step is adding the Cisco Catalyst WS-C2960C-12PC-L, which enables the VLAN segmentation that `labnet` and the IoT devices need. The Bedrock server is already on `192.168.10.x` post-migration. The monitoring stack and the security layer come after the switch.

The next post covers the Catalyst setup: IOS initial configuration, VLAN design, and the SPAN port that will eventually feed the monitoring stack.