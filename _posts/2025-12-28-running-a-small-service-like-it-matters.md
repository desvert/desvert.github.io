---
layout: post
title: "Running a Small Service Like It Matters"
date: 2025-12-28 00:57:07
categories: [blog]
tags: []

---

## Running a Small Service Like It Matters


### Lessons from a Containerized Minecraft Bedrock Server

When people hear “Minecraft server,” they usually think of something casual or temporary. Spin it up, play for a while, shut it down. This project started that way, but it didn’t stay there for long.

I wanted a small, always-on service that my family could rely on, and I treated it the same way I would any other Linux service: predictable startup, safe backups, clear configuration, and minimal babysitting. The fact that it happened to be a game server was secondary.

What follows isn’t a tutorial. It’s a short write-up of the operational decisions that turned a hobby server into something closer to a production workload.

---

### Why containerize it at all?

The Bedrock Dedicated Server runs fine on bare metal, but I didn’t want another snowflake service living directly on the host. Containerizing it solved several problems immediately.

It created a clean separation between the host OS and the application, gave me a single command to bring the service up after a reboot, and made restart behavior predictable if the server ever crashed. It also gave me a clear place to define configuration and persistent storage.

Docker Compose became the contract for how the service runs. Once that file existed, the server stopped feeling fragile and started feeling reproducible.

---

### Persistent data and the “don’t lose the world” rule

World data, permissions, and server properties are bind-mounted to the host. That decision was non-negotiable.

Containers are disposable. Data is not.

Keeping state outside the container means the server can be destroyed and recreated without losing anything important. It also makes backups straightforward and keeps configuration transparent instead of hidden inside container layers.

This mirrors how I’d treat any stateful service. The scale is smaller, but the rules don't change.

---

### Graceful backups without kicking players

The most interesting part of this setup ended up being backups.

Stopping the server to back it up technically works, but it’s disruptive and unnecessary. Bedrock supports save control commands that allow world writes to be paused, flushed, and resumed. That made it possible to perform live backups safely.

The backup flow looks like this:

1. Temporarily pause world writes
2. Confirm all pending data is flushed
3. Archive the persistent data directory
4. Resume normal operation

From a player’s perspective, the server stays online. From an operator’s perspective, backups are consistent and low risk. This is exactly how I’d want backups to behave on any always-running service.

---

### Configuration drift is real, even on a family server

Game rules are world-level configuration, and they can be changed accidentally. Rather than relying on memory or discipline, I automated enforcement.

A small host-side script waits for the server to become ready, then applies a known-good set of gamerules using console commands. The script is idempotent, safe to run repeatedly, and wired into systemd so it executes automatically at boot.

This is the same pattern used in larger environments to prevent configuration drift. The scale is smaller, but the principle is identical.

---

### Permissions, caching, and reality

One unexpected lesson was client-side permission caching. Even after operator permissions were correctly set on the server, connected clients didn’t always pick them up immediately. A reconnect was required.

It was a good reminder that correctness isn’t just about configuration files. You also have to understand how clients interact with services and where state is cached. It’s a small detail, but it's the kind that causes real confusion if you don’t account for it.

---

### Observability and “quiet” operation

Logs are handled by Docker with rotation enabled to prevent disk growth over time. Backup scripts log their own activity. There’s no dashboard or alerting system because it doesn’t need one, but there is enough visibility to troubleshoot when something goes wrong.

The goal wasn’t maximum instrumentation. It was a system that runs quietly until it needs attention.

---

### Why this project mattered

This server reinforced a simple idea: scale doesn’t change fundamentals.

Even a small service benefits from automated startup, graceful backups, explicit configuration, and documentation of intent. Treating a "toy" service seriously turned it into a useful environment for practicing Linux administration, container management, and operational thinking.

Those habits translate cleanly to larger systems. 

---

### Related links

- Project repository:  
    [https://github.com/desvert/containerized-bedrock-server](https://github.com/desvert/containerized-bedrock-server)
