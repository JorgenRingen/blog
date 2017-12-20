---
author: "Jørgen Ringen"
title: "Logging: Simple way to trace related log statements in an application"
date: 2016-08-01
slug: logging_rename_thread_name
---

Ever seen this kind of output in your logs?

```
[0m18:33:51,083 INFO  [org.example.business.PersonResource] (default task-2) Validation error. Returning status-code 400
```

What's the problem with this log-message? We know that some validation has failed and that a status-code 400 was returned. Other than that it doesn't provide a lot more information other than the time of the event and the name of the class that logged the message.

How can we find any related log-statements to the error? Except from matching the timestamp with other log-statements that happened in the +/- ~500ms range, it's very hard. In order to find related log-statements we need to provide a unique identifier that's part of all the log-statements related to the request. That way we can trace the request through our application flow.

If you look carefully at the log-statement it says `(default task-2)`. This is actually the name of the thread that executed the request. But since application servers use thread pooling, threads are reused across requests, so we can't really use this name either.

To get a unique thread-name for every request we can actually rename the name of the thread. This must happen in the outermost boundary of the application, before any relevant logging happens. In Java EE this can be done in a [javax.servlet.Filter](https://tomcat.apache.org/tomcat-5.5-doc/servletapi/javax/servlet/Filter.html) declared first in the filterchain, before any logging happens.

Heres an example of how this can be implemented:

```java
public class ThreadRenamingFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        Thread.currentThread().setName(UUID.randomUUID().toString());
    }

    ...
```

Now our log output will look like this:

```
[0m18:58:27,811 INFO  [org.example.business.PersonResource] (ac599f24-6368-4dbe-a2ec-ad98e6581df7) Validation error. Returning status-code 400
```

"ac599f24-6368-4dbe-a2ec-ad98e6581df7" is the unique identifier for the thread and it will be printed in all the log-statements that the request generated.

But why stop there?

There are a lot of other useful things we can capture in the thread name:

```java
@Override
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
    String requestURI = ((HttpServletRequest) request).getRequestURI();
    Principal userPrincipal = ((HttpServletRequest) request).getUserPrincipal();
    String caller = userPrincipal == null ? "anonymous" : userPrincipal.getName();

    Thread.currentThread().setName(
                    "UUID: [" + UUID.randomUUID().toString() + "], " +
                    "user: [" + caller + "], " +
                    "URI: [" + requestURI + "]");

    chain.doFilter(request, response);
}
```

```java
[0m19:04:15,138 INFO  [org.example.business.PersonResource] (UUID: [c71b5ae6-7cae-4555-bc97-f2000c3ff90b], user: [anonymous], URI: [/my_application/resources/person]) Validation error. Returning status-code 400
```

This is more informative and helpful while debugging! Now we can see which URI was requested, the user principal that made the request, etc.

This solution is only useful for tracing internal in the application. I'll write a follow-up article shortly on how to do tracing across applications.

**Protip**: If you use [logback](https://logback.qos.ch/) you can add attributes in [MDC](https://logback.qos.ch/manual/mdc.html) which can be declared in logback.xml to get the same kind of functionality.