---
author: AnirudhAnand
comments: true
date: 2014-05-01 05:16:47+00:00
layout: post
slug: installing-skype-ubuntu-14-04-64-bit-trusty-tahr
title: Installing Skype in Ubuntu 14.04 64-bit (Trusty Tahr)
wordpress_id: 1511
categories:
- tutorials
- linux
summary: How to install skype in Ubuntu-14.04 64-bit system
---

Skype is a freemium instant messaging service which is currently developed by the Microsoft  team. Microsoft has acquired Skype two years back for a whooping $8.5 Million and ever since its update to Linux has been floppy. Skype is still used by Millions of people around the world and luckily Microsoft didn't discontinue the support of Skype for Linux. 

But a major problem is that Skype is released only for a 32-bit architecture and not for a 64 bit. So installing Sype on a 64-bit architecture takes a little more effort than the other. 
If you use a 32-bit Ubuntu, you can simply issue this command to download it:
    
   `sudo apt-get install skype`

If you use a 64-bit Ubuntu architecture, you have to add 32-bit architecture support and then update your system to install skype:
    
   `sudo dpkg --add-architecture i386`
    
   `sudo add-apt-repository "deb http://archive.canonical.com/ $(lsb_release -sc) partner"`
    
   `sudo apt-get update && sudo apt-get install skype`
    

The first command is to enable Multiarch support in Ubuntu for 64-bit users. Multiarch as the name suggest is enabled so that both the architecture will be supported by Ubuntu. Three years back, Skype has been added to canonical repository. So we have to manually add this repo to our repository. Then you can install Skype with the last command.