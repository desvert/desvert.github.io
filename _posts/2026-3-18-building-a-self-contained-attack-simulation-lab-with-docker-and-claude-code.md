---
layout: post
title: "Building a Self-Contained Attack Simulation Lab with Docker and Claude Code"
date: 2026-03-18
categories:
    - blog
tags:
    - docker
    - network-forensics
    - dfir
    - suricata
    - zeek
    - mcp
    - homelab
excerpt: "A Docker Compose lab that spins up a vulnerable target, runs 16 phases of automated attacks, and captures everything -- PCAP, Suricata alerts, and Zeek logs -- ready for analysis with the MCP toolchain."
---

# Building a Self-Contained Attack Simulation Lab with Docker and Claude Code

The SOC triage lab I wrote about a couple weeks ago is built to analyze network forensics data -- PCAPs, Suricata alerts, Zeek logs. What it doesn't do is generate that data. For the analysis toolchain to be useful for practice, I need realistic, repeatable samples to feed it. I could download example PCAPs from the internet, but that only goes so far. I wanted something I could run on demand, control the attack types, and extend over time.

So I built `mcp-test-env`: a Docker Compose stack that spins up a vulnerable target, runs a scripted sequence of attacks against it, and captures everything to disk. One command starts the whole thing. When the attacker finishes its run, you have a PCAP, Suricata `eve.json` with IDS alerts, and a full set of Zeek NSM logs -- all from the same traffic, ready to load into the analysis tools.

This post covers the design decisions that weren't obvious, and how I used Claude Code to accelerate the implementation without losing ownership of the architecture.

---

## What It Does

The stack is five containers coordinated through Docker Compose:

- **victim** -- Metasploitable2, an intentionally vulnerable Ubuntu 8.04 image with vsftpd 2.3.4, Apache/DVWA, UnrealIRCd, Tomcat, VNC, and more. It sits at a fixed IP on a named Docker bridge.
- **attacker** -- Ubuntu 22.04 with nmap, nikto, hydra, curl, and netcat. On startup it waits for the victim to be reachable, then runs 16 attack phases in sequence: port scans, web scans, FTP/SSH brute force, HTTP injection attempts (SQLi, LFI, XSS, Shellshock), banner grabs, DNS lookups, and Tomcat credential brute force.
- **tcpdump** -- captures every frame on the lab bridge to `output/pcap/capture.pcap` with no filtering and full snaplen.
- **suricata** -- runs Emerging Threats Open rules against live traffic, writing alerts and metadata to `output/suricata/eve.json`.
- **zeek** -- runs `zeek -i lab-br0 local`, generating `conn.log`, `http.log`, `dns.log`, `ssl.log`, `files.log`, `notice.log`, `weird.log`, and more in `output/zeek/`.

After a full run (15-25 minutes depending on hardware), `output/` contains everything needed to practice analysis, triage, or alert correlation.

---

## The Part That Wasn't Obvious: Sensor Networking

The networking design took some thought. The intuitive approach is to put all five containers on the same Docker bridge network and let the sensor containers sniff from inside. That doesn't work.

Docker bridge networks do L2 switching: a container only sees frames addressed to itself or broadcast traffic. Even if you grant `NET_RAW` capability and enable promiscuous mode, a container inside the bridge subnet cannot sniff unicast frames passing between other containers. The sensors would see nothing.

The solution is to run the sensor containers on `network_mode: host` instead of inside the bridge. From the host network, they can see `lab-br0` -- the Linux bridge interface that Docker creates to back the `labnet` network -- and capture every frame that crosses it. The attacker and victim stay inside the bridge and communicate normally. The sensors watch from outside.

That raises a second problem: Docker generates a random bridge interface name (`br-a3f9c12d8e1b`) for each Compose project. The sensors need a stable interface name to reference in their start scripts. The fix is one line in `docker-compose.yml`:

```yaml
networks:
  labnet:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: lab-br0
```

This pins the bridge name. Every sensor start script can reference `lab-br0` directly, no discovery logic needed.

---

## How Claude Code Fit In

The architecture -- five containers, host-network sensors, named bridge, startup sequencing -- was designed before I wrote a line of code. I knew what I needed; I just didn't want to spend two days writing Dockerfiles and shell scripts for a stack whose purpose is to support the analysis toolchain, not be the learning objective itself.

I used Claude Code with directed prompts to implement the stack from the architecture I'd already designed. Then I did a second pass to review every file against a checklist I wrote: Does the Compose file sequence startup correctly? Do the start scripts wait for the right conditions before capturing? Does `suricata.yaml` have the right `HOME_NET` and Community ID settings? Do the volume mounts point to the right subdirectories?

That review pass caught a handful of issues -- volume mount paths were inconsistent, and the `suricata-update` fallback for offline runs needed a fix. The final documentation review happened in a separate Claude.ai session, which let me cross-check the README claims against the actual output from a real run.

The workflow felt right. I owned the design decisions and the review. Claude Code handled the mechanical implementation work that wasn't where I needed to spend time. The result is a stack I understand well enough to extend and debug, not a black box.

---

## What Came Out of Building It

Two things broke on the first run.

The Zeek sensor got stuck in its wait loop and never connected to `lab-br0`. The start script was using `ip` to check for the interface -- but the Alpine base image doesn't ship with `ip`, so the command was failing silently and the loop never exited. The fix was replacing the `ip` call with a direct filesystem check:

```
[ -d /sys/class/net/lab-br0 ]; do
```

`/sys/class/net/` is part of the Linux kernel's sysfs and is always available regardless of what network tools the image ships with. One line change, and the sensor started reliably.

Suricata came up but wouldn't run. The suricata-update step pulled a partial ruleset -- some rules failed to load -- and the default behavior was to abort rather than continue with what it had. The start script needed an explicit fallback: if the update produces an incomplete ruleset, touch the rules file so Suricata has something to load and can still start. That fallback is now in sensor/suricata/start.sh and documented in the Troubleshooting section of the README.

Both issues were caught on the first real run, which is exactly what the first real run is for.

---

## Where This Is Going

The immediate next step is connecting this to the `mcp-netparse` toolchain -- mounting `output/` as the evidence volume and running triage queries against the PCAP and logs from a real attack run. That closes the loop between sample generation and analysis practice.

Longer term, I want to add a Zeek custom policy for more targeted detection and experiment with swapping Metasploitable2 for lighter victims when I only need specific attack types. The Customization section in the README covers how to do both.

---

Repository: [https://github.com/desvert/mcp-test-env](https://github.com/desvert/mcp-test-env)

