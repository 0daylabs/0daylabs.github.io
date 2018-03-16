---
author: AnirudhAnand
layout: post
title: eLearnSecurity Practical Web Defense (eWDP) course review
categories:
- ctf
summary: Practical Web Defense is a unique course which focus on both attacking and *defending* web Applications unlike the traditional courses which focuses only on attacking applications.
description: Practical Web Defense is a unique course which focus on both attacking and *defending* web Applications unlike the traditional courses which focuses only on attacking applications.
---

## Introduction

As a CTF-lover, I always like attacking web applications more than patching the vulnerabilities within it. But patching vulnerabilities correctly without affecting the functionality of the application is indeed an important skill, which is why I decided to take a look at the [eLearnSecurity's Practical Web Defense](https://www.elearnsecurity.com/course/practical_web_defense/) course.

The course is designed by Abraham Aranguren, who is the project leader of [OWASP OWTF](https://www.owasp.org/index.php/OWASP_OWTF). Back in 2014 I was lucky enough to get selected for Google Summer of Code (**GSoC**) as a part of OWTF, which is when Abe (Abraham) gave me free access to the **elite version** of PWD course (Thanks Abe !). Initially I didn't get much time to concentrate on the course but last 2 months (Yes, after 2 years), I got enough free time so I went back and looked into the course materials (which was absolutely awesome !).


## Pre-requisites

The course starts from absolute basics so I think there is no need of any pre-requisites. If you know a bit of `PHP` (or created a basic website with PHP before), that is all is required to learn this course (provided you work hard during the course).


## Course

The course starts with basic tool introductions	(like ZAP, OWTF) and goes through a vareity of modules which essentially contains both exploiting and patching almost all basic web application vulnerabilities. There are 16 modules in total which convers almost everything in OWASP top 10. Each module has:

1. **Slides**: which explains all the theory needed to understand an attack and to patch it as well.

2. **Videos**: which are very eloborate and clearly explains every single lab exercises (some are upto 3 hours length and over 25 hours of video series !).

3. **PDFs** (Available only for elite version): You can download theory in PDF format, the study materials are now portable !

4. **Labs**: All modules have assosiative labs in which a completely vulnerable web application is given and our aim is to find vulnerabilities, exploit it and patch all the vulnerabilities without affecting the functionality of the application.

To be frank, labs are the awesome part of the course. In most of the other courses, we will simply learn to attack and exploit an applications but this is not complete learning. What about patching or defending the application from these vulnerabilities ?
This is the part were eWDP has its uniqueness ! It not only teach you to find and exploit the vulnerabilities but also explains how to patch it according to different contexts (which is very important to learn and fun too ;) )!


## Hera Labs:

Each module has its own `Hera Lab` which is where most of the learning happens. The video labs focus on finding out and patching most of the critical vulnerabilities but there are always **extra miles** which we are supposed to solve by ourselves (for the complete learning.) Note that solving extra miles is actually not necessary but it's highly recommended because submitting the extra miles will give you additional points which can help you pass the final exam more easily.

Each lab comes with a lab module PDF which has complete intended solutions for all main exercieses (ofcourse extra miles is not included, we should try it ourselves) so that once solved, we can cross check our solution with the intended one (more learning !).


## Exam

Exam has 2 parts:

1. **Multiple Choice Questions (MCQ)**:

    The first part has around 50 MCQs which are a bit tricky. You need to score a minimum of (75%) to go to the next level of exam. Make sure you start the exam only after you have gone through the course properly (including all videos and slides), otherwise you will waste your exam voucher (failing to get 75%, you have to buy a new exam voucher again to write the exam).


2. **Practical exam**:

    This is much more interesting than the MCQs. Here you are given an intentionally vulnerable application which has a lot of vulnerabilities (yes a lot !). We are given 3 seperate directories namely **`vulnerable`**, **`fixed`** and **`virtually_patched`** each having the same application files initially. Our objectives are to:

    1. Find and exploit all the vulnerabilities and demonstrate the POC using the `vulnerable` directory.

    2. Patch all the vulnerabilities we exploited in the `fixed` (You have the permission to modify the code only on this directory) and make sure that vulnerabilities are patched in such a way that functionalitiy of the application remains untouched !

    3. Write `ModSecurity` rules to patch all the vulnerabilities we found in the `virtually_patched` directory. Here you are not suppose to edit the vulnerable code but should patch all the vulnerabilities by writing ModSecurity rules to block them.


We have 14 days to complete the final exam and upload the files as a tar ball and then wait for the result. I had to wait for more than a month to get the result, but yeah the course was totally worth it.


## Conclusion

As you have seen from the review, **`eWDP`** is a course with a difference. You are not only learning how to exploit the application but also learning to patch the vulnerabilities and block the attacks using ModSecurity (Virtual Patches) which can be considered as a more complete learning !

For people who want to take this course, if you are interested in learning the defensive side, then this course will really help you lay down your basics. Even as a pentester you need to know how to secure your web applications so that you can recommend secure coding practices to your clients to fix the vulnerabilities you found. So without a doubt, I would strongly recommend this course for people who really want to learn both sides of Web Apllication Pentesting :)


## References

1. [eLearnSecurity Web Defense Professional](https://www.elearnsecurity.com/certification/ewdp/)
2. [Practical Web Defense course syllabus](https://www.elearnsecurity.com/collateral/Syllabus_PWD.pdf)
