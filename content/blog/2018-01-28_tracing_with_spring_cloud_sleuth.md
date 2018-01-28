---
author: "Jørgen Ringen"
title: "Logging: Distributed tracing with Spring Cloud Sleuth"
date: 2018-01-28
slug: tracing_with_spring_cloud_sleuth
---

[Spring Cloud Sleuth](https://cloud.spring.io/spring-cloud-sleuth/) is a framework for enhancing logging and diagnostics, especially in a distributed microservice architecture. Sleuth makes it possible to correlate log-statements relating to a specific requests, scheduled jobs, etc within an application or even across multiple application.
This is really useful when using a log-aggregation tool like [Splunk](https://www.splunk.com/) or the [ELK-stack](https://www.elastic.co/webinars/introduction-elk-stack)

Terminology:

- SLF4J - Standard logging API for java
- Trace - Can be thought of as a single request  in an application or spanning across multiple applications.
- Spans - Can be thought of as sections/parts of a trace. A single trace can be composed of multiple spans correlating to a specific step or section of the request.

## Tracing inside a single application
Add spring-cloud-starter-sleuth as a dependency to your spring boot application:
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
```

Write a log statement by leveraging the SLF4J Api
```java
@RestController
public class FooController {

	private static final Logger LOGGER = LoggerFactory.getLogger(FooController.class);

	@GetMapping
	public String foo() {
		LOGGER.info("FooController#foo");
		return "foo @ " + System.currentTimeMillis();
	}
}
```

Output
```
INFO [foo-service,005f6c05aa1d5be8,005f6c05aa1d5be8,false] --- com.example.foo.FooController : FooController#foo
```

The information in the brackets are added by sleuth according to the following following format: [application-name,trace-id,span-id,export]

- Application name - spring.application.name property
- Trace-id
- Span-id
- Export - If the log statement is exported to an aggregator like Zipkin

If you add multiple log-statements within the request you’ll see that they can be correlated by using the trace id and/or the span-id.

### Adding a span
By creating a span we can “group” parts of a request together. Take a look at the following example:

```java
@Service
public class FooService {

	private static final Logger LOGGER = LoggerFactory.getLogger(FooService.class);

	@Autowired
	private Tracer tracer;

	public void fooLogic() {
		LOGGER.info("Inside fooLogic");

		Span fooLogicSpan = tracer.createSpan("fooLogic-span");
		LOGGER.info("Doing some logging inside a new span");
		LOGGER.info("Doing some more logging inside the span");
		tracer.close(fooLogicSpan);

		LOGGER.info("Exiting fooLogic");
	}
}
```

Output:
```
[foo-service,11befeaae16ddffd,11befeaae16ddffd,false] - Inside fooLogic
[foo-service,11befeaae16ddffd,b7b14df164349fee,false] - Doing some logging inside a new span
[foo-service,11befeaae16ddffd,b7b14df164349fee,false] - Doing some more logging inside the span
[foo-service,11befeaae16ddffd,11befeaae16ddffd,false] - Exiting fooLogic
```

Notice that the log-statements inside the span are assigned a unique span-id.

## Tracing across multiple applications
Sleuth also provides tracing across multiple applications. By creating RestTemplate as a bean, sleuth will automatically add an interceptor to the RestTemplate. The interceptor will add tracing headers to outbound requests. Sleuth will automatically pick up the headers in the receiving application(s) and use the trace and span id’s for logging. This means that we get tracing across multiple applications out of the box. 
Sleuth also works with a lot of other libraries like for example Hystrix, RxJava, Feigh, flux web client in Spring 5, Zuul, ++.

```java
@Bean
public RestTemplate restTemplate() {
	// sleuth wil automatically add interceptors
	return new RestTemplate();
}
```

```java
@Service
public class FooService {

	private static final Logger LOGGER = LoggerFactory.getLogger(FooService.class);

	private final Tracer tracer;
	private final RestTemplate restTemplate;

	@Autowired
	public FooService(Tracer tracer, RestTemplate restTemplate) {
		this.tracer = tracer;
		this.restTemplate = restTemplate;
	}

	public String getBarMessage() {
		LOGGER.info("Inside getBarMessage");
		String barMessage = getMessageFromBarService();
		LOGGER.info("Exiting getBarMessage");
		return barMessage;
	}

	private String getMessageFromBarService() {
		Span getBarSpan = tracer.createSpan("getBarSpan");
		LOGGER.info("Calling bar-service to get message");
		String barMessage = restTemplate.getForObject("http://localhost:9002", String.class);
		LOGGER.info("Received message='{}' from barService", barMessage);
		tracer.close(getBarSpan);
		return barMessage;
	}
}
```

Bar-application:
```java
@RestController
public class BarController {

	private static final Logger LOGGER = LoggerFactory.getLogger(BarController.class);

	@Autowired
	private BarService barService;

	@GetMapping
	public String bar() {
		LOGGER.info("BarController#bar received request");
		return barService.bar();
	}
}
```

```java
@Service
public class BarService {

	private static final Logger LOGGER = LoggerFactory.getLogger(BarController.class);

	public String bar() {
		LOGGER.info("BarService#bar");
		return "bar @ " + System.currentTimeMillis();
	}
}
```

When calling our foo-application, which in turn calls our bar-application, we get the following output:

**Foo log**
```
[foo-service,e40993b7256eda9d,e40993b7256eda9d,false] : Inside getBarMessage
[foo-service,e40993b7256eda9d,71b4a062bf8f970c,false] : Calling bar-service to get message
[foo-service,e40993b7256eda9d,71b4a062bf8f970c,false] : Received message=bar @ 1517154796990 from barService
[foo-service,e40993b7256eda9d,e40993b7256eda9d,false] : Exiting getBarMessage
```

**Bar log**
```
[bar-service,e40993b7256eda9d,77ac3f0ab64dcecf,false] : BarController#bar received request
[bar-service,e40993b7256eda9d,77ac3f0ab64dcecf,false] : BarService#bar
```

Notice that the span-id is propagated as well as the trace-id.

If we inspect the http-headers in the request to BarController we’ll see the following:
```
"x-b3-traceid"="e40993b7256eda9d"
"x-b3-spanid"="0bcc87a9c8ea5fe3"
"x-b3-parentspanid"="71b4a062bf8f970c"
```

This was just a small subset of the functionality in spring sleuth. Example-code: https://github.com/JorgenRingen/spring-sleuth-zipkin-poc

In a following example we’ll look at log-aggregation and visualisation by adding [ZipKin](https://github.com/openzipkin/zipkin) to our example.