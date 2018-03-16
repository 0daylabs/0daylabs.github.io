---
author: AnirudhAnand
layout: post
title: 'Bypassing PHP strcmp() - CSAW 2015 web 200 writeup'
categories:
- webchallenges
- ctf-web
keywords: CTF, webhacking
summary: CSAW 2015 Web 200 challenge writeup
---

Well this challenge was very easy and I guess it has the highest number of solves in web category during CSAW 2015. Here we have a simple login form and and we should login to get the flag. But the register is not working and SQL injection also doesn't seems to work. So I was wondering how can the check be in server side and if it is using php function `strcmp()` then things would be pretty easy. So I tried capturing the request with tamper data and replaced the `password=lol` with `password[]=lol` and it easily bypassed the authentication. And yea it gives back the flag too :)

FOr those who don't know how this works, lets say we have an example code like this:

{%highlight python lineos %}

$username = $_POST['username'];
$password = $_POST['password'];

$real_password = "original password here";

if (strcmp($password, $real_password) == 0) {
 
 echo "flag{}";
}

{%endhighlight%}

If this is the code, if we give post request like this `password[]=lol` then the $password becomes an array. Now comparing this, instead of throwing an error, it returns NULL and in PHP NULL == 0, which means string comparison passed and we got the flag :)