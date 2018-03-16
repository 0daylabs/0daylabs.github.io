---
author: AnirudhAnand
layout: post
title: SSRF vulnerability on Google's Feedburner
categories:
- bugbounty
summary: SSRF bug in Google's feedburner.
---

Recently after getting an SSRF on [Microsoft's Bing Webmaster central]({{ site.url }}/2015/08/09/SSRF-in-Microsoft-bing/), I decided to test the same attack on any of the Google acquisitions and feedburner was a great choice. So I quickly went ahead to test the site and I saw an option to add new feed from a URL. Let's try injecting port numbers into it and see what happens:

As usual, I entered the url `https://scanme.nmap.org:22` and to my surprise, I got back a result saying: `Received HTTP error code -1 while fetching source feed`. Well, it looks like this is vulnerable to SSRF. Let's confirm it by trying with a closed port; so this time I changed the port number from 22 to 25, which is a closed port, I got another error saying: `An error occurred connecting to the URL: unknown error`. Well, I guess the vulnerability is confirmed now. I got 2 different errors depending on whether the port number is closed or not.

I tried to expand this by trying to scan the internal networks but unfortunately I had no luck there. Then I tried using other protocols like `file://` but this was also restricted. Only `http://` or `https://` can be used. Sadly I reported the bug to Google hoping that they will atleast fix it but won't reward since it's a low priority bug. The reply they gave was: `The ability to scan a port is not in itself a vulnerability`. It seems they have already explained about this on their [bug hunters university](https://sites.google.com/site/bughunteruniversity/nonvuln/ip-port-scanning-via-google-services)

Even through Google said that this is not a vulnerability, at a later point of time I saw that this issue has been being fixed. And as of now, no matter what port number you give, it always returns the error: `An error occurred connecting to the URL: internal error`. Looks like they patched the vulnerability but didn't give any credits for it.
