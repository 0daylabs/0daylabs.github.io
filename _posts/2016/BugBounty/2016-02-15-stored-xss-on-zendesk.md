---
author: AnirudhAnand
layout: post
title: Markdown based Stored XSS in Zendesk !
categories:
- bugbounty
summary: How markdown can help in triggering XSS ?
---

It all started after reading a bug report [#82725](https://hackerone.com/reports/82725) on hackerone, where an attacker can exploit markdown comment syntax by embedding malicious JavaScript. So the original bug report was to create malicious comments which when clicked can execute JavaScript, which looks like this:

{% highlight javascript lineos %}
[clickme](javascript:alert(1))
{% endhighlight %}

Now that was a simple markdown XSS and I was disappointed as I was not the one to find it first. So I decided to play with the same bug and see if I can come up with a bypass for it. But unfortunately, things were not as easy as it looked. Zendesk team did a good work fixing it and no matter what I tried, it didn't work. Then I came across an [awesome blog](http://shubh.am/exploiting-markdown-syntax-and-telescope-persistent-xss-through-markdown-cve-2014-5144/){:target="_blank"} by Shubham Shah where he explains a lot of good markdown based XSS vectors which includes:

{% highlight javascript lineos %}
[clickme](javascript:prompt(document.cookie))
[clickme](j    a   v   a   s   c   r   i   p   t:prompt(document.cookie))
![clickme](javascript:prompt(document.cookie))\
<javascript:prompt(document.cookie)>  
<&#x6A&#x61&#x76&#x61&#x73&#x63&#x72&#x69&#x70&#x74&#x3A&#x61&#x6C&#x65&#x72&#x74&#x28&#x27&#x58&#x53&#x53&#x27&#x29>
{% endhighlight %}

But none of the above seemed to be working (Zendesk filters are good !). I was about to stop looking, but suddenly something crossed my mind. The original report was based on XSS vectors which starts with `javascript:` which they obviously blocked with a strong filter after the bug was resolved. So I should be looking for a payload which can trigger JavaScript but should not use the word `javascript` inside the payload. Can you guess a payload (it's so easy ) ?

{% highlight javascript lineos %}
[clickme](data:text/html;base64,PHNjcmlwdD5hbGVydCgnWFNTJyk8L3NjcmlwdD4K)
{% endhighlight %}

I knew this, but it never crossed my mind until I read Shubham's blog. The simple base64 payload with `data:text/html;base64` executes properly in any browser. As the last report was fixed by filtering out the keyword `javascript` and not all generic markdown based XSS, I bypassed their existing filters and triggered an XSS using the above payload(A very simple one right ?).

Zendesk team was kind enough to pay me $500 for the report.