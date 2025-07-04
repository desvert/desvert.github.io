---
layout: post
title: Detecting a Brute-Force SSH Attack Using System Logs
date: 2025-07-03 23:09:54
categories:
  - blog
tags:
  - ctf
  - htb
  - forensic
  - logs
excerpt: ""
---
# Detecting a Brute-Force SSH Attack Using System Logs

## Introduction

In this post, I walk through a real-world-style investigation I performed using system logs from a Linux server. The task was to determine whether the server had been targeted by a brute-force SSH attack and, if so, identify the attacker's IP and login attempts.

The process involved reading and interpreting `/var/log/auth.log`, searching for patterns that indicate brute-force behavior, and understanding the structure of SSH log messages.

---

## The Scenario

I was handed a snippet of an `auth.log` file and asked to determine whether a brute-force login attempt had occurred. Here's a portion of the log I started with:

```
Mar  6 06:31:40 ip-172-31-35-28 sshd[2389]: Connection closed by invalid user svc_account 65.2.161.68 port 46744 [preauth]
Mar  6 06:31:40 ip-172-31-35-28 sshd[2391]: Connection closed by invalid user svc_account 65.2.161.68 port 46746 [preauth]
Mar  6 06:31:40 ip-172-31-35-28 sshd[2393]: Connection closed by invalid user jsmith 65.2.161.68 port 46748 [preauth]
```

Immediately, there were a few indicators that something was off.

---

## Step 1: Recognizing the Pattern

Each line follows the same general structure:

```
<Date> <Time> <Hostname> sshd[<PID>]: Connection closed by invalid user <username> <IP> port <port> [preauth]
```

A few things stood out:

- Multiple connection attempts happened at the exact same second (`06:31:40`)
- Each attempt used a different PID (new process, new connection)
- The log message includes "invalid user", meaning the username doesn't exist on the system
- All attempts came from the same IP address: `65.2.161.68`
- Each attempt used a different port, which is consistent with new TCP sessions

This strongly suggested an automated brute-force or dictionary attack.

---

## Step 2: Filtering Logins by IP and Username

To confirm this pattern across the full log, I filtered it using `grep` and `awk`. If you have access to the full `/var/log/auth.log` file, you can run:

```bash
grep "sshd.*invalid user" /var/log/auth.log | awk '{print $9, $10}'
```

This command extracts just the username and IP address fields for each failed attempt.

To see which usernames were targeted:

```bash
grep "invalid user" /var/log/auth.log | awk '{print $9}' | sort | uniq -c | sort -nr
```

To count how many times the same IP tried to log in:

```bash
grep "invalid user" /var/log/auth.log | awk '{print $10}' | sort | uniq -c | sort -nr
```

---

## Step 3: Investigating the Attacker

With the IP address `65.2.161.68` identified as the likely source of the brute-force activity, I did a quick WHOIS lookup:

```bash
whois 65.2.161.68
```

This revealed that the IP belongs to Amazon AWS. That isn't unusual. Many automated attacks come from cloud-hosted VPS instances due to their low cost and ease of deployment.

Optionally, you could feed the IP into threat intelligence tools like AbuseIPDB or VirusTotal to see if it's been reported for abuse.

---

## Step 4: Mitigation Strategies

If this were a live system, I would take the following steps:

1. **Install fail2ban** to automatically ban IPs after too many failed login attempts:

    ```bash
    sudo apt install fail2ban
    ```

2. **Block the IP manually** with `iptables` or `ufw`:

    ```bash
    sudo ufw deny from 65.2.161.68 to any port 22
    ```

3. **Change the default SSH port**.

4. **Require public key authentication** instead of password-based logins.

5. **Use a VPN or jump box** to restrict SSH access entirely.

---

## Reflections

What I found most interesting about this task is how much information is exposed in plain-text logs. With a little bit of parsing and pattern recognition, you can reconstruct an attack timeline and respond accordingly.

No special tools were required. Just command-line utilities, an understanding of log formats, and a bit of curiosity.

---

## Tools Used

- `/var/log/auth.log`
- `grep`, `awk`, `sort`, `uniq`
- `whois`
- Optional: `fail2ban`, `ufw`

---

## Final Thoughts

This exercise helped reinforce the importance of log monitoring. Attacks like this are happening constantly across the internet, and the clues are usually right in front of us. Learning how to spot them is a valuable skill, especially for anyone pursuing cybersecurity or system administration roles.

In future posts, I plan to explore automated log analysis and alerting using tools like `logwatch`, `logrotate`, and even basic ELK setups.
