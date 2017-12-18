---
author: "Jørgen Ringen"
title: "JavaEE: Using JSR310 time-types in JAX-RS"
date: 2015-05-21
slug: javaee_jsr310_types_with_jaxrs
---

With Java 8 came JSR310, a new and improved date and time API for Java.
This post will show you how you can use types from JSR310 directly in JAX-RS s by only leveraging the Java EE api.

JavaEE dependency:
```xml
<dependency>
    <groupId>javax</groupId>
    <artifactId>javaee-api</artifactId>
    <version>7.0</version>
    <scope>provided</scope>
</dependency>
```

Create a simple Person-class with a field, birthDate, of type java.time.LocalDate from JSR310:

```java
public class Person {

    ...
    private LocalDate birthDate;
    ...
}
```

Lets say we wan't to use the Person entity directly as an input parameter for a JAX-RS endpoint like this:

```java
@POST
@Consumes(MediaType.APPLICATION_JSON)
public Response createPerson(Person person, @Context UriInfo info) {
    Person createdPerson = personService.createPerson(person);
    URI uri = info.getAbsolutePathBuilder().path("/" + person.getId()).build();
    return Response.created(uri).build();
}
```

This won't work out of the box as JAX-RS has no way of knowing how to instantiate a java.time.LocalDate instance from the JSON input string (for example "2015/21/05").
Moreover, JAX-RS have no way of knowing what date-format the caller is using and LocalDate doesn't even have a constructor taking a String argument, which is a requirement for JAX-RS json type-conversion.

On Wildfly we get an exception like this:

```
[33m11:47:03,278 WARN  [org.jboss.resteasy.core.ExceptionHandler] (default task-2) Failed executing POST /person: org.jboss.resteasy.spi.ReaderException: com.fasterxml.jackson.databind.JsonMappingException: Can not instantiate value of type [simple type, class java.time.LocalDate] from String value ('2015-21-05'); no single-String constructor/factory method
Unfortunately there is no elegant uniform way to overcome this by only using the Java EE API. The various JAX-RS implementations (Jersey, Resteasy, CXF, etc) all provide "tools" for achieving this. Jackson, for example, have a JSR 310 datatype library which solves the problem but this has a drawback.
```

---

To resolve the problem by only leveraging the Java EE api we can use the XmlJavaTypeAdapter from JAXB:

```java
public class LocalDateAdapter extends XmlAdapter<String, LocalDate> {

    @Override
    public LocalDate unmarshal(String dateInput) throws Exception {
        return LocalDate.parse(dateInput, DateTimeFormatter.ISO_DATE);
    }

    @Override
    public String marshal(LocalDate localDate) throws Exception {
        return DateTimeFormatter.ISO_DATE.format(localDate);
    }
}
```

Now annotate the field with the adapter:

```
@XmlJavaTypeAdapter(LocalDateAdapter.class)
private LocalDate birthDate;
```

Instead of annotating the field we can declare the adapter in a package-info.java file together with our DTO's. This avoids cluttering of the java-code.

```
@XmlJavaTypeAdapters({
    @XmlJavaTypeAdapter(type = LocalDate.class,
                        value = LocalDateAdapter.class)
})
package org.example;

import java.time.LocalDate;
import javax.xml.bind.annotation.adapters.XmlJavaTypeAdapter;
import javax.xml.bind.annotation.adapters.XmlJavaTypeAdapters;
```

This solution should work across all Java EE servers regardless of JAX-RS implementation (jersey, resteasy, etc).