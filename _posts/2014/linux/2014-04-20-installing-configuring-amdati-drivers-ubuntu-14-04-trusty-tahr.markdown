---
author: AnirudhAnand
comments: true
date: 2014-04-20 10:08:05+00:00
layout: post
slug: installing-configuring-amdati-drivers-ubuntu-14-04-trusty-tahr
title: Installing and Configuring AMD/ATI drivers in Ubuntu 14.04 (Trusty Tahr)
wordpress_id: 1476
categories:
- tutorials
- linux
summary: How to solve the problems that occur in Ubuntu while installing AMD/ATI drivers
---

Ubuntu fans will be very excited with a more stable release of Ubuntu 14.04 ( trusty Tahr) which is much better and stable than previous Ubuntu releases especially 13.xx series. But as usual, configuring graphics card is always a headache for any people with a dedicated Graphics card other than the normal Intel card. The situation becomes worse for people with AMD Processors and Dual graphics (Hybrid graphics) because overheating will happen as a result of un-configured Graphics cards. Also, configuring isn't an easy. As always, you will always come up with dependency clashes, driver issues with desktop managers like **gdm**, **lightdm**  etc.. So I will share with you my experience on how I installed and configured graphics drivers successfully on my **AMD Dual graphics** enabled notebook.

The recommended method in AMD site won't work here because installing directly by executing the the catalyst.run file will create dependency clash with the desktop manager  and will result in failing the loading of desktop manager after the reboot.

**Easy Method: (might not work sometimes)**

You can take **Additional drivers** option that comes along with **Software & updates** in Ubuntu which will show you the available drivers (both **proprietary** and open source) and you can choose any one of them. If you need to have more control on your graphics card, it is recommended to go for the Proprietary install.

**Manual Method:**

Sometimes, installing through additional drivers will fail. If thats the case, we have to do this manually via command line (like I did because for me, Additional driver installation failed):
	
  * Remove/purge current **fglrx** and **fglrx-amdcccle**. We have to reconfigure and install this again for things to work. To do that, follow the commands:

{% highlight bash lineos %}     
    sudo apt-get remove --purge fglrx fglrx-amdcccle
    sudo apt-get remove --purge fglrx-updates fglrx-amdcccle-updates
{%endhighlight%}
	
  * Now reboot the system and login back again.
	
  * Install the Linux generic headers (It comes along with 14.04 but one of my friends had to do this before continuing or else he was getting errors. So its better to try installing this):

{% highlight bash lineos %} 
    sudo apt-get install linux-headers-generic
{%endhighlight%}
   
  * Now lets install back the drivers
 {% highlight bash lineos %} 
    sudo apt-get install fglrx fglrx-amdcccle
{%endhighlight%}

  * Once the installation is complete, make sure you give this command before rebooting
  {% highlight bash lineos %} 
    sudo aticonfig --initial
{%endhighlight%}

  * If you are using AMD switchable graphics (just like I do) instead of the above command, give this
{% highlight bash lineos %}     
    sudo aticonfig --adapter=all --initial
{%endhighlight%}
	
  * Reboot once again and hurrayyyy.. Once you login back, you can configure your graphics card like you want. :D

Hope this method works for you. If you encounter any problems in between, do comment down and we are happy to help you.