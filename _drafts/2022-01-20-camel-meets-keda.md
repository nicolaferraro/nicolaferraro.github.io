---
title:  "Camel meets KEDA"
modified: 2022-01-20T02:00:00+02:00
last_modified_at: 2022-01-20T02:00:00+02:00
tags: [Apache Camel, Kamelets, Camel K, Kubernetes, Openshift, KEDA, JBoss Fuse]
categories: [Dev]
excerpt: "Camel K integrations can automatically configure KEDA autoscalers to promptly respond to load."
header:
    og_image: images/post-logo-camel-keda.png
    image: images/post-logo-camel-keda.png
    teaser: images/post-logo-camel-keda.png
---

[KEDA](https://keda.sh) (Kubernetes Event Driven Autoscalers) is a fantastic project ([CNCF](https://cncf.io) incubating) that provides Kubernetes-based autoscalers to help applications scaling out according to the demand when they are listening to several kinds of event sources. In Camel K we've long supported [Knative](https://knative.dev) for providing a similar functionality when integrations are triggered by HTTP calls, so supporting KEDA was something planned since long time, since it enables full autoscaling from a wider collection of sources. The KEDA integration has now been merged and it will be available in **Camel K 1.8.0**. This post will explain some details of the solution. If you're impatient, you can just jump to the end to see the [demo recording](#demo)

## Why KEDA and Camel K?

Users may want to setup integrations that automatically scale, e.g. depending on the **number of messages left in a Kafka topic** (for their consumer group). The integration code may do e.g. transformations, aggregations and send data do a destination and it would be great if the number of instances deployed to a Kubernetes cluster could increase when there's work left behind, or they can **scale to zero** when there's no more data in the topic.
This is what KEDA does by itself with scalers (Kafka is [one of the many scalers available in KEDA](https://keda.sh/docs/2.5/scalers/)). What you have now is that KEDA is now automatically configured by Camel K when you run integrations, so you get the autoscaling features out of the box (you just need to turn a flag on).

## How does it work?

In Camel K 1.8.0 a new [KEDA trait](https://camel.apache.org/camel-k/next/traits/keda.html) has been introduced. 
The trait allows to manually tweak KEDA configuration to make sure that some *ScaledObjects* (KEDA concept) are generated as part of the *Integration* reconciliation, but this is mostly an internal detail. The interesting part about the KEDA trait is that it can recognize special KEDA markers in Kamelets and automatically create a KEDA valid configuration when those Kamelets are used as sources. So users can use Kamelets to create bindings as usual and, if they **just enable a KEDA flag** via an annotation, they get an event driven autoscaler automatically configured.

The Kamelet catalog embedded in next release ([v0.7.0](https://github.com/apache/camel-kamelets/tree/v0.7.0)) contains two Kamelets enhanced with KEDA metadata: `aws-sqs-source` and `kafka-source`. These are just two examples of the many Kamelets that can be augmented in the future. The metadata configuration system is open and and Kamelets can be marked at any time to work with KEDA: this means that you don't need to wait for a new Camel K release to enable KEDA on a different source, but more importantly, also that you can mark your own Kamelets with KEDA metadata to enable autoscaling from your internal organization components.
The Kamelet developer guide contains a new section on [how to mark a Kamelet with KEDA metadata](https://camel.apache.org/camel-k/next/kamelets/kamelets-dev.html#_keda_integration), but essentially, markers are used to map Kamelet properties into KEDA configuration options, so that when you provide a Kamelet configuration, the corresponding KEDA options can be generated from it.

## A binding example

Before seeing the demo, the 

## Demo


