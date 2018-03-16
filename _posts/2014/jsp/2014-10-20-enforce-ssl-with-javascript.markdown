---
author: AnirudhAnand
comments: true
date: 2014-10-20 06:56:06+00:00
layout: post
slug: enforce-ssl-with-javascript
title: 'Enforce SSL with JavaScript: Forcing browser to use "https:"'
wordpress_id: 262
categories:
- javascript
summary: How to force the browser to use the secure https protocol over the insecure http protocol.
---

Usually people don't care about the protocol they use to visit a website. Some websites like Facebook or Google+ will redirect you automatically to **SSL/TLS** encryption (**https://**) rather than using HTTP. The reason is because the SSL encryption will provide added security to your website and it will help us in blocking people  from spying us especially when we are on a public WiFi. For example, if we are using HTTP over a public network, other people can easily spy you with sniffer tools like **WireShark**. But if we are using an HTTPS protocol, even though people can still sniff with WireShark, there will be nothing useful in the packets as all the packets will be encrypted and protected with **SSL/TLS** encryption. So, to an extend you are safe.

So consider you have a website and you need to force redirect all "http:" traffic to "**https:"**. How can you do that ? The easiest way is to add the following code to your HTML head section of the root document (may be index.html ?  )

{% highlight javascript lineos %}   
    if (window.location.protocol != "https:")
    {
        window.location.href = "https:" + window.location.href.substring(window.location.protocol.length);
    }
{% endhighlight%}

The above code will force/redirect your browser to use "https:" if it is not already using. So lets see what this code does. First, it will check for the current protocol to see whether its using "https:" or not. **Window.location.protocol **will return the current protocol that we use. So we will check if it is equal to "https:". If not, we will redirect the browser to the same URL with the protocol being changed to "https:".

**Note:**

In the above code, we didn't use document.location and used window.location because window.location is more cross browser compatible. Even though now, all the browsers support document.location, people using old browsers will get compatibility issues. Since window.location works on all browsers, just to be on safe side, we used it.

Source: [Stackoverflow](stackoverflow.com/questions/4723213/detect-http-or-https-then-force-https-in-javascript)
