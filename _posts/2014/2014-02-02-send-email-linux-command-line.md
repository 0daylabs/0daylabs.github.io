---
author: AnirudhAnand
layout: post
title: (How to) Send an Email from Linux command line ?
categories:
- tutorials
- linux
summary: Linux Command line is the most powerful tool that you can get in any Linux machine. We can monitor the on going process, delete/copy/make directories/files etc.. But can we send an email with it ?
---

Linux **Command line** is the most powerful tool that you can get in any Linux machine. We can monitor the on going process, delete/copy/make directories/files etc.. and much more. Today, I successfully configured my Linux to send an email to any person from command line ! Yes that's right. A bit Geekish huh ? Well, after all, it is a feature of Linux Terminal. So Lets see how can we send an Email with Linux terminal :

1) First, we have to configure the mail so that we can send a message with a body and subject. For that, give the following command in Linux terminal : 

`sudo dpkg-reconfigure exim4-config`

This will help us in Configuring the mail.

2) In the General type mail configuration choose the option as shown in the following screenshot

![Send mail with Linux Terminal](/images/2014/Send-mail-with-Linux-Terminal.png)

3) For all the other options , just click ok and go forward until you encounter something like this :** Delivery method for local mail. **There, choose the option `**mbox format in /var/mail**`.


4) The above 2 options are only the basic ones. You can read about rest of the options that is asked so that you can again customize the way you send the particular email.  Now as the mail is configured, lets see how can we send it through the command line.

5) To send a mail, the command is just one line like this :
    
`echo “ The body of the mail.” | mail -s “Subject of the mail”
 you@youremailid.com`

Now if your body of the mail is very large, then you can save the body message in to a file and then you can **send the contents of the file** (and not the file itself as an attachment) with the following command :
    
`mail -s "Subject goes here” you@youremailid.com < 
 /home/user-name/bodymessege.log`


6) You can also send **BCC** (Blind Carbon copy) of the same message to multiple email address. To do that, you can give the -b parameter :
    
`echo “Body message” | mail -s “Subject goes here” 
 you@yourmail.com -b anothermail@yourmail.com`


Hope our readers like this post. If you have any problem in configuring this, do drop down a comment and we are happy to help.
