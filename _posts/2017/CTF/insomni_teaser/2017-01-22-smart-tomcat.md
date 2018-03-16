---
author: AnirudhAnand
layout: post
title: Exploiting internal tomcat server with SSRF - Insomnihack teaser 2017 Web 50 writeup
categories:
- ctf
summary: Exploiting internal tomcat server (with default credentials) using SSRF (Insomnihack teaser 2017 Web 50 writeup)
description: Exploiting internal tomcat server (with default credentials) using SSRF (Insomnihack teaser 2017 Web 50 writeup)
---

## Introduction

After a break I started participating in CTFs again (The new year resolution was to participate in every single CTFs this year, lets see.. :P) and this year's first CTF was Insomnihack teaser. The CTF was overall good and I guess most of the teams enjoyed playing it. So here is the writeup of Web 50:

## Challenge Description:

>Normal, regular cats are so 2000 and late, I decided to buy this allegedly smart tomcat robot
>Now the damn thing has attacked me and flew away. I can't even seem to track it down on the broken search interface... Can you help me ?
>
>[Search interface](http://smarttomcat.teaser.insomnihack.ch)

We are given a search interface in which we can enter the co-ordinates (latitude and longitude) and it searches for the missing cat. I decided to see how the parameters are getting send using burp and the results were interesting !

There is a parameter named `u` with the following value `http://localhost:8080/index.jsp?x=1&y=1` when we enter `1` for both search values. This looks quite interesting so the first thing that came to my mind was RFI (Remote File Inclusion). Althrough I tried several methods to get code execution via RFI, it all failed so I decided to check the question from a different angle.

The name of the challenge is `smart tomcat` so there has to be something related to tomcat. So I tried to access an invalid page in the server which responded me back with the tomcat standard error page with version number (7.0.68) but that didn't really lead me anywhere.

## Solution

The next best thing to check is sensitive directories within tomcat and a quick Googling told me there is a tomcat manager usually on `/manager` path on the server. So lets try visit the same with `u=http://localhost:8080/manager` but unfortunately it was protected with HTTP basic authentication and we are not given any credentials ! After all this is a 50 point problem so it had to have the default credentials isn't it ?

So lets try logging in with the default tomcat credentials which is `tomcat:tomcat` but how do we send the credentials ? We don't have direct access to the server (that is we can visit it only via the `u` parameter and a direct access to it over browser URL was blocked) but we do can send the HTTP basic credentials over the URL itself in a standard format.

So finally, when we send `u=http://tomcat:tomcat@localhost:8080/manager/html` it prints out the flag ! You can also do the same thing with curl: `curl -d "u=http://tomcat:tomcat@localhost:8080/manager/html" http://smarttomcat.teaser.insomnihack.ch/`

## References

1) [https://github.com/psmiraglia/ctf/blob/master/kevgir/001-tomcat.md](https://github.com/psmiraglia/ctf/blob/master/kevgir/001-tomcat.md)
