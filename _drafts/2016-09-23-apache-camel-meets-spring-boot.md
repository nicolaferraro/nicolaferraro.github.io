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
Next version of Apache Camel (2.18.0) is about to come with a long series of new features and components,
but the great news here is that much effort has been made to improve its compatibility with *Spring-Boot*.

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

**But, what does it mean being optimized for Spring-Boot?**

Spring-Boot is an opinionated framework, meaning that developers using it are choosing a specific set of technologies,
products and patterns to develop their applications. One of the implications is that Spring-Boot "requires" you to use
specific versions of some third party artifacts to play well in the ecosystem.

Starters bring with them a set of transitive dependencies perfectly compatible
with the ones expected by the Spring-Boot framework. Versions are aligned with latest version of Spring-Boot (currently `1.4.1.RELEASE`),
logging is configured to match the Spring-Boot standard logging framework, and so on.
Starters are meant to be included in a `pom.xml` and used, without hassle.

A new BOM (Bill of Material) is provided with the new version of Camel. So, the new configuration for the managed dependencies will be:

```xml
...
  <dependencyManagement>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>1.4.1.RELEASE</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
    <dependency>
      <groupId>org.apache.camel</groupId>
      <artifactId>camel-spring-boot-dependencies</artifactId>
      <version>2.18.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencyManagement>
...
```
The new `camel-spring-boot-dependencies` BOM has no conflicts with the Spring framework's `spring-boot-dependencies` BOM.
This was not the case with classical Camel BOM (`camel-parent`). Since Camel now supports several platforms, a separate BOM was necessary.

## Configuration of Starters
One of the most important features included in the starters is the possibility to configure Camel components using the new Spring-Boot configuration mechanism.
