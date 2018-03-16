---
author: AnirudhAnand
layout: post
title: 'Local File inclusion + File Upload = Remote Code execution - MMACTF 2015 web 300 writeup'
categories:
- webchallenges
- ctf-web
keywords: XSS, webhacking
summary: A combination of Local File inclusion + Arbitrary File Upload leads to Remote Code execution - MMACTF web 300 writeup
---

We are greeted with a page which has both register and a login option. After playing around a bit with it, I decided to register a new account and to my surprise I saw a file upload option (man I love file uploads). Also I noticed the URL structure which looks like this:

`http://magiagents.chal.mmactf.link/index.php?page=settings`

It seems they are including pages so I tried to use `php://filters` to print out the source code in base64 and it did :-).

`http://magiagents.chal.mmactf.link/index.php?page=php://filter/convert.base64-encode/resource=home`

Using the same method I got the source code for all the pages including `db.php`, `settings.php`, and `login.php`. From the `settings.php` I understood that the `sha1` of the uploaded image is taken as the filename and if it is has a jpg/png extension, it is also appended to the final filename and is stored in a directory `avators`. If it has any other extensions, it simply neglects it.
 
 One quick thing came to mind is to upload a php shell and include it via the LFI but a problem that I faced is that the the server was automatically appending `.php` extension to which ever filename and they are using the latest version of php where nullbyte `%00` injection will not work. Well, I didn't had a clue what to do next.
 
 I started looking at the other available php wrappers and I came across an interesting one called `zip:///path/to/filename#dir/file` and that looks pretty interesting. So I wrote a simple php shell, compressed it (.zip) and renamed it to `shell.jpg` and I uploaded it. So now I have my shell in the zip format. Then I included the location of the shell via zip wrapper:
 
 `http://magiagents.chal.mmactf.link/index.php?page=zip://avators/bi0sca70c2c8ad1dc9376848131c3b90d99f2d3833ee.jpg%23shell`
 
 So the zip intakes the image for extraction and its a valid zip. So it extracted, I gave a name `shell` to the extracted file and the server added the `.php` extension for me which easily executed. From the `home.php` source code, I knew the location of the flag and all I did was to use webshell to print it for me.
 
 The challenge was really amazing and I am excited to learn new concepts like this. Thanks for the MMACTF organizers for taking efforts to create good challenges.
