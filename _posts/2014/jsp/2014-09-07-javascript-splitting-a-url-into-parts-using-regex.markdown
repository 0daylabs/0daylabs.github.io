---
author: AnirudhAnand
comments: true
date: 2014-09-07 06:24:24+00:00
layout: post
slug: javascript-splitting-a-url-into-parts-using-regex
title: 'Javascript: Splitting a URL into parts using Regex'
wordpress_id: 5772
categories:
- javaScript
summary: How to split a URL into different parts in JS by using regex
---

**JavaScript** (js) is a lightweight, interpreted, object-oriented language with [first-class functions](https://en.wikipedia.org/wiki/First-class_functions), most known as the scripting language for Web pages. Most of the web application developers, designers etc.. use js on a daily basis. Recently I got selected for [Google Summer of code 2014](https://www.google-melange.com/gsoc/homepage/google/gsoc2014), and a part of my project was related to JavaScript where I had to make use of JavaScript for splitting a URL into corresponding parts. When I googled this, I came across many tutorials but later on I set down to write a function making use of the powerful JavaScript Regex (thanks to my friend, Arjun, who helped me in writing this). Here is the code that I wrote:

{% highlight javascript lineos %} 
    function subDomain(url) {
        url = url.replace(new RegExp(/^s+/), "");
        url = url.replace(new RegExp(/s+$/), "");
        url = url.replace(new RegExp(/\/g), "/");
        url = url.replace(new RegExp(/^http://|^https://|^ftp:///i), "");
        url = url.replace(new RegExp(/^www./i), "");
        url = url.replace(new RegExp(//(.*)/), "");
        if (url.match(new RegExp(/.[a-z]{2,3}.[a-z]{2}$/i))) {
            url = url.replace(new RegExp(/.[a-z]{2,3}.[a-z]{2}$/i), "");
        } else if (url.match(new RegExp(/.[a-z]{2,4}$/i))) {
            url = url.replace(new RegExp(/.[a-z]{2,4}$/i), "");
        }
        var subDomain = (url.match(new RegExp(/./g))) ? true : false;
        return (subDomain);
    }
    function parseUrl(url) {
        var parser = document.createElement('a');
        parser.href = url;
        var datas = [parser.hostname, parser.port, parser.pathname];
        return (datas);
    }
    function printer(test_var) {
        var k = "";
        k = test_var.toString();
        var k1 = addhttp(k);
        q = k1[0];
        scheme = k1[2];
        k = k1[1];
        k = parseUrl(k);
        return k;
    }
    function addhttp(url) {
        var k3 = ""
        if (url.match(/http:///)) {
            url = url.substring(7);
            k3 = "http://"
        }
        if (url.match(/https:///)) {
            url = url.substring(8);
            k3 = "https://"
        }
        var k = "";
        k = url;
        if (url.match(/^www./)) {
            url = url.substring(4);
        }
        if (!/^(f|ht)tps?:///i.test(url)) {
            url = "http://" + url;
        }
        var k2 = [k, url, k3]
        return k2;
    }
    //var q = 'https://demo.testfire.net:1234/hey'
    //var output = document.getElementById('output');
    var temp = "";
    var hosts = "";
    var port = "";
    var scheme = "";
    var path = "";
    var sub = "";
    var datas = printer(q);
    sub = datas[0];
    port = datas[1];
    path = datas[2];
    if (!subDomain(sub)) {
        hosts = sub;
    } else {
        hosts = sub.split(".")[1] + "." + sub.split(".")[2];
    };
    if (!port) {
        port = 80;
    }
    if (!scheme) {
        scheme = 'http://';
    }
{% endhighlight %}

One obvious question that comes to everyone's mind is that, what the hell does this code do ?. I can explain that. Even though this code looks very huge, its functionality is very simple. Here I'm assuming that you have the URL to split it in a variable named "**q**". If so, the URL is passed as an argument to the function printer and the splitting occurs. After the splitting, 5 new JavaScript variables are created namely **scheme**, **host**, **port**, **path **and **sub**.

For example, consider that the URL saved in the variable **q** is "**http:/security.securethelock.com:443/hey**". After the splitting, the 5 new variables will be created with the following values in it:

**scheme**: http://

**host**: securethelock.com

**port**: 443

**path**: hey

**sub**: security.securethelock.com

If **port** is not specified, the default value, 80,  will be stored. If q is a main domain (and not a subdomain) like "http://www.securethelock.com", **sub = host =** securethelock.com . If path is not specified, the corresponding variable will be empty.
