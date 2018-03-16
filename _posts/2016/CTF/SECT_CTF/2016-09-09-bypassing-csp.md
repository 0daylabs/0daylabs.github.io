---
author: AnirudhAnand
layout: post
title: Bypassing path restriction on whitelisted CDNs to circumvent CSP protections - SECT CTF Web 400 writeup
categories:
- ctf
summary: How can we bypass CSP using whitelisted CDNs and path traversal (SECT CTF 2016 web 400 writeup)
description: How can we bypass CSP using whitelisted CDNs and path traversal (SECT CTF 2016 web 400 writeup)
---

## Introduction

So it all started when I was casually browsing twitter and noticed a tweet from [@avlidienbrunn](https://twitter.com/avlidienbrunn):

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Made some CTF challenges for <a href="https://twitter.com/SEC_T_org">@SEC_T_org</a> CTF at <a href="https://t.co/wEwrgeLEic">https://t.co/wEwrgeLEic</a> check it out if you have spare time :) <a href="https://twitter.com/hashtag/ctf?src=hash">#ctf</a> <a href="https://twitter.com/hashtag/web?src=hash">#web</a> <a href="https://twitter.com/hashtag/pwn?src=hash">#pwn</a></p>&mdash; ­Mathias Karlsson (@avlidienbrunn) <a href="https://twitter.com/avlidienbrunn/status/773789219613503488">September 8, 2016</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

Ok, so there is a CTF going on (which was not listed on CTFtime.org) and since avlidienbrunn created the web challenges, I decided to take a look because I was sure that the challenges would be really good. And it turns out that I was not mistaken. Indeed great challenges :)

## Challenge:

I was able to solve some quick XSS challengs but then I came across a challenge which was vulnerable to Reflected XSS but is protected by CSP:

Challenge URL: [http://xss3.sect.ctf.rocks/?xss=stuff](http://xss3.sect.ctf.rocks/?xss=stuff)

A quick injection revealed that there was no filter and whatever we inject is simply echoed back at the end of the response. But an XSS is not possible (as of now) because the CSP blocks it. The CSP header looks like this:

`Content-Security-Policy: default-src 'none'; style-src 'self'; img-src 'self'; script-src https://cdnjs.cloudflare.com/ajax/libs/jquery/`

So we can see that `cdnjs` is actually whitelisted CDN, which means we can load any files from that server. But this was not as easy as I thought it would be. If we could import outdated Angularjs from the server, we could use it to execute JavaScript but what is the use of importing `jquery` ? The main issues I faced here were:

1) Importing `jquery` was useless because we could import a vulnerable version of `jquery` from the server but what is the use if we cannot add our own jquery code into the DOM after this import ?

2) Since the URL in the CSP contains the path (`/ajax/libs/jquery/`), I didn't know any possible way to load other files like Angularjs from the same server (which resides on `/ajax/libs/angular.js/`) which could have made our work much easier.


## Solution:

Since loading `jquery` is useless, the only way is to somehow import outdated Angularjs from the server, so I started trying path traversal so that I can try loading from other directories. I literally fuzzed the application until it gave an interesting response. Typical injections like `../` and `..%2f` didn't work but, to my surprise, double encoding worked ! HELL YEAH !

So when we try to load `<script src="https://cdnjs.cloudflare.com/ajax/libs/jquery/..%252fangular.js/1.0.8/angular.js"></script>`, firefox loaded the old version of Angularjs properly for me ! I never knew that double encoding could bypass CSP protections on whitelisted CDN path !

Cool, so now we can load Angularjs and outdated versions of it can actually help us in bypassing the CSP with `ng-onclick` instead of javascript event handlers (Thanks to [Mario Heiderich](https://github.com/cure53/XSSChallengeWiki/wiki/H5SC-Minichallenge-3:-%22Sh*t,-it's-CSP!%22)). So using this, I successfully created a payload which can execute `alert(1)` but with a minor problem. It actually needs a small user interaction i.e, a click from the user. But this will not be accepted as a solution because final payload should run without user interaction.

Payload: `?xss=<body class="ng-app"ng-csp ng-click=$event.view.alert(1337)><script src="https://cdnjs.cloudflare.com/ajax/libs/jquery/..%252fangular.js/1.0.8/angular.js"></script>`

Now things became really difficult. So I decided to visit [mario's challenge](https://github.com/cure53/XSSChallengeWiki/wiki/H5SC-Minichallenge-3:-%22Sh*t,-it's-CSP!%22) again to see how people actually solved the challenge with payload without user interaction and one of them looked really cool:

{%highlight javascript lineos %}
"ng-app ng-csp>
<base href=//ajax.googleapis.com/ajax/libs/>
<script src=angularjs/1.0.1/angular.js></script>
{% raw %}
<script src=prototype/1.7.2.0/prototype.js></script>{{$on.curry.call().alert(1337
{% endraw %}
{%endhighlight%}

So if we can also load the `prototype.js`, we can use `curry` property along with the `call()` which returns us a `window` object and inturn use it to call our little `alert(1)` without any user interaction !

So here goes the final payload:

{%highlight javascript lineos %}

<script src="https://cdnjs.cloudflare.com/ajax/libs/jquery/..%252fprototype/1.7.2/prototype.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jquery/..%252fangular.js/1.0.1/angular.js"></script>
<div ng-app ng-csp>
{% raw %}
{{$on.curry.call().alert(1)}}
{% endraw %}
</div>

{%endhighlight%}

This was indeed a great challenge from [@avlidienbrunn](https://twitter.com/avlidienbrunn). I throughly enjoyed playing the CTF :)


## References:
These are some of the awesome references you can use to learn more about similar challenges:

1. [Shit, it's CSP!](https://github.com/cure53/XSSChallengeWiki/wiki/H5SC-Minichallenge-3:-%22Sh*t,-it's-CSP!%22)
