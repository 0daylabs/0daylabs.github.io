---
author: AnirudhAnand
layout: post
title: SSRF attack using Microsoft's bing webmaster central
categories:
- bugbounty
summary: SSRF bug in Microsoft Bing Webmasters Tool.
---

Recently when I was browsing through the internet, I came accross an awesome blogpost by Riyaz walikar on [Cross Site Port Attacks (SSRF/XSPA)](http://www.riyazwalikar.com/2012/11/cross-site-port-attacks-xspa-part-1.html). If you haven't read it already, I strongly suggest you to read it.

After reading the article, I was thinking of trying this attack at some place and the first thing that comes to my mind is Google's webmasters tool but Riyaz has already reported the same. So no point in trying. So whats next? Then it came to my mind, what about Microsoft's [Bing Webmaster Central](https://www.bing.com/webmaster)? Alright, lets have a look.

I went to the webmaster central and tried to add a new site `https://scanme.nmap.org:22` (basicaly because its not illegal to port scan the site as they give the permission to do so). Bing suggested me to add the verification tag to site but I didn't bother because I cannot do it anyway. Then I simply clicked on verify and look at the result I got back from bing:

![Microsoft's bing SSRF]({{ site.url }}/images/Protocol_violation_for_open_ports.png)

`Protocol violation` means the server expected the protocol to be http but since ssh was running in port 22, it gives back the error message. Now let us see what happens when I give a port number which is closed (22 was open). So I changed the port number from `22` to `15` and got back another error message:

![Microsoft's bing SSRF]({{ site.url }}/images/Connecion_failure when_port_is_closed.png)

`Connect failure` means that the port is closed. By using this mechanism we can use the Bing to port scan any internet facing webservers. 


After this, I tried to do something else. Instead of giving the remote webservers, I started giving the possible internal IP address and for my surprize the bing is yeilding back the same result as it gave for remote servers ! Thats awesome isn't it? Well, yes atleast when Bug hunters are concerned!

I reported this to Microsoft few weeks back and the issue has been successfully fixed by the Microsoft. Also thanks to Microsoft for listing my name in their [Security hall of fame](https://technet.microsoft.com/en-us/security/cc308589). 

This is my first bug report. Hopefully I can find more in the coming days.
