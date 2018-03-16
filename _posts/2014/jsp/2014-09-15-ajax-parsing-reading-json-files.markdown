---
author: AnirudhAnand
comments: true
date: 2014-09-15 10:26:58+00:00
layout: post
slug: ajax-parsing-reading-json-files
title: 'Ajax: Parsing and Reading JSON files'
wordpress_id: 87
categories:
- javascript
summary: How to parse and read json files in AJAX
---

Recently we looked into how can we [read an XML file using Ajax]({{ site.url }}/2014/09/14/ajax-parsing-reading-xml-files-using-ajax/). Now let us see how to do the same with JSON files. Actually Ajax can read any text file. The trick is to know how to parse the files in such a way that we can extract out information we need easily. JSON is a widely used data interchange format. It is very easy for the browsers to parse and generate and is based on JavaScript programming language. Now let us look into how can we parse a JSON file using Ajax:

A sample JSON file will be similar to something like this:

  ``{"employees":[
        {"firstName":"John", "lastName":"Doe"}, 
        {"firstName":"Anna", "lastName":"Smith"},
        {"firstName":"Peter", "lastName":"Jones"}
    ]}

This could be a sample JSON file.  Now let us see how can we parse and read the contents from JSON file and print only the content that we wants. In the modern day browsers, parsing JSON is very simple. Modern browsers supports parse command which can be used to **Parse** a JSON file and save it to a variable. Let us see how can we do this:

{% highlight html lineos %}
    var request = new XMLHttpRequest();
    request.open('GET', 'info.json');
    request.onreadystatechange = function() {
      if((request.readyState===4) && (request.status===200))
      {
        var items = JSON.parse(request.responseText);
      }
    }
    request.send();
{% endhighlight %}

Parsing JSON is as simple as above.  All most all modern browsers will support **JSON.parse** method so parsing JSON is much simpler now a days.. Now let us see how we can print the result in a list:

{% highlight html lineos %}
    var request = new XMLHttpRequest();
    request.open('GET', 'info.json');
    request.onreadystatechange = function() {
      if((request.readyState===4) && (request.status===200))
      {
        var items = JSON.parse(request.responseText);
        var out = '<ul>';
        for (var key in items)
        {
          out += '<li>' +items[key].name + '</li>';
        }
      out += '</ul>';
      document.getElementByID('id').innerHTML = out;
      }
    }
    request.send();
{% endhighlight %}

This will add the parsed JSON elements to the webpage. Even though [**Ajax **was originally designed to work with XML files]({{ site.url }}/2014/09/14/ajax-parsing-reading-xml-files-using-ajax/), it can work with JSON and other formats as well and sometimes  the process is simpler than processing XML. As you can see here, parsing and reading JSON was much simpler than XML.
