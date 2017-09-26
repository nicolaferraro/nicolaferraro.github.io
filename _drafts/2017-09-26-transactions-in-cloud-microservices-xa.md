---
title:  "Transactions in Cloud Microservices: XA"
modified: 2017-09-26 09:15:00 +0200
last_modified_at: 2017-09-26 09:15:00 +0200
tags: [Kubernetes, Openshift, Narayana, Spring-Boot]
categories: [Dev]
header:
    image: post-logo-kubernetes.png
    teaser: post-logo-kubernetes.png
---
One of the major concerns that many developers have, when they start to look at microservices and cloud technologies is: *"how do we handle transactions now?"* 
And the response to this question always looks like: *"you need to do things in a novel way!"*, *"now your system is distributed, think to the CAP theorem!"* ...
And similar bullshit.

You may say that distributed RPC is something that belongs to the past, that we'll never see transaction servers again (although I'm not sure ;D).
But transactions are perceived as a requirement for many kind of applications.
Nobody would migrate an existing application to a microservice architecture if **that means paying a dozen of people to resubmit failed requests into the system, manually!**
And the same applies for greenfield projects. If the "old concept of transaction" cannot be used in the new architectures, people need a **realistic alternative**.

Before diving into what can be done to handle transactions in cloud microservices, let's talk a bit about the old concepts.

## And then there was ACID

As you know, a transaction is a sequence of operations respecting the ACID properties: 
*Atomicity* (i.e. all operations or none), 
*Consistency* (i.e. data is changed from a valid state into another valid state), 
*Isolation* (i.e. transactions cannot be influenced by other concurrent transactions) and 
*Durability* (i.e. persistent once it's committed).

Implementing an ACID transaction into a single database is now a easy task and most database vendors offer support for them since many years.
The problem with ACID transactions started to arise about 10 years ago with the advent into the hype of NoSQL databases: they are distributed, 
so they're subject to the CAP theorem that basically states that no distributed system can obtain full *Availability* and *Consistency* at the same time.

In the end, the CAP theorem practically means that **any distributed database that wants to respect the ACID properties, cannot be always available.
And it [holds for everybody except Google](https://research.google.com/pubs/pub45855.html)**.

But this is not a post about SQL vs NoSQL. Let's talk about microservices.

## Microservices are distributed

A picture here is better than a thousand words. What happens when we shift from a monolithic architecture to microservices?

--- picture monolith with 2 domains into 2 ms ---

This is what happens. The business logic that once lived in a single database is spread across multiple databases.
**And now, if you want to keep the old process "as is", you need distributed transactions**. 

Or you can *refactor your old process*. The idea here is: maybe ACID is too strong, you probably don't need every property
guaranteed by a transaction and you can split things *into multiple stages*.

Another idea is using *event sourcing*. In many cases event sourcing is just a synonym for delaying the problem and facing it later, 
but sometimes it does solve the problem. Anyway, using event sourcing is more than a technological shift. We will talk about it in another post.

A possible refactoring looks like this:

--- picture put a message channel between the microservices ---

