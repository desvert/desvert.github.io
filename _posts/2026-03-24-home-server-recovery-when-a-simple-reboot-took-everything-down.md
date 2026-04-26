---
layout: post
title: "Home Server Recovery: When a Simple Reboot Took Everything Down"
date: 2026-03-24
categories:
    - blog
tags:
    - homelab
    - linux
    - networking
    - docker
    - sysadmin
    - incident-response
excerpt: "A routine reboot triggered a chain of failures: DNS, networking, Docker, and eventually the Minecraft server itself. Here's how it unraveled, what I broke along the way, and how I rebuilt it cleaner than before."
---

# Home Server Recovery: When a Simple Reboot Took Everything Down

My home server runs a mix of Docker services, lab work, and a Minecraft Bedrock server for my kids. It's not production infrastructure, but it's also not something I want broken for days.

I SSH'd in one evening and saw the usual message: restart required for updates. No big deal. I rebooted.

When it came back up, the Minecraft server was offline. That led me down a chain of compounding failures across DNS, networking, Docker, and the server itself. Most of the night to fully untangle.

Here's the breakdown.

---

## Layer 1: DNS

First thing I noticed after the reboot: I could ping `1.1.1.1` but couldn't resolve `google.com`. That distinction matters. Full network failure and DNS failure look similar on the surface but point to completely different causes. Raw IP working meant this wasn't routing. DNS was broken specifically.

I started poking at `resolvectl` and `nslookup`. Both throwing errors. The resolver config wasn't right, but I wanted to understand why before touching anything, especially since I was still connected over SSH.

That's where I made my first mistake.

---

## The Self-Inflicted Wound

While troubleshooting, I ran:

```bash
sudo iptables -F
```

Anyone who's managed a Linux box remotely knows what comes next. SSH session froze. Connection dropped. Remote access gone.

I knew better. It still happened. Flushing firewall rules over SSH without a rollback plan is how you turn a DNS problem into a full access problem. The safer move is a dead man's switch before touching anything:

```bash
(sleep 60 && sudo iptables -P INPUT ACCEPT) &
sudo iptables -F
```

That gives you a minute to cancel if things go sideways. I didn't have that in place. I moved to physical access -- keyboard and monitor directly on the machine.

---

## Layer 2: The Network Stack

On the console, things looked worse than expected. The system was booting into emergency mode and `nmcli` showed network interfaces listed as unmanaged. NetworkManager had lost control of them after the reboot. Both WiFi and Ethernet were effectively dead.

Getting NetworkManager back in control and manually reconnecting WiFi restored layer 3. The gateway was reachable, external IPs responded.

DNS still wasn't working.

---

## Layer 3: DNS, Again

Restoring network connectivity didn't fix the resolver. After some digging, `systemd-resolved` was running but the system wasn't actually using it. `/etc/resolv.conf` wasn't linked correctly. The fix:

```bash
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo resolvectl dns wlp2s0 192.168.0.1
sudo systemctl restart systemd-resolved
```

After that, domain resolution came back. Network and DNS working, SSH access restored.

---

## Layer 4: Docker

I expected Docker to recover on its own once networking was stable. It didn't. The service failed on startup with an exit code error. Pulling the journal logs surfaced the actual problem:

```
invalid character '}' after object key:value pair
```

Malformed `/etc/docker/daemon.json`. A missing comma between two config blocks. Docker won't start if that file exists and contains invalid JSON.

```bash
python3 -m json.tool /etc/docker/daemon.json
```

That validation step is worth running before restarting Docker after any config change. One comma fixed it.

---

## Layer 5: Bedrock

With networking and Docker restored, I expected the Minecraft container to just work. The container would start, but logs immediately showed:

```
Could not connect to Minecraft services
```

DNS inside the container was broken initially. Outbound connectivity was inconsistent. When I enabled online mode, the server would shut itself down. The host was fixed. The container environment wasn't.

---

## The Version Mismatch

This part took longer than it should have.

I tried updating Bedrock to the latest version. Files on disk were correct, hashes matched, and running the binary directly on the host showed the right version. The server started cleanly, connected to Minecraft services, everything worked. Inside Docker, it was still reporting an older version.

Once I ran the binary manually and watched it come up on `1.26.3.1` while the container reported `1.26.0.2`, the conclusion was clear: the container wasn't using the files I thought it was. Some combination of volume mounts, entrypoint behavior, or cached state was getting in the way. The problem wasn't the server or the world data. The problem was the abstraction.

---

## The Better Question

At that point I stopped trying to fix Docker's behavior and asked a more useful question: do I actually need Docker for this?

For a home server running a single game server for my kids, the answer was no. Docker makes sense when you need isolation, portability, or environment consistency across machines. None of those applied here. The abstraction was adding complexity without returning anything.

I pulled Bedrock out of Docker entirely.

---

## New Setup

Running Bedrock directly on the host was the right call. The server started cleanly, reported the correct version, connected to Minecraft services without issues.

Systemd handles the service lifecycle. The server runs inside a named tmux session (`bedrock`). A second session (`bedrock-admin`) is split into two panes: the top pane attached to the live server console, the bottom running a live log view:

```bash
tail -f ~/bedrock_server/bedrock.log
```

Instead of `docker exec` to send commands, tmux does the job directly:

```bash
tmux send-keys -t bedrock "command" Enter
```

Wrapped in a small helper so it's a one-liner from anywhere:

```bash
bedrockcmd() {
  tmux send-keys -t bedrock "$*" Enter
}
```

No container boundary to fight through.

---

## Backups and Updates

The backup script was already in place. I updated it to work with the tmux-managed process instead of Docker. It issues `save hold`, waits for `save query` to confirm, then runs `save resume` and archives the data directory. Runs on cron daily.

The update workflow is similarly straightforward: stop the service, extract the new Bedrock release, archive old runtime files, deploy, restart. No Docker rebuilds, no image cache confusion.

---

## What Came Out of This

Don't flush iptables over SSH without a rollback. I knew this. It still happened. Set up a dead man's switch before touching firewall rules remotely.

Network failures exist in layers. IP connectivity, DNS, and name resolution are distinct problems. So are NetworkManager interface ownership and the resolver configuration. Treating them as one issue wastes time and obscures the actual diagnosis.

Containers add abstraction, and abstraction complicates debugging. That's a design characteristic, not a flaw. When something goes wrong inside a container, you're debugging the container, the host, and whatever's running inside it. For a home lab use case, that overhead isn't always worth it.

Validate your assumptions. I spent more time than I should have on the version mismatch because I assumed the container was using the files I could see on disk. Running the binary directly settled it in about 30 seconds.

---

## Where This Is Going

The immediate goal is better documentation for the recovery steps -- something more structured than incident notes. I'm also looking at lightweight monitoring for network and DNS health so failures like this get caught before they compound.

There's probably a broader post here about when abstraction helps versus when it gets in the way. It's a question that comes up in OT/ICS contexts too. The tradeoffs show up at different scales, but the shape of the problem is the same.