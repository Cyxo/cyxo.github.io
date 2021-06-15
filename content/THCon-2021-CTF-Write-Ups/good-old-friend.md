---
title: "THCon21 - Good old friend"
date: 2021-06-14T16:29:02+02:00
draft: false
tags: [reverse engineering, android]
# image: "/image/blog-pic.jpg"
description: "An Android reverse engineering challenge from THCon21"
showDate: true          # to enable/disable showing dates
math: true              # to enable showing equations (katex)
---

# THCon21 - Good old friend

[Go back to write-ups list](../)

> **Category:** Reverse
>
> **Creator:** zephyr
>
> **Description:**
>
> Good old friend
>
> **Attachments:**
> - [goodOldFriend.apk](/files/thcon21/goodOldFriend.apk)

## Analysing the APK

Android reverse engineering is by far my favorite subject. So let's dive right into the analysis.

I opened the APK file in a tool called JADX (the GUI version because I'm a fake hacker who doesn't use a terminal and a keyboard only) and searched for a class that would be called MainActivity (which was very likely to be in the `party.thcon.y2021.level1` package).

*Note: in case there are more Activities and classes, you should look for the `AndroidManifest.xml` file in the resources. Then search for the activity that has the intent `android.intent.category.LAUNCHER`. That's the activity launched when you click on the app's icon.*

What caught my attention in the code was the loading of a native library (which includes a `checkInput` function).

![Native library loading](/image/thcon21/load_library.png)

*The rest of the code in this activity checks whether the phone is rooted or not.*

Using recursive search (Ctrl+Shift+F), I could find where `checkInput` is used, and it confirms that the function is called on the content of the password input on MainActivity.

We can find the native library in the resources, under the `lib` folder. I decided to extract the x86_64 one because it's easier to reverse.

## Analysing the native library

I opened the .so file in IDA and filtered the functions list to find a function containing `checkInput`. Thus I found the function called
```
Java_party_thcon_y2021_level1_MainActivity_checkInput
```

Here is the most interesting part (I renamed the variables for more clarity)

![Native library code](/image/thcon21/native_aes.png)

And here are the values of the 2 xmmwords and the encrypted_password unk:

![Binary data](/image/thcon21/unks.png)

So at first I thought I would just decrypt the password using the following parameters:

```
key  = 3C4FCF098815F7ABA6D2AE2816157E2B
iv   = 000102030405060708090A0B0C0D0E0F
pass = E47BC2DFFAA645CB89A87780BB1619EF5DAA2AADF4CDA3EBD1884E64A2B43B68
```

But that didn't give me anything readable. Then my mate [@Slowerzs](https://slowerzs.github.io/) who had the library opened in Ghidra had the idea that the pass, key or IV may be flipped. So we tried flipping them and the key and IV were indeed in the wrong byte order. Using these parameters, decrypting the pass gave us the flag

```
key  = 2B7E151628AED2A6ABF7158809CF4F3C
iv   = 000102030405060708090A0B0C0D0E0F
flag = THCon21{C_1$_n3v3r_2_f@r}\x00\x00\x00\x00\x00\x00\x00
```

*Note: the native library used for the AES encryption, called tiny-AES, does not support padding, which is why the flag was padded with null bytes.*

[Go back to write-ups list](../)