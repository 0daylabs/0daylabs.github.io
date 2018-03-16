---
author: AnirudhAnand
layout: post
title: (How to) Configuring OpenVPN in Linux
categories:
- tutorials
- linux
summary: Installing and configuring OpenVPN in Linux
---

**OpenVPN** is a open source application that enables you to configure Virtual Private Network (VPN). For those who doesn't know about OpenVPN, it is used to create secure P2P connections and remote access facilities. So it ensures that the users are safe from hawks who try to sniff or eavesdrop your network.

Using OpenVPN is very simple. In most of the distributions it is installed by default. If it is not, you can download from the repository by doing a simple,

`sudo apt-get install openvpn`

After you have installed OpenVPN,  you need VPN servers to be configured to use with it. Generally it is not free. But there are some sites which provide it for ***free*** (Yeah,very good people :D). There are many sites out there. But I found better than others (the reason being *no* signing up/registering, comparatively faster than other VPN providers).

So just go to download any of the servers available. Next step is to extract the file and save in some location,

`tar -xvf filename`

Now,open your terminal (**Remember,you need to be a super user to perform this actions. Else you will get errors while starting your connection**) and don't forget to switch to the folder which has the extracted files. Let's here take an example of OpenVPN-Euro1 server. Next step will be to, configure it:

`openvpn --config vpnbook-euro1-tcp443.ovpn`

Use **tcp443** always. In case it doesn't connect try using the **udp**. After executing the command, you would be prompted for authentication. The username is vpnbook and you can find the password. VPNBook often changes the password to make sure that this is used by people only who are active (The passwords change once in two weeks if I am not wrong). After this you would get a line stating,

**Sequence Initialized.**

That's it you are done. If you face any difficulties while configuring, don't hesitate to comment down. We are happy to help.. :)
