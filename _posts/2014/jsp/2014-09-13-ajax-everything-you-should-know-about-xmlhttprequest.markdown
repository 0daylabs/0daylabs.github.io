---
author: AnirudhAnand
comments: true
date: 2014-09-13 16:17:34+00:00
layout: post
slug: ajax-everything-you-should-know-about-xmlhttprequest
title: 'Ajax: Everything you should know about XMLHttpRequest() '
wordpress_id: 50
categories:
- javascript
---

**Ajax** (Asynchronous Javascript and XML)  is  web development technique for creating better, faster, and more interactive web applications using which they can send and retrieve data from the server asynchronously (in the background) without interfering with the display and behavior of the existing page.

So what does this really mean? Consider you have a webpage in which you have some information in the sidebar which needs to be loaded when user clicks or hovers above it. In a normal case, if new information has to be added to the web page, it must give another request to the server and the entire web page should get reloaded. That consumes a lot of time/bandwidth. But if we use Ajax, the browser sends a request to the server but only necessary information is send back by the server and is displayed in the web page without getting reloaded.

So in practical how does this work ? This is being done by using **XMLHttpRequest()** object. Let us look into a sample code to understand how XMLHttpRequest() works generally.
    
{% highlight html lineos %}  
    var request = new XMLHttpRequest();
    request.open('GET', 'info.txt', false);
    request.send();
    console.log(request);
{% endhighlight%}

This is a very simple XMLHttpRequest(). What it does is, it creates an object named **request**. Then by using the object, we call the function **request.open() **with the 3 must have arguments namely:

1)  The type of HTTP request (GET/POST)

2) The content to be retrieved

3) Specifying whether to use synchronous or asynchronous request.

In the above code, I used synchronous method to access data (we will get into synchronous and asynchronous types later in this post). In order to see how the code works, I added an extra line to the code like this: **console.log(request); . **What it does is that, it dumps the information about the request into the browser console window and we can see if there is any issues with the request we made (makes it easier for debugging). A successful request console log looks something like this:

From this you can see that when a request becomes successful when **readyState** =4 and **status** = 200. The responseText property always stores the test of the response. So in order to write the response to the webpage, we can modify the above code like this:

{% highlight html lineos %}
    var request = new XMLHttpRequest();
    request.open('GET', 'info.txt', false);
    request.send();
    console.log(request);
    if(request.status === 200)
    {
       document.writeln(request.responseText);
    }
{% endhighlight%}

This will write the responseText into the webpage. Using **document.writeln** is not recommended to write any of the responses to webpages as it can get exploited to **Cross Domain XSS.** But here, let us use the method and in future tutorials, we will use more advanced techniques. Now let us see what is the difference between synchronous and asynchronous requests.


In the code above, we give the 3rd parameter of **open()** function as '**false' **to make it **synchronous** request. While doing a synchronous request, browser will wait until it receives the response or the function completes. But this is not necessarily a good option. let us see why. Consider you need to send around 100 requests like this:

{% highlight html lineos %}
    for( var i = 0; i < 100; i++ )
    {
      var request = new XMLHttpRequest();
      request.open('GET', 'info.txt', false);
      request.send();
      console.log(request);
      if(request.status === 200)
      {
         document.writeln(request.responseText);
      }
    }
{% endhighlight%}

Try executing the above code. You will see that browser will take sometime before showing the output. This happens because browser is waiting to show the output once it completes the execution. This will take considerable amount of time to display if the requests are larger than 100. So a better way is to write asynchronous requests.

Ajax (Asynchronous Javascript and XML) really means that we should be using an Asynchronous call :P .. So let us modify the above code so that we change it from synchronous to asynchronous request.

{% highlight html lineos %}
    var request = new XMLHttpRequest();
    request.open('GET', 'info.txt');
    request.onreadystatechange = function() {
      if((request.readyState===4) && (request.status===200))
      {
        console.log(request);
        document.writeln(request.responseText);
      }
    }
    request.send();
{% endhighlight%}

When the readyState = 4 and status = 200 (which means the call was success and response is ready) we can simply write the responseText to the web page. Here it is not mandatory to specify **true **as a 3rd parameter in open() function because by default, browsers will use asynchronous method which need not be specified explicitly. If you want, you can simply add **true** as a 3rd parameter, even then script will work fine.

NOTE:

While using XMLHttpRequest(), request should be made only to the same domain from which the web page gets loaded. The browsers will strictly follow **same origin policy. **If you want do a [Cross Domain Ajax call]({{ site.url }}/2014/09/06/cors-how-cross-domain-call-works-in-browsers/), there should be some modifications for the above code. Read more about Cross Origin Resource Sharing: [CORS: How cross domain call works in browsers?]({{ site.url }}/2014/09/06/cors-how-cross-domain-call-works-in-browsers/)
