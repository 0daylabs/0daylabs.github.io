---
author: AnirudhAnand
layout: post
slug: man-in-the-middle-attack-using-evil-twins-in-kali-linux
title: Man in the Middle attack using Evil twins in Kali-Linux !
categories:
- kali-linux
summary: How to setup and deploy a man-in-the-middle attack inside a WIFI network using Kali-Linux
---
**Man in the middle attacks (MITM)** are one of the easiest ways in which an attacker can steal user credentials from the victim. What basically attacker does is that, he will establish a connection with the victim somehow and will route the victim's traffic through him. Now the attacker can analyse the packets send by the victim to steal his credentials and other valuable things. One of the most common ways to do it is to use ettercap or _**Evil twin**'s attack in Kali-Linux. We will see how can we use **Evil twin** attack here.

In practical cases, when your PC scans for available Wi-Fi networks, if there are 2 networks with same SSID's (or same name) , then the PC will display only 1 which has stronger signal to your Wi-Fi card. This means that, if we are nearer to the victim than the router, then we can create a Wi-Fi hotspot with the same name and the victim's PC will automatically connect to us. Then we can use the **DHCP3** to route the traffic to Internet from our PC. So the victim will never come to know that we are playing in between him and the internet. Creating the _**Evil twin**_ of the real [SSID]({{ site.url }}/2014/09/28/find-hidden-ssids-using-backtrackkali-linux/) will do the trick, hence the name evil twin attack. Lets see how we can do man in the middle attack using evil twins :

1) Create an **airmon-ng** monitoring interface. Use this command to do that:

`sudo airmon-ng start wlan0`

This will start a monitoring mode normally mon0 or mon1. Now we need to dump the results of the monitoring to our terminal.

2) To dump the results, we use **airodump-ng**:

`sudo airodump-ng mon0`

This will show lots of info's on your terminal like available Wi-Fi networks, channels, users, Mac-address etc. Now from the list , for example, if there is a hotspot named "Public Wi-Fi", we need to create its evil twin.

3) To do that, we use **airbase-ng**:

`sudo airbase-ng --essid "Public Wi-Fi" -c 1 mon0`

Now this will start a new hotspot with the name "Public Wi-Fi" the evil twin of the real one. Now what we do is, send Deauth attack to the real Wi-Fi network so that all users connected to it will get disconnected and join to our evil twin.

4) To do a **deauth** attack, we use :

`sudo aireplay-ng -0 0 -a XX:XX:XX:XX:XX:XX mon0`


Now this will keep on giving deauth attack to the router until we press _**ctrl+c**_. _**XX:XX:XX:XX:XX:XX**_ is the mac-address of the router. So if we keep up the deauth attack , say for 20 secs, that's enough for users to get disconnected and reconnect to our evil twin network. Thats it. We got the victim connected to our PC.

Now you can configure DHCP3 to route the packets to the Internet or you can use Social Engineering Toolkit in Kali-Linux to steal passwords. You can also use Wireshark to do the same. Sometimes, for security reasons the SSID's will be hidden or the network will be filtered using Mac-address. If so you can read the posts on [how to bypass Mac-address filters]({{ site.url }}/2014/09/28/bypass-mac-address-filtering-using-backtrackkali-linux/) and [how can you find out Hidden SSID's using Kali-Linux]({{ site.url }}/2014/09/28/find-hidden-ssids-using-backtrackkali-linux/).