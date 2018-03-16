---
author: AnirudhAnand
comments: true
date: 2014-09-07 06:46:18+00:00
layout: post
slug: cors-how-cross-domain-call-works-in-browsers
title: 'CORS: How cross domain call works in browsers?'
wordpress_id: 5773
categories:
- javascript
summary: Usage and working of CORS in browsers.
---

**Cross Origin Resource Sharing (CORS)** is one of the most useful features of HTML5. CORS is a mechanism that allows many resources (e.g., fonts, JavaScript, etc.) on a web page to be requested from another domain outside of the domain where the resource originated from. In short, if you want  to use a JavaScript code inside your webpage which is already being saved as a .js file in another domain, instead of copying the code to the webpage again, you can do a Cross Domain call to the JavaScript file and can make use of it in the webpage. By default, such "cross domain" calls are not allowed by the browsers because of [same origin security policy](http://en.wikipedia.org/wiki/Same_origin_policy). We can achieve this with the help of XMLHttpRequest.

**XMLHttpRequest** is an API available to scripting languages like JavaScript which can be used for cross domain calls. It works in a simply way just like browsers, it sends an **HTTP** request to a web server and loads the results back to the webpage from which we called. While sending the request, the browser add an HTTP header which states the origin from which the request has been generated. The web server will receive this request and checks if the originated domain is allowed to access the requested resource.  Let us see an example where I host a script on a site which intakes a url/domain name as a parameter and returns its IP address:

{% highlight html lineos %}    
    var url = "http://examplesite.com/ip.php?domain=" + encodeURI(q);
    var http = new XMLHttpRequest();
    var ip = ''
    http.open("GET", url, true);
    http.responseType = "text";
    http.onreadystatechange = function() {
        if (http.readyState == 4 && http.status == 200) {
            ip = http.responseText
        }
    }
    http.send(null);
{% endhighlight%}


Let us see what the above script do. Let us assume that we have a JavaScript variable named "**q"** which contains the URL of which we need to get the IP address. So first we are creating a new variable named **url **in which we are creating the absolute path to where we host our ip.php which returns the corresponding IP address. The ip.php file looks like this:

{% highlight php lineos %}
    <?php
    header('Access-Control-Allow-Origin: *');
    header('Access-Control-Allow-Methods: GET');
    foreach ($_GET as $key)
        echo gethostbyname($key);
    ?>
{% endhighlight%}

Assume that the ip.php file is hosted in `http://examplesite.com/ip.php`. As you can see, ip.php takes a variable as an argument. So while creating an http request, we are sending the URL (of which we need to get ip) to the ip.php using a **GET** request. If you pass google.com as a URL, the request goes like this: **http://examplesite.com/ip.php?domain=google.com**. Now, lets get back to the main script. The statement **var http = new XMLHttpRequest()** will initiate a new request and store it in a variable named http.

The `onreadystatechange()` stores a function (or the name of a function) to be called automatically, each time the **readyState** property changes. When the readyState is equal to 4 and status is 200, the cross domain call request gets back its response and it can be stored into a variable.

The `readyState` equals to **4** means that the request is successfully completed (the ip.php function is correctly executed) and the response is ready.

While writing the `ip.php`, I included 2 statements `header('Access-Control-Allow-Origin: *')`; which is very necessary for this script to work. Here `Access-Control_Allow-Origin: *`  means that any domain can make a `cross domain call` (althrough this should don't be the case with real world applications) to this file. If you want to restrict it to limited number of sites, you can replace your domain name in the place of * . If this header is missing, you cannot make a cross domain call because by default, it will block all requests generating from external domain according to the [same origin security policy](http://en.wikipedia.org/wiki/Same_origin_policy).