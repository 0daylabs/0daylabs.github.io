---
author: AnirudhAnand
comments: true
date: 2014-01-15 18:26:09+00:00
layout: post
slug: fix-gdm3-doesnt-start-installing-atiamd-drivers-debian
title: '[Fix] gdm3 doesn''t start after installing ATI/AMD drivers in Debian !'
wordpress_id: 939
categories:
- tutorials
- linux
summary: How to fix gdm3 problems while using ATI drivers in debian
---

Recently, I wrote an article on [**[How to] Install AMD/ATI drivers in Debian**]({{ site.url }}/2014/01/15/how-to-install-amdati-drivers-in-debian/). There in the article, I have warned my readers, it better to install the drivers by the way I have explained there and not by downloading corresponding drivers from their site and Installing it. Why I told you because if you install by downloading the driver from their site, there is a good chance that your gdm3 crashes while rebooting your machine. Now what if it is already happened ? Now your gdm3 really crashed , what to do ? Re-installing gdm3 or installing another desktop manager like KDE or lightdm won't work. Its because they also will have the same compatibility issue as before. So, if your gdm 3 already crashed, we have to Un-install the drivers and then reboot so that gdm3 will load again and then we have to re-install it in such a way thatt it won't cause a problem with the display manager. Now follow the steps to recover gdm3 :

1) Reboot your machine again and from **GRUB,** choose the Debian recovery mode ( usually the 2nd option) and login to the command line as a root.

2) Now go to `/usr/share` . There you can see several folders. They will be a folder named some what like "**AMD driver**" or "**ATI ....**. ". If you see a folder related to AMD, that will be the one where we have our _**Uninstall.sh**_ script. Now all we have to do is to run that script and reboot after it.

3) Your gdm3 will again start to work properly. To run the "_**Uninstall.sh**_", do the following commands :

{% highlight bash lineos %} 
   #Moving to the directory
    
   cd /usr/share
    
   #Giving the Shell the executable permission
    
   sudo chmod 777 uninstall.sh
    
   #Executing the script
    
   sudo ./uninstall.sh

{%endhighlight%}

That's it. Your gdm3 will again start to function properly from now. Now you can Install the Grpahics driver in the safe way so that it won't be a mess again. You can read my article to learn more about installing drivers the easy way : **[[How to] Install AMD/ATI drivers in Debian ?]({{ site.url }}/2014/01/15/how-to-install-amdati-drivers-in-debian/)**
