---
author: AnirudhAnand
layout: post
title: 'Breaking the CTF framework - CSAW 2015 web 600 writeup'
categories:
- webchallenges
- ctf-web
keywords: CTF, webhacking
summary: CSAW 2015 Web 600 challenge writeup
---

Well, after I saw the challenge for the first time, I was like "dafaq! Should we find a zero day in CSAW CTF Framework now ?". I immediately searched their framework source code hoping it to be open sourced and I was correct. I got the source code from their [github](https://github.com/isislab/CTFd/) and now what ? Its hell long and I don't wanna sit and read the entire thing so I looked into the issues and recent commits. I was damn sure that this has to do something with a recent commit. Then one of the commits got my attention: [commit](https://github.com/ajvpot/CTFd/commit/9578355143d7af675fc4776b0f2de802be91e261)

And the commit msg was: `Fix authentication for certain admin actions`. That itself is suspicious right ? Well atleast to me it was. So here is an interesting part of the code:


{%highlight python lineos %}

@admin.route('/admin/chal/new', methods=['POST'])
@admins_only
def admin_create_chal():
    files = request.files.getlist('files[]')
    # Create challenge
    chal = Challenges(request.form['name'], request.form['desc'], request.form['value'], request.form['category'])
    db.session.add(chal)
    db.session.commit()

{%endhighlight%}

Well, lets visit the URL and I got a message saying `Method not Allowded` and I didn't get redirected to a login page ~! Strange isn't it ? For any of the URL other than the 3 URL's, if we try to access `/admin/` its gonna redirect you to login but not this, interesting ! Lets try giving other types of requests then. So I fired up the python terminal and did a post request, and there is the flag in the response !

{%highlight python lineos %}

import requests

req = requests.post(https://ctf.isis.poly.edu/admin/chal/delete)
req.text

{%endhighlight%}

Now that was a pretty easy challenge. :)