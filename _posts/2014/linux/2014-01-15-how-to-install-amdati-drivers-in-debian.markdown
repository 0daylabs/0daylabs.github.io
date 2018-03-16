---
author: AnirudhAnand
comments: true
date: 2014-01-15 18:19:03+00:00
layout: post
slug: how-to-install-amdati-drivers-in-debian
title: '[How to] Install AMD/ATI drivers in Debian ?'
wordpress_id: 930
categories:
- tutorials
- linux
summary: Installing AMD/ATI drivers in Debian
---

After having a fresh Install of Linux on a new system, the next hardest thing to do is to configuring Graphics card. Especially if we have a dedicated Graphics card or may be two Graphics card in Cross-firing mode or SLI mode. I still remember the day when I installed Debian on my laptop for the first time, I didn't knew how to configure the Graphics card. By default, the Graphics card was up and running in high performance mode. I have an HP gaming laptop with Cross-firing Graphics at that time. Soon after the Install. I saw my system heating up like it was put on a gas stove. Soon I saw my system lagging very much and after some time, it automatically shutdown. I was really wondering what could have happened ? When I again turned on the laptop, I saw a message saying " Your system has been undergone **Thermal Shutdown**." I was shocked logged back in to my Windows until the lap has been fully recovered. Later only I learned, How to configure Graphics cards, especially _**AMD/ATI**_. SO lets us see how to configure the Graphics card :

Before starting, I strongly recommend to follow the below method. Downloading and Installing **AMD/ATI** drivers from their site is not recommended but still you can give it a try if you want. Such Installation will most often have a clash with the** gdm3** causing it will refuse to start again after the installation.

Immediately after the installation, the first thing you should do is to configure your Graphics card or else you will face similar situation that I faced. If your ATI card is  Radeon HD 7000, Radeon HD 6000 and Radeon HD 5000 series GPU, the follow the below tutorial :

1) Open `/etc/apt/sources.list` to add the ATI driver source to the source.lists. To do that , give the following command:


{% highlight bash lineos %} 
   nano /etc/apt/sources.list 
   
   #Add the following command to the end of the file
   
   # Debian 7 "Wheezy"
   deb http://http.debian.net/debian/ wheezy main contrib non-free
{%endhighlight%}

2) After adding the above commands, save the file and exit. Now enter the following commands one by one :

{% highlight bash lineos %}
   sudo aptitude update
    
   sudo aptitude -r install linux-headers-$(uname -r|sed 's,[^-]*-[^-]*-,,') fglrx-driver
{%endhighlight%}


3) Now restart the system. Your card will be configured automatically for the best results.

Now, If you have Radeon HD 4000, Radeon HD 3000 and Radeon HD 2000 series GPUs, do the following :

1) Add the following line to the /etc/apt/sources.list , :

  {% highlight bash lineos %}  
   nano /etc/apt/sources.list
   
   add the following line to it : 
   
   # Backported packages for Debian 7 "Wheezy"
   deb http://http.debian.net/debian/ wheezy-backports main contrib non-free

{%endhighlight%}


2) Now launch a terminal and give the following commands one by one :

{% highlight bash lineos %}
   sudo aptitude update
   
   #Install the Appropriate Linux headers
   sudo aptitude install linux-headers-$(uname -r|sed 's,[^-]*-[^-]*-,,')
   
   sudo aptitude -r -t wheezy-backports install fglrx-legacy-driver
{%endhighlight%}

3) Restart your system and you will have the driver Installed with optimum conditions Auto configured. Hurrayy ! That's it. Your AMD/ATI card is configured and is ready to go. :)

**NOTE:** 

If your gdm3 is already crashed or not working properly, you follow the article: "[[Fix] gdm3 doesn't start after installing ATI/AMD drivers in Debian !]({{ site.url }}/2014/01/15/fix-gdm3-doesnt-start-installing-atiamd-drivers-debian/) " to fix it and then do this Tutorial.

