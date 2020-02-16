---
title:  "Serverless Integration with Camel K"
modified: 2020-02-24 12:30:00 +0100
last_modified_at: 2020-02-24 12:30:00 +0100
tags: [Knative, Apache Camel, Openshift, Kubernetes, Serverless, JBoss Fuse]
categories: [Dev]
header:
    image: post-logo-apache-camel-k-native.png
    teaser: post-logo-apache-camel-k-native.png
---

Apache Camel K is reaching its maturity with current 1.0.0-RC2 release, just one step away from 1.0.0.
We've been working hard in the past months to add awesome features to Camel K, but also to improve its stability
and performance. This post contains a list of cool stuff that you'll find in latest release.

First of all, if you're living under a rock and it's the first time you hear about Camel K, 
you can read some introductory blog posts here ([1](/2018/10/15/introducing-camel-k/), [2](/2018/10/15/camel-k-on-knative/))
or look at the [new Apache Camel website](https://camel.apache.org/) that contains a [Camel K section](https://camel.apache.org/camel-k/latest/)
with a lot of material that is automatically generated from the [Github repository](https://github.com/apache/camel-k).

## IDE integration

Camel K development model is minimalistic: you need just to write a single file with your integration routes and you can immediately run it on any Kubernetes cluster. This way of defining things is common to many FaaS platforms (although Camel K is not properly a FaaS platform, but a lightweight integration platform) and it's technically difficult to provide IDE support, such as code completion and other utilities, to developers.

But now we've it. The XXX team has created some cool extensions for VS Code that make the development experience with Camel K even more exciting. You don't need to remember the Camel DSL syntax, the IDE will give you suggestions and error highlighting.

Code completion is not only limited to Java code, you also have suggestions and documentation out of the box when writing the Camel URIs and property files. And you can also run integrations and monitoring them using the IDE and nothing else.

Just install the VS Code XXX pack to have all these cool features available.

## Serverless

Serverless is the most important keyword FIXME when thinking to Camel K. Although you can have a wonderful Camel K experience
even without serverless features (e.g. if your cluster does not ship [Knative](https://knative.dev)), it's an area where we've
done a lot of improvements in recent months.



## Fast startup and low memory 

Camel 3.1.0
Quarkus

## Fast build times

## Better CLI

## Master routes

## CronJobs

