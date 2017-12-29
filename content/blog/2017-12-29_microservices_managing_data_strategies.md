---
author: "Jørgen Ringen"
title: "Strategies for managing data in microservices"
date: 2017-12-29
slug: stategies_managing_data_in_microservices
---

# Strategies for managing data in microservices
In this post, we’ll look at some common patterns for managing data in a distributed microservice architecture. Managing data in a monolithic application is fairly easy and well understood, but in a microservice architecture it can be a lot more challenging and different patterns are needed. By just re-using patterns from the monolithic world we often end up with poor results and this anti-pattern is often known as the [“distributed monolith”](https://www.microservices.com/talks/dont-build-a-distributed-monolith/).

---

### Private data-stores and synchronous calls between service interfaces
Each microservice manages their own data in private data-stores. Every piece of data is owned by a single service. When a service needs to exchange data with other services it calls their public api by using REST, gRPC, etc. Designing stable, reusable and encapsulated API’s are crucial.

![Synchronous communication](/img/synchronous_communication.png)

**Pros**

- Easy to implement
- Very well understood pattern and programming paradigm
- Fairly easy to follow the program flow between the services assuming good [logging and tracing facilities](http://microservices.io/patterns/observability/distributed-tracing.html) are in place
- Easy to reason about [eventual consistency](https://en.wikipedia.org/wiki/Eventual_consistency) in a synchronous architecture

**Cons**

- Availability can be significantly decreased if one or more services stops responding. This can have a ripple-effect throughout the architecture if there are many runtime-dependencies. This can be mitigated to some extent by using [circuit breaker](https://en.wikipedia.org/wiki/Circuit_breaker) and similar patterns.
- Transactions spanning across multiple services are hard to implement. Distributed transactions should be avoided because of [the CAP theorem](https://en.wikipedia.org/wiki/CAP_theorem). Can be mitigated by using the [saga pattern](http://microservices.io/patterns/data/saga.html), workflows or [compensating transactions](https://en.wikipedia.org/wiki/Compensating_transaction) for example.
- Implementing queries that “joins” across multiple services are challenging. Can result in poor performance as you typically need to ask one service before you can call another.
- Poor scalability and performance because of synchronous calls. Can to some extent be mitigated by using caching.

---

### Private data-stores and asynchronous events
Each microservice manages their own data in private data-stores. Every piece of data is _owned_ by a single service, but data can be replicated across services as reference-data.  Each microservice stores the necessary reference-data it needs in an optimized data structure. Services can publish business events when state changes by the use of [message-oriented middleware](https://en.wikipedia.org/wiki/Message-oriented_middleware) (ActiveMQ, RabbitMQ, Kafka, etc) so that the other services can update their reference-data accordingly in an asynchronously manner. The events will typically include an event-type plus data describing what has happened. This pattern is often referred to as [event-driven architecture](https://en.wikipedia.org/wiki/Event-driven_architecture).

![Eventdriven architecture](/img/eda_communication.png)

**Pros**

- Great scalability and performance
- Availability - each microservice operates on its own data and are not runtime-dependent on others.
- “Quality of service” guarantees are provided by message-oriented middleware (guaranteed delivery, FIFO, etc)
- Easy to add new microservices that can start subscribing to events

**Cons**

- Can be more complex to implement
- Asynchronous communication can be more challenging than synchronous communication
- Need for a middleware component for messaging
- Harder to reason about eventual consistency as updates happens asynchronously
- Single point of failure in the message-oriented middleware

---

### Event-sourcing and Streaming
This pattern is somewhat similar to “private data-store, async events”, but it reduces the need for storing data locally in the services. Services publishes business events to an event-store. Services can subscribe to, and process, events or they can do queries directly towards the [event store](https://martinfowler.com/eaaDev/EventSourcing.html). This pattern is often used in conjunction with [CQRS](http://microservices.io/patterns/data/cqrs.html). There are some very interesting technologies emerging in this field, like [KSQL](https://www.confluent.io/blog/ksql-open-source-streaming-sql-for-apache-kafka/) which is a query language for [Kafka](https://kafka.apache.org).

![Event-sourcing](/img/event_sourcing.png)

**Pros**

- Highly scalable and performant
- Event-store provides strong audit capabilities
- Potential for high recoverability as events can be re-processed
- Easy to add new microservices that can read from and publish to the event-store
- Flexible pattern that can spur new ideas, creativity and opportunities.

**Cons**

- Arguably somewhat immature technologies, but gaining a lot of traction
- Can be more complex to implement

---

### Shared metadata-libraries
This is an old but useful pattern, even in a distributed microservice world. Read-only and immutable metadata is stored in libraries and used by the different microservices. Examples: Countries, states, colours, etc.

![Metadata libraries](/img/metadata_libraries.png)

**Pros**

- Very well-understood and easy to implement
- Productivity gain
- Shared model across services

**Cons**

- Change in data often requires each services to be re-built.

---

### Closing words…
As always there’s no [silver bullet](https://acntech.no/the-silver-bullet-obsession-in-software-development/) when it comes to managing data in microservices and software architecture is an evolutionary process as well. It’s important to choose the right tools for the job and different use-cases requires different patterns. Some data needs some degree of synchronous communication and some data fits more closely with asynchronous communication. It’s highly likely that you’ll find data in both categories, and data that’s somewhere  in between, in every software system. Each of the patterns can easily be used in conjunction with each other.

One thing that’s certain is that you have to have a bigger toolbox in the microservice-world and things like events and asynchronous communication should definitely be first-class citizens in that toolbox.

This post was inspired by a great talk by [Randy Shoup](https://twitter.com/randyshoup) at QCon New York 2017: [Managing data in microservices](https://www.infoq.com/presentations/microservices-data-centric)