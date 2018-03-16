---
author: AnirudhAnand
layout: post
title: Remote Code Execution via Python __import__() - MMACTF 2016 Tsurai Web 300 writeup
categories:
- ctf
summary: Manipulating Python's __import__() statement to import attacker controlled modules (MMACTF 2016 web 100 writeup)
description: Manipulating Python's __import__() statement to import attacker controlled modules (MMACTF 2016 web 100 writeup)
---

## Introduction

After a successful completion of [MongoDB NoSQL injection]({%post_url 2016/CTF/MMACTF/2016-09-05-mongo-db-password-extraction-mmactf-100 %})(Web 100), I moved on to a more challenging question, which is tsuari web, a 300 point problem.

## Challenge

A quick look at the challenge tells us that there are options to `register`, `login` and also `upload files` of any type to the server via image upload (Never more interesting). We are also given the source code of the website which seems to be written in Flask. A quick peek at the code reveals some interesting information:

Challenge URL: [http://tweb.chal.ctf.westerns.tokyo](http://tweb.chal.ctf.westerns.tokyo)

File: app.py

{%highlight python lineos %}

from flask import Flask
from flask import session, request, redirect, render_template, url_for, send_from_directory
import os, sys


AUTHFILE = 'passwd'
SALT = os.environ['TW_SALT']

app = Flask('Tsurai Web', static_folder='templates/assets')
app.config['SECRET_KEY'] = os.environ['TW_SECRET']

sys.path.append('data')

@app.route('/')
def index():
    if not session.get('username'):
        return render_template('index.html')

    config = __import__(h(session.get('username')))
    password = request.args.get('password')
    msg = request.args.get('msg')

    if password:
        return render_template('albums.html',
                msg="Your password is {}".format(password),
                imgs=config.imgs)
    elif msg:
        return render_template('albums.html',
                msg=msg,
                imgs=config.imgs)
    else:
        return render_template('albums.html',
                msg="Hello, {} !".format(session.get('username')),
                imgs=config.imgs)


@app.route('/login', methods=['POST'])
def login():
    username = request.form['username']
    password = request.form['password']

    if auth(username, password):
        session['username'] = username
        return redirect('/')
    else:
        return render_template('index.html', error="Login failed.")


@app.route('/logout')
def logout():
    session.pop('username', None)
    return redirect('/')


@app.route('/register', methods=['POST'])
def register():
    username = request.form['username']

    if not is_valid(username):
        return render_template('index.html', error="Invalid username.")

    if user_exists(username):
        return render_template('index.html', error="User already exists.")

    password = genpw()
    hashed = h(password+SALT)
    open(AUTHFILE, 'a').write('\n{username}:{hashed}'.format(**locals()))
    os.mkdir('data/{}'.format(h(username)))
    open('data/{}.py'.format(h(username)), 'w').write("imgs = {}".format(repr([])))

    session['username'] = username

    return redirect('/?password='+password)


@app.route('/upload', methods=['POST'])
def upload():
    if not session.get('username'):
        return redirect('/')

    if 'file' not in request.files:
        redirect('/?msg='+q("No file specified."))

    config = __import__(h(session.get('username')))
    imgs = config.imgs

    req_file = request.files['file']

    if not req_file or req_file.filename == "":
        return redirect('/?msg='+q("No file specified."))

    fname = req_file.filename

    if fname in imgs:
        return redirect('/?msg='+q('File already exists.'))

    imgs.append(fname)
    req_file.save(os.path.join("data/{}".format(h(session.get('username'))), os.path.basename(fname)))
    open("data/{}.py".format(h(session.get('username'))), 'w').write("imgs = {}".format(repr(imgs)))

    return redirect('/?msg='+q("Upload succeccful."))


@app.route('/show')
def show():
    if not session.get('username'):
        return redirect('/')

    filename = request.args.get('filename')
    return send_from_directory("data/{}".format(h(session.get('username'))), filename)


def is_valid(username):
    import re
    if not re.match(r"\A[0-9a-zA-Z]{,20}\Z", username):
        return False
    else:
        return True


def user_exists(username):
    auths = open(AUTHFILE).read().strip().split('\n')
    for l in auths:
        if ':' not in l:
            continue
        u, _ = l.split(':', 1)
        if u == username:
            return True
    return False


def auth(username, password):
    auths = open(AUTHFILE).read().strip().split('\n')
    for l in auths:
        if ':' not in l:
            continue
        u, p = l.split(':', 1)
        hashed = h(password+SALT)
        if (u, p) == (username, hashed):
            return True

    return False


def h(s):
    from hashlib import md5
    return md5(s).hexdigest()


def q(s):
    from urllib import quote
    return quote(s)


def genpw():
    import random, string
    return ''.join([random.choice(string.printable[:62]) for _ in range(0x10)])


if __name__ == '__main__':
    app.run()

{%endhighlight%}


There are several interesting things to note here:

1) You can see an `__import__()` call whose input is basically the MD5 hash of our username. If there were no hashing, this is a direct code execution if we can control the username but thats not the case here.

2) There is a file named `userhash.py`, where userhash is the `MD5(username)` which is where user informations is saved (or I guess that's where it is stored).

3) There is a directory named `userhash/` where userhash is again the `MD5(username)` where the uploaded files of each user is saved.

4) We can upload any kinds of files to the server where filename can be controlled by us or server saves the files inside the `userhash/` directory with the exam name with which we upload it (There are some client side protections but can easily be bypassed using burp). Interesting !


## Solution

Now it is obvious on how to exploit the scenario but that was not the case when I was solving it. If you look at the way how `__import__()` works, you can see that it tries to import the `__init__.py` from the function which we imported. So how can we make use of this here ?

1) Upload a file named `__init__.py` to the `userhash/` directory. So next time when the `config = __import__(h(session.get('username')))` is loaded, instead of `userhash.py`, the import will actually execute the init file we uploaded !

2) `return render_template('albums.html', msg="Hello, {} !".format(session.get('username')), imgs=config.imgs)` tells us that the names of the images are taken from an array named `imgs[]` and is shown to the user when he logs in.

3) So by uploading an `__init__.py` we essentially control the config and inturn the `config.imgs`, so what if we can execute commands and save the output as image names ? Even through image doesn't exist, the application still shows us the output this way.

After messing around couple of times, here is the final `__init__.py` I uploaded to get the flag:

{%highlight php lineos %}
x = __import__("subprocess")
imgs = []
imgs.append(x.check_output('cat flag', shell=True))
{%endhighlight%}

## References

These are some of the awesome references which came in handy during solving this problem:

1. [The import system - python 3.5.2 Documentation](https://docs.python.org/3/library/functions.html#__import__)
