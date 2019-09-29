---
author: AnirudhAnand
layout: post
title: Dive deep into Android Application Security - OWASP MSTG Uncrackable level 1 writeup
categories:
- android
summary: Uncrackable Apps for Android is a collection of mobile reversing challenges maintained by the OWASP MSTG (Mobile Security Testing Guide) authors. Cracking and solving these challenges is a fun way to learn Android security. Here, in this writeup, we will dive deep into Android and use several different ways to solve the same problem !
description: Uncrackable Apps for Android is a collection of mobile reversing challenges maintained by the OWASP MSTG (Mobile Security Testing Guide) authors. Cracking and solving these challenges is a fun way to learn Android security. Here, in this writeup, we will dive deep into Android and use several different ways to solve the same problem !
---

## Introduction

Android is the most popular mobile operating system with over 85% in market share and as a result it’s important to look into the security aspects of the same as well. [Mobile Security Testing Guide](https://github.com/OWASP/owasp-mstg/){:target="_blank"} (MSTG) is one of the Flagship OWASP project which is a comprehensive manual for mobile app security development, testing and reverse engineering. The authors of MSTG have created some crackme's for both Android and iOS platform using which we can dive deep into the application security of the respective platforms. In this article, we will focus on solving [uncrackable level 1](https://github.com/OWASP/owasp-mstg/raw/master/Crackmes/Android/Level_01/UnCrackable-Level1.apk){:target="_blank"} for Android.


Let's install the APK in an emulator and see how it works. I am using genymotion personal license with Google Nexus 5 (6.0 - API 23) device.

{% highlight bash lineos %}
[anirudhanand@Android]$ adb devices
List of devices attached
192.168.56.102:5555	device

[anirudhanand@Android]$ adb install UnCrackable-Level1.apk
Performing Push Install
UnCrackable-Level1.apk: 1 file pushed. 22.7 MB/s (66651 bytes in 0.003s)
	pkg: /data/local/tmp/UnCrackable-Level1.apk
Success
{%endhighlight%}

If we run the application, it detects the root access to the device and exit immediately. Root detection is one of the common techniques developers use to prevent installing their applications on the rooted device. Let's decompile the APK and look into the source code to see if we can bypass this.


## Decompiling into Java Source code

One of the tools used to decompile the APK into java source code is [Jadx](https://github.com/skylot/jadx){:target="_blank"}.

{% highlight bash lineos %}

[anirudhanand@Android]$ ls
UnCrackable-Level1.apk

[anirudhanand@Android]$ jadx UnCrackable-Level1.apk
INFO  - output directory: UnCrackable-Level1
INFO  - loading ...
INFO  - Can't find 'R' class in app package: owasp.mstg.uncrackable1
INFO  - App 'R' class not found, put all resources ids to : 'owasp.mstg.uncrackable1.R'
INFO  - processing ...
WARNING: An illegal reflective access operation has occurred
WARNING: Illegal reflective access by com.rits.cloning.Cloner (file:/usr/local/Cellar/jadx/1.0.0/libexec/lib/cloning-1.9.12.jar) to field java.util.TreeSet.m
WARNING: Please consider reporting this to the maintainers of com.rits.cloning.Cloner
WARNING: Use --illegal-access=warn to enable warnings of further illegal reflective access operations
WARNING: All illegal access operations will be denied in a future release
INFO  - done

{%endhighlight%}

We can see the source code on `./UnCrackable-Level1/sources/sg/vantagepoint/uncrackable1` where `MainActivity.java` has been decompiled successfully by the Jadx. 

{% highlight java lineos %}

10: import android.widget.EditText;
11: import owasp.mstg.uncrackable1.R;
12: import sg.vantagepoint.a.b;
13: import sg.vantagepoint.a.c;
14: 
15: public class MainActivity extends Activity {
16:     private void a(String str) {
17:         AlertDialog create = new Builder(this).create();
18:         create.setTitle(str);
19:         create.setMessage("This is unacceptable. The app is now going to exit.");
20:         create.setButton(-3, "OK", new OnClickListener() {
21:             public void onClick(DialogInterface dialogInterface, int i) {
22:                 System.exit(0);
23:             }
24:         });
25:         create.setCancelable(false);
26:         create.show();
27:     }
28: 
29:     /* access modifiers changed from: protected */
30:     public void onCreate(Bundle bundle) {
31:         if (c.a() || c.b() || c.c()) {
32:             a("Root detected!");
33:         }
34:         if (b.a(getApplicationContext())) {
35:             a("App is debuggable!");
36:         }
37:         super.onCreate(bundle);
38:         setContentView(R.layout.activity_main);
39:     }

{%endhighlight%}

On line number 12, 13 there are 2 imports which is nothing but importing from the location `./UnCrackable-Level1/sources/sg/vantagepoint/a/` which has 3 files: a.java, b.java, c.java

Both b.java and c.java files are being imported to the `MainActivity.java` file. The program has 2 protections inbuilt:

### Root Detection:

This checks if the app has been installed on a rooted device and if so, exit immediately. The function is called within `onCreate` but it's defined in c.java file:

 {% highlight java lineos %}
01: package sg.vantagepoint.a;
02: 
03: import android.os.Build;
04: import java.io.File;
05: 
06: public class c {
07:     public static boolean a() {
08:         for (String file : System.getenv("PATH").split(":")) {
09:             if (new File(file, "su").exists()) {
10:                 return true;
11:             }
12:         }
13:         return false;
14:     }
15: 
16:     public static boolean b() {
17:         String str = Build.TAGS;
18:         return str != null && str.contains("test-keys");
19:     }
20: 
21:     public static boolean c() {
22:         for (String file : new String[]{"/system/app/Superuser.apk", "/system/xbin/daemonsu", "/system/etc/init.d/99SuperSUDaemon", "/system/bin/.ext/.su", "/system/etc/.has_su_daemon", "/system/etc/.installed_su_daemon", "/dev/com.koushikdutta.superuser.daemon/"}) {
23:             if (new File(file).exists()) {
24:                 return true;
25:             }
26:         }
27:         return false;
28:     }
29: }

{%endhighlight%}

So for Root detection, they are doing 3 tests to confirm if the device is rooted or not:

1. **Su Binary**: The function checks if SU binary exists or not. If so, the device as rooted.
2. **test-keys**: During the release of kernel, keys are being used to sign it which is either `release-keys` or `test-keys`. If it's the latter, it means the kernel was signed with a custom key generated by a 3rd party developer. This is an indication that the device might be rooted (This info is located in the file `/system/build.prop`).
3. **/system/** : These are common files/binaries which is accessible in a rooted device which is otherwise not accessible (on a non rooted device).


### Debuggable App:

By modifying the `AndroidManifest.xml`, app can be debuggable so at the run time we can connect into the app using JDB (java debugger) and modify its behaviour. In order to prevent this, a check is implemented to see if the app is debuggable and if so, the system will exit.


Now let's look at the `verify` function where the secret is being verified.


 {% highlight java lineos %}
41:     public void verify(View view) {
42:         String str;
43:         String obj = ((EditText) findViewById(R.id.edit_text)).getText().toString();
44:         AlertDialog create = new Builder(this).create();
45:         if (a.a(obj)) {
46:             create.setTitle("Success!");
47:             str = "This is the correct secret.";
48:         } else {
49:             create.setTitle("Nope...");
50:             str = "That's not it. Try again.";
51:         }
52:         create.setMessage(str);
53:         create.setButton(-3, "OK", new OnClickListener() {
54:             public void onClick(DialogInterface dialogInterface, int i) {
55:                 dialogInterface.dismiss();
56:             }
57:         });
58:         create.show();
59:     }

 {%endhighlight%}


The function takes an input from the user and pass it to `a.a()`:

{% highlight java lineos %}


File: ./uncrackable/UnCrackable-Level1/sources/sg/vantagepoint/uncrackable1/a.java
01: package sg.vantagepoint.uncrackable1;
02: 
03: import android.util.Base64;
04: import android.util.Log;
05: 
06: public class a {
07:     public static boolean a(String str) {
08:         byte[] bArr;
09:         String str2 = "8d127684cbc37c17616d806cf50473cc";
10:         byte[] bArr2 = new byte[0];
11:         try {
12:             bArr = sg.vantagepoint.a.a.a(b(str2), Base64.decode("5UJiFctbmgbDoLXmpL12mkno8HT4Lv8dlat8FxR2GOc=", 0));
13:         } catch (Exception e) {
14:             StringBuilder sb = new StringBuilder();
15:             sb.append("AES error:");
16:             sb.append(e.getMessage());
17:             Log.d("CodeCheck", sb.toString());
18:             bArr = bArr2;
19:         }
20:         return str.equals(new String(bArr));
21:     }
22: 
23:     public static byte[] b(String str) {
24:         int length = str.length();
25:         byte[] bArr = new byte[(length / 2)];
26:         for (int i = 0; i < length; i += 2) {
27:             bArr[i / 2] = (byte) ((Character.digit(str.charAt(i), 16) << 4) + Character.digit(str.charAt(i + 1), 16));
28:         }
29:         return bArr;
30:     }
31: }


    // Verify() internally calls sg.vantagepoint.a.a.a()


File: ./uncrackable/UnCrackable-Level1/sources/sg/vantagepoint/a/a.java
01: package sg.vantagepoint.a;
02: 
03: import javax.crypto.Cipher;
04: import javax.crypto.spec.SecretKeySpec;
05: 
06: public class a {
07:     public static byte[] a(byte[] bArr, byte[] bArr2) {
08:         SecretKeySpec secretKeySpec = new SecretKeySpec(bArr, "AES/ECB/PKCS7Padding");
09:         Cipher instance = Cipher.getInstance("AES");
10:         instance.init(2, secretKeySpec);
11:         return instance.doFinal(bArr2);
12:     }
13: }

{%endhighlight%}

Class `a` which has a parameter named `str` (which is nothing but user input) and has a predefined variable named str2, which looks like the encrypted string. The encrypted string is decrypted on the run time and is compared with the string we entered. If both values match, we have successfully completed the challenge.


There are multiple ways in which we can solve the challenge:

* Repackaging
* Frida Instrumentation
* JDB (Runtime debugging)

Let's look into some of the above techniques:

## Bypassing root detection with Repackaging

One of the ways to solve this challenge is to decompile the app with apktool, modify the smali byte code and reinstall the app to control the function flow. Using this technique, Root detections can be bypassed in several ways:

1. Modify the return of each of the functions inside class `c` to always return false whether they have detected root or not.

2. Modify the function `onClick` inside the MainActivity.java to return void instead of calling `system.exit` so that even through root is detected, the app won't exit.


Let's use the latter (2) to evade the root detection:

{% highlight bash lineos %}
[anirudhanand@Android]$ apktool d UnCrackable-Level1.apk -o uncrackable_dissas
I: Using Apktool 2.4.0 on UnCrackable-Level1.apk
I: Loading resource table...
I: Decoding AndroidManifest.xml with resources...
I: Loading resource table from file: /var/folders/9h/2_57gxb50pqg6nxhtmt9csk00000gn/T/1.apk
I: Regular manifest package...
I: Decoding file-resources...
I: Decoding values */* XMLs...
I: Baksmaling classes.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...

[anirudhanand@Android]$ pwd
./uncrackable_dissas/smali/sg/vantagepoint/uncrackable1


[anirudhanand@Android]$ ls
MainActivity$1.smali MainActivity$2.smali MainActivity.smali   a.smali

{%endhighlight%}

If we look at the functions inside `MainActivity$1.smali`, a function named `OnClick` is calling `system.exit(0)`. Let's modify the function to simply return void and ignore the exit call.


{% highlight bash lineos %}

File: MainActivity$1.smali

35: # virtual methods
36: .method public onClick(Landroid/content/DialogInterface;I)V
37:     .locals 0
38: 
39:     const/4 p1, 0x0
40: 
41:     invoke-static {p1}, Ljava/lang/System;->exit(I)V       # remove this line
42: 
43:     return-void
44: .end method


Modified File: MainActivity$1.smali

35: # virtual methods
36: .method public onClick(Landroid/content/DialogInterface;I)V
37:     .locals 0
38: 
39:     const/4 p1, 0x0
40: 
42: 
43:     return-void
44: .end method


[anirudhanand@Android]$ apktool b uncrackable_dissas -o modified_uncrackable.apk
I: Using Apktool 2.4.0
I: Checking whether sources has changed...
I: Smaling smali folder into classes.dex...
I: Checking whether resources has changed...
I: Building resources...
I: Building apk file...
I: Copying unknown files/dir...
I: Built apk...


[anirudhanand@Android]$ java -jar sign.jar modified_uncrackable.apk

[anirudhanand@Android]$ adb uninstall owasp.mstg.uncrackable1

[anirudhanand@Android]$ adb install modified_uncrackable.s.apk

{%endhighlight%}

Let's run the newly installed APK and we can see that clicking on "ok" won't exit the application. Essentially what we did here was to modify the function call "onClick" and we removed the `system.exit()` function invocation line: `invoke-static {p1}, Ljava/lang/System;->exit(I)V`. Then we recomplied, signed (using [sign.jar](https://github.com/appium/sign){:target="_blank"}) and reinstalled the application to bypass this check.


## Leaking the secret with runtime instrumentation - Frida

Frida is a dynamic runtime instrumentation toolkit using which we can hook functions, spy on crypto APIs or trace private application code on runtime. In short, using frida, we can redefine functions, leak function variables and what not ? In order to run Frida, make sure to install frida server on your rooted device and it's running in the background as root (Frida - Install [Client](https://www.frida.re/docs/installation/){:target="_blank"} and [Server](https://www.frida.re/docs/android/){:target="_blank"}). Once the server is running, we can write our own script to leak the secret out of the APK during run time.

A sample frida script (modifying function implementation) will look like this:


{% highlight java lineos %}
Java.perform(function () {
	// Name of the class to start hooking.
	var MainActivity = Java.use("com.example.name.classname");
	MainActivity.function_name.implementation = function() {
	// write the modified function code here
	return false;
}
	console.log("function simply returned.")
});
{%endhighlight%}

In very simple terms, we can write JavaScript code to tell frida to use a class and hook its corresponding functions and reimplement it. In the above example, we hooked a function named `function_name` from a class and rewrote its implementation to make sure function always returns false during runtime.


From the initial source code analysis, we know the decryption of the encrypted string is happening inside `sg/vantagepoint/a/a.java` where the return of the function `a()` has our secret. So using Frida, we can do the following to solve the challenge:

1. Hook the function `a()` inside the class `sg.vantagepoint.a.a`.
2. Modify the function implementation and call the function internally to leak its return value (byte array).
3. Convert the returned byte array to ascii and append it to a string to retrieve our secret.

So our final exploit code looks like the following:

{% highlight java lineos %}
Java.perform(function () {
	var aes = Java.use("sg.vantagepoint.a.a");

	// Hook the function inside the class.
	aes.a.implementation = function(var0, var1) {

		// Calling the function itself to get its return value
		var decrypt = this.a(var0, var1);
		var flag = "";

		// Converting the returned byte array to ascii and appending to a string
		for(var i = 0; i < decrypt.length; i++) {
				flag += String.fromCharCode(decrypt[i]);
           }

        // Leaking our secret
        console.log(flag);
        return decrypt;
	}
});
{%endhighlight%}

Run the above exploit (exploit.js) using Frida: `frida -U -f owasp.mstg.uncrackable1 -l exploit.js --no-pause`. Once you run it, the program will be launched inside the emulator. Give some random input so that the function gets invoked at least once and look back in the Frida terminal to see the leaked secret.

## References

1. Frida: [https://frida.re](https://frida.re){:target="_blank"}

2. Eduardo Novella's solution (must read) to uncrackable level 1 (completely using Frida): [https://enovella.github.io/android/reverse/2017/05/18/android-owasp-crackmes-level-1.html](https://enovella.github.io/android/reverse/2017/05/18/android-owasp-crackmes-level-1.html){:target="_blank"}
