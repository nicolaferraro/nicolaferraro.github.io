---
title:  "Apache Camel meets Spring-Boot"
modified: 2016-09-23 23:00:00 +0200
last_modified_at: 2016-09-23 23:00:00 +0200
tags: [Apache Camel, JBoss Fuse, Microservices, Spring Boot]
categories: [Dev]
header:
    image: post-logo-camel.png
    teaser: post-logo-camel.png
---
Next version of Apache Camel (2.18.0) is about to come with a long list of new features and components.

Even if previous release of Camel was `2.17.3`, the new version is actually a major release, containing a 
lot of improvements over previous version.

Now Camel compiles against **Java 8** and most part of the code has been allowed to use constructs like lambdas, 
but also the API interfaces have been adapted to support lambdas in the user code, eg. in transformations.
 
Camel `2.18.0` comes with a new set of components ideal for building microservices. The list contains the new circuit breakers 
in `camel-hystrix`, load balancing with `camel-ribbon`, distributed tracing with `camel-zipkin`. 
These are just some of the new components of a long list that include also my pet component `camel-telegram`, 
to quickly create applications integrated with the popular messaging app.

A Camel Spring-Boot application now automatically provides reliable health checks at the endpoint `/health`, 
leveraging the Spring-Boot actuator module. 
This is a fundamental requirement for any microservice living in a cloud environment, because it allows the cloud provider (eg. Kubernetes) 
to detect any anomaly and take corrective actions. 

Indeed, one of the big news in the context of Microservices is that much effort has been made to improve its compatibility with **Spring-Boot**.

Spring-Boot is one of the top trending frameworks in the *Microservices* era,
one of the fundamental building blocks for creating cloud enabled applications, at least in the Java universe.
Camel integration with Spring has ancient roots and recent releases included some features that simplified
the usage of Camel into Spring-Boot applications, but *this time we have done much more*.

## Camel Spring-Boot Starters
We have done a major refactoring of the code, separating the pure components from platform related features.
In the case of Spring-Boot, this led to the creation of **Component Starters**.

A component starter is just like a normal Camel component, but it's *optimized for the Spring-Boot framework*.
Maven users should just add a `-starter` suffix to the name of the component to switch to its starter version:

```xml
...
  <dependency>
    <groupId>org.apache.camel</groupId>
    <artifactId>camel-http-starter</artifactId>
    <version>2.18.0</version>
  </dependency>
...
```

*But, what does it mean being optimized for Spring-Boot?*

Spring-Boot is an opinionated framework, meaning that developers using it are choosing a specific set of technologies,
products and patterns to develop their applications. One of the implications is that Spring-Boot "requires" you to use
specific versions of some third party artifacts to play well in the ecosystem.

Starters bring with them a set of transitive dependencies perfectly compatible
with the ones expected by the Spring-Boot framework. Versions are aligned with latest version of Spring-Boot (currently `1.4.1.RELEASE`),
logging is configured to match the Spring-Boot standard logging framework, and so on.
Starters are meant to be included in a `pom.xml` and used, without hassle.

A new BOM (Bill of Material) is provided with the new version of Camel: `camel-spring-boot-dependencies`. It has no conflicts with the Spring framework's `spring-boot-dependencies` BOM
and it replace the classical Camel BOM (`camel-parent`) in Spring-Boot applications.

## Configuration of Starters
One of the most important features included in the starters is the possibility to configure Camel components using the new Spring-Boot configuration mechanism.

Once a starter is included in the `pom.xml` file, the IDE will suggest all the configuration options available for that starter.

![Auto Configuration](/images/spring-boot-configuration.gif){: .align-center}

This helps configuring the application easily, using the `application.properties` file.

A related feature that spring-boot offers out of the box is the possibility to override such configuration using environment variables or
Java options. This is a great feature when you plan to run the application in a cloud platform like **Kubernetes** or **Openshift**,
since you can exploit it to override some environment related settings using environment variables set up by the cloud provider automatically (eg. external services).
A Kunernetes' **ConfigMap** can also be bound to environment variables, providing a way to inject external configuration into the application.

Trends are changing but camels are creatures that can adapt well to every environment! 
