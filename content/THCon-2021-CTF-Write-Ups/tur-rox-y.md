---
title: "THCon21 - TUR-ROX-Y"
date: 2021-06-14T16:29:04+02:00
draft: true
tags: [reverse engineering, video game]
# image: "/image/blog-pic.jpg"
description: "A video game reverse engineering challenge from THCon21"
showDate: true          # to enable/disable showing dates
math: true              # to enable showing equations (katex)
---

# THCon21 - TUR-ROX-Y

[Go back to write-ups list](../)

> **Category:** Reverse
> 
> **Creator:** Réchèr
> 
> **Description:**
> 
> A journey in Turing-approxi-complete video games, chapter "Y".
> 
> "ZZT" ? What is that strange file extension ? It may be something old. If it's old, there is a MUSEUM about it. Whatever it is, I suppose it has nothing to do with Catherine ZETA-Jones.
> 
> **Attachments:**
> - [THCON21Y.ZZT](/files/thcon21/THCON21Y.ZZT)

## First step: discovery

So you thought that TUR-ROX-Z was complicated enough? Naaah, we want harder than that.

I opened this challenge directly in the KevEdit editor. We can see that there are many boards of interest.

![Visiting the world](/image/thcon21/thcon21y.gif)

So again, we have a flowchart, with this time a lot more useful information. We have a python script to check some PZS code that we'll have to extract. And then we have many boards called Puzzle Script, some with one block of code (L, probably for long), some with two (S, probably for short), which are annoying to read due to the colorful background.

## Extracting the PuzzleScript

Fortunately, the flowchart gives us [a library to parse ZZT files](https://github.com/DrDos0016/zookeeper). Now the idea was to find all Puzzle Script boards and parse them. We first have to read the PuzzleScript page number at the top of the boxes (because they have nothing to do with the number in the board name), and then read all the code in these boxes. Once we have all the pages, we can assemble them in the right order. Here is my python script to do so:

```py
import zookeeper

zoo = zookeeper.Zookeeper("THCON21Y.ZZT")

txt = {}
for board in zoo.boards:
    # Check if it's a Long or Short board
    if "L" in board.title:
        print("=" * 80)
        print(board.title)
        page = ""
        text = ""
        line = ""
        lasty = 0
        for elem in board.elements:
            # Grid size is 60x25
            x = elem.tile % 60
            y = elem.tile // 60
            if y == 2 and x > 6 and x < 27:
                page += chr(elem.character)
            if (y != lasty or y == 23) and y > 4:
                # On a new line, append the line read to the page's text. Ignore trailing spaces.
                text += line.strip() + "\n"
                line = ""
                lasty = y
            if y > 3 and y < 23 and x > 1 and x < 58:
                line += chr(elem.character)
            if y == 23:
                break
        txt[page] = text
    elif "S" in board.title:
        print("-"*80)
        print(board.title)
        # There are 2 boxes so we handle them separately
        page1 = ""
        text1 = ""
        line1 = ""
        page2 = ""
        text2 = ""
        line2 = ""
        lasty = 0
        for elem in board.elements:
            # Grid size is 60x25
            x = elem.tile % 60
            y = elem.tile // 60
            if y == 2 and x > 3 and x < 24:
                page1 += chr(elem.character)
            elif y == 2 and x > 33 and x < 54:
                page2 += chr(elem.character)
            if (y != lasty or y == 23) and y > 4:
                # On a new line, append the line read to the both page texts.
                text1 += line1.strip() + "\n"
                line1 = ""
                # There is one empty second box somewhere
                if not "Useless" in page2:
                    text2 += line2.strip() + "\n"
                    line2 = ""
                lasty = y
            if y > 3 and y < 23 and x > 1 and x < 28:
                line1 += chr(elem.character)
            elif y > 3 and y < 23 and x > 31 and x < 58:
                line2 += chr(elem.character)
            if y == 23:
                break
        txt[page1] = text1
        # Again, ignore if it's the empty box
        if not "Useless" in page2:
            txt[page2] = text2

res = ""
for i in range(1, 27):
    page = "PuzzleScript Page %02d" % i
    res += txt[page]

with open("res.pzs", "w") as f:
    f.write(res)
```

This gives us a big PuzzleScript file. We can check it using the script given in the pzs_check board:

```py
import hashlib

with open("res.pzs", "r") as f:
    pzs_code = f.read()

pzs_code = pzs_code.strip()
assert " \n" not in pzs_code
assert "\n\n" not in pzs_code
assert len(pzs_code) == 7214
assert "\r\n" not in pzs_code
b_pzs = bytes(pzs_code, encoding="ascii")
hash = hashlib.sha256(b_pzs).hexdigest().upper()
print("THCon21{Y" + hash[-2:] + "----}")
```

The result is `THCon21{Y95----}`, which gives us another part of the flag. Nice!

## Running the PuzzleScript

The flowchart of the challenge mentions a website called [PuzzleScript](https://www.puzzlescript.net/) to run this PZS. PuzzleScript is a game engine where you can easily script games that run in your browser (although I find that their scripting language is not the most humanly readable...).

When we start the game, we get a message saying we need to use the levers to get 5 purple grapes.

![First level](/image/thcon21/first_level.png)

Looking at the beginning of the code, we can see the definitions of the fruit sprites. Their names are in fact numbers from 0 to 6.

![Fruit values](/image/thcon21/fruit_values.png)

Here are the values:
- Cherry: 0
- Banana: 1
- Apple? : 2
- Orange: 3
- Grapes: 4
- Mango: 5
- Pear: 6

Now each lever increments one or more fruits in the 5-fruits basket. For example, in this level, the first lever increases the first fruit by 4. The second lever increases the second fruit by 1, the third fruit by 2, the fourth fruit by 3 and the fifth fruit by 5.

![Second lever](/image/thcon21/lever2.png)

If a fruit goes above 6, it will be reset to zero and the fruit on its left will be increased by one, like a base-7 number.

## Solving the puzzle

At the beginning I wanted to use the solver Z3. But since there were only 256 lever combinations and I'm too bad at Z3, we can just do a script that simulates the levers and tries all the combinations. And here it is for the first level:

```py
# Numpy has a cool function to convert a number to a specific base
from numpy import base_repr

lever_values = [
    "40000",
    "01235",
    "00400",
    "04000",
    "01256",
    "00040",
    "11111",
    "00004"
]

# Ugly function that converts a number to an array of bits
def int_to_bits(i):
    return list(map(int, bin(i)[2:].rjust(8,"0")))

# Test all lever combinations
for i in range(255):
    levers = int_to_bits(i)
    fruits = 0
    for j, lever in enumerate(levers):
        if lever:
            fruits += int(lever_values[j], 7)

    if base_repr(fruits, 7) == "44444":
        print(levers)
        break
```

This outputs: `[1, 0, 1, 1, 0, 1, 0, 1]`

![Level 1 solved](/image/thcon21/level1.png)

## The End

So this was only a training level. But once you get the idea, you just have to change the `lever_values` to match the next levels. They both give you 2 hexadecimal characters (from the binary value made by the levers) that form the missing part of the flag.

![Level 2 solved](/image/thcon21/level2.png)

![Level 3 solved](/image/thcon21/level3.png)

So we get the final flag: `THCon21{Y95E5CD}`

Again, this must have been tons of work to create this challenge, so kudos to the author. This concludes the Turing-approxi-complete video games series.

[Go back to write-ups list](../)