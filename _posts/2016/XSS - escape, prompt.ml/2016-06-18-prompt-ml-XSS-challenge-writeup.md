---
author: AnirudhAnand
layout: post
title: Prompt.ml - XSS Challenges writeup
categories:
- webchallenges
- xss
keywords: XSS, webhacking
summary: If you haven't seen this already, this is a series of <a href="http://prompt.ml">XSS challenges</a> by <a href="https://twitter.com/filedescriptor">Filedescriptor</a>. The challenges were really good and if you haven't attempted to solve it, you should definitely try yourself before reading the writeups here.
---

This is another great XSS challenge series inspired by [alert(1)](https://blog.0daylabs.com/2016/06/17/escape-alf-nu-XSS-writeup-part-1/) but with a more greater difficulty. I have learned a lot solving these challenges and I strongly recommend you try solving the challenges yourself before reading the writeups here.


#### Level 0:

{% highlight javascript lineos %}

function escape(input) {
    // warm up
    // script should be executed without user interaction
    return '<input type="text" value="' + input + '">';
}

{% endhighlight %}

Like always it starts with a simple warmup challenge without any filters. Just escape out of the attribute context and insert your own payload.

Payload: `"><svg/onload="prompt(1)`


#### Level 1:

{% highlight javascript lineos %}

function escape(input) {
    // tags stripping mechanism from ExtJS library
    // Ext.util.Format.stripTags
    var stripTagsRE = /<\/?[^>]+>/gi;
    input = input.replace(stripTagsRE, '');

    return '<article>' + input + '</article>';
}

{% endhighlight %}

Now this is a modified version of the first one but with an included filter from an ExtJS library which strips the tags. The important thing to understand here is the REgex: `/<\/?[^>]+>/gi;`. This REgex matches a string which starts with `<`, which may or may not be followed by `/` (not necessarily), then can contain any characters other than `>` and finally it should have a closing tag `>`.

So the point here is to note that the string matches the regex only if it contains a closing tag `>`. Otherwise it won't match and strip it. So the best thing to do is to inject a tag without the closing `>` character (commented out the rest).

Payload: `<svg/onload=prompt(1)//`


#### Level 2:

{% highlight javascript lineos %}

function escape(input) {
    //                      v-- frowny face
    input = input.replace(/[=(]/g, '');

    // ok seriously, disallows equal signs and open parenthesis
    return input;
}

{% endhighlight %}

This was a pretty good challenge where I learned a new concept. Here it filters `=(` which mean we cannot use either one of them. We can ofcourse inject tags but since prompt() function needs to use `(`, I wasted a lot of time thinking about how to bypass this (using encodings, backticks (in IE) etc..) but nothing works. But as always, [mario (0x6D6172696F)](https://twitter.com/0x6D6172696F) really made my day when I saw his bypass payload (Its amazing to see the way he thinks ! I am a big fan of him :P).

The solution here is that if we inject the script tags inside an `svg`, then we can use hex encoding or html encoding to bypass the filter because of the XML parsing property. The elements inside the `svg` is considered to be XML elements and XML parsing is carried out with it in which we can use `&lpar;` or `&#40` as an alternative to `(` and the parser will convert it into `(` at the time of execution.

You can verify this by giving 2 differeny payloads. If we give `<script>prompt&lpar;1)</script>`, this will not fire the prompt(1) as its simply a script context where no explicit decoding is done by the javascript engine. But if we enclose it within an `svg` tag, you can see that it fires the payload, making us execute the prompt(1). Thanks [mario](https://twitter.com/0x6D6172696F) !

Payload: `<svg><script>prompt&lpar;1)</script>`


#### Level 3:

{% highlight javascript lineos %}

function escape(input) {
    // filter potential comment end delimiters
    input = input.replace(/->/g, '_');

    // comment the input to avoid script execution
    return '<!-- ' + input + ' -->';
}

{% endhighlight %}

This one was fairly easy. We can see that whatever userinput comes in, it goes inside the HTML comments so the only way to solve this is to escape out of comment context and inject our tags. But since `-->` is filtered, I had no clue on how to solve this but luckly `--!>` helped. This is when I understood that there are 2 ways to close a HTML comment.

Payload: --!><svg/onload=prompt(1)


#### Level 4:

{% highlight javascript lineos %}

Text Viewer
function escape(input) {
   // make sure the script belongs to own site
   // sample script: http://prompt.ml/js/test.js
   if (/^(?:https?:)?\/\/prompt\.ml\//i.test(decodeURIComponent(input))) {
       var script = document.createElement('script');
       script.src = input;
       return script.outerHTML;
   } else {
       return 'Invalid resource.';
   }
}

{% endhighlight %}
