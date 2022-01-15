---
title: "Discord's Image Preview - A boon for Phishing Attacks"
date: 2022-01-15
draft: false
tags: [phishing]
# image: "/image/blog-pic.jpg"
description: "How to hijack Discord's preview of images "
showDate: true          # to enable/disable showing dates
math: false              # to enable showing equations (katex)
---

## Intro

Phishing attacks on Discord are a big trend right now and hacked accounts sending fake Free Nitro links have become part of the landscape of Discord servers.

Fortunately people are starting to be more aware of phishing attacks, and the hackers start to lack realistic domain names for their phishing attacks (and thus we get links like `https://discqrde.club/boost` *- please don't check this*)

So I was wondering if hackers would end up finding better ways to trick their targets using features from Discord. And for this reason, I started looking at how Discord's image preview works.

When you send a link that leads to an image on Discord, the application will replace the URL by a preview of the actual image. And I know it's not just your computer fetching the image because the preview is downscaled if your image was too big. So there must be a bot from Discord that grabs the image, downscales it if needed, and returns a link to some Discord server-side cache with the preview of your image. So it might be possible to send an image link to the user that will display a totally different page when they open the image in their browser.

## Studying Discord's image bot

In order to find solid evidence that there is a bot from Discord and try learn how it works, I set up a small web server based on Flask to send an image and record the incoming traffic :

```py
from flask import Flask, send_file, request

app = Flask(__name__)

@app.route("/test<whatever>.jpg")
def send_image(whatever):
    ua = request.headers.get("User-Agent")
    print(ua)
    return send_file("test.jpg", mimetype="image/jpeg")

app.run(host='0.0.0.0', port=4444)
```

Then I sent a link to `http://[mydomain]:4444/test1.jpg` and it got converted to a preview of my `test.jpg` file by Discord. At the same time, I recorded the following logs in my console:

```
Mozilla/5.0 (compatible; Discordbot/2.0; +https://discordapp.com)
35.196.132.85 - - [15/Jan/2022 18:59:50] "GET /test1.jpg HTTP/1.1" 200 -

Mozilla/5.0 (Macintosh; Intel Mac OS X 11.6; rv:92.0) Gecko/20100101 Firefox/92.0
35.196.13.1 - - [15/Jan/2022 18:59:53] "GET /test1.jpg HTTP/1.1" 200 -
```

> Note: at the beginning, I tried sending links without a domain name, but Discord didn't want to show their previews. Anyway with [ngrok](https://ngrok.com/) it's fairly easy to get a domain name pointing to your machine.

So here we got two different User-Agent and IP adresses (which are owned by Discord, you can check using nslookup or whois). One of the User-Agent is easily recognizable, but the other one could look like a regular Firefox browser on a Macintosh computer.

Interestingly, Discord only checks the link the first time and then stores the preview in their cache, so for testing purposes I added the `whatever` parameter so that I could get as many test links as I wanted. Even if your server goes down or the content on the link changes, Discord will send the cached preview. Can we exploit that to display a preview different from the actual link? Well not really, since the cache lasts less than an hour. But that's a good start.

To go deeper, I edited the python server like this:
```py
def send_image(whatever):
    ua = request.headers.get("User-Agent")
    print(ua)
    if "Discordbot" in ua:
        print("Discordbot went there")
        return send_file("test.jpg", mimetype="image/jpeg")
    else:
        return "placeholder"
```

When I sent a new link, it replaced the link with a loading animation, but the preview never loaded and I ended up with their beautiful poop image:

![Discord's poop placeholder](/image/misc/discord_poop.png)

At the same time on my server, I got the usual connection from Discordbot 2.0, and then I got a continuous flow of requests from different IP adresses with the Macintosh user agent. So my guess is that Discordbot checks the MIME type of the link, and if it's an image, it asks the Macintosh bot to make a preview.

## Hijacking the preview

Now that we understand a bit more how Discord generates the preview of an image link, let's see how we could trick the user and show them a different content in their browser.

First, we can consider the previous experiment as a satisfying phishing method: seeing the poop image instead of a preview, the user will probably open the link in their browser and see the placeholder instead.

But I tried to modify the server once again to send the real image to Macintosh user agents as well:

```py
if "Discordbot" in ua or "Macintosh" in ua:
    return send_file("test.jpg", mimetype="image/jpeg")
else:
    return "placeholder"
```

> Yes this makes Mac users invulnerable, but I don't really care since this is just a proof of concept.

So now the preview shows correctly in Discord, and when you open the image in the browser, it shows our placeholder!

All there is left to do is get creative to find ways to trick the user into opening our image in their browser:

- As I said earlier, Discord's poop placeholder could already be enough
- The image could contain really small text that's hard to read
- The fake preview could be a GIF of Discord's loading to make the user think it loads forever
- And if the preview looks "broken", the user may not be surprised to see a file (an image of course *ahem*) downloading in their browser or to see a (very legitimate) Discord authentication page.

So I hope I've been able to convince you of the risks behind Discord's image preview. I had already heard about "infinite loading images getting your account hacked" from some people on Discord, but never had a chance to see it with my own eyes. Feel free to try it yourself (for education purposes of course).