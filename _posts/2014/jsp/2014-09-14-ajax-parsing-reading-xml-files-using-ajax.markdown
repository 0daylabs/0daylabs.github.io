---
author: AnirudhAnand
comments: true
date: 2014-09-14 17:12:34+00:00
layout: post
slug: ajax-parsing-reading-xml-files-using-ajax
title: 'Ajax: Parsing and Reading XML files'
wordpress_id: 80
categories:
- javascript
summary: How to parse and read xml files in AJAX
---

Ajax was originally created to read data stored in the XML format. But it evolved so much so that you can read even JSON and much more with it. Before understanding how to parse an XML file, you should understand how [XMLHttpRequest()]({{ site.url }}/2014/09/13/ajax-everything-you-should-know-about-xmlhttprequest/) works. If you don't know much about it, read our article: [Everything you should know about XMLHttpRequest()]({{ site.url }}/2014/09/13/ajax-everything-you-should-know-about-xmlhttprequest/). So let us see how can we parse an XML file using Ajax. Just like **responseText, **we have something else called **responseXML. **Let us see a sample code to understand how it works:

Consider we have a very simple XML file like this:

    <!--  Edited by XMLSpy  -->
    <note>
    <to>Tove</to>
    <from>Jani</from>
    <heading>Reminder</heading>
    <body>Don't forget me this weekend!</body>
    </note>


The above code is taken from [w3schools](http://www.w3schools.com/xml/note.xml) just for the sake of an example. So this is a very simple XML file. Consider we need to write some information, say **from **tag, from the XML document to the webpage. How can we do that ? Lets see the code to do the same:

{% highlight html lineos %}
    var request = new XMLHttpRequest();
    request.open('GET', 'info.xml');
    request.onreadystatechange = function() {
      if((request.readyState===4) && (request.status===200))
      {
        var from = request.responseXML.getElementsByTagName('from')[0].firstChild.nodeValue;
        document.write(from);
        console.log(request);
      }
    }
    request.send();
{% endhighlight%}

Now let us see how the above code works (make sure you read our post about XMLHttpRequest() if you don't understand other parts of the code). As we can see, when the readyState = 4 and request status =  200, it enters the if condition and assign a new value to the JavaScript variable **from**. In this case, the value of from will be "**Jani**". So how did that happen? The **request.responseXML** will return the entire XML document and **getElementsByTagName **is a JavaScript DOM method to get all elements which belongs to a particular tag name. Here the tagname is 'from'. The getElementsByTagName will return a list of all elements with the specified tag name.


In order to choose the first element, we append **[0]** to it. This will return the entire from tag to the variable from and now if we print the variable, the value will be **<from>Jani</from>**. But we don't want the from tags to get included in the variable and need only the name. The firstChild is to select the first value from a given list. To get only the text of the particular node, we include **nodeValue. **Now this will write only the name "Jani" to the JavaScript variable **from **and won't include quotes or tags.

What if there are more than one names and you need to print a list of names to the page ? The common idea is to write the contents inside the for loop. So the code can be modified as:

{% highlight html lineos %}
    var request = new XMLHttpRequest();
    request.open('GET', 'info.xml');
    request.onreadystatechange = function() {
      if((request.readyState===4) && (request.status===200))
      {
        var items = request.responseXML.getElementsByTagName('tag-id');
        var out = '<ul>';
        for (var i = 0; i < items.length; i++)
        {
          out+= '<li>' + items[i].firstChild.nodeValue + '</li>';
        }
      out += '</ul>';
      document.getElementById("id").innerHTML = out;
     }
    }
    request.send();
{% endhighlight%}

The above code will iterate through the entire list returned by `getElementByTagName()` and create a new unordered list. We add each element as a list item and append it to a variable named '**out**'. Then we send the variable back to HTML page and replace a particular div id's `innerHTML` with the new list we created.