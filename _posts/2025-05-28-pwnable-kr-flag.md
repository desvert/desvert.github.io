---
layout: post
title: Pwnable.kr - flag
date: 2025-05-28
categories:
  - blog
tags:
  - pwn
  - binary
  - ctf
excerpt: ""
---
## Pwnable.kr - flag

This was an interesting and challenging exercise for me. The clue given:

>Papa brought me a packed present! let's open it.
>Download : `http://pwnable.kr/bin/flag`
>This is reversing task. all you need is binary


```
$ file flag
flag: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, no section header
```

I checked `strings flag | less` to see if anything jumped out, and it was mostly character salad. I did notice this string at this point, but I didn't investigate it further until later:
```
$Info: This file is packed with the UPX executable packer http://upx.sf.net $
$Id: UPX 3.08 Copyright (C) 1996-2011 the UPX Team. All Rights Reserved. $
```

  I ran `gdb flag`, hoping to crack the binary open and see what lays within, but no luck.
```
Reading symbols from flag...
(No debugging symbols found in flag)
(gdb) disas main
No symbol table is loaded.  Use the "file" command.
(gdb) file
No executable file now.
No symbol file now.
```

After some google-fu, I discovered that this file is stripped, meaning that there isn't an easy way to get `gdb` to open it up. I decided to execute the file and see what it does. But first, I fired up a Kali VM to act as a sandbox, because I'm just not that trusting. First I had to `sudo chmod +x flag` to make it executable.
```
$ ./flag
I will malloc() and strcpy the flag there. take it.
```

That sounded familiar from when I learned C language a few years ago. Admittedly, my C is rusty, so off to check the man-pages and some more googling. That helped refresh my memory a bit, but I was still not sure where to start attacking this binary. 

The next night, I returned to see if I could make any progress. I tried using `radare2` to see if I could find a starting point, and while I did get some interesting output, I was feeling pretty stumped.
```
[0x0044a4f0]> i
fd       3
file     flag
size     0x51db8
humansz  327.4K
minopsz  1
maxopsz  16
invopsz  1
mode     r-x
format   elf64
iorw     false
block    0x100
type     EXEC (Executable file)
arch     x86
baddr    0x400000
binsz    811736
bintype  elf
bits     64
canary   false
class    ELF64
crypto   false
endian   little
havecode true
laddr    0x0
lang     c
linenum  true
lsyms    true
machine  AMD x86-64 architecture
nx       false
os       linux
pic      false
relocs   true
rpath    NONE
sanitize false
static   true
stripped false
subsys   linux
va       true
```

My takeaway from `radare2` is that I will need to spend some time learning how to use it effectively, but it looks like an awesome tool.

I started looking for some clues from people that have done this CTF before, but wasn't getting any hits in the Discord servers I lurk in. So I found a walkthrough, and quickly found that I had overlooked a **big** clue at the beginning: the `strings` output that "this file is *packed* with UPX..." coupled with the clue given at the beginning "Papa brought me a *packed* present!" Finally I had a jumping-off point! I set the walkthrough aside, and started digging some more. A quick search led me to https://upx.github.io, where I downloaded the `upx` tool. 
```
$ upx -h
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2024
UPX 4.2.4       Markus Oberhumer, Laszlo Molnar & John Reiser    May 9th 2024

Usage: upx [-123456789dlthVL] [-qvfk] [-o file] file..

Commands:
  -1     compress faster                   -9    compress better
  --best compress best (can be slow for big files)
  -d     decompress                        -l    list compressed file
  -t     test compressed file              -V    display version number
  -h     give this help                    -L    display software license

Options:
  -q     be quiet                          -v    be verbose
  -oFILE write output to 'FILE'
  -f     force compression of suspicious files
  --no-color, --mono, --color, --no-progress   change look
[ ... ]
```

I used the `-d` switch to unpack the file.
```
$ upx flag -d
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2024
UPX 4.2.4       Markus Oberhumer, Laszlo Molnar & John Reiser    May 9th 2024

        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
    887219 %3C-    335288   37.79%   linux/amd64   flag

Unpacked 1 file.
```

Then I checked the file information again, and was really happy to see that it is now "not stripped!"
```
$ file flag
flag: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, for GNU/Linux 2.6.24, BuildID[sha1]=96ec4cc272aeb383bd9ed26c0d4ac0eb5db41b16, not stripped
```

So I started `gdb` again and this time was able to disassemble the main function. A quick scan showed one line with a comment `# 0x6c2070 <flag>`. That looks promising. Let's see what's at that address...
```
(gdb) disas main
Dump of assembler code for function main:
   0x0000000000401164 <+0>:	push   rbp
   0x0000000000401165 <+1>:	mov    rbp,rsp
   0x0000000000401168 <+4>:	sub    rsp,0x10
   0x000000000040116c <+8>:	mov    edi,0x496658
   0x0000000000401171 <+13>:	call   0x402080 <puts>
   0x0000000000401176 <+18>:	mov    edi,0x64
   0x000000000040117b <+23>:	call   0x4099d0 <malloc>
   0x0000000000401180 <+28>:	mov    QWORD PTR [rbp-0x8],rax
   0x0000000000401184 <+32>:	mov    rdx,QWORD PTR [rip+0x2c0ee5]        # 0x6c2070 <flag>
   0x000000000040118b <+39>:	mov    rax,QWORD PTR [rbp-0x8]
   0x000000000040118f <+43>:	mov    rsi,rdx
   0x0000000000401192 <+46>:	mov    rdi,rax
   0x0000000000401195 <+49>:	call   0x400320
   0x000000000040119a <+54>:	mov    eax,0x0
   0x000000000040119f <+59>:	leave
   0x00000000004011a0 <+60>:	ret
End of assembler dump.
(gdb) x/1s *0x6c2070
0x496628:	"UPX...? sounds like a delivery service :)"
(gdb)
```

And there is the flag, revealed!