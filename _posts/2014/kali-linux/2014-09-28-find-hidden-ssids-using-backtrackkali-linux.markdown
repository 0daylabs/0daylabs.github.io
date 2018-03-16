---
author: AnirudhAnand
layout: post
title: '[How to] Find out Hidden SSIDs using Backtrack/Kali Linux'
categories:
- kali-linux
summary: How to find out hidden WIFI SSID'S using Kali-Linux ?
---

In the last article, we saw [how can we enable ssh in Backtrack/Kali Linux]({{ site.url }}/2014/09/28/enabling-ssh-connectivity-backtrackkali-linux/) so that we can control it remotely without physically present before the system. Also we have covered the basic networking techniques in **Backtrack/Kali**. Now let us move to different kinds of attacks, how it works and how can we stop it.

In this article, we will teach you how to discover SSIDs that is hidden from normal views. **SSIDs (Service Set identifier)** is nothing but the network name that we give during the configuration of the router or Access point. For security reasons sometimes people may hide it while configuring Access points to avoid normal people from accessing it. So let us see how can we find out such a hidden network. To find this out, we will use 3 inbuilt tools from Backtrack/Kali namely **airmon-ng**, **airodump-ng**, **aireplay-ng**.

First, we have to mon­i­tor the wire­less card. For that we use **airmon-ng**. Open up a new terminal and give this command:

`sudo airmon-ng`

This should list all the interfaces(both wired and wireless) like on the screen shot. Now lets start monitoring by giving the command :

`sudo airmon-ng start wlan0`

This will begin a monitoring service normally called mon0 (check out the screen shot). Now we have to dump the information collected by this monitoring. In order to do this, we will use** airodump-ng**. Give the command :

`sudo airodump-ng mon0`

This will show all the **SSID's** available in the network. Here, in the screen shot, I have not included any hidden SSID's as I haven't created any. If there are any hidden SSID's, it will show names similar to this:

![Airomon in action](/images/2014/Airomon.jpg)

But here, let us consider **ACCS-Student** shown on the screen shot is hidden. You can understand from the screen shot that all of the wifi that I have used is working on **channel 11**. Normally it won't be like this but here, in this special case all wifi's are on same channel. So now lets us give the next command :

`airodump-ng -c 11 mon0`

This command will dump info about the **SSID's** working on that specific channel. Now you can do 2 things:

You can wait till a user who knows about this hidden **SSID** to connect himself to that network while we are monitoring and the same will produce the SSID name on your screen. So what if, you don't want to wait ??
You can do a **Deauth attack** on the SSID. That will disconnect all the users who are using the network. That will force them to rejoin while we are monitoring and we will easily get the SSID. **Deauth attack** command is :

`aireplay-ng -0 3 -a mac-address-of-hidden-SSID mon0`

This will sent a **Deauth notification** exactly 3 times to the SSID which will result in disconnection of all users currently using it. That will make them rejoin soon and that will get our SSID. Once you get the **SSID** you can tell the BackTrack/Kali Linux to associate with it by giving the command (Consider the hidden SSID we found out was **ACCS-Student**  :

`iwconfig wlan0 essid ACCS-Student channel 11`

**NOTE:**
	
  * Sending a **Deauth attack** may not work sometimes. It depends on so many factors. But in almost all cases it will work.
	
  * This article is for education purposes only. It is not recommended to use these attacks illegally over public networks.