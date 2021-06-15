---
title: "THCon21 - ELF x64 - BaseJumper CrackMe"
date: 2021-06-14T16:29:01+02:00
draft: true
tags: [reverse engineering]
# image: "/image/blog-pic.jpg"
description: "An ELF x64 reverse engineering challenge from THCon21"
showDate: true          # to enable/disable showing dates
math: true              # to enable showing equations (katex)
---

# THCon21 - ELF x64 - BaseJumper CrackMe

[Go back to write-ups list](../)

> **Category:** Reverse
> 
> **Creator:** Podalirius
> 
> **Description:**
> 
> Find the validation password.
> 
> **Attachments:**
> - [elf_x64_basejumper_crackme.bin](/files/thcon21/elf_x64_basejumper_crackme.bin)

## Analysis

I opened this binary in IDA and jumped to the `main` function. After a *lot* of `char` variables definitions (which probably contain the flag encrypted in some way), we can see the user input verification part:

![Verification input](/image/thcon21/input_verif.png)

As we can see, it loops through the characters of `argv[1]` and `break` whenever a character doesn't match the condition.

From there, I could have tried to understand the condition, and I would probably have had to retrieve the value of all the variables defined at the top of the `main` function. But I was kinda lazy. So I went for a different yet interesting method: using [valgrind](https://valgrind.org/).

## Valgrind

Valgrind is a tool to analyze the performance of a program. One of its functionnality - callgrind - allows you to count how many instructions were run before the program exits. In our case, if we enter a correct character, the program will go through one more iteration of the loop, thus increasing the number of instructions run.

Using this idea, I made a little script to bruteforce the characters one by one using valgrind. This method can be used for any challenge that early-exits when checking a password char by char (with some change to the charset).

```py
import subprocess as sp
import re

# from string import printable
printable = "0123456789ABCDEF"
command = "./elf_x64_basejumper_crackme.bin"

def count_instructions(command, password):
    try:
        print(sp.check_output([f"valgrind --tool=callgrind --callgrind-out-file=/dev/null {command} {password}"], stderr=sp.STDOUT, shell=True))
    except Exception as e:
        result = e.output.decode()
        count = int(re.search(r"Collected : (\d+)", result).group(1))
        return count

lastpassword = "z"
password = ""
while lastpassword != password:
    lastpassword = password
    # "z" is not in the alphabet, so we use it to get the number of instructions for a password that is sure to fail
    default_count = count_instructions(command, password+"z")
    for c in printable:
        count = count_instructions(command, password+c)
        if count != default_count:
            password = password + c
            break
    print(password)

print(password)
```

*Note: I guessed that the input would be uppercase hexadecimal after bruteforcing the 4 first characters (and considering it was the case for the previous chall from the same author). Reducing the alphabet from 100 to 16 highly sped up the bruteforce.*

And here we can see it peacefully recovering the characters from the password.

![Valgrind](/image/thcon21/valgrind.gif)

The final password is
```
4B5A4357515244434749324853544B594F524A46474D4B4F4B5252464B564A544B4E4B4751534443495A484553563258474649464F564C494A5A53554B3342534A354C54533453574B5634454D564B46474646464D57425148553D3D3D3D3D3
```
so it took a bit of time to recover (the GIF twice the real speed).

## Basejumping

Now that we have this big hexadecimal string, it's time we get the flag. Once we decode the hexadecimal (base 16), we get another encoded string:

```
KZCWQRDCGI2HSTKYORJFGMKOKRRFKVJTKNKGQSDCIZHESV2XGFIFOVLIJZSUK3BSJ5LTS4SWKV4EMVKFGFFFMWBQHU=====
```
Uppercase letters + numbers and 5 `=` of padding at the end, that is obviously base 32. Once decoded, we get:

```
VEhDb24yMXtRS1NTbUU3SThHbFNIWW1PWUhNeEl2OW9rVUxFUE1JVX0=
```
Which looks a lot like base 64 and gives the flag once decoded:

```
THCon21{QKSSmE7I8GlSHYmOYHMxIv9okULEPMIU}
```

To be honest, it took me a while to write the valgrind script, probably more than I would've spent understanding the condition. But I'm glad I did it, because this technique can be used for a lot of challenges, so now I have it ready.

[Go back to write-ups list](../)