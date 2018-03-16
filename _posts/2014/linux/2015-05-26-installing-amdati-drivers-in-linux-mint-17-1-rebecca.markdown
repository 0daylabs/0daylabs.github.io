---
author: AnirudhAnand
comments: true
date: 2015-05-26 15:34:30+00:00
layout: post
slug: installing-amdati-drivers-in-linux-mint-17-1-rebecca
title: '[How to] Install ATI AMD drivers in Linux Mint 17.1 Rebecca'
wordpress_id: 5901
categories:
- linux
- tutorials
summary: Instructions on installing ATI/AMD drivers in linux Mint 17.1
---

Linux Mint, as you all know, is one of the most popular Linux distribution till date and it is rated as the most downloaded Linux OS by [distrowatch](http://distrowatch.com/dwres.php?resource=popularity). The very recent version of Linux Mint, called "**Rebecca**", is a long term support edition from the Mint team till 2019. Rebecca comes with several new features that makes the desktop experience smoother than ever. It comes pre-packed with several improvements which include system performance improvement, more privacy and language settings, and improved hardware support.

As always, one of the biggest challenges for a person installing a latest Linux version is that, it should support the graphics card drivers. And if you mess up with graphics configuration, you will end up crashing the xserver and it will fail to start. So here is a good way to install AMD/ATI drivers properly in Linux Mint without affecting any of the configuration:

  * Before installing the driver, make sure you install** Linux generic headers**.
    
    `sudo apt-get install linux-headers-generic`

  * Linux Mint doesn't come with fglrx which is actually good for us, as we don't have to remove it and then re-install it like the way we do in **[Ubuntu 14.04]({{ site.url }}/2014/04/20/installing-configuring-amdati-drivers-ubuntu-14-04-trusty-tahr/)**.
	
  * Install **fglrx** via the command below:
    
    `sudo apt-get install fglrx`
	
  * If it shows any dependency problems, then do an apt-clean and try again. Execute the following commands and then try again.
    
    `sudo apt-get autoclean`

    `sudo apt-get update`
	
  * If fglrx is installed properly, it is time to install the GUI for controlling the graphics properties. Install fglrx-amdcccle (This is very similar to catalyst control center):
    
    `sudo apt-get install fglrx-amdcccle`
	
  * Now let us generate a fresh xorg file (if you are having multiple AMD graphics card or dual graphics, use the next command):

    
    `sudo amdconfig --initial`

  * If you are having multiple AMD cards or Dual graphics card, use the following command to generate new xorg file:
    
    `sudo amdconfig --adapter=all --initial`

  * Now you can reboot the machine and login to see the drivers working perfectly.

**Note:**

The above method has already been experimented on the latest Linux Mint 17.1 Rebecca release and I am sure it will work well without any problems for you. Also, the same method will work with Linux Mint 17 quiana too.

If you come across any problems while doing so, let us know via comments and we are happy to help.