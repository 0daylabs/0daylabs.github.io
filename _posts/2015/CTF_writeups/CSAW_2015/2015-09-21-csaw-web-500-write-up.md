---
author: AnirudhAnand
layout: post
title: 'First Ashley Madison now Weebdate - CSAW 2015 web 500 writeup'
categories:
- webchallenges
- ctf-web
keywords: CTF, webhacking
summary: CSAW 2015 Web 500 challenge writeup
---

This challenge was pretty difficult and I should say even more difficult than Web600. So first I went to the `Weebdate` site and registered a new account. It games me a TOTP key which I should use to generate a one time login key and login with it. So I used a standard [python library](https://github.com/pyotp/pyotp) to get my key (which changes every 30 sec) and I logged in. I immediately clicked on `edit profile` and there we have an option to add images from other sources. Looks suspecious ? How about an LFI ? We cannot use `php://filter` since we are not even sure that we are dealing with php here. So I tried giving arbitrary inputs and I got an awesome error back: `Malformed url ParseResult(scheme='', netloc='', path=u'/', params='', query='', fragment='')`. That looks pretty interesting. So we need a scheme. How about `file://` ?

Well file protocol worked and I tried including apache conf files, and all those and ultimately reached here: `file://localhost/var/www/weeb/server.py` and that pretty much gave me the source code. So after going through it, I understood they are using some random`utils` files which is not a standard python file and I tried printing its source code: `file://localhost/var/www/weeb/utils.py` which has the functions we need.

{%highlight python lineos %}
def generate_seed(username, ip_address): 
    return int(struct.unpack("I", socket.inet_aton(ip_address))[0]) + struct.unpack("I", username[:4].ljust(4,"0"))[0] 

def get_totp_key(seed): 
    random.seed(seed) return pyotp.random_base32(16, random)
{%endhighlight%}

So these functions can return me the OTP. But now I need the password and Ipaddress. The only possible way to get it is from the database. I got stuck at this point for a long time and then got a link `http://54.210.118.179/csp/view/411` which downloads a JSOn. When I added a single quote to the URL, it gives me back internal server error which pretty much means that we got an sql injection point. So time to run sqlmap: `python sqlmap.py --level=5 --risk=3 -u "http://54.210.118.179/csp/view/3444" -a` which dumps me everything. I was eagerly searching for IP address which I got but instead of getting a password, I got a hash. Now I should crack the hash ?

I tried some common bruteforce approch guessing that password hash will be sha256(username+pass) and after a while I got the password which was `zebra`. Now we have everything we want to run the 2 functions that we got above. Just run it with the data we got and flag is ours !