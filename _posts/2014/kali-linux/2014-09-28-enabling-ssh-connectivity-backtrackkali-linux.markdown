---
author: AnirudhAnand
layout: post
title: '[How to] Enabling ssh connectivity in Backtrack/Kali Linux'
categories:
- Kali-Linux
summary: 
---

**Backtrack/Kali ** is the most advanced Linux based penetration testing distribution till date. It contains pre-installed tools from Information gathering to advanced **wireless attacks** and **Forensics**. We are launching a series of tutorials that will start from the basics and will cover the most advanced attacks with Backtrack/Kali.

Let us first enable **ssh** connectivity on Backtrack/Kali. By default, **ssh** connectivity will not be enabled. We have to enable it manually. Now lets us see how can we do that. Before making our system able to receive ssh, we should first **create/Generate ssh private and public key.** Generating these keys are very simple. Just use the command

`sshd -generate`

And the Backtrack/Kali will automatically create it for you. After creating the keys, we have to tell the backtrack/Kali to accept ssh connections. To do that, use either one of the following command:

`/etc/init.d/ssh start 
    or
start ssh`

(It is always better to use the first command as I have heard people complaining that the second command is simply not working.) Now you have successfully taught Backtrack/Kali to accept ssh connection. But still there is a problem. Once you reboot the system, it will forget these settings. In other words, you are enabling ssh temporarily and once system reboots, it will forget all the settings.

So what should we do so that it will remember these settings forever ? Use the following command to update and permanently store the ssh settings :

`update-rc.d -f ssh defaults`

So you will be wondering what this command will do? Basically what it does is, it will over write the default ssh settings to the new one you just did. So this will ensure that your the changes will be permanent and will not perish if system reboots.

**Note:**

While enabling **ssh**, you should make sure that your system users has really strong **alpha - numeric** passwords especially root user. It is because an attacker can do a brute force attack over ssh and if your password is not strong enough, he can break it and can cause system damage or theft of your precious data.